# 5. Service boundaries and data ownership

Date: 2026-09-20

## Status

Proposed

## Context

Independent deployment and exclusive data ownership are architectural drivers (NFR-7.1, NFR-7.3). Deliverable 2 divides the product by responsibility and lists each service's data store and collaborators. The repo currently contains a draft Run Scheduling API contract, but no service implementation.

## Decision

Use these eight service boundaries from Deliverable 2:

| Service | Owns | Data store |
| --- | --- | --- |
| Buddy Discovery | Runner search, buddy requests, blocking, candidate projections | Discovery DB (PostgreSQL + PostGIS) |
| Run Conditions | Forecast, air quality, suitability, alternative slots | Weather cache (Redis) |
| Run Scheduling | Runs, participants, reminders, forecast rechecks | Run DB (PostgreSQL + PostGIS) |
| Profile | Runner profiles and field visibility | Profile DB (PostgreSQL) |
| Reputation | Attendance, ratings, reliability, running records | Reputation DB (PostgreSQL) |
| Trust & Safety | Verification badges, reports, moderation, suspension | Trust & Safety DB (PostgreSQL) |
| Identity | Registration, login, tokens, OTP, password reset | Identity DB (PostgreSQL) |
| Notification | Delivery, contact projection, preferences | Notification DB (PostgreSQL) |

Each service owns its schema and writes only its own store. Other services receive needed state through APIs or events. Discovery's candidate data is a projection; Identity and Trust & Safety remain separate so possession of a phone number and approval of a badge are distinct responsibilities.

## Consequences

- Services can evolve and deploy independently, but distributed flows need explicit contracts and failure handling.
- Candidate, verification, rating, and contact projections can lag behind their sources; consumers must handle stale or duplicate updates.
- Eight services create substantial operational and contract-testing work for a semester project. The team should review whether all boundaries need separate deployables in the first implementation.
- Exact location and public-profile rules need consistent enforcement across Discovery, Profile, and Run Scheduling (NFR-5.1).

Source: Deliverable 2, “Microservice Design with Collaborations,” service table; Deliverable 1, NFR-7.1 and NFR-7.3.
