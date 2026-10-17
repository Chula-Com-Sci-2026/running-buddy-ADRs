# 26. Short-lived access tokens with rotating cookie-based refresh sessions

Date: 2026-10-17

## Status

Accepted for the local Deliverable 4 implementation with project-owner approval. This does not claim teammate review.

Refines [ADR-0013](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md) by resolving its browser-token-storage follow-on decision and documenting the implemented refresh lifecycle. Short-lived JWTs and refresh rotation are retained; historical ADR-0004 remains superseded. Password hashing is covered by [ADR-0023](0023-bcrypt-for-team-identity-credentials.md), and service authorization by [ADR-0022](0022-service-authorization-for-the-local-stack.md). OTP delivery and location disclosure are outside this decision. Local signing-key trust is addressed only in ADR-0022; no new key-distribution choice is made here.

## Context

Services need short-lived authenticated requests, while runners should remain signed in across browser reloads (NFR-5.4). A token held only in JavaScript memory is lost on reload. Refresh credentials must support expiry and revocation without exposing them to ordinary frontend JavaScript.

Long-lived access tokens simplify reloads but extend the lifetime of a stolen bearer token. An opaque server session could centralize revocation, but would require a different session-validation contract for the current HTTP and gRPC services. Persisting refresh tokens in browser-readable storage exposes them directly to script access.

## Decision

Use short-lived JWT access tokens backed by rotating refresh sessions in an HttpOnly cookie.

For the active team Identity implementation, access tokens default to 15 minutes and remain in frontend memory. Refresh tokens expire after seven days; Identity stores their hashes and atomically consumes an old refresh session before issuing its successor. The refresh cookie uses `SameSite=Lax`, the authentication-path scope and `Secure` in production. Local HTTP is a development exception.

The frontend restores an access token through refresh after reload, shares concurrent refresh work and retries a protected request once. A rejected refresh returns the runner to sign-in. Local browser requests use the frontend's same-origin `/api` proxy so `localhost` and `127.0.0.1` are not mixed within a session. Logout revokes the supplied refresh session and clears its cookie.

## Consequences

- Reload recovery does not require persisting tokens in localStorage or sessionStorage.
- Rotation and hashed session records add database writes and concurrency handling. The current implementation rejects a consumed refresh token; revocation of an entire token family on replay remains unimplemented.
- Revoking a refresh session does not immediately invalidate an already issued access token; it can remain usable until expiry.
- HttpOnly limits direct cookie access, but does not eliminate script-injection or cross-site request risks. Production deployment still needs HTTPS and appropriate request-origin protections.
- Separate accounts need isolated browser cookie contexts for the demo. The development proxy is not a production hosting configuration.

Implementation evidence: `team-identity-service/services/session.js`, `controllers/session.js`, and `cant-run-alone-frontend/src/api/gateway.ts`, `src/App.tsx`, `vite.config.ts` in the sibling repositories.
