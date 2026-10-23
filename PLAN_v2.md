# AICTE FlowGuard — Implementation Plan v2 (Fresh Repo)

Problem statement: Integrated AICTE Approval Workflow Automation, Compliance Monitoring and Alert Management System (SIH).

**What changed from v1:** v1 was implementation-first. v2 is **domain-first**: freeze the AICTE process, rules and authorities (Part A), then derive the state machine, data model, APIs, UI and infrastructure from it (Part B onward). It also trims over-engineering (roles, storage, infra, hash chain) so the demo journey works reliably.

**Standing rules (unchanged):** no demo data, no mock contexts, no hard-coded numbers; every dashboard figure is a SQL aggregate over real rows; empty system shows empty states; server is the source of truth for authorization; automation supports but never replaces the competent authority.

---

# PART A — DOMAIN FIRST (do this before writing code)

## A1. The AICTE Rule Matrix (Phase 0 deliverable, blocks Phase 3)

A spreadsheet/markdown table, reviewed by you and, if possible, a mentor or someone who has handled AICTE approvals. **Every row must cite a section/page of the Approval Process Handbook (APH) or an AICTE circular.** Rows without a source are marked UNVERIFIED and are not loaded as production config.

| ID | Application type | Conditions (what makes this rule apply) | Required stages (ordered) | Responsible authority per stage | Stage SLA + counting basis | APH reference | Status (VERIFIED / UNVERIFIED) |
|---|---|---|---|---|---|---|---|

Sheets to fill alongside it:
1. **Application types** actually in scope (see A2).
2. **Stage catalogue**: Scrutiny, Re-scrutiny, EVC, SAC/SHC, Decision, and anything else the APH defines; entry criteria, exit criteria, outputs.
3. **Authority matrix**: for every stage and every transition, which role may perform it. Anything not listed is forbidden.
4. **Deficiency catalogue**: the reason codes for returns/reverts and what the institution must do.
5. **Undertaking & document catalogue**: which approvals carry undertakings, what evidence is expected, lease/other document types, validity rules, renewal rules.
6. **Timeline policy** (see A5).
7. **Escalation matrix**: who is notified at each level and when.

**Research status (honest):** I could not retrieve the APH PDF itself from the AICTE site, so I have not verified stage sequences or timelines. A secondary summary of the handbook (TeamLease RegTech, handbook reported issued 6 Feb 2026) lists situations such as extension of approval, hibernation, merger, off-campus provision, new BCA/BBA/BMS programmes and ODL/online approvals, but gives no stage or timeline detail. Treat those only as candidate application types to confirm against the handbook. Download the current APH and the circulars yourself and fill the matrix from them.

**Rule:** the matrix is the spec. Workflow templates, routing rules, SLAs and the authority matrix in the software are loaded *from* the matrix, and each loaded rule keeps its APH reference in the database (`sourceRef` column) so the UI can show "why".

## A2. What exactly is an "application"? (domain model)

An application is a request by an institution, for a given academic year, for a specific approval action:

```
Application = Institution + AcademicYear + ApplicationType + RequestedAction
              + [ProgrammeRequest: programme/course, level, intake] (0..n)
              + Documents + (history: observations, deficiencies, returns, inspections,
                committee recommendations, decision, conditions/undertakings, approval order)
```

Concepts to model explicitly: AcademicYear, ApplicationType (config), RequestedAction (e.g. new, extension, change, closure, as defined in the matrix), ProgrammeRequest, Deficiency, Inspection/EvcCase, CommitteeRecommendation, Decision, ApprovalOrder, Undertaking, ComplianceHistory. Representations/appeals only if the APH defines them; otherwise leave out.

Be able to answer a judge: "What is being approved?" → "This institution, for this year, for these programmes/intakes, under this application type; the decision and its conditions are recorded against exactly that."

## A3. Classification → Routing → Template (no vague JSONB)

```
Application data (typed fields)
        ↓
Classification step (deterministic, rule-based)
        ↓
Classification result (typed: applicationType, institutionCategory[new/existing], requestedAction, flags such as requiresEvc)
        ↓
RoutingRule (ordered by priority; conditions over the typed classification fields)
        ↓
WorkflowTemplate (specific version)
```

