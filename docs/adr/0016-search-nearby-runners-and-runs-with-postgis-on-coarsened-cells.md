# 16. Search nearby runners and runs with PostGIS on coarsened cells

Date: 2026-10-03

## Status

Accepted

Supersedes [2. Database selection and geospatial indexing](0002-database-selection-and-geospatial-indexing.md)

[ADR-0030](0030-area-cells-on-a-0-01-degree-grid.md) settles its cell scheme and size: a grid of 0.01° in latitude and longitude.

## Context

Two use cases are radius searches combined with other filters:

- **Find a running buddy (UC-02).** Runners within a radius of the searcher's area, filtered by pace range, availability window, experience level, and verified status, in under 3 s at p95 over 10,000 profiles (NFR-1.1). Buddy Discovery runs this on its own copy of profile data.
- **Browse nearby runs (UC-08).** Upcoming runs within a radius, filtered by date and target pace, in under 2 s at p95 (NFR-1.2). Run Scheduling runs this on its runs.

A runner's exact area must never be exposed, directly or by repeating searches from different places to triangulate it (NFR-4.1). A run's meeting point is the only precise location shared, and only with the runners who may see that run. Both services already use PostgreSQL ([15. Give each service its own PostgreSQL database](0015-give-each-service-its-own-postgresql-database.md)).

Options considered:

- **PostGIS with a GiST index (`ST_DWithin`).** Radius and attribute filters in one SQL query, inside the database each service already runs. Performance at 10,000 profiles must be measured.
- **MongoDB with `2dsphere` indexes.** Good geospatial queries, but a second primary database technology, against ADR-0015.
- **Elasticsearch (or another search engine) with geo queries.** The fastest for complex ranked search, but a third data system to run and keep in sync, which the team cannot afford this semester.
- **Plain latitude and longitude columns with a bounding box and distance formula.** No extension needed, but no spatial index, so it is unlikely to meet NFR-1.1 as data grows, and the team would hand-write geometry code.

## Decision

We will run both radius searches with PostGIS and GiST indexes: in Discovery DB for runners and in Run DB for runs. Buddy Discovery stores each runner's location only as a coarsened cell of roughly 1-2 km, received from Profile, and filters and ranks on cell centers. The exact preferred area stays in Profile DB. Meeting points are stored exactly in Run DB.

## Consequences

- Both searches are single SQL queries inside the owning service, with no extra search system; a load test with 10,000 profiles must confirm NFR-1.1 and NFR-1.2.
- Neither responses nor repeated searches can reveal a runner's exact area, because Buddy Discovery never holds it.
- Ranking loses some precision: two runners in the same cell look equally close.
- Run Scheduling must hide an invite-only run's meeting point from runners who are not invited.
- Follow-on decision: the cell scheme (for example a geohash or H3 grid) and its size.

Source: Deliverable 1 (revised), "ADR-004: Database Selection and Geospatial Indexing," and the location-privacy rule from "ADR-003: Authentication, Phone Verification, and Zero Trust Authorization," split per review feedback into one ADR per data need.
