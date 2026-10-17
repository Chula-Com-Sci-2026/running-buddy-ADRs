# 22. Service authorization in the local stack

Date: 2026-10-16

## Status

Accepted for local Deliverable 4 implementation, under delegated owner approval. Refines [ADR-0014](0014-authorize-every-request-in-the-service-that-owns-it.md) for implemented local permissions. Partially supersedes the public-key validation assumption in [ADR-0013](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md) for local token trust only. Credentials, OTP and browser session storage are outside this decision.

## Context

A runner must not read another runner's requests or records, answer someone else's request, or rate a run without confirmed attendance. Gateway-only authorization is simpler but leaves internal endpoints dependent on network isolation. Independent service validation costs a little repeated middleware but keeps each service responsible for its business permissions. This decision concerns authorization and token trust, not selection of a password or OTP mechanism.

## Decision

Every internal HTTP/gRPC operation validates the token signature, fixed algorithm, issuer, audience, expiry and UUID subject. Each service derives the acting runner from the verified subject and enforces resource ownership and lifecycle permissions. The gateway forwards the original token. Discovery checks request participants; Scheduling checks membership/capacity and owns run completion; Reputation checks confirmed participation through Scheduling before ratings or record creation; Notification filters by recipient.

For the local-only deployment use HS256 with a randomly generated uncommitted shared secret. Keep service ports internal and bind the gateway to loopback. This is authenticated local service collaboration, not a claim of full production zero-trust infrastructure. A production/team replacement should use asymmetric verification keys to avoid giving every verifier token-signing capability.

## Consequences

Caller-supplied runner IDs cannot override authenticated identity. Services remain usable behind another conforming gateway. Shared local signing trust is a limitation: compromise of any verifier exposes signing authority. Switching to another Identity implementation requires compatible issuer/audience/keys and tests, not just changing its HTTP address. Revoked refresh tokens cannot refresh again, while existing access tokens expire after 15 minutes.
