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

Soft-deletes the account by setting its `isActive` flag to `false`; the row and
its history are retained. The call is idempotent, so deleting an already
inactive user succeeds without changing state. Sessions for the account are
invalidated immediately, and subsequent `getUser` calls return the tombstone
profile with `isActive: false`.

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |
| `REDIS_URL` | — | Redis connection string used for the response cache and rate-limit counters. |
| `CACHE_TTL` | `900` | Cache entry lifetime in seconds (15 minutes). |
| `RATE_LIMIT_PER_MINUTE` | `100` | Maximum requests per client per minute (100 req/min). |

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

- Password hashing: bcrypt with a per-password salt, using `BCRYPT_ROUNDS` as
  the cost factor. Plaintext passwords are never persisted or written to logs.
- JWT expiry: tokens carry a short-lived `exp` claim derived from `SESSION_TTL`
  and are rejected after it passes; clients must re-authenticate to obtain a new
  token rather than extending an existing one.
- Deactivation: soft-deleting a user via `deleteUser` invalidates outstanding
  tokens even if their `exp` claim has not yet elapsed.
