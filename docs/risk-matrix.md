# Risk matrix

Architectural risks found while building, deploying and load-testing the system (Deliverable 5, 31 Oct 2026). Each risk is backed by a measurement or by code, and its score is **likelihood × impact** on a 1–5 scale.

| Score | Level | Response |
| --- | --- | --- |
| 20–25 | Critical | Must fix before claiming the requirement |
| 10–16 | High | Plan a fix; mitigate now |
| 5–9 | Medium | Mitigate or accept with monitoring |
| 1–4 | Low | Accept |

## Matrix

| Likelihood ↓ / Impact → | 1 Negligible | 2 Minor | 3 Moderate | 4 Major | 5 Severe |
| --- | :---: | :---: | :---: | :---: | :---: |
| **5 Almost certain** | | | | **R1** | |
| **4 Likely** | | | | **R2** | |
| **3 Possible** | | R4, R5 | | | |
| **2 Unlikely** | | | | | R3, R6 |
| **1 Rare** | | | | | |

## Risks

| ID | Risk | Evidence | L | I | Score | Mitigation | Status |
| --- | --- | --- | :---: | :---: | :---: | --- | --- |
| R1 | **The runs list and runner search cannot serve 500 concurrent users** (NFR-1.1, NFR-1.2, NFR-2.1). `GET /runs` returns every upcoming run with one participant query per run; runner search loads every profile and filters in Python. | At ramp to 500 VUs: nearby-runs p95 **11.5 s** against a 1 s target; Scheduling at its 1-CPU limit; adding replicas saturated the 6-vCPU node instead of adding capacity (`load-tests/README.md`). | 5 | 4 | **20** | Proximity query with PostGIS and pagination ([0016](adr/0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md)); one query for participants; connection pooling in the Python services. | Open, fix planned |
| R2 | **A burst of logins exhausts Identity's database pool.** Login holds a pooled connection inside a `FOR UPDATE` transaction during the ~240 ms bcrypt check; the pool has 10 connections and a 2 s acquire timeout. | 4 of 55 logins returned 500 when 50 VUs logged in at the same instant (baseline run `20261031T042116Z`). `team-identity-service/controllers/session.js`. | 4 | 4 | **16** | Compare the hash outside the transaction; size the pool to the CPU budget; rate-limit login at the gateway. | Open |
| R3 | **One shared HS256 signing secret.** Every service can verify and also mint tokens, so one compromised service can impersonate any user to all others (NFR-5.3). | [0022](adr/0022-service-authorization-for-the-local-stack.md) limits this trust model to the local demo. | 2 | 5 | **10** | Asymmetric signing (RS256/EdDSA) with Identity publishing its public keys; services hold verification keys only. Secret kept in a Kubernetes Secret, never committed. | Accepted for the demo |
| R6 | **Single node, single database instance.** One VM runs every service, PostgreSQL, Redis and RabbitMQ; losing it stops the whole system (NFR-3.1). | k3s single node in Colima VM `cra-k8s`; one PostgreSQL StatefulSet. | 2 | 5 | **10** | Out of scope for the course. In production: multi-node cluster, managed PostgreSQL with replicas, daily backups (NFR-6.6). Data already survives pod restarts on persistent volumes. | Accepted |
| R4 | **An in-flight gRPC call fails when a Discovery pod is deleted.** The gateway's HTTP/2 connection to that pod breaks. | 749 of 750 calls succeeded; 1 returned 503 with a ~2 s stall (`k8s/README.md`, gRPC check). HTTP services lost no requests in the same test. | 3 | 2 | **6** | Gateway retry for idempotent reads (proposed); graceful shutdown with `preStop` already in place. | Open, low priority |
| R5 | **Weather provider outage or forecast horizon.** Run Conditions depends on Open-Meteo/CAMS; slots beyond about 4.5 days have no air-quality forecast. | Cache-miss latency 1.6 s per fetch (cold-cache run); horizon bug found in run `20261031T042044Z`. | 3 | 2 | **6** | Redis cache (15 min TTL) plus a 6-hour stale fallback with a staleness flag (NFR-3.2); out-of-horizon slots fail visibly with 503 rather than inventing data; scheduling stays usable. | Mitigated |

Evidence is summarized in the [`running-buddy-integration`](https://github.com/Chula-Com-Sci-2026/running-buddy-integration) READMEs (`load-tests/README.md`, `k8s/README.md`); run IDs name the raw result folders kept on the test machine.
