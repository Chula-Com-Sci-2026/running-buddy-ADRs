# Running Buddy App

A web app that helps runners find each other, schedule spontaneous group runs at a pinned meeting
point, and plan around the weather. Term project for **2110521 Software Architectures**.

## Documentation

### Architecture (final presentation, Part 1)

| Document | What it covers |
| --- | --- |
| [Requirements](docs/requirements.md) | Use cases, functional and non-functional requirements (NFR IDs used everywhere else) |
| [Goals, constraints and key decisions](docs/architecture/goals-and-constraints.md) | Architectural drivers and the ADRs that answer them |
| [C4 Level 1: System context](docs/architecture/c4-level-1-system-context.md) | Users and external systems |
| [C4 Level 2: Containers](docs/architecture/c4-level-2-container.md) | As-built services, databases, broker and protocols; changes since the Deliverable 2 design |
| [Sequence: buddy request](docs/architecture/sequence-buddy-request.md) | Gateway → gRPC → outbox → RabbitMQ → notification |
| [Deployment overview](docs/architecture/deployment.md) | k3s and Docker Compose deployments, service discovery |
| [Deliverable 2 target design](docs/architecture/README.md) | Original microservice design, operations and collaborators, phase 1 scope |

### Quality and risk (final presentation, Part 2)

