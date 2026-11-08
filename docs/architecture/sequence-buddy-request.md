# Sequence: send a buddy request and notify the recipient

Representative business scenario for UC-02 Find a Compatible Running Buddy. It crosses the gateway, the gRPC service, the database, the message broker and a second service. Checked against `buddy-discovery-service` (outbox model and publisher) and `notification-service/worker.py`.

![Buddy request sequence](images/sequence-buddy-request.png)

```mermaid
sequenceDiagram
    autonumber
    actor Ray as Ray (sender)
    participant G as API Gateway
    participant D as Buddy Discovery (gRPC)
    participant DDB as Discovery PostgreSQL
    participant O as Discovery outbox publisher
    participant Q as RabbitMQ (cra.events)
    participant W as Notification worker
    participant NDB as Notification PostgreSQL
    participant N as Notification API
    actor Mali as Mali (recipient)

    Ray->>G: POST buddy request (JWT, recipient id)
    G->>D: SendBuddyRequest over gRPC, JWT in metadata
    D->>D: Validate token, check sender is not recipient
    D->>DDB: One transaction: insert buddy_request + outbox_event
    DDB-->>D: Commit
    D-->>G: Request (status pending)
    G-->>Ray: 201 pending

    loop Every 2 s (bin/outbox-worker)
        O->>DDB: Select unpublished outbox events
        O->>Q: Publish BuddyRequestSent (persistent, routing key = event type)
        Q-->>O: routed = true (otherwise retry next loop)
        O->>DDB: Mark event published
    end

    Q->>W: Deliver to queue notifications.buddy.v1
    W->>NDB: INSERT notification ON CONFLICT (event_id, recipient_id) DO NOTHING
    NDB-->>W: Commit
    W->>Q: ack (invalid event: nack to dead-letter queue)

    Mali->>G: GET notifications (JWT)
    G->>N: REST list for the authenticated recipient
    N->>NDB: Select own notifications
    NDB-->>N: Rows
    N-->>G: Notifications
    G-->>Mali: Buddy request from Ray
```

## Why it is built this way

- **Transactional outbox (steps 4–5).** The request and its event commit together, so a broker outage cannot lose the event or leave an event for a request that never existed.
- **The sender does not wait for the broker.** Ray gets an answer as soon as the database commits. If RabbitMQ or Notification is down, the request still succeeds and the notification arrives later (NFR-3.3).
- **At-least-once delivery, idempotent consumer.** The publisher may publish an event twice after a crash; the unique `(event_id, recipient_id)` key stops a duplicate notification. The worker acknowledges only after its commit.
- **Poison messages.** An invalid event is rejected without requeue and goes to the `notifications.dead` queue for inspection.
- **Authorization at every hop.** The gateway forwards Ray's token; Discovery and Notification each validate it and apply their own rules ([0014](../adr/0014-authorize-every-request-in-the-service-that-owns-it.md)).