- Every classification input is either (a) a field the institution enters, (b) a field derived from stored data (e.g. existing vs new from the institution master record), or (c) a field an officer sets with a recorded reason. The plan names the source of every field in the matrix.
- Rule conditions are a small, typed predicate set (equals, in, and/or), validated by Zod, not free-form JSON.
- The **evaluation trace** (which rules were checked, which matched, why, with `sourceRef`) is stored on the application and shown in the UI. An officer can correct a classification with a reason, which re-routes and is audited.
- If no rule matches: the application goes to a "manual routing" queue for the regional officer. It never silently gets a default template.

## A4. Automation boundary (shown in the UI and in the demo)

| Automatic (system does it) | Human (system only supports) |
|---|---|
| Classification and routing per verified rules | Scrutiny and judging document adequacy |
| Creating the next stage's task and queue entry after a human completes a stage | Completing a stage (the human act) |
| EVC queue placement | Inspection findings; committee recommendations |
| Deadline computation, reminders, escalation, notifications | Verifying compliance evidence or renewed documents |
| Case summary generation (factual compilation) | Final decision and conditions (competent authority only) |
| Dashboards, reports, audit capture | Return/revert (human, with reason) |

Wording rule: "automatic movement" means *administrative* movement to the next queue after a human completes the prior stage. No step ever produces approve/reject by itself. The UI banner on decision screens: "Decisions are taken by the competent authority; system outputs are advisory."

## A5. Timeline (SLA) policy, fully specified

Each stage SLA row has: `duration`, `unit` (CALENDAR_DAYS | WORKING_DAYS), `startsOn` event, `ownerParty` (OFFICER | INSTITUTION | COMMITTEE), `sourceRef`.

- **Separate clocks by party.** An officer clock and an institution clock never mix. When an officer returns an application, the officer clock *pauses* and an institution response clock starts with its own configured days. On resubmission, the institution clock stops and the officer clock resumes.
- **Resume policy is explicit per stage, config-chosen:** `RESUME_REMAINING` (officer gets the days that were left) or `RESET_FULL`. Example in the matrix: "Scrutiny 15 WD; returned day 5; institution takes 10 days; resubmitted → under RESUME_REMAINING officer has 10 WD left." (Numbers here are an illustration of the mechanism, not AICTE rules.)
- **Extensions:** an authorised role can grant an extension with reason and new due date; stored as `SlaExtension` rows, audited, never by editing the base SLA.
- **Calendar:** working days use a `Holiday` table maintained by admin (national and regional holidays).
- Due dates are computed once from the config **snapshot** the application was routed with (see B3), stored on the `SlaClock`, and recalculated only by explicit events (pause, resume, extension).

## A6. Alert vs escalation (distinct lifecycles)

- **Alert** (a reminder/warning about an item): `OPEN → ACKNOWLEDGED → RESOLVED`. Alert types by offset: UPCOMING (e.g. due in N days), DUE, OVERDUE.
- **Escalation** (a record that responsibility has been pushed up): `NOT_ESCALATED → LEVEL_1 → LEVEL_2 → LEVEL_3 → CLOSED`, with `escalatedAt`, `toRole/toUser`, `reason`.
- Resolving the underlying item resolves its alerts and **closes** the escalation, but the escalation history rows stay (so reporting can say "was escalated to L2 for 4 days"). Reports count both "ever escalated" and "currently escalated".
- Alert rule → recipient resolution → channels (see B8). Offsets and levels come from the escalation matrix in A1.

## A7. Institution lifecycle

Distinguish **Institution master record** (an AICTE entity: aicteId, name, type, region, status) from **portal accounts** (people who act for it).
- `Institution` (master) 1—n `InstitutionMembership` (user, membershipRole: ADMIN | NODAL_OFFICER | STAFF).
- MVP: one ADMIN-level user per institution is sufficient, but the table exists so adding more users later is not a migration.
- Created by a regional/bureau officer or imported; the portal invitation is a separate step (email link → set password).

---

# PART B — ARCHITECTURE (derived from Part A)

## B1. Stack (simplified)

| Layer | Choice | Note |
|---|---|---|
| Monorepo | pnpm workspaces | Turborepo optional |
| Web | Next.js 14 App Router, TypeScript, Tailwind, shadcn/ui, TanStack Query, React Hook Form + Zod, Recharts | |
| API | NestJS, Passport-JWT, class-validator, Swagger | |
| DB | PostgreSQL 16 + Prisma | |
| Jobs | BullMQ + Redis | scanner, outbox dispatcher, escalation |
| Files | `StorageAdapter` interface; **MVP driver = local disk volume**, production-ready driver = any S3-compatible bucket | drops MinIO/ClamAV from the critical path |
| Email | Nodemailer + SMTP (Mailpit in dev, real SMTP in prod) | |
| Shared | `packages/shared`: Zod schemas, enums, permission and role constants | |

