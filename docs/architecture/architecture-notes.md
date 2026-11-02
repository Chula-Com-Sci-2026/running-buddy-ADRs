# Can't Run Alone - Architecture Notes

Background for `service-operations-collaborators.md`, which holds only the deliverable table. These notes keep the reasoning behind it: actors, business use cases, subdomains, the context map, the model, failure handling, walkthroughs, and the decisions still open. `microservice-architecture.drawio` draws the same system (page 1: architecture diagram, page 2: the table, page 3: context map). Architecture version 1 with Kafka. This is the target architecture; phase 1 builds six of the eight services without the event broker, see [the architecture overview](README.md).

Arrow and collaborator convention: **A → B means A starts the interaction.** Requests from the Web App go through the API Gateway as REST/JSON. Two service-to-service requests use gRPC: Run Scheduling → Run Conditions, and Trust & Safety → Identity. Everything else between services goes through the Kafka event broker: a service publishes events to Kafka and consumes events from it, and in both cases the service starts the call (Kafka consumers poll), so the arrow points from the service to Kafka. Responses are not drawn. The context map on page 3 uses a different notation, explained in its own section below.

## Actors

| Actor | Kind | Uses the system through |
|---|---|---|
| Visitor | Person | Web App, only to register and verify a phone number (UC-01, UC-05) |
| Runner | Person | Web App (React SPA on a mobile or desktop browser) |
| Organizer (a Runner) | Person | Web App |
| Participant (a Runner) | Person | Web App |
| Admin | Person | Web App (admin console) |
| SMS/OTP Provider | External system | Called by Identity Service through the SMS/OTP Adapter |
| Email Provider | External system | Called by Identity Service (one-time passwords, password-reset links) and Notification Service (notifications), each through its own Email Adapter |
| Weather & Air-Quality Provider | External system | Called by Run Conditions Service through the Weather Adapter |
| Map Provider (OpenStreetMap tiles) | External system | Called directly by the Web App (Leaflet, [ADR-0011](../adr/0011-render-maps-with-leaflet-and-openstreetmap-tiles.md)) |

## The 3 Business Use Cases

Registration and login (UC-01, UC-06) are account basics, not business use cases, so the three business use cases are:

| # | Business use case | Primary actor | What the actor wants to achieve | Also needs | Services involved |
|---|---|---|---|---|---|
| BUC-1 | UC-02 Find a Compatible Running Buddy | Runner | Find runners who match their pace, area, and free time, and send one a buddy request | UC-14 Respond to a Buddy Request, UC-11 Report | Buddy Discovery, Profile, Trust & Safety, Reputation, Notification |
| BUC-2 | UC-03 Schedule a Run with Weather Check and Pinned Meeting Point | Organizer | Fix a time, a public meeting point, and a pace, see whether the weather and PM2.5 are safe, and get people to join | UC-09 Check Weather, UC-10 Alternative Slots, UC-08 Browse and Join a Run | Run Scheduling, Run Conditions, Buddy Discovery, Trust & Safety, Notification, Map Provider |
| BUC-3 | UC-04 Confirm Attendance, Rate Buddies, and Log Running Record | Participant | Get credit for a run they actually attended, rate the people they ran with, and log distance and time | UC-11 Report a review | Run Scheduling, Reputation, Buddy Discovery, Trust & Safety, Notification |

## Business capabilities and subdomains

The three business use cases need the business capabilities below. Each capability is one subdomain, owned by one service.

The core domain, the part that makes Can't Run Alone worth building, is **matching runners by pace, area, and free time, and judging whether a run's place and time are safe to run in**. The ADR's problem description names exactly these as what Strava, Facebook groups, and Meetup lack. Everything else exists to support them.

| Capability (subdomain) | Needed by | Type | Service | How much to invest |
|---|---|---|---|---|
| Buddy matching: ranking candidates by pace overlap, area proximity, and schedule overlap; buddy requests | BUC-1, BUC-2 | Core | Buddy Discovery | The deepest model and the strongest owner; build the first end-to-end slice through it |
| Run suitability: the run-suitability indicator and better time slots | BUC-2 | Core | Run Conditions | The deepest model and the strongest owner |
| Run scheduling: runs, meeting points, invitations, participants | BUC-2, BUC-3 | Supporting | Run Scheduling | Just enough model to keep its invariants |
| Runner profiles | BUC-1 | Supporting | Profile | A simple model |
| Reputation: attendance, ratings, reliability, run records | BUC-3, BUC-1 | Supporting | Reputation | A simple model |
| Trust and safety: verified badge, verification review, reports, suspensions | BUC-1, BUC-2, BUC-3 | Supporting | Trust & Safety | A simple model |
| Accounts, credentials, sessions, one-time passwords | Every signed-in step; not a business use case | Generic | Identity | Keep thin, so an off-the-shelf identity product could replace it |
| Notification delivery | BUC-1, BUC-2, BUC-3 | Generic | Notification | Keep thin, with no other context's rules inside it |
| Weather and air-quality data, map tiles, SMS, email | All three, and registration | Generic | External providers, behind adapters | Bought |

