# RAASPAL — Project Index

> Read this file first to understand the current project state before starting any task.
> For change history and decisions, see [[history]].
>
> **This vault has one branch, `main`, shared by every developer.** `git pull` before you start
> writing and again right before you push — see [README](README.md). Two people appending to these
> files on the same day will conflict otherwise.

---

## Project Overview

**RAASPAL AI Robot Solution & Proposal Generator**

An internal RAASPAL platform where the team uploads a customer survey form, AI extracts requirements, the backend loads robot data from PostgreSQL, AI recommends robot solutions, and AI generates a proposal from the selected option. The platform also syncs robot telemetry and delivers automated monthly performance reports to customers by email. The initial MVP is complete; the project is now building post-MVP features.

---

## Service Map

| Service | Location | Stack |
|---|---|---|
| Backend API | [[RaasPal-Internal-Ops-backend]] | Spring Boot 3.4.5 · Java 21 · PostgreSQL |
| Frontend Web (console) | [[robot-recommendation-web-raaspal]] | Next.js 16 · TypeScript · Tailwind v4 |
| Frontend Web (inventory) | [[raaspal-rims]] | Next.js 16 · own component library · **fully wired to the backend** |
| Database | Supabase PostgreSQL (cloud) | Session-mode pooler · Flyway migrations |
| AI Provider | [[ClaudeAiService]] / [[MockAiService]] | Anthropic Claude API · auto-selects by key |
| Hosting (backend) | Render → **migrating to AWS Lightsail** | 8 GB / 2 vCPU / Singapore · Docker + Nginx |

---

## Current Status

### Deployment Snapshot — verified 2026-08-31

> Measured on 2026-08-31 against the live Render API and the three working trees, not
> transcribed from memory. Everything here can drift the moment someone deploys, so
> re-check rather than trust it. The sections below this one describe features and
> decisions; this one describes *what is actually running*.

| Repo | HEAD | Tree | Deployed |
|---|---|---|---|
| [[RaasPal-Internal-Ops-backend]] | `cf04b84` on `main` — PR #6 merged 2026-09-11 (case report + Solution AI + PM planner V39) | clean | ⚠️ **NOT deployed** — Render still runs the pre-case-report build; Lightsail needs `bash deploy/deploy.sh` + `MONDAY_API_TOKEN` (see [[history]] 2026-09-11b) |
| [[robot-recommendation-web-raaspal]] | `888a621` CM report one-page fit | clean | ❓ Vercel state not verified |
| [[raaspal-rims]] | `44ed937` store-room statuses + packaging | clean | ❓ Vercel state not verified |

**Live data facts** (Supabase — one database, shared by local development and production):

- Highest applied migration is **V38 = case-report tables** (`V38__add_case_report_tables.sql`, the full
  parked V34 design: `case_ticket`, `case_ticket_update`, `case_ticket_status_history`, `case_ticket_override`,
  `case_report_definition`, `case_report_recipient`, `case_branch_alias`, `case_robot_location`,
  `case_report_run`). **Applied to production 2026-09-11.**
