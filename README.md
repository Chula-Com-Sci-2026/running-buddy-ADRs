# Running Buddy App

A web app that helps runners find each other, schedule spontaneous group runs at a pinned meeting
point, and plan around the weather. Term project for **2110521 Software Architectures**.

## Documentation

- **Requirements:** [`docs/requirements.md`](docs/requirements.md) _(to be added)_
- **Architecture Decision Records:** [`docs/adr/`](docs/adr/)
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

## Getting started

_Local run instructions to be added once the service structure is decided (see ADR-0002)._
