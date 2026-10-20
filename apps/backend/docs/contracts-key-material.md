# Public key material: encoding and validation

Source: [`src/lib/keys.ts`](../src/lib/keys.ts) (the single validator),
[`src/schemas/auth.schemas.ts`](../src/schemas/auth.schemas.ts),
[`src/routes/devices.ts`](../src/routes/devices.ts),
[`apps/web/src/lib/x3dh.ts`](../../web/src/lib/x3dh.ts) (reference client).

This document gives the exact encodings and byte lengths the server accepts. The shape of each request is in the
[REST schema reference](contracts-rest-schemas.md); only key formats are covered here. The server stores and
verifies **public** material only: every schema below is `.strict()`, so a stray `privateKey` or `sessionState`
field fails validation instead of being dropped (see [crypto invariants](concepts-crypto-invariants.md)).

## Encoding rules (all key types)

- Standard base64 (alphabet `A-Za-z0-9+/`), **padded**. `isValidBase64` requires `^[A-Za-z0-9+/]*={0,2}$` and a
  length that is a multiple of 4. Base64url (`-`, `_`), unpadded and whitespace-containing values are rejected.
- Length is checked on the **decoded** bytes, computed from the string length and padding.
- A bad value returns `400 { "error": "Validation failed", "issues": [{ "field": "...", "message": "..." }] }`.

## Accepted key types

| Key type        | JSON field                                            | Encoding | Form                            | Decoded length                   | Accepted at                                                |
| --------------- | ----------------------------------------------------- | -------- | ------------------------------- | -------------------------------- | ---------------------------------------------------------- |
| Identity key    | `identityPublicKey` (also `device.identityPublicKey`) | base64   | Ed25519, **SPKI DER**           | **44** (`ED25519_SPKI_BYTES`)    | `POST /auth/verify`, device registration                   |
| Signed prekey   | `signedPreKey.publicKey`                              | base64   | X25519, **raw**                 | **32** (`ED25519_RAW_KEY_BYTES`) | `POST /devices/:id/prekeys`                                |
| One-time prekey | `oneTimePreKeys[].publicKey`                          | base64   | X25519, **raw**                 | **32**                           | `POST /devices/:id/prekeys`                                |
| Signature       | `signedPreKey.signature`                              | base64   | Ed25519 signature, raw          | **64** (`ED25519_SIG_BYTES`)     | `POST /devices/:id/prekeys`                                |
| MLS key package | `keyPackages[].keyPackage`                            | base64   | TLS-serialised MLS `KeyPackage` | **32 to 4096** (inclusive)       | `POST /devices/:id/mls-key-packages` (max 100 per request) |

Notes:

- The constant name `ED25519_RAW_KEY_BYTES` is reused for prekeys, but the prekeys are **X25519** public keys. Both
  are 32 bytes; only the identity key is an Ed25519 key.
- `keyId` is a non-negative integer (`z.number().int().nonnegative()`).
- For MLS the server cannot parse a `KeyPackage` without an MLS implementation, so the size window is the only
  structural check. The endpoint adds device-ownership checks and rejects an `expiresAt` that is not in the future
  with `400 { "error": "expiresAt must be in the future" }`. See [MLS key packages](mls-key-packages.md).
- `POST /auth/verify` requires `device.identityPublicKey` to equal the top-level `identityPublicKey`.

## Raw versus SPKI

Node's `crypto.verify` needs the identity key as SPKI DER, so that is what the server stores and what clients
send. The web client works with raw 32-byte Ed25519 keys internally (`@noble/curves`), and converts at the wire
boundary in `apps/web/src/lib/x3dh.ts`:

| Function                          | Does                                                                                           |
| --------------------------------- | ---------------------------------------------------------------------------------------------- |
| `rawEd25519PublicKeyToSpki(raw)`  | prepends the 12-byte RFC 8410 header `30 2a 30 05 06 03 2b 65 70 03 21 00` to the 32 raw bytes |
| `spkiToRawEd25519PublicKey(spki)` | throws unless the input is exactly 44 bytes (`12 + 32`), then returns the last 32 bytes        |

`toBase64` / `fromBase64` in the same file wrap `btoa` / `atob` (standard, padded). The client uses
`spkiToRawEd25519PublicKey` on a bundle's `identityPublicKey` before verifying the signed prekey or deriving X25519
keys; the prekeys in a bundle are already raw and need no conversion.

## What the signed-prekey signature covers

On `POST /devices/:id/prekeys` the server calls
`verifyEd25519Signature(device.identityPublicKey, signedPreKey.publicKey, signedPreKey.signature)`, which:

1. Base64-decodes the stored identity key (44-byte SPKI DER).
2. Base64-decodes `signedPreKey.publicKey` (32 raw bytes). **These bytes are the signed message.**
3. Base64-decodes the signature (64 bytes).
4. Runs `crypto.verify(null, message, { key, format: 'der', type: 'spki' }, signature)`, a pure Ed25519 verification
   (no pre-hash).

So the signature is over the **32 raw bytes of the signed prekey's public key only**. It does not cover `keyId`, a
prefix, or the base64 text. The client produces it with `ed25519.sign(publicKey, identity.privateKey)`
(`generateSignedPreKey`). A verification failure, or any exception, returns
`400 { "error": "Signed prekey signature is invalid" }`. The verifier is whichever identity key is registered
for the device in the URL, not one supplied in the body. Initiators repeat the check on the fetched bundle before
starting X3DH (`initiateSession`).

## Examples

The valid values below come from one real keypair: the signature is a real Ed25519 signature by the identity key
over the signed prekey bytes, so the combined payload passes the signature check too.

### Identity key (44-byte SPKI)

Valid:

```json
{ "identityPublicKey": "MCowBQYDK2VwAyEAH6JwScWbg0BzKfoxkggh2dgC9gpQkcAsvbDNGLEQ+jo=" }
```

Rejected, because a raw 32-byte key was sent where SPKI is required:

```json
{ "identityPublicKey": "H6JwScWbg0BzKfoxkggh2dgC9gpQkcAsvbDNGLEQ+jo=" }
```

```text
identityPublicKey must be 44 bytes (got 32)
```

### Signed prekey (32-byte raw key + 64-byte signature)

Valid:

```json
{
  "signedPreKey": {
    "keyId": 1,
    "publicKey": "x645YvSqiDkxrn0ZnWSXZbugQa1pVtGEa1/PpTrgUVU=",
    "signature": "0k3GQRAthm1VjJEUuMyvGep1tG1Mi3QphlopRmyDK48K+Aa8I0kXICubLUUtBkZ0TYhNrjdnwfl5Lgv0v5OuAg=="
  }
}
```

Rejected, because the signature is 32 bytes instead of 64 (a truncated value):

```json
{
  "signedPreKey": {
    "keyId": 1,
    "publicKey": "x645YvSqiDkxrn0ZnWSXZbugQa1pVtGEa1/PpTrgUVU=",
    "signature": "x645YvSqiDkxrn0ZnWSXZbugQa1pVtGEa1/PpTrgUVU="
  }
}
```

```text
signature must be 64 bytes (got 32)
```

Rejected at the signature step (`400 Signed prekey signature is invalid`): a well-formed 64-byte signature made by
a different identity key, or over different bytes (for example the base64 text, or `keyId || publicKey`).

### One-time prekey (32-byte raw key)

Valid:

```json
{
  "oneTimePreKeys": [{ "keyId": 100, "publicKey": "x645YvSqiDkxrn0ZnWSXZbugQa1pVtGEa1/PpTrgUVU=" }]
}
```

Rejected, because the SPKI-wrapped form was sent (44 bytes):

```json
{
  "oneTimePreKeys": [
    { "keyId": 100, "publicKey": "MCowBQYDK2VwAyEAH6JwScWbg0BzKfoxkggh2dgC9gpQkcAsvbDNGLEQ+jo=" }
  ]
}
```

```text
publicKey must be 32 bytes (got 44)
```

Also rejected: any extra field, for example `{ "keyId": 100, "publicKey": "...", "privateKey": "..." }` fails the
`.strict()` object check with `Validation failed`.

### Signature (64 bytes)

Valid: `0k3GQRAthm1VjJEUuMyvGep1tG1Mi3QphlopRmyDK48K+Aa8I0kXICubLUUtBkZ0TYhNrjdnwfl5Lgv0v5OuAg==`

Rejected, because it is not valid base64 (base64url alphabet and no padding):

```text
0k3GQRAthm1VjJEUuMyvGep1tG1Mi3QphlopRmyDK48K-Aa8I0kXICubLUUtBkZ0TYhNrjdnwfl5Lgv0v5OuAg
```

```text
signature must be valid base64
```

### MLS key package (32 to 4096 bytes)

Valid, for a 48-byte package (placeholder bytes; the server only checks size):

```json
{
  "cipherSuite": 1,
  "keyPackages": [
    { "keyPackage": "BwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcHBwcH" }
  ]
}
```

Rejected, because the decoded package is 16 bytes (below the 32-byte minimum):

```json
{ "cipherSuite": 1, "keyPackages": [{ "keyPackage": "BwcHBwcHBwcHBwcHBwcHBw==" }] }
```

```text
keyPackage must be 32–4096 bytes (got 16)
```

An empty string fails earlier with `keyPackage is required`, and anything above 4096 decoded bytes fails with the
same range message.
