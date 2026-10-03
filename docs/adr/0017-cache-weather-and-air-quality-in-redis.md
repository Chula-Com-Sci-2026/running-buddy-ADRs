# 17. Cache weather and air quality in Redis

Date: 2026-10-03

## Status

Accepted

Supersedes [2. Database selection and geospatial indexing](0002-database-selection-and-geospatial-indexing.md)

## Context

The Run Conditions Service shows the forecast, PM2.5, and a run-suitability level for a place and time (UC-09, FR-5.1, FR-5.2). It gets them from an external Weather & Air-Quality Provider, which limits and may charge for calls. Cached readings must come back in under 500 ms at p95, and a cache miss in under 3 s (NFR-1.3). When the provider is down, the service must serve the last cached forecast with a staleness warning, and scheduling must keep working (NFR-3.2). Readings are short-lived values keyed by area and time; losing them costs a provider call, not business data. The forecast attached to a scheduled run is business data and belongs to Run Scheduling.

Options considered:

- **Redis.** In-memory key-value store with very fast reads; keys by area cell and time bucket fit naturally. A second database technology to run.
- **A cache table in a PostgreSQL database owned by Run Conditions.** No new technology, durable. Slower than memory for a hot key-value workload, and cache housekeeping (expiry, cleanup) must be written by hand.
- **In-process memory cache.** Fastest and nothing to run. Each instance has its own cache that empties on every restart, so after a restart during a provider outage there is no last forecast to serve, which breaks NFR-3.2.

## Decision

We will cache weather and air-quality readings in Redis, owned by the Run Conditions Service, keyed by area cell and time bucket. Each entry records when it was fetched: it counts as fresh for a short period and is kept much longer as a stale fallback, rather than expiring when it stops being fresh. The forecast attached to a run is stored with the run in Run DB.

## Consequences

- Cache hits meet NFR-1.3 easily, and provider calls stay within its limits.
- A last forecast is available during a provider outage, so NFR-3.2 holds.
- The project runs two database technologies, PostgreSQL and Redis. Redis holds only data that can be fetched again, so it needs no backup.
- Losing Redis degrades the weather display until the cache refills, but never removes a run's attached forecast.
- Follow-on decisions: the fresh period, the stale retention period, and the cell and time-bucket sizes.

Source: Deliverable 1, "Database Selection and Geospatial Indexing," split per review feedback into one ADR per data need.
