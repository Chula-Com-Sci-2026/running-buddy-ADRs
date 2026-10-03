# 4. Authentication, phone verification, and authorization

Date: 2026-09-20

## Status

Superseded by [12. Verify phone ownership with SMS one-time passwords](0012-verify-phone-ownership-with-sms-one-time-passwords.md)

Superseded by [13. Authenticate with short-lived JWTs and rotating refresh tokens](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md)

Superseded by [14. Authorize every request in the service that owns it](0014-authorize-every-request-in-the-service-that-owns-it.md)

## Context

Every request needs authentication and authorization (NFR-4.3), and discovery must not expose a runner's exact location (NFR-5.1). Phone verification provides the trust signal required by FR-3.1. Tokens must be short lived and refresh tokens rotated (NFR-4.4); discovery needs rate limiting (NFR-4.5). The team considered SMS, email, and app-based OTP, along with server sessions and longer-lived JWTs.

## Decision

Authenticate with email and password hashed with Argon2id. Verify phone ownership through SMS OTP via a provider adapter, with email OTP as a fallback when SMS delivery fails. Issue 15-minute JWT access tokens and rotating refresh tokens. Each service validates and authorizes every request, including service-to-service calls. Discovery coarsens location to roughly 1–2 km at read time; exact coordinates are disclosed only to runners who have joined the same run.

## Consequences

- OTP needs expiry, attempt limits, and per-number rate limiting; SMS also has a per-message cost.
- Provider failure affects registration but should not stop existing users' core flows. The adapter allows provider replacement (NFR-7.2).
- Independent token validation supports service scaling. Early revocation needs a deny-list; otherwise an access token remains usable until its 15-minute expiry.
- Per-service authorization adds work and latency. Shared middleware can keep enforcement consistent.
- Location disclosure must be checked at the service boundary, including run details and discovery results.

Source: Deliverable 1, “ADR-003: Authentication, Phone Verification, and Zero Trust Authorization.”