Repo layout: `apps/web`, `apps/api`, `apps/worker`, `packages/shared`, `infra/`, `docs/rule-matrix/`, `PLAN.md`.

## B2. Roles: 7 for the MVP (down from 10), extensible by design

Roles are DB rows; permissions are the real unit. MVP roles (home route in brackets):

1. **ADMIN** (`/admin`): config, users, regions, matrix import, go-live checklist, audit view.
2. **BUREAU_OFFICER** (`/bureau`): all-region dashboards, escalations received, reports; also holds read-only audit permission (replaces a separate AUDITOR).
3. **REGIONAL_OFFICER** (`/regional`): own-region queue, assignment, ageing, EVC scheduling (coordinator permission), undertaking and document alerts, compliance verification permission.
4. **SCRUTINY_OFFICER** (`/scrutiny`): assigned worklist, observations, return, complete stage.
5. **COMMITTEE** (`/committee`): assigned EVC/SAC/SHC cases, upload report, submit recommendation (advisory).
6. **COMPETENT_AUTHORITY** (`/authority`): decision queue, record decision and conditions.
7. **INSTITUTION** (`/institution`): own applications, responses to returns, undertakings, evidence upload, document validity.

Splitting `EVC_COORDINATOR`, `COMPLIANCE_VERIFIER`, `AUDITOR` into separate roles later is a permissions reassignment, not a rebuild.

**Authorization layering:** `User → Role → Permissions → Data scope → Resource-level check`.
- Data scope kinds: REGION, INSTITUTION, ASSIGNMENT (assigned stage/case), COMMITTEE (member of committee), GLOBAL.
- Implemented once in a `ScopeService` that returns a Prisma `where` fragment per resource type; controllers never build their own filters.
- Resource check on every mutation: the actor must hold the permission **and** be in scope **and** the state machine must allow the transition for that actor.

## B3. Configuration versioning and snapshots

All rule-like config is versioned and immutable once published: workflow templates, routing rules, SLA policies, escalation policies, alert rules, document checklists, notification templates.

- A published `ConfigVersion` has `id`, `publishedAt`, `publishedBy`, and points to its rule rows.
- When an application is routed it stores `configVersionId` and a `ruleSnapshot` JSONB of the resolved template, SLA policy, escalation policy and checklist **as applied**. Later admin edits create a new version and affect only new applications.
- Changing a live application's rules is a deliberate, audited "re-baseline" action by an authorised role.

## B4. Workflow state machine

Single module (`workflow.service`) is the only writer of stage state. Every transition runs in one DB transaction that: validates actor and scope, applies the state change, updates `SlaClock`s, writes the audit event, and writes an **outbox event** (B7).

States per `StageInstance`: PENDING, ACTIVE, COMPLETED, RETURNED, SKIPPED. Application status derived from stages plus the decision.

Transitions:
- `classifyAndRoute` (automatic on submission; can be corrected by officer with reason)
- `assign` / `reassign` (regional officer)
- `completeStage` (human) → system activates next stage per snapshot; EVC stage activation creates `EvcCase` QUEUED
- `returnApplication` (human, reason code + text mandatory) → officer clock pauses, institution clock starts, institution notified
- `resubmit` (institution) → institution clock stops; stage resumes per resume policy
- `recordRecommendation` (committee; advisory)
- `recordDecision` (competent authority only): outcomes defined in the matrix; "approved with undertakings" requires creating Undertaking rows in the same transaction
- Illegal transitions return a typed 409 with a code the UI displays.

Constraints: partial unique index "one ACTIVE stage per application"; check `dueAt >= startedAt`.

Tests: table-driven per template (happy path, return/resubmit with both resume policies, reassignment, extension, illegal transitions, authority checks).

## B5. Data model (Prisma outline)

Identity/config: `User`, `Role`, `Permission`, `RolePermission`, `UserScope` (kind + target id), `Region`, `Holiday`, `RefreshSession`, `ConfigVersion`, `ApplicationType`, `StageType`, `WorkflowTemplate`, `WorkflowTemplateStage`, `ClassificationField`, `RoutingRule` (typed conditions, priority, `sourceRef`), `SlaPolicy`, `EscalationPolicy`, `AlertRule`, `NotificationTemplate`, `DeficiencyReason`, `DocumentChecklistItem`.

Institutions: `Institution`, `InstitutionMembership`, `AcademicYear`.

