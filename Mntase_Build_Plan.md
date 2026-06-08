# Mntase PWA and Data Engine Build Plan

Based on `Mntase_Data_Engine_Spec.pdf` and the four supplied UI prototypes.

## Planning assumptions

- Target pilot window: June-November 2026.

## Recommended stack

### Frontend and PWA

- React 19 and React Router
- Tailwind CSS and the existing Radix UI component set
- React Hook Form and Zod for forms and validation
- Workbox for service-worker generation, caching, and offline queues
- IndexedDB through Dexie for offline check-ins and pending events
- Web Push API, with Firebase Cloud Messaging as the recommended delivery provider
- Recharts for operational dashboards and campaign reporting

### Backend and APIs

- FastAPI and Pydantic
- REST APIs for the pilot, retaining the current `/api` structure
- JWT access and refresh authentication
- Phone OTP for Mntases; retain email/password authentication for staff and clients
- Background processing with Cloud Tasks and Cloud Scheduler
- Idempotency keys on event ingestion, check-ins, campaign interactions, and points awards

### Data platform

- Supabase-managed PostgreSQL as the target source of truth
- SQLAlchemy 2 and Alembic for application access and schema migrations
- Append-only `events` table using PostgreSQL `jsonb`
- Relational projection tables for members, consent, organisation graph, campaigns, wallets, and subscriptions
- PostgreSQL materialized views for audience, wellbeing, and campaign aggregates
- dbt Core for repeatable analytics transformations and tests
- Supabase Storage or Google Cloud Storage for campaign assets and report files

### Infrastructure and operations

- Google Cloud Run for the FastAPI API and React web application
- Cloud SQL PostgreSQL is an acceptable alternative if the team wants all production services in GCP; do not operate both Cloud SQL and Supabase
- Google Secret Manager for credentials and VAPID/FCM keys
- Cloud Build for CI/CD
- Cloud Scheduler for nightly rollups and notification schedules
- Cloud Tasks for delivery jobs and retryable background work
- Sentry for frontend and backend error reporting
- Google Cloud Logging and Monitoring for service health, job failures, and alerting
- PostHog for privacy-conscious product analytics, with PII excluded
- Pytest for backend tests, React Testing Library for components, and Playwright for critical end-to-end journeys

## Database rewrite and cutover strategy

The current application uses MongoDB, while the data-engine design is relational and explicitly specifies PostgreSQL. Recursive organisation relationships, consent-state joins, audience cubes, materialized views, audit trails, and dbt transformations fit PostgreSQL better.

The database will be rewritten and moved to PostgreSQL in one coordinated cutover:

1. Finalise the complete PostgreSQL schema for identity, operational, analytics, and financial data.
2. Rewrite all application repositories and APIs to use PostgreSQL.
3. Build repeatable MongoDB-to-PostgreSQL extraction, transformation, and loading scripts.
4. Rehearse the full migration against production-like snapshots until row counts, relationships, balances, consent states, and campaign totals reconcile.
5. Freeze production writes for the agreed maintenance window.
6. Run the complete migration, validation suite, and smoke tests.
7. Switch every application surface to PostgreSQL in the same production release.
8. Keep MongoDB as a read-only rollback snapshot for a short, defined recovery window, then decommission it.

There will be no production dual writes and no domain-by-domain transition. PostgreSQL becomes the sole source of truth at cutover. The release must have a timed rollback decision point and documented recovery procedure if reconciliation or smoke tests fail.

## Ordered build plan

### Phase 0: Scope, architecture, and environments

**Duration:** 1-2 weeks

**Build**

- Confirm the pilot journeys and acceptance criteria.
- Create development, staging, and production environments.
- Define event naming, payload conventions, schema versions, and idempotency rules.
- Document privacy classifications and retention requirements.
- Decide between Supabase PostgreSQL and Cloud SQL PostgreSQL.
- Define the migration maintenance window, reconciliation thresholds, rollback decision point, and MongoDB decommission date.
- Add automated frontend and backend checks to CI.

**Deliverable**

A signed-off architecture, prioritised pilot backlog, deployed staging environment, versioned event catalogue, and approved one-time database cutover runbook.

### Phase 1: Data foundation and consent spine

**Duration:** 4 weeks

**Build**

