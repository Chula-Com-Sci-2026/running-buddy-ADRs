# Architecture goals, constraints and key decisions

The architectural drivers behind the as-built system. Requirement IDs refer to [`../requirements.md`](../requirements.md); ADR numbers refer to [`../adr/`](../adr/).

## Goals

| Goal | Drivers | How the architecture addresses it |
| --- | --- | --- |
| Find a compatible running buddy quickly | NFR-1.1, UC-02 | Buddy Discovery is a dedicated core-domain service reached over gRPC; it delegates candidate search to Profile ([0008](../adr/0008-use-a-microservices-architecture.md), [0018](../adr/0018-build-phase-1-with-six-services.md)) |
| Plan a run around weather and air quality | NFR-1.3, NFR-3.2, UC-03 | Run Conditions isolates the external provider behind an adapter and caches readings in Redis with a TTL and a stale fallback ([0006](../adr/0006-service-collaboration-and-provider-isolation.md), [0017](../adr/0017-cache-weather-and-air-quality-in-redis.md)) |
| Protect runners' location privacy | NFR-4.1 | Searches work on coarsened area cells; exact meeting points are shown only to joined participants ([0016](../adr/0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md), [0030](../adr/0030-area-cells-on-a-0-01-degree-grid.md)) |
| Keep failures of non-critical services contained | NFR-3.3, NFR-3.4 | Notifications travel through RabbitMQ with a transactional outbox, so a Notification outage never fails a buddy request ([0021](../adr/0021-rabbitmq-for-local-domain-event-delivery.md)) |
| Let each team member own and change a service independently | Course team of 5, NFR-2.3 | One service per bounded context, each with its own database and contract ([0005](../adr/0005-service-boundaries-and-data-ownership.md), [0015](../adr/0015-give-each-service-its-own-postgresql-database.md)) |
| Secure every call, not only the edge | NFR-5.3, NFR-5.4 | The gateway forwards the caller's JWT; every service validates it and authorizes the action itself ([0013](../adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md), [0014](../adr/0014-authorize-every-request-in-the-service-that-owns-it.md), [0022](../adr/0022-service-authorization-for-the-local-stack.md)) |

## Constraints

- **Course checklist.** The system must include at least one REST service, one gRPC service, one message-broker service, an API gateway, a relational and a NoSQL database, and a service-discovery mechanism. Protocol choices per service were made with this list in mind.
- **Team and time.** Five members, one semester. Phase 1 builds a reduced service set first ([0018](../adr/0018-build-phase-1-with-six-services.md)).
- **Mixed stacks.** Services were built by different members in Rails, Python and Node.js ([0007](../adr/0007-backend-language-and-framework.md), [0020](../adr/0020-python-stack-for-local-delivery-services.md), [0027](../adr/0027-typescript-for-the-profile-service.md)). Contracts (OpenAPI, protobuf, event JSON schemas) are the integration point.
- **Local deployment only.** Everything runs on one laptop: Docker Compose for the demo, a single-node k3s cluster for the deployment and load tests. There is no TLS and no high-availability data tier.
- **Local trust model.** Services share an HS256 signing secret; this is acceptable only for the local demo ([0022](../adr/0022-service-authorization-for-the-local-stack.md)).

## Key decisions

| Decision | ADR |
| --- | --- |
| Microservices, one per bounded context, behind an API gateway | [0008](../adr/0008-use-a-microservices-architecture.md) |
| React single-page application; the browser talks only to the gateway | [0009](../adr/0009-build-the-web-client-as-a-react-single-page-application.md), [0024](../adr/0024-frontend-framework.md) |
| Database per service on PostgreSQL; Redis as the NoSQL weather store | [0015](../adr/0015-give-each-service-its-own-postgresql-database.md), [0017](../adr/0017-cache-weather-and-air-quality-in-redis.md) |
| RabbitMQ with a transactional outbox for domain events | [0021](../adr/0021-rabbitmq-for-local-domain-event-delivery.md) |
| Short-lived JWTs with rotating refresh cookies | [0013](../adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md), [0026](../adr/0026-session-management.md) |
| Authorization inside each owning service | [0014](../adr/0014-authorize-every-request-in-the-service-that-owns-it.md) |

Where the build differs from the Deliverable 2 design, see the note in [`c4-level-2-container.md`](c4-level-2-container.md).
