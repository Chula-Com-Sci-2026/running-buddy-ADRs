# 2. Database selection and geospatial indexing

Date: 2026-09-20

## Status

Accepted

## Context

Runner search must finish within 2 seconds at p95 over 10,000 profiles (NFR-1.1), and nearby runs within 1 second at p95 (NFR-1.2). Both combine radius queries with other filters. Services must own their data (NFR-7.3), while scheduling and ratings need transactional integrity and runs must survive restarts (NFR-3.4).

The team considered MongoDB with `2dsphere` indexes and a relational database paired with Elasticsearch. MongoDB offers flexible documents but adds transactional and operational tradeoffs. A separate search engine creates another data system and a synchronization problem.

## Decision

Use PostgreSQL 16 with PostGIS as the primary datastore, with one logical database per service and no cross-database joins. Index coordinates with GiST on `geography(Point, 4326)` and use B-tree indexes for accompanying pace, availability, and experience filters. Use Redis for cached weather and air-quality readings, keyed by location cell and time bucket with an explicit TTL. MongoDB is not the primary store.

## Consequences

- PostGIS supports nearby-runner and nearby-run queries without a separate search engine; the latency targets still require measurement.
- Transactions protect scheduling and rating invariants. Persistence and daily backups must be configured to meet NFR-3.4 and NFR-6.6.
- Schema changes require migrations. `jsonb` can hold rapidly changing profile attributes where appropriate.
- Cross-service reporting goes through service APIs or events, not SQL joins across service databases.
- Redis adds an operational dependency, but losing cached readings does not lose authoritative run data.

Source: Deliverable 1, “ADR-001: Database Selection and Geospatial Indexing.”
