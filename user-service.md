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
its history are retained, so `getUser` keeps returning the profile with
`isActive: false` and any outstanding session tokens are rejected on their next
use. The call is idempotent: deleting an already inactive account succeeds
without further changes.

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |
| `REDIS_URL` | — | Redis connection string used for the response cache. |
| `CACHE_TTL` | `900` | Cache entry lifetime in seconds (15 minutes). |
| `RATE_LIMIT` | `100` | Maximum requests per minute, per account and source address. |

Reads are served through a Redis cache with a 15-minute TTL, so profile changes
made by `updateUser` and `deleteUser` invalidate the affected entry immediately
rather than waiting for expiry. Requests are rate limited at 100 per minute per
account and per source address; excess requests receive `429 Too Many Requests`
with a `Retry-After` header.

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

Token expiry is bounded by `SESSION_TTL` (one hour by default) and is set as the
`exp` claim at issue time, so a token cannot outlive its configured lifetime.
Because `deleteUser` only flips `isActive`, revocation relies on that flag check
rather than on deleting the credential row.
