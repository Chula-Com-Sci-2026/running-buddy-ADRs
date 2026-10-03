# 7. Backend stack for discovery, reputation, and trust services

Date: 2026-09-27

## Status

Proposed

## Context

The Buddy Discovery, Reputation, and Trust & Safety services need implementations. Buddy Discovery is built in phase 1; Reputation and Trust & Safety follow in phase 2 ([18. Build phase 1 with six services](0018-build-phase-1-with-six-services.md)). The team wants the full stack to start locally with one command and domain logic covered by unit tests. Because the system uses independently deployable microservices, this decision should not constrain the API Gateway or other services.

What the architecture asks of these three services:

- **Buddy Discovery (phase 1):** a REST API behind the API Gateway, PostGIS radius search ([16. Search nearby runners and runs with PostGIS on coarsened cells](0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md)), a scheduled job to expire buddy requests, and in phase 1 a REST events endpoint for `ProfileUpdated` and `PhoneVerified` plus REST calls to Notification ([19. Call services over REST until the phase 2 event broker](0019-call-services-over-rest-until-the-phase-2-event-broker.md)). From phase 2 it consumes and publishes those events through Kafka instead.
- **Reputation (phase 2):** a REST API behind the gateway, a scheduled job to close attendance windows, and Kafka events in both directions.
- **Trust & Safety (phase 2):** a REST API behind the gateway, Kafka events in both directions, and one gRPC client call to Identity's `lockAccount()`.

None of the three serves gRPC. The project's gRPC requirement is met by Run Scheduling calling Run Conditions, and later by Trust & Safety calling Identity.

The main candidates are Ruby on Rails in API-only mode and Rust with Axum. Rails provides conventions for REST routing, JSON handling, PostgreSQL persistence, migrations, validation, testing, logging, and request middleware, and Ruby has supported gRPC and Kafka client libraries. Axum with Tonic offers strong compile-time guarantees, predictable performance, and first-class asynchronous gRPC, but requires more infrastructure decisions and implementation code.

## Decision

Use Ruby on Rails in API-only mode for Buddy Discovery in phase 1, and for Reputation and Trust & Safety when phase 2 builds them. Keep each service independently deployable, with its own Rails application and database; share contracts and small infrastructure libraries, not domain models or database access.

Each service exposes REST/JSON operations to the API Gateway, described in OpenAPI. Events follow ADR-0019 in phase 1 and Kafka in phase 2 ([6. Service collaboration and provider isolation](0006-service-collaboration-and-provider-isolation.md)); handlers are written once and called by either transport. Trust & Safety calls Identity's `lockAccount()` through a gRPC client generated from Identity's Protocol Buffers contract.

The API Gateway and all other services are explicitly outside this decision and may use other languages and frameworks. Interoperability depends on versioned OpenAPI, Protocol Buffers, and event schemas rather than shared implementation technology. Pin Ruby, Rails, PostgreSQL, gRPC-client, and Kafka-client versions when scaffolding rather than relying on unbounded latest versions.

## Consequences

- Rails reduces delivery risk for Buddy Discovery now and gives Reputation and Trust & Safety a ready template in phase 2; it aligns well with the PostgreSQL decision.
- A consistent language simplifies their local setup, reviews, testing, observability, and shared authentication middleware.
- Phase 1 needs no gRPC or Kafka libraries in Rails. Phase 2 adds a Kafka client to all three and a gRPC client to Trust & Safety.
- Ruby gRPC calls are blocking, so Trust & Safety must set a deadline on `lockAccount()` and keep retrying in the background rather than hold a request open. The expected course workload does not justify Rust primarily for throughput.
- Rails processes generally use more memory and provide fewer compile-time guarantees than Rust; contract tests, static analysis, validation, and disciplined boundaries are therefore important.
- Rust remains a valid future choice for a measured performance bottleneck, but mixing stacks now would increase operational and learning cost.
- Polyglot services add build and operational variation, but language-neutral contracts keep that variation behind service boundaries.

Sources: Rails API-only Applications guide; official gRPC Ruby documentation; ADR-0005, ADR-0006, ADR-0015, ADR-0018, ADR-0019.
