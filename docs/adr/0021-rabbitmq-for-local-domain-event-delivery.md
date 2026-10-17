# 21. RabbitMQ for local domain event delivery

Date: 2026-10-16

## Status

Accepted for the local Deliverable 4 implementation, under the project owner's delegated approval. Partially supersedes [ADR-0019](0019-call-services-over-rest-until-the-phase-2-event-broker.md): its deferral of broker delivery for buddy notifications, for the local Deliverable 4 deployment only. Also supersedes the Kafka candidate in proposed ADR-0006 for this local flow; synchronous REST collaborations remain.

## Context

A saved buddy request must produce a recipient notification without requiring Notification to be available during the request. Deliverable 4 needs observable broker collaboration. Kafka offers retained ordered streams and replay but adds partition/offset operations unnecessary for this notification flow. RabbitMQ offers durable queues, acknowledgements and dead-letter routing suited to work delivery, but does not provide Kafka-style retained stream replay through ordinary queues.

## Decision

Use RabbitMQ with a durable topic exchange, persistent event messages and a durable Notification queue. Discovery writes events in a transactional PostgreSQL outbox. Its worker publishes through the RabbitMQ management HTTP API and marks an event published only after successful routing. Notification acknowledges after its database commit and deduplicates by event ID and recipient. Invalid events go to a dead-letter queue. Transient database/broker failures retry with backoff.

## Consequences

Delivery is at least once; duplicates are expected and handled. Broker management credentials are confined to local infrastructure. Unrouted events stay in the outbox for retry. Outbox retention supports diagnosis/republication. A future Kafka implementation needs a transport adapter and compatible event contracts; a URL change alone is insufficient. No synchronous business operation depends on notification delivery.
