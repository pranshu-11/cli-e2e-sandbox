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
its history are retained; no credentials or profile data are erased. Subsequent
`getUser` calls return the tombstone profile with `isActive: false`, and any
outstanding session tokens for the account are rejected. The operation is
idempotent: deleting an already-inactive user succeeds without changes.

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

Reads are served through a Redis cache with a 15-minute TTL (`CACHE_TTL=900`);
writes invalidate the affected user's entries. Requests are rate limited to
100 per minute per client, and exceeding the limit returns `429`.

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

- **Password hashing** — bcrypt with a configurable cost factor
  (`BCRYPT_ROUNDS`, default `12`). Plaintext passwords are never persisted or
  written to logs, and hashes are compared in constant time.
- **JWT expiry** — tokens carry an `exp` claim bounded by `SESSION_TTL`
  (default `3600` seconds). Expired tokens are rejected on every request, and
  deactivating an account invalidates its outstanding tokens immediately.
