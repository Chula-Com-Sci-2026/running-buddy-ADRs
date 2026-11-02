# 19. Call services over REST until the phase 2 event broker

Date: 2026-10-03

## Status

Accepted

Amends [6. Service collaboration and provider isolation](0006-service-collaboration-and-provider-isolation.md)

Partially superseded for local buddy-notification broker delivery by [ADR-0021](0021-rabbitmq-for-local-domain-event-delivery.md). Other synchronous REST collaborations remain.

Partially superseded for `ProfileUpdated` by [ADR-0028](0028-rabbitmq-for-profile-updated.md).

## Context

In the target design every fact that other services react to travels through Kafka ([6. Service collaboration and provider isolation](0006-service-collaboration-and-provider-isolation.md)). Review feedback (as of 2026-10) suggests leaving a full event broker as follow-on work, and the course brief allows a first version that uses REST only; the final deliverable must still show REST, gRPC, and asynchronous messaging. Phase 1 has six services ([18. Build phase 1 with six services](0018-build-phase-1-with-six-services.md)) and only these facts cross a service boundary:

| Publisher | Event | Subscribers |
| --- | --- | --- |
| Profile | `ProfileUpdated` | Buddy Discovery |
| Identity | `PhoneVerified` | Buddy Discovery, Run Scheduling |
| Buddy Discovery | `BuddyRequestSent`, `BuddyRequestAnswered` | Notification |
| Run Scheduling | `RunScheduled`, `RunChanged`, `RunCancelled`, `ReminderDue`, `ForecastChanged` | Notification |

A Notification outage must not block scheduling or matching (NFR-3.3).

Options considered:

- **REST now, broker later.** No broker to run in phase 1. The publisher calls each subscriber and must not wait on it; an event can be lost if a subscriber stays down.
- **A minimal Kafka now.** A single broker with only the phase 1 events. No migration later and no lost events, but the team still learns and runs Kafka in phase 1, which is the cost the feedback suggests deferring.
- **Redis Streams now, Kafka later.** Asynchronous from the start using Redis, which the stack already has. A migration of the event layer later and two messaging technologies to document and test.

## Decision

We will deliver phase 1 events by REST. Each publisher sends an event as an HTTP POST to each subscriber's internal events endpoint, with the same event name and payload that will later go on Kafka. Publishing goes through one publisher interface in each service, so phase 2 replaces only the HTTP sender with a Kafka producer.

- The publisher uses a short timeout and a few retries, and never fails the runner's action because delivery failed.
- Subscribers are listed in each publisher's configuration. Their events endpoints are not routed by the gateway and accept only service tokens ([14. Authorize every request in the service that owns it](0014-authorize-every-request-in-the-service-that-owns-it.md)).
- Subscribers handle each event idempotently by event ID.
- The one synchronous call, Run Scheduling to Run Conditions, stays gRPC.

## Consequences

- Phase 1 runs no broker, and event handlers written now carry over to phase 2 unchanged.
- Publishers know their subscribers, a coupling Kafka removes in phase 2.
- A subscriber that stays down past the retries loses that event: a runner misses an in-app notification, or a candidate in Buddy Discovery stays stale until the runner's next profile change or verification. NFR-3.3 holds for availability, but delivery is not guaranteed.
- If lost updates matter in testing, add the transactional outbox early; phase 2 needs it for Kafka anyway.
- Event payloads are a published contract from phase 1 and must be versioned, because Kafka consumers will read the same shapes.

Source: review feedback on Deliverable 2; course brief for the first architecture version.
