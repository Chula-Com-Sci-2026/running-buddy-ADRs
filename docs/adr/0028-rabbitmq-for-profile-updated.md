# 28. RabbitMQ for ProfileUpdated

Date: 2026-11-01

## Status

Accepted

Partially supersedes [ADR-0019](0019-call-services-over-rest-until-the-phase-2-event-broker.md) for `ProfileUpdated`: Profile delivers it through RabbitMQ instead of REST calls in the local implementation, as [ADR-0021](0021-rabbitmq-for-local-domain-event-delivery.md) did for buddy events. The target Kafka design in ADR-0006 is unchanged.

## Context

[ADR-0005](0005-service-boundaries-and-data-ownership.md) and [ADR-0016](0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md) have Buddy Discovery search its own copy of profile data, kept current by `ProfileUpdated` (ADR-0019's event table). ADR-0019 delivers events as HTTP POSTs to endpoints that accept only service tokens, and a subscriber that stays down past the retries loses the event. ADR-0021 has since put RabbitMQ and a transactional outbox in the local stack, but its queues offer no replay, so a new or wiped consumer needs another way to load existing profiles. The flow must respect NFR-3.3 and FR-2.8, and the architecture notes leave open whether hiding a field also removes it from matching.

## Decision

After every change to a profile, Profile publishes `ProfileUpdated` to the durable topic exchange `cra.events`, routing key `ProfileUpdated`, in `events.md`'s envelope. The payload is a full snapshot: the runner's public card with every key present, the point only as its cell ([ADR-0030](0030-area-cells-on-a-0-01-degree-grid.md)), and a `revision` that is 1 on creation and grows by 1 with each change. A consumer replaces its copy with any event whose revision is at least the one it holds.

Hidden means hidden from matching too: a hidden field is sent as `null`, so a runner is left out of searches that filter on a field they hid, and included when that filter is off. This answers the architecture notes' open question.

The change and its event commit in one PostgreSQL transaction. A relay inside Profile sends unsent events in order, and marks one sent only once the broker has confirmed it and routed it to a queue. Profile connects as its own broker user, `profile`, which may configure and write `cra.events` and nothing else. `npm run events:republish` sends every profile's current card again, under its current revision, to fill a new or wiped consumer.

The contract is `profile-events.schema.json`, in the format of `buddy-events.schema.json`, and a section in `events.md`, both in `running-buddy-contracts`.

## Consequences

- A save never waits on the broker: a RabbitMQ outage delays delivery and nothing else (NFR-3.3).
- Delivery is at least once, so an event can arrive more than once.
- Until a queue is bound for `ProfileUpdated`, events wait in Profile's outbox, then go out in order.
- No service tokens are needed, and a Kafka producer in phase 2 is one more adapter behind the same publisher interface.
- Revisions restart at 1 when Profile's database is reset, so a consumer's copy must be reset with it.
- Hiding has a cost: a runner who hides a field drops out of every search that filters on it, and a hidden pace, experience or availability can't be used for ranking.
- Sent events stay in the outbox, and nothing cleans them up yet.

Implementation evidence: `docs/profile-events.schema.json` and `compose.local-integration.yaml` in the `profile-service` repository.
