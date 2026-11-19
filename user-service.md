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
references stay valid; no hard delete is performed. Sessions belonging to the
account stop validating immediately, and `getUser` continues to return the
profile with `isActive: false`.

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |
| `REDIS_URL` | — | Redis connection string used for the shared cache. |
| `CACHE_TTL` | `900` | Cache entry lifetime in seconds (15 minutes). |
| `RATE_LIMIT_PER_MINUTE` | `100` | Maximum requests per client per minute. |

Profile reads are cached in Redis with a 15-minute TTL and invalidated on
`updateUser` and `deleteUser`. Requests are rate limited to 100 per minute per
client, counted in the same Redis instance so the limit holds across replicas.

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

- **Password hashing.** Passwords are stored only as bcrypt hashes at the
  configured cost factor (`BCRYPT_ROUNDS`, default `12`). Plaintext passwords
  never reach the database, the cache, or the logs, and verification uses a
  constant-time comparison.
- **Token expiry.** JWTs carry a short-lived `exp` claim (`SESSION_TTL`, default
  `3600` seconds). Expired tokens are rejected before any database access, and
  clients must re-authenticate to obtain a new one. Tokens are not refreshed
  implicitly, and deactivating an account invalidates its outstanding tokens.