Applications: `Application` (institution, academicYear, applicationType, requestedAction, classification JSONB-typed-by-Zod, routingTrace, configVersionId, ruleSnapshot, status), `ProgrammeRequest`, `ApplicationDocument` (versioned), `StageInstance`, `SlaClock`, `SlaExtension`, `Observation`, `ApplicationReturn`, `Recommendation`, `Decision`, `ApprovalOrder`.

EVC: `EvcCase` (status QUEUED, SCHEDULED, COMPLETED, DELAYED; scheduledFor, dueBy, isNewInstitution, reportFileId), `Committee`, `CommitteeMember`, `EvcAssignment` (case, committee).

Compliance: `Undertaking`, `ComplianceSubmission`, `ValidityDocument` (type, validFrom, validUntil, status), `DocumentRenewal`.

Alerts: `Alert`, `Escalation` (separate table and lifecycle), `Notification`, `OutboxEvent`, `NotificationDelivery`.

Audit/files: `AuditEvent` (append-only), `FileObject`.

Indexes/constraints: partial unique on active stage; unique `Alert.dedupeKey`; check `validUntil >= validFrom`; revoke UPDATE/DELETE on `AuditEvent` for the app DB role; indexes on (regionId, status), (dueAt), (institutionId). Optional RLS for institution tables as defence-in-depth.

## B6. Auth and role redirects (unchanged principle)

Login returns `{ user, role, homePath, permissions }` and sets httpOnly cookies (access 15 min, rotating refresh). Client `router.replace(homePath)`. `/dashboard` server-redirects by role. Each route group layout enforces `requireRole`; wrong role redirects to its own home. API guards (`JwtAuthGuard`, `PermissionsGuard`, `ScopeService`) do the real enforcement. Lockout, forced first-login password change, reset by email, session revoke. Playwright acceptance per role: landing URL and sidebar contents.

## B7. Reliable side effects: outbox pattern

External effects (email, any future API push) cannot join a Postgres transaction. So:

```
DB transaction: business change + AuditEvent + OutboxEvent
        ↓ (commit)
Worker polls OutboxEvent (SKIP LOCKED) → renders notification → sends → marks delivered
        ↓ failure
Retry with backoff; visible in admin delivery log
```

Hackathon version: one `OutboxEvent` table and one worker loop; no message broker beyond BullMQ for scheduling.

## B8. Alerts, recipients, escalation engine

- `AlertRule` → **recipient resolver** (by role + scope, or by assignee, or the institution) → channels (IN_APP always; EMAIL per template; SMS adapter off by default). Mandatory vs optional is a flag on the rule; users may mute only optional ones (simple `NotificationPreference` row; MVP can ship with mandatory-only).
- **Scanner** (BullMQ repeatable; every 15 min plus daily morning IST run): SQL finds items crossing each rule offset, creates `Alert` with `dedupeKey = rule:entity:level:dueDate`, enqueues outbox events. Idempotent by the unique key.
- **Escalation job**: for open alerts past the matrix threshold, create/advance the `Escalation` record, alert the next role in scope, audit it. Closing rules per A6.
- Admin-only "Run scanner now" button calls the exact same job function (used for demos, never to fake alerts).

## B9. Feature modules (scope per module)

1. **Admin/config:** users, regions, holidays, matrix import (CSV/XLSX → draft `ConfigVersion` with validation report → publish), template/routing/SLA/escalation editors with a "route this sample application" simulator, **go-live checklist** (blocks publishing until required config exists and SMTP test passes).
2. **Institutions and intake:** CRUD, CSV/XLSX bulk import with row-level validation preview, invitation flow. Import is the real data path for existing institutions, applications, undertakings and lease records.
3. **Applications/scrutiny:** submission with checklist-driven documents, worklists (server-side pagination/filter/sort), detail view with timeline from audit events, observations, return/resubmit, generated case summary PDF labelled "system-compiled summary, not a decision".
4. **EVC (simplified):** queue → select date → select committee → block double-assignment of a committee on the same date → schedule → mark completed with report upload; delay detection by scanner; new-institution flag uses its own timeline. No calendar/availability/travel engine.
5. **Undertakings and compliance:** created at decision time or imported; institution uploads evidence; verifier queue (permission held by regional officer in MVP) verifies or rejects with reason; rejection reopens.
6. **Lease/document validity:** record validity, derived EXPIRING/EXPIRED status from alert-rule windows, renewal upload and verification, history retained.
7. **Dashboards:** role-based, from SQL aggregates; widgets: status by stage, ageing buckets, SLA breach rate, workload per officer, EVC queue/delays, undertakings due/overdue, documents expiring, escalations by level; every widget clicks through to the filtered worklist that produced it.
8. **Reports:** pendency, EVC status, undertaking compliance, document validity, overdue, region performance; export CSV/XLSX/PDF.
9. **Audit:** `AuditService.record()` in every transaction (actor, role, action, entity, before/after, reason, IP); viewer with filters and per-application timeline; CSV export. Hash chain is an optional hardening item (Phase 8), not core.

