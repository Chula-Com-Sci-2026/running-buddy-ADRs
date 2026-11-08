# Quality attribute: resilience

The measurable quality attribute we demonstrate: **the system keeps serving requests while service instances fail and are replaced** (NFR-3.3, NFR-3.4, NFR-2.3).

## Quality attribute scenario

| Part | Value |
| --- | --- |
| Source | Infrastructure failure (simulated by an operator) |
| Stimulus | Both running Profile pods are deleted, one after the other |
| Artifact | Profile service behind the API gateway, 2 replicas on k3s |
| Environment | Normal operation, steady load of 20 requests/s through the gateway for 100 s |
| Response | Kubernetes starts replacement pods; the Service routes only to ready pods; callers keep using the same DNS name |
| Response measure | **0 failed requests**, replacement Ready within 30 s (proposed target) |

## Result

Measured with `k8s/scripts/replacement-demo.sh` on 31 Oct 2026, six runs, each 2,000 requests to `GET /api/v1/me/profile`. These runs used the Python Profile; since 8 Nov 2026 Profile is the team TypeScript service, so re-run the scenario before presenting it as a measurement of the current Profile:

| Run | Succeeded | Failed | p50 | p95 | Max |
| --- | ---: | ---: | ---: | ---: | ---: |
| `20261031T035915Z` | 2,000 | 0 | 11 ms | 22 ms | 676 ms |
| `20261031T071537Z` | 2,000 | 0 | 11 ms | 43 ms | 983 ms |
| `20261031T072628Z` | 2,000 | 0 | 11 ms | 20 ms | 133 ms |
| `20261031T073218Z` | 2,000 | 0 | 11 ms | 23 ms | 1,263 ms |
| `20261031T094417Z` | 2,000 | 0 | 11 ms | 19 ms | 227 ms |
| `20261031T105121Z` | 2,000 | 0 | 11 ms | 21 ms | 268 ms |
| **Total** | **12,000** | **0** | | | |

- In the first run each replacement pod was Ready about **3 s** after deletion; pod IPs changed (10.42.0.26/.35 → 10.42.0.37/.38) while `profile` kept resolving to the same ClusterIP 10.43.28.236.
- The maximum latency is the one slow second right after a deletion, while kube-proxy drops the old endpoint. The p95 stays near 20 ms.

## What makes it work

- **Readiness probes** keep a pod out of the Service until its database connection works.
- **`preStop` sleep of 5 s** lets kube-proxy remove the endpoint before the server stops, so no request reaches a dying pod.
- **Stateless services** keep all state in PostgreSQL, so any replica can answer any request.
- **DNS-based service discovery** means callers never hold a pod address (see [`architecture/deployment.md`](architecture/deployment.md)).

## gRPC: the same test on Buddy Discovery

With 2 Discovery pods at 10 requests/s, deleting the busiest pod gave **749 of 750** successful calls; the one call in flight on the broken HTTP/2 connection returned 503. Spreading load across gRPC pods also needed the headless Service, `round_robin`, and a 30 s maximum connection age (150/150 calls instead of 300/0). A gateway retry for idempotent reads would remove the remaining failure; it is tracked as risk R4 in [`risk-matrix.md`](risk-matrix.md).

## Limits

Single node and single VM: this shows instance-level resilience, not node or zone failure, and it does not establish the 99% uptime of NFR-3.1. Raw evidence for every run (probe log, timeline, pod events, DNS and endpoints before and after) is in [`running-buddy-integration/load-tests/results/`](https://github.com/Chula-Com-Sci-2026/running-buddy-integration/tree/main/load-tests/results), one folder per run ID above; the method is in [`k8s/README.md`](https://github.com/Chula-Com-Sci-2026/running-buddy-integration/blob/main/k8s/README.md).
