# 8. Use a microservices architecture

Date: 2026-10-03

## Status

Accepted

## Context

Load on the system is uneven. Runner search and nearby-run queries peak at 05:00-07:00 and 17:00-20:00 at about 5× average traffic (NFR-2.2), while account and moderation traffic stays low, and any part of the system must scale horizontally without code changes (NFR-2.3). Two capabilities depend on external providers that can be slow or down (SMS/OTP and weather), and a failure in weather or notification must not take login, profile, search, or scheduling down with it (NFR-3.3). The course term project asks for a microservice design, and by the final deliverable for REST, gRPC, and asynchronous messaging between services.

The team considered three styles:

- **Single-process monolith.** Fastest to start and simplest to run. No fault isolation: a hung weather call can use up the threads that serve login and search, which breaks NFR-3.3.
- **Modular monolith.** One deployable with enforced module boundaries. Usually the better choice for a small team, because it avoids the cost of a distributed system. All modules still share one process and one release, so it cannot keep a weather or notification failure away from the core flows, or scale discovery separately for the peaks.
- **Microservices.** Process-level isolation and per-service scaling, at the cost of network calls, eventual consistency between services, and more infrastructure for a five-person team to run in one semester.

## Decision

We will build the backend as microservices: one independently deployed service per bounded context, each owning its own data, reached by the web client through one API gateway.

The service boundaries are in [5. Service boundaries and data ownership](0005-service-boundaries-and-data-ownership.md), and how services talk to each other is in [6. Service collaboration and provider isolation](0006-service-collaboration-and-provider-isolation.md). Phase 1 builds only part of the target design ([18. Build phase 1 with six services](0018-build-phase-1-with-six-services.md)).

## Consequences

- Weather and notification failures stay inside their own services, and Buddy Discovery and Run Scheduling can scale for the peaks without scaling the rest.
- Team members can each own one or two services.
- There are no ACID transactions across services, so each business rule must live inside one service. Effects that cross services, such as a profile change reaching search results, are eventually consistent.
- The team pays for several deployables, a gateway, and later a broker. One shared service template and one Docker Compose file keep that cost down, and phase 1 postpones the broker ([19. Call services over REST until the phase 2 event broker](0019-call-services-over-rest-until-the-phase-2-event-broker.md)).
- Following one request across services needs correlation IDs and central logs.
- Boundaries drawn this early may be wrong. If two services keep changing together, merge them rather than keep coordinating their releases.
- Follow-on decisions: the service list, the collaboration style, one database per service ([15. Give each service its own PostgreSQL database](0015-give-each-service-its-own-postgresql-database.md)), and the production platform.

Source: Deliverable 1 (revised), "ADR-001: Architectural Style: Microservices."
