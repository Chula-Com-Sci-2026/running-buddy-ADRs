# 11. Render maps with Leaflet and OpenStreetMap tiles

Date: 2026-10-03

## Status

Accepted

Supersedes [3. Frontend framework and UI styling](0003-frontend-framework-and-ui-styling.md)

## Context

Organizers pin a meeting point on a map (UC-03), and runners browse upcoming runs on a map (UC-08). The map is the heaviest asset on these screens, and first contentful paint must stay under 3 s on 5G (NFR-1.4). Distances and radius search are computed in the backend ([16. Search nearby runners and runs with PostGIS on coarsened cells](0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md)), so the client needs only display, panning, and pin placement, not routing or geocoding. The map tile service is an external system (the Map Provider actor), so it can change its terms or pricing.

Options considered:

- **Leaflet with OpenStreetMap-based tiles.** Small, mature, open-source library that does exactly display and pins; tile provider is configurable. Raster tiles look less polished than vector maps, and a tile provider's usage policy must be respected.
- **Google Maps JavaScript API.** Familiar look and rich place data. Needs an API key and billing account, and ties the client to one vendor's API and terms.
- **MapLibre GL (vector maps).** Smooth vector rendering and styling. A larger bundle and WebGL requirement on low-end phones, and more setup than this project needs.

## Decision

We will render maps with Leaflet and OpenStreetMap-based tiles, behind a thin map wrapper component so the tile provider, or the library, can be replaced without touching the screens that use it.

## Consequences

- The map stays small and free of vendor lock-in; swapping the tile provider is a configuration change inside the wrapper.
- The wrapper is the anticorruption layer between the web client and the Map Provider; screens must not import Leaflet directly.
- The map bundle must still be lazy-loaded to protect NFR-1.4.
- Public OpenStreetMap tile servers have usage limits; production needs a tile provider whose terms fit the expected traffic, a follow-on choice.

Source: Deliverable 1 (revised), "ADR-002: Frontend Framework and UI Styling," split so the map library is its own decision.
