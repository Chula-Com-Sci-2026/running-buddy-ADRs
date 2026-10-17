# 20. Python APIs for the remaining local delivery services

Date: 2026-10-16

## Status

Accepted for the local Deliverable 4 implementation, under the project owner's delegated approval. Teammates may supply replacements behind the same contracts.

## Context

Profile, Run Conditions, Run Scheduling, Notification and the gateway need local implementations; a Python Identity implementation also exists as a fallback. Discovery and Reputation already use Rails and must retain their working histories. The driver is completing the three business flows with independently replaceable services. Extending Rails everywhere provides consistency but adds framework setup; FastAPI provides typed validation and generated OpenAPI with a smaller implementation surface, at the cost of a second runtime and duplicated infrastructure helpers.

## Decision

Use Python 3.13 and FastAPI for Profile, Run Conditions, Run Scheduling, Notification and the gateway, with pinned dependencies, psycopg for PostgreSQL and a Redis client for Run Conditions. Retain Rails for Discovery and Reputation. Each repository contains its own infrastructure helpers, migrations and container build; no service imports another service's domain code or queries another service's database.

## Consequences

The active team Identity implementation uses Node/Express; Python Identity is retained only as a fallback. Together with Rails and Python, there are three active runtime stacks to maintain. Compatibility depends on HTTP, gRPC, JWT and event contracts, not language. Local implementations can be replaced through configured endpoints only when those contracts match. This records the Python-service scope alongside proposed ADR-0007's stack discussion without changing its Rails choice for Discovery and Reputation.