| Document | What it covers |
| --- | --- |
| [Risk matrix](docs/risk-matrix.md) | Six measured risks scored by likelihood × impact, with mitigations |
| [Quality attribute: resilience](docs/quality-attribute-resilience.md) | Pod-replacement scenario: 12,000 of 12,000 requests succeeded |
| [Load tests (k6)](https://github.com/Chula-Com-Sci-2026/running-buddy-integration/blob/main/load-tests/README.md) | Scenarios, NFR targets, measured results and bottlenecks |

### Deployment process

| Document | What it covers |
| --- | --- |
| [Kubernetes (k3s) deployment](https://github.com/Chula-Com-Sci-2026/running-buddy-integration/blob/main/k8s/README.md) | Cluster, image build, secrets, migrations, seed, checks; service-discovery measurements |
| [Docker Compose stack](https://github.com/Chula-Com-Sci-2026/running-buddy-integration/blob/main/README.md) | Live-demo startup, seed accounts, business flows, database evidence |

### Decisions and contracts

| Document | What it covers |
| --- | --- |
| [Architecture Decision Records](docs/adr/) | Numbered ADRs; decision log below |
| [Service and gateway contracts](https://github.com/Chula-Com-Sci-2026/running-buddy-contracts) | OpenAPI files, the Discovery protobuf and event schemas |
| [Run Scheduling API (draft)](docs/api/run-scheduling.openapi.yaml) | Early draft contract |
| [Repo setup handoff](docs/REPO-SETUP-HANDOFF.md) | How this repository was created |

## Architecture Decision Records (ADRs)

We record every significant architecture decision as an ADR — a numbered markdown file in
[`docs/adr/`](docs/adr/) — and we manage them with **[adr-tools (npryce)](https://github.com/npryce/adr-tools)**.
ADRs live next to the code they describe so that a decision and its implementation version together
and flow through the same pull-request review.

### Working with ADRs

Install the tool (see the [adr-tools README](https://github.com/npryce/adr-tools) — `brew install
adr-tools` on macOS; clone + PATH on Linux/WSL/Git Bash), then from the repo root:

```bash
# create the next numbered ADR
adr new "Short title of the decision"

# create an ADR that supersedes an earlier one (e.g. ADR 4)
adr new -s 4 "Replacement decision"

# regenerate the table of contents
adr generate toc
```

**Conventions**

- Numbering is append-only; never renumber or delete an ADR.
- Status flows `Proposed → Accepted`, then `Superseded by NNNN` if reversed. Superseded ADRs stay.
- One ADR per pull request, reviewed by the team before merge.
- Each ADR references the architectural driver / NFR it addresses.

### Decision log

| Repo ADR | Decision | Source decision | Status |
| --- | --- | --- | --- |
| [0001](docs/adr/0001-record-architecture-decisions.md) | Record architecture decisions | Repo setup | Accepted |
| [0002](docs/adr/0002-database-selection-and-geospatial-indexing.md) | Database and geospatial indexing | Deliverable 1 ADR-001 | Superseded by 0015, 0016, 0017 |
| [0003](docs/adr/0003-frontend-framework-and-ui-styling.md) | Frontend framework and styling | Deliverable 1 ADR-002 | Superseded by 0009, 0010, 0011 |
| [0004](docs/adr/0004-authentication-phone-verification-and-authorization.md) | Authentication and authorization | Deliverable 1 ADR-003 | Superseded by 0012, 0013, 0014 |
| [0005](docs/adr/0005-service-boundaries-and-data-ownership.md) | Service boundaries and data ownership | Deliverable 2 | Proposed, amended by 0018 |
| [0006](docs/adr/0006-service-collaboration-and-provider-isolation.md) | Service collaboration and provider isolation | Deliverable 2 | Proposed, amended by 0019 |
| [0007](docs/adr/0007-backend-language-and-framework.md) | Backend stack for discovery, reputation, and trust services | Implementation planning | Proposed |
| [0008](docs/adr/0008-use-a-microservices-architecture.md) | Use a microservices architecture | Deliverable 1 (revised) ADR-001 | Accepted |
| [0009](docs/adr/0009-build-the-web-client-as-a-react-single-page-application.md) | Build the web client as a React single-page application | Deliverable 1 (revised) ADR-002 | Accepted; photo-file exception in 0029 |
| [0010](docs/adr/0010-style-the-web-client-with-tailwind-css.md) | Style the web client with Tailwind CSS | Deliverable 1 (revised) ADR-002 | Superseded locally by 0025 |
| [0011](docs/adr/0011-render-maps-with-leaflet-and-openstreetmap-tiles.md) | Render maps with Leaflet and OpenStreetMap tiles | Deliverable 1 (revised) ADR-002 | Accepted |
| [0012](docs/adr/0012-verify-phone-ownership-with-sms-one-time-passwords.md) | Verify phone ownership with SMS one-time passwords | Deliverable 1 (revised) ADR-003 | Accepted |
| [0013](docs/adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md) | Authenticate with short-lived JWTs and rotating refresh tokens | Deliverable 1 (revised) ADR-003 | Accepted; local refinements in 0022, 0023, 0026 |
| [0014](docs/adr/0014-authorize-every-request-in-the-service-that-owns-it.md) | Authorize every request in the service that owns it | Deliverable 1 (revised) ADR-003 | Accepted |
| [0015](docs/adr/0015-give-each-service-its-own-postgresql-database.md) | Give each service its own PostgreSQL database | Deliverable 1 (revised) ADR-004 | Accepted |
| [0016](docs/adr/0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md) | Search nearby runners and runs with PostGIS on coarsened cells | Deliverable 1 (revised) ADR-004 | Accepted; cell scheme in 0030 |
| [0017](docs/adr/0017-cache-weather-and-air-quality-in-redis.md) | Cache weather and air quality in Redis | Deliverable 1 (revised) ADR-004 | Accepted |
| [0018](docs/adr/0018-build-phase-1-with-six-services.md) | Build phase 1 with six services | Deliverable 2 review | Accepted |
| [0019](docs/adr/0019-call-services-over-rest-until-the-phase-2-event-broker.md) | Call services over REST until the phase 2 event broker | Deliverable 2 review | Broker deferral partially superseded locally by 0021 and 0028 |
| [0020](docs/adr/0020-python-stack-for-local-delivery-services.md) | Python APIs for the remaining local delivery services | Deliverable 4 local implementation | Accepted locally; partially superseded for Profile by 0027 |
| [0021](docs/adr/0021-rabbitmq-for-local-domain-event-delivery.md) | RabbitMQ for local domain event delivery | Deliverable 4 local implementation | Accepted locally |
| [0022](docs/adr/0022-service-authorization-for-the-local-stack.md) | Service authorization in the local stack | Deliverable 4 local implementation | Accepted locally |
| [0023](docs/adr/0023-bcrypt-for-team-identity-credentials.md) | Bcrypt for credentials in the team Identity implementation | Deliverable 4 local implementation | Accepted locally |
| [0024](docs/adr/0024-frontend-framework.md) | React for the browser application | Deliverable 4 local implementation | Accepted locally |
| [0025](docs/adr/0025-ui-styling.md) | Shared CSS and reusable components for UI styling | Deliverable 4 local implementation | Accepted locally |
| [0026](docs/adr/0026-session-management.md) | Short-lived access tokens with rotating cookie-based refresh sessions | Deliverable 4 local implementation | Accepted locally |
| [0027](docs/adr/0027-typescript-for-the-profile-service.md) | TypeScript for the Profile service | Profile alignment | Accepted |
| [0028](docs/adr/0028-rabbitmq-for-profile-updated.md) | RabbitMQ for ProfileUpdated | Profile alignment | Accepted |
| [0029](docs/adr/0029-object-storage-for-profile-photos.md) | Object storage for profile photos | Profile alignment | Accepted |
| [0030](docs/adr/0030-area-cells-on-a-0-01-degree-grid.md) | Area cells on a 0.01° grid | Profile alignment | Accepted |

Repository numbers continue after the existing ADR-0001; the Deliverable 1 numbers are source-document labels.

Local acceptance is authorized by the project owner and does not claim teammate review. Published ADRs 0001–0019 keep their numbers and original decision bodies. The earlier split decisions remain authoritative unless a later record explicitly supersedes a scoped part. ADRs 0024–0026 are the three focused records approved in this revision; 0020–0023 reconcile previously unpublished implementation decisions whose local numbers collided with the merged history. Broader scope, OTP and geospatial implementation gaps still belong in the progress report.

## Getting started

See the [integration repository](https://github.com/Chula-Com-Sci-2026/running-buddy-integration) for the implemented stack, startup and verification commands. Each service has its own repository under [Chula-Com-Sci-2026](https://github.com/Chula-Com-Sci-2026).
