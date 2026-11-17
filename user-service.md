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

## Configuration

| Setting | Default | Description |
| --- | --- | --- |
| `USER_SERVICE_PORT` | `8080` | Port the HTTP server binds to. |
| `DATABASE_URL` | — | Postgres connection string. |
| `SESSION_TTL` | `3600` | Session token lifetime in seconds. |
| `BCRYPT_ROUNDS` | `12` | Cost factor used when hashing passwords. |

## Security

Passwords are hashed with bcrypt and never logged. Session tokens are JWTs
signed with the service key; the `exp` claim is enforced on every request and
tokens are rejected once the account is deactivated.

### deleteUser(userId)

Soft-deletes by setting `isActive: false`.
