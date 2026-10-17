# 13. Authenticate with short-lived JWTs and rotating refresh tokens

Date: 2026-10-03

## Status

Accepted

Supersedes [4. Authentication, phone verification, and authorization](0004-authentication-phone-verification-and-authorization.md)

Locally refined by [ADR-0026](0026-session-management.md) for browser session storage and refresh lifecycle. Its public-key validation assumption is partially superseded locally by [ADR-0022](0022-service-authorization-for-the-local-stack.md); its Argon2id assumption is partially superseded for team Identity by [ADR-0023](0023-bcrypt-for-team-identity-credentials.md).

## Context

Runners sign in with email and password (FR-1.1, FR-1.2); passwords are stored with Argon2id (NFR-5.2). After sign-in, every request to the gateway and to each service must prove who the caller is (NFR-5.3). Access tokens must expire within 15 minutes and refresh tokens must rotate on use (NFR-5.4). Services must scale horizontally (NFR-2.3), so checking a caller should not mean a call to Identity on every request. When an account is suspended or deleted, its access must end quickly.

This ADR covers only how a signed-in runner is recognized. Phone verification is [12. Verify phone ownership with SMS one-time passwords](0012-verify-phone-ownership-with-sms-one-time-passwords.md); what a caller may do is [14. Authorize every request in the service that owns it](0014-authorize-every-request-in-the-service-that-owns-it.md).

Options considered:

- **Server-side sessions.** A session ID in a cookie, looked up in a session store. Revocation is immediate and simple. Every service must reach the same store on every request, which couples all services to it and makes it a single point of failure.
- **Long-lived JWTs.** Stateless and simple: any service checks the signature. A stolen token works until it expires, and nothing can revoke it, which breaks NFR-5.4.
- **Short-lived JWT access tokens with rotating refresh tokens.** Any service checks the access token offline with Identity's public key; refresh tokens are stored by Identity, so it can rotate and revoke them. Revoking an access token early needs a deny-list, so a revoked token still works until it expires.

## Decision

We will issue JWT access tokens that expire after 15 minutes, signed by the Identity Service, and refresh tokens that Identity stores and replaces on every use. The gateway and every service validate access tokens with Identity's public signing keys, without calling Identity.

## Consequences

- Any service can recognize a caller on its own and scale independently.
- Suspending or deleting an account revokes its refresh tokens at once, but its last access token keeps working for up to 15 minutes; a deny-list is needed only if that window becomes unacceptable.
- Identity must publish and rotate its signing keys, and every service must refresh them.
- A refresh token that is reused after rotation signals theft, and Identity should revoke the whole chain.
- Follow-on decision: where the web client keeps tokens, for example the refresh token in an HttpOnly cookie and the access token in memory.

Source: Deliverable 1 (revised), "ADR-003: Authentication, Phone Verification, and Zero Trust Authorization," split per review feedback so authentication and authorization are separate decisions.