## B10. "Real-time" definition

Near-real-time via TanStack Query: refetch on window focus, **30-second polling** on dashboards and worklists, manual refresh button, and cache invalidation after every mutation. A "last updated hh:mm:ss" stamp on dashboards. No WebSockets/SSE in the MVP (documented as a future upgrade). State this plainly in the pitch.

## B11. Decision-support layer ("case intelligence"), rule-based first

No AI approving or rejecting anything. Phase 7 adds rule/SQL-based insights, all explainable and all linking to the underlying cases:
- Bottleneck view: average time per stage, EVC average delay, region with highest pending share, officer workload imbalance.
- Risk indicators: application at risk of breaching SLA (elapsed share of clock above a configurable threshold), repeated deficiency returns, long-pending cases, documents expiring soon without a renewal submitted.
- Each indicator shows the rule that produced it.

**Optional assistive AI (only after the core demo path is green; a stretch, not a dependency):**
- Document field extraction (e.g. lease start/expiry dates) that **pre-fills a form an officer must confirm**; the confirmed values, not the AI output, are the record. Store the extraction as a suggestion with a model/version stamp.
- Case summarisation for the officer from structured fields (counts of documents, observations, returns, EVC status, pending undertakings).
- Natural-language analytics is *not* in scope. If ever added, it maps questions to a fixed catalogue of vetted report queries rather than generating free SQL.
- Note for pitching: "AICTE" stands for All India Council for Technical Education, and the problem statement does not ask for AI; position this as workflow automation plus rule-based decision support, with assistive AI as an optional extension.

## B12. Integration (stated honestly)

`IntegrationAdapter` interface (`fetchInstitutions`, `fetchApplications`, `pushStatus`). MVP implementation: **file-import adapter** (CSV/XLSX with a published column spec and validation). Present it as "import-based integration prototype with an adapter-ready architecture", not as live AICTE integration. A real API adapter is a documented extension point pending authorised access and specs. No mock adapter is shipped.

## B13. Security baseline (right-sized)

httpOnly Secure cookies, bcrypt cost 12, rate limiting on login/upload, Helmet, CORS allowlist, CSRF protection (SameSite + custom header), validation everywhere, parameterised queries, secrets only from env, MIME/size allowlist and sha256 on upload, short-lived signed download URLs (or API-streamed downloads with a permission check for the local-disk driver), role/scope checks on every file access. Deferred: ClamAV, field-level encryption, KMS rotation (documented as roadmap).

## B14. Deployment (minimum moving parts)

Pick one before Phase 1 and deploy an empty site in Phase 0:
- **Option 1 (recommended):** one small VPS, `docker compose` with web, api, worker, postgres, redis, caddy (auto-HTTPS), local volume for files and nightly `pg_dump`.
- **Option 2:** managed split (web on Vercel; api, worker, Postgres, Redis on Railway/Render) with an S3-compatible bucket for files.
Health endpoints `/health` and `/ready`, structured logs, Sentry optional. CI: typecheck, lint, unit tests, API e2e against a Postgres service container, Playwright role tests, the no-demo-data check, build.

## B15. Bootstrap (the only seed)

`pnpm bootstrap` (idempotent): migrate, upsert roles and permissions from `packages/shared`, create the first ADMIN from `BOOTSTRAP_ADMIN_EMAIL/PASSWORD` (forced password change), initialise storage. Nothing else. Regions, templates, rules, institutions and applications enter only through the UI/imports.

---

# PART C — DELIVERY

## C1. Phases (exit criteria are testable; durations are estimates, scale to your deadline)