- Create `events`, `members`, `consent_ledger`, `org_nodes`, `org_edges`, `audit_log`, and `event_ingestion_keys`.
- Implement the append-only event ingestion API.
- Implement versioned consent grants and revocations.
- Add current-consent projections and enforce consent in service-layer reads and writes.
- Add audit logging for access to member-derived data.
- Create migrations, indexes, backup policy, and restore test.
- Rewrite all application data access against PostgreSQL repository interfaces.
- Build the complete MongoDB extraction, transformation, reconciliation, and import tooling.
- Rehearse the migration with production-like data and record its expected duration.

**Deliverable**

The complete PostgreSQL model is ready for cutover, all application data access uses it, migration rehearsals reconcile successfully, events are ingested exactly once, and protected operations are rejected without active consent.

### Phase 2: Mntase onboarding and attribution

**Duration:** 4 weeks

**Build**

- Add referral routes such as `/j/{mpintshi_code}?src={source}`.
- Add phone registration and OTP verification.
- Resolve the referral code and create the member-to-Mpintshi relationship.
- Implement name capture, demographic fields required by the pilot, and consent screens.
- Emit `member.registered` and `consent.set`.
- Add the Lite-to-Verified member state and Mpintshi vouch queue.
- Preserve referral source and campaign parameters throughout onboarding.

**Deliverable**

A member can follow a referral link, verify their phone, provide consent, join the correct community, and appear in the Mpintshi verification queue.

### Phase 3: PWA shell, offline support, and push

**Duration:** 3 weeks

**Build**

- Convert the Mntase prototype into routed React screens.
- Add the web manifest, icons, install prompts, update handling, and service worker.
- Add IndexedDB storage and an offline event queue.
- Store browser push subscriptions and notification preferences.
- Configure FCM/Web Push credentials and test delivery.
- Add deep links from notifications into check-ins, surveys, and offers.

**Deliverable**

The Mntase application is installable, handles weak connectivity, and can receive a test push notification.

### Phase 4: Nightly check-in, points, streak, and wallet

**Duration:** 4-5 weeks

**Build**

- Implement mood and per-meal fullness check-ins.
- Emit `checkin.submitted` with ordinal values and local timestamps.
- Sync queued offline submissions idempotently.
- Implement a points ledger; never store only a mutable balance.
- Emit `points.awarded` and calculate wallet balance from ledger entries.
- Implement streak rules with timezone-safe day boundaries.
- Add the home, check-in, wallet, history, and consent-management screens.
- Schedule goodnight reminders and missed-check-in notifications.

**Deliverable**

A member can complete the nightly ritual online or offline, earn points once, grow a streak, and see the wallet update.

### Phase 5: Wellbeing rollups and Mpintshi application

**Duration:** 4-5 weeks

**Build**

- Add dbt models for daily mood distribution, seven-day trends, meal coverage, food-stress score, and streaks.
- Aggregate at community, ward, province, and national levels.
- Apply minimum-group suppression to community insights.
- Build the Mpintshi home, members, verification, recruitment, campaign, and earnings screens.
- Add quiet-member nudges without exposing private individual answers.
- Add QR and shareable referral links.

**Deliverable**

Mpintshis can recruit and verify members, monitor privacy-safe community participation, and see attributed earnings.

### Phase 6: Audience index and privacy enforcement

**Duration:** 4 weeks

**Build**

- Create the demographics projection and materialized audience cube.
- Implement age, gender, province, and ward filters.
- Return counts and breakdowns only.
- Enforce the hard k-anonymity floor of 100 in the API.
- Block campaign submission when the matched audience is below the floor.
- Add query performance tests with pilot-scale and projected-scale data.

**Deliverable**

The customer portal receives sub-second audience counts without receiving member identities, and unsafe audience selections cannot proceed.

### Phase 7: Campaign lifecycle and member participation

**Duration:** 5-6 weeks

**Build**

- Implement campaign drafts, audience specifications, content, budget, questions, and status history.
- Emit `campaign.created`, `campaign.submitted`, and `campaign.launched`.
- Build survey delivery, opening, answering, completion, and points awards.
- Build ad delivery, opening, clicking, and conversion tracking.
- Add scheduled campaign delivery through Cloud Tasks.
- Create near-real-time campaign aggregates for delivery, opens, responses, clicks, conversions, CPA, and completion.
- Attribute signups, responses, and conversions to Mpintshis.

**Deliverable**

A client can submit a campaign, approved campaigns can reach matched members, and interactions update campaign results and Mpintshi earnings.

