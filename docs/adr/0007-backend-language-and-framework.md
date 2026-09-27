# 7. Backend stack for discovery, reputation, and trust services

Date: 2026-09-27

## Status

Proposed

## Context

The Buddy Discovery, Reputation, and Trust & Safety services need implementations. The full stack must start locally with one command (NFR-8.3), domain logic needs at least 70% unit-test coverage (NFR-8.1), and the project must demonstrate REST, gRPC, and message-broker communication. Because the system uses independently deployable microservices, this decision should not constrain the API Gateway or other services.

The main candidates for these three services are Ruby on Rails in API-only mode and Rust with Axum. Rails provides conventions for REST routing, JSON handling, PostgreSQL persistence, migrations, validation, testing, logging, and request middleware. Ruby has supported gRPC and Kafka libraries. Axum with Tonic offers strong compile-time guarantees, predictable performance, and first-class asynchronous gRPC, but requires more infrastructure decisions and implementation code.

## Decision

Use Ruby on Rails in API-only mode for the Buddy Discovery, Reputation, and Trust & Safety services only. Keep each service independently deployable, with its own Rails application and database; share contracts and small infrastructure libraries, not domain models or database access.

Expose Buddy Discovery through a Protocol Buffers/gRPC contract so any API Gateway implementation can call it regardless of language. Expose REST/JSON operations where the agreed collaboration contract calls for REST, and use Kafka for the domain events recorded in ADR-0006.

The API Gateway and all other services are explicitly outside this decision and may use other languages and frameworks. Interoperability depends on versioned OpenAPI, Protocol Buffers, and event schemas rather than shared implementation technology. Pin Ruby, Rails, PostgreSQL, gRPC, and Kafka-client versions when scaffolding rather than relying on unbounded latest versions.

## Consequences

- Rails reduces delivery risk across the three assigned services and aligns well with the existing PostgreSQL decision.
- A consistent language simplifies their local setup, reviews, testing, observability, and shared authentication middleware.
- Ruby gRPC calls are blocking, so callers and Buddy Discovery must set deadlines and avoid unbounded request fan-out. The expected course workload does not justify Rust primarily for throughput.
- Rails processes generally use more memory and provide fewer compile-time guarantees than Rust; contract tests, static analysis, validation, and disciplined boundaries are therefore important.
- Rust remains a valid future choice for a measured performance bottleneck, but mixing stacks now would increase operational and learning cost.
- Polyglot services add build and operational variation, but language-neutral contracts keep that variation behind service boundaries.

Sources: Rails API-only Applications guide; official gRPC Ruby documentation; ADR-0002, ADR-0005, ADR-0006; NFR-8.1 and NFR-8.3.