Core means the reason a runner would choose this app. Supporting means specific to this app, but not why anyone chooses it. Generic means the same in any app. Trust & Safety is supporting rather than core: safety matters to runners, but phone OTP, reports, and suspensions work the same way in any community app.

## Service / Operations / Collaborators, with descriptions

The same operations as the deliverable table, with what each one does. Services are listed core first, then supporting, then generic. For each Kafka event, the Collaborators column names the service at the other end. `[scheduled]` marks a job the service runs on its own timer; "on" names the Kafka events that trigger an operation.

| Service | Operations | Collaborators |
|---|---|---|
| **Buddy Discovery Service**<br>Core subdomain<br>Candidates, ranking, buddy connections (requests, buddies, cooling-off), blocks<br>*Data: Discovery DB (PostgreSQL + PostGIS)* | `searchRunners()` - radius, pace, availability, experience, verified-only; ranked by pace, area, and schedule overlap; returns each candidate's public card exactly as Profile published it (BUC-1 steps 1-3)<br>`sendBuddyRequest()` - refuses suspended, blocked, or cooling-off candidates (BUC-1 step 5)<br>`getBuddyRequests()` - the candidate's pending requests and their notes (UC-14)<br>`respondToBuddyRequest()` - accept or decline (UC-14)<br>`listBuddies()` - accepted buddies, to invite to a run (FR-4.7, BUC-2 step 2)<br>`blockRunner()` - blocked runners never see each other (BUC-1 S-1)<br>`expireBuddyRequests()` [scheduled] - expiry and cooling-off (FR-4.4)<br>`updateCandidate()` on `ProfileUpdated`, `RunnerVerified`, `VerificationRejected`, `AccountSuspended`, `RatingUpdated` - keep the candidates current | **Event Broker (Kafka)** publishes `BuddyRequestSent`, `BuddyRequestAnswered` (consumed by Notification); consumes `ProfileUpdated` from Profile, `RunnerVerified`, `VerificationRejected`, `AccountSuspended` from Trust & Safety, and `RatingUpdated` from Reputation |
| **Run Conditions Service**<br>Core subdomain<br>Run-suitability indicator and alternative time slots, over a forecast and PM2.5 cache<br>*Data: Weather cache (Redis, NoSQL)* | `getRunConditions()` - forecast, PM2.5, and suitability level for a place and time; stale cache with a warning if the provider is down (UC-09, NFR-3.2); gRPC, called only by Run Scheduling<br>`suggestAlternativeSlots()` - better time slots when suitability is poor (UC-10); gRPC, called only by Run Scheduling | **Weather & Air-Quality Provider** (via Weather Adapter) `getForecast()`, `getAirQuality()` |
| **Run Scheduling Service**<br>Supporting subdomain<br>Runs, meeting points, invitations, participant lists, attached forecasts, reminders, cancellation records<br>*Data: Run DB (PostgreSQL + PostGIS)* | `previewRunConditions()` - forecast, PM2.5, and suitability for the pinned place and time, plus alternative slots when suitability is poor (BUC-2 steps 3-4, 4a)<br>`createRun()` - verified organizer only; pinned point, time, distance, pace, limit, public or invite-only; if Run Conditions does not answer in time the forecast is saved as "pending"; publishes `RunScheduled` (BUC-2 steps 1-5, 3b)<br>`updateRun()`, `cancelRun()` - publish `RunChanged` / `RunCancelled`; a cancellation inside the late-cancellation window is flagged `late` and kept as a cancellation record (BUC-2 S-1, FR-6.7, FR-10.3)<br>`browseUpcomingRuns()` - nearby runs on a map or list with their forecast; invite-only runs only to invitees (UC-08)<br>`joinRun()`, `leaveRun()` - participant limit enforced (UC-08)<br>`sendRunReminders()` [scheduled] - publishes `ReminderDue` (BUC-2 step 6)<br>`recheckForecasts()` [scheduled] - re-checks upcoming runs, attaches pending forecasts, publishes `ForecastChanged` when the suitability level changes (BUC-2 S-2, 3b)<br>`endRun()` [scheduled] - at expected end time, publishes `RunEnded` with the participant list (starts BUC-3)<br>`updateRunnerStatus()` on `RunnerVerified`, `VerificationRejected`, `AccountSuspended` - verified / suspended copy used by `createRun()`, invitations, and suspended organizers' runs | **Run Conditions Service** (gRPC) `getRunConditions()` - preview, and the forecast attached on create, edit, and re-check<br>**Run Conditions Service** (gRPC) `suggestAlternativeSlots()` - when suitability is poor<br>**Event Broker (Kafka)** publishes `RunScheduled`, `RunChanged`, `ReminderDue`, `ForecastChanged` (consumed by Notification), `RunCancelled` (Notification, Reputation), and `RunEnded` (Reputation); consumes `RunnerVerified`, `VerificationRejected`, `AccountSuspended` from Trust & Safety |
| **Profile Service**<br>Supporting subdomain<br>Runner profiles: pace, experience, area, availability, gear, field visibility<br>*Data: Profile DB* | `createProfile()`, `updateProfile()` - the pace, area, availability, and experience that BUC-1 matches on; each change publishes `ProfileUpdated`<br>`setFieldVisibility()` - which fields other runners may see (FR-2.8); the only place this rule is applied; also publishes `ProfileUpdated`<br>`getPublicProfile()` - the public card: public fields and a coarsened 1-2 km area only (BUC-1 step 4) | **Event Broker (Kafka)** publishes `ProfileUpdated` (consumed by Buddy Discovery) - the public card (visibility already applied, a hidden field as `null`), with the area only as a 1-2 km cell |
| **Reputation Service**<br>Supporting subdomain<br>Attendance, ratings, aggregate ratings, organizer reliability, run records<br>*Data: Reputation DB* | `openAttendance()` on `RunEnded` - opens the confirmation window for the listed participants and publishes `AttendanceOpened` (BUC-3 step 1)<br>`confirmAttendance()` - two confirmations complete the run (BUC-3 steps 2-3)<br>`closeAttendanceWindows()` [scheduled] - fewer than two confirmations: no ratings (BUC-3 3a)<br>`rateParticipant()` - confirmed participants only; recomputes the aggregate and publishes `RatingUpdated` once three ratings exist (BUC-3 steps 4-6)<br>`getRatingSummary()` - aggregate on a profile, hidden until three ratings exist (BUC-1 step 4, BUC-3 6a)<br>`getReviews()` - the written reviews a runner received, each of which can be reported (UC-11, BUC-3 S-3)<br>`recordLateCancellation()` on `RunCancelled` - a `late` cancellation counts against the organizer's reliability (FR-6.7)<br>`getOrganizerReliability()` - Admin reviews late cancellations (FR-6.7)<br>`logRunRecord()`, `getRunHistory()`, `setRunHistoryVisibility()` - run records, only for runs this service marked completed (BUC-3 S-1, S-2, FR-8.1) | **Event Broker (Kafka)** publishes `AttendanceOpened` (consumed by Notification) and `RatingUpdated` (Buddy Discovery); consumes `RunEnded`, `RunCancelled` from Run Scheduling |
| **Trust & Safety Service**<br>Supporting subdomain<br>Verified badge, verification review queue, reports, suspensions, admin audit log<br>*Data: Trust & Safety DB* | `grantVerifiedBadge()` on `PhoneVerified` - grants the verified badge, adds the verification to the admin queue, and publishes `RunnerVerified` (UC-01 steps 4 and 6, FR-3.2)<br>`getVerificationStatus()` - the verified badge on a public profile (BUC-1 step 4)<br>`reportRunner()`, `reportReview()` - report a runner (BUC-1 S-2) or an abusive review (BUC-3 S-3) into the admin queue (FR-4.6)<br>`getModerationQueue()` - completed verifications and open reports, for the Admin (UC-12, UC-13)<br>`reviewVerification()` - approve, or reject with a reason; a rejection removes the badge and publishes `VerificationRejected`; recorded in the audit log (FR-3.4, FR-10.3)<br>`suspendAccount()` - recorded in the audit log; publishes `AccountSuspended` and calls Identity `lockAccount()` (UC-13, FR-10.2) | **Identity Service** (gRPC) `lockAccount()` - when an account is suspended<br>**Event Broker (Kafka)** publishes `RunnerVerified`, `VerificationRejected`, `AccountSuspended` (consumed by Buddy Discovery, Run Scheduling); consumes `PhoneVerified` from Identity |
| **Identity Service**<br>Generic subdomain<br>Accounts, credentials, sessions, phone and email one-time passwords<br>*Data: Identity DB* | `register()`, `login()`, `logout()`, `refreshToken()` - account basics, JWT with rotating refresh tokens ([ADR-0013](../adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md)); `register()` publishes `ContactChanged`<br>`sendPhoneOtp()`, `verifyPhoneOtp()` - proves the runner holds the phone number and publishes `PhoneVerified` (UC-05); if SMS fails, emails a one-time password instead (UC-01 3b)<br>`verifyEmailOtp()` - confirms the email address only, with no badge (FR-3.5)<br>`requestPasswordReset()`, `resetPassword()` - reset by emailed link (FR-1.3)<br>`lockAccount()` - gRPC, called only by Trust & Safety: revokes refresh tokens and refuses login ([ADR-0013](../adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md)) | **SMS/OTP Provider** (via SMS/OTP Adapter) `sendSms()`<br>**Email Provider** (via Email Adapter) `sendEmail()` - email one-time password and password-reset link<br>**Event Broker (Kafka)** publishes `PhoneVerified` (consumed by Trust & Safety) and `ContactChanged` (Notification) |
| **Notification Service**<br>Generic subdomain<br>Notification delivery, contact addresses, per-category preferences<br>*Data: Notification DB* | `sendNotification()` on `BuddyRequestSent`, `BuddyRequestAnswered`, `RunScheduled`, `RunChanged`, `RunCancelled`, `ReminderDue`, `ForecastChanged`, `AttendanceOpened` - in-app and email notifications, filtered by the recipient's preferences; run changes and cancellations always sent (FR-9)<br>`updateContact()` on `ContactChanged` - the runner's email address for delivery<br>`getNotifications()` - the runner's in-app inbox<br>`updatePreferences()` - turn categories on or off (FR-9.2) | **Email Provider** (via Email Adapter) `sendEmail()`<br>**Event Broker (Kafka)** consumes `ContactChanged` from Identity; `BuddyRequestSent`, `BuddyRequestAnswered` from Buddy Discovery; `RunScheduled`, `RunChanged`, `RunCancelled`, `ReminderDue`, `ForecastChanged` from Run Scheduling; and `AttendanceOpened` from Reputation |

