# User Service

The user service owns accounts, credentials, and session lifecycle for the
platform API.

## Overview

`user-service` exposes a small HTTP surface in front of the `users` table and
the credential store. It is stateless: every request carries a bearer token and
the service validates it before touching the database.

## API

### `authenticateUser(email, password)`

Looks up the account by email, verifies the supplied password against the
stored hash, and returns a signed session token on success. Failed attempts are
rate limited per account and per source address.

### `getUser(userId)`

Returns the public profile for a user: id, email, display name, and the
`isActive` flag. Soft-deleted accounts are returned with `isActive: false` so
callers can render a tombstone instead of a 404.

### `updateUser(userId, patch)`

Applies a partial update to the mutable profile fields. Email changes require a
verification round-trip before they take effect.

### `deleteUser(userId)`

Soft-deletes an account by setting its `isActive` flag to `false`. The row and
its credentials are retained so the account can be restored and so audit
references stay valid; no hard delete is performed. The call is idempotent —
deleting an already inactive user succeeds without changing state. Any cached
profile for the user is invalidated as part of the operation.

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |
| `REDIS_URL` | — | Redis connection string used for the profile cache. |
| `CACHE_TTL` | `900` | Cache entry lifetime in seconds (15 minutes). |
| `RATE_LIMIT_PER_MINUTE` | `100` | Maximum requests per minute, per client. |

User profiles are cached in Redis with a 15-minute TTL (`CACHE_TTL=900`);
`deleteUser` and `updateUser` invalidate the affected entry. Requests are rate
limited to 100 per minute per client, and exceeding the limit returns `429`
with a `Retry-After` header.

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

- **Password hashing.** Passwords are stored only as bcrypt hashes using the
  configured `BCRYPT_ROUNDS` cost factor. Plaintext passwords are never
  persisted, logged, or returned by any endpoint, and verification uses a
  constant-time comparison.
- **JWT expiry.** Session tokens carry an `exp` claim set from `SESSION_TTL`
  (default one hour). Expired tokens are rejected before any database access,
  and tokens for deactivated accounts are refused even if they have not yet
  expired.
