# 29. Object storage for profile photos

Date: 2026-11-01

## Status

Accepted

Makes an exception to [ADR-0009](0009-build-the-web-client-as-a-react-single-page-application.md) for photo files only: the browser sends and loads them directly from storage, not through the API gateway.

## Context

A runner's profile can have a photo (FR-2.1). Each service owns its PostgreSQL database ([ADR-0015](0015-give-each-service-its-own-postgresql-database.md)), and the web client talks only to the API gateway (ADR-0009). Storing photos must also respect NFR-3.3 and NFR-7.3.

Options considered:

- **Image files in Profile's PostgreSQL, served through the gateway.** No new component, but every upload and every view passes through the gateway and Profile, and the database and its backups carry the images.
- **S3-compatible object storage (MinIO locally) with presigned uploads.** The browser uploads straight to storage and loads photos from a public address, so Profile and the gateway carry no image traffic, at the cost of one more container.
- **An external image host.** No container, but a third-party dependency and a privacy question for runners' photos (NFR-4.3).

## Decision

Profile stores photos in an S3-compatible bucket (MinIO locally), under keys `photos/{runner id}/{random uuid}`. Anyone with a photo's address can read it, but the bucket can't be listed. `POST /api/v1/me/profile/photo-uploads` returns a presigned PUT link for a fresh key. `PUT /api/v1/me/profile/photo` attaches only a key that Profile made for this runner and whose file exists, and replacing or removing a photo deletes the previous file, best effort. `photoUrl` is the photo's public address, so it loads with no token.

This is the exception to ADR-0009: the browser talks to storage directly for two things only, uploading to a presigned link that covers one file and expires within minutes, and loading photos from their public addresses. Everything else still goes through the gateway. A photo's address doesn't expire; only the random uuid in its key keeps it from being guessed.

Locally, MinIO is published on `127.0.0.1` only, like the gateway in [ADR-0022](0022-service-authorization-for-the-local-stack.md). If it is down, Profile still starts and serves profiles; only getting an upload link and attaching a photo fail.

## Consequences

- Photo files bypass the gateway and Profile, which only sign links and check keys.
- Field visibility (FR-2.8) can't hide a photo, so a runner who doesn't want theirs seen removes it.
- Profile doesn't yet limit an upload's file type or size; this must be decided before production.
- Account deletion (phase 2) must delete a runner's photo files too.
- In production, any S3-compatible service works; its address and credentials are configuration.

Implementation evidence: `docs/openapi.yaml` (the photo routes) and `compose.local-integration.yaml` in the `profile-service` repository.
