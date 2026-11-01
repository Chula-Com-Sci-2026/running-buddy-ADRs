# 30. Area cells on a 0.01° grid

Date: 2026-11-01

## Status

Accepted

Settles the cell scheme and size that [ADR-0016](0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md) left open: a grid of 0.01° in latitude and longitude.

## Context

ADR-0016 has Buddy Discovery store each runner's location "only as a coarsened cell of roughly 1-2 km, received from Profile", as NFR-4.1 requires, and leaves the cell scheme and size open. Profile receives an exact point and a place name for each runner's area.

Options considered:

- **Geohash, 6 characters** (about 1.2 × 0.6 km). A text code that needs a library in each runtime to turn back into a point or find its neighbours, and its cells aren't square.
- **H3, resolution 7** (hexagons, about 1.2 km a side). Even distances to neighbours, but a native library in every runtime and an extension for PostGIS.
- **Latitude and longitude rounded to a 0.01° grid.** An ordinary point that PostGIS and plain great-circle maths use directly, with no library; about 1.1 × 1.1 km in Bangkok.

## Decision

A runner's area cell, `areaCell` `{latitude, longitude}`, is their exact point with each coordinate rounded to two decimal places. This grid is chosen for its simplicity and ease of implementation: no service needs a library for it, and PostGIS uses a cell as an ordinary point. Profile computes it and doesn't store it, so changing the size later means changing Profile's rounding and its contract, then republishing every profile ([ADR-0028](0028-rabbitmq-for-profile-updated.md)). Other runners, events and searches get only the area name and the cell; the exact point is stored, but only the runner themselves gets it back. Profile's temporary search measures great-circle distances between cells, shown to one decimal of a kilometre.

## Consequences

- A cell is about 1.1 km across, within NFR-4.1's "roughly 1-2 km", so a position shown to others is off by at most about 0.8 km.
- Searching from different places can't narrow a runner down further than their cell, because every distance is measured between cells.
- Ranking is less precise: two runners in the same cell are 0 km apart, as ADR-0016 expects.
- A runner whose point sits near a cell edge moves to the neighbouring cell after a small change of point.

Implementation evidence: `src/domain/area-cell.ts` and `docs/domain.md` ("Area cells") in the `profile-service` repository.
