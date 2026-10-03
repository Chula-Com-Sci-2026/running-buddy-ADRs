# Requirements

Requirements for **Can't Run Alone**, the running buddy web app, from Deliverable 1 (revised after review). Items marked *(phase 2)* are out of the first delivery; see [Delivery phases](#delivery-phases).

---

## Problem Description

Recreational runners in Bangkok want to run with other people, but have no reliable way to find someone who runs at their pace, in their area, at the times they are free. Existing tools do not solve this: Strava is built around recording runs after they happen and does not match people by pace, area, and free time; Facebook groups and Meetup require manual scrolling with no pace matching and no identity verification; and none of them connect a planned run to the weather.

Weather is not a side concern in Bangkok. Heat, sudden rain, and PM2.5 decide whether a run is a good idea, and today runners discover the problem when they arrive at the park. Safety is the second most important concern raised in interviews: running with someone found online means meeting a stranger in person, so runners want to know that the other person is real and accountable, and to meet in a public place.

The system should let a runner state their pace, usual running area, weekly availability, and experience level; propose compatible partners nearby; let either runner schedule a run at a pinned public meeting point; show the weather forecast and air quality for that place and time; and hold the community accountable through phone-verified profiles and post-run ratings.

These problems came from user interviews conducted as part of the Javelin Board validation:

| Problem (customer's words) | Addressed by | Use cases |
|---|---|---|
| No running buddy | Discovery & matching, run scheduling | UC-02, UC-14, UC-03, UC-08 |
| Want to socialize through running | Group runs, buddy requests, ratings | UC-02, UC-14, UC-03, UC-08, UC-04 |
| Want more knowledge about running | Experience level on the profile, and an experience-level filter in discovery so newer runners can find more experienced ones | UC-07, UC-02 |
| Want weather info to plan runs | Weather info, weather-aware scheduling | UC-09, UC-03, UC-10 |
| Safety concern | Verified profiles, rating system, public meeting points, blocking and reporting | UC-01, UC-05, UC-04, UC-03, UC-02, UC-11 |

Validation evidence:

| Experiment | Riskiest assumption | Method & success criterion | Result | Verdict |
|---|---|---|---|---|
| 1 | Customers have this problem | Interview 4 people; all 4 must confirm they want a running buddy | 4 / 4 | Criterion met: assumption validated |
| 2 | Customers will use the app | Interview 4 people; all 4 must say they would use it | 3 / 4 | Criterion not met (4 / 4 required): assumption not yet validated |

**Experiment 2 follow-up:** [Team to fill in from the interview notes: why the fourth interviewee said they would not use the app, and whether the team persevered, pivoted, or re-ran the experiment.]

---

## Target Customers

University students and early-career office workers in Bangkok who run 2-3 times a week.

This segment was chosen deliberately rather than targeting all runners: they are reachable for interviews and testing, they are the least likely to already belong to a fixed running group, they run in shared public spaces (Benjakitti Park, Lumpini Park) where meeting points are natural and reasonably safe, and Bangkok's heat, rain, and PM2.5 make weather planning a real weekly concern for them.

---

## Actors and System Scope

### Actors

| Actor | Kind | Role in the system |
|---|---|---|
| Visitor | Primary (person) | Someone without an account. The only thing a visitor does is register (UC-01). |
| Runner | Primary (person) | A registered member. Verifies their phone, manages their account and profile, finds buddies and answers buddy requests, and reports misuse. |
| Organizer | Primary (person), a kind of Runner | A runner who creates and manages a run (UC-03). |
| Participant | Primary (person), a kind of Runner | A runner who joins a run and, after it, confirms attendance, rates, and logs the run (UC-08, UC-04). |
| Admin *(phase 2)* | Primary (person) | A moderator who reviews verifications and reports and suspends accounts (UC-12, UC-13). |
| SMS/OTP Provider | Secondary (external system) | Sends one-time passwords to Thai mobile numbers (UC-05). |
| Email Provider | Secondary (external system) | Sends password-reset links and the email one-time password used when SMS fails (UC-06, UC-05). |
| Weather & Air-Quality Provider | Secondary (external system) | Supplies the forecast and PM2.5 for a place and time (UC-09). |
| Map Provider | Secondary (external system) | OpenStreetMap-based map tiles used to pin meeting points and show runs on a map (UC-03, UC-08, [ADR-0011](adr/0011-render-maps-with-leaflet-and-openstreetmap-tiles.md)). |

Notifications are sent by the system itself (FR-9), so notification is a function inside the system boundary, not an actor. Time-triggered steps (attendance prompts, reminders, forecast re-checks, and buddy request expiry) are started by the system's own scheduler; the "Temporal" trigger type in UC-04 records this.

### Scope

- **In scope:** a web application for desktop and mobile browsers covering registration with phone verification, runner profiles, buddy discovery and requests, run scheduling with weather and air quality, attendance confirmation and ratings, manually logged running records, notifications, and, from phase 2, admin moderation.
- **Out of scope:** native mobile apps ([ADR-0009](adr/0009-build-the-web-client-as-a-react-single-page-application.md)) and live GPS tracking during a run (see the note under FR-8).

### Delivery phases

Review feedback asked the team to finish buddy search, run scheduling, and the weather check before anything else, and to leave the admin features for later ([ADR-0018](adr/0018-build-phase-1-with-six-services.md)).

- **Phase 1:** registration with phone verification, account basics, runner profiles, buddy discovery and requests, blocking, run scheduling with weather and air quality, joining runs, and in-app notifications. Use cases UC-01 to UC-03, UC-05 to UC-10, and UC-14.
- **Phase 2:** attendance, ratings, and running records (UC-04); reporting (UC-11); admin review and moderation (UC-12, UC-13); data export and account deletion; email notifications.

In phase 1 a verified phone number is the verified badge, because admin review of verifications (FR-3.4) arrives in phase 2.

---

## Scenario (use-case & description)

### Use case summary

UC-01 to UC-04 are the main scenarios and are described in full below. UC-05 to UC-14 complete the use case diagram and are summarized here so that every use case on the diagram traces to requirements. FR-9.1 (notifications) applies across UC-02, UC-03, UC-04, and UC-14, so it is not listed against one use case.

| ID | Use case | Primary actor | Secondary actor | Include / extend | Summary | Requirements | Phase |
|---|---|---|---|---|---|---|---|
| UC-01 | Register and Verify Identity | Visitor | - | Includes UC-05 | Described in full below. | FR-1.1, FR-3.1, FR-3.2, FR-3.5 | 1 |
| UC-02 | Find a Compatible Running Buddy | Runner | - | - | Described in full below. | FR-2.7, FR-3.3, FR-4.1 to FR-4.3, FR-4.5, FR-7.4 | 1 |
| UC-03 | Schedule a Run with Weather Check and Pinned Meeting Point | Organizer | Map Provider | Includes UC-09; extended by UC-10 | Described in full below. | FR-5.3, FR-5.4, FR-6.1 to FR-6.3, FR-6.6 to FR-6.8 | 1 |
| UC-04 | Confirm Attendance, Rate Buddies, and Log Running Record | Participant | - | - | Described in full below. | FR-7.1 to FR-7.4, FR-8.1, FR-8.3 | 2 |
| UC-05 | Verify Phone Number by OTP | Runner | SMS/OTP Provider, Email Provider | Included by UC-01 | The SMS/OTP Provider sends a one-time password to the runner's Thai mobile number, and the system marks the number verified when the right code is entered. If the SMS/OTP Provider is unavailable, the system confirms the email address by email OTP instead; the phone stays unverified and gets no badge. Runs inside UC-01, and on its own when a runner whose verification failed or was postponed tries again. | FR-3.1, FR-3.2, FR-3.5 | 1 |
| UC-06 | Manage Account | Runner | Email Provider | - | Log in and out, reset a forgotten password by emailed link, export personal data, turn notification categories on or off, and delete the account (personal data removed within 30 days). | FR-1.2 to FR-1.5, FR-9.2 | 1 (export, deletion: 2) |
| UC-07 | Manage Runner Profile | Runner | - | - | Edit display name, photo, bio, pace range, usual distance, experience level, preferred running area, weekly availability, and gear; choose which fields are public; view own run history. | FR-2.1 to FR-2.6, FR-2.8, FR-8.2 | 1 (run history: 2) |
| UC-08 | Browse and Join a Run | Participant | Map Provider | - | Browse upcoming public runs, and invite-only runs the runner was invited to, on a map or as a list together with their forecast; join a run up to its participant limit (a full run cannot be joined); leave a run before it starts. | FR-5.3, FR-6.4, FR-6.5 | 1 |
| UC-09 | Check Weather and Air Quality | - (included by UC-03) | Weather & Air-Quality Provider | Included by UC-03 | Get the forecast and PM2.5 for a place and time and compute the run-suitability indicator. If the provider is unavailable, serve the last cached forecast with a staleness warning. | FR-5.1, FR-5.2 | 1 |
| UC-10 | Suggest Alternative Time Slots | - (extends UC-03) | - | Extends UC-03 when the run-suitability indicator is poor | Offer time slots with better conditions; the organizer accepts one, keeps the original time, or abandons creating the run. | FR-5.5 | 1 |
| UC-11 | Report a User or Review | Runner | - | - | Report a runner, or an abusive written review, for inappropriate behavior; the report enters the admin queue. Reachable from any profile (as in UC-02) and from any received review (as in UC-04). | FR-4.6 | 2 |
| UC-12 | Review Verification Queue | Admin | - | - | Review completed phone verifications and approve or reject each with a reason; a rejection removes the verified badge. | FR-3.4, FR-10.1, FR-10.3 | 2 |
| UC-13 | Moderate Reports and Suspend Accounts | Admin | - | - | Review reports about users and written reviews, and suspend an account; a suspended runner no longer appears in discovery and cannot receive buddy requests. | FR-10.2, FR-10.3 | 2 |
| UC-14 | Respond to a Buddy Request | Runner | - | - | The candidate reads a pending request and its note and accepts or declines it; the requester is notified. An unanswered request expires, and after a decline or expiry the requester cannot re-send to the same candidate within a cooling-off period. After an accept, either runner can schedule a run with the other already invited (UC-03). | FR-4.3, FR-4.4, FR-4.7 | 1 |

### UC-01: Register and Verify Identity

| Field | Value |
|---|---|
| Use case name | Register and Verify Identity |
| ID | UC-01 |
| Importance level | High |
| Primary actor | Visitor |
| Use case type | Business / Account Management |
| Trigger | Visitor chooses "Sign up" on the landing page. |
| Trigger type | External |

**Stakeholders and interests**
- Visitor: wants to join quickly and start finding running partners.
- Runner (other members): wants confidence that the people they meet are real, verified people.
- Admin: wants a reviewable verification trail and the ability to reject bad actors.

**Brief description:** A visitor creates an account with email, phone number, and password, then verifies ownership of the phone number by one-time password. A verified badge is attached to the profile and shown in search results. The completed verification is also added to the admin verification queue so an admin can review it later (UC-12).

**Relationships**
- Association: Visitor.
- Include: UC-05 Verify Phone Number by OTP, always (the SMS/OTP Provider, and the Email Provider for the fallback, take part through UC-05).
- Extend: none.
- Generalization: none.

**Precondition:** The visitor has a valid Thai mobile number.

**Postcondition:** A Runner account exists with a completed profile and a verified phone number; the verified badge is displayed, and the verification is in the admin verification queue.

**Normal flow of events**
1. Visitor submits email, phone number, and password, and accepts the terms and the stated purposes of data collection (NFR-4.3).
2. System validates the input and creates an unverified account.
3. System verifies the phone number (include UC-05):
   1. System asks the SMS/OTP Provider to send a one-time password to the phone number.
   2. Visitor enters the OTP.
   3. System verifies the OTP and marks the phone number as verified.
4. System adds the verification to the admin verification queue. *(phase 2)*
5. Visitor completes the runner profile: display name, photo, short bio, pace range in min/km, usual distance, experience level, preferred running area, and weekly availability.
6. System stores the profile and displays the verified badge.

**Subflows:** None. Logging in and out, password reset, data export, and account deletion are UC-06 Manage Account; the admin's review of verifications is UC-12 Review Verification Queue.

**Alternate/exceptional flows**
- 2a. Email or phone number already registered → registration is rejected with a message pointing to password reset.
- 2b. Input is invalid (for example a malformed email, a weak password, or a non-Thai number) → the system shows what to correct.
- 3a. The OTP expires, or a wrong OTP is entered three times → the account stays unverified and the visitor may request a new OTP, subject to rate limiting.
- 3b. SMS/OTP Provider unavailable → the system sends a one-time password to the email address instead, so the visitor can confirm the email and finish registration ([ADR-0012](adr/0012-verify-phone-ownership-with-sms-one-time-passwords.md)). The account stays phone-unverified, with no verified badge, until the runner completes UC-05 later. No partial account data is lost.

### UC-02: Find a Compatible Running Buddy

| Field | Value |
|---|---|
| Use case name | Find a Compatible Running Buddy |
| ID | UC-02 |
| Importance level | High |
| Primary actor | Runner |
| Use case type | Business / Discovery |
| Trigger | Runner opens the "Find buddies" screen. |
| Trigger type | External |

**Stakeholders and interests**
- Runner (searcher): wants partners who actually match their pace, area, and free time.
- Runner (candidate): wants relevant requests only, and control over what their profile exposes.
- Admin: wants blocking and reporting to work so the community stays safe.

**Brief description:** A runner searches for other runners by radius, pace range, availability window, and experience level. The system ranks candidates by pace overlap, area proximity, and schedule overlap, and the runner sends a buddy request to a candidate, who accepts or declines it in UC-14.

**Relationships**
- Association: Runner (the searcher).
- Include: none.
- Extend: none.
- Generalization: none.
- Data used from other use cases: verification status from UC-01 and UC-05, and the aggregate rating produced by UC-04. Radius and distance are computed by the system ([ADR-0016](adr/0016-search-nearby-runners-and-runs-with-postgis-on-coarsened-cells.md)), so no external map actor takes part.

**Precondition:** Runner is authenticated and has a profile with at least pace, area, and availability set.

**Postcondition:** A pending buddy request exists and the candidate has been notified.

**Normal flow of events**
1. Runner sets search filters: radius, pace range, availability window, experience level, and optionally "verified runners only".
2. System queries candidate profiles within the radius of the runner's preferred running area, excluding blocked users in both directions and suspended accounts.
3. System ranks candidates by pace overlap, area proximity, and schedule overlap, and returns the ranked list with each candidate's verified badge and, once at least three ratings exist, aggregate rating *(phase 2)*.
4. Runner opens a candidate's public profile, which shows only the fields that candidate made public and a coarsened area of roughly 1-2 km rather than an exact location.
5. Runner sends a buddy request with a short note.
6. System notifies the candidate, who answers in UC-14.

**Subflows**
- S-1 Block another user; blocked users never appear in each other's results.
- S-2 Report another user for inappropriate behavior (UC-11); the report enters the Admin queue. *(phase 2)*

Profile field visibility, which the candidate controls, is set in UC-07 Manage Runner Profile.

**Alternate/exceptional flows**
- 3a. No candidate matches the filters → the system reports zero results and suggests widening the radius or pace range; no partial or misleading match is shown.
- 5a. Candidate is suspended by an Admin between search and request → the request is refused. *(phase 2)*
- 5b. Runner already sent this candidate a request that was declined or expired within the cooling-off period → the request is refused (FR-4.4).

### UC-03: Schedule a Run with Weather Check and Pinned Meeting Point

| Field | Value |
|---|---|
| Use case name | Schedule a Run with Weather Check and Pinned Meeting Point |
| ID | UC-03 |
| Importance level | High |
| Primary actor | Organizer (a Runner) |
| Use case type | Business / Transactional |
| Trigger | Runner chooses "Create a run", or accepts a buddy request and proposes a time. |
| Trigger type | External |

**Stakeholders and interests**
- Organizer: wants to fix a time, place, and pace and have people show up.
- Participant: wants to know exactly where to meet and whether conditions are safe to run in.
- Admin: wants meeting points to be public places and cancellations to be traceable.

**Brief description:** A runner creates a run with a pinned meeting point, date, time, distance, target pace, and participant limit. The system always retrieves the weather forecast and air quality for that location and time and displays a run-suitability indicator. The run is then visible to nearby runners (public) or to the invited runners (invite-only), who join it through UC-08.

**Relationships**
- Association: Organizer; Map Provider (the meeting point is pinned on the map).
- Include: UC-09 Check Weather and Air Quality, always, for the run's place and time.
- Extend: UC-10 Suggest Alternative Time Slots extends this use case at the extension point "run conditions shown" (step 4), only when the run-suitability indicator is poor.
- Generalization: Organizer is a specialization of Runner.

**Precondition:** Organizer is authenticated and has a verified phone number.

**Postcondition:** A scheduled run exists with a pinned location, a participant list, and an attached forecast; invited runners, if any, have been notified.

**Normal flow of events**
1. Organizer pins a meeting point on the map and enters date, time, distance, target pace, and participant limit.
2. Organizer sets the run as public or invite-only; for an invite-only run, the organizer chooses the runners to invite, such as the buddy whose request was accepted.
3. System gets the forecast and PM2.5 for that location and time (include UC-09).
4. System computes and displays a run-suitability indicator from temperature, humidity, rain probability, and PM2.5. *Extension point: run conditions shown.*
5. Organizer confirms; the system creates the run, shows it to nearby runners if it is public, and notifies the invited runners if it is invite-only.
6. System sends a reminder to all participants before the run starts.

**Subflows**
- S-1 Organizer edits or cancels the run; all participants are notified.
- S-2 System re-checks the forecast for upcoming runs and notifies participants when it changes materially (FR-5.4).

**Alternate/exceptional flows**
- 1a. The date and time are in the past → the system asks for a future time.
- 3a. Weather & Air-Quality Provider unavailable → the system serves the last cached forecast for that area and time with a staleness warning; scheduling remains fully usable.
- 3b. No forecast exists yet for that date, or none is cached → the run is created with its forecast marked "pending", and S-2 attaches the forecast once it is available.
- 4a. Run-suitability indicator is poor → UC-10 applies: the system offers alternative time slots with better conditions; the organizer may accept one, keep the original time, or abandon creating the run.
- S-1a. Organizer cancels within the late-cancellation window before the start time → participants are notified immediately and the cancellation is recorded against the organizer's reliability (FR-6.7).

### UC-04: Confirm Attendance, Rate Buddies, and Log Running Record *(phase 2)*

| Field | Value |
|---|---|
| Use case name | Confirm Attendance, Rate Buddies, and Log Running Record |
| ID | UC-04 |
| Importance level | Medium |
| Primary actor | Participant (a Runner) |
| Use case type | Business / Reputation & Record-keeping |
| Trigger | The run's expected end time passes (start time plus distance × target pace). |
| Trigger type | Temporal (system-triggered), then External for each participant's confirmation. |

**Stakeholders and interests**
- Participant: wants credit for the run and an honest reputation.
- Runner (future partner): wants ratings to reflect people who actually showed up.
- Admin: wants abusive reviews reported and reviewable.

**Brief description:** After a scheduled run, participants confirm attendance. Once at least two participants confirm, the run is completed, and only those confirmed participants may rate each other on punctuality, pace accuracy, and friendliness. Participants may also log the distance and duration they ran.

**Relationships**
- Association: Participant.
- Include: none.
- Extend: none.
- Generalization: Participant is a specialization of Runner.
- Data used from other use cases: the run scheduled in UC-03 and its participant list from UC-08. The aggregate rating produced here is shown in UC-02 search results.

**Precondition:** A scheduled run exists whose expected end time has passed, and the participant was on its participant list.

**Postcondition:** The run is marked completed; ratings are recorded and included in each rated runner's aggregate; any logged distance and duration appear in the participant's run history.

**Normal flow of events**
1. System prompts participants to confirm attendance within the confirmation window after the run's expected end time.
2. Participants confirm attendance.
3. Once at least two participants confirm, the system marks the run completed.
4. System opens ratings for the confirmed participants only.
5. Participants rate the others on punctuality, pace accuracy, and friendliness, and may add a short written review.
6. System recomputes each rated runner's aggregate rating, keeping it hidden on the profile until at least three ratings exist.

**Subflows**
- S-1 Log running record (optional): participants log distance and duration for the run, and the system updates the participant's run history with cumulative distance and time.
- S-2 Participants choose whether their run history is publicly visible.
- S-3 Participants report an abusive written review to the Admin (UC-11).

**Alternate/exceptional flows**
- 3a. The confirmation window closes with fewer than two confirmations → the run is not completed, no ratings may be given, and no reputation effect is recorded.
- 5a. A participant who did not confirm attempts to rate → the system refuses (FR-7.3).
- 6a. Rated runner has fewer than three ratings → the aggregate stays hidden, and only the verified badge and profile details are shown.

---

## Functional Requirements

"User" below means a registered runner unless stated otherwise.

**FR-1 Account & Identity System**
- FR-1.1 The system shall allow a visitor to register with an email address, phone number, and password; each email address and phone number can be registered only once.
- FR-1.2 The system shall allow a user to log in and log out.
- FR-1.3 The system shall allow a user to reset a forgotten password by an emailed link.
- FR-1.4 The system shall allow a user to delete their account, removing their personal data (within 30 days, NFR-4.2). *(phase 2)*
- FR-1.5 The system shall allow a user to export their personal data (NFR-4.3). *(phase 2)*

**FR-2 Runner Profile System**
- FR-2.1 The system shall allow a user to create and edit a profile with display name, photo, and short bio.
- FR-2.2 The system shall allow a user to set a pace range in min/km and a usual distance.
- FR-2.3 The system shall allow a user to set an experience level of beginner, intermediate, or advanced.
- FR-2.4 The system shall allow a user to set a preferred running area.
- FR-2.5 The system shall allow a user to set weekly availability by day and time range.
- FR-2.6 The system shall allow a user to list gear such as shoes and watch.
- FR-2.7 The system shall allow a user to view another runner's public profile.
- FR-2.8 The system shall allow a user to control which profile fields are publicly visible.

**FR-3 Profile Verification System**
- FR-3.1 The system shall verify a user's phone number by a one-time password sent by SMS; an OTP expires, allows at most three attempts, and new OTP requests are rate-limited.
- FR-3.2 The system shall display a verified badge on the profile and in search results.
- FR-3.3 The system shall allow a user to filter discovery results to verified runners only.
- FR-3.4 The system shall allow an admin to review each completed phone verification and approve or reject it with a reason; a rejection removes the verified badge. *(phase 2)*
- FR-3.5 When SMS delivery fails, the system shall let the visitor confirm their email address by email OTP and finish registration without a verified badge.

**FR-4 Discovery & Matching System**
- FR-4.1 The system shall allow a user to search runners within a radius of their preferred running area, by pace range, availability window, and experience level.
- FR-4.2 The system shall rank candidates by pace overlap, area proximity, and schedule overlap.
- FR-4.3 The system shall allow a user to send a buddy request with a short note, and the recipient to accept or decline it.
- FR-4.4 The system shall expire an unanswered buddy request, and shall stop the requester from re-sending to the same candidate within a cooling-off period after a decline or expiry.
- FR-4.5 The system shall allow a user to block another user, so that blocked users never appear in each other's results.
- FR-4.6 The system shall allow a user to report another user, or a written review, for inappropriate behavior; the report enters the admin queue. *(phase 2)*
- FR-4.7 Once a buddy request is accepted, the system shall let either runner start scheduling a run with the other already invited.

**FR-5 Weather System**
- FR-5.1 The system shall show the current weather, forecast, and PM2.5 air quality for a given location and time.
- FR-5.2 The system shall show a run-suitability indicator derived from temperature, humidity, rain probability, and PM2.5.
- FR-5.3 The system shall display weather alongside every scheduled run.
- FR-5.4 The system shall notify participants when the forecast for an upcoming run changes materially, meaning its run-suitability indicator changes level.
- FR-5.5 The system shall suggest alternative time slots with better conditions when the run-suitability indicator is poor.

**FR-6 Run Scheduling System**
- FR-6.1 The system shall allow a verified user to create a run with a pinned meeting point, date, time, distance, target pace, and participant limit.
- FR-6.2 The system shall allow a user to set a run as public or invite-only.
- FR-6.3 The system shall allow the organizer to invite specific runners, such as an accepted buddy, to a run.
- FR-6.4 The system shall allow a user to browse upcoming runs nearby, on a map and as a list; an invite-only run is shown only to the runners invited to it.
- FR-6.5 The system shall allow a user to join a run, up to its participant limit, and to leave it before it starts.
- FR-6.6 The system shall allow the organizer to edit or cancel a run, notifying participants.
- FR-6.7 The system shall record a cancellation made within the late-cancellation window before the start time against the organizer's reliability, which admins can review. *(phase 2; phase 1 only flags the cancellation as late)*
- FR-6.8 The system shall send participants a reminder before the run starts.

**FR-7 Attendance & Rating System** *(phase 2)*
- FR-7.1 The system shall prompt participants to confirm attendance within a confirmation window after a run's expected end time (start time plus distance × target pace).
- FR-7.2 The system shall mark a run completed once at least two participants confirm attendance within the window.
- FR-7.3 The system shall allow only participants who confirmed attendance of a completed run to rate each other on punctuality, pace accuracy, and friendliness, with an optional short written review.
- FR-7.4 The system shall show a runner's aggregate rating on their profile and in search results only once at least three ratings exist.

**FR-8 Running Record System** *(phase 2)*
- FR-8.1 The system shall allow a participant to log distance and duration for a completed run.
- FR-8.2 The system shall allow a user to view their run history with cumulative distance and time.
- FR-8.3 The system shall allow a user to choose whether their run history is publicly visible.

Note: running records are entered manually after a run, **not tracked live**. Live GPS tracking needs background location, which a web application cannot provide.

**FR-9 Notification System**
- FR-9.1 The system shall notify users of buddy requests and responses, run invitations, run changes and cancellations, reminders, weather alerts, and attendance confirmation prompts.
- FR-9.2 The system shall allow a user to turn off each notification category, except run changes and cancellations, which are always sent.

Note: phase 1 delivers notifications in-app only; email notifications follow in phase 2.

**FR-10 Administration System** *(phase 2)*
- FR-10.1 The system shall allow an admin to review the verification queue (see FR-3.4).
- FR-10.2 The system shall allow an admin to review reports about users and written reviews, and to suspend an account; a suspended account no longer appears in discovery and cannot receive buddy requests.
- FR-10.3 The system shall keep an audit log of admin decisions (verification approvals and rejections, and suspensions) and of run cancellations.

---

## Non-functional Requirements

**NFR-1 Performance**
- NFR-1.1 Runner search shall return results in under 3 s at p95, with 10,000 profiles in the database.
- NFR-1.2 The nearby-runs map query shall return in under 2 s at p95.
- NFR-1.3 Weather data shall be served from cache in under 500 ms at p95; a cache miss shall not exceed 3 s.
- NFR-1.4 First contentful paint shall be under 3 s on a 5G connection.

**NFR-2 Scalability**
- NFR-2.1 The system shall support 5,000 registered users and 500 concurrent users without breaching the performance targets.
- NFR-2.2 The system shall handle peak load at 05:00-07:00 and 17:00-20:00, estimated at 5× average traffic.
- NFR-2.3 Any service shall be horizontally scalable without code changes.

**NFR-3 Availability & Reliability**
- NFR-3.1 The system shall maintain 99% daily uptime for core flows: login, search, and run scheduling.
- NFR-3.2 If the weather provider is unavailable, the system shall serve the last cached forecast with a staleness warning, and scheduling shall remain fully usable.
- NFR-3.3 Failure of a non-critical service (weather or notification) shall not cascade into failure of login, profile, search and matching, or run scheduling.
- NFR-3.4 No scheduled run data shall be lost on service restart.

**NFR-4 Privacy**
- NFR-4.1 The exact coordinates of a runner's preferred running area shall never be exposed to other users, whether directly or through repeated searches; discovery works only with a coarsened area of roughly 1-2 km. The only precise location shared is a run's pinned public meeting point, and for an invite-only run it is shown only to the invited runners.
- NFR-4.2 Personal data shall be deleted within 30 days of an account deletion request; the traffic logs that NFR-6.5 requires to be kept are the only exception. *(phase 2, with FR-1.4)*
- NFR-4.3 The system shall comply with Thailand's PDPA: stated purpose, consent at collection, and a working data export and deletion path.

**NFR-5 Security**
- NFR-5.1 All traffic shall use TLS 1.3; no endpoint shall be reachable over plain HTTP.
- NFR-5.2 Passwords shall be stored with a slow, salted password hash (Argon2id, which is memory-hard, or bcrypt), never in plaintext or reversible form.
- NFR-5.3 Every API endpoint, except the public ones for registration, OTP verification, login, and password reset, shall authenticate and authorize the caller; no endpoint shall be trusted merely because the call originated inside the cluster.
- NFR-5.4 Access tokens shall expire within 15 minutes, and refresh tokens shall rotate on use.
- NFR-5.5 Discovery endpoints shall be rate-limited to prevent bulk profile scraping.
- NFR-5.6 The application shall have no unresolved high-severity findings in the OWASP Top 10 categories, as reported by an automated security scan run before the demo, and automated tests shall check that every endpoint rejects callers without permission.

**NFR-6 Operational, Cultural, and Legal**
- NFR-6.1 The system shall display correctly on both desktop and mobile devices.
- NFR-6.2 The system shall support current versions of major browsers on Windows, macOS, iOS, and Android.
- NFR-6.3 The system shall provide a Thai language user interface, with English as an option.
- NFR-6.4 The system shall use Bangkok local time (ICT, UTC+7) as the default timezone for all scheduled runs.
- NFR-6.5 The system shall comply with Thailand's Computer-Related Crime Act B.E. 2550 (2007) as amended in B.E. 2560 (2017), including keeping computer traffic data for at least 90 days, and with the Personal Data Protection Act B.E. 2562 (2019) (see NFR-4.3).
- NFR-6.6 The system shall back up profile, run, and rating data daily.
