# Run Scheduling Service contract (draft)

[run-scheduling.openapi.yaml](run-scheduling.openapi.yaml) is the **internal service contract** for the Run Scheduling Service in the [microservice architecture](../architecture/README.md). The React app does not call this service. It calls the API Gateway using the separate `cant-run-alone-frontend/docs/api/frontend-gateway.openapi.yaml` contract; the gateway authenticates/routes requests and adapts the response for the UI.

## Gateway-to-service mapping

| Frontend gateway request | Run Scheduling operation | Gateway adaptation |
| --- | --- | --- |
| `GET /runs` | `GET /runs` | Unwrap `items`; enrich conditions/organizer display for the current UI. |
| `GET /runs/{runId}` | `GET /runs/{runId}` | Map `startsAt` to date/time, `conditions` to weather, IDs to display names. |
| `POST /runs` | `POST /runs` | Combine date + time with `Asia/Bangkok` offset; map flat latitude/longitude to `location`; map visibility values. |
| `PUT /runs/{runId}/participants/me` | Same path and method | Map participation result to the gateway's `{ id, joined }` shape. |
| `DELETE /runs/{runId}/participants/me` | Same path and method | Preserve `204 No Content`; the current frontend request helper must not attempt to parse JSON for this response. |
| Not yet in gateway/UI | `POST /runs/conditions-preview`, `PATCH /runs/{runId}`, `PUT /runs/{runId}/cancellation` | Add gateway operations before exposing these flows to the browser. |

For example, the existing UI's `2026-09-22` + `18:30` ICT becomes `2026-09-22T18:30:00+07:00` in the service contract. The service stores an absolute instant; the gateway controls locale formatting.

Example service request after gateway adaptation:

```http
POST /runs HTTP/1.1
Authorization: Bearer <runner-token>
Content-Type: application/json

{
  "title": "Easy evening 5K",
  "meetingPoint": "Lumphini Park Gate 4",
  "location": { "latitude": 13.7317, "longitude": 100.5415 },
  "startsAt": "2026-09-22T18:30:00+07:00",
  "distanceKm": 5,
  "pace": "6:00–6:30",
  "participantLimit": 8,
  "visibility": "public"
}
```

The service returns `201 Created`, a `Location: /runs/{runId}` header, and a `Run` body with the organizer ID, participation count/status, and checked conditions. The gateway then adapts that body to its existing frontend `Run` shape.

## Collaborations outside this OpenAPI file

The Run Scheduling Service calls Run Conditions Service (`getRunConditions`, and `suggestAlternativeSlots` for unsuitable/uncertain previews). Its HTTP/gRPC wire contract belongs to that service, not this one. The service also publishes `RunScheduled`, `RunChanged`, `RunCancelled`, `ReminderDue`, `ForecastChanged`, and `RunEnded`; it consumes `RunnerVerified`, `VerificationRejected`, and `AccountSuspended`. Scheduled jobs send reminders, recheck forecasts, end runs, and update runner status. In phase 1 these events are REST POSTs to each subscriber's internal events endpoint, not Kafka messages ([ADR-0019](../adr/0019-call-services-over-rest-until-the-phase-2-event-broker.md)), and `RunEnded` waits for phase 2 with Reputation. These are **not public HTTP operations** and should be described with an event contract (for example AsyncAPI) if the team implements them. Use an outbox and idempotent consumers so DB updates and event delivery remain consistent.

## Decisions for team review

- **Authentication:** The spec assumes the gateway forwards a runner JWT and the service verifies it. Confirm token issuer/audience and service-to-service authentication before implementation.
- **Eligibility:** The spec requires verified, unsuspended organizers and participants. Confirm whether unverified runners may browse or join; the architecture names verification/suspension events but does not define this policy.
- **Privacy:** Exact meeting-point name and coordinates are withheld from non-participants. The current frontend mock and gateway schema expose them in public run cards/details, so those must be changed together if this policy is adopted.
- **Conditions failures:** `503` means no usable current/cached conditions. Agree on forecast age limits and whether creation can proceed with stale conditions.
- **Invitation access:** The architecture mentions invitations, but no invitation operation is defined in the design table or this first contract. Define invite semantics before implementing invite-only runs.
- **Pagination:** The service uses cursor pagination; the existing gateway returns a plain array. Confirm whether the gateway will keep adapting that shape or expose pagination to the frontend.

This is a proposed example contract, not evidence that a backend service already implements it.
