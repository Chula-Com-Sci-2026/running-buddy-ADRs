# Running Buddy App

A web app that helps runners find each other, schedule spontaneous group runs at a pinned meeting
point, and plan around the weather. Term project for **2110521 Software Architectures**.

## Documentation

- **Requirements:** [`docs/requirements.md`](docs/requirements.md) _(to be added)_
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
| [0002](docs/adr/0002-database-selection-and-geospatial-indexing.md) | Database and geospatial indexing | Deliverable 1 ADR-001 | Accepted |
| [0003](docs/adr/0003-frontend-framework-and-ui-styling.md) | Frontend framework and styling | Deliverable 1 ADR-002 | Accepted |
| [0004](docs/adr/0004-authentication-phone-verification-and-authorization.md) | Authentication and authorization | Deliverable 1 ADR-003 | Accepted |
| [0005](docs/adr/0005-service-boundaries-and-data-ownership.md) | Service boundaries and data ownership | Deliverable 2 | Proposed |
| [0006](docs/adr/0006-service-collaboration-and-provider-isolation.md) | Service collaboration and provider isolation | Deliverable 2 | Proposed |

Repository numbers continue after the existing ADR-0001; the Deliverable 1 numbers are source-document labels.

## Getting started

_Local run instructions will be added with the service implementation (see proposed ADR-0005)._
