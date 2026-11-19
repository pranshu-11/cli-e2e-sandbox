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
its history are retained; no credentials or profile data are removed. Subsequent
`getUser` calls return the tombstone (`isActive: false`), and any outstanding
session tokens for the account are rejected.

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

Reads are served through a Redis cache with a 15-minute TTL; writes invalidate
the affected entries. Requests are rate limited at 100 requests per minute per
client, and exceeding the limit returns `429 Too Many Requests`.

## Security

Passwords are hashed with bcrypt and never logged: plaintext passwords are
never persisted, and the cost factor is configurable via `BCRYPT_ROUNDS`.
Session tokens are JWTs signed with the service key; the `exp` claim is enforced
on every request and tokens are rejected once the account is deactivated. The
token expiry is set from `SESSION_TTL`, so expired tokens are refused and
clients must re-authenticate to obtain a new one.
