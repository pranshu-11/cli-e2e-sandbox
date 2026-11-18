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
its credentials are retained so the account can be restored and so `getUser`
can still return a tombstone. Sessions for the account stop validating
immediately, because token checks reject deactivated accounts.

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |
| `REDIS_URL` | — | Redis connection string used for the response cache. |
| `CACHE_TTL` | `900` | Cache entry lifetime in seconds (15 minutes). |
| `RATE_LIMIT_PER_MINUTE` | `100` | Maximum requests per client per minute. |

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

- Password hashing uses bcrypt at the configured `BCRYPT_ROUNDS` cost. Plaintext
  passwords are never persisted, logged, or returned by any endpoint.
- JWT session tokens carry an `exp` claim bounded by `SESSION_TTL` (one hour by
  default). Expired tokens are rejected before any database access, and clients
  must re-authenticate to obtain a fresh token.
