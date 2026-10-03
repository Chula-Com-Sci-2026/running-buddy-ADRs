# 15. Give each service its own PostgreSQL database

Date: 2026-10-03

## Status

Accepted

Supersedes [2. Database selection and geospatial indexing](0002-database-selection-and-geospatial-indexing.md)

## Context

Each microservice owns its data and no other service reads it directly ([8. Use a microservices architecture](0008-use-a-microservices-architecture.md)). Most service data is relational and guarded by rules that need transactions inside one service: a run never exceeds its participant limit (FR-6.5), at most one pending buddy request exists per pair of runners (FR-4.4), and from phase 2 only confirmed participants rate each other (FR-7.3). No scheduled run may be lost on restart (NFR-3.4), and profile, run, and rating data are backed up daily (NFR-6.6). The team has more operating experience with PostgreSQL than with document databases, and little time to run several database technologies.

This ADR covers the system-of-record store for each service. Nearby search is [16. Search nearby runners and runs with PostGIS on coarsened cells](0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md); the weather cache is [17. Cache weather and air quality in Redis](0017-cache-weather-and-air-quality-in-redis.md).

Options considered:

- **One shared database for all services.** Easiest to set up and query across. Services become coupled through shared tables: one schema change can break several services, which defeats separate deployment.
- **PostgreSQL, one logical database per service.** ACID transactions for each service's rules, mature tooling, and one technology for the whole team. Schema changes need migrations, and reports that span services cannot be a single SQL join.
- **MongoDB per service.** A flexible schema suits evolving profile fields. Most of our data is relational with cross-row rules, multi-document transactions are an add-on to the document model rather than the normal way of working, and the team knows it less well.
- **A different database per service as each fits best.** The best fit per service, but every extra technology multiplies backup, monitoring, and learning cost for a five-person team.

## Decision

We will give each service that stores data its own logical PostgreSQL 16 database, with its own credentials, and allow no queries across service databases.

## Consequences

- Transactions inside each database enforce that service's rules; write-ahead logging plus daily backups cover NFR-3.4 and NFR-6.6.
- One database technology keeps the team's operating load low.
- Data needed by another service travels through APIs or events, never through a join, so copies such as Buddy Discovery's candidates can lag behind their source.
- If all logical databases share one PostgreSQL instance, that instance is a single point of failure for every service; the deployment decision must give it a standby or accept the risk against NFR-3.1.
- Schema changes need migrations. If profile attributes change much faster than expected, `jsonb` columns can absorb the variation.

Source: Deliverable 1, "Database Selection and Geospatial Indexing," split per review feedback into one ADR per data need.
