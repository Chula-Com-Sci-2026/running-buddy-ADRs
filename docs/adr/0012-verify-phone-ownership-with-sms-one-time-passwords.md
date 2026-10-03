# 12. Verify phone ownership with SMS one-time passwords

Date: 2026-10-03

## Status

Accepted

Supersedes [4. Authentication, phone verification, and authorization](0004-authentication-phone-verification-and-authorization.md)

## Context

Safety was the second most important concern in user interviews: running with someone found online means meeting a stranger. A verified badge on profiles and search results (FR-3.2), and a "verified runners only" filter (FR-3.3), are how the product answers that, so verification is a product requirement, not only a security control. Thai mobile numbers are registered to their holder's identity document, so proving possession of a number is a meaningful identity signal. Almost every target customer (students and early-career workers in Bangkok) has a Thai mobile number. Registration must still finish if the SMS provider is down (FR-3.5).

This ADR covers only how the badge is earned. Sign-in and sessions are [13. Authenticate with short-lived JWTs and rotating refresh tokens](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md); permissions are [14. Authorize every request in the service that owns it](0014-authorize-every-request-in-the-service-that-owns-it.md).

Options considered:

- **SMS one-time password to a Thai mobile number.** Fast, familiar, and tied to a registered identity. Costs money per message, depends on an external provider, and needs anti-abuse limits.
- **Email one-time password.** Free and simple, but anyone can create an email address in seconds, so it proves almost nothing about a real person.
- **Authenticator-app code (TOTP).** No per-message cost, but proves possession of an app, not of a phone number or identity, and adds setup friction at the moment a new runner is deciding whether to bother.
- **Identity-document check (eKYC) through a vendor.** The strongest signal, but it needs a commercial agreement, handles sensitive personal data under PDPA (NFR-4.3), and is out of reach for a semester project.

## Decision

We will verify phone ownership with an SMS one-time password, sent by the Identity Service through an SMS/OTP adapter. A verified phone earns the verified badge.

When SMS delivery fails, Identity emails a one-time password instead. That confirms the email address only: the runner can finish registration, but stays without the badge until an SMS one-time password succeeds.

In phase 1 the phone-verified fact is the badge. From phase 2, Trust & Safety grants the badge and admins can review and reject it (FR-3.4).

## Consequences

- The badge is backed by real possession of a Thai mobile number, the trust signal the product depends on.
- The email fallback keeps registration working during an SMS outage without weakening the badge.
- Runners without a Thai mobile number cannot get the badge; this excludes few target customers.
- SMS has a per-message cost and needs one-time password expiry, at most three attempts, and per-number rate limits (FR-3.1).
- The provider sits behind an adapter, so it can be swapped without touching domain logic, and an outage degrades registration only.
- Follow-on decisions: the one-time password lifetime and rate-limit values, and whether a runner whose badge an admin rejected can earn it again.

Source: Deliverable 1, "Authentication, Phone Verification, and Zero Trust Authorization," split per review feedback so each ADR weighs one set of options.
