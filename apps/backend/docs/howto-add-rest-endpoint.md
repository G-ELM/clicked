# How to add a REST endpoint

The conventions a new route is expected to follow, with one worked example from schema to test. Every pattern below is lifted from a route that already exists; where the codebase is inconsistent, this page says which behaviour to copy.

## Checklist

A new route is done when it has all of these:

1. A router file under `src/routes/`, exporting a typed `Router`.
2. A mount line in `src/app.ts`. **Without this the route does not exist** (see [the cautionary example](#cautionary-example-auditlogsts)).
3. `requireAuth` unless the endpoint is deliberately public.
4. A Zod schema in `src/schemas/`, passed to `validate()`, for any request body.
5. A rate limiter, with its bucket declared in `src/config/rateLimits.ts`.
6. A `recordAuditEvent` call when the action is security-relevant.
7. Errors in the standard `{ error }` shape.
8. Tests in `src/__tests__/` that mock Redis and Postgres.
9. A row in the REST docs (`docs/api-*.md`, and `docs/contracts-error-catalog.md` for any new error string).

## Worked example: rename a device

`PUT /device-labels/:deviceId` lets a signed-in user rename one of their own devices. It is small, authenticated, takes a body, and is mildly security-relevant: an attacker with a stolen session could rename their own device to look harmless in the device list.

This is an illustrative example, not a shipped endpoint. It uses real columns (`devices.deviceName`) and real helpers.

### 1. Schema: `src/schemas/deviceLabel.schemas.ts`

```ts
import { z } from 'zod';

export const RenameDeviceSchema = z.object({
  deviceName: z
    .string()
    .trim()
    .min(1, 'deviceName is required')
    .max(100, 'deviceName must be at most 100 characters'),
});
```

Mirror the style of `DeviceSchema` in `src/schemas/auth.schemas.ts`: one message per rule, so the `issues[]` array tells the client exactly what to fix.

`validate()` (`src/middleware/validate.ts`) parses **`req.body` only**. It replaces `req.body` with the parsed value, so handlers read the cleaned data. Route params and query strings are not validated by it; check them in the handler (see step 2).

### 2. Router: `src/routes/deviceLabels.ts`

```ts
import { Router, type Router as RouterType } from 'express';
import { and, eq, isNull } from 'drizzle-orm';
import { db } from '../db/index.js';
import { devices } from '../db/schema.js';
import { requireAuth, type AuthRequest } from '../middleware/auth.js';
import { validate } from '../middleware/validate.js';
import { rateLimit } from '../middleware/rateLimit.js';
import { RenameDeviceSchema } from '../schemas/deviceLabel.schemas.js';
import { actorFromRequest, recordAuditEvent } from '../services/auditLog.js';

export const deviceLabelsRouter: RouterType = Router();

deviceLabelsRouter.use(requireAuth);

deviceLabelsRouter.put(
  '/:deviceId',
  rateLimit('device_rename'),
  validate(RenameDeviceSchema),
  async (req: AuthRequest, res) => {
    const deviceId = req.params['deviceId'] as string;
    const callerId = req.auth!.userId;
    const { deviceName } = req.body as { deviceName: string };

    try {
      const device = await db.query.devices.findFirst({
        where: and(
          eq(devices.id, deviceId),
          eq(devices.userId, callerId),
          isNull(devices.revokedAt),
        ),
      });

      if (!device) {
        res.status(404).json({ error: 'Device not found' });
        return;
      }

      await db.update(devices).set({ deviceName }).where(eq(devices.id, deviceId));

      void recordAuditEvent({
        action: 'device_renamed',
        ...actorFromRequest(req),
        targetType: 'device',
        targetId: deviceId,
        metadata: { previousName: device.deviceName, newName: deviceName },
      });

      res.json({ id: deviceId, deviceName });
    } catch {
      res.status(500).json({ error: 'Failed to rename device' });
    }
  },
);
```

Points that are conventions, not accidents:

- **`requireAuth` once, at the top of the router** (`router.use`), so a route added later cannot forget it. `auditLogs.ts` and `files.ts` do the same.
- **Middleware order is auth, then rate limit, then validation.** `rateLimit` keys on `user:<id>` once `req.auth` is set and falls back to the IP otherwise (`defaultIdentifier`), so it must run after `requireAuth` for a per-account limit. Validation goes last so a rejected body still costs the caller budget.
- **Scope every query to the caller.** The `userId` condition is the authorization check. A device that belongs to someone else returns the same `404` as one that does not exist, which avoids confirming that an id is real.
- **Wrap the handler body in `try`/`catch`.** There is no global Express error handler, so an uncaught exception returns Express's HTML 500 page instead of JSON.
- **`void recordAuditEvent(...)`** (see [Audit events](#audit-events)).

### 3. Rate limit bucket: `src/config/rateLimits.ts`

Add the bucket to `RATE_LIMIT_DEFAULTS`. `RateLimitBucket` is derived from its keys, so `rateLimit('device_rename')` only type-checks once this exists.

```ts
device_rename: {
  limit: 20,
  windowSeconds: MINUTE,
  description: 'Device label changes',
},
```

Operators can override it with `RATE_LIMIT_DEVICE_RENAME=<limit>[/<windowSeconds>]`; the bucket name is upper-cased to form the variable. Choose generous defaults: limits exist to stop abuse, not to shape normal use. A rejected call returns `429` with `{ error: 'Too many requests', bucket, retryAfterSeconds }` and `RateLimit-*` / `Retry-After` headers.

### 4. Mount it in `src/app.ts`

```ts
import { deviceLabelsRouter } from './routes/deviceLabels.js';
// ...
app.use('/device-labels', deviceLabelsRouter);
```

Put the line with the other `app.use('/…', …Router)` mounts, **after** `app.use(rateLimit('global_ip', …))` so the catch-all per-IP ceiling covers it. Do not put it after the `/local-storage` block or the `/metrics` handler.

### 5. Audit action (only because this example records one)

`recordAuditEvent` takes an `AuditAction`, which is the Postgres enum `audit_action` (`auditActionEnum` in `src/db/schema.ts`). Add the value and generate a migration:

```ts
export const auditActionEnum = pgEnum('audit_action', [
  // ...existing values
  'device_renamed',
]);
```

Then follow [`migrations.md`](./migrations.md): `pnpm db:generate`, read the SQL, commit schema and generated files together. If the action is not in the enum, TypeScript rejects the call.

### 6. Test: `src/__tests__/deviceLabels.test.ts`

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import request from 'supertest';
import express from 'express';

const mockFindFirst = vi.fn();
const mockWhere = vi.fn().mockResolvedValue(undefined);
const mockSet = vi.fn(() => ({ where: mockWhere }));
const mockUpdate = vi.fn(() => ({ set: mockSet }));
const mockRecordAuditEvent = vi.fn().mockResolvedValue(undefined);

vi.mock('../db/index.js', () => ({
  db: { query: { devices: { findFirst: mockFindFirst } }, update: mockUpdate },
}));

vi.mock('../db/schema.js', () => ({
  devices: { id: 'id', userId: 'userId', revokedAt: 'revokedAt', deviceName: 'deviceName' },
}));

vi.mock('drizzle-orm', () => ({
  and: vi.fn((...args: unknown[]) => args),
  eq: vi.fn((col: unknown, val: unknown) => ({ col, val })),
  isNull: vi.fn((col: unknown) => ({ isNull: col })),
}));

vi.mock('../lib/redis.js', () => ({
  get redis() {
    return null;
  },
}));

vi.mock('../middleware/auth.js', () => ({
  requireAuth: (req: express.Request, _res: express.Response, next: express.NextFunction) => {
    (req as express.Request & { auth: { userId: string; deviceId: string } }).auth = {
      userId: 'user-1',
      deviceId: 'device-1',
    };
    next();
  },
}));

vi.mock('../services/auditLog.js', () => ({
  recordAuditEvent: mockRecordAuditEvent,
  actorFromRequest: () => ({ actorUserId: 'user-1', actorDeviceId: 'device-1' }),
}));

const { deviceLabelsRouter } = await import('../routes/deviceLabels.js');
const { clearLocalRateLimitCounters } = await import('../services/rateLimiter.js');

function makeApp() {
  const app = express();
  app.use(express.json());
  app.use('/device-labels', deviceLabelsRouter);
  return app;
}

beforeEach(() => {
  vi.clearAllMocks();
  clearLocalRateLimitCounters();
});

describe('PUT /device-labels/:deviceId', () => {
  it('renames an owned device and records an audit event', async () => {
    mockFindFirst.mockResolvedValue({ id: 'dev-1', deviceName: 'Old' });

    const res = await request(makeApp()).put('/device-labels/dev-1').send({ deviceName: 'Laptop' });

    expect(res.status).toBe(200);
    expect(res.body).toEqual({ id: 'dev-1', deviceName: 'Laptop' });
    expect(mockSet).toHaveBeenCalledWith({ deviceName: 'Laptop' });
    expect(mockRecordAuditEvent).toHaveBeenCalledWith(
      expect.objectContaining({
        action: 'device_renamed',
        targetId: 'dev-1',
        metadata: { previousName: 'Old', newName: 'Laptop' },
      }),
    );
  });

  it('rejects an empty name with the validation shape', async () => {
    const res = await request(makeApp()).put('/device-labels/dev-1').send({ deviceName: '  ' });

    expect(res.status).toBe(400);
    expect(res.body.error).toBe('Validation failed');
    expect(res.body.issues[0]).toEqual({ field: 'deviceName', message: 'deviceName is required' });
    expect(mockUpdate).not.toHaveBeenCalled();
  });

  it('returns 404 for a device the caller does not own', async () => {
    mockFindFirst.mockResolvedValue(undefined);

    const res = await request(makeApp()).put('/device-labels/other').send({ deviceName: 'X' });

    expect(res.status).toBe(404);
    expect(res.body).toEqual({ error: 'Device not found' });
    expect(mockRecordAuditEvent).not.toHaveBeenCalled();
  });
});
```

Add a separate test file or `describe` that mounts the router **without** the `middleware/auth.js` mock and asserts a missing `Authorization` header returns `401`; the testing guide says auth behaviour is tested with the real middleware.

Run it with `pnpm --filter backend test -- deviceLabels`.

## Cautionary example: `auditLogs.ts`

`src/routes/auditLogs.ts` defines `auditLogsRouter` with `GET /` (a user's own security history) and a passing test, `src/__tests__/auditLog.test.ts`. The test builds its own Express app and mounts the router itself:

```ts
app.use('/audit-logs', auditLogsRouter);
```

`src/app.ts` never imports or mounts it, so `GET /audit-logs` returns `404` in a running server and the tests stay green. This is the failure the mount step prevents: a route test that mounts its own router cannot tell you the real app mounts it.

It also shows two gaps against the checklist above: the route has no rate limiter, and it parses `limit`, `cursor` and `action` by hand instead of through a schema.

How to avoid repeating it:

- After adding a router, grep for it: `grep -n "deviceLabelsRouter" apps/backend/src/app.ts` must find an import **and** an `app.use`.
- For anything user-facing, add at least one test that imports `app` from `../app.js` and requests the real path (see `src/__tests__/health.test.ts` for the pattern).

The unmounted route is already called out in the [error catalog](./contracts-error-catalog.md) (note 2 under "The common shape").

## Audit events

Call `recordAuditEvent` (`src/services/auditLog.ts`) when an action changes who can access an account or what an incident responder would need to reconstruct a compromise. The existing actions are:

| Action                                       | Recorded when                                          |
| -------------------------------------------- | ------------------------------------------------------ |
| `device_linked`, `device_revoked`            | Device lifecycle changes                               |
| `logout_everywhere`                          | All sessions invalidated                               |
| `key_bundle_drained`                         | Someone fetched a prekey bundle for an account         |
| `auth_failed`                                | A valid credential was replayed after losing authority |
| `file_access_denied`                         | A user was refused access to a file                    |
| `group_member_added`, `group_member_removed` | Group membership changed                               |

Do not audit ordinary reads, message sends, or anything high volume: an attacker who can trigger an event for free can flood the table. `requireAuth` deliberately audits only a validly signed token whose device is revoked, not malformed or expired tokens, for this reason.

Usage rules:

- Spread `...actorFromRequest(req)` for routes behind `requireAuth`. It fills `actorUserId`, `actorDeviceId`, `ipAddress` and `userAgent`. Before authentication, use `requestContext(req)` instead.
- Call it **after** the action succeeds, and write `void recordAuditEvent(...)`. It never throws or rejects: a failed write is logged and swallowed so an audit outage cannot break the security control it observes.
- Set `targetType` / `targetId` to what was acted on, and `subjectUserId` when the event concerns a different account than the actor's (it defaults to the actor).

### Which metadata is safe

The log must never contain message content. `sanitizeMetadata` enforces a safety net, but do not rely on it:

- **Safe:** ids, counts, booleans, enum-like labels, platform names, device names, whether something was a self-action (`selfRevocation`), remaining-device counts.
- **Never record:** plaintext, ciphertext, envelopes, request bodies, tokens, secrets, passwords, signatures, private keys, prekeys. Keys whose lowercased name (ignoring `_` and `-`) contains `ciphertext`, `plaintext`, `content`, `message`, `body`, `text`, `envelope`, `payload`, `token`, `secret`, `password`, `signature`, `privatekey` or `prekey` are replaced with `"[redacted]"`, which also means a legitimate key such as `messageCount` is redacted; pick another name.
- **Limits per row:** 20 keys, strings truncated at 256 characters, arrays at 20 entries, one level of nesting. Never spread `req.body` into `metadata`.

The rows are append-only, enforced by a database trigger, and are readable by the account through `GET /audit-logs` once that route is mounted.

## Error responses

REST errors are `{ "error": string }` with the HTTP status carrying most of the meaning. Extra fields sit beside `error`, never nested under it.

| Situation             | Status | Body                                                                             |
| --------------------- | ------ | -------------------------------------------------------------------------------- |
| Missing or bad token  | `401`  | `{ error: 'Missing or invalid Authorization header' }` (from `requireAuth`)      |
| Body fails the schema | `400`  | `{ error: 'Validation failed', issues: [{ field, message }] }` (from `validate`) |
| Not yours / not found | `404`  | `{ error: 'Device not found' }`                                                  |
| Conflicts with state  | `409`  | `{ error: 'Cannot revoke your only active device' }`                             |
| Over the rate limit   | `429`  | `{ error: 'Too many requests', bucket, retryAfterSeconds }`                      |
| Unexpected failure    | `500`  | `{ error: 'Failed to …' }`, produced by your own `catch`                         |

Match on status first and the string second. Any new error string must be added to the [error catalog](./contracts-error-catalog.md) in the same PR so clients can find it. Do not leak internal detail (SQL errors, stack traces) in `error`.

## Test conventions

These follow the [backend testing guide](./testing.md), which in turn follows the repo-wide [testing strategy](../../../docs/testing.md). The short version:

- **Never a real service.** No test starts Postgres, Redis or S3. Mock `../db/index.js`, `../db/schema.js`, `drizzle-orm`, and `../lib/redis.js` (return `null` for the degraded path unless the test is about caching).
- **Import the router after the mocks** with a top-level `await import(...)`, never a static import.
- **Mount only the router under test** in a throwaway Express app, plus one test through `app` to prove the mount.
- **Mock `requireAuth`** to inject `req.auth = { userId, deviceId }` for tests about something else, and use the real middleware for auth tests.
- **Do not mock `validate()`.** Exercise the real `400` path.
- **Mock `../services/auditLog.js`** unless the test is about auditing, and assert on the call, including that forbidden keys are absent from `metadata`.
- **Reset rate-limit state** in `beforeEach` with `clearLocalRateLimitCounters()` (and `resetRateLimitBucket(bucket)` when Redis is real or mocked), or one test's calls count against the next.
- **Any new table or Drizzle operator** the route uses must be added to the `db/schema.js` and `drizzle-orm` mocks, or the failure appears as an import error far from the cause.
- Files live in `src/__tests__/*.test.ts`; lint and format cover tests too (`pnpm --filter backend lint`, `format:check`).
