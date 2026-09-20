# 6. Service collaboration and provider isolation

Date: 2026-09-20

## Status

Proposed

## Context

Weather-provider failure must leave scheduling usable with a visibly stale cached forecast (NFR-3.2), and failures in weather or notification must not cascade into profile, matching, or scheduling (NFR-3.3). External providers must be replaceable behind adapters (NFR-7.2). Deliverable 2 lists direct calls for run conditions and account locking, plus broker events for changes that other services consume.

## Decision

Use direct service calls when a user action needs an immediate answer: Run Scheduling calls Run Conditions for checked conditions and alternative slots; Trust & Safety calls Identity to lock a suspended account. Run Conditions alone calls the weather and air-quality providers through adapters and uses its Redis cache. If a provider is unavailable, return a cached forecast with its age and a staleness warning when one exists.

Publish domain events for state changes and downstream reactions:

| Publisher | Events | Main consumers |
| --- | --- | --- |
| Identity | `PhoneVerified`, `ContactChanged` | Trust & Safety, Notification |
| Profile | `ProfileUpdated` | Buddy Discovery |
| Trust & Safety | `RunnerVerified`, `VerificationRejected`, `AccountSuspended` | Buddy Discovery, Run Scheduling |
| Buddy Discovery | `BuddyRequestSent`, `BuddyRequestAnswered` | Notification |
| Run Scheduling | `RunScheduled`, `RunChanged`, `RunCancelled`, `ReminderDue`, `ForecastChanged`, `RunEnded` | Notification, Reputation |
| Reputation | `AttendanceOpened`, `RatingUpdated` | Notification, Buddy Discovery |

Reputation also consumes `RunCancelled` to record late cancellations. Deliverable 2 names Kafka for Identity and Notification's broker collaboration; use a shared event broker, with Kafka as the documented candidate, subject to team review of the operational cost. Keep provider calls behind adapters. Treat event contracts, delivery guarantees, and retry policy as implementation decisions still to be specified.

## Consequences

- Notifications and read-model updates can proceed asynchronously, reducing coupling to non-critical services, but users may briefly see stale data.
- Event consumers need idempotency and a way to recover missed events. An outbox is a suitable implementation pattern for publishing alongside database changes.
- Synchronous Run Conditions calls need a timeout and cached fallback so a provider outage does not block scheduling. The maximum usable cache age still needs a policy decision.
- Broker operations and event schema versioning add complexity. The draft Run Scheduling API contract is not an implemented event contract.

Source: Deliverable 2, “Microservice Design with Collaborations,” service collaborations; Deliverable 1, NFR-3.2, NFR-3.3, and NFR-7.2.
