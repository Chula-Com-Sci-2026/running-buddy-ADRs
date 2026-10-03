# 14. Authorize every request in the service that owns it

Date: 2026-10-03

## Status

Accepted

Supersedes [4. Authentication, phone verification, and authorization](0004-authentication-phone-verification-and-authorization.md)

## Context

Every endpoint except registration, one-time-password verification, login, and password reset must authorize its caller, and no endpoint may trust a call just because it came from inside the cluster (NFR-5.3). Most rules are about ownership of data that only one service holds: only a run's organizer may edit or cancel it, an invite-only run is visible only to its invitees, only the candidate may answer a buddy request, and only verified runners may create runs. Some calls have no user behind them, such as the scheduled forecast re-check and service-to-service pushes. The course guidance is to authenticate at the gateway and check business-rule permissions in the service that owns the operation, rather than wiring an identity service to every other service.

Who the caller is comes from [13. Authenticate with short-lived JWTs and rotating refresh tokens](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md); this ADR decides where permissions are checked.

Options considered:

- **Authorize at the gateway only.** One place to write rules; services trust whatever reaches them. The gateway does not hold the data that ownership rules need, and any call that bypasses it, from another service or a misconfigured route, is unchecked, which breaks NFR-5.3.
- **Authorize in each owning service (zero trust).** Each service re-validates the token and checks role and ownership against its own data. Rules sit next to the data they need. Every service carries authorization code, and a shared middleware library must be kept up to date in all of them.
- **A central policy service (for example Open Policy Agent).** Rules are written once in a policy language. Ownership rules still need each service's data, so services must send it with every decision request; it adds a network hop and another component to run.

## Decision

We will authorize every request in the service that owns the operation. The gateway rejects requests without a valid token and applies rate limits (NFR-5.5), but grants nothing. Each service validates the token again and checks the caller's role (runner, and admin from phase 2) and ownership of the data involved. Calls without a user carry a short-lived token issued to the calling service.

## Consequences

- Permission rules live next to the data they protect, and a call that bypasses the gateway is still checked.
- Each service carries authorization code; a shared middleware library handles token validation, so only the business rules differ per service. Upgrading that library touches every service.
- Service-to-service calls need service identities and tokens, including the phase 1 REST pushes ([19. Call services over REST until the phase 2 event broker](0019-call-services-over-rest-until-the-phase-2-event-broker.md)).
- Rules that depend on another service's data, such as "verified runners only", use a local copy of that data, which can lag behind its source.
- Automated tests should check that each endpoint rejects callers without permission (NFR-5.6).

Source: Deliverable 1 (revised), "ADR-003: Authentication, Phone Verification, and Zero Trust Authorization," split per review feedback so authentication and authorization are separate decisions.