- ⚠️ **V38 collision, resolved 2026-09-11 — read this before touching `feat/re-kpi-dashboard`.** That
  branch's `V38__add_case_ticket_sync.sql` also creates `case_ticket`. It was written against a vault copy
  that had not been pulled, and the case-report V38 reached production first. **Decision: production keeps
  the case-report V38; the KPI branch reworks after the case-report merge lands.** Concretely, on that
  branch: (1) its V38 becomes `ALTER TABLE case_ticket ADD COLUMN` for the columns the case-report table
  lacks — `service_line`, `ticket_no`, `issue_level`, `close_date`, `is_closed`, `serials_normalised` — plus
  `CREATE TABLE case_ticket_sync_run`, numbered **V40** (V39 went to the PM planner in PR #5); (2) its
  V39–V41 become **V41–V43**; (3) its local Docker DB is rebuilt. ⛔ Until then, **do not start that branch's backend against production** — Flyway will see prod's
  v38 is "add case report tables", not "add case ticket sync", and refuse to start. Two overlapping date
  columns need one owner: case-report `re_action_date` vs KPI `action_date`.
- **V39 is taken — `V39__add_pm_planning_tables.sql` (PM 52-week planner), on `main` since PR #5/#6, applies on
  the next deploy. Next free migration: V40.** Coordinate — the KPI rework will claim V40–V43.
- `robot_inventory_temp` holds 92 rows — `IN_STOCK=45`, `DEMO=47`.
- **`UNDER_REPAIR` and `RETURNED_FROM_CUSTOMER` are live but unused.** The states work end to end;
  nobody has set one yet, so the catalogue currently renders two bands rather than four.
- `packaging` exists on every row and is null on all of them — nobody has recorded it yet.

⚠️ **Local development and production share one Supabase database.** Starting the backend locally
applies any pending migration to production. This has caused a real outage — see the V34 entry in
[[history]], where a `DROP COLUMN` took down every inventory query while Render still ran the old code.
Expand and contract: add, deploy, *then* drop.

⚠️ **A committed `V*.sql` is already applied — never edit one in place.** Flyway checksums applied
migrations and refuses to start on a mismatch, so an edit breaks the next deploy with no local symptom.
Add a new migration instead. Learned twice: V32 → V33, and again with V37 on 2026-08-31.

### Robot Inventory Management System (RIMS) — ✅ RUNNING ON REAL DATA

A second Next.js app for **inventory staff** (stock levels, reorder alerts, purchase orders), moved into this
workspace on 2026-08-10 from `D:\Work\SoftwareWorkSpace\`. Own repo (`swanhtetaung01/RaasPal-RIMS`), own
component set (`authorized-form`, `command-palette`, `parts-table`, `robot-stock-form`), own
`lib/auth.ts` + `lib/rbac.ts`.

**✅ Seed data fully removed 2026-08-14.** Every screen — dashboard, robots, inventory, activity, sidebar,
command palette — reads the backend or shows an empty state. Auth is the platform's own JWT; robot stock lives
in the standalone `robot_inventory_temp` table ([[V30__add_robot_inventory_temp]]), deliberately unconnected to
[[robot_units]] so a plain count is honest. Parts change only by recording a movement. `lib/seed.ts`,
`lib/store.ts` and `data/users.json` are now unreferenced and can be deleted.

- **Shared backend, separate frontend.** Inventory becomes a bounded module in the existing Spring Boot app
  (`com.raaspal.robotrecommendation.inventory.*`, own tables, `/api/v1/inventory/**`, Flyway **V28+** — V26
  is still reserved for partner live status). Same precedent as [[cvte]] / [[partner]] / [[cm]]: one app, hard
  internal boundaries, FKs pointing *into* existing tables only so the module stays extractable.
- **Why share at all:** auth already exists, [[Robot]]/[[RobotSpec]] is the natural parent of a parts catalogue,
  CM reports consume parts (a stock movement waiting to happen), and [[RobotUnit]]/[[Deployment]] says which
  spares matter. Also Supabase free tier caps at 2 projects — a third isn't available.
- **The real work is replacing seed data with API calls**, rewiring `lib/auth.ts` to `POST /api/v1/auth/login`,
  and making `lib/rbac.ts` read backend-issued roles. ⚠️ Client-side RBAC is navigation, **not** enforcement —
  the backend must reject inventory writes from non-inventory users on its own.
- `lib/actions.ts` suggests Server Actions; if the backend is called server-to-server, RIMS needs **no CORS
  entry** and no `NEXT_PUBLIC_API_URL` exposing the API to the browser (tighter than the console app).
- **✅ APPLIED to Supabase 2026-08-11 — schema is at v29.** Verified: 152 `robot_units`, **151 RENT + 1 IN_STOCK**
  (one unit has no active deployment — diagnostic still unrun; it is either junk data or the first real stock robot),
  **99 linked** to a catalogue model (73 Omnie + 20 Phantas exact + 6 Phantas revisions via the family rule),
  **6 versioned**. The 53 unlinked are the models with no `robots` row yet (M50 16, M50 Pro 12, M75 9, M40 8,
  Beetle 3, M50 Pro Roller 2, + M40 Spray / M50 Blend / one blank).
- **Phase 0 done 2026-08-11** — `INVENTORY_STAFF` added to [[Role]] (no migration: `users.role` is `VARCHAR(20)`,
  no CHECK), `/api/v1/inventory/**` gated to `ADMIN`+`INVENTORY_STAFF` in [[SecurityConfig]]. **95 tests pass.**
  ⚠️ `GET /api/v1/auth/me` and `POST /api/v1/users` **already existed** — Phase 0 was far smaller than planned.
  ⚠️ [[UserController]] has **no `@PreAuthorize`**, so any authenticated user can create users, including an
  inventory account minting itself an admin. Pre-existing, but it matters once there is more than one account.
- **Catalogue reality (queried 2026-08-11):** 8 models, all with specs. **VERIFIED = AGIBOT C5, KEENON C40,
  Gausium Omnie.** The datasheet is **Gausium-only**, so importing it alone leaves the two non-Gausium VERIFIED
  models without specs. **Decision: import Gausium now, other brands later** — safe only because `robot_specs`
  stays live. Drop-gate query is in [[history]] 2026-08-13.
- **Schema originally written 2026-08-11 — [[V28__add_inventory_and_robot_lifecycle]] + [[V29__add_cleaning_specs_and_display_specs]].** V28: `robot_units` gains `status` (**IN_STOCK | RENT | SOLD**), `version`, `robot_type`,
  `robot_id`, `location`; new `inventory_items` + `stock_movements` (append-only ledger, `quantity_change`).
  V29: `robot_specs_cleaning` (101 columns from the real Gausium datasheet) + `robot_display_specs` (JSONB,
  admin-curated `[{label,value,unit}]`). ⚠️ **Local dev and prod share one Supabase DB — starting the app locally
  applies these to production.** `pg_dump` → rehearse on local Docker Postgres → apply. `mvn test` cannot validate
  them (`spring.flyway.enabled=false`, H2 `create-drop`).
- **A SOLD robot keeps its active deployment** — RAASPAL monitors and reports on it after ownership transfers.
  So `status='IN_STOCK'` alone excludes both rented and sold units, and RIMS needs no special-casing.
- **Two store-room-only states added 2026-08-28 — [[V37__add_robot_stock_lifecycle]], applied and deployed.**
  `UNDER_REPAIR` and `RETURNED_FROM_CUSTOMER` join [[RobotUnitStatus]] for `robot_inventory_temp` only. They
  widen a *new* `isStockRoomStatus()`; `isWarehouseVisible()` is deliberately untouched, because it guards
  [[RobotUnitService]] in five places and a `robot_units` row must never take a shelf-only state. `IN_STOCK`
  and `DEMO` keep their stored values and are merely relabelled **New Stock** and **Demo Unit**, so not one
  existing row was rewritten. V37 also widens `status` to `VARCHAR(32)` (`RETURNED_FROM_CUSTOMER` is 22 chars)
  and adds `packaging VARCHAR(16)` (`BOX | UNBOX`, nullable), deliberately **outside** the identity index.
- **Pending:** its own `CLAUDE.md`; three uncommitted files predating the move.

### Hosting — 🚧 MIGRATING Render → AWS Lightsail

Backend moving from Render to a **Lightsail 8 GB instance** (Singapore, US$44/mo, 2 vCPU / 160 GB / 5 TB),
Docker + Nginx + Let's Encrypt, Supabase retained as the database for now. Deployment files designed but
**not yet written** (`deploy/` — compose, env example, nginx conf, deploy script, runbook).

- **Domains:** `api.raaspal.com` **A** → Lightsail static IP; `app` / `rims` **CNAME** → `cname.vercel-dns.com`.
  Product-neutral API name because one backend now serves two frontends. All three under `*.raaspal.com` makes
  the httpOnly `raaspal_token` cookie shareable at `Domain=.raaspal.com` → SSO (needs a new `/api/v1/auth/me`).
- ⚠️ **Copy `PARTNER_JWT_SECRET` and `JWT_SECRET` off Render verbatim** — regenerating breaks live PCS partner
  tokens and logs out all staff. Render is the only place some production secret values currently exist.
- ⚠️ **Parallel-run limits:** `DB_POOL_MAX=5` on Lightsail (Supabase caps at 15 total), and keep the schedulers
  **off** there — two instances hitting the delivery cron at the same minute could double-send customer emails.
- ⚠️ **Never set `server.forward-headers-strategy`** — `ForwardedHeaderFilter` strips `X-Forwarded-*`, which
  [[ClientIpResolver]] reads directly (rightmost). Nginx `$proxy_add_x_forwarded_for` is already correct.
- **No data migration needed:** the DB is Supabase (shared) and Render's disk was ephemeral, which is exactly
  why CM signatures were stored in-row. Uploads become genuinely persistent for the first time on Lightsail.
- **DECIDED 2026-08-10 — stay on Supabase free.** Measured baseline: **55 MB of the 500 MB** quota (11%), of
  which only ~27 MB is application data — the other ~28 MB is Supabase's own `auth`/`storage`/`realtime`
  schemas, for features used at 4 MAU / 0 GB / 0 messages. Telemetry is the big table but it is *predictable*
  (626 B/row × a known 152-robot fleet ≈ 14 MB/month). **Runway ≈ 20–30 months.** Storage is not the
  constraint and must not drive the hosting decision.
- ⚠️ **The one unmeasured variable is `cm_reports`** — 1 row, 80 kB, which is mostly index overhead, so the real
  per-report cost is still unknown. At ~150 KB/report the runway is ~20 months; at the 512 KB signature ceiling
  it is **~5 months**. A 6× swing from one design choice. Re-measure after ~10 real reports:
  `SELECT count(*), pg_size_pretty(pg_total_relation_size('cm_reports')) FROM cm_reports;`
  **Note the in-row base64 decision was forced by Render's ephemeral disk — that constraint disappears on
  Lightsail**, so moving signatures to disk becomes available exactly when it might be needed.
- **Deferred:** moving Postgres onto the Lightsail box (retires the 500 MB ceiling *and* the 15-connection
  pooler cap) — deliberately **after** the host migration stabilises, so one change is isolated. If it happens,
  do it **before inventory ships**: migrating is only ever easier than it is today, and once warehouse staff
  depend on it daily the maintenance window gets much harder to schedule.

### Backend — COMPLETE ✅

All backend steps are done and connected to Supabase:

- JWT authentication ([[AuthTokenFilter]], [[UserDetailsServiceImpl]])
- File upload and storage (`uploads/customer-surveys/`)
- AI requirement extraction — [[RequirementExtractionService]]
- Robot catalog retrieval from PostgreSQL — [[RobotCatalogData]]
- AI recommendation engine — [[RobotRecommendationAiService]]
- AI proposal generation — [[ProposalGenerationAiService]]
- [[ClaudeAiService]] — real Claude AI (auto-activates when `ANTHROPIC_API_KEY` is set)
- [[MockAiService]] — silent fallback when no API key (`@ConditionalOnMissingBean`)

### Frontend — COMPLETE ✅

Next.js 16 frontend built end-to-end:

- Login with JWT auth guard via [[proxy.ts]]
- Upload → Extract → Recommend → Propose full flow (with [[FlowStepper]] pipeline indicator)
- shadcn/ui components, Zustand state, TanStack Query
- API layer in [[lib/api.ts]] wired to real backend
- i18n via next-intl (English + Thai; Chinese removed 2026-07-01)
- **UI/UX "Operations Console" pass (2026-07-22):** sidebar now visible from 1024px (was 1680px),
  shared [[status-badge]]/[[skeleton]]/[[empty-state]] primitives, denser list pages,
  Tools tabs URL-synced (`?tab=`). See [[history]] session 2026-07-22.
- **Report automation promoted (2026-07-23):** report delivery is now a top-level **Reports**
  sidebar item (2nd, after Team Dashboard) with its own `/reports` hub (tabs: Manage automation /
  Email customers / Report preview, moved out of Tools). Team Dashboard reframed around delivery:
  report KPI tiles, MonthlyDeliveryCard (run status + coverage), RecentDeliveries feed; AI solution
  flow demoted to a shortcut. Tools now = Monitor/Customers/Robots. See [[history]] session 2026-07-23.
- **Reports hub split into sub-sections (2026-08-05):** the four existing tabs are all *robot
  performance* reporting, so they sit under one group and **Corrective maintenance** is a sibling
  group (tabs: New report / History). The group is **derived from `?tab=`** via `TAB_GROUP` in
  [[ReportsClient]] rather than a second query param, so existing `?tab=automation` / `?tab=autoxing`
  links still work. See [[history]] session 2026-08-05.

### Database — CONNECTED ✅

- Supabase PostgreSQL connected via session-mode pooler (port 5432, `sslmode=require`)
- Flyway migrations ran successfully
- Credentials stored in [[application-local.properties]] (gitignored)

### CVTE C3 Online/Offline Status — NEW ✅ (separate module, unconfigured)

A small standalone add-on tracking CVTE C3 robot online/offline status via the Kava Open Gateway API.
**Deliberately kept separate from [[Robot]]/[[RobotSpec]]** — own package `com.raaspal.robotrecommendation.cvte.*`,
own table `cvte_devices`, own `/api/v1/cvte/devices*` endpoints. Not yet wired to a real Kava account
(`CVTE_KAVA_APP_ID`/`CVTE_KAVA_APP_SECRET`/`CVTE_KAVA_BASE_URL` are unset — `isConfigured()` guards return
a clear error until then). Frontend page at `/cvte` ("CVTE C3 Status" in sidebar) plus a compact
[[CvteStatusSummary]] widget on the Team Dashboard. See [[history]] session 2026-06-08 for full details.

### AutoXing Delivery Report — on-demand preview ✅ (unconfigured)

New robot brand **AutoXing** (delivery robots) added as an **on-demand delivery report preview only** —
no persistence, no [[TelemetryAdapter]] sync, no scheduler. Own package
`com.raaspal.robotrecommendation.telemetry.adapters.autoxing.*`: `AutoxingApiClient` (token sign
`MD5(appId+timestamp+appSecret)` + AppCode header; `/robot/v2.0/{id}/state`, `POST /statis/v2.0/task`,
`/task/v3/{id}`), in-memory `AutoxingAuthService` (600s non-refreshable token, re-sign per run — deliberately
NOT the DB-backed Gausium pattern), `AutoxingReportService` (≤30-day window, aggregates delivery/call/charging
daily stats + live status, re-auth-and-retry). Endpoint `GET /api/v1/autoxing/report/preview?robotId=&from=&to=`.
Frontend: **AutoXing report** tab in the Reports hub (`/reports?tab=autoxing`, [[AutoxingReportPanel]]) rendering
[[DeliveryReportView]] — a full RAAS PAL executive report for delivery robots (Part 1 summary, Part 2 rings +
daily volume chart, Part 3 daily activity table, recommendations). **Customer/site auto-resolve** from AutoXing
directories (businessId→business name, buildingId→building name, areaId→area/floor; note `data.lists` plural for
business/building vs `data.list` singular for robot/area). Optional robot name/model overrides (AutoXing has no
real model — only category `餐厅`). **Live status is shown in the panel only, never in the period report.**
Config `app.autoxing.api.*` (`AUTOXING_API_APP_ID/APP_SECRET/APP_CODE`, base-url defaults to global
`apiglobal.autoxing.com`). **Auth confirmed working live:** requires global host + `Authorization: APPCODE <code>`
(Alibaba Cloud API Gateway; `app.autoxing.api.appcode-scheme` defaults **true**). Statistics schema verified
(`count/date/duration(ms)/mileage(m)/robot`); report populates (robot returned 861 tasks in a 20-day test).
Still needs the updated backend deployed to Render + `AUTOXING_API_BASE_URL` unset/global. See [[history]] session 2026-07-23.

### Partner Middleware API (PCS) — ✅ (built, needs deploy + onboarding)

A read-only, **OAuth-authenticated** surface (`/api/partner/v1`) so a distributor/service partner such as
**PCS** can pull data for only the robots it services. Own package `com.raaspal.robotrecommendation.partner.*`,
tables `partners` / `partner_api_keys` + `deployments.partner_id` (V22).

- **Scoping lives on `deployments.partner_id`** — *not* on the customer or the robot. One partner can service
  robots across many customers; a robot's serial reaches the partner automatically via its deployment. The
  partner id always comes from the authenticated key, **never** a request parameter.
- **Auth — OAuth 2.0 client credentials (V24)** — `client_id` + `client_secret` → `POST /api/partner/v1/oauth/token`
  → **24-hour** bearer → [[PartnerJwtAuthFilter]] → [[PartnerPrincipal]]. Credentials accepted form-encoded (per RFC
  6749), as JSON, or via HTTP Basic; every rejection is an identical `invalid_client` so client ids cannot be
  enumerated. Only a SHA-256 hash is stored (secret shown once at mint). **Partner tokens are signed with their
  own secret** ([[PartnerTokenService]], `app.partner.jwt.secret`) — [[JwtUtils]] carries no audience or type
  claim, so sharing it would make staff-vs-partner separation hold only by luck. The credential is
  **re-checked on every request** via the `apiKeyId` claim, so revoking takes effect immediately rather than
  when the token expires. [[ApiKeyAuthFilter]] is retired (kept one release, no longer a bean).
- **Two separate security chains** — [[PartnerSecurityConfig]] `@Order(1)` matches `/api/partner/**`
  (stateless, CSRF off, no CORS); [[SecurityConfig]] JWT chain `@Order(2)` catch-all. ⚠️ The `@Order` **must be
  on the `@Bean` method** — putting it on the class broke the Render deploy.
- **Endpoints** — `GET /me`, `GET /robots`, `GET /robots/{serialNumber}/task-reports` (paged; `month=YYYY-MM`
  **or** `from`/`to=YYYY-MM-DD` day-range, Asia/Bangkok). Unknown vs not-owned serials return an **identical
  404** (no existence probing); page size capped at 100.
- **Reads come from our own `robot_task_reports`** — never a live Gausium call, so brand credentials stay
  internal and it works for any synced brand. Freshness depends on the telemetry sync below.
- **Admin** — `/api/v1/partners` ([[PartnerAdminController]]): create/enable/disable partners, mint/revoke keys,
  assign a deployment or **bulk-assign** many. UI: Tools → **Partners** ([[PartnersPanel]]) with copy-once key
  display and a searchable multi-select for bulk assignment (built for PCS's ~70 robots).
- **Hardening (V23)** — **per-key rate limit** (Caffeine fixed window, default 120/min → 429 + `Retry-After`,
  `X-RateLimit-*` headers; budget is **per instance**), **access audit** (`partner_api_access_logs`: who fetched
  what/when/with which key/outcome, incl. rejected attempts; nightly retention prune, default 90 days;
  `GET /api/v1/partners/{id}/access-log`), **key expiry** (`expires_at`, nullable = never; optional
  `expiresInDays` on mint; UI shows expiry, *Expired* vs *Revoked*, and an *Expiring soon* nudge), and strict
  **`month` validation** (a typo is now a 400, not a silently empty result). Config `app.partner.*`.
- **Production hardening, round 2 (2026-07-30)** — three defects found by auditing rather than by failure:
  - **Filter scope.** All partner filters were `@Component`s, and Spring Boot registers every `Filter` bean at
    `/*` **independently of `securityMatcher`** — so each also ran on every staff request, Swagger page and
    health ping. The audit filter (which has no partner-path check) was writing a row per non-partner request:
    an insert apiece on the free tier, each with `partner_id = NULL`, indistinguishable from a failed partner
    auth. Not a bypass (container copies sort after `springSecurityFilterChain`), but it buried the signal the
    table exists for. Fixed with disabled `FilterRegistrationBean`s in [[PartnerSecurityConfig]].
  - **Unthrottled token endpoint.** It is `permitAll` and the rate limiter meters per *authenticated* partner,
    so an anonymous loop bought a credential lookup + an audit insert per request, unbounded. Now metered per
    IP (default 20/min). Not about guessing secrets — 256 bits is unreachable — but about the DB work and audit
    growth an anonymous caller can drive.
  - **Spoofable client IP.** `X-Forwarded-For` was read **leftmost**, which is the caller's own claim, so the
    new throttle could have been reset by varying a header. Now rightmost ([[ClientIpResolver]]), the value the
    proxy appended. **Mutation-verified:** restoring leftmost makes the forgery test fail 429 → 401.
  - **Placeholder-secret guard.** [[PartnerSecretStartupCheck]] refuses to boot when `app.partner.jwt.secret`
    is still the placeholder committed to [[application.properties]] — that key is public, and it signs the
    token carrying the partner id all scoping depends on. `app.security.allow-placeholder-secrets=true` in
    [[application-local.properties]] keeps local dev working; a staff-secret placeholder is logged, not fatal,
    since that service is already deployed.
- **Tests** — **65 partner tests** (12 auth + 15 scoping + 13 hardening + 14 OAuth + 2 filter scope + 3 token
  throttle + 6 secret guard); 83 in the suite. Scoping was **mutation-verified** against a cross-tenant leak,
  as were token revocation and the IP fix. Earlier hardening tests caught a real bug where the audit recorded
  `partner_id = NULL` for successful requests because Spring clears the SecurityContext before an outermost
  filter resumes.

See [[history]] sessions 2026-07-25, 2026-07-30. **Go-live:** ⚠️ **set `PARTNER_JWT_SECRET` on Render — the app
now refuses to start without it**; redeploy backend, deploy frontend to Vercel, then create PCS → mint
credential → bulk-assign its deployments. **Pending:** `ADMIN`-only admin endpoints.

### Telemetry Sync Pipeline — ✅ (built, scheduler off by default)

[[TelemetrySyncScheduler]] keeps `robot_task_reports` current so the report pages and the partner API serve
fresh data. Previously **nothing** did this — `syncRange`/`syncYesterday` existed but had no callers, so data
only landed via manual sync.

- Daily cron (default **00:00 Asia/Bangkok**, changed from 03:00 on 2026-07-30 at the user's request),
  `@ConditionalOnProperty(app.telemetry.sync-enabled)` — off by default in code but **`TELEMETRY_SYNC_ENABLED=true`
  is set on Render**, along with the `GAUSIUM_API_*` credentials, so the fleet syncs nightly in production;
  non-overlapping (`AtomicBoolean`) and never throws.
- **Look-back window** (default 3 days) rather than "yesterday only" — brand APIs backfill late tasks and
  robots go offline; safe because sync is **idempotent** on external task id.
- `syncAllActive` is driven by one fetch-joined query of active deployments (no N+1), **batch dedup** (one `IN`
  query + one `saveAll`), per-robot error isolation, returns a `SyncSummary`. Deliberately **not**
  `@Transactional` — a multi-minute fleet sync must not pin a DB connection.
- Unconfigured/unsupported brands are skipped **once per brand** via `TelemetryAdapter.isConfigured()` +
  `TelemetryAdapterRegistry.findAdapter()`, so it is safe to enable before Gausium credentials exist.
- Manual/backfill: `POST /api/v1/telemetry/sync-all?from=&to=`.

Config `app.telemetry.sync-enabled|sync-cron|sync-zone|sync-lookback-days` (`TELEMETRY_SYNC_*`).
**Go-live:** set the `GAUSIUM_API_*` keys → verify via `sync-all` → then `TELEMETRY_SYNC_ENABLED=true`.
See [[history]] session 2026-07-25.

### Telemetry — Robot Task Report Sync — Phase 1 ✅ (unconfigured)

New `com.raaspal.robotrecommendation.telemetry.*` + `robotunit.*` packages (Adapter Pattern), separate
from [[Robot]]/[[RobotSpec]] and from [[cvte]]. [[RobotUnit]]/[[Deployment]] (V11) link a physical robot
(by serial number) to a customer site. [[RobotTaskReport]] (V12 + V13) stores per-task cleaning data
synced from brand APIs in one shared table with nullable brand-specific columns. [[TelemetryAdapterRegistry]]
picks a [[TelemetryAdapter]] by brand; [[GausiumAdapter]] is the first implementation, backed by
[[GausiumApiClient]] (OAuth + paginated V2 List Robot Task Reports) and [[GausiumOAuthService]] (DB-backed
token cache/refresh via `gausium_oauth_tokens`). [[TelemetrySyncService]] loops active [[RobotUnit]]s,
fetches reports per brand, and dedupes via `existsByExternalTaskId`. **Wired to a live Gausium account and
running in production** — verified 2026-08-10 (see the production-scale figures below). See [[history]]
session 2026-06-12 for the original build.

### Corrective Maintenance (CM) Report Auto-Creation — ✅ (built 2026-08-05, V27 applied)

Replaces the manual "copy a Monday.com ticket → Excel → print PDF" loop for cleaning-robot service reports.
Paste the ticket → AI extracts the nine fields → operator reviews and corrects → attaches both signature
photos → `window.print()` renders the exact Thai paper form (`รายงานการซ่อมบำรุงแก้ไข`). Saved and searchable.

- **Package `com.raaspal.robotrecommendation.cm.*`** + `cm_reports` (**V27** — V26 stays reserved for partner
  live status). `/api/v1/cm-reports`: `POST /parse`, plus CRUD and `GET ?q=` (ticket no. / customer / serial).
- **Parse and save are separate endpoints** — an abandoned parse leaves no row; re-parsing never duplicates.
- **AI on `claude-haiku-4-5-20251001`** ([[CmReportExtractionService]], implemented in **both**
  [[ClaudeAiService]] and [[MockAiService]] — they are paired by `@ConditionalOnExpression`, so adding it to
  only one breaks the context). ~$0.009/report. The prompt's rule is **transcription, not authorship**: Thai
  must survive byte-for-byte onto a signed document, so no translating/summarising and `null` over invention.
  The MVP rules are deliberately not reused — the `Needs confirmation` sentinel would print as literal text.
  Failure degrades to an empty draft rather than throwing. [[MockAiService]] does a real Thai-label scan, so
  the feature is usable locally with no API key.
- **Signatures are uploaded per report** (not fixed assets), stored as base64 `data:` URIs **in the row** —
  *not* via [[FileUploadService]], because that writes to local disk and **Render's disk is ephemeral**, which
  would silently break reprints. Client downscales to a 600px long edge; the service enforces a hard 512 KB
  ceiling and an `image/*` check (the server is the boundary, the resize is convenience).
- **Report labels are hard-coded Thai, not i18n** — a signed customer document must not change wording when a
  staff member switches the app to English. Buddhist-era dates via [[thai-date]] (parsed field-by-field, so no
  UTC off-by-one-day). Steps stored newline-separated, numbered at render time.
- Frontend: [[CmReportPanel]], [[CmReportHistoryPanel]], [[CorrectiveMaintenanceReportView]], [[signature-image]].
- **95 backend tests pass** (11 new). V27 verified applied against live Supabase; full API round-trip exercised
  with real Thai data. ⚠️ **Visual fidelity vs the paper form is still unverified** — see [[history]] 2026-08-05.

### RE KPI Dashboard — three monday boards → live KPI page — 🚧 (built 2026-09-08 on `feat/re-kpi-dashboard`, V38–V41 pending on prod, not deployed)

> ⚠️ **Migration numbers in this section are superseded — see the V38 collision note under Deployment
> Snapshot.** V38 went to production as the case-report tables on 2026-09-11, so this branch's V38 must become
> an `ALTER TABLE case_ticket` and — now that the PM planner holds V39 on `main` — its later migrations shift
> to **V41–V43**, with the `ALTER` at **V40**. Do not start this branch against production until that rework
> is done.

Replaces the hand-built RE KPI deck. **Data source: monday.com** (Excel later, deferred). Backend and
frontend branches share the name. Runbook: `RaasPal-Internal-Ops-backend/docs/kpi-local-testing.md`.

- **Boards** (all column ids verified live 2026-09-08): Cleaning Tickets `3451717331`, Delivery Tickets
  `1647612496`, Installation Tickets `3109668017` → `case_ticket` (V38) + `ticket_type`/`action_date`/
  `install_date` (V39), nullable `service_line` (V40). **The S/N is the foreign key across the boards**:
  installations are classified cleaning/delivery by finding their serial on a CM board.
- **Formulas (RE team, 2026-09-08):** 1st Time Install = no CM on the same S/N within 30 days of the
  TimeLine's later date; First Time Fix = no further CM on the same S/N within 14 days; SLA = RE Action
  date within 7 days of Open Date (no close-date column exists). Blank serial → success, reported;
  unclassifiable → fleet total only; no action date → unknown, never a breach.
- **Per-board category filter (V41):** `columns.category` + `include-categories`. Used only on the
  installation board (install-shaped Job Types, provisional); the CM boards count every row — a
  parts-shipment filter matched the deck's cleaning total only by coincidence and was reverted. Left-out
  rows are reported as `excludedByCategory`.
- **Validated against the Jan–Jun 2026 deck:** delivery CM 816 vs 815 and every month within one;
  cleaning Jan/Feb/Mar/Jun within two, **May 133 vs 38 open**; FTF ≈75% vs 72.3%; installs 59 vs 23 —
  **open**. See [[history]] 2026-09-08.
- **API:** `GET /api/v1/kpi/cm-cases?from=YYYY-MM&to=YYYY-MM` (installation + CM counters per month, per
  line, `unclassifiedTickets`, `definitions`, `provisional`), `POST /monday/sync` (background), `GET
  /monday/sync/{status,runs}`, `GET /monday/config`, `GET /monday/boards` (ids only), `GET
  /monday/boards/{id}` (columns + status labels, no rows). ADMIN / RAASPAL_TEAM.
- **Console** `app/[locale]/kpi/{report,utilization,repeat-cost,csat}` (one route each, sidebar dropdown;
  old `?tab=` links redirect): the report has four live tiles (1st Time Install, Total CM, FTF, SLA) and
  PM Complete as a badged placeholder — sourceable from `PM Yip-upload` + the PM contract boards once the
  visits-due rule is known. Utilization / Repeat Cost are still deck constants.
- **CSAT (2026-09-09/10)** has its own page and is *not* on the report — it is a monthly hand tally, not
  a ticket computation. Source: the RE team's four survey workbooks (installation, MA = PM, CM cleaning,
  CM delivery) in `app.kpi.csat.folder`, replaced monthly (S3 later, same `CsatWorkbookSource` interface).
  `GET /api/v1/kpi/csat?from&to`, `/csat/source`, `POST /csat/reload`. **Rule: one survey in one month =
  the month sheet's own Top Box cell (`I17 = AVERAGE(Q10:Q14)`), read by the "Top Box" label, never
  recomputed; totals over months/surveys (no sheet holds them) = all fives over all ratings, the deck's
  way.** Every deck figure reproduces exactly. Top Box only; no mean; no cross-check warnings. No migration.
- **Operational:** scheduler off unless `KPI_MONDAY_SYNC_ENABLED=true`; monday client has 15 s/90 s
  timeouts + one retry (a hung request once parked the sync for 29 min); page cap 500×100 (50×50 truncated
  the delivery archive). A first full sync of three boards is ~5 min and ~150 API calls.
- ⚠️ **STRICT user rule:** never pull ticket rows into a transcript — schema, column names, counts only.
- ⚠️ Local: `sh mvnw` (wrapper not executable, `mvn` not on PATH); Jenkins owns 8080 → API on 8081;
  docker Postgres `raaspal-kpi-pg` on 5433; token lives in gitignored `application-local.properties`.

### Contract Start Date — reports clip to when each robot started — ✅ (deployed 2026-08-20)

A robot deployed mid-month was reporting a full month, counting work done before the customer had it.
`deployments.contract_start_date` (**V33**, nullable) fixes it: [[ReportPreviewService]] drops tasks before
that date, so the first month is clipped and every later month is a normal full month.

- ⚠️ **It is per DEPLOYMENT, not per customer.** [[V32__add_customer_contract_start_date]] put it on
  `customer_profiles` and was wrong — one customer rents or buys different robots at different times
  (IFS has robots across many sites, each with its own start). **V33** moves it to `deployments` and drops
  the customer column; V32 had shipped hours earlier and was verified empty across all 67 customers, so no
  data was lost. V32 could not simply be rewritten because it was already applied in production — Flyway
  validates checksums on startup.
- **Null = report the whole month**, so this is inert until dates are filled in.
- ⚠️ **`deployments.deployed_at` is NOT a substitute** — it is `now()` at registration, so existing values
  record when a robot was typed into the system, not when its contract began.
- ⚠️ **The date is interpreted in `app.reports.business-zone` (default `Asia/Bangkok`), not UTC**, though
  `reportMonth` is bucketed in UTC. Cleaning robots run overnight: 02:00 Bangkok on the start date is
  19:00 UTC the previous day, and that is the customer's work. Both sides of the boundary are tested.
- The report header still reads the month name ("July 2026") — the customer knows their own start date.
- Editing a robot evicts the report cache, or a corrected start date would keep serving the old report.
- Set under **Tools → Robots** (register/edit form). 8 tests (suite **115**). See [[history]] 2026-08-19.
### Per-Company Report Review + Per-Robot Exclusions — ✅ (deployed 2026-08-20)

**Reports → Company report** ([[CustomerBundlePanel]], `?tab=company`): pick a company and month, see every
robot's page stacked as the customer will, untick robots with no activity, save, send. Built because a
customer like **IFS** has robots across many sites and offline ones render a page of zeros, which reads as a
broken report — CS previously had only a per-robot preview and no whole-bundle review step.

- **Exclusions persist server-side** (`customer_report_exclusions`, **V31**) and are applied inside
  [[CustomerReportBundleService]] `build()` — the single path the customer's public link *and* the delivery
  email both read. A UI-only filter would have let staff approve one thing and the customer open another.
- **`buildPreview()` is a separate staff path** returning *all* robots with `hasData` / `excluded` flags, via
  its own [[CustomerBundlePreviewResponse]]. Robot ids and exclusion state deliberately never reach the
  customer-facing [[CustomerReportBundleResponse]].
- **A filter, never a deletion.** Un-ticking restores the robot immediately; telemetry is untouched. Keyed on
  (customer, **month**, robot) so a robot idle in July reports normally in August with no manual undo.
- `hasData` = `totalTasksCompleted > 0`; UI badges **"N tasks"** vs **"No activity"** plus a one-click
  *"Drop the N with no activity"*. **Send is blocked while the selection is unsaved**, so the email can never
  differ from the screen.
- Endpoints `GET|PUT /api/v1/reports/customer-bundle/{preview,exclusions}`. 6 tests (suite **101**).
  Verified live: IFS link went 3 → 2 → 3 robots across exclude/restore; PCS : Makro (71 robots) previews in
  ~38s cold. See [[history]] session 2026-08-16.

### Automated Monthly Report Delivery — ✅ (built, scheduler off by default)

**Final delivery model: one email per customer per month → one link to a combined web page showing all
their robots' reports.** The old n8n/LINE/Supabase/xlsx pipeline was **removed entirely** (MonthlyReportService,
ReportGenerator*, N8n*, Supabase*, MonthlyReportScheduler, WeeklyReportScheduler, ReportController + their tests).
Reports are the **public web page** ([[MonthlyReportView]]); the email just carries the link (no attachments).

- **Per-robot aggregation** — [[ReportPreviewService]] builds one `MonthlyPerformanceReport` per robot from its
  synced `robot_task_reports`. Public single-robot page: `/[locale]/report/[token]` ([[ReportLink]], V19).
- **Weekly is a period filter on the same report, preview-only (2026-08-31).** [[ReportPeriod]] resolves either a
  month (`YYYY-MM`, read off the indexed `report_month`) or an **ISO week** (`YYYY-Www`, Mon–Sun) — a week
  straddles months, so it is a `start_time` range anchored in **Asia/Bangkok**, matching [[PartnerDataService]].
  `GET /api/v1/reports/preview` takes `month` **or** `week` (exactly one; a bad week is a 400, a bad month is
  still tolerated). Same layout, same metrics, so weekly figures reconcile with monthly.
  ⚠️ **Sharing and email remain monthly-only** — `report_links.report_month` is `VARCHAR(7)` and one token is
  keyed per robot+month — so both buttons are disabled with an explanation in weekly mode. Weekly *sending*,
  the WEEKLY cadence, and a weekly bundle are all still unbuilt. See [[history]] session 2026-08-31.
- **Customer bundle** — [[CustomerReportBundleService]] aggregates **all** a customer's robots for a month into
  `CustomerReportBundleResponse`. One stable token per customer+month ([[CustomerReportLink]], V20,
  [[CustomerReportLinkService]]). Public combined page: `/[locale]/report/customer/[token]`
  ([[CustomerBundleClient]]) — stacked robot reports, print page-breaks, Download PDF (window.print).
- **Email** — [[ReportEmailService]] `send()` (per-robot, manual preview) and `sendBundle()` (per-customer,
  automated) via `spring-boot-starter-mail`. SMTP unconfigured → clear BadRequestException.
- **Scheduler** — [[ReportDeliveryScheduler]] (cron `app.reports.email-scheduler-cron`, default 2nd of month
  08:00 UTC, previous month), `@ConditionalOnProperty(app.reports.email-scheduler-enabled=true)` — **off by
  default**. Delegates to [[ReportDeliveryService]]: sync telemetry for the month → send each MONTHLY-cadence
  customer their bundle → record every attempt in `report_sends` (V21, [[ReportSend]]) as SENT/FAILED/SKIPPED.
  **Idempotent** — a customer already SENT for the month is skipped, so re-runs never double-send.
- **Admin UI** — Tools → **Manage report automation** tab ([[ReportAutomationPanel]]): pick month, **Run delivery
  now** (idempotent), view **history** (SENT/FAILED/SKIPPED per customer), **resend** failures. Endpoints:
  `POST /api/v1/reports/delivery/run|send`, `GET /api/v1/reports/delivery/history` ([[ReportDeliveryController]]).
  Cadence is per-deployment (`MONTHLY`/`WEEKLY`/`OFF`, V16); only MONTHLY is wired for automated send.

See [[history]] session 2026-07-01 for full details. **Go-live:** set `REPORT_EMAIL_SCHEDULER_ENABLED=true`
+ `MAIL_USERNAME`/`MAIL_PASSWORD` (Gmail App Password) + `PUBLIC_BASE_URL` on Render.

### Robot Registration + Deployment API — Phase 1 ✅ (report-targeting feature)

Admin can now register robots and target reports. **Phase 1 (backend) done:** `RobotUnit`/`Deployment` had
**no create API** before (seed SQL only). New `robotunit` controller/service/DTOs: register a robot by SN +
deploy to a customer, set **per-deployment cadence** (`MONTHLY`/`WEEKLY`/`OFF`, `V16` on `deployments`), and
**bidirectional search** — `GET /api/v1/robot-units?customerId=` (customer→robots) or `?serialNumber=`
(robot→customer), both returning `RobotUnitResponse` carrying robot **and** owning customer. Endpoints:
`POST /api/v1/robot-units`, `GET /api/v1/robot-units`, `PATCH .../deployments/{id}/cadence`,
`DELETE .../deployments/{id}`. Compiles clean. **Admin UI done:** Tools → **Robots** tab ([[RobotsPanel]])
registers a robot to a customer (dropdown), changes cadence inline, and deactivates; Tools → **Customers**
tab manages customers; Tools → **Report Preview** tab previews the report page. Full UI loop:
add-customer → register-robot → preview. **Pending:** Phase 2 (cadence-aware schedulers +
per-customer/per-robot send grouping, where `send_mode` lives). See [[history]] session 2026-06-21.

### Customer Management (CRUD) — ✅

Admin can now add/edit/remove customers (Tools → **Customers** tab; `customerApi` → `/api/v1/customers`).
**`CustomerProfile` decoupled from `User`** (V17: `user_id` now nullable, kept UNIQUE) — customers are
report *recipients*, not login accounts. V17 also added **`contact_email`** (for the email-delivery pivot;
this closes the earlier "customer needs an email field" TODO). Delete is blocked while the customer has
deployed robots. Unblocks the register-robot UI (customer dropdown). Frontend: [[CustomersPanel]]. Backend:
`customer/{controller,service,dto}`. See [[history]] session 2026-06-21.

---

## Main User Flow

```text
RAASPAL team member logs in
→ Uploads customer survey (Excel / PDF / image)
→ AI extracts structured requirements        ← [[RequirementExtractionService]]
→ Backend loads robot catalog from PostgreSQL
→ AI compares requirements with catalog      ← [[RobotRecommendationAiService]]
→ AI returns 2–3 ranked robot options
→ User selects one option
→ AI generates proposal (Markdown)           ← [[ProposalGenerationAiService]]
→ Backend saves recommendation and proposal history
```

---

## AI Provider Configuration

| Environment Variable | Effect |
|---|---|
| `ANTHROPIC_API_KEY` not set | [[MockAiService]] activates (hardcoded responses) |
| `ANTHROPIC_API_KEY=sk-ant-...` | [[ClaudeAiService]] activates (real Claude API) |
| `ANTHROPIC_MODEL` | Override extraction/recommendation model (default: `claude-sonnet-4-6`) |
| `ANTHROPIC_PROPOSAL_MODEL` | Override proposal generation model (default: `claude-opus-4-8`) |

Set in [[application-local.properties]] for local dev, or as env vars in production.
**Both are configured as of 2026-08-05** — local and Render each hold a real key, so [[ClaudeAiService]]
is the active bean in both and [[MockAiService]] now only serves the test profile (whose
`app.anthropic.api-key` is deliberately left unset, keeping the suite free and offline).

**Not every call honours `ANTHROPIC_MODEL`.** Three features pin their own model in code because the
task is narrow enough that the default would be overkill — changing the env var will not move them:

| Feature | Model | Where |
|---|---|---|
| Requirement extraction, recommendation | `ANTHROPIC_MODEL` (`claude-sonnet-4-6`) | constructor `@Value` |
| Proposal generation | `ANTHROPIC_PROPOSAL_MODEL` (`claude-opus-4-8`) | constructor `@Value` |
| CM report extraction | `claude-haiku-4-5-20251001` | `CM_MODEL` in [[ClaudeAiService]] |
| Slide manifest | `claude-haiku-4-5-20251001` | `SLIDE_MODEL` |
| EN→TH translation | `claude-haiku-4-5-20251001` | `TRANSLATION_MODEL` |

---

## Key Configuration Files

- [[application.properties]] — main config with environment-variable placeholders
- [[application-local.properties]] — local secrets, gitignored (Supabase + Anthropic key)
- [[pom.xml]] — Maven dependencies (no Anthropic SDK — uses Spring `RestClient` directly)
- [[next.config.ts]] — Next.js + next-intl plugin config
- [[proxy.ts]] — Next.js 16 middleware: i18n routing + JWT auth guard

---

## Pending / Not Yet Implemented

- [ ] **RE KPI dashboard — three boards live locally on `feat/re-kpi-dashboard` (2026-09-08), unmerged, V38–V41 unapplied on prod.** Next: PM Complete (waiting on the visits-due rule), Job Type filter on the installation board, RE sign-off on the SLA definition and the cleaning Apr/May gap, rotate the monday token, then merge + deploy
- [x] Robot catalog populated — 6 robots in Supabase (Gausium MIRA, KEENON C40, CENOBOT L3/L4/L50/SP50)
- [x] Robot catalog UI — horizontal list grouped by brand (A-Z), pagination, search by brand/model
- [x] Backend deployed → `https://ai-robotrecommendationsystem-backend.onrender.com`
- [x] Robot pricing fields — `rental_price` / `selling_price` added via Flyway V6; entity + DTOs updated
- [x] Robot keyword search — [[RobotRepository]] `search()` JPQL query on model/brand
- [x] Verify-password endpoint — `POST /api/v1/auth/verify-password` (authenticated)
- [x] Robot delete guard — checks recommendation references (409), deletes spec first; [[V7__rename_concierge_to_mowing.sql]] renames CONCIERGE → MOWING
- [x] Robot type renamed — `CONCIERGE` → `MOWING` in enum, DB migration, and all UI
- [x] i18n wired — all pages use `useTranslations()`; Thai and Chinese fully active
- [x] Favicon — RaasPal "R" logo (`app/icon.png`), old `favicon.ico` removed
- [x] CVTE C3 online/offline status tracking — separate module, backend + frontend done (see session 2026-06-08)
- [x] Telemetry Phase 1 — robot_units/deployments (V11), robot_task_reports + Gausium OAuth tokens (V12/V13), TelemetryAdapter/Registry/SyncService, GausiumAdapter/ApiClient/OAuthService — `mvn test` passes (see session 2026-06-12)
- [x] `CustomerProfile.lineNotifyToken` field added (see session 2026-06-13)
- [x] V14 migration — residual-% columns widened to `DECIMAL(5,2)` (see session 2026-06-13)
- [x] Old n8n/LINE/Supabase/xlsx report pipeline **REMOVED** (MonthlyReportService, ReportGenerator*, N8n*, Supabase*, MonthlyReportScheduler/WeeklyReportScheduler, ReportController + tests) — replaced by the email + web-page delivery below (see session 2026-07-01)
- [x] Automated monthly report delivery — per-customer bundle page (`/report/customer/[token]`, V20), `report_sends` history (V21), [[ReportDeliveryScheduler]] (`@ConditionalOnProperty`, off by default) + [[ReportDeliveryService]] (idempotent), admin tab [[ReportAutomationPanel]] (run/history/resend). `mvnw test` passes; `npm run build` clean (see session 2026-07-01)
- [ ] Configure real `CVTE_KAVA_BASE_URL`/`CVTE_KAVA_APP_ID`/`CVTE_KAVA_APP_SECRET` to activate CVTE sync against the live Kava gateway
- [x] Partner middleware API (PCS) — `/api/partner/v1` with `X-API-Key` auth, partner-scoped reads, admin API + Tools → Partners UI (see sessions 2026-07-25)
- [x] Partner API hardening — V23: per-key rate limiting, access audit + retention, key expiry/rotation nudge, strict `month` validation; 32 partner tests (see session 2026-07-25)
- [x] Telemetry sync pipeline — [[TelemetrySyncScheduler]] + reworked `syncAllActive` (batch dedup, look-back window, off by default) (see session 2026-07-25)
- [x] **Gausium telemetry sync is LIVE in production** — confirmed 2026-08-10 by direct query against Supabase:
  `robot_task_reports` holds **41,831 rows** (25 MB, **~626 bytes/row**) across **152 `robot_units` / 152
  `deployments`** serving **60 `customer_profiles`**. The `GAUSIUM_API_*` credentials and
  `TELEMETRY_SYNC_ENABLED=true` are set on Render. ⚠️ **These must be carried to Lightsail or the nightly sync
  silently stops** — nothing fails loudly, the data just goes stale.
- [ ] Onboard PCS: create partner → mint key → bulk-assign its ~70 deployments → deliver key securely
- [x] PCS-facing partner API documentation — served from the frontend at `/gs-middleware-api.html` (`public/`, bypasses the [[proxy.ts]] auth guard because the matcher excludes file extensions); also published as a shareable page
- [x] **Partner API OAuth (client credentials) — BUILT 2026-07-30.** V24 adds `partner_api_keys.client_id` (nullable → backfilled → unique, expand/contract because local and prod share one Supabase DB). [[PartnerTokenService]] / [[PartnerOAuthController]] / [[PartnerJwtAuthFilter]]; `X-API-Key` retired on data endpoints. 14 OAuth tests.
- [x] **Partner filter scope + token throttle + IP resolution + placeholder-secret guard — 2026-07-30.** See the Partner Middleware section above; V25 adds `(robot_unit_id, start_time DESC)` for the paginated report queries.
- [ ] Deferred follow-on: **PCS self-service portal** (`partner_users`, login, email password reset, PCS minting its own credentials) — what actually closes the departing-employee gap. Until then, credential expiry bounds it. Design notes in `~/.claude/plans/i-am-currently-in-jolly-deer.md`.
- [ ] **Partner live status (real-time battery/position/progress) — DESIGNED, NOT BUILT.** Same plan file. Verified possible against a live robot via `GET /v1alpha1/robots/{sn}/status`, same OAuth token as the sync. Decisions taken: **poll on our schedule, never pass through** (80 robots × open dashboards would hand our Gausium quota to PCS), adaptive interval (30s running / 5min idle, partner robots only), latest-row-only storage. ⚠️ The plan claims `V24`, since taken — **live status must use `V26`** (V25 is the report index).
- [x] **Corrective Maintenance report auto-creation — BUILT 2026-08-05.** `cm` package, **V27** (`cm_reports`), Haiku-backed extraction, Reports hub split into sub-sections. 95 tests pass; V27 applied to Supabase (see session 2026-08-05)
- [ ] **Verify the CM report renders identically to the paper form** — compare `/en/reports?tab=cm-new` against `Pandora - CM Report-17 June 2026.pdf`; no browser automation was available when it was built
- [ ] **Verify `COMPANY_FOOTER` in [[CorrectiveMaintenanceReportView]]** against the company registration — transcribed from a scan (`พอล` vs `พาล` for "PAL" undecidable from the image), and it sits beside a tax ID
- [x] **CM report fits one page — BUILT 2026-08-31.** Three faults: every row carried `print:break-inside-avoid` (a row taller than a sheet gets pushed to a fresh page and overflows anyway, leaving page one two-thirds empty); a `1fr` grid track would not shrink, so long runs ran off the paper (`min-w-0` + `overflow-wrap` — **Thai has no inter-word spaces**, so this hits real text); and body type is now binary-searched to the largest size fitting one A4, floor 8px. **Not yet verified on paper** (see session 2026-08-31)
- [x] **Per-company report review + per-robot exclusions — BUILT 2026-08-16.** V31 `customer_report_exclusions`, Reports → Company report tab, exclusions honoured by the customer's public link (see session 2026-08-16)
- [x] **Contract start date clipping — BUILT 2026-08-19** (V32 → corrected by **V33**). Reports no longer include work done before a **robot's** contract began; the date lives on the deployment, not the customer
- [x] **Robot lifecycle status + packaging (RIMS) — BUILT 2026-08-28, V37 applied and backend deployed.** Four store-room states, one band per state in the catalogue, plus a packaging field. Requested by P'Pom via the CS team (see sessions 2026-08-28 and 2026-08-31)
- [ ] **Mixed-shelf packaging is not representable** — `packaging` holds one value for a whole row, so a shelf of three with one boxed cannot be recorded, and `uq_robot_inventory_temp_identity` blocks the obvious workaround of a second row for the same model+status. Needs **V38** adding `boxed_quantity INTEGER` (additive, safe against the running backend). The code was written and reverted, so restoring it is small
- [ ] ⚠️ **RIMS frontend is committed (`44ed937`) but its deployment is unverified.** If the live RIMS predates
  it, editing *any* robot that has a photo fails with "Image must be an http(s) URL…" — whatever field is
  actually being changed — because the picker used to post back the image endpoint's URL instead of the image.
  Worth confirming on Vercel before telling the warehouse the feature is available
- [x] **Weekly performance report (preview only) — BUILT 2026-08-31.** [[ReportPeriod]] + `buildForWeek`, `week=YYYY-Www` on `/api/v1/reports/preview`, Monthly/Weekly toggle in [[ReportPreviewPanel]]. No migration. 134 tests pass; `npm run build` clean (see session 2026-08-31)
- [x] **Daily Pending Case Report — MK sheet BUILT 2026-09-11, merged to `main` in PR #6.** V38 applied. Reports → Pending cases → MK pending generates the `Raw_Delivery` layout from the live delivery board; SLA 3 days in the six greater-Bangkok provinces / 5 elsewhere, Days inclusive. Reports freeze on first generation and a past date with no frozen run is **refused**, not fabricated. **Solution column is written by Haiku from the comment thread** ([[CaseSolutionAiService]]); a typed board value wins. 200 tests pass (see sessions 2026-09-11 and 2026-09-11b)
- [ ] **PR #6 review follow-ups** (not blocking): Haiku calls run inside `CaseReportRunService.rowsFor`'s transaction; two concurrent first generations race on `uq_case_report_run_day`; `DELETE /mk/run` can discard an unrecoverable past draft. Listed on the PR.
- [ ] ⚠️ **Turn on the daily case-report scheduler** (`CASE_REPORT_SYNC_ENABLED=true`, 06:15 Bangkok). It snapshots both boards then freezes the day's report. **Every day it is off is a day that can never be reported on** — nothing before 2026-09-11 exists.
- [ ] ⚠️ **Case report is merged but not deployed.** Deploy `main` (`cf04b84`) on Lightsail with `bash deploy/deploy.sh`; `deploy/api.env` needs `MONDAY_API_TOKEN` (and `ANTHROPIC_API_KEY`, or Solution lines come from the mock). Flyway will apply V39 on that start. Never deploy a `main` older than `faa74a9` — it lacked V38 and crash-loops against prod. Local testing uses MODE B in the console's `.env.local`.
- [ ] **Case report Excel export** not built (`poi-ooxml` already in `pom.xml`); AOT and ALL_PENDING generators not built. AOTGA has no SLA column and needs AI over comment threads — do it last.
- [ ] **Weekly *sending* is not built** — needs a period-aware [[ReportLink]] (`report_month` is `VARCHAR(7)`; widen it or add a period column — **V39 or later**, V38 is taken), then email + the public token page. The `WEEKLY` cadence on `deployments` (V16) is still inert, and the customer bundle is still monthly-only
- [ ] **Backfill contract start dates per robot** under Tools → Robots — **1 of 163 deployed robots done as of 2026-08-20**. Until a robot has a date it reports whole months, so mid-month starts are still over-reported
- [ ] ⏰ **BEFORE 2026-09-02 — rename two report labels** (decided 2026-08-19, deliberately deferred so July reports match what customers already received). `report.totalTasksCompleted` → "Total Tasks Run", `report.taskCompletionRate` → "Area Completion Rate", in `messages/en.json` + `messages/th.json`. **Frontend i18n only** — the backend `Ring("Task Completion Rate")` string is an internal key mapped by `RING_KEYS`. August reports send 08:00 on 2 Sep (`0 0 8 2 * *`), so it must land before then. See [[history]] 2026-08-19c
- [x] **Backend deployed 2026-08-20** — verified live: `/customer-bundle/*` responds, `deployments.contract_start_date` present (V31 + V33 applied), MAIL_CC active. July reports sent.
- [ ] Report automation go-live: set `REPORT_EMAIL_SCHEDULER_ENABLED=true` + `MAIL_USERNAME`/`MAIL_PASSWORD` (Gmail App Password) + `PUBLIC_BASE_URL` on Render
- [ ] Deploy updated frontend to Vercel (merge dev-1 into main)
- [ ] Set `CORS_ALLOWED_ORIGINS=https://your-vercel-url.vercel.app` in Render env vars after Vercel deploy
- [x] **`ANTHROPIC_API_KEY` set on Render (2026-08-05)** — [[ClaudeAiService]] is now the active bean in production, so AI runs for real everywhere (local and Render). This was the last thing gating CM report extraction, which would otherwise have degraded silently to [[MockAiService]]'s exact-`label :` scanner.
- [ ] Add proposal template via `POST /api/v1/proposal-templates`
- [ ] Test full flow end-to-end with real API key
- [ ] Thai translations for Add/Edit robot form fields (low priority)
- [ ] Customers, Solutions, Reports pages (nav exists, no pages yet)

## Candidate Next Steps / Backlog

No longer hard "out of scope" — the MVP gate is lifted. Evaluate each per request and
weigh the trade-offs (especially before adding heavy infrastructure):

- Report view caching (in-memory first; Redis only if needed) + larger DB connection pool
- Weighted scoring engine / manual formula-based matching
- pgvector / embeddings / combination engine
- Public customer login / self-service dashboard
- PowerPoint generation
- Gamma API integration
- Advanced audit log system
