# 18. Build phase 1 with six services

Date: 2026-10-03

## Status

Accepted

Amends [5. Service boundaries and data ownership](0005-service-boundaries-and-data-ownership.md)

## Context

The target architecture has eight services, an API gateway, and a Kafka event broker ([5. Service boundaries and data ownership](0005-service-boundaries-and-data-ownership.md), [6. Service collaboration and provider isolation](0006-service-collaboration-and-provider-isolation.md)). Review feedback on Deliverable 2 (as of 2026-10) found the table and diagram complete, but the scope large in services and infrastructure. It recommends finishing buddy search, run creation, and the weather check first, and treating Reputation, Trust & Safety, multi-channel notification, and a full event broker as follow-on work. Feedback on Deliverable 1 also suggests leaving the admin features out for now. The team is five people with one semester.

The two business use cases that phase 1 must complete are UC-02 Find a Compatible Running Buddy and UC-03 Schedule a Run with Weather Check. They need Profile, Buddy Discovery, Run Scheduling, and Run Conditions. They also lean on sign-in, the verified badge (the safety signal from interviews), and telling runners about requests and run changes.

Options considered:

- **All eight services in phase 1.** Delivers every use case, but ignores the feedback and risks finishing nothing end to end.
- **Six services: the four core ones plus a thin Identity and an in-app Notification.** Completes UC-02 and UC-03 with sign-in and phone verification, and defers Reputation, Trust & Safety with its admin console, and the email channel.
- **Only the four core services.** Smallest build, with sign-in from an off-the-shelf identity provider and no Notification. Drops phone verification, the safety signal runners asked for, and means runners only see new requests and run changes when they look.

## Decision

We will build phase 1 with six services behind the API gateway: Identity (sign-in, tokens, phone and email one-time passwords), Profile, Buddy Discovery, Run Scheduling, Run Conditions, and Notification (in-app only).

We will build in phase 2: Reputation (UC-04: attendance, ratings, run records, organizer reliability), Trust & Safety with the admin console (UC-11 to UC-13: badge review, reports, suspensions, audit log), the email notification channel, and the Kafka event broker.

Phase 1 substitutes for the deferred services as follows:

- The phone-verified fact from Identity is the verified badge; Identity sends it straight to Buddy Discovery and Run Scheduling.
- Buddy Discovery ranks without ratings.
- Run Scheduling still flags late cancellations, but nothing scores reliability yet, and it does not publish run-ended events.
- Runners can block each other, but cannot report anyone.

The full table is in [the architecture overview](../architecture/README.md#phase-1-scope).

## Consequences

- The team can finish buddy search, run scheduling, and the weather check end to end before adding more services.
- The target design in ADR-0005 stays valid; phase 2 adds services without redrawing the boundaries of phase 1 ones.
- UC-04 Confirm Attendance, Rate Buddies, and Log Running Record is not delivered in phase 1.
- Safety rests on phone verification and blocking alone. Until Trust & Safety exists, removing a bad actor means a manual operation on Identity's data.
- Two of the three services in [7. Backend stack for discovery, reputation, and trust services](0007-backend-language-and-framework.md), Reputation and Trust & Safety, wait until phase 2.
- In phase 2, Trust & Safety takes over the badge from Identity's phone-verified fact, so Buddy Discovery and Run Scheduling must switch their badge source then.

Source: review feedback on Deliverables 1 and 2.