### Phase 8: Head Office approval and fairness engine

**Duration:** 4 weeks

**Build**

- Add versioned fairness and policy rules.
- Check anonymity, narrow reach, protected targeting, content flags, and eligibility/exclusion wording.
- Produce pass, warning, and failure results with recommendations.
- Build approve, request-changes, reject, and reopen actions.
- Record the reviewer, ruleset version, checks, comments, and timestamps.
- Add optional Mpintshi feedback before final approval.

**Deliverable**

Every campaign follows an auditable customer-to-Head-Office approval loop before launch.

### Phase 9: Customer portal and Head Office analytics

**Duration:** 4-5 weeks

**Build**

- Connect the supplied customer and Head Office prototypes to real APIs.
- Add live performance and results views.
- Add Head Office mood, food-security, network, verification, and campaign dashboards.
- Implement ward-level anonymised report generation.
- Add methodology metadata and sign-off fields to exports.
- Add scheduled correlation queries for pilot insight headlines.

**Deliverable**

Clients and Head Office can operate the pilot and retrieve privacy-safe campaign and wellbeing outputs without manual database work.

### Phase 10: Hardening, pilot, and production launch

**Duration:** 4-5 weeks

**Build**

- Run accessibility, mobile-browser, low-bandwidth, and offline testing.
- Perform consent-revocation, anonymity, duplicate-event, and points-abuse tests.
- Add rate limits, OTP abuse protection, webhook validation, and security headers.
- Run load tests for event ingestion, audience matching, notifications, and dashboards.
- Complete backup restoration and incident-response exercises.
- Pilot with a small Mpintshi network, fix issues, then expand in controlled cohorts.
- Add operational dashboards and alerts for event lag, rollup failures, push failures, and campaign delivery.

**Deliverable**

A monitored, recoverable, security-reviewed production pilot with documented support and incident procedures.

## Recommended calendar

| Period | Main outcome |
|---|---|
| 8-19 June 2026 | Architecture, environments, event catalogue, PostgreSQL decision |
| 22 June-10 July | Event log, consent ledger, audit foundation |
| 13 July-7 August | Member onboarding, attribution, PWA installation, push foundation |
| 10 August-4 September | Check-ins, offline queue, points, streaks, wallet |
| 7-25 September | Wellbeing rollups and Mpintshi application |
| 28 September-16 October | Audience index and campaign lifecycle |
| 19 October-6 November | Fairness engine, approval loop, customer and Head Office wiring |
| 9-27 November | Hardening, controlled pilot, reporting, production readiness |

This calendar covers the focused pilot scope. Completing every phase and all four product surfaces will extend into early 2027.

## First two sprints

### Sprint 1: 8-19 June

- Finalise event taxonomy and payload schemas.
- Provision PostgreSQL staging.
- Add SQLAlchemy, Alembic, and repository boundaries.
- Implement `events` and `consent_ledger`.
- Define the full MongoDB-to-PostgreSQL field mapping and cutover runbook.
- Add ingestion idempotency and consent service tests.
- Convert the Mntase prototype into a React route skeleton.

### Sprint 2: 22 June-3 July

- Add referral-code resolution.
- Implement phone OTP request and verification.
- Create member and attribution projections.
- Build the first complete migration script and reconciliation report.
- Build onboarding and consent UI.
- Emit `member.registered` and `consent.set`.
- Demonstrate the full referral-to-registered-member flow in staging.

## Scope controls

Build for the pilot:

- Member PWA, Mpintshi application, customer portal, and Head Office surface
- Consent, attribution, check-ins, surveys, ads, points, campaign metrics, fairness checks, push, and anonymised reporting
- One PostgreSQL database with projections and materialized aggregates

Defer:

- Three-cloud data vaults
- NIMS or alternative credit scoring
- Public partner API
- Voice collection and multilingual sentiment
- USSD
- Dedicated graph database
- Kafka or a large streaming platform
- Machine-learning content moderation

## Definition of pilot success

- At least 90% of registration and check-in events process without manual intervention.
- Duplicate requests never award duplicate points or duplicate responses.
- Consent revocation excludes a member from subsequent protected processing.
- No client endpoint exposes raw member-level data.
- Audience queries below 100 are blocked.
- Nightly rollups and notification jobs are monitored and recoverable.
- One survey and one advert can complete the full create-review-launch-results lifecycle.
- Mpintshi attribution and earnings reconcile against member and campaign events.