| Phase | Scope | Exit criteria |
|---|---|---|
| 0 Domain freeze + foundations | Rule Matrix v1 with sources, authority matrix, SLA policy, escalation matrix, demo journey script (C2); repo, CI, compose, empty site deployed | Matrix reviewed; every rule VERIFIED or marked UNVERIFIED; empty site reachable by URL |
| 1 Auth, RBAC, scopes, audit | Login, sessions, 7 roles, ScopeService, AuditService | Playwright: each role lands on its home; cross-region and cross-institution reads denied |
| 2 Config and intake | Versioned config, matrix import, routing simulator, institutions + import, go-live checklist | Admin configures everything via UI; publishing blocked until checklist green |
| 3 Workflow core | Applications, classification+routing trace, stage engine, SLA clocks, returns/resubmission, documents, worklists | Table-driven transition tests green; Only-Scrutiny flow end to end |
| 4 EVC | Queueing, scheduling, committee, completion, delay detection | Scrutiny+EVC flow end to end |
| 5 Decision, undertakings, validity | Recommendation, decision, undertakings, evidence, verification, lease/doc renewal | Approved-with-undertaking lifecycle incl. rejection and renewal |
| 6 Alerts and escalation | Outbox, scanner, notifications, escalation lifecycle, delivery log | Due-soon item produces alert + real email; overdue item escalates per matrix; resolve closes escalation but keeps history |
| 7 Dashboards, reports, insights | Role dashboards with click-through, exports, rule-based risk/bottleneck views | Every widget count equals its drill-down list count |
| 8 Hardening | Security pass, indexes, a11y, backup restore test, optional audit hash chain, optional assistive AI | Acceptance checklist (C3) green |
| 9 Demo prep | Operate the system through the journey in C2 on the deployed site; backup recording; slides | Two clean rehearsals |

## C2. Single demo journey (design the system around this)

Institution XYZ submits a new application → system classifies and routes (routing trace visible) to Scrutiny → EVC → SAC/SHC *(or whichever the verified matrix says for that type)* → scrutiny officer returns it for a deficiency with reason → institution resubmits (clocks pause/resume visibly) → scrutiny completed by the officer → EVC enters queue automatically → EVC scheduled → committee uploads report and recommendation → competent authority records decision "approved with undertakings" → institution submits compliance evidence → verifier verifies → a lease document nears expiry → reminder → becomes overdue → escalation to the next level → dashboards update → audit trail shows every step.

Run it as real users on the deployed site with real rows (imported institutions; due dates entered close to today; "Run scanner now" for the time-based part).

## C3. Acceptance checklist

- [ ] No mock/dummy/faker/DemoContext in app code (CI grep).
- [ ] Fresh DB + bootstrap → every dashboard shows zeros with empty states.
- [ ] Rule Matrix rows loaded as config carry `sourceRef`; UNVERIFIED rows not published.
- [ ] Each role lands on its own home route; forbidden routes redirect; forbidden API calls return 403.
- [ ] Region, institution, assignment and committee scope tests pass.
- [ ] All verified templates run end to end; return/resubmit tested under both resume policies.
- [ ] Application keeps its config snapshot after admin changes config.
- [ ] Scanner idempotent; alert and escalation lifecycles behave per A6.
- [ ] Outbox: killing the worker mid-send and restarting does not lose or duplicate notifications.
- [ ] Real email delivered on the deployed site.
- [ ] Dashboard counts equal drill-down counts.
- [ ] Backup restored once successfully.
- [ ] Public HTTPS URL up; README gets a stranger running locally in 10 minutes.

## C4. Guardrails against repeating the first build's problems

- `scripts/no-demo-data.sh` in CI; ESLint ban on importing mock/fixture paths outside `__tests__`.
- Dashboards are API aggregates only; the browser never computes them.
- One source of truth for role → home path in `packages/shared`, with a unit test.
- E2E suite runs on every PR against a freshly bootstrapped DB.

## C5. Prompt for Antigravity

> Build exactly per PLAN v2, one phase at a time, starting with Phase 0 (do not write application code until the Rule Matrix and authority/SLA/escalation sheets exist in `docs/rule-matrix/`). No mock data, DemoContext, faker, or hard-coded numbers. After each phase run typecheck, lint, tests and the no-demo-data script, start the stack and verify by actually using it, and report what you verified. Do not start the next phase until the current phase's exit criteria pass. Ask before changing the data model, role matrix or any rule marked UNVERIFIED.

## C6. Open decisions (answer in Phase 0)

1. The verified application types in scope and their stage sequences (from the APH).
2. Authority matrix: who may do/decide what at each stage.
3. Stage SLAs, calendar vs working days, and the resume policy per stage.
4. Escalation levels and recipients.
5. Which undertakings and validity documents are in scope, with validity and renewal rules.
6. Deployment target and domain; SMTP provider.
7. Whether to include the optional assistive-AI stretch.
