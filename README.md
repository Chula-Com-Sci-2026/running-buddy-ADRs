# Running Buddy App

A web app that helps runners find each other, schedule spontaneous group runs at a pinned meeting
point, and plan around the weather. Term project for **2110521 Software Architectures**.

## Documentation

- **Requirements:** [`docs/requirements.md`](docs/requirements.md)
- **Microservice architecture:** [`docs/architecture/`](docs/architecture/) (target design and phase 1 scope)
- **Architecture Decision Records:** [`docs/adr/`](docs/adr/)
- **Run Scheduling Service API (draft):** [`docs/api/run-scheduling.openapi.yaml`](docs/api/run-scheduling.openapi.yaml)
- **Repo setup handoff:** [`docs/REPO-SETUP-HANDOFF.md`](docs/REPO-SETUP-HANDOFF.md)

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
| [0009](docs/adr/0009-build-the-web-client-as-a-react-single-page-application.md) | Build the web client as a React single-page application | Deliverable 1 review | Accepted |
| [0010](docs/adr/0010-style-the-web-client-with-tailwind-css.md) | Style the web client with Tailwind CSS | Deliverable 1 review | Accepted |
| [0011](docs/adr/0011-render-maps-with-leaflet-and-openstreetmap-tiles.md) | Render maps with Leaflet and OpenStreetMap tiles | Deliverable 1 review | Accepted |
| [0012](docs/adr/0012-verify-phone-ownership-with-sms-one-time-passwords.md) | Verify phone ownership with SMS one-time passwords | Deliverable 1 review | Accepted |
| [0013](docs/adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md) | Authenticate with short-lived JWTs and rotating refresh tokens | Deliverable 1 review | Accepted |
| [0014](docs/adr/0014-authorize-every-request-in-the-service-that-owns-it.md) | Authorize every request in the service that owns it | Deliverable 1 review | Accepted |
| [0015](docs/adr/0015-give-each-service-its-own-postgresql-database.md) | Give each service its own PostgreSQL database | Deliverable 1 review | Accepted |
| [0016](docs/adr/0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md) | Search nearby runners and runs with PostGIS on coarsened cells | Deliverable 1 review | Accepted |
| [0017](docs/adr/0017-cache-weather-and-air-quality-in-redis.md) | Cache weather and air quality in Redis | Deliverable 1 review | Accepted |
| [0018](docs/adr/0018-build-phase-1-with-six-services.md) | Build phase 1 with six services | Deliverable 2 review | Accepted |
| [0019](docs/adr/0019-call-services-over-rest-until-the-phase-2-event-broker.md) | Call services over REST until the phase 2 event broker | Deliverable 2 review | Accepted |

Repository numbers continue after the existing ADR-0001; the Deliverable 1 numbers are source-document labels.

## Getting started

_Local run instructions will be added with the service implementation (see proposed ADR-0005)._
