# How to add a socket event end-to-end

Source: [`src/lib/eventEnvelope.ts`](../src/lib/eventEnvelope.ts),
[`src/socket/dispatcher.ts`](../src/socket/dispatcher.ts),
[`src/socket/messaging.ts`](../src/socket/messaging.ts),
[`src/config/rateLimits.ts`](../src/config/rateLimits.ts),
[`src/index.ts`](../src/index.ts) (per-socket middleware).

Adding an inbound event touches several places, and **missing one fails silently**. An event that is not in
`KNOWN_EVENT_TYPES` is rejected with an error. An event that is known but has no registered handler is dropped
with only a server-side `console.warn`: the client gets no error, no `dispatch_ack`, nothing. There is no raw
`socket.on(type, ...)` fallback (#342), so a bare `socket.emit('my_event', ...)` never reaches a handler either.

## Checklist

Work through these in order.

1. **Register the type** in `KNOWN_EVENT_TYPES` in `src/lib/eventEnvelope.ts`. Put inbound (client to server) types
   under the inbound comment and outbound ones under the outbound comment. Without this the dispatcher answers with
   `Unknown event type: <type>` and never looks for a handler.
2. **Write and register the handler** inside `registerMessagingHandlers` in `src/socket/messaging.ts` with
   `dispatcher.register('<type>', async (payload) => { ... })`. Register before the `dispatcher.listen()` call at
   the end of the function. A handler that is never registered is the silent-drop case above.
3. **Run the payload invariant checks** as the first statement of the handler if it carries a client payload that
   could hold key or session material (see [Invariants](#invariants-every-payload-carrying-handler-must-run)).
4. **Map the event to a rate-limit bucket** in `SOCKET_EVENT_BUCKETS` in `src/config/rateLimits.ts` if it should not use
   `socket_default`, adding a new bucket to `RATE_LIMIT_DEFAULTS` when none fits (see
   [Rate limiting](#rate-limiting-and-the-socket_event_buckets-mapping)).
5. **Emit errors** in the shape the [error catalog](contracts-error-catalog.md#socket-errors-come-in-two-shapes) documents.
6. **Add a unit test** with the dispatcher harness (see [Testing](#testing-the-handler)).
7. **Update the docs** (see [Update the docs](#update-the-docs)).

If the event is outbound only (server to client), step 1 is still required so the registry stays the single source of
truth, but there is no handler to register. Emit it with `socket.emit(...)` / `io.to(room).emit(...)` or, for the
enveloped form, `dispatcher.emit(type, payload)`.

## Worked example: `star_message`

A client marks a message as starred. Minimal, for illustration only.

**1. `src/lib/eventEnvelope.ts`**

```ts
export const KNOWN_EVENT_TYPES = new Set([
  // Inbound (client → server)
  'auth',
  // ...existing types...
  'star_message',
```

**2 and 3. `src/socket/messaging.ts`**, next to the other `dispatcher.register` calls:

```ts
dispatcher.register('star_message', async (payload) => {
  const forbiddenField = findForbiddenSessionStateField(payload);
  if (forbiddenField) {
    socket.emit('error', {
      event: 'star_message',
      code: 400,
      message: `Field "${forbiddenField}" is not permitted: the server never stores session or private-key state`,
    });
    return;
  }

  const { conversationId, messageId } = payload as { conversationId?: string; messageId?: string };
  if (!conversationId?.trim() || !messageId?.trim()) {
    socket.emit('error', {
      event: 'star_message',
      message: 'conversationId and messageId are required',
    });
    return;
  }

  const membership = await db.query.conversationMembers.findFirst({
    where: and(
      eq(conversationMembers.conversationId, conversationId),
      eq(conversationMembers.userId, userId),
    ),
  });
  if (!membership) {
    socket.emit('error', { event: 'star_message', message: 'Not a member of this conversation' });
    return;
  }

  // ...persist the star...
  socket.emit('star_message_ack', { conversationId, messageId });
});
```

`star_message_ack` would also need to be added to `KNOWN_EVENT_TYPES`, as an outbound type.

**4. `src/config/rateLimits.ts`**

```ts
const SOCKET_EVENT_BUCKETS: Record<string, RateLimitBucket> = {
  send_message: 'socket_send_message',
  // ...
  star_message: 'socket_typing', // pick, or add, a bucket whose budget fits
};
```

## What the dispatcher does before a handler runs

Clients send `socket.emit('dispatch', envelope)`. `EventDispatcher.listen()` runs these checks in order and stops at
the first failure. A failed check returns before any handler code executes. Failures 1 to 4 emit an enveloped
`error` (Shape B, see the [error catalog](contracts-error-catalog.md)).

| #   | Check                | Detail                                                                                                                                                                                                        | On failure                                                                |
| --- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 0   | Authenticated socket | `socket.auth` must be set                                                                                                                                                                                     | `error` payload `Unauthenticated`                                         |
| 1   | Envelope schema      | `EventEnvelopeSchema`: `eventId` non-empty string, `type` non-empty string, `timestamp` positive integer, `payload` an object (defaults to `{}`)                                                              | `Malformed envelope` with Zod `details`                                   |
| 2   | Known type           | `type` must be in `KNOWN_EVENT_TYPES`                                                                                                                                                                         | `Unknown event type: <type>` (also `console.warn`)                        |
| 3   | Timestamp freshness  | `now - SOCKET_EVENT_MAX_AGE_MS <= timestamp <= now + SOCKET_EVENT_MAX_FUTURE_SKEW_MS`. Defaults: **300000 ms (300 s)** into the past, **30000 ms (30 s)** into the future                                     | `Stale or invalid envelope timestamp`                                     |
| 4   | Replay rejection     | Redis `SET NX EX` on a key scoped to **`(deviceId, eventId)`**, TTL `REPLAY_PROTECTION_TTL_SECONDS` (default **300**, valid range 1 to 86400). Fails open when Redis is unavailable (or with no Redis client) | `dispatch_ack` `{ eventId, duplicate: true }`; the handler is **not** run |
| 5   | Handler lookup       | `handlers.get(type)`                                                                                                                                                                                          | `console.warn` only, **silent drop** (no ack, no error)                   |
| 6   | Run handler          | `await handler(payload)`                                                                                                                                                                                      | success: `dispatch_ack` `{ eventId, duplicate: false }`                   |

Consequences when writing a handler:

- Clients must generate a fresh `eventId` per emission and a current `Date.now()` `timestamp`. Reusing an `eventId`
  within the TTL from the same device produces `duplicate: true` and does not re-run the handler, so retry logic should
  reuse an `eventId` only when it is deliberately retrying the same event.
- The replay window should be at least as long as the freshness window to be a meaningful replay defence. With the
  defaults both are 300 s.
- If the handler **throws**, the dispatcher only logs `handler error for "<type>"` and swallows the exception. No
  `dispatch_ack` is sent and no error is emitted to the client. A handler therefore has to catch its own failures and
  emit the error itself (see [Errors](#emitting-errors)). Do not rely on a throw to tell the client something went wrong.
- Three checks happen **before** the dispatcher, in the per-socket middleware in `src/index.ts`, on every incoming
  Socket.IO event: a revoked-device check, the payload size cap (`MAX_PAYLOAD_SIZE`, default 16384 bytes), and the
  per-event rate limit.

## Rate limiting and the `SOCKET_EVENT_BUCKETS` mapping

`socketEventBucket(event)` returns the mapped bucket or `socket_default` (10 per second). The charge is made per
device (`device:<deviceId>`), shared across nodes through Redis. Existing buckets: `socket_default`,
`socket_send_message`, `socket_typing`, `socket_ask_assistant`.

The limiter runs in `socket.use(...)`, which sees the **Socket.IO event name**, not the envelope `type`. For
enveloped traffic that name is always `dispatch`, so `socketEventBucket('dispatch')` currently resolves to
`socket_default` for every enveloped event, regardless of the `SOCKET_EVENT_BUCKETS` entry for its `type`. Keep the
mapping up to date anyway, because it documents the intended budget and applies if the limiter is moved to see the
envelope type, but do not assume it is the effective limit today. Confirm in `src/index.ts` before relying on a
specific bucket, and see [rate limits](../../../docs/security/rate-limits.md).

## Invariants every payload-carrying handler must run

The server relays ciphertext and public routing metadata only. See
[server-side crypto invariants](concepts-crypto-invariants.md).

- **No session state.** Run `findForbiddenSessionStateField(payload)` from `lib/signalInvariants.ts` as the
  **first statement**, before destructuring the payload, any membership check, or any `db` call. If it returns a name,
  emit the error below and `return`. It checks the top level and every entry of `payload.envelopes`, and matches on
  key presence, so `null` and empty values are still rejected.
- **Ciphertext only.** Never accept, store or log plaintext, private keys, ratchet or session state. Message bodies must
  arrive as ciphertext, and the persistence path applies the ciphertext invariant (`lib/ciphertextInvariant.ts`).
  Do not add fields to a payload that would carry anything else.
- **Size.** If the payload contains per-device envelopes, also use `checkEnvelopeSizes` from `services/rateLimit.ts`
  (`MAX_ENVELOPE_SIZE`, default 4096 bytes per ciphertext), as `send_message` does.
- Do not log payload contents; log ids only.

A handler with no client-supplied payload (such as `heartbeat`) does not need the invariant checks.

## Emitting errors

Errors from a handler use the **bare** Shape A documented in the
[error catalog](contracts-error-catalog.md#socket-errors-come-in-two-shapes): an `error` event whose payload names the
failing client event.

```ts
socket.emit('error', {
  event: 'star_message', // the client event that failed
  message: 'Not a member of this conversation', // always present
  code: 400, // optional; see the catalog for the codes used
});
```

Rules:

- Always include `event` and `message`. Add `code` only for the cases the catalog already defines it for, such as the
  `400` for forbidden session-state fields.
- Reuse existing `message` strings for the same failure (`Not a member of this conversation`, `Invalid conversationId`)
  so clients can handle them uniformly.
- Do not use `createEnvelope('error', ...)` in a handler. That is Shape B and belongs to the dispatcher layer.
- `return` after emitting, and never throw to signal a client error (see above).
- Add any new error string to the catalog.

## Testing the handler

Dispatcher behaviour has its own harness in [`src/__tests__/dispatcher.test.ts`](../src/__tests__/dispatcher.test.ts)
(and `src/socket/dispatcher.spec.ts`, which uses `ioredis-mock`). The helpers are local to those files; copy them
into a new test file:

- `makeSocket(auth)` builds an `EventEmitter`-backed fake `AuthSocket` with a mocked `emit`, `join`, `disconnect`, an
  `emitted` array recording every emit, and a `trigger(event, data)` that fires listeners through the real emitter.
- `makeIo()` builds a minimal `Server` double and `makeRedis('OK' | null)` a stub whose `set` result drives the replay check.

```ts
import { describe, it, expect, vi } from 'vitest';
import { EventDispatcher } from '../socket/dispatcher.js';

it('runs the star_message handler through the envelope path', async () => {
  const { socket, trigger } = makeSocket();
  const dispatcher = new EventDispatcher(makeIo(), socket, makeRedis('OK') as never);
  const handler = vi.fn().mockResolvedValue(undefined);

  dispatcher.register('star_message', handler);
  dispatcher.listen();

  trigger('dispatch', {
    eventId: 'evt-1',
    type: 'star_message',
    timestamp: Date.now(),
    payload: { conversationId: 'c1', messageId: 'm1' },
  });

  await new Promise((r) => setTimeout(r, 10));
  expect(handler).toHaveBeenCalledWith({ conversationId: 'c1', messageId: 'm1' });
});
```

Test handler logic (not just routing) by calling `registerMessagingHandlers(io, socket)` with `vi.mock`ed `db`, as
[`signalInvariants.socket.test.ts`](../src/__tests__/signalInvariants.socket.test.ts) does, then `trigger('dispatch', ...)` and
assert on `emitted`. Cover at least: the happy path, the forbidden-field rejection (and that no `db` call happened), the
membership failure, and a `duplicate: true` replay. Also assert the type is in `KNOWN_EVENT_TYPES` so step 1 cannot be forgotten. Run with
`pnpm --filter backend test`.

## Update the docs

Finally, document the event so clients can use it:

- Add it to the **WebSocket event catalog**, [`api-websocket-events.md`](api-websocket-events.md) (inbound or outbound
  section, plus the table of contents).
- Add its payload schema to the **payload reference**, [`contracts-websocket-payloads.md`](contracts-websocket-payloads.md),
  including the `KNOWN_EVENT_TYPES` list in section 2.
- Add any new error strings to [`contracts-error-catalog.md`](contracts-error-catalog.md), and any new bucket to the
  [rate limit docs](../../../docs/security/rate-limits.md).
