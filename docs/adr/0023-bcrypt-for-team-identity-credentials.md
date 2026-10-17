# 23. Bcrypt for credentials in the team Identity implementation

Date: 2026-10-17

## Status

Accepted for the local demo using the team Identity PR branch, under delegated owner approval. Partially supersedes the Argon2id password-hashing assumption recorded in the Context of [ADR-0013](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md), inherited from historical ADR-0004, for that implementation; the preserved Python fallback continues to use Argon2id.

## Context

The team's Node/Express Identity Service already stores bcrypt password hashes. Replacing it with Argon2id would require a new dependency and a credential migration or dual-verification strategy. Keeping bcrypt preserves its user/password-reset compatibility and lets the tested service replace the temporary implementation. Argon2id remains the preferred memory-hard alternative, but migrating all existing hashes is outside this integration PR.

## Decision

Keep bcrypt in the team Identity Service, use cost 12 for newly registered accounts, and reject passwords longer than 72 UTF-8 bytes rather than silently truncating them. Existing bcrypt hashes remain verifiable; no local Argon2 account database is imported into the team's schema. Stable demo UUIDs are reseeded with bcrypt hashes in a separate local database. Password-reset legacy behavior remains subject to its own subsequent review.

## Consequences

The implementation differs from the original combined ADR's Argon2id choice and must be described accurately in the submission. The local Python fallback and team service maintain independent credential databases; switching endpoints does not migrate passwords. There is no change to JWT issuer/audience or downstream authorization requirements.
