# One-time prekey low-water signalling

Source: [`src/services/prekeyLowSignal.ts`](../src/services/prekeyLowSignal.ts),
[`src/routes/users.ts`](../src/routes/users.ts) (bundle fetch),
[`src/routes/devices.ts`](../src/routes/devices.ts) (prekey upload, revocation).

See also: [E2EE key management](concepts-e2ee-key-management.md),
[Devices API](api-devices.md#post-devicesidprekeys), [WebSocket events](api-websocket-events.md).

## Why the server signals

A device's one-time prekeys (OPKs) are consumed one per X3DH bundle fetch and are **never regenerated
server-side**, because the private halves exist only on the device. Once a device has none left, an
initiator receives a bundle without a one-time key and falls back from 4-DH to 3-DH, losing the forward
secrecy contribution of that key. The server cannot create keys, so it tells the device to top up
_before_ the pool reaches zero.

## When the signal fires

| Item         | Value                                                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Threshold    | `PREKEY_LOW_THRESHOLD`, default **20**. Read once at process start; must be a positive integer, otherwise the default is used. |
| Condition    | unconsumed OPKs remaining **below** the threshold (`remaining < threshold`)                                                    |
| Trigger path | the bundle-fetch route in `routes/users.ts`, **only when a one-time key was actually claimed**                                 |

The count only ever drops on a bundle fetch, so that is the single place the check runs. After the
claim transaction the route recounts the device's unconsumed OPKs and calls
`signalPrekeysLowIfNeeded(deviceId, remaining)` fire-and-forget: it never blocks, delays or fails the
bundle response, and never throws. A fetch that finds no OPK to claim (the pool is already empty) does not
re-run the check.

Nothing else emits it. There is no periodic sweep, no check on connect, and no check on upload other than
releasing the latch (below).

## The latch

Without a latch, a device sitting at 12 keys would be signalled on every one of the next dozen fetches. The
signal is therefore latched per device:

- **Redis key:** `prekeys:low:<deviceId>`, value `1`, **TTL 30 days** (`LATCH_TTL_SECONDS`).
- **Acquire:** one atomic `SET key 1 EX 2592000 NX`. Exactly one caller sees `OK` and emits, so two gateways racing
  on concurrent fetches still produce a single event.
- **Without Redis** (not configured, as in single-node dev and unit tests), or if the Redis call throws, an
  in-process `Set` provides the same semantics for that process only. It is not shared across nodes, so a
  multi-node deployment running without Redis may signal once per node, and the set is lost on restart.
- **Emit:** only the caller that acquired the latch emits. Everyone else returns silently.

### What releases the latch

| Event                                                                                     | Where                             |
| ----------------------------------------------------------------------------------------- | --------------------------------- |
| A `signalPrekeysLowIfNeeded` call that sees `remaining >= threshold`                      | `prekeyLowSignal.ts`              |
| A successful `POST /devices/:id/prekeys` after which the recounted pool is `>= threshold` | `routes/devices.ts`               |
| Device revocation (its prekeys are deleted, so there is nothing left to replenish)        | `revokeDeviceRow` in `devices.ts` |
| The 30-day TTL expiring (a device that has ignored the signal this long is told again)    | Redis                             |

Releasing deletes the Redis key and removes the id from the in-process set. A device that replenishes only
partway (for example from 5 to 15, still under 20) keeps the latch and is **not** signalled again until it
climbs back to the threshold and drops under it again, or the TTL passes. The upload route recounts
rather than adding up `current + uploaded`, because duplicate `keyId`s are dropped by `ON CONFLICT DO NOTHING`.

## The `prekeys_low` event

Delivered to the Socket.IO room `device:<deviceId>`, which each socket joins on connect
(`index.ts`). Only that device's own sockets get it, on whichever gateway holds the connection (via the
Redis adapter). Other devices on the same account do not, since only the owning device holds the private
keys it can replenish.

```json
{
  "deviceId": "b6b6c3b0-...",
  "oneTimePreKeysRemaining": 19,
  "threshold": 20
}
```

It is a plain server push: it is not an envelope, carries no `eventId`, and is not acknowledged.

### What the client should do

Upload a fresh batch with `POST /devices/:id/prekeys`, which takes `signedPreKey` and a non-empty
`oneTimePreKeys` array (see the [prekey upload contract](api-devices.md#prekey-upload-contract)):

- The server enforces a **cap of 200 unconsumed one-time prekeys per device** (`OTP_CAP`). An upload is
  trimmed to the remaining room (`200 - current`) and the response says so with `capped: true`; if the device is
  already at 200 the request fails with `422`.
- The server sets **no batch-size limit** other than that cap and requires at least one key. A client should
  generate enough to refill to the cap, `200 - oneTimePreKeysRemaining` keys, using new `keyId`s that do not
  collide with ones already stored, and send a fresh signed prekey signed by the identity key (the endpoint
  verifies the signature).
- Uploading fewer keys than needed to reach the threshold leaves the latch set and the device quiet, so top up
  well above `threshold`, not just to it.
- The `200` response returns `oneTimePreKeysRemaining`, the recount after the insert, so the client can confirm
  it is back above the threshold.

At the time of writing, no client code in this repository listens for `prekeys_low`; the contract above is
what the server expects.

## Offline devices and reconnecting

Socket.IO delivers `prekeys_low` only to sockets present in the room at the moment of the emit. It is **not
queued, buffered or replayed**. A device that is offline when the latch is acquired never receives the event,
and because the latch stays set, the server will not send another until it is released (replenish to the
threshold, revocation, or the 30-day TTL).

Therefore the event is a hint, not the source of truth. A client should check its own count:

1. On every (re)connect, and on app start, call `GET /devices` and read `oneTimePreKeysRemaining` for the entry
   with `current: true` (the count of unconsumed OPKs for that device).
2. If it is below the threshold (use 20, or the `threshold` from the last event), generate and upload a batch as
   above.
3. Keep listening for `prekeys_low` while connected, to react before the next reconnect.

Uploading releases the latch, which re-arms the signal for the next crossing.