The API Gateway is not a business service: it rejects requests without a valid JWT (checked with Identity's public signing keys, not a call per request), routes the rest to the services above, terminates TLS, and rate-limits discovery (NFR-5.5). Every service checks the JWT again and authorizes by role and ownership ([ADR-0014](../adr/0014-authorize-every-request-in-the-service-that-owns-it.md)), so no service calls Identity to check permissions.

Service discovery: each service is reached by its name through platform DNS (Docker Compose DNS locally, Kubernetes Service DNS in production), so no separate registry service is needed.

## Context map

Each service is one bounded context: the boundary inside which one model, and one meaning of each term, applies. The context map shows how the contexts depend on each other. The **upstream** context (U) supplies a model; the **downstream** context (D) adapts to it. Page 3 of `microservice-architecture.drawio` draws the same map.

| Downstream (D) | Upstream (U) | Relationship | Through | Who translates |
|---|---|---|---|---|
| Trust & Safety | Identity | Customer/Supplier | `PhoneVerified`; gRPC `lockAccount()` | Trust & Safety turns its suspension into Identity's lock |
| Buddy Discovery | Profile | Customer/Supplier | `ProfileUpdated` | Profile publishes the public card, with hidden fields as `null`; Buddy Discovery builds a Candidate from it |
| Buddy Discovery | Trust & Safety | Open host service, published language | `RunnerVerified`, `VerificationRejected`, `AccountSuspended` | Nobody: consumers keep the flags as published |
| Run Scheduling | Trust & Safety | Open host service, published language | `RunnerVerified`, `VerificationRejected`, `AccountSuspended` | Nobody: consumers keep the flags as published |
| Buddy Discovery | Reputation | Customer/Supplier | `RatingUpdated` | Buddy Discovery stores the aggregate on the Candidate |
| Run Scheduling | Run Conditions | Customer/Supplier | gRPC `getRunConditions()`, `suggestAlternativeSlots()` | Run Scheduling turns a run (meeting point, start time, distance × target pace) into a place and a time window |
| Reputation | Run Scheduling | Customer/Supplier | `RunEnded`, `RunCancelled` | Reputation keeps the participant list as it was when the run ended |
| Notification | Identity, Buddy Discovery, Run Scheduling, Reputation | Conformist | The 9 events Notification consumes | Notification takes each event as published and maps it to a notification category |
| Run Conditions | Weather & Air-Quality Provider | Anticorruption layer | Weather Adapter | The adapter turns provider responses into forecast and air-quality values; no provider type gets past it |
| Identity | SMS/OTP Provider, Email Provider | Anticorruption layer | SMS/OTP Adapter, Email Adapter | The adapters |
| Notification | Email Provider | Anticorruption layer | Email Adapter | The adapter |
| Web App | Map Provider | Anticorruption layer | Leaflet wrapper ([ADR-0011](../adr/0011-render-maps-with-leaflet-and-openstreetmap-tiles.md)) | The wrapper |

- **Customer/Supplier:** the upstream plans its API or events around the downstream's needs, and the downstream's contract tests run in the upstream's pipeline. **Open host service, published language:** one documented protocol for several consumers. **Conformist:** the downstream adopts the upstream's model as it is. **Anticorruption layer:** the downstream translates the upstream's model into its own at the edge.
- **Dependencies point one way.** From the most upstream: Identity, Profile, and Run Conditions; then Trust & Safety; then Run Scheduling; then Reputation; then Buddy Discovery; Notification only consumes. No context depends on a context downstream of it, so there are no cycles.
- **Every event schema is a published language:** documented, versioned, and kept separate from the publisher's internal model, so a publisher can change its model without breaking its consumers.
- **The Web App composes screens across contexts**, for example the public profile in BUC-1 step 2. That is composition in the user interface, not a dependency between contexts.

## Ubiquitous language

Each term has exactly one meaning inside the context that owns it. Terms that sound alike in different contexts are deliberately different words, so code, events, and conversation never mix them up.

| Term | Context | Meaning |
|---|---|---|
| Runner ID | All | The account ID that Identity issues at registration. Every context refers to a runner by this ID and nothing else. |
| Account | Identity | Credentials (email, phone number, password hash) and sessions. |
| Phone verified | Identity | The runner proved they hold the phone number by SMS OTP. A fact, not a badge. |
| Locked | Identity | The account cannot log in or refresh a token. Only Trust & Safety locks an account. |
| Verified badge | Trust & Safety | Granted when the phone is verified; removed if an admin rejects the verification. A "verified runner" has the badge. |
| Report | Trust & Safety | A runner's complaint about another runner or a written review, waiting in the admin queue. |
| Suspension | Trust & Safety | An admin's decision, with a reason: the runner disappears from discovery, cannot receive buddy requests, and the account is locked. |
| Profile | Profile | What a runner says about themself: pace range, usual distance, experience level, area, availability, gear. |
| Public card | Profile | The part of a profile other runners may see, with the area coarsened to a 1-2 km cell. |
| Candidate | Buddy Discovery | Another runner as a searcher sees them: public card, area cell, verified and suspended flags, aggregate rating. |
| Buddy request | Buddy Discovery | A request with a short note, pending until it is accepted, declined, or expires. |
| Buddy | Buddy Discovery | A runner whose buddy request was accepted, in either direction. |
| Cooling-off period | Buddy Discovery | After a decline or expiry, the time during which the requester cannot ask the same candidate again. |
| Run | Run Scheduling | A scheduled group run: meeting point, start time, distance, target pace, participant limit, public or invite-only. |
| Expected end time | Run Scheduling | Start time plus distance × target pace. |
| Participant | Run Scheduling | A runner on a run's participant list. |
| Late cancellation | Run Scheduling | A cancellation inside the late-cancellation window before the start time. |
| Run conditions | Run Conditions | The forecast, PM2.5, and run-suitability level for a place and time. |
| Alternative slot | Run Conditions | A time slot with better conditions than the one the organizer chose (UC-10). |
| Confirmed participant | Reputation | A participant who confirmed attendance within the confirmation window. |
| Completed run | Reputation | A run with at least two confirmed participants. Only Reputation knows whether a run was completed; Run Scheduling never does. |
| Aggregate rating | Reputation | Computed from a runner's ratings, and hidden until three ratings exist. |
| Reliability | Reputation | The late cancellations counted against an organizer. |
| Run record | Reputation | The distance and duration a participant logs for a completed run. A runner's run history is the list of their run records. |
| Notification | Notification | A message about something that happened in another context, in a category the runner can turn off, except run changes and cancellations. One-time passwords and reset links are not notifications. |

Run records live in Reputation because a participant can log one only for a run that Reputation marked completed (FR-8.1).

## Aggregates and invariants

An aggregate is a cluster of data that one transaction keeps consistent; code outside it reaches it only through its root. Each invariant below is enforced inside one aggregate, so no rule needs a transaction across services.

| Service | Aggregate (root) | Invariants it enforces |
|---|---|---|
| Buddy Discovery | Buddy Connection, one per pair of runners: their requests, buddy status, and blocks | At most one pending request per pair; no new request during the cooling-off period (FR-4.4); no request between runners where either blocked the other (FR-4.5) |
| Buddy Discovery | Candidate | A read model rebuilt from events; it never holds an exact location or a field Profile did not publish (NFR-4.1, FR-2.8) |
| Run Conditions | None: cache entries are values keyed by area cell and time bucket | A reading is fresh for a short time, then served only as stale, with a warning (NFR-3.2) |
| Run Scheduling | Run, with its participants, invitations, and attached forecast | Participants never exceed the limit (FR-6.5); leaving only before the start (FR-6.5); an invite-only run is visible only to invitees (FR-6.4); only the organizer edits or cancels ([ADR-0014](../adr/0014-authorize-every-request-in-the-service-that-owns-it.md)) |
| Profile | Runner Profile, with its visibility settings | Only public fields and the coarsened area leave it (FR-2.8, NFR-4.1) |
| Reputation | Run Attendance, one per run: confirmations and the ratings given for that run | Only listed participants confirm, and only inside the window (FR-7.1); two confirmations complete the run (FR-7.2); only confirmed participants rate (FR-7.3) |
| Reputation | Runner Reputation, one per runner | The aggregate rating stays hidden until three ratings exist (FR-7.4); reliability counts late cancellations (FR-6.7) |
| Reputation | Run Record | Only for a run this service marked completed (FR-8.1) |
| Trust & Safety | Runner Trust, one per runner: badge, verification review, suspension | The badge exists only after the phone was verified; a rejection removes it (FR-3.2, FR-3.4) |
| Trust & Safety | Report | Names exactly one runner or one review, by ID (FR-4.6) |
| Identity | Account, with its one-time passwords and refresh tokens | A one-time password expires and allows at most three attempts (FR-3.1); a refresh token works once (NFR-5.4); a locked account cannot log in or refresh |
| Notification | Recipient: contact address and preferences | Run changes and cancellations are always sent (FR-9.2) |

Registering each email address and phone number only once (FR-1.1) is a unique index across accounts in Identity DB, not a rule inside one aggregate.

**Rules that cross services, and so lag behind** (eventually consistent):
- `createRun()` accepts only verified organizers, and `sendBuddyRequest()` refuses suspended candidates, using copies built from Trust & Safety events. Until the event arrives, a runner who just lost the badge can still create a run, and a runner who was just suspended can still receive a request.
- Search results show a profile change, a new badge, or a new rating only after the event arrives ([ADR-0015](../adr/0015-give-each-service-its-own-postgresql-database.md)).
- A suspended runner's access token keeps working for up to 15 minutes ([ADR-0013](../adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md)).

## Kafka topics and failure handling

- **One topic per publishing service:** `identity.events`, `safety.events`, `profile.events`, `buddy.events`, `run.events`, `reputation.events`. Events are keyed by the runner or run they are about, so events for one runner or one run stay in order.
- **Transactional outbox.** A service writes each event to an outbox table in its own database, in the same transaction as its own change, and a relay publishes it to Kafka. A change is therefore never saved without its event, or the other way round.
- **Idempotent consumers, limited retries, dead-letter topics** ([ADR-0006](../adr/0006-service-collaboration-and-provider-isolation.md)). A message that keeps failing goes to a dead-letter topic instead of blocking the others.
- **A Notification outage only delays messages.** Nothing waits for Notification, so scheduling and matching keep working (NFR-3.3), and the messages are sent when it recovers.
- **No secrets on Kafka.** One-time passwords and password-reset links never pass through the broker: Identity sends them through its own adapters.
- **Both synchronous service-to-service calls (gRPC) have a deadline and a circuit breaker.**
  - When the weather provider is down, Run Conditions answers from its Redis cache with a staleness warning (NFR-3.2).
  - When Run Conditions itself does not answer, `createRun()` still creates the run with the forecast "pending", and `recheckForecasts()` attaches it later (BUC-2 3b).
  - When Identity does not answer `lockAccount()`, the suspension still takes effect in Buddy Discovery and Run Scheduling through `AccountSuspended`. Trust & Safety keeps retrying the lock, and the runner's access token expires within 15 minutes anyway ([ADR-0013](../adr/0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md)).

## Use case walkthroughs

Actor → operation → owning service → collaborators → where the data is stored.

**BUC-1 Find a Compatible Running Buddy (Runner)**
1. Runner → `searchRunners()` → Buddy Discovery. It reads only its own candidates (Discovery DB) and calls no other service at search time (NFR-1.1).
2. Runner opens a candidate → `getPublicProfile()` (Profile), `getVerificationStatus()` (Trust & Safety), `getRatingSummary()` (Reputation). The Web App composes the page through the gateway.
3. Runner → `sendBuddyRequest()` → Buddy Discovery, which checks the Buddy Connection for that pair (pending request, cooling-off, block), stores the request in Discovery DB, and publishes `BuddyRequestSent`; Notification consumes it and emails the candidate through the Email Provider.
4. Candidate (Runner) → `getBuddyRequests()`, then `respondToBuddyRequest()` → Buddy Discovery, which publishes `BuddyRequestAnswered`; Notification tells the requester.
5. Safety: `blockRunner()` → Buddy Discovery; `reportRunner()` → Trust & Safety (admin queue in Trust & Safety DB).
6. The candidates stay current because Buddy Discovery consumes `ProfileUpdated` (Profile), `RunnerVerified`, `VerificationRejected`, and `AccountSuspended` (Trust & Safety), and `RatingUpdated` (Reputation).

**BUC-2 Schedule a Run with Weather Check and Pinned Meeting Point (Organizer)**
1. Organizer's Web App → Map Provider for map tiles, to pin the meeting point.
2. For an invite-only run, Organizer → `listBuddies()` → Buddy Discovery, to pick the accepted buddies to invite.
3. Organizer → `previewRunConditions()` → Run Scheduling → Run Conditions `getRunConditions()` over gRPC → Weather & Air-Quality Provider, or the Redis cache. If suitability is poor, Run Scheduling also calls Run Conditions `suggestAlternativeSlots()`.
4. Organizer confirms → `createRun()` → Run Scheduling (checks its verified copy, kept current from Trust & Safety events) → Run Conditions `getRunConditions()` (forecast attached to the run). The run, its participants, and its forecast are stored in Run DB, and `RunScheduled` is published; Notification consumes it and invites the invitees.
5. Participants → `browseUpcomingRuns()`, `joinRun()` → Run Scheduling (participant limit enforced in one Run DB transaction).
6. Timers in Run Scheduling: `sendRunReminders()` publishes `ReminderDue`; `recheckForecasts()` calls Run Conditions and publishes `ForecastChanged` when the suitability level changes. Notification consumes both.
7. Organizer → `cancelRun()` → Run Scheduling, which flags a cancellation inside the late-cancellation window as `late` and publishes `RunCancelled`. Notification tells the participants, and Reputation counts a late cancellation against the organizer's reliability.

**BUC-3 Confirm Attendance, Rate Buddies, and Log Running Record (Participant)**
1. Run Scheduling `endRun()` [scheduled] publishes `RunEnded` with the participant list; Reputation consumes it, opens attendance, and publishes `AttendanceOpened`; Notification consumes that and prompts the participants.
2. Participant → `confirmAttendance()` → Reputation. Two confirmations mark the run completed (Reputation DB).
3. Participant → `rateParticipant()` → Reputation, which enforces "confirmed participants only" by itself, then publishes `RatingUpdated` once three ratings exist; Buddy Discovery consumes it.
4. Participant → `logRunRecord()` → Reputation (run history in Reputation DB).
5. Participant → `getReviews()` → Reputation, then `reportReview()` → Trust & Safety (admin queue).

**Registration and moderation (UC-01, UC-05, UC-12, UC-13)**
1. Visitor → `register()` → Identity (account in Identity DB), which publishes `ContactChanged`; Notification stores the email address.
2. Visitor → `sendPhoneOtp()`, `verifyPhoneOtp()` → Identity → SMS/OTP Provider; Identity publishes `PhoneVerified`. If SMS fails, Identity emails a one-time password through its own Email Adapter, and `verifyEmailOtp()` confirms the email address only.
3. Trust & Safety consumes `PhoneVerified`, grants the verified badge, adds the verification to the admin queue, and publishes `RunnerVerified`; Buddy Discovery and Run Scheduling update their copies.
4. Admin → `getModerationQueue()`, then `reviewVerification()` → Trust & Safety; a rejection publishes `VerificationRejected`.
5. Admin → `suspendAccount()` → Trust & Safety, which records the decision in the audit log, publishes `AccountSuspended`, and calls Identity `lockAccount()` over gRPC.

## Data ownership

Each service's data is private and reached only through its API or its events. Seven services each own a relational PostgreSQL 16 logical database, with PostGIS where noted, and the Run Conditions Service owns a Redis store, which is NoSQL (key-value) ([ADR-0015](../adr/0015-give-each-service-its-own-postgresql-database.md), [ADR-0017](../adr/0017-cache-weather-and-air-quality-in-redis.md)). This gives the project one relational and one NoSQL database type.

| Data store | Owner | Key data |
|---|---|---|
| Discovery DB (PostgreSQL + PostGIS) | Buddy Discovery Service | Candidates (public card, 1-2 km area cell, verified and suspended flags, aggregate rating), buddy connections (requests, buddies, cooling-off), blocks |
| Weather cache (Redis, NoSQL) | Run Conditions Service | Forecast and PM2.5 by area cell and time bucket, with fetch time for fresh / stale |
| Run DB (PostgreSQL + PostGIS) | Run Scheduling Service | Runs, meeting points, invitations, participants, attached forecasts, cancellation records, runner status copy (verified, suspended) |
| Profile DB (PostgreSQL) | Profile Service | Profiles, exact preferred area (never shared), field visibility |
| Reputation DB (PostgreSQL) | Reputation Service | Attendance windows, confirmations, ratings, reviews, aggregates, organizer reliability, run records |
| Trust & Safety DB (PostgreSQL) | Trust & Safety Service | Verified badges, verification review queue, reports, suspensions, admin audit log |
| Identity DB (PostgreSQL) | Identity Service | Accounts, password hashes, refresh tokens, phone and email one-time passwords, password-reset tokens, account locks |
| Notification DB (PostgreSQL) | Notification Service | Contact email addresses, per-category preferences, in-app notifications |
| Event Broker (Kafka) | Shared infrastructure | The six event topics and their dead-letter topics; no service reads another service's database |

Every service that publishes events keeps its own outbox table (see Kafka topics and failure handling).

## Choices beyond the course brief

The course brief lets version 1 use REST only, one database type, and no separate authentication service. This design goes further in five places, each for a stated reason:
- **A message broker (Kafka).** The brief advises against adding a broker just to make the architecture look complex. Here it carries a requirement: a Notification outage must not block scheduling or matching (NFR-3.3), and [ADR-0006](../adr/0006-service-collaboration-and-provider-isolation.md) already publishes facts as events. With REST calls instead, Run Scheduling and Buddy Discovery would wait on Notification. Phase 1 uses non-blocking REST calls with short timeouts instead, accepting that an event can be lost ([ADR-0019](../adr/0019-call-services-over-rest-until-the-phase-2-event-broker.md)); Kafka arrives in phase 2.
- **gRPC for the two internal calls** (Run Scheduling → Run Conditions, Trust & Safety → Identity). The syllabus requires at least one gRPC service, and neither operation has a public route.
- **Two database types** (PostgreSQL and Redis). Cached forecasts are short-lived key-value data that do not belong in a relational store ([ADR-0017](../adr/0017-cache-weather-and-air-quality-in-redis.md)).
- **A separate Identity Service.** The brief says an authentication service need not be split out without a clear architectural reason. The reason here: verifying a phone number by SMS OTP is a product requirement, because the verified badge is the safety signal runners asked for ([ADR-0012](../adr/0012-verify-phone-ownership-with-sms-one-time-passwords.md)), and it calls an external SMS provider through an adapter. As the brief asks, Identity is not wired to every service: the API Gateway authenticates each request, and each service authorizes by its own business rules.
- **A context map (page 3).** The brief asks only for the table and the architecture diagram. Page 3 is extra; its lines are relationships between models, not calls, so it has no arrows and does not change the arrow rule of page 1.

## Changes from Deliverable 1

ADR-001 below means Deliverable 1's microservices ADR, now recorded as [ADR-0008](../adr/0008-use-a-microservices-architecture.md), with the service list in [ADR-0005](../adr/0005-service-boundaries-and-data-ownership.md).

- **Eight services, not seven: Trust & Safety is split out of Identity.** ADR-001's Identity owned both account security and the safety model (verification review, reports, suspensions, audit log). Account security is generic, while the safety model is specific to this app. The two also serve different people (every runner versus the Admin) and use different words. ADR-001 itself builds one service per bounded context. Split, Identity can stay thin, or be replaced by an off-the-shelf identity product, without taking moderation with it.
- **The Weather Service is renamed Run Conditions Service.** Its value to the business is the run-suitability indicator and alternative slots (core domain), not the weather data, which it buys from the provider. The name comes from the ADR's own extension point "run conditions shown" (UC-03 step 4).
- **Organizer reliability moves from Run Scheduling to Reputation.** Reliability is a trust signal about a runner, like a rating, and the no-show idea in the ADR's Appendix H would feed it from attendance, which Reputation owns. Run Scheduling still decides whether a cancellation is late, because it knows the start time, and says so in `RunCancelled`.
- **Identity sends its own emails.** The email one-time password and the password-reset link go through an Email Adapter in Identity, next to its SMS/OTP Adapter, instead of an `EmailRequested` event to Notification. They are part of securing an account, not notifications (no category, no preference, FR-9.2), and this keeps one-time codes and reset tokens out of Kafka.
- **`RunnerStatusChanged` is replaced by events that name what happened:** `PhoneVerified` (Identity), and `RunnerVerified`, `VerificationRejected`, `AccountSuspended` (Trust & Safety).
- **Profile applies field visibility once.** `ProfileUpdated` carries a ready-made public card, so Buddy Discovery never applies FR-2.8 a second time.
- **Kafka is the message broker.** Deliverable 1 left the broker product open; [ADR-0006](../adr/0006-service-collaboration-and-provider-isolation.md) names Kafka for phase 2, and phase 1 runs without a broker ([ADR-0019](../adr/0019-call-services-over-rest-until-the-phase-2-event-broker.md)).
- **Discovery is named Buddy Discovery Service.** The course guidance asks for names that say what a service matches or finds; "Discovery" alone does not.
- **The forecast re-check job runs in Run Scheduling, not in Run Conditions.** The attached forecast and the participant list both live in Run Scheduling ([ADR-0017](../adr/0017-cache-weather-and-air-quality-in-redis.md)), so Run Scheduling re-checks by calling Run Conditions and publishes `ForecastChanged`.
- **Service-to-service requests use gRPC.** ADR-001 makes synchronous calls REST/JSON; the two internal calls use gRPC instead (see Choices beyond the course brief).
- **Only Run Scheduling calls Run Conditions.** The forecast window comes from the run's own fields (start time plus distance × target pace), and the preview must match the forecast that `createRun()` attaches, so Run Scheduling makes both calls and the API Gateway has no route to Run Conditions.
- **Kubernetes is assumed for production.** Service discovery relies on Kubernetes Service DNS, but ADR-001 leaves the production platform to a later ADR, which should record this choice.
- **Deferred to a later progress:** deleting an account's data across all services within 30 days (FR-1.4, NFR-4.2) and data export (FR-1.5).

## Open questions

Decisions the ADR does not settle, surfaced while modeling:
- Does hiding a profile field, such as pace, also remove it from matching, or only from display? Profile decides either way, in one place. **Answered by [ADR-0028](../adr/0028-rabbitmq-for-profile-updated.md):** from matching too. `ProfileUpdated` sends a hidden field as `null`; a runner is left out of searches that filter on a field they hid, and included when that filter is off.
- Can a runner whose verification an admin rejected get the badge back by verifying again by SMS OTP?
- When an admin upholds a report about a written review, is the review hidden? FR-10.2 only suspends accounts, and Reputation has no operation to hide a review.
- How long may a Trust & Safety decision take to reach Buddy Discovery and Run Scheduling? For that long, a just-suspended runner can still be found and sent a request.
