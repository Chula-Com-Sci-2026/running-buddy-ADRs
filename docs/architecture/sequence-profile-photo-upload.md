# Sequence: upload a profile photo

How a runner adds a photo to their profile (UC-01 Create a Runner Profile). It is the one flow where the browser talks to something other than the API Gateway: the image goes straight to object storage on a signed URL, so the gateway and Profile only handle small JSON requests. Checked against `profile-service` (`create-photo-upload.ts`, `attach-photo.ts`, `minio-photo-storage.ts`), `api-gateway/app.py` and `cant-run-alone-frontend/src/api/gateway.ts`.

![Profile photo upload sequence](images/sequence-profile-photo-upload.png)

```mermaid
sequenceDiagram
    autonumber
    actor Ray as Ray (runner)
    participant W as Web App
    participant G as API Gateway
    participant P as Profile
    participant S as Photo storage (MinIO)
    participant PDB as Profile PostgreSQL
    participant Q as RabbitMQ (cra.events)

    Ray->>W: Choose an image (JPEG, PNG or WebP, up to 5 MB)
    W->>W: Check type and size before any request
    W->>G: POST /me/profile/photo-uploads (JWT)
    G->>P: Forward with JWT
    P->>PDB: Profile exists?
    P->>S: HEAD bucket (storage answers?)
    P->>P: Sign PUT for a fresh key photos/{runnerId}/{uuid}, 5 min
    P-->>W: 201 uploadUrl, photoRef (via gateway)
    W->>S: PUT image bytes to uploadUrl (no token, signature in URL)
    S-->>W: 200
    W->>G: PUT /me/profile/photo {photoRef} (JWT)
    G->>P: Forward with JWT
    P->>S: HEAD object (key is this runner's and was uploaded?)
    P->>PDB: One transaction: set photo_ref + outbox ProfileUpdated
    P-->>W: 200 profile with photoUrl (via gateway)
    P--)Q: Outbox relay publishes ProfileUpdated (new photoUrl)
    P--)S: Delete the replaced photo, best effort
    W->>S: GET photoUrl as an image (public read)
```

## Why the bytes skip the gateway

- **Gateway and Profile stay small.** They never buffer images, so a large upload cannot tie up the gateway's workers or Profile's memory. Storage is built for that load.
- **The signature is the permission.** The URL works for one key, one method (PUT) and five minutes. Profile checks the key belongs to the caller before attaching it, so a runner cannot adopt another runner's photo.
- **A failed upload leaves no broken image.** The profile only changes in step 14, after storage confirms the object exists. Until then the old photo stays.
- **Storage outages fail early.** Profile asks storage before signing, so an outage is a 503 at step 6 instead of a network error in the browser's upload.

## Deployment detail

The signed URL and `photoUrl` use the storage address the browser can reach: `http://127.0.0.1:9000` in Compose, NodePort `30091` on Kubernetes (see [`deployment.md`](deployment.md)). Profile reaches storage inside the network at `profile-storage:9000`.
