# Operational and utility endpoints

Source: [`src/app.ts`](../src/app.ts),
[`src/routes/security.ts`](../src/routes/security.ts),
[`src/routes/localStorage.ts`](../src/routes/localStorage.ts),
[`src/middleware/transportSecurity.ts`](../src/middleware/transportSecurity.ts),
[`src/middleware/rateLimit.ts`](../src/middleware/rateLimit.ts).

These routes are registered directly in `app.ts` (or mounted from a tiny router) rather than in
a feature router. They serve orchestrators, scrapers, mobile bootstrap and local development, not
product features.

## Summary

| Route                              | Auth                                  | Global IP rate limit | Plaintext over `http` when TLS is enforced |
| ---------------------------------- | ------------------------------------- | -------------------- | ------------------------------------------ |
| `GET /health`                      | none                                  | **exempt**           | allowed                                    |
| `GET /metrics`                     | none (see [protection](#get-metrics)) | applies              | rejected (`403 tls_required`)              |
| `GET /me`                          | `requireAuth` (Bearer JWT)            | applies              | rejected                                   |
| `GET /security/transport-policy`   | none                                  | applies              | rejected                                   |
| `/local-storage/*` (dev/test only) | signed URL (HMAC + expiry)            | applies              | n/a (not mounted in production)            |

### Rate-limiter ordering

`app.ts` registers `app.use(rateLimit('global_ip', { identifier: ipIdentifier }))` **after**
`GET /health` and **before** every other route. Express runs handlers in registration order, so:

- `/health` is registered first and answers before the limiter runs. It is never counted and never
  returns `429`. The comment in `app.ts` states the intent: orchestrator probes must not be rate
  limited.
- `/metrics`, `/me`, `/security/transport-policy` and `/local-storage/*` are all registered after the
  limiter, so each request is charged to the caller's IP in the `global_ip` bucket
  (default 600 requests per minute, `config/rateLimits.ts`). A `429` response is
  `{ "error": "Too many requests", "bucket": "global_ip", "retryAfterSeconds": n }` with
  `RateLimit-*` and `Retry-After` headers.

Consequence for scrapers: a Prometheus server scraping every 15 s uses 4 requests per minute per
instance, far below the ceiling, but a shared egress IP (for example many pods behind one NAT scraping
the same gateway) shares one budget.

### Transport-security middleware

`enforceTransportSecurity` runs before any route. When TLS is enforced (the default outside
`development`/`test`, see [`ENFORCE_TLS`](../../../docs/environment-variables.md)) it rejects
non-HTTPS requests with:

```json
{
  "error": "tls_required",
  "message": "This endpoint is only available over https. Reconnect using https:// or wss://."
}
```

The only path exempt is `/health` (`PLAINTEXT_ALLOWED_PATHS`). Load balancers health-check the pod
directly before TLS termination and a `403` would pull a healthy gateway out of rotation.
`enforceOriginPolicy` also runs first on every route: a request carrying an `Origin` header that is not
on the `ALLOWED_ORIGINS` allowlist gets `403 origin_not_allowed`. Requests without an `Origin`
header (curl, Prometheus, orchestrator probes) are not affected.

## `GET /health`

Liveness and readiness probe.

**Auth:** none. **Rate limit:** exempt. **TLS:** allowed over plaintext.

**Response `200`:**

```json
{
  "status": "ok",
  "db": "connected",
  "node": "v22.11.0",
  "version": "1.0.0"
}
```

**Response `503`** (degraded):

```json
{
  "status": "error",
  "db": "unreachable",
  "node": "v22.11.0",
  "version": "1.0.0"
}
```

### Semantics for orchestrators

- The only dependency checked is **PostgreSQL**, via `SELECT 1` through the Drizzle `db` client. Any
  error from that query produces the `503`. There is no query timeout of its own, so a hung database
  connection holds the probe until the orchestrator's own probe timeout fires.
- **Redis, the object store, Stellar/Horizon and the Socket.IO server are not checked.** Redis is
  treated as degradable throughout the gateway (rate limiting, presence and latches fall back to
  in-process state), so a Redis outage leaves `/health` at `200`.
- `status` and `db` are fixed literals: `ok`/`connected` on success, `error`/`unreachable` on failure.
  `node` is `process.version` and `version` is `version` from `apps/backend/package.json`, so both are
  always present, including in the degraded body.
- The route is cheap and unauthenticated, so use it for both liveness and readiness. Because it
  depends on the database, a pure **liveness** probe will restart pods during a database outage,
  which will not fix the outage. If that is a concern, point the liveness probe at a TCP check and
  keep `/health` for readiness.

## `GET /metrics`

Prometheus scrape endpoint (#393).

**Auth:** none in the application. **Rate limit:** applies (`global_ip`).
**Response `200`:** Prometheus text exposition, `Content-Type` taken from
`registry.contentType` (`text/plain; version=0.0.4; charset=utf-8`). It contains the application
counters, gauges and histograms defined in `lib/metrics.ts` plus the default Node process metrics.
The metric list is maintained in [`docs/observability.md`](../../../docs/observability.md#metrics-reference);
it is not repeated here.

Metrics carry no message content: `lib/metrics.ts` takes no ciphertext or free-text labels.
They still reveal traffic volume, instance count, version/runtime details and per-instance load, so
they should not be public.

### Protecting `/metrics` in production

The application does **not** authenticate this route: there is no `requireAuth`, token or IP allowlist
on it. Protection must come from the deployment:

1. **Do not expose it on the public ingress.** Route only the API paths you intend to publish, or
   explicitly block `/metrics` at the edge (load balancer, ingress or WAF rule returning `404`).
2. **Scrape over the private network** (cluster-internal Service, VPC, service mesh), with a network
   policy or security group limiting the source to the Prometheus server.
3. **TLS enforcement still applies.** `/metrics` is not in `PLAINTEXT_ALLOWED_PATHS`, so a scraper
   hitting the pod over plain `http` when TLS is enforced gets `403 tls_required`. Either scrape
   through the TLS-terminating proxy (so `X-Forwarded-Proto: https` is present and `TRUST_PROXY` covers
   that hop), or scrape over mTLS. Setting `ENFORCE_TLS=false` for this reason weakens every other
   route and is not recommended.
4. **The global limiter applies.** Keep the scrape interval modest and make sure the scraper's IP is
   not shared with unrelated traffic (see above).

## `GET /me`

Returns the identity carried by the caller's token.

**Auth:** `requireAuth`. The request needs `Authorization: Bearer <jwt>`. Beyond signature and expiry, the
middleware checks that the `(userId, deviceId)` pair exists in `devices` and is not revoked. See
[JWT and auth contract](contracts-jwt-auth.md). **Rate limit:** applies (`global_ip`, by IP).

**Response `200`:**

```json
{
  "user": {
    "userId": "9f1c6e2a-...",
    "walletAddress": "GABC...",
    "deviceId": "b6b6c3b0-...",
    "iat": 1767225600,
    "exp": 1767830400
  }
}
```

`user` is the verified JWT payload (`userId`, `walletAddress`, `deviceId`, plus the standard `iat`
and `exp` claims added by the JWT library). It is read from the token, not from the database, so
it does not reflect later profile edits.

**Errors** (all `401`):

| Body `error`                              | Cause                                         |
| ----------------------------------------- | --------------------------------------------- |
| `Missing or invalid Authorization header` | no `Bearer ` header                           |
| `Invalid or expired token`                | bad signature, or the 7-day token has expired |
| `Token missing deviceId`                  | legacy token without a `deviceId`             |
| `Device not found or has been revoked`    | device row missing or `revokedAt` set         |

## `GET /security/transport-policy`

Public TLS and certificate-pinning policy for mobile clients, so pin rotation does not need an app
release (#374). See [`docs/security/`](../../../docs/security) for how clients consume it.

**Auth:** none, deliberately: a client must read it before it holds a token, and every value is public.
**Rate limit:** applies (`global_ip`).
**Headers:** `Cache-Control: public, max-age=<pinning.maxAgeSeconds>`.

**Response `200`:**

```json
{
  "tls": {
    "required": true,
    "minimumVersion": "TLSv1.2",
    "hsts": "max-age=31536000; includeSubDomains; preload"
  },
  "pinning": {
    "enforced": true,
    "hosts": ["api.example.com"],
    "pins": ["sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="],
    "backupPins": ["sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="],
    "maxAgeSeconds": 5184000,
    "reportUri": null
  }
}
```

- `tls.required` is `isTlsEnforced()`; `tls.hsts` is the header value, or `null` when HSTS is disabled.
- `pinning.enforced` is `true` only when **both** `pins` and `backupPins` are non-empty, so a client never
  hard-fails against a set with no spare key.
- Pins must match `sha256/<44-char base64>`; malformed entries are logged and dropped.
- `maxAgeSeconds` defaults to 60 days (`TLS_PIN_MAX_AGE_SECONDS`); `reportUri` comes from
  `TLS_PIN_REPORT_URI`, else `null`.
- Values come from `TLS_PINNED_SPKI_SHA256`, `TLS_BACKUP_SPKI_SHA256`, `TLS_PINNED_HOSTS`,
  `TLS_PIN_MAX_AGE_SECONDS` and `TLS_PIN_REPORT_URI`.

## `/local-storage/*` (development and test only)

File server for the filesystem-backed object store, so presigned URLs issued locally are real working
URLs. See [Local Object Store](concepts-local-object-store.md).

**Auth:** none from `requireAuth`. Like an S3 presigned URL, each request carries its own
`expires` (Unix seconds) and `sig` (HMAC-SHA256 hex over `METHOD:key:expires`) query parameters,
method-bound and time-limited. **Rate limit:** applies (`global_ip`), as it is registered after the limiter.

| Method | Path                   | Success                                   | Failure                                                                                         |
| ------ | ---------------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `PUT`  | `/local-storage/<key>` | `200` empty body (raw body up to `150mb`) | `403 { "error": "Invalid or expired signed URL" }`, `500 { "error": "Failed to store object" }` |
| `GET`  | `/local-storage/<key>` | `200`, stored bytes and `Content-Type`    | `403` as above, `404 { "error": "Object not found" }`                                           |

The HMAC secret is random per process, so a URL signed by one process cannot be verified by another or after a
restart.

### When it is mounted

```ts
if (process.env['NODE_ENV'] !== 'production') {
  app.use('/local-storage', localStorageRouter);
}
```

It is mounted whenever `NODE_ENV` is anything other than the exact string `production`, including
`development`, `test` and **unset**. It is not mounted when `NODE_ENV === 'production'`.

### Can it be enabled in production?

Not by any flag, header or configuration option: there is no environment variable that forces the mount,
and `APP_ENV` is not consulted for it. With `NODE_ENV=production`, the route does not exist (requests
fall through to Express's default `404`), and `storage.ts` and `objectStore.ts` use the real S3 store
instead of the local one.

The guarantee is only as good as `NODE_ENV` itself. If a production deployment forgets to set
`NODE_ENV=production`, the route **is** mounted and uploads go to local disk (this is also called out in
[environment-variables.md](../../../docs/environment-variables.md)). Set `NODE_ENV=production` in every
production environment and verify with `curl -i https://<host>/local-storage/x` returning `404`, not `403`.
