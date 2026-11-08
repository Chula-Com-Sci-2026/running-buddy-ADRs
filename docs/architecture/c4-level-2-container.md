# C4 Level 2: Containers (as built)

The containers that run in the Deliverable 5 stack, checked against `running-buddy-integration/compose.yaml` and `k8s/apps/`. Arrow convention: **A → B means A starts the call.** Responses are not drawn.

![C4 Level 2 containers](images/c4-level-2-container.png)

```mermaid
flowchart LR
    runner(["<b>Runner</b><br/>[Person]"])

    subgraph system["Can't Run Alone"]
        direction LR
        spa["<b>Web App</b><br/>[React SPA, Vite]"]
        gw["<b>API Gateway / BFF</b><br/>[Python, FastAPI]<br/>Routing, JWT forwarding"]

        subgraph services["Domain services"]
            direction TB
            identity["<b>Identity</b><br/>[Node.js, Express]<br/>Accounts, OTP, sessions"]
            profile["<b>Profile</b><br/>[TypeScript, Hono]<br/>Runner profiles, search"]
            discovery["<b>Buddy Discovery</b><br/>[Rails, gRPC]<br/>Candidates, buddy requests"]
            scheduling["<b>Run Scheduling</b><br/>[Python]<br/>Runs, membership, attendance"]
            conditions["<b>Run Conditions</b><br/>[Python]<br/>Weather and air quality"]
            reputation["<b>Reputation</b><br/>[Rails]<br/>Ratings, running records"]
            notification["<b>Notification</b><br/>[Python, FastAPI]<br/>Inbox, preferences"]
            trust["<b>Trust &amp; Safety</b><br/>[Python]<br/>Reports, badges, suspensions"]
        end

        subgraph async["Asynchronous workers"]
            direction TB
            outbox["<b>Discovery outbox publisher</b>"]
            nworker["<b>Notification worker</b>"]
            tworker["<b>Trust &amp; Safety worker</b><br/>+ outbox publisher"]
            iworker["<b>Identity email worker</b>"]
        end

        mq[("<b>RabbitMQ</b><br/>[Message broker]<br/>topic exchange cra.events")]
        redis[("<b>Redis</b><br/>[NoSQL key-value, AOF]<br/>Weather cache")]
        photos[("<b>Profile photo storage</b><br/>[MinIO, S3 API]")]
        pg[("<b>PostgreSQL 16</b><br/>[RDBMS]<br/>One database and role per service")]
    end

    weather["<b>Open-Meteo / CAMS</b><br/>[External system]"]

    runner --> spa
    spa -->|"REST/JSON + JWT"| gw
    gw -->|"REST"| identity
    gw -->|"REST"| profile
    gw ==>|"gRPC + JWT"| discovery
    gw -->|"REST"| scheduling
    gw -->|"REST"| conditions
    gw -->|"REST"| reputation
    gw -->|"REST"| notification
    gw -->|"REST"| trust

    discovery -->|"REST search"| profile
    scheduling -->|"REST weather check"| conditions
    reputation -->|"REST membership, attendance"| scheduling
    conditions -->|"HTTPS"| weather
    conditions --> redis

    outbox -.->|"publish BuddyRequestSent / Answered"| mq
    mq -.->|"consume"| nworker
    tworker -.->|"publish / consume"| mq
    profile -.->|"publish ProfileUpdated (outbox)"| mq
    profile --> photos
    identity -.->|"email job"| mq
    mq -.->|"identity.email.v1"| iworker

    identity & profile & discovery & scheduling & reputation & notification & trust --> pg
    outbox & nworker & tworker --> pg

    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef container fill:#438dd5,stroke:#2e6295,color:#ffffff
    classDef store fill:#2f6f4f,stroke:#1f4a35,color:#ffffff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#ffffff
    class runner person
    class spa,gw,identity,profile,discovery,scheduling,conditions,reputation,notification,trust,outbox,nworker,tworker,iworker container
    class mq,redis,pg,photos store
    class weather ext
```

Solid arrows are synchronous calls; the thick arrow is the gRPC call; dotted arrows are asynchronous messages through the broker.

## How the course requirements map to containers

| Requirement | Container |
| --- | --- |
| REST service | Reputation, Profile, Run Scheduling, Run Conditions, Notification, Trust &amp; Safety, Identity |
| gRPC service | Buddy Discovery (`buddy_discovery.v1`, port 50051) |
| Message broker service | RabbitMQ: Discovery outbox → Notification worker; Profile publishes `ProfileUpdated`; Trust &amp; Safety; Identity email worker |
| API gateway | API Gateway / BFF (the only container the browser calls) |
| RDBMS | PostgreSQL, one database and owning role per service ([0015](../adr/0015-give-each-service-its-own-postgresql-database.md)) |
| NoSQL | Redis key-value store for weather and air quality, persisted with AOF ([0017](../adr/0017-cache-weather-and-air-quality-in-redis.md)) |
| Service discovery | Platform DNS: Compose service names, Kubernetes Services + CoreDNS (see [`deployment.md`](deployment.md)) |

## Changed since the Deliverable 2 design

[`microservice-architecture.png`](microservice-architecture.png) is the Deliverable 2 **target** design. The build differs in these ways:

- **Buddy Discovery is the gRPC service.** The design gave Discovery a REST API and Run Conditions and Identity gRPC APIs. In the build, the gateway calls Discovery over gRPC, and Conditions and Identity are REST.
- **RabbitMQ, not Kafka.** Domain events go through RabbitMQ with a transactional outbox ([0021](../adr/0021-rabbitmq-for-local-domain-event-delivery.md)).
- **Discovery reads Profile through its API.** Profile already publishes `ProfileUpdated`, but Discovery does not consume it into its own read model yet.
- **Profile and Notification are the team implementations** (since 8 Nov 2026): Profile in TypeScript with photo storage in MinIO, Notification in FastAPI with per-category preferences.
- **No admin console and no email/SMS delivery.** Moderation is an API used by configured moderator accounts; OTPs and emails go to a console adapter.

## Deployment notes

- Docker Compose runs a Python Identity by default; the demo and the Kubernetes deployment use the team's Node.js Identity, selected with `compose.team-identity.yaml`. Both expose the same contract.
- Trust &amp; Safety runs in Compose only; the Kubernetes manifests do not include it yet.
