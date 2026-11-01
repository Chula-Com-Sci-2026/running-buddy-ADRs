# 27. TypeScript for the Profile service

Date: 2026-11-01

## Status

Accepted

Partially supersedes [ADR-0020](0020-python-stack-for-local-delivery-services.md) for Profile only.

## Context

ADR-0020 names Python and FastAPI for Profile. It allows that "Teammates may supply replacements behind the same contracts", and says local implementations "can be replaced through configured endpoints only when those contracts match". A TypeScript Profile, `profile-service`, implements the profile requirements (FR-2.1 to FR-2.8, with weekly availability as free text for now), meets its part of NFR-4.1 and [ADR-0016](0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md), and publishes `ProfileUpdated` for Buddy Discovery. To do that, it departs from today's `profile.openapi.json` in the eight ways listed under Decision. So this replacement comes with an updated contract: `profile.openapi.json` changes to match it.

## Decision

Profile is the TypeScript service `profile-service`, on its own PostgreSQL database ([ADR-0015](0015-give-each-service-its-own-postgresql-database.md)). It verifies tokens as [ADR-0022](0022-service-authorization-for-the-local-stack.md) describes, takes the runner's id from `sub`, and accepts and returns `X-Correlation-Id`. It keeps the team contract's paths and field names, adds `/ready`, `/docs` and `/openapi.yaml`, and departs from the contract in eight named ways:

1. **Visibility:** `PUT /api/v1/me/profile/visibility` sets which fields other runners see (FR-2.8).
2. **Photo:** `POST /api/v1/me/profile/photo-uploads`, `PUT` and `DELETE /api/v1/me/profile/photo`, and `photoUrl` (FR-2.1). Storage is [ADR-0029](0029-object-storage-for-profile-photos.md).
3. **Usual distance:** an optional `usualDistance`: `Under 5 km`, `5–10 km` or `Over 10 km` (FR-2.2).
4. **Gear:** an optional free-text list (FR-2.6).
5. **Area for other runners:** other runners see only the area name and `areaCell`. The exact point (`latitude`, `longitude`) is stored, but only the runner themselves gets it back (NFR-4.1, [ADR-0030](0030-area-cells-on-a-0-01-degree-grid.md)).
6. **The public card:** `GET /api/v1/runners/{id}` returns every key, always; a field the runner hid or never set is `null`.
7. **Errors:** every error body is the frontend's `ApiErrorBody` (`code`, `message`, `correlationId`, and `fieldErrors` on a 422 when a field can be named). Profile also answers 401, 404, 500 and 503.
8. **Runner search:** `GET /api/v1/runners` returns other runners' public cards, each with `distanceKm` measured between area cells, nearest first. It stays only until Discovery's own search is live.

## Consequences

- `compose.local-integration.yaml` in `profile-service` runs it in the local stack in place of the Python Profile, with its own PostgreSQL, MinIO and broker user.
- The gateway needs to forward the visibility and photo routes to Profile.

Implementation evidence: `docs/openapi.yaml`, `docs/domain.md` and `compose.local-integration.yaml` in the `profile-service` repository.
