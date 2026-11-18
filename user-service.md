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

Soft-deletes the account by setting its `isActive` flag to `false`. The row and
its credentials are retained so the account can be restored and so audit
references stay valid; no data is physically removed. The call is idempotent —
deleting an already inactive user succeeds without changing the record. Any
outstanding session tokens for the account stop being accepted immediately.

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |
| `REDIS_URL` | — | Redis connection string used for the response cache. |
| `CACHE_TTL` | `900` | Cache entry lifetime in seconds (15 minutes). |
| `RATE_LIMIT` | `100` | Maximum requests per minute, per client. |

Cached responses are keyed per user and invalidated on write, so `updateUser`
and `deleteUser` never serve a stale profile. Requests above `RATE_LIMIT` are
rejected with `429 Too Many Requests` and a `Retry-After` header.

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

- **Password hashing.** Passwords are stored only as bcrypt hashes, using the
  cost factor from `BCRYPT_ROUNDS`. Plaintext passwords are never persisted,
  logged, or returned by any endpoint, and verification uses a constant-time
  comparison.
- **JWT token expiry.** Session tokens carry an `exp` claim derived from
  `SESSION_TTL` and are rejected once expired; there is no unbounded or
  non-expiring token. Deactivating an account via `deleteUser` invalidates its
  tokens before their natural expiry.
