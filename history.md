# RAASPAL AI Project — Development History

> Condensed log. See [[index]] for current project state.

---

## 2026-05-27 — Backend Steps 1–12

Built all of [[RaasPal-Internal-Ops-backend]]: Spring Boot 3.4.5 + Java 21, JWT auth stack ([[AuthTokenFilter]], [[UserDetailsServiceImpl]], [[JwtUtils]]), user management, file upload service, robot catalog entities + CSV/Excel import, AI service interfaces + [[MockAiService]], requirement/recommendation/proposal REST endpoints, Flyway V1–V12.

---

## 2026-05-28 — Frontend Step 13

Built [[robot-recommendation-web-raaspal]]: shadcn/ui + Tailwind v4, [[lib/api.ts]] Axios client with JWT interceptor, Zustand auth store, [[proxy.ts]] i18n + auth guard middleware, login page, full Upload → Extract → Recommend → Propose flow wired to real backend.

---

## 2026-05-29 — Supabase + ClaudeAiService

Connected backend to Supabase (session-mode pooler, port 5432, sslmode=require). Built [[ClaudeAiService]] using Spring `RestClient` — no Anthropic SDK needed. Excel surveys parsed via Apache POI, PDFs/images sent as native Claude document/image blocks. Fixed [[MockAiService]] conditional: replaced `@ConditionalOnMissingBean` (unreliable on `@Service`) with `@ConditionalOnExpression("'${app.anthropic.api-key:}' == ''")`.

**Key decisions:** Session-mode pooler (transaction mode breaks Hibernate prepared statements). RestClient over SDK (zero extra deps). Proposal output as MARKDOWN for future PDF flexibility.

---

## 2026-05-30 — Opus/Sonnet Split + Robot Catalog UI + Import Fixes

**AI model split:** `extract()` and `recommend()` use Sonnet 4.6; `generateProposal()` uses Opus 4.8 (separate `proposalModel` field + `ANTHROPIC_PROPOSAL_MODEL` env var). Cost ~$0.10–$0.13/generation.

**Robot catalog UI:** Add Robot page (`/robots/new`) with 73-field single-robot form + Excel/CSV bulk import tab. [[RobotDetailModal]] opens on card click (scrollable, all specs, chip tags for booleans, Esc/backdrop close). Edit Robot page (`/robots/[id]/edit`) pre-fills all fields from backend. Robot cards show photo if `imageUrl` set, else Bot icon. All 73 spec fields mirrored in [[types/api.ts]] `RobotSpecRequest` + `RobotSpecResponse`. `robotApi` extended with `create`, `update`, `delete`, `importCatalog`.

**Backend enum fixes:** Added `PENDING`, `REJECTED` to [[TestStatus]]; added `CONCIERGE` to [[RobotType]]; added `MEDIUM`, `PREMIUM` to [[BudgetBand]]. These were missing, causing 500s on import.

**Import 500 fix:** Axios instance default `Content-Type: application/json` was overriding FormData multipart. Fix: pass `headers: { 'Content-Type': undefined }` in `importCatalog` and `fileApi.upload` so the browser sets the correct boundary automatically.

**Robot data:** Created `docs/Robot Datasheet - Ready Robots.csv` with 6 high-quality robots (Gausium MIRA, KEENON C40, CENOBOT L3/Scrubber L4/L50/SP50). Full CSV has 37 robots but most are too sparse. Import confirmed working — 6 robots now in Supabase.

---

## 2026-05-31 — PowerPoint Export + Backend Deployed to Render

**PowerPoint export:** Added `ProposalExportService.java` — loads `proposal-template.pptx` from classpath (`src/main/resources/templates/`), keeps cover slide (slide 1) with text replacement ("Project Name" → proposal title, Name/Title/Date fields), removes Gausium slides 2–49, adds one navy content slide per proposal Markdown section (cyan title, white body, bullet detection), closes with a Thank You slide. Endpoint: `GET /api/v1/proposals/{id}/export/pptx`. Frontend: "Download PowerPoint" button in `ProposalViewClient.tsx` using `proposalApi.exportPptx(id)` with `responseType: 'blob'`. Template is 94 MB — GitHub warned but accepted it.

**Render deployment:** Backend deployed to `https://ai-robotrecommendationsystem-backend.onrender.com`. Fixed two issues: (1) `mvnw: Permission denied` → added `RUN chmod +x mvnw` to Dockerfile. (2) Was pushing to `dev` but Render watched `main` → merged dev into main and pushed. All env vars set. Supabase connected, Flyway validated, service live.

**Enum fixes committed:** `TestStatus` (added PENDING, REJECTED), `RobotType` (added CONCIERGE), `BudgetBand` (added MEDIUM, PREMIUM) — all in the Deployment commit on dev/main.

---

## 2026-05-31 — Vercel Deployment Prep

**Goal:** Make [[robot-recommendation-web-raaspal]] deployable to Vercel pointing at the Render backend.

**Changes made:**

*Frontend:*
- `.gitignore` — changed `.env*` (blocks all env files) to `.env.local` / `.env*.local` (standard Next.js pattern so `.env.production` can be committed safely)
- `.env.production` — new file: `NEXT_PUBLIC_API_URL=https://ai-robotrecommendationsystem-backend.onrender.com`; Vercel reads this automatically on build
- [[lib/api.ts]] — increased Axios timeout from 30 s → 60 s (Render free tier cold start can be ~50 s)
- [[AppTopBar.tsx]] — made `searchPlaceholder` prop optional (was required, caused TS build failures on pages that omitted it)
- [[types/api.ts]] — added missing `CreateUserRequest` export (imported by `lib/api.ts`, was undefined)
- [[RecommendationClient.tsx]] — fixed all field references to match actual backend DTOs: `RecommendationItem` → `RecommendationItemResponse`; `recommendation.items` → `recommendation.options`; `recommendation.aiSummary` → `recommendation.aiExplanation`; `item.rank` → `item.rankPosition`; `item.robotId/robotName/robotBrand` → `item.robot.id/.model/.brand`; `item.aiReason` → `item.aiReasoning`; `strengths/limitations` (arrays) → `whyRecommended/limitations` (strings displayed as paragraphs)

*Backend:*
- [[application.properties]] — added `app.cors.allowed-origins=${CORS_ALLOWED_ORIGINS:http://localhost:3000}`
- test [[application.properties]] — added same property for H2 test context
- [[SecurityConfig.java]] — injected `@Value("${app.cors.allowed-origins}")`, parses comma-separated list so multiple origins (localhost + Vercel URL) can be whitelisted simultaneously

**Result:** Both builds pass — `mvn clean test`: BUILD SUCCESS (1 test); `npm run build`: exit 0, all routes pre-rendered.

## Remaining Deploy Steps

- [ ] Push code to GitHub / connect Vercel to repo → deploy frontend
- [ ] After getting Vercel URL: set `CORS_ALLOWED_ORIGINS=https://your-app.vercel.app` in Render env vars → redeploy backend
- [ ] Add `ANTHROPIC_API_KEY` to Render env vars once obtained → real AI activates
- [ ] Add proposal template via `POST /api/v1/proposal-templates` after frontend is live
- [ ] Test full Upload → Extract → Recommend → Propose → Download PPTX flow end-to-end

---

## Session: 2026-06-01 — Robot Pricing Fields + Password Verify Endpoint + Robot Search

**Date:** 2026-06-01
**Tags:** #session #backend #database

### Summary

Added `rental_price` and `selling_price` columns to the robots table via Flyway migration V6. Updated [[Robot]] entity, [[RobotRequest]], [[RobotResponse]], and [[RobotService]] to carry the new pricing fields. Added keyword search support to [[RobotRepository]] via a custom `@Query` matching model and brand. Added a `POST /api/v1/auth/verify-password` endpoint backed by new [[VerifyPasswordRequest]] DTO and [[AuthService#verifyPassword()]] — returns 200 if the authenticated user's current password matches, error otherwise. Frontend build was run (`.next` artifacts updated) but no frontend source files were changed.

### Files Modified

- [[V6__add_robot_pricing.sql]] (`src/main/resources/db/migration/V6__add_robot_pricing.sql`) — new Flyway migration: adds `rental_price` and `selling_price` DECIMAL(12,2) columns to `robots` table
- [[Robot]] (`robot/entity/Robot.java`) — added `rentalPrice` and `sellingPrice` BigDecimal fields
- [[RobotRequest]] (`robot/dto/RobotRequest.java`) — added `rentalPrice` / `sellingPrice` fields
- [[RobotResponse]] (`robot/dto/RobotResponse.java`) — added `rentalPrice` / `sellingPrice` to record + `from()` mapping
- [[RobotService]] (`robot/service/RobotService.java`) — updated create/update logic for pricing fields
- [[RobotRepository]] (`robot/repository/RobotRepository.java`) — added `search(@Param keyword)` JPQL query (LIKE on model and brand)
- [[VerifyPasswordRequest]] (`auth/dto/VerifyPasswordRequest.java`) — new record DTO with `@NotBlank password`
- [[AuthService]] (`auth/service/AuthService.java`) — added `verifyPassword(email, request)` method
- [[AuthController]] (`auth/controller/AuthController.java`) — added `POST /api/v1/auth/verify-password` endpoint (requires auth)

### Decisions Made

- **Pricing stored as DECIMAL(12,2):** Supports both rental and outright-sale pricing; nullable so existing robots are unaffected until data is populated.
- **Verify-password as authenticated endpoint:** Caller must already hold a valid JWT; the endpoint just re-checks the password (useful for confirming identity before sensitive actions in the frontend).
- **Search via JPQL LIKE:** Simple keyword search sufficient for MVP catalog size; no full-text index needed yet.

### Unresolved / Next Steps

- [ ] Push to GitHub and deploy frontend to Vercel
- [ ] Set `CORS_ALLOWED_ORIGINS` in Render after Vercel URL is known
- [ ] Add `ANTHROPIC_API_KEY` to Render env vars
- [ ] Add proposal template via API
- [ ] Test full end-to-end flow with real AI key

---

## Session: 2026-06-01 — Delete Guard, Robot UI Overhaul, i18n, Favicon, Search

**Date:** 2026-06-01
**Tags:** #session #backend #frontend #database #config

### Summary

Completed several features across backend and frontend. Backend: fixed robot delete to guard against FK violations by checking recommendation references first, deleting the spec row before the robot, and returning 409 CONFLICT via a new `IllegalStateException` handler — all changes survived a mid-session computer crash and were recovered from the working tree. Frontend: rebuilt the robots page from a card grid into a horizontal list grouped by brand (A-Z, then model A-Z within each brand) with client-side pagination (9 per page). Renamed robot type `CONCIERGE` → `MOWING` across the Java enum, a V7 Flyway migration, and all frontend files. Wired `useTranslations()` into every hardcoded-English component (robots, proposals, upload, recommendation, proposal, user menu) so Thai and Chinese switch correctly. Replaced the default Next.js favicon with the RaasPal "R" logo by adding `app/icon.png` and deleting the old `app/favicon.ico`. Added a live search bar on the robots page filtering by brand or model.

### Files Modified

**Backend:**
- [[GlobalExceptionHandler]] (`common/exception/GlobalExceptionHandler.java`) — added `IllegalStateException` → 409 CONFLICT handler
- [[RecommendationItemRepository]] (`recommendation/repository/RecommendationItemRepository.java`) — added `existsByRobot_Id`
- [[RobotService]] (`robot/service/RobotService.java`) — delete now checks recommendation references and deletes spec first
- [[RobotType]] (`common/enums/RobotType.java`) — `CONCIERGE` → `MOWING`
- [[V7__rename_concierge_to_mowing.sql]] (`db/migration/V7__rename_concierge_to_mowing.sql`) — new Flyway migration

**Frontend:**
- [[RobotsClient]] (`app/[locale]/robots/RobotsClient.tsx`) — horizontal list grouped by brand, A-Z sort, pagination, search bar, i18n
- [[RobotDetailModal]] (`components/RobotDetailModal.tsx`) — `CONCIERGE` → `MOWING` in typeBg map
- [[ProposalsClient]] (`app/[locale]/proposals/ProposalsClient.tsx`) — full i18n
- [[UploadClient]] (`app/[locale]/generate-solution/[solutionType]/upload/UploadClient.tsx`) — full i18n + ROBOT_TYPE_MAP updated
- [[RecommendationClient]] (`app/[locale]/generate-solution/[solutionType]/recommendation/RecommendationClient.tsx`) — full i18n
- [[ProposalClient]] (`app/[locale]/generate-solution/[solutionType]/proposal/ProposalClient.tsx`) — full i18n
- [[UserMenu]] (`components/UserMenu.tsx`) — full i18n
- [[AddRobotClient]] (`app/[locale]/robots/new/AddRobotClient.tsx`) — `CONCIERGE` → `MOWING`
- [[EditRobotClient]] (`app/[locale]/robots/[id]/edit/EditRobotClient.tsx`) — `CONCIERGE` → `MOWING`
- [[en.json]] / [[th.json]] / [[zh.json]] (`messages/`) — added `robots.*`, `proposals.*`, `userMenu.*`, `generateSolution.upload/recommendation/proposal.*`, `generateSolution.forms.mowing.*`
- `app/icon.png` — RaasPal logo added as browser favicon
- `app/favicon.ico` — deleted (was overriding icon.png)

### Decisions Made

- **Client-side sort + pagination for robots:** All 200 robots loaded once; sort/filter/paginate in the browser. Simple and instant for MVP catalog size.
- **Brand-grouped horizontal list:** Replaces card grid; one robot per row grouped under a brand header with dividers — cleaner for a catalog that grows by brand.
- **CONCIERGE → MOWING:** Renamed to better reflect the actual third use-case. V7 migration handles any existing DB rows.
- **Client-side search:** Partial match on brand and model against the already-loaded dataset — no new API call needed.
- **i18n via useTranslations only:** All visible UI text in client components now flows through next-intl so Thai/Chinese switch correctly.
- **favicon.ico must be deleted:** Next.js serves `favicon.ico` in preference to `icon.png`; deleting the old file lets the RaasPal PNG take over.

### Unresolved / Next Steps

- [ ] Deploy updated frontend to Vercel (merge dev-1 into main)
- [ ] Set `CORS_ALLOWED_ORIGINS` in Render after Vercel URL is confirmed
- [ ] Add `ANTHROPIC_API_KEY` to Render env vars to activate real AI
- [ ] Add proposal template via `POST /api/v1/proposal-templates`
- [ ] Test full Upload → Extract → Recommend → Propose flow end-to-end with real AI key
- [ ] Thai/Chinese translations for Add/Edit robot form fields (low priority)

---

## Session: 2026-06-06 — AI-Powered PPTX Generation + UI Fixes

**Date:** 2026-06-06
**Tags:** #session #backend #frontend #ai

### Summary

Resolved a cascade of PPTX export issues and implemented AI-powered slide generation. Starting from a 500 error on `/export/pptx`, we traced and fixed: (1) AWT headless mode missing on Render (added `java.awt.headless=true` in `main()`); (2) OOXML corruption from `getBackground().setFillColor()` and `clearText()` on text boxes — fixed by using `XSLFAutoShape(ShapeType.RECT)` for all rectangles/backgrounds and reusing `getTextParagraphs().get(0)` for text. The PPTX then opened correctly but was visually plain. The user requested Claude AI to generate richer slide content, leading to a three-file addition: `SlideManifest.java` (typed DTO), `ProposalGenerationAiService.generateSlideManifest()` (default returns null), and `ClaudeAiService.generateSlideManifest()` (calls Haiku to produce JSON with title/key_stats/content/table/closing slides). `ProposalExportService` was rewritten to inject the AI service, call the manifest at export time, and render five typed slide layouts with cyan accents, stat boxes, bullet lists, an XSLFTable for specs, and a closing card. Graceful fallback to markdown-section parsing is used when AI is unavailable.

On the frontend, all remaining hardcoded English strings in solutions and proposals pages were replaced with `useTranslations()`. The critical discovery was that Tailwind v4 maps `dark:` to OS media query by default — fixed globally with `@custom-variant dark (&:where(.dark, .dark *));` in `globals.css`. Additional fixes: logo shadow hidden in light mode, missing-info box made readable in dark mode, page headers (`getTranslations()` in async server components), and search bars wired on solutions and proposals pages.

### Files Modified

**Backend:**
- [[RobotRecommendationApiApplication]] (`RobotRecommendationApiApplication.java`) — `System.setProperty("java.awt.headless", "true")` before Spring boot
- [[GlobalExceptionHandler]] (`common/exception/GlobalExceptionHandler.java`) — added generic `Exception` handler with full stack-trace logging
- [[SlideManifest]] (`proposal/dto/SlideManifest.java`) — new record DTO: `SlideData(type, title, subtitle, bullets, stats, headers, rows)` + `StatItem(label, value)`
- [[ProposalGenerationAiService]] (`ai/service/ProposalGenerationAiService.java`) — added `default generateSlideManifest(String)` returning null
- [[ClaudeAiService]] (`ai/service/ClaudeAiService.java`) — implemented `generateSlideManifest()` using `claude-haiku-4-5-20251001`; `buildSlideManifestPrompt()` and `parseSlideManifest()`
- [[ProposalExportService]] (`proposal/service/ProposalExportService.java`) — full rewrite: injects AI service, manifest-based rendering (5 slide types), XSLFAutoShape backgrounds, XSLFTable for spec table, markdown fallback

**Frontend:**
- [[globals.css]] (`app/globals.css`) — `@custom-variant dark (&:where(.dark, .dark *));` — fixes ALL dark: utilities to use CSS class not OS media query
- [[AppSidebar]] (`components/AppSidebar.tsx`) — logo shadow prefixed `dark:` to hide in light mode
- [[RecommendationClient]] (`app/[locale]/generate-solution/[solutionType]/recommendation/RecommendationClient.tsx`) — missing-info box: `dark:border-amber-700/50 dark:bg-amber-900/30 dark:text-amber-400`
- [[SolutionsClient]] (`app/[locale]/solutions/SolutionsClient.tsx`) — full rewrite: owns layout, search state, all strings via `useTranslations('solutions')`
- [[ProposalsClient]] (`app/[locale]/proposals/ProposalsClient.tsx`) — full rewrite: owns layout, search state, delete inline UI, all strings via `useTranslations('proposals')`
- [[solutions/page.tsx]] (`app/[locale]/solutions/page.tsx`) — simplified to one-liner (async server headers moved into client)
- [[proposals/page.tsx]] (`app/[locale]/proposals/page.tsx`) — simplified to one-liner
- [[en.json]] / [[th.json]] / [[zh.json]] (`messages/`) — added `solutions.*` and `proposals.*` namespaces with search, eyebrow, delete, confirm keys

### Decisions Made

- **`XSLFAutoShape(RECT)` for all backgrounds/bars:** Calling `getBackground().setFillColor()` or `clearText()` on a text box produces malformed OOXML that PowerPoint rejects — shapes avoid both issues.
- **Haiku for slide manifest:** Fast and cheap for the JSON structuring task; Opus is reserved for proposal narrative; Sonnet for recommendations.
- **Default method on interface:** `MockAiService` gets null for free; `ClaudeAiService` provides a rich manifest; `ProposalExportService` handles both via the same fallback path.
- **Global `@custom-variant dark`:** One line in globals.css fixes all present and future dark-mode classes without touching individual components.

### Unresolved / Next Steps

- [ ] Deploy updated backend to Render (commit `3dcd711` on `dev`)
- [ ] Merge frontend `dev-1` into `main` for Vercel deployment
- [ ] Set `CORS_ALLOWED_ORIGINS` in Render after Vercel URL is confirmed
- [ ] Add `ANTHROPIC_API_KEY` to Render env vars to activate real AI
- [ ] Test full PPTX export with live AI (manifest → rich slides)

---

## Session: 2026-06-06 — Solution Naming Feature

**Date:** 2026-06-06
**Tags:** #session #backend #frontend #database

### Summary

Added user-defined names to solutions/recommendations. Previously all entries in the Solutions list showed as "Robot recommendation". Now users are prompted to enter a solution name on the Upload page before generating; the name is persisted on the backend and displayed in the solutions list.

### Files Modified

**Backend:**
- [[V8__add_recommendation_name.sql]] (`db/migration/V8__add_recommendation_name.sql`) — adds nullable `name VARCHAR(255)` column to `recommendations` table
- [[Recommendation]] (`recommendation/entity/Recommendation.java`) — added `name` field
- [[GenerateRecommendationRequest]] (`recommendation/dto/GenerateRecommendationRequest.java`) — added `name` field (nullable String)
- [[RecommendationResponse]] (`recommendation/dto/RecommendationResponse.java`) — added `name` to record and `from()` mapping
- [[RecommendationService]] (`recommendation/service/RecommendationService.java`) — sets `name` from request when building the entity

**Frontend:**
- [[en.json]] / [[th.json]] / [[zh.json]] (`messages/`) — added `solutionNameLabel`, `solutionNamePlaceholder`, `solutionNameRequired` keys under `generateSolution.upload`
- [[types/api.ts]] (`types/api.ts`) — added `name: string | null` to `RecommendationResponse`
- [[lib/api.ts]] (`lib/api.ts`) — `recommendationApi.generate()` now accepts optional `{ name?, optionCount? }` body
- [[UploadClient]] (`app/[locale]/generate-solution/[solutionType]/upload/UploadClient.tsx`) — added required "Solution name" text input; submit is disabled until both name and file are provided; name is URL-encoded and passed as `solutionName` query param on navigation
- [[RecommendationClient]] (`app/[locale]/generate-solution/[solutionType]/recommendation/RecommendationClient.tsx`) — reads `solutionName` from search params; passes it as `{ name }` body to `generate()`
- [[SolutionsClient]] (`app/[locale]/solutions/SolutionsClient.tsx`) — displays `rec.name` if set, falls back to top robot brand+model, then generic label

### Decisions Made

- **Name collected at upload step:** Natural start of the session; name is known before any AI API calls happen.
- **URL query param for handoff:** Avoids Zustand/session-storage complexity; name travels with the URL so deep-link and browser-back still work.
- **Nullable on backend:** Old recommendations have no name; frontend falls back gracefully so the Solutions list still renders for historical records.

### Unresolved / Next Steps

- [ ] Deploy updated frontend to Vercel (merge dev-1 into main)
- [ ] Set `CORS_ALLOWED_ORIGINS` in Render after Vercel URL is confirmed
- [ ] Add `ANTHROPIC_API_KEY` to Render env vars to activate real AI
- [ ] Test full Upload → Name → Extract → Recommend → Propose flow end-to-end

---
## Session: 2026-06-08 — CVTE C3 Online/Offline Status Tracking (New Module)

**Date:** 2026-06-08
**Tags:** #session #backend #frontend #ai #database #config

### Summary

Added a small, self-contained "CVTE C3 online/offline status" tracking feature on top of the Kava Open Gateway API (per the supplied Kava API doc — signature/HMAC_MD5/MD5 auth, device list/detail endpoints). Per explicit user direction, this is a **fully separate module** — its own entity, table, repository, service, and controller — intentionally not merged into or coupled with [[Robot]]/[[RobotSpec]] (the user wants to test it standalone before deciding whether to integrate later). Backend exposes 4 JWT-protected REST endpoints (auto-protected by the existing `anyRequest().authenticated()` rule — no `SecurityConfig` changes needed) for searching/syncing devices from Kava, listing tracked devices, and polling status on demand or via an optional `@ConditionalOnProperty`-gated scheduler. Frontend adds a dedicated "CVTE C3 Status" page (table view, sync form, manual refresh, 45s auto-refresh, online/offline/unknown badges) plus — per user request — a compact "less detail" status summary widget on the Team Dashboard, both calling only the Spring Boot backend (never Kava directly). Backend `mvn test` and frontend `npm run build`/`tsc --noEmit` all pass.

### Files Modified

**Backend (`com.raaspal.robotrecommendation.cvte.*` — new package, fully separate from `robot.*`):**
- [[V9__add_cvte_devices.sql]] (`db/migration/V9__add_cvte_devices.sql`) — new `cvte_devices` table (device_id, factory_sn, device_name, org_code, online_status, running_state, battery_percentage, last_checked_at, last_message + timestamps), unique constraints + indexes on org_code/device_name
- [[CvteDevice]] (`cvte/entity/CvteDevice.java`) — JPA entity, `@CreationTimestamp`/`@UpdateTimestamp`
- [[CvteDeviceRepository]] (`cvte/repository/CvteDeviceRepository.java`) — `findByDeviceId`, `findByFactorySn`, null-safe JPQL `search()` by factorySn/deviceName/orgCode
- [[CvteDeviceResponse]], [[CvteDeviceSyncRequest]] (`cvte/dto/`) — response record with `from()` factory; sync request record
- [[KavaSignatureUtil]] (`cvte/client/KavaSignatureUtil.java`) — implements the Kava signing rules: sorts `x-kv-*` + query params by ASCII name, concatenates name+value (skips empties), digests with MD5(`secret+str+secret`) or HMAC_MD5 keyed by secret, renders 32-char uppercase hex; also `contentMd5()` for `x-kv-content-md5`
- [[KavaApiException]], [[KavaDeviceStatus]], [[KavaApiResult]] (`cvte/client/`) — error wrapper; normalised internal status record (tolerates boolean vs. integer `onlineStatus` across list/detail endpoints); generic `(data, code, message)` result wrapper with `summary()` for the "last API message" field
- [[KavaApiClient]] (`cvte/client/KavaApiClient.java`) — `RestClient`-based signed HTTP client (`searchDevices`, `getDeviceDetail`, `isConfigured()`); builds/signs `x-kv-*` headers; tolerant JSON parsing (`booleanOrNull`, etc.); secret never logged or exposed
- [[CvteDeviceSyncService]] (`cvte/service/CvteDeviceSyncService.java`) — `syncByCriteria` (search Kava + upsert matches), `pollAll`/`pollOne` (refresh known devices via device-detail endpoint, tolerant of per-device failures)
- [[CvteDevicePollingScheduler]] (`cvte/service/CvteDevicePollingScheduler.java`) — `@Scheduled` background refresh, `@ConditionalOnProperty(app.cvte.kava.polling-enabled=true)`, disabled by default
- [[CvteDeviceController]] (`cvte/controller/CvteDeviceController.java`) — `GET /api/v1/cvte/devices`, `POST /api/v1/cvte/devices/sync`, `POST /api/v1/cvte/devices/poll-now`, `POST /api/v1/cvte/devices/{deviceId}/poll-now`
- [[RobotRecommendationApiApplication]] — added `@EnableScheduling`
- [[application.properties]] — added `app.cvte.kava.*` properties bound to `CVTE_KAVA_BASE_URL/APP_ID/APP_SECRET/SIGN_TYPE/POLLING_ENABLED/POLLING_INTERVAL_MS`
- [[README]] (`RaasPal-Internal-Ops-backend/README.md`) — documented the 4 endpoints, 6 new env vars, and a "how to sync a device" walkthrough

**Frontend:**
- [[types/api.ts]] — added `CvteDeviceResponse`, `CvteDeviceSyncRequest`
- [[lib/api.ts]] — added `cvteApi` (`getAll`, `sync`, `pollAll`, `pollOne`)
- [[CvteStatusBadge]] (`components/CvteStatusBadge.tsx`) — shared online/offline/unknown badge (green/gray/amber)
- [[CvteStatusSummary]] (`components/CvteStatusSummary.tsx`) — compact Team Dashboard widget: online/offline/total counts + short device preview list + "View all" link to `/cvte`
- `app/[locale]/cvte/page.tsx` + [[CvteStatusClient]] (`app/[locale]/cvte/CvteStatusClient.tsx`) — full status page: sync form (factorySn/deviceName/orgCode), search, manual refresh + per-device "poll now", 45s `refetchInterval` auto-refresh, status table
- [[AppSidebar]] — added "CVTE C3 Status" nav entry (`/cvte`, `Radio` icon)
- [[page]] (`app/[locale]/page.tsx`, Team Dashboard) — embedded `<CvteStatusSummary />` between "How it works" and "Quick access"
- [[en.json]] / [[th.json]] / [[zh.json]] — added `nav.cvteStatus`, `teamDashboard.cvteSummary.*`, and full `cvte.*` namespace (syncForm, table, status labels) in all three locales

### Decisions Made

- **Fully separate entity/module, not merged into Robot/RobotSpec:** Explicit user instruction — "make a separate new entity for this... I want to test it for now... don't mix the codes yet." The `cvte` package shares no code, tables, or DTOs with `robot.*`; integration (if any) is deferred to a future decision.
- **Normalised `Boolean online` in [[KavaDeviceStatus]]:** The Kava list endpoint returns `onlineStatus` as a boolean while the detail endpoint returns it as an integer (1/0) — a tolerant parser (`booleanOrNull`) hides this inconsistency from the rest of the app.
- **Polling is opt-in (`CVTE_KAVA_POLLING_ENABLED=false` by default):** Manual sync + "poll now" cover the MVP need; background polling is available via `@ConditionalOnProperty` without forcing it on every deployment.
- **Dashboard summary widget added per user request mid-session:** Originally only a dedicated `/cvte` page was planned; user asked for "the tracking to also appear in the team dashboard with less detail," so [[CvteStatusSummary]] was added as a compact, separate component reusing [[CvteStatusBadge]].

### Unresolved / Next Steps

- [ ] Set real `CVTE_KAVA_BASE_URL` / `CVTE_KAVA_APP_ID` / `CVTE_KAVA_APP_SECRET` in local + Render env vars to test against the live Kava gateway (currently unconfigured — `isConfigured()` guards return a clear error)
- [ ] Decide later whether/how CVTE devices should relate to the Robot/RobotSpec catalog (explicitly deferred by user)
- [ ] Future extensions intentionally deferred: location/map view, movement detection, task tracking, robot control, alerts, telemetry history

---
## Session: 2026-06-12 — Telemetry Phase 1: Robot Units, Sync Core & Gausium Adapter

**Date:** 2026-06-12
**Tags:** #session #backend #database #config

### Summary

Built Phase 1 of the new `telemetry/` (and supporting `robotunit/`) packages: a brand-agnostic pipeline that syncs per-task cleaning reports from robot brand APIs into a shared `robot_task_reports` table, using the Adapter Pattern so future brands plug in without touching the registry or sync service. [[RobotUnit]]/[[Deployment]] (V11) model physical robots and which customer/site they're deployed at. [[RobotTaskReport]] (V12 + V13) stores the synced data with brand-specific columns left nullable. The first adapter, [[GausiumAdapter]], is backed by [[GausiumApiClient]] (OAuth token endpoint + paginated V2 List Robot Task Reports, contract confirmed against the official Gausium PDFs — note the real path is `/openapi/v2alpha1/robots/{sn}/taskReports`, not `/v2alpha1/...`) and [[GausiumOAuthService]] (persists/refreshes the access+refresh token pair in `gausium_oauth_tokens`, 5-minute expiry buffer; `expires_in` is an absolute epoch-ms timestamp per the docs). [[TelemetrySyncService]] loops all [[RobotUnit]]s with an active [[Deployment]], resolves the adapter by brand via [[TelemetryAdapterRegistry]], fetches reports for a date range, dedupes on `external_task_id`, and persists via `toEntity()` on [[TelemetryTaskReport]]. This is the foundation for Phase 2 (`report/` package), which will generate a monthly Excel report matching `docs/Cleaning plan_*.xlsx` exactly and POST it to a configurable n8n webhook for Email/LINE delivery, with a `testMode` flag so the internal team can validate the pipeline before real customers receive it. `mvn test` passes (BUILD SUCCESS).

### Files Modified
- [[V11__add_robot_units_and_deployments.sql]] (`db/migration/V11__add_robot_units_and_deployments.sql`) — `robot_units` (serial_number unique, brand, model, name) + `deployments` (robot_unit_id, customer_profile_id, site, is_active, deployed_at) tables
- [[RobotUnit]], [[Deployment]] (`robotunit/entity/`) + [[RobotUnitRepository]], [[DeploymentRepository]] (`robotunit/repository/`)
- [[V12__add_telemetry_tables.sql]] (`db/migration/V12__add_telemetry_tables.sql`) — `robot_task_reports` (28+ columns, shared across brands) + `gausium_oauth_tokens` + `customer_profiles.line_notify_token`
- [[V13__add_operator_to_robot_task_reports.sql]] (`db/migration/V13__add_operator_to_robot_task_reports.sql`) — adds `operator VARCHAR(255)` (Gausium-only field, kept for future use even though it's not in the Excel report)
- [[TelemetryAdapter]], [[TelemetryAdapterRegistry]], [[TelemetryTaskReport]] (added `operator` + `toEntity()`), [[TelemetrySyncService]] (`telemetry/core/`)
- [[RobotTaskReport]] (added `operator` column) + [[RobotTaskReportRepository]] (`telemetry/entity/`, `telemetry/repository/`)
- [[GausiumApiException]], [[GausiumOAuthToken]] + [[GausiumOAuthTokenRepository]], [[GausiumOAuthTokenResponse]] (`telemetry/adapters/gausium/`)
- [[GausiumApiClient]], [[GausiumOAuthService]], [[GausiumAdapter]], [[GausiumTaskReport]], [[GausiumConsumablesResidual]] (`telemetry/adapters/gausium/`)
- [[BadRequestException]] + [[GlobalExceptionHandler]] (`common/exception/`) — 400 handler, used by [[TelemetryAdapterRegistry]] for unknown brands
- [[application.properties]] — new `app.gausium.api.*` section (base-url, client-id, client-secret, open-access-key), added without touching the existing `app.cvte.kava.*` section

### Decisions Made

- **Adapter Pattern, fully brand-agnostic core:** [[TelemetryAdapterRegistry]] picks an adapter via `supports(brand)`; [[TelemetrySyncService]], [[RobotTaskReport]] and [[TelemetryTaskReport]] have no Gausium-specific logic. Adding a second brand only means adding a new adapter bean.
- **`operator` kept despite not being an Excel column:** User questioned this since it's not one of the 28 report columns; agreed to still capture it (V13 + DTO/entity field) as raw data from Gausium's API for potential future use.
- **Gausium OAuth `expires_in` is an absolute epoch-ms timestamp**, not a duration — confirmed by reading the official "Get/Refresh OAuth Token" PDFs; [[GausiumOAuthService]] stores it directly via `Instant.ofEpochMilli(...)` with a 5-minute refresh buffer.
- **V2 task reports path corrected:** confirmed via the official PDF to be `/openapi/v2alpha1/robots/{robotSerialNumber}/taskReports` (the earlier working spec was missing the `/openapi` prefix). Query uses non-deprecated `startTimeMin`/`startTimeMax` (robot-local `yyyy-MM-dd HH:mm:ss`), `total` parsed leniently from `JsonNode` since the API returns it as a string.
- **CVTE package untouched:** per explicit user instruction, the new `app.gausium.api.*` config is a separate section; no [[cvte]] code, tables, or properties were modified or reused as a template.

### Unresolved / Next Steps

- [ ] Phase 2 (`report/` package): `ReportGenerator`/registry + `GausiumReportGenerator` producing the exact 28-column "task-queue-list" sheet matching `docs/Cleaning plan_*.xlsx` (SimSun 14pt bold header, Grey25% fill, thin borders, no alternating rows/freeze pane)
- [ ] `MonthlyReportService` — multipart POST of generated `.xlsx` + JSON metadata to a configurable n8n webhook (Email/LINE handled by n8n)
- [ ] `ReportController` (`POST /api/v1/reports/monthly/generate?month=...&testMode=true`, ADMIN-only) + `MonthlyReportScheduler` (daily sync + monthly report crons, disabled by default via `app.reports.scheduler-enabled`)
- [ ] Add `CustomerProfile.lineNotifyToken` field (DB column already added via V12)
- [ ] Set real `GAUSIUM_API_CLIENT_ID` / `GAUSIUM_API_CLIENT_SECRET` / `GAUSIUM_API_OPEN_ACCESS_KEY` to activate Gausium sync against the live API (currently unconfigured — `isConfigured()` guard returns a clear error)

---

## Session: 2026-06-13 — Telemetry Phase 2: lineNotifyToken field, residual-pct precision fix, GausiumReportGenerator

**Date:** 2026-06-13
**Tags:** #session #backend #database

### Summary
Continued Telemetry Phase 2. Added the [[CustomerProfile]] `lineNotifyToken` field (column already existed from V12). The user then provided a real reference file, `docs/Cleaning plan_20260612093537.xlsx`, which was unzipped and inspected (`xl/styles.xml`, `xl/sharedStrings.xml`, `xl/worksheets/sheet1.xml`) to extract the exact 28-column "task-queue-list" layout: headers, column widths (column U hidden), and header styling (SimSun 14pt bold, Grey25% fill, full thin border, center/wrap/vcenter, locked). Walked the user through column-by-column source mapping against [[RobotTaskReport]] and the Gausium V1/V2 "List Robot Task Reports" API docs; the user gave final sign-off on all 28 columns (see Decisions). While mapping columns P/Q/R (Brush/Filter/Squeegee residual %), discovered the entity/DTO modeled these as `Integer` but Gausium's API and the reference report show 2-decimal values (e.g. 99.92) — added [[V14__widen_residual_pct_to_decimal.sql]] and updated the DTO/entity/adapter chain to `BigDecimal`/`Double`. Implemented the new `report/` package: [[ReportGenerator]] interface + [[ReportGeneratorRegistry]] (mirrors [[TelemetryAdapter]]/[[TelemetryAdapterRegistry]]), and [[GausiumReportGenerator]] which builds the full 28-column XSSFWorkbook via Apache POI (already a dependency). Added [[GausiumReportGeneratorTest]] covering brand-matching, header text/styling, hidden column U, and row-2 formatted values against the reference Excel's sample data. `mvn -DskipITs test` — BUILD SUCCESS, 3/3 tests pass (1 pre-existing + 2 new).

### Files Modified
- [[CustomerProfile]] (`customer/entity/CustomerProfile.java`) — added `lineNotifyToken` field mapped to existing `line_notify_token` column (V12)
- [[V14__widen_residual_pct_to_decimal.sql]] (`db/migration/V14__widen_residual_pct_to_decimal.sql`) — widens `brush_residual_pct`, `filter_residual_pct`, `suction_blade_residual_pct` from INT to `DECIMAL(5,2)`
- [[GausiumConsumablesResidual]] (`telemetry/adapters/gausium/dto/GausiumConsumablesResidual.java`) — `brush`/`filter`/`suctionBlade` changed `Integer` → `BigDecimal`
- [[RobotTaskReport]] (`telemetry/entity/RobotTaskReport.java`) — `brushResidualPct`/`filterResidualPct`/`suctionBladeResidualPct` changed `Integer` → `BigDecimal` with `precision = 5, scale = 2`
- [[TelemetryTaskReport]] (`telemetry/core/TelemetryTaskReport.java`) — same three fields changed `Integer` → `Double`; `toEntity()` now uses existing `toBigDecimal(Double)` helper for them
- [[GausiumAdapter]] (`telemetry/adapters/gausium/GausiumAdapter.java`) — maps `residual.brush()/filter()/suctionBlade()` via `.doubleValue()` into the now-`Double` [[TelemetryTaskReport]] fields
- [[ReportGenerator]] (`report/core/ReportGenerator.java`) — new interface: `getBrand()`, `supports(brand)`, `generate(List<RobotTaskReport>)` → `byte[]` xlsx
- [[ReportGeneratorRegistry]] (`report/core/ReportGeneratorRegistry.java`) — new registry, picks a generator by `supports(brand)`, throws [[BadRequestException]] if none match
- [[GausiumReportGenerator]] (`report/generators/gausium/GausiumReportGenerator.java`) — new: builds the 28-column "task-queue-list" XSSFWorkbook (headers, column widths incl. hidden column U, SimSun/Grey25%/thin-border header style, Calibri body style) from [[RobotTaskReport]] rows
- [[GausiumReportGeneratorTest]] (`report/generators/gausium/GausiumReportGeneratorTest.java`) — new JUnit5+AssertJ test: brand matching, header text/style/hidden-column assertions, and row-2 value formatting against the reference Excel's sample row

### Decisions Made
- **Column source mapping for [[GausiumReportGenerator]] (all 28 columns confirmed with user):** A/B start/end time, C map name, D cleaning plan, E/F robot name + S/N, G task completion %, H planned area, I/J total time (text + decimal hours), K actual cleaning area, L work efficiency, M water usage, N/O start/end battery %, P/Q/R brush/filter/squeegee residual %, S/T planned/actual polishing area, W uncleaned area (= H − K), X "Plan running time" reuses the working-time clock format (`HH: MM: SS`), Y "Task type" = raw `cleaningMode` (e.g. "Floor Washing"), AA download link = `taskReportPngUri`. Columns U ("Receive task report time"), V ("Task start mode"), Z ("Remarks") and AB ("Task status") are left blank — not exposed by Gausium's public Task Reports API.
- **AB "Task status" deferred:** user will obtain the `taskEndStatus` (int) → label mapping table from the Gausium supplier later and provide it in a future session; code currently emits `""` with a comment flagging the placeholder.
- **Residual-percentage precision bug found opportunistically:** while mapping columns P/Q/R, realized `Integer` couldn't hold Gausium's real `99.92`-style values — fixed via V14 + DTO/entity/adapter chain rather than deferring, since it would have broken real-API sync once Gausium credentials are configured.
- **POI confirmed sufficient:** `poi-ooxml` 5.3.0 already in [[pom.xml]] ([[ProposalExportService]] precedent) — no new dependency needed for [[GausiumReportGenerator]].

### Unresolved / Next Steps
- [ ] AB "Task status" — map `taskEndStatus` int codes to labels once the user provides the table from the Gausium supplier
- [ ] `MonthlyReportService` — multipart POST of generated `.xlsx` + JSON metadata to a configurable n8n webhook
- [ ] `ReportController` (`POST /api/v1/reports/monthly/generate?month=...&testMode=true`, ADMIN-only) + `MonthlyReportScheduler`
- [ ] Set real `GAUSIUM_API_CLIENT_ID` / `GAUSIUM_API_CLIENT_SECRET` / `GAUSIUM_API_OPEN_ACCESS_KEY` to activate Gausium sync against the live API

---

## Session: 2026-06-13 — Telemetry Phase 2: monthly report delivery (Supabase Storage + n8n + LINE)

**Date:** 2026-06-13
**Tags:** #session #backend #database #deployment

### Summary
Built the delivery half of Telemetry Phase 2 (the [[GausiumReportGenerator]] from earlier today produces the files; this session ships them). After a design discussion the user chose: **deliver via n8n** (backend POSTs JSON to one webhook; n8n owns the LINE Messaging API call + Flex message + credentials), **LINE-only** channel, **one xlsx per robot** but **one LINE message per customer** (bundle all of a customer's robot download links into one Flex message to minimise LINE push cost — LINE charges per push beyond the free monthly quota; n8n does not change that cost). Key correction surfaced to the user: **LINE Notify was shut down 2025-03-31**, so the existing `line_notify_token` is dead; LINE delivery now needs the **LINE Messaging API** (an Official Account + a recipient id), and since LINE can't attach files the message carries **download links**. Files are hosted on **Supabase Storage with signed 30-day URLs** (backend uploads + signs; n8n only needs LINE creds). Trigger is a **monthly cron (off by default) + a manual ADMIN endpoint that defaults to `testMode=true`** — test mode generates + uploads the files and returns the signed URLs **without** sending to n8n/LINE, so the team can verify everything with zero LINE spend. Added a standalone (non-Flyway) seed script + a `MonthlyReportService` unit test. `mvn -DskipITs test` — BUILD SUCCESS, 7/7 tests pass (1 pre-existing context test + 2 GausiumReportGenerator + 4 new MonthlyReportService).

### Files Modified
- [[V15__add_line_user_id_to_customer_profiles.sql]] (`db/migration/`) — adds `line_user_id VARCHAR(255)`; deprecation comment on `line_notify_token`
- [[CustomerProfile]] (`customer/entity/`) — added `lineUserId` field (holds LINE user/group/room id interchangeably); `@Deprecated` on `lineNotifyToken`
- [[RobotTaskReportRepository]] (`telemetry/repository/`) — added `findByReportMonthWithRefs(month)` JPQL with `join fetch robotUnit/customerProfile` (needed since `open-in-view=false` + LAZY refs)
- [[SupabaseStorageService]] + `SupabaseStorageException` (`report/storage/`) — RestClient upload (`POST /storage/v1/object/{bucket}/{path}`, `x-upsert`) + sign (`POST /storage/v1/object/sign/...`, `expiresIn`) → absolute signed URL; `isConfigured()` guard
- [[N8nReportClient]] + `N8nDeliveryException` + [[MonthlyReportPayload]] (record w/ nested `RobotReportLink`) (`report/delivery/`) — RestClient JSON POST to `app.reports.n8n.webhook-url`; `isConfigured()` guard
- [[MonthlyReportService]] + [[MonthlyReportSummary]] (`report/service/`) — loads month, groups by customer→robot, generates xlsx per robot via [[ReportGeneratorRegistry]], uploads+signs, bundles one payload per customer; `testMode` uploads but never sends; per-robot/per-send errors are caught + counted, not fatal
- [[ReportController]] (`report/controller/`) — `POST /api/v1/reports/monthly/run?month=YYYY-MM&testMode=true` (testMode **defaults true** for safety); authenticated (no method-role gating wired yet)
- [[MonthlyReportScheduler]] (`report/scheduler/`) — `@ConditionalOnProperty(app.reports.scheduler-enabled=true)` + `@Scheduled(cron=...)`; sends previous month live; mirrors [[CvteDevicePollingScheduler]]
- [[application.properties]] — new `app.reports.*` + `app.supabase.*` section (all blank/false by default)
- `docs/seed-test-monthly-report.sql` (NEW, **not** a Flyway migration) — idempotent test fixture: 1 user + customer (`line_user_id` placeholder) + robot + active deployment + 3 task reports for `2026-05`, with a commented cleanup block
- [[MonthlyReportServiceTest]] (`report/service/`, NEW) — Mockito: test-mode uploads-per-robot + zero sends; live-mode one-payload-per-customer + skips null `lineUserId`; object-path `customerId/month/serial.xlsx`; empty month no-ops

### Decisions Made
- **n8n owns LINE, backend owns files:** backend uploads to Supabase + signs, then POSTs a clean per-customer JSON payload (`customerId/Name`, `lineUserId`, `reportMonth`, `robots[]{robotName, serialNumber, downloadUrl}`) to n8n. The LINE channel token + Flex JSON + the actual push live **only in n8n**, isolating LINE's volatility from the deploy cycle.
- **Cost model made explicit:** n8n does not make LINE free — LINE charges per **push** message beyond the OA free monthly quota (reply messages are free). So file-per-robot but **message-per-customer** keeps message count ~= customers, not customers×robots.
- **`testMode` defaults to true** on the manual endpoint so nobody accidentally spends LINE messages; live send requires `testMode=false` and a configured n8n webhook (guarded by `BadRequestException`).
- **Internal test path without real Gausium data:** standalone seed SQL (dev DB only) → `testMode=true` returns signed Supabase URLs to verify the Excel; then point a test customer's `line_user_id` at a team LINE group/user id captured by the n8n webhook for a real `testMode=false` send.
- **`line_user_id` is recipient-type-agnostic** (user/group/room) so the same column supports a team-group test and real per-customer sends without schema change.

### Unresolved / Next Steps
- [ ] Build the **n8n flow** (webhook → host/sign already done by backend, so just: receive payload → build LINE Flex with one button per robot → push to `lineUserId`) and create the **LINE Official Account**; capture a team LINE group id for internal testing
- [ ] Create the Supabase private bucket `monthly-reports` and set `SUPABASE_URL`/`SUPABASE_SERVICE_KEY` (+ `REPORTS_N8N_WEBHOOK_URL`) in env
- [ ] AB "Task status" column mapping in [[GausiumReportGenerator]] — still awaiting `taskEndStatus`→label table from the Gausium supplier
- [ ] Set real `GAUSIUM_API_CLIENT_ID`/`SECRET`/`OPEN_ACCESS_KEY` to sync real task reports; flip `app.reports.scheduler-enabled=true` only after the n8n flow + LINE OA are verified

---

## Session: 2026-06-14 — Frontend Tools page (robot monitoring + report automation tabs)

**Date:** 2026-06-14
**Tags:** #session #frontend

### Summary
Added a consolidated **Tools** page to [[robot-recommendation-web-raaspal]] with two tabs: **Monitor robots** (the existing CVTE C3 status UI) and **Manage report automation** (new UI driving the monthly report system from session 2026-06-13). The report tab lets the team pick a month (defaults to previous month), **Generate & preview** (calls `POST /api/v1/reports/monthly/run?testMode=true` → uploads files + shows signed Supabase download links, no LINE spend), or **Send now to LINE** (testMode=false, behind an inline confirm warning about LINE cost). Results render as stat chips (customers / report files / messages sent / no-recipient) plus a per-customer list of robot download links. Refactored the CVTE monitoring out of the old standalone page into a reusable self-contained [[CvteMonitorPanel]] (own search box, no page chrome). The sidebar "CVTE C3 Status" item was replaced by a single **Tools** item; `/cvte` now renders the Tools page on the monitor tab (legacy link kept), and the Team Dashboard's [[CvteStatusSummary]] "View all" now points to `/tools`. Backend [[ReportController]] was changed to wrap its result in `ApiResponse<MonthlyReportSummary>` (matching every other controller / the frontend's `ApiResponse<T>` typing). i18n keys added to en/th/zh. `npm run build` ✓ (TypeScript + 50 static pages, /tools generated for all 3 locales); backend `mvn compile` BUILD SUCCESS.

### Files Modified
- [[ReportController]] (`report/controller/`) — now returns `ApiResponse.success(message, summary)` instead of a bare record
- `types/api.ts` — added `MonthlyReportSummary`, `MonthlyReportPayload`, `RobotReportLink`
- `lib/api.ts` — added `reportApi.runMonthly(month, testMode)`
- [[CvteMonitorPanel]] (`components/`, NEW) — self-contained CVTE sync+table+poll panel with its own search (extracted from the deleted `CvteStatusClient`)
- [[ReportAutomationPanel]] (`components/`, NEW) — month picker, preview/send-now (with confirm), result summary + download links; reads axios `response.data.message` on error
- `app/[locale]/tools/ToolsClient.tsx` + `page.tsx` (NEW) — page chrome (sidebar/topbar) + tab switcher; `ToolsClient` takes an optional `initialTab`
- `app/[locale]/cvte/page.tsx` — now renders `<ToolsClient initialTab="monitor" />`; `CvteStatusClient.tsx` deleted
- [[AppSidebar]] — replaced "CVTE C3 Status" (Radio) nav item with "Tools" (Wrench, `/tools`)
- [[CvteStatusSummary]] — "View all" link `/cvte` → `/tools`
- `messages/{en,th,zh}.json` — new `nav.tools`, `tools.*`, `reports.*` namespaces

### Decisions Made
- **Consolidate under one Tools page rather than separate nav items:** CVTE monitoring + report automation are both internal ops tools; tabs keep the sidebar short. Legacy `/cvte` route preserved (renders Tools/monitor) so existing links/bookmarks/dashboard widget keep working.
- **Send-now defaults to safe:** the live "Send now to LINE" button requires an inline confirm step (warns about LINE push cost); the endpoint itself also defaults `testMode=true`. Preview uploads real files + returns signed URLs so the team verifies output before any send.
- **Self-contained panels:** each tab panel owns its own search/state so it can be embedded without depending on the shared top bar — `CvteMonitorPanel` moved its search out of `AppTopBar` into the panel.

### Unresolved / Next Steps
- [ ] Same go-live items as 2026-06-13: create Supabase `monthly-reports` bucket, set `SUPABASE_*` + `REPORTS_N8N_WEBHOOK_URL`, build the n8n flow + LINE OA, capture a team LINE id for testing
- [ ] Optional: surface scheduler on/off + last-run status on the report tab (backend doesn't expose this yet — currently just an explanatory note)

---

---
## Session: 2026-06-16 — Login page "Bold & Premium" redesign

**Date:** 2026-06-16
**Tags:** #session #frontend

### Summary
Enhanced the UI's first impression by redesigning the login page in a "Bold & Premium" direction (user-selected from offered choices; scope limited to the login page this pass). Replaced the single centred card with a split-screen layout: a vibrant teal→blue animated-aurora brand panel on the left (glow orbs, logo lockup, headline, and three AI feature highlights) beside a frosted-glass sign-in card on the right. The brand panel collapses below `lg`, leaving the glass card over a soft gradient backdrop with a compact brand header. Form inputs were rounded/glassed and the submit button became a gradient with brand glow. All auth logic, cold-start retry, and i18n approach left untouched — purely visual. `npm run build` passes (login prerenders for en/th/zh).

### Files Modified
- [[globals.css]] (`app/globals.css`) — added `float-orb`/`float-orb-alt`/`aurora-pan` keyframes + opt-in animation utilities, with `prefers-reduced-motion` guard
- [[login/page.tsx]] (`app/[locale]/login/page.tsx`) — rewrote as split-screen gradient brand panel + glass card
- [[LoginForm]] (`app/[locale]/login/LoginForm.tsx`) — rounded glass inputs, gradient glow submit button (logic unchanged)

### Decisions Made
- **Bold & Premium over Modern Polish / Clean Enterprise:** user choice; highest visual impact for the customer/stakeholder-facing entry point.
- **Scope limited to login this pass:** user opted to validate the direction on one page before rolling it across the shell/dashboard.
- **Animations opt-in + reduced-motion safe:** scoped utility classes so no other page is affected.

### Unresolved / Next Steps
- [ ] If direction is approved, extend "Bold & Premium" to the global shell (sidebar + top bar) and dashboard hero
- [ ] Consider i18n strings for the new login copy (currently hardcoded English, matching prior login page)
---

---
## Session: 2026-06-16 — Hybrid premium UI: dashboard hero + global shell

**Date:** 2026-06-16
**Tags:** #session #frontend

### Summary
Continued the UI enhancement with the agreed "hybrid" strategy: Bold & Premium on showcase surfaces, Modern SaaS Polish on the shared frame. Upgraded the Team Dashboard hero to the same teal→blue animated-aurora gradient + glow-orbs + glass-pill language as the login page (replacing the flat `--app-hero` block), and added depth/gradient icon chips + smoother hover to the how-it-works steps and quick-link cards. Polished the global shell: sidebar now has a gradient logo chip, a gradient active-item pill with a left accent bar, brand-tinted icons, and a gradient user avatar; the top bar's notification button is rounded with a brand hover + unread dot. Reused the existing `animate-aurora`/`animate-float-orb` utilities (no new CSS). `npm run build` compiles + prerenders all 50 pages.

### Files Modified
- [[page.tsx]] (`app/[locale]/page.tsx`) — Bold & Premium hero (aurora + orbs + glass pill), upgraded step & quick-link cards
- [[AppSidebar]] (`components/AppSidebar.tsx`) — gradient logo chip, gradient active pill + accent bar, brand-hover icons, gradient avatar
- [[AppTopBar]] (`components/AppTopBar.tsx`) — rounded brand-hover notification button with unread dot

### Decisions Made
- **Hybrid over single style everywhere:** Bold & Premium reads as noisy behind dense data; reserve it for showcase surfaces (login ✅, dashboard hero ✅) and use Modern Polish on the working frame so the app stays cohesive and readable.

### Unresolved / Next Steps
- [ ] Apply Modern SaaS Polish to working pages: robots catalog, solutions, proposals list, generate-solution flow, forms
- [ ] Optionally extend Bold & Premium to the proposal view (customer-facing output)
---

---
## Session: 2026-06-16 — Hybrid premium UI rollout: all remaining surfaces

**Date:** 2026-06-16
**Tags:** #session #frontend

### Summary
Completed the hybrid UI enhancement across the rest of the app. Added a reusable `.bg-aurora` utility (teal→blue gradient) in [[globals.css]] so every hero banner shares one premium gradient with minimal markup. Applied Bold & Premium to the customer-facing proposal view (gradient header banner + glow orb + glass pill, plus a gradient top-bar accent on the document body) and to the generate-solution hub. Swapped all four flow-step banners (upload / recommendation / proposal / solutionType form) from the flat `--app-hero` block to the aurora gradient. Lifted the shared shadcn primitives — `Button` default now has a brand-glow shadow + `active:scale`, rounded-lg; `Card` is rounded-2xl — which polishes every page at once. Converted leftover `--app-hero` icon chips/avatars (Proposals/Solutions empty states, TopNavigationMenu) to gradient chips, and aligned the mobile `TopNavigationMenu` (the real nav below 1680px) with the sidebar's gradient logo chip + gradient active pill + accent bar. `npm run build` compiles + prerenders all 50 pages.

### Files Modified
- [[globals.css]] (`app/globals.css`) — added reusable `.bg-aurora` gradient utility
- [[button.tsx]] / [[card.tsx]] (`components/ui/`) — brand-glow shadow + active-scale button, rounded-2xl card
- [[ProposalViewClient]] (`proposals/[id]/`) — Bold & Premium gradient header + glow orb + gradient body accent
- [[generate-solution/page.tsx]] + `[solutionType]/{page,upload,recommendation,proposal}` — aurora flow banners
- [[ProposalsClient]] / [[SolutionsClient]] — gradient empty-state icon chips
- [[TopNavigationMenu]] — gradient logo chip + gradient active pill + accent bar + gradient avatar

### Decisions Made
- **Single `.bg-aurora` utility over repeated inline gradients:** one source of truth for the brand gradient; banners become a one-line class swap and stay consistent.
- **Primitive-level polish (Button/Card):** lifts every page uniformly without touching each page's markup.

### Unresolved / Next Steps
- [ ] Optional: richer Markdown rendering for the proposal body (currently a styled <pre>)
- [ ] Optional: i18n the hardcoded English copy added to the login page
---

---
## Session: 2026-06-16 — Public /register page (UI only)

**Date:** 2026-06-16
**Tags:** #session #frontend

### Summary
Added a public self-service sign-up page at `/[locale]/register`, mirroring the login page's Bold & Premium split-screen (gradient brand panel + frosted-glass card). UI-only by design: the backend exposes no `POST /api/v1/auth/register` (only `/api/v1/auth/login` is public; `POST /api/v1/users` is JWT-protected). The form does full client-side validation (name, email format, password ≥8, confirm match) and is gated behind a `REGISTRATION_ENABLED = false` flag — a valid submit shows a "self-service sign-up isn't switched on yet, ask an admin" confirmation instead of calling the API. A future-ready `authApi.register` helper + `RegisterRequest` type were added so going live is: ship the endpoint, whitelist it in SecurityConfig, flip the flag (success path already auto-logs-in + redirects like LoginForm). `/register` was whitelisted in [[proxy.ts]] PUBLIC_PATHS so it's reachable without a JWT. Cross-links added between login ("Create one") and register ("Sign in"). `npm run build` prerenders all 53 pages.

### Files Modified
- [[proxy.ts]] (`proxy.ts`) — added `/register` to PUBLIC_PATHS
- [[api.ts]] (`lib/api.ts`) — `authApi.register` stub (⚠ endpoint not live)
- [[types/api.ts]] (`types/api.ts`) — new `RegisterRequest` type
- `app/[locale]/register/page.tsx` (new) — Bold & Premium split-screen sign-up
- `app/[locale]/register/RegisterForm.tsx` (new) — validated form, `REGISTRATION_ENABLED` flag + placeholder
- [[login/page.tsx]] (`app/[locale]/login/page.tsx`) — "Create one" link to /register

### Decisions Made
- **UI-only behind a feature flag, not a fake API call:** the backend register endpoint doesn't exist and a public one conflicts with the MVP "no public self-service" rule; the flag makes future wiring a one-line change without shipping a broken call today.
- **No role field on the public form:** a self-service sign-up must not let users self-assign ADMIN; role would default server-side when the endpoint is built.

### Unresolved / Next Steps
- [ ] Backend: add `POST /api/v1/auth/register` (returns AuthResponse, default role SPECIALIST) + whitelist in [[SecurityConfig]], then set `REGISTRATION_ENABLED = true`
- [ ] Decide whether public sign-up is actually desired vs. internal "Add team member" admin UI (this contradicts the documented MVP "no public self-service" rule)
- [ ] i18n the hardcoded English copy on login + register
---

---
## Session: 2026-06-19 — Task-status mapping + weekly report cadence

**Date:** 2026-06-19
**Tags:** #session #backend #ai #report

### Summary
Two backend features for the upcoming report demo. (1) **Task status (column AB)** — the supplier finally provided the `taskEndStatus` code table, so `GausiumReportGenerator` now maps 0→"Normal completion", 1→"Manual termination", 2→"Abnormal termination", 3→"Startup failure" (null/unknown → blank). The field was already plumbed end-to-end (Gausium DTO → adapter → `RobotTaskReport.taskEndStatus`); only the xlsx rendering was a blank placeholder. (2) **Weekly report cadence** — generalized the month-only pipeline to support an ISO-week date range without disturbing the monthly path. Refactored `MonthlyReportService` to extract a shared `run(periodLabel, reports, testMode)` plus `requireConfigured()`; `generateAndSend(month)` is byte-for-byte unchanged, new `generateAndSendWeekly(weekStart)` filters by a `start_time` range. Added repo `findByStartTimeBetweenWithRefs`, `POST /api/v1/reports/weekly/run?weekStart=YYYY-MM-DD&testMode=true`, a separate `WeeklyReportScheduler` (own `app.reports.weekly-scheduler-enabled` toggle, Mon 06:00 cron), and config defaults. Period label is `YYYY-Www` (e.g. 2026-W25), reusing the payload/summary `reportMonth` field as a generic period label so n8n/frontend shapes are untouched. `mvn test` 10/10 (added a weekly service test + a task-status mapping test).

**Build/test note:** no system `mvn` on PATH; use IntelliJ's bundled Maven at `…/IntelliJ IDEA 2026.1.1/plugins/maven/lib/maven3/bin/mvn` with `JAVA_HOME=C:\Program Files\Java\jdk-21.0.10`. The IntelliJ JBR is JDK 25, which breaks Mockito inline mocks ("Could not modify all classes") — must run tests under JDK 21.

### Files Modified
- [[GausiumReportGenerator]] (`report/generators/gausium/`) — `formatTaskStatus()` + column AB rendering + Javadoc
- [[MonthlyReportService]] (`report/service/`) — shared `run()`/`requireConfigured()`, `generateAndSendWeekly()`, `resolveWeek()`/`WeekRange`
- [[RobotTaskReportRepository]] (`telemetry/repository/`) — `findByStartTimeBetweenWithRefs`
- [[ReportController]] (`report/controller/`) — `POST /weekly/run`
- `report/scheduler/WeeklyReportScheduler.java` (new) — weekly cron, own toggle
- [[application.properties]] — `app.reports.weekly-scheduler-{enabled,cron}`
- Tests: `GausiumReportGeneratorTest` (+status mapping), `MonthlyReportServiceTest` (+weekly)

### Decisions Made
- **Reuse MonthlyReportService/Summary/Payload for weekly** rather than rename everything: the `reportMonth` field becomes a generic period label; avoids rippling into n8n + frontend types. Clean rename is a possible later follow-up.
- **Separate WeeklyReportScheduler** so monthly/weekly cadences toggle independently.
- **Unknown/null status → blank** (not a bare number), matching the generator's existing "no source → blank" convention.

### Unresolved / Next Steps
- [ ] (Optional) Frontend trigger for weekly in [[ReportAutomationPanel]] — currently only `reportApi.runMonthly` is wired; add a `runWeekly` + button
- [ ] Live config still required for an actual LINE send: Supabase bucket/keys, n8n flow, LINE OA (unchanged from before)
- [ ] Laos reachability: VPN works (ISP/geo block of `.vercel.app`/`.onrender.com`); durable fix = custom domain — deferred
---

### Addendum (same day) — Weekly frontend trigger wired
Completed the optional next-step: [[ReportAutomationPanel]] now has a Monthly/Weekly segmented toggle. Weekly swaps the month input for a "Week starting" date picker (defaults to previous week's Monday, max today) and the mutation calls new `reportApi.runWeekly`. Result/confirm copy reuses the summary's `reportMonth` (carries the `2026-Www` label) plus a generic `periodLabel`. Added i18n keys `cadenceLabel/cadenceMonthly/cadenceWeekly/weekStartLabel` to en/th/zh. `npm run build` compiles clean. Weekly is now end-to-end: endpoint + scheduler + UI.

---
## Session: 2026-06-21 — Report delivery pivot: drop n8n/LINE → email + web report page

**Date:** 2026-06-21
**Tags:** #session #frontend #ai

### Summary
Manager changed report delivery from LINE (via n8n) to **email**, and asked for a richer "Monthly Robot Performance Report" with header info + 5 parts (Executive Summary, Value Delivered, Robot Performance, Robot Health Status traffic-light, Recommendations). Decided to **drop n8n entirely** (send email straight from the backend, simpler to maintain) and make the report a **public, automatically-generated web page** linked from the email — no human edits, no PPT/Excel. Built the frontend report page first (UI structure) with sample data; backend email + public-report endpoint are the next steps. Charting is done with plain CSS bars (no new dependency); colours use existing design tokens (brand teal `primary`, traffic lights `success`/`warning`/`danger`). Report entity is **brand-namespaced** (Gausium) to mirror the backend ReportGenerator-by-brand adapter pattern, since other brands' report types will be added later. `npm run build` compiles clean; new route `ƒ /[locale]/report/[token]` server-renders.

### Files Modified
- `lib/reports/types.ts` (new) — shared primitives (`HealthState`, `ReportKpi`, `PerformanceBar`, `HealthItem`, `Recommendation`) + base `MonthlyPerformanceReport` with `RobotBrand` discriminator
- `lib/reports/gausium.ts` (new) — `GausiumMonthlyReport` entity (`brand: 'GAUSIUM'`) + `sampleGausiumReport` preview data
- `components/report/MonthlyReportView.tsx` (new) — brand-agnostic Server Component rendering header info block + Parts #1–#5, CSS bar chart, traffic-light health, print-friendly
- `app/[locale]/report/[token]/page.tsx` (new) — public report route; renders sample data, `token` reserved for real fetch
- [[proxy.ts]] — added `/report` to `PUBLIC_PATHS` so customers reach the page without a JWT

### Decisions Made
- **Drop n8n, email from backend:** one service, one place for secrets, no webhook hop — lower maintenance. (Existing [[N8nReportClient]]/`MonthlyReportPayload.lineUserId` to be retired when backend email lands.)
- **Web report page over Excel/PPT/PDF:** a web page renders live from data with zero human editing — best fit for "fully automatic," reuses Next.js stack. Optional auto-PDF later.
- **Public token route, not customer login:** customers have no accounts; `/report/<token>` is a read-only shareable page guarded by an unguessable token (mirrors signed-URL idea), not the out-of-scope customer dashboard/login.
- **Brand-namespaced report entity:** `GausiumMonthlyReport extends MonthlyPerformanceReport`; shared view renders the common 5-part layout, brand-specific data entities added per brand (matches backend adapter-by-brand).
- **CSS bar chart, no charting lib:** prints cleanly, nothing to break on upgrade (no Recharts dependency added).

### Unresolved / Next Steps
- [ ] Backend: add customer **email** field to [[CustomerProfile]] (next free Flyway is now `V17`; today it has only `companyName`/`contactPhone`/`lineUserId`)
- [ ] Backend: `spring-boot-starter-mail` + SMTP config + `EmailReportClient`, retire [[N8nReportClient]]
- [ ] Backend: public report endpoint `GET /api/v1/reports/public/{token}` returning the `MonthlyPerformanceReport` JSON; generate + store report tokens
- [ ] Wire the page to real data (replace `sampleGausiumReport` with a fetch keyed on `token`)
- [ ] Decide SMTP relay (Gmail SMTP to start vs Brevo/SendGrid) — needs a company email account/domain (manager)
- [ ] Optional: "Download PDF" of the report page (client component + `window.print()` or jspdf)
- [ ] i18n for the report page (currently English-only; th/zh later)
---

---
## Session: 2026-06-21 — Robot registration + deployment API (Phase 1 of report-targeting feature)

**Date:** 2026-06-21
**Tags:** #session #backend #database

### Summary
Manager wants an admin feature to register robots by SN, link them to customers, set per-robot report timing (Monthly/Weekly/Off), choose **per-customer vs per-robot** sending, and **search both directions** (customer→robots, robot SN→customer). Found there was **no API/UI to create `RobotUnit`/`Deployment`** at all (only the dev seed SQL), and cadence was global. Built **Phase 1 (backend foundation)**: robot registration + customer linking + per-deployment cadence + bidirectional search. Compiles clean (`mvnw -o clean compile` → BUILD SUCCESS). Phases 2 (cadence-aware schedulers + per-customer/per-robot send grouping) and 3 (admin UI) still pending.

### Files Modified
- `db/migration/V16__add_report_cadence_to_deployments.sql` (new) — adds `report_cadence VARCHAR(20) NOT NULL DEFAULT 'MONTHLY'` to `deployments`
- `robotunit/entity/ReportCadence.java` (new) — enum `MONTHLY | WEEKLY | OFF`
- [[Deployment]] (`robotunit/entity/`) — added `reportCadence` field (`@Enumerated(STRING)`, default MONTHLY)
- [[RobotUnitRepository]] — added `existsBySerialNumber`
- `robotunit/dto/RegisterRobotRequest.java`, `UpdateCadenceRequest.java`, `RobotUnitResponse.java` (new) — `RobotUnitResponse` carries both sides of the link (robot + its `DeploymentInfo` customer) so one payload serves both search directions
- `robotunit/service/RobotUnitService.java` (new) — register (unique-SN guard), listAll, listByCustomer, getBySerialNumber, updateCadence, deactivate
- `robotunit/controller/RobotUnitController.java` (new) — `/api/v1/robot-units`

### Endpoints (Phase 1)
- `POST /api/v1/robot-units` — register robot (SN/brand/model/name) + deploy to `customerProfileId` (+ site, cadence)
- `GET /api/v1/robot-units` — all; `?customerId=` → that customer's robots; `?serialNumber=` → the robot + its customer
- `PATCH /api/v1/robot-units/deployments/{deploymentId}/cadence` — set Monthly/Weekly/Off
- `DELETE /api/v1/robot-units/deployments/{deploymentId}` — deactivate

### Decisions Made
- **Cadence lives on `Deployment`, not global:** a deployment = one robot at one customer, so per-deployment cadence lets each customer/robot run independently.
- **One `RobotUnitResponse` for both search directions:** embedding the customer (`DeploymentInfo`) in the robot payload means the UI shows the robot when searching by customer and the customer when searching by SN — no separate shapes.
- **Register + deploy in one call:** a robot is only useful once tied to a customer (reports are per-customer), so registration creates the deployment too. Re-deployment/transfer deferred.
- **`send_mode` (per-customer vs per-robot) deferred to Phase 2:** it only matters at send/grouping time; will decide whether it lives on `CustomerProfile` or as a run option then.

### Unresolved / Next Steps
- [ ] Phase 2: make `MonthlyReportScheduler`/`WeeklyReportScheduler` cadence-aware (filter deployments by `reportCadence`); `OFF` skipped
- [ ] Phase 2: per-customer vs per-robot send grouping in [[MonthlyReportService]] (+ where `send_mode` is stored)
- [ ] Phase 3: admin "Robots & Deployments" UI — bidirectional search, register form, cadence + send-mode controls
- [ ] Customer needs a real **email** field (V17) for the email delivery pivot (separate but related)
---

### Addendum (same day) — Report layout preview tool (for manager format sign-off)
Added a **"Report Preview"** tab to the Tools page so the report web-page format can be confirmed with the
manager before live telemetry is wired. New `components/ReportPreviewPanel.tsx` (client): lists robots via
`robotUnitApi.list()`, client-side search by SN/name/customer, plus a **"Preview with sample data"** shortcut
that always works even with zero robots registered. Selecting a robot renders the real
`MonthlyReportView` with that robot's **real header** (customer/site/SN) + a month picker; metrics are
representative sample values (amber "layout preview only" banner makes that explicit). Wiring: `robotUnitApi`
(+ `RobotUnitResponse`/`DeploymentInfo`/`ReportCadence`/`RegisterRobotRequest` types) in `lib/api.ts` +
`types/api.ts`; `lib/reports/preview.ts` (`buildPreviewReport`, `monthRangeLabel`); third tab in
`tools/ToolsClient.tsx` (plain label, no i18n key needed). `npm run build` compiles clean. Note:
`MonthlyReportView` (no `'use client'`) is rendered inside the client panel — fine, it's pure presentational.
Reachable at **/en/tools → Report Preview tab** (no new route/nav entry).
---

---
## Session: 2026-06-21 — Customer management (CRUD) + decouple customer from login user

**Date:** 2026-06-21
**Tags:** #session #backend #frontend #database

### Summary
To register robots you first need customers, and there was **no customer API/UI** (only entity + repo; seed SQL only). Built admin **customer CRUD**, backend + a Tools "Customers" tab. Key schema change: **decoupled `CustomerProfile` from `User`** — `findByUser_Id` was declared but never called anywhere, so the required user link was dead weight. In this MVP customers are report *recipients*, not login accounts, so `user_id` is now **nullable** (kept UNIQUE for a possible future customer login). Also added the **`contact_email`** column (needed for the email-delivery pivot; captured in the add-customer form). Both backend and frontend `npm run build` / `mvnw compile` pass.

### Files Modified
- `db/migration/V17__decouple_customer_from_user_add_email.sql` (new) — `ALTER user_id DROP NOT NULL` + `ADD contact_email VARCHAR(255)`
- [[CustomerProfile]] — `user` now optional (`@JoinColumn(name="user_id", unique=true)`, no `nullable=false`); added `contactEmail`
- `customer/dto/CustomerRequest.java`, `CustomerResponse.java` (new) — request (`@Email` on contactEmail) + response (incl. `robotCount`)
- `customer/service/CustomerService.java` (new) — list/get/create/update/delete; **delete blocked when `deploymentRepository.existsByCustomerProfileId`** (protects FK)
- `customer/controller/CustomerController.java` (new) — `/api/v1/customers` CRUD
- [[DeploymentRepository]] — added `existsByCustomerProfileId`, `countByCustomerProfileIdAndIsActiveTrue` (robotCount + delete guard)
- Frontend: `customerApi` + `CustomerResponse`/`CustomerRequest` types (`lib/api.ts`, `types/api.ts`); `components/CustomersPanel.tsx` (list/search/add/edit/delete, confirm on delete); third+fourth Tools tabs in `tools/ToolsClient.tsx` ("Customers", plain label)

### Decisions Made
- **Decouple customer from user (user_id nullable):** nothing reads the link, and MVP customers don't log in — so customers become plain admin-managed records. A future customer login can still link a user (column stays UNIQUE).
- **contact_email added now (V17), not a later V18:** the customer form needs it and the email-delivery pivot needs it — one migration covers both, retiring the earlier "V17 customer email" TODO.
- **Delete guard over cascade:** refuse to delete a customer with deployments (clear 400 message) rather than cascade-deleting robots/reports.
- **Customers under Tools tab (plain label):** consistent with Report Preview; avoids new route + nav i18n churn. Tools is becoming the internal admin hub.

### Unresolved / Next Steps
- [ ] Run V17 + restart backend so the schema change + `contact_email` apply before using the Customers tab
- [ ] Phase 2 reporting still pending (cadence-aware schedulers; per-customer vs per-robot send; switch delivery from `lineUserId` → `contactEmail`)
---

### Addendum (2026-06-26) — Email sending feature (report link by email)
Built the backend email sender (was never built before — delivery was still the old n8n client). Added
`spring-boot-starter-mail`; `spring.mail.*` config in [[application.properties]] (Gmail SMTP defaults; blank
creds → clear error). `ReportEmailService.send(serialNumber, month)`: resolves the robot's customer, requires
`contactEmail`, mints the public token via `ReportLinkService.createOrGetToken`, builds the URL
`{app.public.base-url}/{app.public.report-locale}/report/{token}` (defaults `http://localhost:3000` / `th`),
and sends an HTML email (link only — the report is the web page, no attachment) via `JavaMailSender`/
`MimeMessageHelper`. SMTP/auth/empty-email failures surface as `BadRequestException`. `ReportEmailController`
→ `POST /api/v1/reports/email?serialNumber=&month=` (authenticated). Frontend: `reportApi.sendEmail` +
**"Send report email"** button in the Report Preview robot view (with success/error status). `mvnw compile` +
`npm run build` clean. **To test:** set `MAIL_USERNAME`/`MAIL_PASSWORD` (Gmail app password) in
[[application-local.properties]], set a customer's contactEmail to a teammate, restart, then Send report email.
**Follow-up (same day):** email body rewritten to a **formal short message** (greeting + one line + View report
button + "contact your RAASPAL representative" + "Best regards, RAASPAL Team" + disclaimer); mentions a PDF copy
is downloadable from the report page. Added a **"Download PDF"** button on the public report page
(`PublicReportClient`) via `window.print()` (zero deps, perfect fidelity for Thai + SVG charts); page chrome
(switcher + button) is `print:hidden` and outer bg `print:bg-white` so the PDF shows only the report. New i18n
key `report.downloadPdf` (en/th/zh). Decision: email stays **link-only** (no file attachment); PDF is
self-serve from the page. Note: a robot SN can't be registered to two customers (register blocks duplicate SN),
so to email internal staff, set an existing customer's contactEmail to the staff address.

### Addendum (same day) — Public token report links (real data, the customer/email URL)
The public report page now serves **real aggregated data** via an unguessable token — the same
`/report/{token}` URL the monthly email will use. Backend: `report_links` table (V19; token unique +
unique (serial_number, report_month) so one stable link per robot/month), `ReportLink` entity/repo,
`ReportLinkService` (`createOrGetToken` mints/reuses a SecureRandom base64url token; `resolve` → token →
`ReportPreviewService.build`), `ReportLinkController` — `POST /api/v1/reports/links` (auth, mint) +
`GET /api/v1/reports/public/{token}` (**permitAll** in [[SecurityConfig]]). Frontend: `reportApi.createLink`
+ `publicReport`; the public page is now `PublicReportClient` (client) that fetches `publicReport(token)` and
renders only the report + `LanguageSwitcher` (close-only, no in-app nav); token `example` still shows the
sample. `ReportPreviewPanel` gained an **"Open shareable link"** button (robot view) → mints token → opens
`/{locale}/report/{token}` in a new tab. `mvnw compile` + `npm run build` clean. **Prereq:** restart backend
(V19 + new endpoints + security whitelist); the robot needs synced tasks for the chosen month or the link
shows zeros.

### Addendum (same day) — Public translatable example report page
The report page is now (1) shareable without login and (2) translatable via a switcher. Public route
`/[locale]/report/[token]` (already whitelisted in [[proxy.ts]]) renders the **sample** report — so
`/<locale>/report/example` is a no-login example link to show customers; it now has a `LanguageSwitcher` in a
top bar and calls `setRequestLocale`. `MonthlyReportView` labels were moved to **next-intl** ('report'
namespace added to en/th/zh): title, Customer/Site/Robot/SN, Part titles, all Part-1 row labels, Part-2 detail
labels, recommendations heading, disclaimer; well-known ring labels (Task Completion/Cleaning Coverage Rate)
and consumable names (Brush/Filter/Squeegee) translate via a key map with raw-value fallback. **Data values**
(names, counts, task-type/status breakdowns, recommendations text) stay backend-English — full data
translation would need backend i18n. `ReportPreviewPanel` got an "Open public example page (no login)" link
(`/{locale}/report/example`). `useTranslations` used in the view (isomorphic — works in the server report
page and the client preview panel). `npm run build` clean. (Customer-questions block removal completed
earlier this session: frontend display + backend `customerQuestions` field both gone.)

### Addendum (same day) — On-demand Gausium sync trigger ("Sync from Gausium" button)
The telemetry sync pipeline (`TelemetrySyncService` → `GausiumAdapter` → `GausiumApiClient` V2 List Robot Task
Reports) had no trigger — only a scheduler. Added **`POST /api/v1/telemetry/sync/{serialNumber}?from=&to=`**
(`TelemetryController`) + `TelemetrySyncService.syncBySerialNumber()` returning a `SyncResult(serialNumber,
saved, skipped)`; `syncRobotUnit` now returns the count. Brand-API failures (creds not configured / auth /
robot not bound) surface as `BadRequestException` with the message. Frontend: `telemetryApi.sync()`;
`ReportPreviewPanel` gained a **"Sync from Gausium"** button (robot selection only) that pulls the selected
month's range ([month-01 .. month-lastday]) then invalidates the `report-preview` query to re-aggregate. Shows
success/error inline. `mvnw compile` + `npm run build` clean. **Real-data prereqs:** set
`GAUSIUM_API_*` creds in [[application-local.properties]] + robot SN bound to that Gausium account + restart
backend; then Sync → preview shows live numbers. (Seed `docs/seed-task-reports-for-robot.sql` remains the
offline fallback.)

### Addendum (same day) — Live report preview endpoint (real telemetry aggregation)
Report preview now shows **real aggregated data** for a registered robot, not sample metrics. New backend
`GET /api/v1/reports/preview?serialNumber=&month=YYYY-MM` → `ReportPreviewService` aggregates that robot's
`robot_task_reports` for the month into the `MonthlyPerformanceReport` shape: totals (tasks, operating time,
area, water), avg productivity (area ÷ hours), **battery = Σmax(0,start−end) ÷ area ×100 %/100 sqm**, rings
(avg completion %, coverage = Σactual÷Σplanned ×100), task type (most common cleaningMode), task status
(most common taskEndStatus mapped), tasks/day (tasks ÷ distinct active days), avg run time, consumables from
the latest task's residual % (good ≥80 / monitor ≥50 / action), rule-based recommendations. Empty month →
zeros + explanatory recommendation. DTO `report/dto/ReportPreviewResponse.java`, controller
`ReportPreviewController.java`. Frontend: `reportApi.preview()`; `ReportPreviewPanel` now fetches live data
via TanStack Query for real robots (sample button still static). `mvnw compile` + `npm run build` clean.
**Data dependency:** a robot needs synced task reports to show numbers — `docs/seed-task-reports-for-robot.sql`
seeds 3 June tasks for a robot by SN (default `GS208-0410-L3P-X100`) until Gausium sync is live.

### Addendum (same day) — Report redesigned to manager's "Executive Robot Performance Report" v2
Manager replaced the 5-part layout with a new one-pager (mockup + `docs/Executive Robot Performance
Report_template.pptx`). **Rewrote the report model + view**: header (Customer/Site + Robot Name/SN boxes,
title "RAASPAL Executive Robot Performance Report" + "June 2026") with the red **"Customer needs to know:"**
Thai questions (ทำงานคุ้ม/ช่วยคนทำงานได้/ทำงานเร็ว/มีปัญหา) top-right; **Part 1 Executive Summary** (Total Tasks,
Operating time string, Area sqm, Productivity sqm/h, Water L, Battery start→end); **Part 2 Operational
Performance** = two **SVG ring/donut gauges** (Task Completion Rate, Cleaning Coverage Rate) + task details
(Type/Status/Tasks per Day/Avg Run Time); **Part 3 Consumables Status** (Brush/Filter/Squeegee % with
traffic-light dots); **Recommendations**. New types in `lib/reports/types.ts` (`RingMetric`,
`ConsumableStatus`, `ExecutiveSummary`, `OperationalPerformance`, `DEFAULT_CUSTOMER_QUESTIONS`); sample in
`gausium.ts` matches the mockup numbers; `MonthlyReportView.tsx` rebuilt (blue `#4472c4` rings / `#bcccea`
panels / red `#e0202a` questions, no chart lib); `preview.ts` → `monthYearLabel` ("June 2026") +
`buildPreviewReport` now takes `serialNumber` separately; `ReportPreviewPanel` updated. `npm run build` clean.
Old `ReportKpi`/`PerformanceBar`/`HealthItem`/`Recommendation` types removed.

### Addendum (same day) — Robot register UI (Tools → Robots tab)
Built `components/RobotsPanel.tsx`: register a robot (SN/brand[GAUSIUM/KEENON/CENOBOT select]/model/name + **customer dropdown** via `customerApi.list()` + site + cadence) → `robotUnitApi.register`; list with search; **inline cadence change** (`updateCadence`) and **deactivate** (`deactivate`) per deployment; robots with no active deployment show "Inactive". Added "Robots" tab to `tools/ToolsClient.tsx`. Completes the **add-customer → register-robot → Report Preview** loop, all from the UI (no SQL). `npm run build` compiles clean. (Note: ToolsClient line ~28 has a `bg-  [var(--app-bg)]` double-space typo left intentionally per user — not touched.)
---

---
## Session: 2026-07-01 — Automated monthly report delivery (customer bundle + scheduler + history)

**Date:** 2026-07-01
**Tags:** #session #backend #frontend #ai #database #config

### Summary
Replaced the dead n8n/LINE/xlsx report pipeline with a real automated email delivery system built on the existing web report. Each customer now receives **one email per month** with **one link** to a combined page showing **all their robots'** reports (one report per robot). The backend [[ReportDeliveryScheduler]] fires on a cron (2nd of each month, previous month) and, via [[ReportDeliveryService]], syncs telemetry for the month, sends each eligible (MONTHLY-cadence) customer their bundle, and records every attempt in the new `report_sends` table for idempotency + history. Delivery is disabled by default (`app.reports.email-scheduler-enabled=false`) and gated with `@ConditionalOnProperty` so the bean only registers when enabled. A new "Manage report automation" tab ([[ReportAutomationPanel]]) lets the team pick a month, run delivery now (idempotent), view history (SENT/FAILED/SKIPPED per customer), and resend failures. Also formatted report numbers with thousands separators, moved the report footer to a centered "RAAS PAL CO., LTD", restructured the report header, and added Load-more pagination to the Customers list.

### Files Modified
- [[CustomerReportLink]] (`report/entity/CustomerReportLink.java`) — NEW; one token per customer+month
- [[CustomerReportLinkRepository]] (`report/repository/CustomerReportLinkRepository.java`) — NEW
- [[CustomerReportBundleResponse]] (`report/dto/CustomerReportBundleResponse.java`) — NEW; {customerName, periodLabel, robots[]}
- [[CustomerReportBundleService]] (`report/service/CustomerReportBundleService.java`) — NEW; aggregates all a customer's robot reports
- [[CustomerReportLinkService]] (`report/service/CustomerReportLinkService.java`) — NEW; idempotent token mint + resolve
- [[CustomerReportBundleController]] (`report/controller/CustomerReportBundleController.java`) — NEW; POST /links/customer, GET /public/customer/{token}
- [[ReportSend]] (`report/entity/ReportSend.java`) — NEW; delivery record (SENT/FAILED/SKIPPED)
- [[ReportSendRepository]] (`report/repository/ReportSendRepository.java`) — NEW
- [[ReportSendResponse]] (`report/dto/ReportSendResponse.java`) — NEW; history row + customerName
- [[ReportDeliveryService]] (`report/service/ReportDeliveryService.java`) — NEW; shared send/record/idempotency logic
- [[ReportDeliveryScheduler]] (`report/scheduler/ReportDeliveryScheduler.java`) — NEW; cron trigger, `@ConditionalOnProperty`
- [[ReportDeliveryController]] (`report/controller/ReportDeliveryController.java`) — NEW; /delivery/run, /delivery/send, /delivery/history
- [[ReportEmailService]] (`report/service/ReportEmailService.java`) — added `sendBundle(customerProfileId, month)`
- [[application.properties]] — added scheduler flag + cron; removed n8n/Supabase block
- `db/migration/V20__add_customer_report_links.sql`, `V21__add_report_sends.sql` — NEW migrations
- `src/test/resources/application.properties` — added `spring.mail.host`/mail props so full context loads in tests
- Deleted: MonthlyReportScheduler, WeeklyReportScheduler, ReportController, MonthlyReportService/Summary, N8n*, Supabase*, ReportGenerator*, GausiumReportGenerator + their 3 orphaned tests
- [[CustomerBundleClient]] (`app/[locale]/report/customer/[token]/`) — NEW public combined-report page (stacked robots, print page-breaks, Download PDF)
- [[ReportAutomationPanel]] (`components/ReportAutomationPanel.tsx`) — NEW automation tab (run/history/resend)
- [[ToolsClient]] — reports tab now renders [[ReportAutomationPanel]]
- [[MonthlyReportView]] — thousands-separator `formatNumber`; header restructured; footer → centered "RAAS PAL CO., LTD"
- [[CustomersPanel]] — Load-more pagination (10/page, resets on search)
- `lib/api.ts` — added publicBundle, createCustomerLink, runDelivery, sendCustomerBundle, deliveryHistory; removed runMonthly/runWeekly
- `types/api.ts` — added ReportSend/ReportSendStatus; removed old automation types

### Decisions Made
- **One bundle link per customer+month (not per robot):** manager wants a single email covering all robots; per-robot report_links stay for the admin "Open shareable link" preview.
- **Idempotency via report_sends:** a customer marked SENT for a month is skipped, so re-runs / double instances never double-send.
- **Scheduler disabled by default + `@ConditionalOnProperty`:** safe to deploy before SMTP is configured; enable with `REPORT_EMAIL_SCHEDULER_ENABLED=true`.
- **Cron `0 0 8 2 * *` (2nd of month, 08:00 UTC, previous month):** wait a day past month-end so all Gausium telemetry is synced first.
- **Shared [[ReportDeliveryService]]:** scheduler and manual admin endpoint use identical send/record/idempotency logic.

### Unresolved / Next Steps
- [ ] Set `REPORT_EMAIL_SCHEDULER_ENABLED=true` + `MAIL_USERNAME`/`MAIL_PASSWORD` (Gmail App Password) + `PUBLIC_BASE_URL` on Render when going live
- [ ] Load-testing / batching if customer count grows large (200+); current send loop is sequential
- [ ] Optionally add a "Send to customer (all robots)" button in [[ReportPreviewPanel]] using the bundle flow
---

---
## Session: 2026-07-22 — Frontend UI/UX "Operations Console" pass

**Date:** 2026-07-22
**Tags:** #session #frontend

### Summary
Reworked the frontend from a "product showcase" feel into an operations console for team handoff, guided by the local `.claude/skills` design skills (ui-ux-pro-max recommendation for admin/ops products: data-dense minimalism, clarity over decoration). Kept the RAASPAL teal brand, light/dark themes, and all i18n keys (EN/TH). Biggest functional fix: the sidebar previously only rendered at ≥1680px, so on normal work laptops the main navigation was hidden behind a hamburger menu — it now shows from 1024px (lg). The Team Dashboard was rebuilt around live operational data (KPI tiles + recent solutions feed) with the animated hero shrunk to a slim CTA band. `npm run build` passes clean.

### Files Modified
- [[AppSidebar]] (`components/AppSidebar.tsx`) — visible from `lg:` (was `min-[1680px]:`), width w-64/xl:w-72, collapsed rail w-20
- [[TopNavigationMenu]] (`components/TopNavigationMenu.tsx`) — hamburger now `lg:hidden` to match
- [[AppTopBar]] (`components/AppTopBar.tsx`) — search visible from `md:`; removed decorative (non-functional) notification bell
- [[status-badge]] (`components/ui/status-badge.tsx`) — NEW shared StatusBadge (dot + label, WCAG-safe light/dark pairs) + `toneForStatus()` mapping all backend statuses (COMPLETED/PENDING/FAILED/SENT/VERIFIED/UNDER_TESTING/REJECTED/ONLINE/OFFLINE…)
- [[skeleton]] (`components/ui/skeleton.tsx`) — NEW Skeleton + ListSkeleton (replaces Loader2 spinners on list pages)
- [[empty-state]] (`components/ui/empty-state.tsx`) — NEW EmptyState primitive (icon, title, hint, action)
- [[DashboardStats]] (`components/DashboardStats.tsx`) — NEW KPI tile row: solutions/proposals/robots counts via `totalElements` of paged APIs + CVTE online count; 60s refetch; per-tile skeleton, "—" on error
- [[RecentSolutions]] (`components/RecentSolutions.tsx`) — NEW latest-5 recommendations feed linking into each recommendation
- [[page.tsx]] (`app/[locale]/page.tsx`) — dashboard rebuilt: slim aurora CTA band, DashboardStats, RecentSolutions + CvteStatusSummary two-column, quick links, compact how-it-works; max-w-6xl
- [[FlowStepper]] (`components/FlowStepper.tsx`) — NEW 3-step pipeline indicator (Upload → Recommendations → Proposal)
- [[UploadClient]] / [[RecommendationClient]] / [[ProposalClient]] — FlowStepper inserted below the back link
- [[SolutionsClient]] / [[ProposalsClient]] — shared StatusBadge, ListSkeleton loading, denser cards (p-4), max-w-6xl container
- [[RobotsClient]] (`app/[locale]/robots/RobotsClient.tsx`) — shared StatusBadge for testStatus, ListSkeleton loading
- [[ToolsClient]] (`app/[locale]/tools/ToolsClient.tsx`) — tab bar scrolls horizontally on narrow screens; active tab synced to `?tab=` (shareable); exported `ToolTab`/`isToolTab`
- [[tools/page.tsx]] (`app/[locale]/tools/page.tsx`) — reads `?tab=` searchParam and passes initialTab (route now dynamic)
- [[en.json]] / [[th.json]] (`messages/`) — new keys: `teamDashboard.kpi.*`, `teamDashboard.recent.*`, `generateSolution.stepper.*`

### Decisions Made
- **Keep brand, change posture:** RAASPAL teal + light/dark retained; gradients/glow reduced to the primary CTA. Style follows the ui-ux-pro-max "admin/analytics" guidance (density + status clarity over decoration).
- **Sidebar at lg (1024px):** matches Material adaptive-navigation guidance; hamburger only below lg.
- **One StatusBadge for the whole app:** dot + label so state is never colour-only; ad-hoc per-file badges removed (Robots TypeBadge kept — it is a category, not a status).
- **Old `teamDashboard.stats/queue/shortlist/scoring/activity` i18n keys left in place** — unused but harmless; can be pruned later.

### Unresolved / Next Steps
- [ ] Thai translations for Add/Edit robot form fields (pre-existing, low priority)
- [ ] Consider pruning unused legacy `teamDashboard.*` i18n keys (stats/queue/shortlist/scoring/activity)
- [ ] Optional: virtualize Solutions/Proposals lists if history grows past a few hundred rows
---

---
## Session: 2026-07-23 — Report automation promoted to first-class + dashboard reframe

**Date:** 2026-07-23
**Tags:** #session #frontend

### Summary
Repositioned the frontend around its primary job — monthly report automation — after the team confirmed that is now the main operational use (register robots to customers, then deliver monthly bundle emails). Report delivery was previously buried three tabs deep under Tools; it is now a top-level **Reports** sidebar item (2nd, right after Team Dashboard) with its own `/reports` hub, and the Team Dashboard leads with live delivery status instead of the AI-solution flow. `npm run build` passes clean (new `/[locale]/reports` route present, 39 pages).

### Files Modified
- [[AppSidebar]] (`components/AppSidebar.tsx`) — new **Reports** nav item (`/reports`, CalendarClock icon) inserted after Team Dashboard
- `app/[locale]/reports/page.tsx` + `ReportsClient.tsx` — NEW first-class Reports hub: tabs **Manage automation / Email customers / Report preview**, URL-synced `?tab=`, reusing [[ReportAutomationPanel]] / [[CustomerEmailPanel]] / [[ReportPreviewPanel]]
- [[ToolsClient]] (`app/[locale]/tools/ToolsClient.tsx`) + `tools/page.tsx` — trimmed to **Monitor / Customers / Robots** (report/email/preview tabs moved to /reports); VALID_TABS updated
- `lib/report-month.ts` — NEW `previousMonth()` + `monthLabel()` helpers (report month = previous calendar month, matching the 2nd-of-month scheduler)
- `components/ReportDeliveryStats.tsx` — NEW report KPI tiles: Sent / Needs attention / Skipped (this report month, from `reportApi.deliveryHistory`) + Customers (`customerApi.list`)
- `components/MonthlyDeliveryCard.tsx` — NEW dashboard centerpiece: report period, running/idle state (`reportApi.deliveryStatus`, polls 3s while running), coverage bar (customers sent vs total), last-run summary, CTA → /reports
- `components/RecentDeliveries.tsx` — NEW latest-6 `report_sends` feed for the report month with shared StatusBadge; replaces RecentSolutions on the dashboard
- [[page.tsx]] (`app/[locale]/page.tsx`) — dashboard rebuilt around reports: report hero CTA → /reports, ReportDeliveryStats, MonthlyDeliveryCard + RecentDeliveries two-col, CvteStatusSummary, report-first quick links (Email/Preview/Robots/Customers), "How automated delivery works" strip, and a demoted dashed shortcut to Generate Solution
- [[en.json]] / [[th.json]] — new keys: `teamDashboard.reportKpi/delivery/recentDeliveries/reportFlow/solutionShortcut.*`, reworked `teamDashboard.hero.*` + `quickAccess.*`, new top-level `reports.*` (page + tabs); `tools.searchPlaceholder` reworded

### Decisions Made
- **Reports is a top-level destination, not a Tools tab:** report delivery is the app's main use, so it gets its own sidebar entry and `/reports` hub. Tools is now the catalog/admin area (Monitor/Customers/Robots).
- **Dashboard leads with delivery status:** KPI tiles + MonthlyDeliveryCard reflect the current report month (previous calendar month). AI solution flow demoted to a single dashed shortcut — still available, no longer the headline.
- **DashboardStats / RecentSolutions (from the 2026-07-22 pass) kept but no longer on the dashboard** — components still compile/usable; their i18n keys (`teamDashboard.kpi/recent`) left in place.
- **Report tabs still reachable at /reports?tab=email|preview** — quick links deep-link into them.

### Unresolved / Next Steps
- [ ] Optionally prune now-unused [[DashboardStats]] / [[RecentSolutions]] components + their `teamDashboard.kpi/recent` i18n keys
- [ ] Consider a customer-cadence breakdown (MONTHLY/WEEKLY/OFF) tile once WEEKLY sends are wired (Phase 2)
- [ ] Thai review of the new report dashboard strings by a native speaker
---

---
## Session: 2026-07-23 — AutoXing on-demand delivery report (preview only)

**Date:** 2026-07-23
**Tags:** #session #backend #frontend #ai

### Summary
Added a new robot brand — **AutoXing** (delivery robots) — as an on-demand delivery report the team can pull for an urgent report. Deliberately scoped to **preview only**: no persistence, no [[TelemetryAdapter]] sync into `robot_task_reports`, no scheduler/email wiring. AutoXing data is fundamentally different from the existing cleaning pipeline (delivery/call/charging **daily aggregates** from `/statis/v2.0/task`, not per-task cleaning rows), so it gets its own delivery-shaped report DTO + a live pull straight from the AutoXing API, rather than being force-fit into the cleaning-centric [[ReportPreviewService]]. Region = **global** (`apiglobal.autoxing.com`, env-overridable). Backend compiles (`mvnw compile`), frontend `npm run build` clean.

### Files Modified
- `application.properties` — new `app.autoxing.api.base-url/app-id/app-secret/app-code` block (`AUTOXING_API_*`, default base-url = global)
- `telemetry/adapters/autoxing/AutoxingApiException.java` — NEW, with `authFailure` flag for the documented 400 "Authentication failed"
- `telemetry/adapters/autoxing/dto/AutoxingToken.java` — NEW (token + expiry)
- `telemetry/adapters/autoxing/AutoxingApiClient.java` — NEW; RestClient; `fetchToken()` signs `MD5(appId+timestamp+appSecret)` with AppCode in `Authorization` header; `getRobotState` (`/robot/v2.0/{id}/state`), `getTaskStatistics` (`POST /statis/v2.0/task`), `getTaskDetail` (`/task/v3/{id}?needDetail`); unwraps the `{status,message,data}` envelope, flags 400 as auth failure
- `telemetry/adapters/autoxing/AutoxingAuthService.java` — NEW; **in-memory** token cache (NOT the DB-backed Gausium pattern — AutoXing tokens live ~600s and are non-refreshable, so "refresh per run", re-sign near expiry / on auth failure)
- `telemetry/adapters/autoxing/dto/AutoxingDeliveryReport.java` — NEW delivery-shaped report DTO (liveStatus, summary, per-category stats, daily counts, note)
- `telemetry/adapters/autoxing/AutoxingReportService.java` — NEW; validates ≤30-day window, aggregates category daily-stats (defensive JsonNode field parsing — assumes duration ms / mileage meters per docs, noted in the report), best-effort live state, `withReauth()` retries once on auth failure
- `telemetry/controller/AutoxingReportController.java` — NEW `GET /api/v1/autoxing/report/preview?robotId=&from=&to=` (dates optional → last 30 days); authenticated via existing `anyRequest().authenticated()`
- Frontend: `types/api.ts` (`AutoxingDeliveryReport` types), `lib/api.ts` (`autoxingApi.preview`), `components/AutoxingReportPanel.tsx` (NEW — robotId + date range → live report: status card, task/mileage/duration tiles, category table, print-to-PDF), `app/[locale]/reports/ReportsClient.tsx` + `reports/page.tsx` (new **AutoXing report** tab, URL `?tab=autoxing`), `messages/en.json` + `th.json` (`reports.tabs.autoxing`)

### Decisions Made
- **Preview-only scope (per user):** urgent report now; no DB/adapter/scheduler. Report is a live API pull, always current.
- **New delivery report, not the cleaning pipeline:** AutoXing is delivery; its aggregates don't map to sqm/water/brush or per-task `RobotTaskReport`. Own DTO + endpoint keeps semantics correct and avoids touching the cleaning report.
- **In-memory token, not DB:** AutoXing's 600s non-refreshable token makes the Gausium DB-token table the wrong fit; re-sign per run.
- **Field-unit assumptions flagged in the report `note`:** the `/statis/v2.0/task` per-item schema wasn't fully documented; parsing is defensive (count/taskCount, mileage/taskMileage, duration/taskDuration) and durations assumed ms / mileage meters — to be verified against a live response.

### Unresolved / Next Steps
- [ ] User to set `AUTOXING_API_APP_ID` / `AUTOXING_API_APP_SECRET` / `AUTOXING_API_APP_CODE` (+ optional `AUTOXING_API_BASE_URL`) in [[application-local.properties]] and provide the robotId to test
- [ ] Verify the `/statis/v2.0/task` per-item field names + duration/mileage units against a real response; adjust `AutoxingReportService` mapping if needed
- [ ] Success/cancel breakdown not available from statistics endpoint — could enumerate task list + `/task/v3/{id}` per-task detail later if the team needs it
- [ ] If wanted later: promote AutoXing to a full `TelemetryAdapter` + register robots so it joins the automated monthly customer bundle
---

---
## Session: 2026-07-23 — AutoXing auth fix (APPCODE scheme) + live schema confirmed

**Date:** 2026-07-23
**Tags:** #session #backend

### Summary
Resolved the AutoXing `401 "Invalid API key in request"`. Diagnosed by calling the token endpoint directly (PowerShell) across region × header combinations: **only `apiglobal.autoxing.com` (global) + `Authorization: APPCODE <code>` works** — the gateway is Alibaba Cloud API Gateway, which requires the `APPCODE ` prefix; raw AppCode and the China host both 401. Made APPCODE the default. Then ran the full pipeline live (token → `/statis/v2.0/task` → `/robot/v2.0/{id}/state`) and **confirmed the report populates**: robot 2382310202332BC returned 838 delivery + 23 charging = 861 tasks for Jul 1–20, battery 32%, error objects present. The statistics per-item schema (`count/date/duration/mileage/robot`) matches the existing defensive parser exactly; `duration` is **ms**, `mileage` is **meters** — both assumptions confirmed. No mapping change needed.

### Files Modified
- `AutoxingApiClient.java` — AppCode auth-header scheme now **defaults true** (`app.autoxing.api.appcode-scheme`); sends `Authorization: APPCODE <code>`
- `application.properties` — `app.autoxing.api.appcode-scheme=${AUTOXING_API_APPCODE_SCHEME:true}` (was false)
- `AutoxingAuthService.java` — cap token cache at 300s (`MAX_TTL_SECONDS`) since live `expireTime` is an ambiguous unit (600 in docs vs 603800 live); stale-token 401 still self-heals via re-auth retry
- `AutoxingReportService.java` — extract `message` from AutoXing error objects (`{code,level,message,type}`) instead of raw JSON; softened the report note (units now confirmed, not assumed)
- `application-local.properties` — base-url corrected China→global (`apiglobal.autoxing.com`)

### Decisions Made
- **APPCODE scheme is the default:** confirmed the only working format for the live global endpoint; keeps a `false` escape hatch for other gateways.
- **Cap token cache at 300s:** robust to the `expireTime` unit ambiguity; aligns with "refresh per run".

### Unresolved / Next Steps
- [ ] Render: deploy the updated backend (git push → rebuild) so the APPCODE default + error-message parsing go live; ensure `AUTOXING_API_BASE_URL` is unset or the **global** URL (NOT china); creds already set; no flag env var needed (default true)
- [ ] Rotate the credentials that appeared in plaintext screenshots (AutoXing secret/code, Gausium keys, Gmail app password) if those images could be seen by others
---

---
## Session: 2026-07-23 — AutoXing malformed Content-Type fix

**Date:** 2026-07-23
**Tags:** #session #backend

### Summary
After the APPCODE auth fix, the token call reached AutoXing but Spring's RestClient threw
`Invalid mime type "json;charset=UTF-8": does not contain '/'` — AutoXing returns a malformed
`Content-Type` header (missing `application/`), which Spring's message converters reject (raw PowerShell
didn't care, which is why the direct test worked). Rewrote [[AutoxingApiClient]] to read every response via
`RestClient.exchange()` — raw body bytes parsed with the injected `ObjectMapper` — bypassing Content-Type-based
conversion entirely. HTTP 4xx and non-200 envelope status still raise `AutoxingApiException` (auth cases flagged).
Compiles clean.

### Files Modified
- `AutoxingApiClient.java` — inject `ObjectMapper`; new `exchangeForData()` helper uses `.exchange()` + `objectMapper.readTree(bytes)` for token/state/statistics/detail; removed `.retrieve().body(JsonNode.class)` and the `RestClientResponseException` handling

### Decisions Made
- **Read raw + parse manually:** the only robust way to consume AutoXing's malformed `Content-Type`; keeps the envelope/auth-failure handling intact.

### Unresolved / Next Steps
- [ ] Restart local backend and retry the AutoXing tab (expect the report to populate); then deploy to Render
---

---
## Session: 2026-07-23 — AutoXing delivery report in-app (DeliveryReportView)

**Date:** 2026-07-23
**Tags:** #session #backend #frontend

### Summary
Turned the AutoXing delivery-report mockup into a real in-app report. The AutoXing tab now renders a full RAAS PAL "Executive Robot Performance Report" for delivery robots, with **customer and site resolved automatically from AutoXing's own directories** (no RAASPAL registration needed) and optional robot-name/model overrides. Confirmed live against robot 2382310202332BC → Customer **KUBOTA**, Site **KUBOTA Precision Machinery · Floor 1**, 861 tasks / 838 deliveries / 188.5 km over 1–20 Jul. Backend `mvnw compile` and frontend `npm run build` both clean.

**Key product decision:** the robot's **live status is shown in the operator panel only, never inside the printable report** — a period report (June, or 1–20 July) must describe only its window, so a "battery is 32% right now" block was temporally inconsistent.

### Files Modified
- `AutoxingApiClient.java` — new `getRobotSummary` (`/robot/v1.1/list`), `getBusinessList`, `getBuildingList`, `getAreaList`; static `entries()` helper because **AutoXing is inconsistent: business/building use `data.lists` (plural), robot/area use `data.list` (singular)**
- `AutoxingReportService.java` — `resolveContext()` maps businessId→customer name, buildingId→site name, areaId→area/floor, robot model/name; 10-minute in-memory cache for the account-wide business/building directories; all lookups best-effort (degrade to "—"); aggregation now uses AutoXing's own `allStatis` per-day rollup for totals + the daily table, with category sums as fallback; richer `Summary` (delivery share, active days, tasks/active day, avg task time, busiest day)
- `dto/AutoxingDeliveryReport.java` — added robotName/model/customerName/siteBranch, `DailyStat` (date/count/mileage/duration), expanded `Summary`, `LiveStatus.isOnline`
- `AutoxingReportController.java` — optional `robotName` / `model` query params
- `components/report/DeliveryReportView.tsx` — NEW paper report: Part 1 Delivery Summary, Part 2 Operational Performance (delivery-share + active-days rings, daily volume chart, breakdown), Part 3 Daily Activity table, Recommendations derived from the period. Reuses [[MonthlyReportView]]'s PowerPoint palette (`#bcccea` / `#4472c4`); white in both themes so it prints consistently
- `components/AutoxingReportPanel.tsx` — robot name + model inputs, Print/PDF, and a separate **live status card** (online, battery, charging/e-stop/manual, faults) clearly labelled "not included in the report"
- `types/api.ts`, `lib/api.ts` — matching types + `robotName`/`model` params

### Decisions Made
- **Auto-resolve customer/site from AutoXing** rather than requiring RAASPAL registration — the deployment data already lives in AutoXing and can't drift out of sync.
- **Live status out of the report, into the panel** — period reports describe a window; live state is operator context.
- **Name/model overrides** — AutoXing has no real model (only category `餐厅` = "restaurant") and a blank `name`, so the team supplies them; falls back to "AutoXing <model>".

### Unresolved / Next Steps
- [ ] Optionally persist AutoXing robots as [[RobotUnit]]s so they join the automated monthly customer bundle (currently on-demand only)
- [ ] Print stylesheet tuning if the team wants an exact one-page PDF
- [ ] Consider a public shareable link for delivery reports (like [[ReportLink]] for cleaning)
---

## Session: 2026-07-25 — Partner middleware API (PCS): keys, scoped reads, admin UI

**Date:** 2026-07-25
**Tags:** #session #backend #frontend #config

### Summary
Completed the **partner middleware API** — a read-only, API-key-authenticated surface (`/api/partner/v1`) that lets a distributor/service partner such as **PCS** pull data for only the robots it services. The `partner.*` package already had entities/repos/`PartnerApiKeyService`/`PartnerPrincipal` and `V22` (from an earlier unlogged session); this session built everything around them: the admin key-management API, the auth middleware and its own security chain, the scoped read endpoints, and the full admin UI.

**Scoping model:** the partner link lives on **`deployments.partner_id`** — not on the customer and not on the robot. One partner can therefore service robots across many customers, and a robot's serial number reaches PCS automatically through its deployment. Every partner read derives the partner id from the authenticated key, **never** from a request parameter.

**Data source:** partner reads hit **our own `robot_task_reports`**, never a live Gausium call — so brand credentials are never exposed and reads are fast, paged, and brand-agnostic. Freshness therefore depends on the telemetry sync (see the pipeline session below).

A deploy to Render failed first time with *"a filter chain that matches any request has already been configured … will never get invoked"*: `@Order(1)` had been placed on the `PartnerSecurityConfig` **class**, which is ignored for `SecurityFilterChain` ordering. Moving it to the **`@Bean` method** fixed it. Root cause of it reaching production: only `mvnw compile` had been run, which never boots the Spring context — the existing `contextLoads` test would have caught it.

### Files Modified
- [[PartnerAdminController]] (`partner/controller`) — NEW admin API `/api/v1/partners`: create/list partners, rename + enable/disable (disable = instant kill-switch for all its keys), mint key (plaintext returned **once**), list key metadata, revoke, assign a deployment, and **bulk-assign** many deployments
- [[PartnerService]] (`partner/service`) — NEW partner CRUD + `assignDeployment` / `assignDeployments` (bulk, one transaction)
- [[PartnerApiKeyService]] (`partner/service`) — added `listKeys`, plus `authenticate()` + `AuthenticatedPartner` enforcing key-active **and** partner-active
- [[ApiKeyAuthFilter]] (`partner/security`) — NEW `X-API-Key` filter → `PartnerPrincipal`; never throws, fails **closed**, best-effort `last_used_at`
- [[PartnerSecurityConfig]] (`partner/security`) — NEW `@Order(1)` chain matched to `/api/partner/**`: stateless, CSRF off, no CORS, no form/basic login
- [[PartnerAuthEntryPoint]] (`partner/security`) — NEW JSON 401 naming `X-API-Key` (never mentions JWT)
- [[SecurityConfig]] (`auth/security`) — JWT chain pinned `@Order(2)` so the catch-all publishes last
- [[PartnerApiController]] (`partner/controller`) — NEW `/me`, `/robots`, `/robots/{serialNumber}/task-reports` (paged; `month`, or `from`/`to` day-range)
- [[PartnerDataService]] (`partner/service`) — NEW scoped reads; identical 404 for unknown vs not-owned serials; page size capped at 100; Asia/Bangkok day windows
- `partner/dto/*` — NEW `CreatePartnerRequest`, `UpdatePartnerRequest`, `PartnerResponse`, `CreateApiKeyRequest`, `CreatedApiKeyResponse`, `ApiKeyResponse`, `AssignPartnerRequest`, `BulkAssignRequest`, `PartnerRobotResponse`, `PartnerTaskReportResponse`
- [[RobotTaskReportRepository]] — paged/ordered queries + `findByRobotUnitIdAndStartTimeBetween...` for the day filter
- [[RobotUnitResponse]] — `DeploymentInfo` now carries `partnerId` so the UI can show current assignments
- [[RobotUnitService]] / [[DeploymentRepository]] — `listAll()` N+1 removed via `findActiveWithRobotAndCustomer()` fetch-join (~2N+1 queries → 2; was the cause of a ~10s empty panel)
- [[PartnersPanel]] (`components`) — NEW Tools → **Partners** tab: create/enable/disable, mint (copy-once) + revoke keys, and a **persistent searchable multi-select** box to bulk-assign robots (customer shown in a fixed right-hand column so names align vertically)
- [[ToolsClient]] / `tools/page.tsx`, [[lib/api.ts]], `types/api.ts`, `messages/en.json` + `th.json` — tab wiring, `partnerApi`, types, EN/TH copy
- [[PartnerApiSecurityTest]] / [[PartnerDataScopingTest]] (`src/test`) — NEW 19 tests (see next session entry)

### Decisions Made
- **Partner link on the deployment, not the customer/robot** — a partner services a robot *at a site*; this is the only scoping column and keeps the partner module decoupled (plain `UUID`, no JPA relation).
- **`X-API-Key`, not `Authorization: Bearer`** — keeps the partner scheme visibly separate from staff JWTs.
- **Two separate `SecurityFilterChain`s** — a partner key can never reach a staff endpoint and a staff JWT can never reach a partner endpoint. Specific matcher `@Order(1)`, catch-all `@Order(2)`, **ordered on the `@Bean` method**.
- **Identical 404 for unknown vs not-owned serial** — prevents a partner probing which robots exist outside its scope.
- **Admin endpoints left at "any authenticated staff"** rather than `ADMIN`-only — the app currently has no role gates anywhere and there is no seed user, so gating these could have locked the team out. Noted as a one-line tightening for later.
- **Assignment UI inside the Partners tab** (chosen over an inline dropdown on each Robots row) — keeps everything partner-related in one place.
- **Partner reads never proxy Gausium** — read from our DB, so credentials stay internal and the API works for any brand in `robot_task_reports`.

### Unresolved / Next Steps
- [ ] Redeploy backend to Render with the `@Order` fix and confirm `V22` ran on Supabase
- [ ] Deploy frontend to Vercel (new Partners tab)
- [ ] Onboard PCS: create partner → mint key → bulk-assign its ~70 deployments → hand over the key securely
- [ ] Write the **PCS-facing API documentation** (user asked for this once the flow is confirmed working); add an `X-API-Key` security scheme + partner-only group to Swagger
- [ ] Optional hardening: per-key rate limiting, key expiry/rotation, richer access audit, `ADMIN`-only admin endpoints
- [ ] Consider an aggregated `GET /robots/{sn}/summary?month=` (totals rather than raw task rows) if PCS prefers a rollup
---

## Session: 2026-07-25 — Telemetry sync pipeline + partner API test suite

**Date:** 2026-07-25
**Tags:** #session #backend #config

### Summary
Built the **automatic data pipeline** that keeps `robot_task_reports` current, and locked the partner API's security guarantees behind tests.

**The gap:** nothing kept telemetry fresh. [[TelemetrySyncService]] had `syncRange`/`syncYesterday`, but a grep showed **no callers** — data only ever landed via the manual `POST /api/v1/telemetry/sync/{serial}` or during a monthly report run. So the report pages and the new partner API would have served stale or empty data.

**The pipeline:** new [[TelemetrySyncScheduler]] (daily cron, **off by default**) drives `syncAllActive`, which was rewritten around a **look-back window** rather than "yesterday only" — brand APIs backfill late tasks, and a robot offline for a day would otherwise lose that data permanently. Re-syncing is safe because dedup is on external task id (idempotent).

**Tests:** 19 new tests across auth (8) and scoping (11). Crucially they were **mutation-verified** — deleting the `isServicedBy` guard in [[PartnerDataService]] made exactly the two cross-tenant tests fail (`404` → `200`, i.e. partner A reading partner B's robot), proving the suite actually catches a data leak rather than just passing. Full suite: **21 tests, 0 failures**.

### Files Modified
- [[TelemetrySyncScheduler]] (`telemetry/scheduler`) — NEW; `@ConditionalOnProperty(app.telemetry.sync-enabled)`, cron + zone configurable, `AtomicBoolean` guard so a slow run never overlaps the next tick, and it never throws (an exception in a `@Scheduled` method kills the schedule)
- [[TelemetrySyncService]] — `syncAllActive(from,to)` replaces the unused `syncRange`/`syncYesterday`: driven by one fetch-joined query of active deployments (no N+1, customer preloaded), **batch dedup** (one `IN` query + one `saveAll` instead of ~2 queries per report), per-robot error isolation, and a `SyncSummary`. Deliberately **not** `@Transactional` so a multi-minute fleet sync never pins a DB connection. Also fixed a latent **NPE** on tasks with no `startTime` (`report_month` derives from it and is NOT NULL) and now dedupes repeats *within* a paginated batch
- [[TelemetryAdapter]] — added `default isConfigured()`; [[GausiumAdapter]] overrides it
- [[TelemetryAdapterRegistry]] — added non-throwing `findAdapter()` → `Optional`
- [[RobotTaskReportRepository]] — `findExistingExternalTaskIds(ids)` for batch dedup
- [[TelemetryController]] — NEW `POST /api/v1/telemetry/sync-all?from=&to=` for on-demand fleet sync / backfill
- [[application.properties]] — `app.telemetry.sync-enabled|sync-cron|sync-zone|sync-lookback-days` (`TELEMETRY_SYNC_*`, default off, 03:00 Asia/Bangkok, 3-day look-back)
- [[TelemetrySyncSchedulerTest]] — NEW; boots the context **with the scheduler enabled** so a bad cron or unresolved `@Value` fails the build, not the deploy
- [[PartnerApiSecurityTest]] — NEW 8 tests: valid key authenticates and identifies its partner; missing / unknown / malformed / revoked key → 401; **disabled partner's live key → 401** (kill-switch); partner key cannot reach `/api/v1/**`; no-credential request cannot reach `/api/partner/**`
- [[PartnerDataScopingTest]] — NEW 11 tests: `/robots` returns only this partner's robots (excludes another partner's *and* RAASPAL-direct); own robot readable; another partner's and unknown serial both 404 with the same message shape; `month`, single-day, and `from`/`to` (precedence over `month`) filters; invalid + reversed dates → 400; page size clamped to 100

### Decisions Made
- **Look-back window (default 3 days), not "yesterday"** — brand APIs backfill and robots go offline; a single-day sync silently loses data forever. Safe only because sync is idempotent on external task id.
- **Scheduler off by default** (`TELEMETRY_SYNC_ENABLED=false`) — deploying the pipeline must not start calling brand APIs unexpectedly; on-demand sync keeps working regardless.
- **`syncAllActive` is not transactional** — a fleet sync is minutes of external HTTP; one transaction across it would pin a connection and let one robot's failure roll back another's data. Each robot commits independently.
- **Skip unconfigured/unsupported brands once per brand** via `isConfigured()` + `findAdapter()` — previously an unconfigured brand threw once *per robot*, so 70+ robots meant 70+ stack traces per run. Also makes the sync safe to enable before Gausium credentials exist.
- **Verify tests by mutation** — a passing security test proves nothing until you watch it fail; the cross-tenant guard was removed and the suite confirmed to break.
- **Always run `mvnw test`, not just `compile`** — `compile` never boots the Spring context, which is how the `@Order` misplacement reached Render.

### Unresolved / Next Steps
- [ ] Set `GAUSIUM_API_CLIENT_ID` / `_CLIENT_SECRET` / `_OPEN_ACCESS_KEY` on Render, verify with `POST /api/v1/telemetry/sync-all` for one day, **then** set `TELEMETRY_SYNC_ENABLED=true`
- [ ] Consider surfacing data freshness (`synced_at` → a `lastSyncedAt` field) so partners know how current the data is
- [ ] Consider persisting a sync-run history table if the team wants an audit of scheduled runs
---

## Session: 2026-07-25 — Partner API production hardening (rate limit, audit, key expiry)

**Date:** 2026-07-25
**Tags:** #session #backend #frontend #database #config

### Summary
Hardened the partner API for production across four fronts: **per-key rate limiting**, an **access audit trail**, **key expiry + rotation nudges**, and strict **`month` validation**. `V23` adds `partner_api_keys.expires_at` (nullable, so every existing key keeps working) and the `partner_api_access_logs` table. Backend **34 tests pass**; frontend build clean.

**A real bug the new tests caught.** The audit filter sits outermost so it can see the final status (including 401s and 429s), and originally read the caller from the SecurityContext *after* the chain ran. Spring Security's `SecurityContextHolderFilter` **clears the context on the way out**, so successful requests were being recorded with `partner_id = NULL` — an audit trail that silently attributed every call to nobody. Fixed by having [[ApiKeyAuthFilter]] stash the principal in a **request attribute** (which lives for the whole request), with the SecurityContext kept as a fallback. This is exactly the class of defect that only surfaces when you assert on the stored row rather than the response.

**Trade-off taken on the audit store:** a DB table rather than log-only, because "who fetched what, when" needs to be *queryable* for usage review and disputes. Volume is tiny (a partner pulls periodically — PCS's fleet is ~100 requests/day), the write can never fail a response, and the honest cost — unbounded growth — is mitigated by an enforced retention job rather than left as a hope.

### Files Modified
- [[V23__harden_partner_api.sql]] (`db/migration`) — `partner_api_keys.expires_at` (nullable = never expires) + `partner_api_access_logs` with indexes on `(partner_id, requested_at DESC)` and `requested_at`
- [[PartnerRateLimitFilter]] (`partner/security`) — NEW fixed-window counter per partner in a **Caffeine** cache (already a dependency, so **no Redis/new infra**); 429 + `Retry-After`, and `X-RateLimit-Limit`/`-Remaining` on success so a client can pace itself. Runs **after** auth so the budget is per key, not per IP. Documented limitation: the budget is **per instance**, so multiple instances multiply it
- [[PartnerAccessAuditFilter]] (`partner/security`) — NEW; records method/path/query/status/duration/IP per request, **including rejected ones** (null partner — the rows that reveal a leaked or probing key). Prefers `X-Forwarded-For` (Render terminates TLS upstream). Audit failure is swallowed with a warning — observability must never break the API
- [[PartnerApiAccessLog]] + [[PartnerApiAccessLogRepository]] — NEW entity/repo, incl. `deleteOlderThan` for retention
- [[PartnerAuditCleanupScheduler]] (`partner/scheduler`) — NEW nightly prune past `audit-retention-days` (default 90); never throws
- [[PartnerApiKey]] — added `expiresAt` + `isExpired()` / `isUsable()`; expiry is rejected exactly like revocation but needs no admin action
- [[PartnerApiKeyService]] — `generate(partnerId, label, expiresInDays)` (old 2-arg overload kept, so existing callers/tests are unaffected); `resolve` now filters on `isUsable()`
- [[PartnerPrincipal]] — added `apiKeyId` so the audit distinguishes *which* key was used ("PCS called us" vs "PCS's retired server key is still calling us")
- [[ApiKeyAuthFilter]] — publishes `PARTNER_PRINCIPAL_ATTRIBUTE` for the audit filter (see the bug above)
- [[PartnerSecurityConfig]] — wires audit → auth → rate limit, each anchored to a **well-known** filter (`WebAsyncManagerIntegrationFilter`, `UsernamePasswordAuthenticationFilter`, `AuthorizationFilter`) so the order is unambiguous
- [[PartnerDataService]] — `month` now parsed via `YearMonth`; a typo like `2026-7` or `July` is a **400**, not a silently empty result
- [[PartnerAdminController]] / [[PartnerService]] — `expiresInDays` on mint; NEW `GET /api/v1/partners/{id}/access-log` (paged) so the audit is actually visible
- `dto/` — `CreateApiKeyRequest.expiresInDays`, `CreatedApiKeyResponse.expiresAt`, `ApiKeyResponse` gains `expiresAt`/`expired`/`expiringSoon` (7-day rotation warning), NEW `AccessLogResponse`
- [[application.properties]] — `app.partner.rate-limit-enabled|rate-limit-per-minute` (120), `audit-enabled`, `audit-retention-days` (90), `audit-cleanup-cron`
- [[PartnersPanel]] / `types/api.ts` / `messages/en.json` + `th.json` — expiry dropdown on mint (never / 30 / 90 / 365 days) and per-key status showing expiry date, **Expired** vs **Revoked** as distinct states, and an amber **Expiring soon** nudge
- [[PartnerApiHardeningTest]] — NEW 13 tests: future-dated key works, backdated key → 401, minted expiry reported, `expiresInDays < 1` rejected; within-limit requests advertise the budget, exceeding → 429 + `Retry-After`, limit **scoped per partner** (one partner cannot throttle another); success audited with partner + key ids, rejection audited with null partner, audit captures the query string; malformed / non-numeric `month` → 400 and well-formed accepted

### Decisions Made
- **Caffeine fixed-window over Redis** — no new infrastructure for a handful of partners; swappable later without touching callers. The per-instance caveat is documented in config rather than hidden.
- **Rate limit after authentication** — metering per key is the useful unit; unauthenticated requests fall through to the 401 they deserve. (Per-IP throttling of unauthenticated attempts was consciously left out.)
- **Audit in the DB, with enforced retention** — queryable beats greppable for usage review; retention makes the growth cost explicit and bounded.
- **Audit records failures too, with a null partner** — rejected attempts are the most security-relevant rows.
- **Principal via request attribute, not SecurityContext** — the context is cleared before an outermost filter regains control.
- **Expiry nullable, defaulting to never** — hardening must not break the keys already issued.
- **Expired ≠ revoked in the UI** — one needs a fresh key, the other was deliberately killed; collapsing them would mislead.
- **Strict `month` validation** — a silent wrong answer ("this robot did nothing that month") is worse than an error.

### Unresolved / Next Steps
- [ ] Optional: per-IP throttle for unauthenticated partner requests (currently only authenticated callers are metered)
- [ ] Optional: surface the access log in the Partners UI (endpoint exists; no panel yet)
- [ ] Revisit the rate limit if RAASPAL ever runs more than one backend instance (budget multiplies per instance)
---

---
## Session: 2026-07-30 — Partner API hardening round 2: filter scope, token throttle, spoofable IP, secret guard

**Date:** 2026-07-30
**Tags:** #session #backend #ai #database #config

### Summary
Audited the partner API for production readiness rather than waiting for a failure, and found three defects plus
one deployment trap. The largest: every filter in `partner.security` was a `@Component`, and Spring Boot registers
each `Filter` bean with the servlet container at `/*` **independently of** `HttpSecurity.securityMatcher` — so
being added to the partner chain did not remove them from anywhere, it added a second place they ran. Every staff
request, Swagger page and Render health ping was passing through them, and [[PartnerAccessAuditFilter]] has no
partner-path check, so it wrote a `partner_api_access_logs` row for each one: an insert apiece on the free Supabase
tier, every row with `partner_id = NULL`, which is exactly what a failed partner authentication looks like. Never
an authorisation bypass — the container copies sort after `springSecurityFilterChain`, so authorisation had already
happened — but it buried the one signal the audit table exists to surface. A test was written **before** the fix and
confirmed to fail (one staff login produced one partner audit row) so the fix was proven rather than assumed.

Second: the token endpoint is `permitAll` and [[PartnerRateLimitFilter]] meters per *authenticated* partner, which
by construction cannot cover the one route reachable without a credential. An anonymous loop bought a credential
lookup plus an audit insert per request, unbounded. This is not about guessing secrets — 256 bits of entropy is not
reachable at any request rate — it is about the database work and audit growth an unauthenticated caller can drive.
Now metered per IP. Third, and coupled to it: [[ClientIpResolver]] replaced leftmost `X-Forwarded-For` parsing with
rightmost. The leftmost entry is whatever the caller claimed, so the new throttle would have been resettable one
forged header at a time; the rightmost is what the proxy appended. **Mutation-verified** — restoring leftmost makes
the forgery test fail 429 → 401, and nothing else.

Finally, [[PartnerSecretStartupCheck]] now refuses to start when `app.partner.jwt.secret` is still the placeholder
committed to [[application.properties]]. That key signs the bearer carrying the partner id every query is scoped to,
so shipping it means the signing key is not weak but published — anyone could mint a token naming any partner. A
startup warning is not proportionate to that, and nobody reads startup logs on a green deploy. Local development
opts out explicitly via [[application-local.properties]], deliberately a written-down flag rather than a profile
check, since which profile is active depends on how the process was launched. 74 → 83 tests.

### Files Modified
- [[PartnerSecurityConfig]] (`partner/security/PartnerSecurityConfig.java`) — three disabled `FilterRegistrationBean`s; stale "authenticates by X-API-Key" javadoc corrected to the token flow
- [[ApiKeyAuthFilter]] (`partner/security/ApiKeyAuthFilter.java`) — `@Component` removed; documented as retired and why the missing annotation is load-bearing
- [[PartnerRateLimitFilter]] (`partner/security/PartnerRateLimitFilter.java`) — second budget for the `permitAll` token endpoint, metered per IP; false "other chains never register this filter" comment removed
- [[ClientIpResolver]] (`partner/security/ClientIpResolver.java`) — **new**; rightmost `X-Forwarded-For` with the reasoning for it
- [[PartnerSecretStartupCheck]] (`partner/security/PartnerSecretStartupCheck.java`) — **new**; fails startup on a placeholder partner secret, logs the staff one
- [[PartnerPrincipal]] (`partner/security/PartnerPrincipal.java`) — owns `REQUEST_ATTRIBUTE`, moved off the retired filter
- [[PartnerAccessAuditFilter]] (`partner/security/PartnerAccessAuditFilter.java`) — uses [[ClientIpResolver]]; javadoc points at [[PartnerJwtAuthFilter]]
- [[PartnerJwtAuthFilter]] (`partner/security/PartnerJwtAuthFilter.java`) — attribute constant updated
- [[V25__index_task_reports_by_robot_and_start_time.sql]] (`db/migration/`) — **new**; `(robot_unit_id, start_time DESC)` for the paginated partner queries
- [[application.properties]] — `app.partner.token-rate-limit-per-minute`, `app.security.allow-placeholder-secrets`
- [[application-local.properties]] — opts in to placeholder secrets so local runs need no extra environment
- `src/test/resources/application.properties` — token budget raised out of the way of the functional tests
- `PartnerFilterScopeTest` — **new** (2); a staff request must leave no audit row, and every partner filter must be excluded from the container
- `PartnerTokenThrottleTest` — **new** (3); own context with a small budget, including the forged-header case
- `PartnerSecretStartupCheckTest` — **new** (6)
- `public/gs-middleware-api.html` + published artifact — token-endpoint budget and the 429 cause PCS will actually hit

### Decisions Made
- **Disable the container registration rather than drop `@Component`:** the beans are still needed by the chain, and `FilterRegistrationBean(enabled=false)` is the one mechanism that separates "Spring Security may use this" from "the servlet container must not".
- **Test the general rule, not today's four filters:** `PartnerFilterScopeTest` fails for any future filter added to `partner.security` without an exclusion, because `@Component` on a filter is the ordinary thing to do everywhere else in this codebase.
- **`ApiKeyAuthFilter` leaves the container entirely:** it belonged to no chain yet still ran app-wide, so an `X-API-Key` header on any staff path bought a credential lookup and a `last_used_at` write. A retired credential should do no database work.
- **Meter the token endpoint per IP, not per client id:** there is no identity until the exchange succeeds. Accepted limitation — per-instance budget, same as the data limiter.
- **Rightmost `X-Forwarded-For`:** correct or strictly better under every deployment this service has; only wrong behind two trusted proxies (e.g. Cloudflare in front of Render), noted in [[ClientIpResolver]] for whenever that changes.
- **Fail startup on a placeholder partner secret, only warn on the staff one:** the partner API is not live yet, so failing fast costs nothing; the staff service is deployed, and failing its boot would be an outage rather than a fix.
- **Opt out by property, not by profile:** a flag has to be written down deliberately; an active profile is an accident of how the process was launched.
- **V25 is the report index,** so the deferred live-status design must renumber to `V26`.

### Unresolved / Next Steps
- [ ] ⚠️ Set `PARTNER_JWT_SECRET` on Render **before** deploying — the app now refuses to start without it (this is the intended behaviour, not a regression)
- [ ] Set `JWT_SECRET` on Render too if it is still the placeholder; startup logs it as an error
- [ ] Local manual test of the OAuth exchange (Postman raw-body dropdown must be **JSON**, not Text)
- [ ] Onboard PCS: mint a fresh credential, bulk-assign its ~70 deployments, deliver the secret securely
- [ ] Awaiting Gausium's reply on the 20 `cleaningMode` values — most valuable answer is whether a canonical enum exists, so the mapping stops being reactive
- [ ] Optional: surface the access log in the Partners UI (endpoint exists; no panel yet)
- [ ] Revisit both rate limits if RAASPAL ever runs more than one backend instance (each budget multiplies per instance)
---
---
## Session: 2026-07-30 — Partner token lifetime raised to 24 hours

**Date:** 2026-07-30
**Tags:** #session #backend #config

### Summary
Raised the partner bearer lifetime from 1 hour to 24 hours at the user's request, on the reasoning that the
payload is read-only robot cleaning history rather than anything sensitive. The usual objection to long-lived
bearers — that you cannot withdraw one before it lapses — does not apply to this design: [[PartnerJwtAuthFilter]]
re-checks the credential behind the token on **every** request via the `apiKeyId` claim, so revoking a credential
or disabling a partner still kills its live tokens immediately. Expiry therefore bounds only the window in which a
**leaked** token is useful to someone who has not been noticed yet, which is a fair trade for this data.

Changed the default in one place (`app.partner.jwt.expiration-ms`) since it was already externalised as
`PARTNER_JWT_EXPIRATION_MS`, and kept the `@Value` fallback in [[PartnerTokenService]] in step so the two cannot
drift. The test override was raised to match production rather than left at the old hour: as written it pinned a
value that existed only in tests, so the assertion said nothing about what a partner actually receives.

Documentation carried four separate promises of "an hour" — the summary table, the caching advice, the rate-limit
note and the worked example — all updated. Also added explicit advice to read `expires_in` from the response
instead of hard-coding 24 hours, so a future change to the lifetime does not silently break PCS's client.

### Files Modified
- [[application.properties]] — `app.partner.jwt.expiration-ms` default 3600000 → 86400000, with the revocation reasoning recorded
- [[PartnerTokenService]] (`partner/security/PartnerTokenService.java`) — `@Value` fallback kept in step
- `src/test/resources/application.properties` — override raised to the production value, with why
- `PartnerOAuthTest` — `expires_in` assertion 3600 → 86400
- `public/gs-middleware-api.html` + published artifact — all four "1 hour" references, plus the `expires_in` advice

### Decisions Made
- **24 hours is acceptable here because revocation is independent of expiry:** the per-request credential re-check is what makes a long bearer safe, and it already existed.
- **Test config matches production:** an assertion against a test-only lifetime documents nothing about the real contract.
- **Tell PCS to read `expires_in`, not to trust the documented number:** the lifetime is now a value we may tune, so their client should not encode it.

### Unresolved / Next Steps
- [ ] Republished artifact is `33c6dced-…`; the earlier `f8e87e36-…` link from this session no longer exists, so any URL shared from it is dead
- [ ] Still open: the partner/credential **pairing check** — a token's `partnerId` and `apiKeyId` are validated independently, so the signing secret is currently the only barrier to pairing one partner's `sub` with another's usable `apiKeyId`
---
---
## Session: 2026-07-30 — Telemetry sync moved to midnight; production sync confirmed active

**Date:** 2026-07-30
**Tags:** #session #config #backend

### Summary
Confirmed from the Render environment that the nightly telemetry sync **is** running in production:
`TELEMETRY_SYNC_ENABLED=true` plus all three `GAUSIUM_API_*` credentials are set, which are the two independent
switches the sync needs — the scheduler bean only exists when the flag is true
(`@ConditionalOnProperty`), and `GausiumAdapter.isConfigured()` skips the brand when the credentials are blank.
Both being set means [[TelemetrySyncScheduler]] has been keeping `robot_task_reports` current on its own, and the
partner API's freshness does not depend on anyone clicking Sync.

Changed the schedule from 03:00 to **midnight Asia/Bangkok** at the user's request. One line, since the cron was
already externalised. Noted in [[application.properties]] that the 3-day look-back is what makes a midnight run
safe for tasks still in progress at the time: a robot cleaning across midnight has no completed report to fetch
when the run fires, and the following night's window picks it up rather than losing it.

⚠️ `TELEMETRY_SYNC_CRON` set as a Render environment variable would override this default — the user was asked to
confirm it is absent, or to set it to `0 0 0 * * *`.

### Files Modified
- [[application.properties]] — `app.telemetry.sync-cron` default `0 0 3 * * *` → `0 0 0 * * *`, with the midnight/look-back interaction recorded
- [[index]] — telemetry section corrected: the scheduler is off by default *in code* but enabled in production

### Decisions Made
- **Change the code default rather than only the Render variable:** the default is what someone reads to learn the intended schedule, so leaving it at 03:00 while production ran at midnight would be a lie in the most-read place.
- **Midnight accepted despite overnight tasks:** the look-back window already absorbs a task that completes after the run, at the cost of it appearing a day later. Called out rather than silently traded away.

### Unresolved / Next Steps
- [ ] Confirm `TELEMETRY_SYNC_CRON` is not set in Render's environment, or set it to `0 0 0 * * *` — an env var beats the new default
- [ ] Decide whether daily is frequent enough for PCS, or move to `0 0 */6 * * *` for same-day visibility
- [ ] Nothing runs with `refresh = true` on a schedule, so Gausium's later **corrections** to already-stored rows are never picked up (this is what required the manual Refresh for the blank `mapName` rows). Consider a weekly refresh run — deferred pending Gausium's reply on how often they revise data
---
---
## Session: 2026-07-30 — Gausium's own cleaning-mode wording adopted; raw Chinese removed from the partner response

**Date:** 2026-07-30
**Tags:** #session #backend #ai

### Summary
Gausium answered the list of every distinct `cleaningMode` value in the database, so [[CleaningModeLabels]] now
carries **their** English rather than ours. Their answers overturned two reasonable guesses: `洗地` is
**Scrubbing**, not the literal "Floor Washing", and it is a **distinct activity** from `清洗` (**Washing**) — which
we had been one decision away from merging, since 466 tasks looked like a near-duplicate label. Merging them would
have quietly folded two different machine operations into one bucket in PCS's reporting. Also renamed on their
authority: `尘推` → Dust Mopping (confirming it is the same activity as the English `dust mop` code, which had been
inferred), `重度/中度清洁` → Heavy-Duty / Medium-Duty Cleaning, `巡检` → Patrol Inspection, `吸水` → Water Sucking,
`洗扫` → Scrub & Sweep. The wording is now recorded in the class as authoritative and not to be "improved" locally:
where their term differs from the obvious reading, theirs is what PCS and Gausium's own staff will recognise.

Separately, **`cleaningModeRaw` was removed from the partner response** at PCS's request — they asked not to
receive the Chinese at all. Shipping both fields also invited a partner to build against the un-normalised form and
inherit every firmware inconsistency the mapping exists to absorb. The raw value is still **stored** unchanged, so
translation stays a presentation concern: RAASPAL staff can reconcile against Gausium's portal, and a future
mapping correction fixes every historical response without re-syncing a row.

### Files Modified
- [[CleaningModeLabels]] (`telemetry/core/CleaningModeLabels.java`) — all 14 Chinese terms plus their English twins set to Gausium's wording; javadoc records the source, the date, and the two guesses it overturned
- [[PartnerTaskReportResponse]] (`partner/dto/PartnerTaskReportResponse.java`) — `cleaningModeRaw` removed, with why keeping it was worse than dropping it
- `CleaningModeLabelsTest` — all 20 production values re-pinned to the supplied labels; prefix test updated
- `PartnerDataScopingTest` — fixture takes a cleaning mode, June task now carries a real `洗地`; new `taskReportsCarryTheTranslatedModeAndNoChineseAtAll` asserts the translated label, the absence of the raw field, and **no ideographic character anywhere in the body**
- `public/gs-middleware-api.html` + published artifact — full label list corrected, raw field removed from the sample and the field reference, Patrol Inspection flagged as non-cleaning

### Decisions Made
- **The supplier's wording wins over better-reading English:** the point of asking was to be right about meaning, and reconciliation with Gausium's own staff matters more than style. "Water Sucking" is kept verbatim despite reading awkwardly.
- **`洗地` and `清洗` stay separate:** confirmed as different operations, closing an open question that was leaning the other way.
- **Assert "no CJK anywhere in the body", not one field:** a per-field check would pass while another field leaked Chinese, and the request was about the whole response.

### Unresolved / Next Steps
- [ ] Gausium did not confirm whether a **canonical enum** of all possible modes exists, so the mapping is still reactive to firmware that emits something new — an unmapped value reads as `Other` and is logged once
- [ ] "Water Sucking" reads oddly for a customer-facing report (7 tasks); switch to "Water Suction" if RAASPAL prefers house style over the supplier's literal term
- [ ] `mop_wet` / `sweep_vacuum` / `mop` / `sweep` / `vacuum` / `scrub` were not in the supplier's list — they are English codes Gausium already emits, mapped by inference
------
## Session: 2026-08-05 — Corrective Maintenance report auto-creation; Reports hub split into sub-sections

**Date:** 2026-08-05
**Tags:** #session #backend #frontend #ai #database

### Summary
Replaced the manual Corrective Maintenance (CM) workflow — copy a cleaning-robot ticket out of Monday.com,
paste it into an Excel template, print to PDF — with a paste → AI-extract → review → print flow that saves
what it issues. The operator pastes the raw ticket, Claude splits it into the nine report fields, the operator
corrects anything wrong and attaches the two signature photos, and [[CorrectiveMaintenanceReportView]] renders
the exact Thai paper form (`รายงานการซ่อมบำรุงแก้ไข`) for `window.print()`.

**Extraction runs on Haiku 4.5, not the default Sonnet.** Pulling nine labelled fields out of a blob is not a
reasoning task, and at ~$0.009/report cost stops being a consideration at any realistic volume. The prompt's
governing rule is *transcription, not authorship*: the Thai must survive byte-for-byte onto a document a customer
signs, so it is told not to translate, summarise, or correct — and to return `null` rather than invent. The MVP
prompt rules were deliberately **not** reused, because the `Needs confirmation` sentinel would print as literal
text on the form.

**Signatures are uploaded per report, not fixed assets.** The first design embedded the two PNGs in
`RaasPal-Internal-Ops-backend/docs/`; the user corrected this — the technician photographs each signed line and
uploads both through the UI. They are stored as base64 `data:` URIs in the row rather than via
[[FileUploadService]], because that writes to local disk and **Render's disk is ephemeral**, so a redeploy would
silently break reprints of past reports. Since a phone photo is 3–5 MB and base64 inflates it by a third, the
client downscales to a 600px long edge before upload and the service enforces a hard 512 KB ceiling — the resize
is a convenience, the server check is the boundary.

The Reports hub had four tabs that were all *robot performance* reporting, so they were grouped under one
sub-section and CM became a sibling. The group is **derived from the tab value** rather than tracked in a second
query param, so existing `?tab=automation` and `?tab=autoxing` links keep working untouched.

### Files Modified
- [[V27__add_cm_reports.sql]] (`db/migration/`) — `cm_reports`; V26 skipped, still reserved for partner live status
- [[CmReport]] (`cm/entity/`) — only `customerName`/`reportDate` required; partial tickets must be savable
- [[CmReportRepository]] (`cm/repository/`) — `search()` over ticket no. / customer / serial, newest-first
- [[CmReportService]] (`cm/service/`) — parse (persists nothing) + CRUD; signature size/type validation
- [[CmReportController]] (`cm/controller/`) — `/api/v1/cm-reports` incl. `POST /parse`
- [[CmReportDraft]] (`ai/dto/`), [[CmReportExtractionService]] (`ai/service/`) — the AI contract
- [[ClaudeAiService]] — implements the new interface on `claude-haiku-4-5-20251001`; degrades to an empty draft
- [[MockAiService]] — deterministic Thai-label scanner, genuinely usable locally without an API key
- [[AiPromptTemplates]] — `cmReportExtractionSystemPrompt()` incl. the Thai→ISO Buddhist-era date rules
- `CmReportApiTest` — 11 tests; suite now **95, all passing**
- [[CorrectiveMaintenanceReportView]] (`components/report/`) — the printable document, fixed Thai labels
- [[CmReportPanel]] / [[CmReportHistoryPanel]] (`components/`) — build one report / find and reprint a past one
- [[signature-image]] (`lib/`) — canvas downscale + data-URI encode; `createImageBitmap` with an `<img>` fallback
- [[thai-date]] (`lib/`) — Buddhist-era formatting, parsed field-by-field to dodge the UTC off-by-one-day
- [[ReportsClient]] / [[reports/page.tsx]] — group selector; `TAB_GROUP` derives the group from `?tab=`
- [[lib/api.ts]], [[types/api.ts]], `messages/en.json`, `messages/th.json`

### Decisions Made
- **Parse and save are separate endpoints:** an abandoned parse must not leave a row, and re-parsing a bad paste must not duplicate a report.
- **Haiku over Sonnet for extraction:** field-picking, not reasoning; ~3× cheaper and reviewed by a human regardless.
- **AI fills the form, the form is the source of truth:** nothing reaches a signed customer document unreviewed. A `null` field never overwrites something the operator already typed.
- **Signatures inline in the row, not on disk:** Render's ephemeral disk would break reprints. Size is then the whole problem, hence the client resize plus a server ceiling.
- **Report labels are hard-coded Thai, not i18n:** this is a Thai document that gets signed and filed; its wording must not change when a staff member switches the app to English.
- **Group derived from `?tab=`, not a second param:** keeps every existing Reports link working with no migration.
- **Steps stored newline-separated, numbered at render time:** they only ever appear as an ordered list in one cell; a child table would buy nothing.

### Unresolved / Next Steps
- [ ] **Visual fidelity vs the paper form is unverified** — no browser automation in this session. Compare `/en/reports?tab=cm-new` against `Pandora - CM Report-17 June 2026.pdf` (row order, spacing, one-A4-page fit) and adjust the view.
- [ ] **Verify the footer company name** in [[CorrectiveMaintenanceReportView]] (`COMPANY_FOOTER`) against the company registration — transcribed from a scan, and `พอล` vs `พาล` for "PAL" could not be settled from the image. It sits beside a tax ID.
- [ ] Confirm the printed page fits one A4 sheet with **"Background graphics" off** — the table uses real borders for this reason, but it is untested.
- [x] **Extraction verified against live Haiku**, not just [[MockAiService]] (the local `app.anthropic.api-key` is set, so [[ClaudeAiService]] is the active bean). Exercised on deliberately *unlabelled* Thai prose — no `label :` lines, which the mock's scanner cannot parse at all: it converted a Buddhist-era date buried mid-sentence, pulled the ticket number out of running text, dropped the `คุณ` honorific from the officer's name, and split narrative into three discrete repair steps. Real Monday pastes won't be neatly labelled either, so this is the case that mattered.
- [ ] Still watch the first few *real* pastes — the prose test was synthetic, and Monday tickets may carry boilerplate/signatures the prompt hasn't seen.
- [ ] Consider a per-customer default for the provider signature, if it turns out to be the same technician's every time
---

---
## Session: 2026-08-10 — Workspace restructure, RIMS absorbed, AWS Lightsail hosting plan

**Date:** 2026-08-10
**Tags:** #session #deployment #config

### Summary
Two threads. First, planning the move of the backend off Render onto an **AWS Lightsail 8 GB instance**
(Singapore, US$44/mo — 2 vCPU / 160 GB / 5 TB) running Docker + Nginx, with Supabase retained as the
database for now. Second, absorbing the new **Robot Inventory Management System (RIMS)** into this
workspace. RIMS already existed as a standalone Next.js app at `D:\Work\SoftwareWorkSpace\raaspal-rims`
running on seed data; it was moved to `./raaspal-rims` with its git history intact (plain clone, no
worktrees, no `core.worktree`, so the move was path-safe). The decision on RIMS is **shared backend,
separate frontend** — the inventory domain genuinely joins the existing one (auth, [[Robot]]/[[RobotSpec]],
CM reports consuming parts, [[RobotUnit]]/[[Deployment]] driving which spares matter), but its UI already
has its own component library and design language, so merging the frontends would mean discarding working
code to save work already done.

### Files Modified
- [[CLAUDE.md]] (`CLAUDE.md`) — added `./raaspal-rims` to Project Mapping and a RIMS build/test block; noted
  that each service is its own git repo and the workspace root is not one
- [[raaspal-rims]] (`./raaspal-rims`) — moved in from `D:\Work\SoftwareWorkSpace\`; `.next` deleted (it caches
  absolute paths), `node_modules` kept
- [[index]] (`.claude/memory/index.md`) — RIMS + hosting sections

### Decisions Made
- **Inventory shares the backend as a bounded module:** `com.raaspal.robotrecommendation.inventory.*`, own
  tables, own `/api/v1/inventory/**`, Flyway **V28+** (V26 stays reserved for partner live status). Follows the
  precedent already set three times by [[cvte]], [[partner]] and [[cm]] — one app, hard internal boundaries.
  FKs point *into* existing tables, never the reverse, so the module stays extractable.
- **RIMS stays a separate Next.js deployment:** the duplicated-auth argument against a second frontend is moot
  because `lib/auth.ts`, `lib/rbac.ts`, `lib/store.ts` and a full `components/ui/` set already exist there.
- **Client-side RBAC is navigation, not enforcement:** `lib/rbac.ts` must be driven by backend-issued roles, and
  the backend must reject inventory writes from non-inventory users regardless of what the UI renders.
- **`api.raaspal.com` is deliberately product-neutral** — that backend now serves two frontends, so naming it
  after either would have aged badly. **A** record → Lightsail static IP; the two Vercel frontends get **CNAME**
  records (`app`/`rims` → `cname.vercel-dns.com`). All three under `*.raaspal.com` also makes the existing
  httpOnly `raaspal_token` cookie shareable at `Domain=.raaspal.com` — single sign-on across both apps.
- **Database stays on Supabase through the migration:** moving hosts and moving Postgres at once destroys the
  signal about which change broke what. Revisit once Lightsail is stable — the 160 GB disk would retire both the
  500 MB free-tier ceiling and the 15-connection session-pooler cap documented in [[application.properties]].
- **Do NOT set `server.forward-headers-strategy`:** Spring's `ForwardedHeaderFilter` strips `X-Forwarded-*` after
  processing, and [[ClientIpResolver]] reads the raw rightmost `X-Forwarded-For`. Setting it would collapse every
  anonymous caller into `127.0.0.1` and defeat the partner token throttle. `$proxy_add_x_forwarded_for` in Nginx
  appends the real peer on the right, which is exactly what that resolver expects.

### Unresolved / Next Steps
- [ ] **Phase 2 of the workspace move (manual, needs VS Code closed):** `D:\Work\AISolution` →
  `D:\Work\SoftwareWorkSpace\RaasPalOps`. Cannot be done from inside a session whose cwd is the folder.
- [ ] Copy the three path-keyed Claude session stores under `~/.claude/projects/` to their new keys, or the
  transcripts and auto-memory become unreachable: `d--Work-AISolution`,
  `D--Work-AISolution-RaasPal-Internal-Ops-backend`, `d--Work-SoftwareWorkSpace-raaspal-rims`
- [ ] Write the Lightsail deployment files (`deploy/docker-compose.yml`, `api.env.example`,
  `nginx/raaspal-api.conf`, `deploy.sh`, `DEPLOYMENT.md`) — designed this session, not yet written
- [ ] **Harvest every env var off Render before decommissioning it.** Copy `PARTNER_JWT_SECRET` and `JWT_SECRET`
  *verbatim* — regenerating them breaks live PCS partner tokens and logs out all staff respectively.
- [ ] During parallel running: `DB_POOL_MAX=5` on Lightsail (Supabase caps at 15) and keep
  `TELEMETRY_SYNC_ENABLED` / `REPORT_EMAIL_SCHEDULER_ENABLED` **off** there — two instances firing the delivery
  cron at the same minute could double-send customer emails
- [ ] Request three DNS records from the director (A: `api`; CNAME: `app`, `rims`). If DNS is Cloudflare, ask for
  the proxy **off** — two proxies breaks [[ClientIpResolver]]'s rightmost read
- [ ] `raaspal-rims` needs its own `CLAUDE.md`, and three uncommitted files there predate the move
  (`app/login/page.tsx`, `components/shell/app-shell.tsx`, `components/brand.tsx`)
- [ ] Add `GET /api/v1/auth/me` — RIMS can't inherit the console's localStorage state, so it needs an endpoint to
  hydrate the user after a shared-cookie login
---

---
## Session: 2026-08-11 — RIMS backend schema: V28 inventory + robot lifecycle, V29 cleaning specs redesign

**Date:** 2026-08-11
**Tags:** #session #database #backend

### Summary
Designed and wrote the two migrations that give [[raaspal-rims]] a backend. **V28** turns [[RobotUnit]] into an
asset register (it was built in V11 purely as a telemetry anchor, with no way to express "in the warehouse")
and adds fungible stock. **V29** rebuilds the cleaning spec table from the real Gausium datasheet
(`raaspal-rims/docs/DataSheetGS_filled_07152026.xlsx`) and adds a RIMS-owned display-spec table. The
spreadsheet was decoded by unzipping the `.xlsx` and parsing its XML — no Python or Excel tooling is
available on this machine, and `poi-ooxml` would have meant writing a throwaway Java program.

### Files Modified
- [[V28__add_inventory_and_robot_lifecycle]] (`RaasPal-Internal-Ops-backend/src/main/resources/db/migration/`) —
  `robot_units` gains `status`/`version`/`robot_type`/`robot_id`/`location`; new `inventory_items` +
  `stock_movements` + `inventory_item_sku_seq`
- [[V29__add_cleaning_specs_and_display_specs]] (same dir) — `robot_specs_cleaning` (101 spec columns,
  verified against the 103 sheet headers minus Brand/Model) + `robot_display_specs` (JSONB)

### Decisions Made
- **Robot status is 3 values — `IN_STOCK` / `RENT` / `SOLD`** (user's call, overriding an earlier 5-state
  physical lifecycle). Works because **a SOLD robot keeps its active deployment** — RAASPAL still monitors it and
  sends monthly reports after ownership transfers. So "sold" is commercial, not locational, and RIMS filters on
  the single condition `status = 'IN_STOCK'` to exclude both rented and sold units. `IN_REPAIR` was dropped:
  repairs are CM tickets, a separate concern.
- **Serialized vs fungible stock are modelled separately.** Robots stay in `robot_units` (serial, deployment,
  telemetry history); parts go in `inventory_items` + `stock_movements`. A separate `inventory_robots` table was
  rejected — `serial_number` is UNIQUE on `robot_units`, so the same physical robot would fragment across two
  tables, breaking exactly the sold-robot monitoring the system exists for.
- **The FK is the normalisation, not a text rewrite.** `robot_units.brand`/`model` keep their raw imported values;
  `robot_id` carries the curated meaning. So `Phantas V1.1` and `Phantas` both resolve to one catalogue model
  without mutating text that the Gausium sync path might rewrite.
- **Ledger + cached balance, one write path.** `inventory_items.quantity_on_hand` is written *only* by the
  movement service inside the transaction that appends to `stock_movements`. `quantity_delta` was renamed
  **`quantity_change`** — "delta" was not readable to the user, and naming should serve the reader.
- **`stock_movements` is append-only** — no `updated_at`, deliberately. Corrections are compensating
  `ADJUSTMENT` rows. That is what makes it an audit trail rather than a log.
- **Two SKU columns, not one nullable one.** `sku` is ours and always present (auto-generated `INV-000001` from
  `inventory_item_sku_seq` when blank); `supplier_part_no` is the manufacturer's, optional, and **not unique** —
  the same part may come from two suppliers under different codes. Generation happens in the **service**, not a
  column `DEFAULT`: Hibernate always includes mapped columns in its INSERT, so it would send `null` and trip
  `NOT NULL` before the default fired.
- **`robot_specs_cleaning` is a NEW table, expand/contract** — the old `robot_specs` is untouched so
  [[RecommendationService]] keeps working until verified against the new shape. Same pattern as V24.
- **Two spec tables for two audiences.** `robot_specs_cleaning` = typed columns the AI *compares*;
  `robot_display_specs` = JSONB `[{label,value,unit}]` the admin curates and RIMS *displays*. RIMS never reads
  the AI's table, so spec columns can be renamed or split per robot type freely. Contract is 3 fields.
- **JSONB over a row-per-spec-line table** — read whole, written whole, never queried into, and array position
  *is* display order. Trade accepted: duplicate-label and array-size checks move to the service layer.
- **`test_status = VERIFIED` is already the AI's filter** ([[RecommendationService]] L89), so stocked-but-not-sold
  models can live in `robots` as `DRAFT` without polluting recommendations. No new flag needed.

### Datasheet findings (from decoding the xlsx)
- 103 headers, row 2 = units, **rows 3–14 = 12 models**; booleans are `1`/`0`, `N/A` means NULL
- **Brush configuration is spec-distinct** — `Omnie Disc Brush` vs `Omnie Roller Brush` are different machines
  (Disc: scrub 520 / sweep N/A). Settles the long-open Roller/Disc/Blend question: separate **models**, not versions
- ⚠️ **But the fleet can't express it** — 73 units are recorded as plain `Omnie`, 16 as plain `M50`. Which
  physical unit is disc and which is roller is a **warehouse question no migration can answer**
- `Application` (all `1`) and `Floor layout Method` (all `0`) are constant — no signal; kept for sheet fidelity
- `Sensor Distance` 25–150 m is a real LiDAR range, **not** a unit error
- `Vibration more than` is numeric (`2.5`), not text; `HEPA` is text (`H13`), not boolean as in the old table
- Brand is misspelled **`Guasium`** on the two Phantas rows — must be normalised on import

### Unresolved / Next Steps
- [ ] ⚠️ **Local dev and production share one Supabase database.** Starting the app locally with Flyway enabled
  applies V28/V29 to production immediately. `pg_dump` first, rehearse against a local Docker Postgres, then apply.
- [ ] ⚠️ **Migrations are never exercised by `mvn test`** — `spring.flyway.enabled=false` and `ddl-auto=create-drop`
  on H2. Only real Postgres can validate them.
- [ ] V28 backfills all 152 deployed units to `RENT`. **Supply the sold serial numbers** and correct them:
  `UPDATE robot_units SET status='SOLD' WHERE serial_number IN (...)`
- [ ] Entities/repos/services/DTOs for both migrations; JSONB via `@JdbcTypeCode(SqlTypes.JSON)` (Hibernate 6 —
  no `hypersistence-utils` needed). Verify JSON maps on H2 in PostgreSQL mode.
- [ ] Import the 12 datasheet models into `robots` as `DRAFT` (adapting [[RobotImportService]], `poi-ooxml` is
  already a dependency) — deliberately not inserted by the migration, since several near-duplicate existing rows
- [ ] Re-point `robot_units.robot_id` for Omnie/M50 once disc-vs-roller is determined per serial
- [ ] Later migration: `DROP TABLE robot_specs` after the AI is verified against `robot_specs_cleaning`
- [ ] Phase 0 still blocks RIMS login: add `INVENTORY_STAFF` to [[Role]] (no migration needed —
  `role VARCHAR(20)`, no CHECK constraint), create real user accounts (**`users` has 1 row**), add
  `GET /api/v1/auth/me`, gate `/api/v1/inventory/**`
- [ ] **Are any robots physically in the warehouse today?** All 152 have active deployments, so after V28
  `status='IN_STOCK'` returns zero rows — meaning *receive into stock* is the first endpoint to build, not the read
---

---
## Session: 2026-08-13 — V28/V29 applied to production, Phase 0 auth, spec-migration strategy

**Date:** 2026-08-13
**Tags:** #session #database #backend #deployment

### Summary
Applied both migrations to the live Supabase database and completed Phase 0 of the RIMS backend. Sequencing
confirmed by the user: **finish inventory management first, move to AWS Lightsail afterwards** — so the
deployment track is parked deliberately, not forgotten.

### Incident: `robot_specs` renamed outside Flyway
Before the migration could run, `robot_specs` was found renamed to `robot_specs_cleaning` directly in the
Supabase dashboard, with Flyway still at V27. Two consequences: production was broken (the [[RobotSpec]] entity
maps `@Table(name = "robot_specs")`, so every spec query failed on Render, which shares this database), and V29's
`CREATE TABLE robot_specs_cleaning` would have collided. Fixed by renaming back, after which V29 created the new
table cleanly beside the old one as designed. **Rule reaffirmed: schema changes go through Flyway, data fixes may
go through the dashboard.** A dashboard schema edit is invisible to every other environment and to `git`.

### Files Modified
- [[Role]] (`common/enums/`) — added `INVENTORY_STAFF`
- [[SecurityConfig]] (`auth/security/`) — `/api/v1/inventory/**` → `hasAnyRole("ADMIN","INVENTORY_STAFF")`

### Decisions Made
- **Import Gausium specs only for now**, other brands later. Safe *only* because `robot_specs` stays live —
  AGIBOT C5 and KEENON C40 are `VERIFIED` and absent from the Gausium datasheet, so a premature drop would strip
  specs from two models the AI actively recommends. **Drop-gate:**
  ```sql
  SELECT r.brand, r.model FROM robots r
  JOIN robot_specs old ON old.robot_id = r.id
  LEFT JOIN robot_specs_cleaning new ON new.robot_id = r.id
  WHERE r.test_status = 'VERIFIED' AND new.robot_id IS NULL;
  ```
  Only drop `robot_specs` when that returns zero.
- **Do not bulk-copy `robot_specs` → `robot_specs_cleaning`.** Most columns map 1:1 (weight, dimensions, all five
  efficiencies, all four tanks, battery, access dimensions, functions, navigation, floor types) but three are
  lossy: `width_cleaning_mm` splits into sweep/scrub/mop with no way to know which; `hepa` goes boolean → text
  (`true` → inventing `"H13"`); the six `layout_*` tile booleans have no target. Order when brands are added:
  **import the Excel first, then transfer non-Gausium rows** — no upsert needed that way.
- **Specs are not AI-private.** `robot_specs_cleaning` hangs off `robots.id` and any system may read it; the
  **DTO** is what tailors it per consumer. [[robot_display_specs]] is presentation *configuration*, not a second
  copy of the facts. Possible refinement: a "prefill from datasheet" action that copies chosen values out of
  `robot_specs_cleaning` rather than admins retyping them.
- **`robot_units_backup_20260811` can be dropped.** It was taken *before* V28, so it holds only the seven original
  columns — and V28 modified none of them (every `UPDATE` wrote to a column that did not previously exist). It is
  byte-identical to the live table on every column it contains, and has no `status`, so it cannot even help with
  the pending sold-serial correction.
- **Local pool capped at 3** for the migration run — Render is live against the same Supabase pooler, which tops
  out at 15 connections.

### Unresolved / Next Steps
- [ ] ⚠️ **V28/V29 are still uncommitted.** The database is at v29 while the code knows v27, so Flyway will refuse
  to start on Render's next deploy (`Detected applied migration not resolved locally`). **Render is one redeploy
  from being down.** Commit and push to the branch Render builds.
- [ ] Run the `IN_STOCK` diagnostic — `deployment_rows = 0` means junk data to delete; `1` and inactive means a
  genuinely returned robot and the first real stock row
- [ ] Supply the sold serial numbers; all 152 are currently `RENT`
- [ ] Phase 1: [[RobotUnit]] entity + `RobotUnitStatus` enum + `GET /api/v1/robot-units?status=IN_STOCK`
- [ ] Phases 2–6: inventory entities → stock movements → display specs → catalogue fill + importer → RIMS wiring
- [ ] Create real user accounts via the existing `POST /api/v1/users`; `users` still holds one row, and
  `stock_movements.created_by` is meaningless while everyone shares a login
- [ ] Pre-existing: H2 test schema logs `Table "REQUIREMENTS" not found` during FK creation — Hibernate continues,
  so at least one FK is absent from the test schema and the suite validates less than it appears to
- [ ] **No real backups exist.** A same-database table copy never was one. Install PostgreSQL client tools for
  `pg_dump` — it also unlocks rehearsing migrations locally, which was impossible for V28
---

---
## Session: 2026-08-14 — RIMS connected to the backend: identity, stock intake, inventory API

**Date:** 2026-08-14
**Tags:** #session #backend #frontend #database

### Summary
Connected [[raaspal-rims]] to the Java service. RIMS no longer stores accounts or passwords: identity moved to
the backend, `data/users.json` and the six-digit PIN are gone, and warehouse staff receive robots and adjust
part counts through real endpoints. Scope was set by the user: **robot counts + specs, parts inventory,
team-wide read, inventory-staff write, low-stock alerts** — multi-warehouse, lease tiers and marketing copy
from RIMS's mock data were dropped.

### Files Modified
- [[UserController]] / [[UserService]] / [[UserRepository]] — `@PreAuthorize("hasRole('ADMIN')")` on the class;
  new `PATCH /api/v1/users/{id}` with a last-admin guard
- [[RobotUnitService]] / [[RobotUnitController]] — `receiveIntoStock`, `updateStockUnit`, `updateStockStatus`,
  `listWarehouse`; `RegisterRobotRequest` gains `robotType`, registration now sets `status = RENT`
- [[RobotUnitStatus]] — **DEMO added** (no migration: `status` is `VARCHAR(20)`, no CHECK)
- [[V30__add_robot_unit_image]] — `robot_units.image_url TEXT`
- `inventory/*` — `InventoryItem`, `StockMovement`, `MovementType`, repositories, `InventoryService`,
  `InventoryController`
- [[SecurityConfig]] — **`@EnableMethodSecurity`**; `/api/v1/inventory/**` relaxed to `authenticated()` with
  per-method write guards
- Console: `/register` route **deleted**, removed from `PUBLIC_PATHS`, `authApi.register` and `RegisterRequest` gone
- RIMS: `lib/auth.ts` rewritten · new `backend.ts`, `backend-types.ts`, `stock-data.ts`, `stock-actions.ts`,
  `robot-image.ts` · `components/robot/{image-picker,receive-stock-form}.tsx` · `app/(app)/robots/receive/page.tsx`
  · `rbac.ts`, `accounts-manager.tsx`, `authorized-form.tsx`, `sign-in-form.tsx`, `account/page.tsx`

### Decisions Made
- **Robots enter stock by serial, never by a typed quantity.** A serial is the robot's identity in telemetry,
  monthly reports, CM tickets and the partner API. A count with no serials behind it means deploying one later
  requires inventing a serial *and* decrementing a number by hand — two steps that diverge the first time the
  second is forgotten. The form takes many serials at once; the count is derived and cannot drift.
- **Batch intake is all-or-nothing.** One duplicate rejects the whole paste and names it, because a
  half-applied delivery leaves nothing on screen saying which rows landed.
- **DEMO is warehouse-visible; RENT and SOLD are not.** A demo unit is out on trial but still ours and coming
  back. `RobotUnitStatus.isWarehouseVisible()` encodes it, and RENT/SOLD are **refused** by the stock endpoints —
  both follow from a customer agreement, and setting them from a warehouse screen would record a sale with no
  deployment behind it.
- **Robot photos live on `robots.image_url` — the MODEL, not the unit.** ⚠️ Reversed mid-session at the user's
  direction, and they were right. My first design added `robot_units.image_url`, reasoning that many units have
  no catalogue link and that RIMS should not be able to change the catalogue the AI reads. But **RIMS lists
  robots *from* `robots`** — description (brand, model, type, photo) from the catalogue, count from
  `robot_units` filtered to IN_STOCK + DEMO — so everything it shows has a catalogue row by definition, and a
  per-unit photo would have been one picture copied across every serial of a model. The original V30 was
  deleted unapplied; **V30 now widens `robots.image_url` from `VARCHAR(500)` to `TEXT`** so it can hold an
  uploaded base64 `data:` URI (the [[cm_reports]] precedent — Render's disk is ephemeral). Capped
  **server-side** at ~1 MB; the client downscales to 1200px JPEG with quality stepping.
- **`GET /api/v1/inventory/robots` is the RIMS robot list** — `RobotStockSummary`: catalogue description plus
  `inStock` / `demo` / `total`. The two counts stay **separate**: a demo unit is ours and coming back so the
  warehouse must see it, but it is promised to a trial and is not sellable — summing them is how a robot gets
  sold twice. Models with zero stock are still listed, because "sold but out of" and "we don't sell it" are
  different facts. Counted with two grouped queries and a merge, not a join over 152 rows nor a query per model.
- **The photo is set from the robot's page, not the receive form.** On a delivery form it would offer to
  overwrite the model's existing picture on every single delivery of that model.
- **The PIN is gone.** It was a second factor over a JSON file with no other access control. Writes are now
  guarded by a signed JWT plus a server-side role check — a boundary the browser cannot argue with, which the
  PIN never was. [[AuthorizedForm]] keeps the confirmation dialog, which was doing the real work anyway.
- **`editor` became the Inventory role and gained `stock:write`.** It previously could not change numbers — only
  `admin` could — so a warehouse account would have needed full admin *including* `users:manage`.
- **Role mapping is one-way lossy by design.** ADMIN→admin, INVENTORY_STAFF→editor, RAASPAL_TEAM→viewer;
  collapsing back takes the most privileged, so nothing is silently revoked. The account form now offers a single
  role rather than checkboxes that would collapse on save.
- **`getCurrentUser()` calls `/auth/me` every time** rather than trusting cookie claims: a revoked role or
  disabled account takes effect at once instead of lingering for the rest of a 12-hour token.
- **RIMS never exposes the API to the browser** — all calls are server-side with the JWT in an httpOnly cookie,
  and `RAASPAL_API_URL` is not `NEXT_PUBLIC_`. Stricter than the console, and it needs no CORS entry.

### Bugs found and fixed
- **`@PreAuthorize` was inert** — no `@EnableMethodSecurity` anywhere. The annotations would have parsed and
  never run, leaving stock writes open to every authenticated user including CUSTOMER.
- **`UserController` had no role check at all.** Any staff token could `POST /api/v1/users` with
  `role: ADMIN` — no UI needed, just curl. Now ADMIN-only, listing included.
- **20 partner tests broke** on `robot_type NOT NULL` after the entity change; the fixtures build `RobotUnit`
  without it. Fixed with a `CLEANING` default matching V28's backfill — and it caught a real gap, since
  `RegisterRobotRequest` had no `robotType` either and would have hit the same constraint in production.
- Console `/register` was public (in `PUBLIC_PATHS`) though gated behind `REGISTRATION_ENABLED = false` and
  posting to an endpoint that does not exist. Removed entirely.

### Unresolved / Next Steps
- [ ] ⚠️ **V30 is written but NOT applied.** Same caution as V28/V29: local dev and production share one Supabase
  database, so starting the app applies it to production. (V30 is now
  [[V30__widen_robot_image_url]] — the earlier per-unit version was deleted before it ever ran.)
- [ ] **RIMS robots/inventory list pages still read `lib/seed.ts`.** `stock-data.ts` exists to replace
  `store.ts`, but those pages render RIMS's richer `Robot` shape (lease tiers, per-warehouse counts, marketing
  copy) that the backend does not supply — rewiring means trimming that UI. **Receiving works end-to-end today;
  browsing does not.**
- [ ] `data/users.json` and `lib/seed.ts`/`store.ts` are now dead or dying — delete once the pages are rewired
- [ ] `RAASPAL_API_URL` must be set for RIMS (defaults to `http://localhost:8080`)
- [ ] Create real accounts — `users` still holds one row, so `stock_movements.created_by` is not yet meaningful
- [ ] No tests were added for the inventory or stock-intake endpoints
---

---
## Session: 2026-08-14 — RIMS runs on real data: standalone robot stock table, seed removal, live activity ledger

**Date:** 2026-08-14
**Tags:** #session #frontend #backend #database

### Summary
[[raaspal-rims]] no longer renders a single fabricated figure. The session began with a scope change: robot
stock moves to a **standalone table** with no link to [[robot_units]], [[robots]] or anything else — the
warehouse count is its own record and claims nothing about the deployed fleet. [[V30__add_robot_inventory_temp]]
creates `robot_inventory_temp` with no foreign keys at all, plus a `previous_quantity` / `previous_quantity_at`
pair that captures one step back whenever a count changes, which is the undo path for the commonest warehouse
slip (typing 3 where 30 was meant). Backed by [[RobotStockEntry]], [[RobotStockService]] and
[[RobotStockController]], with writes gated to `ADMIN` and `INVENTORY_STAFF`.

With intake working, every remaining screen was still drawing on `lib/seed.ts`, and the numbers it invented were
being read as real — the sidebar claimed 33 cleaning robots when one had been added. All of it is gone. The
dashboard, robots page, sidebar, command palette, inventory page and activity log now read from the database or
show an empty state. Two seed tiles were dropped rather than rewired, because nothing behind them exists:
robot stock value (no price is recorded) and units reserved (no such concept). The activity log needed a new
backend endpoint — movements existed only per item — so `GET /api/v1/inventory/movements` was added and the
page now renders the real `stock_movements` ledger instead of an invented audit trail.

Three login and query bugs surfaced and were fixed along the way: a case-sensitive email lookup that locked out
an account created with a capital letter, and `lower(bytea)` errors in two repositories from an untyped null
parameter — a pattern [[CvteDeviceRepository]] had already solved with `CAST(:keyword AS string)`.

### Files Modified

**Backend — [[RaasPal-Internal-Ops-backend]]**
- [[V30__add_robot_inventory_temp]] (`db/migration/V30__add_robot_inventory_temp.sql`) — standalone stock table, no FKs
- [[RobotStockEntry]] (`inventory/entity/RobotStockEntry.java`) — entity for `robot_inventory_temp`
- [[RobotStockEntryRepository]] (`inventory/repository/`) — split `search`/`searchByStatus` to kill the `lower(bytea)` crash
- [[RobotStockService]] (`inventory/service/`) — rejects RENT/SOLD, caps images at ~1MB, snapshots the previous count only when it actually changes
- [[RobotStockController]] (`inventory/controller/`) — `/api/v1/inventory/robot-stock` CRUD
- [[InventoryService]] (`inventory/service/`) — new `getRecentMovements()`, item names resolved in one query rather than per row
- [[InventoryController]] (`inventory/controller/`) — new `GET /api/v1/inventory/movements`
- [[InventoryItemRepository]] (`inventory/repository/`) — same latent `lower(bytea)` bug, fixed with `CAST(:keyword AS string)`
- [[UserRepository]] (`user/repository/`) — every lookup now `IgnoreCase`
- [[SecurityConfig]] (`config/`) — `@EnableMethodSecurity`, without which every `@PreAuthorize` was inert

**Frontend — [[raaspal-rims]]**
- [[stock-data]] (`lib/stock-data.ts`) — added `listRecentMovements()`; filled the summary fallback's missing `robotsOnDemo`
- [[backend-types]] (`lib/backend-types.ts`) — `StockMovementResponse`, `MOVEMENT_LABELS`, and shared `ROBOT_TYPES` / `ROBOT_TYPE_LABELS` / `asRobotType()`
- [[app-shell]] (`components/shell/app-shell.tsx`) — 7 invented categories → the 4 real robot types, counted in units held
- [[command-palette]] (`components/shell/command-palette.tsx`) — indexes real entries, links `/robots/{id}`
- `app/(app)/page.tsx` — dashboard rebuilt on `listRobotStock()` + `getInventorySummary()`
- `app/(app)/robots/page.tsx` — honours `?type=`; an unknown value shows everything rather than nothing
- `app/(app)/inventory/page.tsx` — rewritten against `inventory_items`
- `app/(app)/activity/page.tsx` — rewritten against the `stock_movements` ledger
- New: `components/inventory/{parts-table,parts-filters,add-part-form,adjust-stock}.tsx`
- Deleted: `inventory-table`, `inventory-filters`, `catalog-toolbar`, `robot-card`, `robot-table`, `activity-list`, `hero-panel`, `media-panel`, `pricing-panel`, `specs-panel`, `stock-panel`, `robot-image`

### Decisions Made
- **Robot stock is a standalone table.** A plain quantity is honest on `robot_inventory_temp` and was not on
  `robot_units`, where a serial number is the robot's identity across telemetry, reports and the partner API.
  Nothing else in the schema was touched.
- **A tile showing an invented number is worse than no tile** — someone acts on it. Stock value and reserved
  units were deleted rather than filled with zeroes.
- **Counts in the sidebar are units held, not rows.** "Cleaning 14" must mean fourteen machines, not fourteen
  records that might hold one or forty each.
- **Parts change by movement, never by overwriting a total.** "-1, miscount" survives; "set it to 47" throws
  away the reason and races with anyone else counting the same shelf.
- **Schema changes go through Flyway; data fixes may go through the Supabase dashboard.** Renaming
  `robot_specs` in the dashboard broke production once already.

### Unresolved / Next Steps
- [ ] No tests cover the inventory, robot-stock or movements endpoints — 95 pass, none of them these
- [ ] `lib/seed.ts` / `lib/store.ts` / `data/users.json` are now fully unreferenced and can be deleted
- [ ] `lib/catalog.ts` still supplies `WAREHOUSES` and `CATEGORY_META` to the accounts and account pages — seed
  data in a smaller place, not yet real
- [ ] `stock_movements.created_by` is recorded but never displayed; the activity log shows a name only once
  the service resolves it
- [ ] AWS Lightsail deployment (5 files designed, none written) — parked until RIMS is finished
---
---
## Session: 2026-08-16 — Per-company report review with per-robot exclusions

**Date:** 2026-08-16
**Tags:** #session #backend #frontend #database

### Summary
Customer success had no way to review a company's monthly report as a whole before sending. The only
surfaces were a *per-robot* preview and the customer's own public bundle link — so for a customer like
**IFS**, with robots spread across many sites, there was no "see it all, then send" step. Worse, robots
that were offline for the month still render a full page of zeros, which reads as a broken report rather
than an accurate one.

New **Reports → Company report** tab: pick a company and month, see every robot's page stacked exactly as
the customer will, untick the ones with no activity, save, then send. Robots are badged **"N tasks"** or
**"No activity"**, and a one-click *"Drop the N with no activity"* handles the common case.

**Exclusions are persisted server-side, not held in the UI.** The monthly email links to the public bundle
page, so a UI-only filter would have meant staff approving one thing and the customer opening another.
[[CustomerReportBundleService]] `build()` now filters excluded robots, which is the single path both the
customer's link and the delivery email read — they cannot disagree by construction. A separate
`buildPreview()` serves staff and deliberately returns *all* robots, including held-back ones, so they can
be ticked back in.

**Exclusion is a filter, never a deletion** — the user asked explicitly whether un-ticking restores the
robot. It does: a row means "skip this robot", deleting it puts the pages straight back, and telemetry is
never touched. Scoped to (customer, month, robot) so a robot idle in July reports normally in August
without anyone remembering to undo anything.

### Files Modified
- [[V31__add_customer_report_exclusions.sql]] — `customer_report_exclusions`; unique on (customer, month, robot). **Note V28–V30 were taken by inventory/lifecycle work since the last session**
- [[CustomerReportExclusion]] (`report/entity/`) + repository
- [[CustomerReportExclusionService]] (`report/service/`) — replaces the set wholesale; `flush()` between delete and insert or the unique constraint fires within the transaction
- [[CustomerReportBundleService]] — `build()` filters exclusions (customer-facing); new `buildPreview()` returns all robots + `hasData`/`excluded`
- [[CustomerBundlePreviewResponse]] (`report/dto/`) — staff-only shape, kept separate from [[CustomerReportBundleResponse]]
- [[CustomerReportBundleController]] — `GET /customer-bundle/preview`, `PUT /customer-bundle/exclusions`
- `CustomerBundleExclusionTest` — 6 tests; suite now **101, all passing**
- [[CustomerBundlePanel]] (`components/`) — the review + curate + send surface
- [[ReportsClient]] / [[reports/page.tsx]] — new `company` tab in the performance group
- [[lib/api.ts]], [[types/api.ts]] (now imports `MonthlyPerformanceReport`), `messages/en.json`, `messages/th.json`

### Decisions Made
- **Exclusions persist and filter the public bundle:** the preview must equal what is sent, or the review step is theatre.
- **Two DTOs, not one:** robot ids, activity flags and exclusion state are internal curation details with no business on the customer's page.
- **Scoped to the month:** an exclusion that carried forward would silently drop a robot from every future report.
- **Replace the set wholesale rather than patch per robot:** the UI saves a list of tickboxes, so "what is ticked" is the whole intent; patching would make an unticked box ambiguous.
- **Send is blocked while the selection is unsaved** — otherwise the customer receives a different set than the one on screen.
- **`hasData` = `totalTasksCompleted > 0`**, computed in the service rather than added to [[ReportPreviewResponse]], whose javadoc pins it to mirroring the frontend type exactly.

### Verification
- `mvnw clean test` — **101 tests, 0 failures**
- `npm run build` — clean
- **V31 applied to live Supabase**; exclusion round-trip exercised against real data (IFS : One Siam): customer link went 3 → 2 → 3 robots as a robot was excluded and re-ticked. Test exclusion cleared afterwards.
- **PCS : Makro (71 robots)** previewed in ~38s cold; all 71 had July data after the manual sync.

### Unresolved / Next Steps
- [ ] **Deploy — the feature is unusable in production until the backend redeploys**, since V31 and the two new endpoints only exist locally so far.
- [ ] The 71-robot preview takes ~38s on a cold cache. Acceptable, but if CS finds it slow, the per-robot reports are already cached ([[ReportCacheService]]) — a warm-up call after the nightly sync would make the first open instant.
- [ ] No audit of *who* excluded a robot. Fine while the team is small; worth `excluded_by` if it is ever disputed.
- [ ] Exclusions are invisible from the "Manage automation" bulk run — a scheduled monthly delivery honours them silently. Consider surfacing "N robots excluded" in the delivery history.
---
---
## Session: 2026-08-18 — Report email sender changed, standing CC, footer banner

**Date:** 2026-08-18
**Tags:** #session #backend #config #deployment

### Summary
Three changes to the monthly performance report email, plus a debugging session worth recording.

**Sender moved** to a dedicated mailbox (`amitta.s@raaspal.com`) via Render env vars. Nothing in code
changed — `MAIL_USERNAME`/`MAIL_PASSWORD`/`MAIL_FROM` already existed. The App Password must come from
the *same* account named in `MAIL_USERNAME`; changing the username invalidates the old password. Spaces
in the 16-char App Password are fine — Google strips them — contrary to the usual advice.

**`MAIL_CC` added** ([[ReportEmailService]]): a standing CC so the team keeps a copy of every report.
Scoped to report emails only — [[CustomerAnnouncementService]] has its own per-send CC box, and a
standing CC there would silently widen a one-off message. Addresses already on the To line are dropped
case-insensitively so nobody is mailed twice and the customer sees no apparent duplication.

**Footer banner added** to the email, below "Customer Success Team". Embedded as a **CID inline part**,
not a hosted URL: remote images are blocked by default in many clients, and a URL would break if the
frontend moved. Base64 data URIs are not an option — Gmail strips them. The source was resized and
recompressed **519KB → 129KB** (1400x320 → 1120x256, still 2× the 560px body width) because it ships on
every email to every customer. A missing image logs a warning rather than failing the send.

### The debugging worth remembering
A send failed with `Email send failed: Domain contains illegal character (check MAIL_USERNAME /
MAIL_PASSWORD)`. The message pointed at the wrong two variables — the fault was a **single mistyped
character in the domain of `MAIL_FROM`** (`raaspal,com` rather than `raaspal.com`), invisible in a
screenshot.

What narrowed it, for next time:
- The delivery history (`GET /api/v1/reports/delivery/history?month=`) records the real outcome. The
  UI's green "Send started" banner means nothing — [[ReportDeliveryService]] `startSendAsync` returns
  immediately and the result lands in `report_sends`.
- JavaMail's messages are precise: **"Domain contains illegal character"** means an ASCII character from
  `()<>,;:\"[]` in the domain. Anything above ASCII (Thai, fullwidth, zero-width) gives **"contains
  control or whitespace"** instead, and a bad CC gives **"Illegal address"** / **"Missing '<'"**. Those
  three messages discriminate between From, Cc and encoding problems.
- Ruled out by probing candidate strings through `InternetAddress.validate()` in a throwaway test —
  faster than reasoning, and it disproved both the underscore and uppercase theories immediately.

### Files Modified
- [[ReportEmailService]] — `app.mail.cc`, CC dedup against the To line, `splitAddresses` extracted and reused by `recipientsOf`, multipart send, CID footer banner with non-fatal fallback
- [[application.properties]] — `app.mail.cc=${MAIL_CC:}`
- `src/main/resources/email/email-footer-banner.png` — the 129KB optimised banner (committed; must be on the classpath at runtime)
- `.gitignore` — the 520KB original `docs/email_bottom_photo.png` excluded
- `ReportEmailCcTest` — 6 tests; suite now **107**

### Decisions Made
- **CC on reports only, not announcements** — a standing CC on a one-off message would surprise its author.
- **CID inline over hosted URL** — survives client image-blocking and frontend redeploys.
- **Optimise before shipping an email asset** — 520KB × every customer × every month is real bandwidth.
- **A missing banner must never fail a send** — decorative assets don't get to block delivery.

### Unresolved / Next Steps
- [ ] **Improve the send error message** — it names `MAIL_USERNAME / MAIL_PASSWORD` for *any* failure, which actively misled this debugging session. Validate `MAIL_FROM` / `MAIL_CC` at startup and name the offending variable and character, so a malformed address fails at boot rather than silently in a background thread. Offered, not yet built.
- [ ] Confirm on a received email that the **Cc** and **banner** actually render — the history proves SMTP accepted the message, not what it contained.
- [ ] The App Password used during setup was pasted into a chat as a screenshot; **rotate it**.
- [ ] No **Reply-To** is set, so customer replies go to the sending mailbox rather than a shared CS inbox.
---
---
## Session: 2026-08-19 — Reports clip to the customer's contract start date

**Date:** 2026-08-19
**Tags:** #session #backend #frontend #database

### Summary
A customer who signed mid-month was receiving a full-month report. Someone starting 15 July got a
"July report" counting two weeks of work done before they were a customer — real numbers, but not
theirs, and not something to put in front of them.

Customers now carry a **contract start date** (**V32**). [[ReportPreviewService]] clips the task list to
it, so July covers 15–31 July and **August onwards is a normal full month with no further action**.
Null — the default for every existing customer — reports the whole month exactly as before, so this
ships inert until someone fills it in.

Chosen over an ad-hoc from/to on the send screen because the underlying fact is *when the contract
began*, not *what period this particular send covers*: set once, it is correct forever, and critically
the **automated monthly run honours it too**. An ad-hoc range would have left the scheduler still
sending wrong first-month reports unless a human intervened every time.

**The period label deliberately still reads "July 2026"**, not "15–31 July" — the customer knows when
their own contract started, and every later month is a whole month anyway.

### The timezone decision
`reportMonth` is bucketed in **UTC** (`YearMonth.from(startTime.atZone(ZoneOffset.UTC))`), but a contract
start is a **Thai business date**. Clipping in UTC would have dropped real work: cleaning robots run
overnight, and 02:00 Bangkok on the start date is 19:00 UTC the day before. So the date is interpreted
in `app.reports.business-zone` (default `Asia/Bangkok`), and two tests pin both sides of that boundary —
02:00 on the start date is included, 23:00 the night before is not.

### Files Modified
- [[V32__add_customer_contract_start_date.sql]] — nullable `contract_start_date DATE` on `customer_profiles`
- [[CustomerProfile]] / [[CustomerRequest]] / [[CustomerResponse]] / [[CustomerService]] — the new field
- [[ReportPreviewService]] — `clipToContractStart` + `contractStartInstant`, `CustomerProfileRepository` dependency, `app.reports.business-zone`
- [[CustomerService]] — `@CacheEvict(ROBOT_MONTHLY_REPORTS)` on update; correcting a wrong start date must not leave the old report cached
- `ContractStartClippingTest` — 7 tests; suite now **114**
- [[CustomersPanel]] — date input under Branch; empty string normalised to null before sending, or the backend cannot parse it as a LocalDate
- [[types/api.ts]], `messages/en.json`, `messages/th.json`

### Decisions Made
- **Contract start on the customer, not a per-send range** — fixes the cause, and the scheduler inherits it.
- **Interpret the date in Bangkok, not UTC** — overnight cleaning shifts make the 7-hour gap real work, not an edge case.
- **Label keeps the month name** — the customer knows their own start date.
- **Evict the whole report cache on customer update** — edits are rare, the cache is cheap to refill, and a stale first-month report is worse than a cache miss.
- **Null means "no clipping"** — inert for every existing customer; no backfill required.

### Unresolved / Next Steps
- [ ] **Deploy** — V32 and the clipping are local only.
- [ ] **Backfill the real contract start dates.** The feature does nothing until someone fills them in; the customers most affected are the recently onboarded ones.
- [ ] A contract **end** date has the same problem in reverse (a customer who left mid-month). Not requested, but the same mechanism would cover it.
- [ ] The send error message still names `MAIL_USERNAME / MAIL_PASSWORD` for any failure — see 2026-08-18. Still worth fixing.
---
---
## Session: 2026-08-19b — Contract start date corrected: per deployment, not per customer

**Date:** 2026-08-19
**Tags:** #session #backend #frontend #database

### Summary
Same-day correction to the feature above. The contract start date was put on **customer_profiles**, which is
the wrong grain: one customer rents or buys different robots at different times. IFS alone has robots across
many sites with different start dates, so a single date per customer cannot clip their reports correctly.

**V33** adds `contract_start_date` to `deployments` — the robot-to-customer link, which is exactly where
"when did this robot start working for this customer" belongs — and drops the customer column.

### Why V33 rather than fixing V32
V32 had already been **committed, pushed and deployed**, so Flyway had applied it in production. Rewriting an
applied migration breaks checksum validation and the app refuses to start. Verified before writing V33 that
**0 of 67 customers** had a date set, so dropping the column loses nothing — had any been populated they would
have needed copying onto each customer's active deployments first.

### `deployed_at` is not a substitute
`deployments.deployed_at` looks like the right field but is set to `LocalDateTime.now()` at registration, so
every existing value records when a robot was typed into the system, not when its contract began. Reusing it
would have clipped reports to import dates — silently wrong. Hence a separate nullable column.

### Files Modified
- [[V33__move_contract_start_date_to_deployment.sql]] — add to `deployments`, drop from `customer_profiles`
- [[Deployment]] — `contractStartDate`, with javadoc distinguishing it from `deployedAt`
- [[RobotUnitResponse]] `DeploymentInfo`, [[RegisterRobotRequest]], [[UpdateRobotRequest]] — the field
- [[RobotUnitService]] — persisted on register and update; `@CacheEvict` moved here from [[CustomerService]]
- [[ReportPreviewService]] — reads `robot.deployment().contractStartDate()`; the `CustomerProfileRepository`
  lookup added earlier is gone, since the value now arrives on the response it already had
- [[CustomerProfile]] / [[CustomerRequest]] / [[CustomerResponse]] / [[CustomerService]] — reverted
- [[CustomersPanel]] → [[RobotsPanel]] — the date input moved with it, i18n keys moved `customers` → `robotsPanel`
- `ContractStartClippingTest` — 8 tests (suite **115**), including a new one proving two robots for the *same*
  customer clip to their own dates, which is the whole point of the change

### Decisions Made
- **Deployment is the right grain** — the contract is per robot placement, not per account.
- **V33 rather than amending V32** — an applied migration cannot be edited without breaking Flyway.
- **Drop the customer column rather than leave it** — verified empty, and a dead column invites future misuse.
- **New column, not `deployed_at`** — that field's existing values are import timestamps.

### Unresolved / Next Steps
- [ ] **Deploy V33.** Until then production has the V32 column, which nothing reads any more.
- [ ] **Backfill per robot** under Tools → Robots.
- [ ] ⚠️ [[CustomerBundlePanel]] and the CM report were not re-checked against the moved field — they do not
  read it, but worth a glance after deploy.
- [ ] A contract **end** date has the same problem in reverse for a robot withdrawn mid-month.
---
## Session: 2026-08-19c — Report label wording deferred to the August reports

**Date:** 2026-08-19
**Tags:** #session #decision #report

### Summary
Investigated a One Bangkok report (robot `GS101-0100-J5P-1000`, July 2026) showing **Task Completion Rate
0.00%** next to **"Total Tasks Completed: 2"**. Fetched the source data straight from the Gausium API and
confirmed **the report is accurate** — no bug.

Gausium's own rows for July: two tasks (4 and 11 July, both 01:00:05, plan `ALL_รวมทั้งชั้น`), both with
`completionPercentage = 0`, `taskEndStatus = 0`, planned ~1,331 sqm each, actual **0 sqm and 0.28 sqm**.
Every printed figure reconciles: coverage 0.28 ÷ 2,663.14 = 0.01%, operating time 10 + 4 = 14 sec.
`durationSeconds` is time *actually cleaning*, not wall-clock — the robot was out 11 and 15 minutes and
cleaned for seconds. **This is an operations problem at the site, not a reporting one.**

### The null concern was disproved
Earlier sessions flagged `d(null) → 0` and `statusLabel(null) → "Completed"` as latent bugs. Sampled
**1,878 real July tasks across 18 robots**: `taskEndStatus` and `completionPercentage` were populated
**100% of the time**. Both branches are unreachable in practice — deliberately **not** changing them, since
the edit would alter no report and only add the appearance of prudence.

### Decision: defer the wording change to next month
The remaining issue is genuinely just labels:
- **"Total Tasks Completed"** is `reports.size()` — a count of tasks *attempted*
- **"Task Completion Rate"** is % of planned *area* covered, not task success

Proposed rename → **"Total Tasks Run"** and **"Area Completion Rate"** (frontend i18n only; the backend
`Ring("Task Completion Rate")` string is an internal key mapped by `RING_KEYS`, so no backend change).

**User decided 2026-08-19: keep the current wording for the July reports** so they stay consistent with what
customers have already received, and **apply the change for the August reports**.

⚠️ **Deadline: before 2026-09-02.** `app.reports.email-scheduler-cron` is `0 0 8 2 * *` — 08:00 on the 2nd,
sending the previous month — so August's reports go out **2 September 2026**.

### Also found, unactioned
Fleet-wide across those 1,878 July tasks, `taskEndStatus` was: **0 Completed 1,423 (75.8%)**,
**1 Manually terminated 380 (20.2%)**, **2 Abnormal termination 75 (4.0%)**. Roughly **1 in 4 scheduled
cleaning tasks does not end normally**, and nothing surfaces that to anyone today. One Bangkok is not an
isolated case.

### Unresolved / Next Steps
- [ ] **Before 2026-09-02:** rename to "Total Tasks Run" / "Area Completion Rate" in `messages/en.json` and `messages/th.json` (keys `report.totalTasksCompleted`, `report.taskCompletionRate`)
- [ ] **Open question deferred with it:** whether "Total Tasks Completed" should instead *count only* `taskEndStatus == 0`. Fleet-wide that would cut the figure by ~24% versus reports customers already hold — more truthful, but visibly different. Renaming avoids the problem; counting differently confronts it.
- [ ] **Operational, not code:** offered a read-only Gausium sweep ranking robots/customers by July completion rate so CS can look before reports go out. Not taken up yet.

---
## Session: 2026-08-21 — monday.com → AI report → LINE delivery feasibility research

**Date:** 2026-08-21
**Tags:** #session #backend #ai #research

### Summary
Researched a proposed post-MVP feature: read the monday.com **Cleaning Tickets** board plus its comment
threads, have AI write a service report, and deliver it to a customer's LINE chat and the RAASPAL office
LINE group. No code written — this session established what the two external platforms actually permit so
the feature can be scoped. Verdict: the pipeline is buildable end to end, but the LINE side imposes three
hard limits that change its shape, and one open operations question must be answered before any code.
Findings published as an artifact: https://claude.ai/code/artifact/4ed1e53f-e78a-44cd-a6c9-85849b0ee916

**monday.com side — no blockers.** `items_page` returns every visible column; `updates` (nested in `items`,
`limit` max 100) returns `text_body`, `created_at`, `creator`, `assets` and nested `replies` — the comment
threads are fully readable. `create_update` / `change_status_column_value` webhooks exist for real-time.
Caveats: formula columns referencing mirrors (and mirrors referencing formulas) return **null** over the API
and cannot be filtered on; `assets.public_url` **expires after 1 hour**; daily API call cap is **1,000** on
Free/Basic/Standard, 10,000 Pro, 25,000 Enterprise.

**LINE side — three hard limits.** (1) The Messaging API has **no file/document message type** — a PDF cannot
be sent, so a report must be a Flex card + link or a rendered image. (2) A customer can only be reached via a
`userId`/`groupId` **already captured** from a `follow`/`join`/`message` webhook or LINE Login — there is no
lookup by phone, email or company name; bulk follower/member ID endpoints need a **verified or premium** OA.
(3) **No unsend API for bots** — a sent report is permanent, which makes a staff approval gate a design
requirement, not a nicety. Also: billing counts **recipients, not messages** (a 20-person group = 20 messages
per push); TH plans are Free ฿0/300, Basic ฿1,280/15,000, Pro ฿1,780/35,000 (verify at lineforbusiness.com).

**Existing code covers more than expected.** [[ReportLink]] + `GET /api/v1/reports/public/{token}` (whitelisted
in [[SecurityConfig]]) and the console route `app/[locale]/report/[token]` are exactly the tokenised public
page a LINE button needs — no new public-access machinery. [[ReportSend]] is the template for a `line_send`
audit/idempotency table, [[ReportDeliveryScheduler]] for the cadence, `telemetry/adapters/{gausium,autoxing}`
for a new `monday` adapter, and [[ClaudeAiService]] + [[AiPromptRules]] for composition.

### Files Modified
- None — research only. Artifact published; no source changes.

### Decisions Made
- **No decisions committed yet** — six options were laid out for the user (which OA sends, pull vs webhook,
  snapshot vs read-through, Flex+link vs image, approval gate, 1:1 vs group). Recorded here so the constraints
  behind them are not re-researched.
- **`customer_profiles.line_notify_token` is dead scaffolding.** LINE Notify was terminated **31 Mar 2025**;
  the column is unreferenced anywhere in the codebase and should be dropped in whatever migration adds the
  LINE tables. `customer_profiles.line_user_id` also exists and is unused — it is reusable as-is.
- **Branch Name → customer mapping must be human-curated, never AI-inferred.** The board identifies customers
  as Thai free text (`Makro สุรินทร์`, `นครราชสีมา 3 (เซฟวัน)`); a wrong match sends one customer another's
  service history. An unmapped branch must block the send.

### Unresolved / Next Steps
- [ ] **Blocking question:** does RAASPAL already have a LINE OA sitting in customer group chats? **Only one
      Official Account can be in a group chat at a time** — if so, a new dedicated reporting OA *cannot join
      those groups*, and the feature must be built on the existing OA's channel or be 1:1 only.
- [ ] Is the OA **verified/premium**? Gates the bulk follower/group-member ID endpoints.
- [ ] Which Cleaning Tickets columns are formula/mirror? Those return null and need a plain-column copy on the board.
- [ ] Which monday plan (daily API call ceiling)? Report language (Thai/EN/both)? Do comment photos need to appear?
- [ ] Customer/group counts, for the LINE message-cost model.
- [ ] ⚠️ If built: the new scheduler must sit behind the same off-switch as [[ReportDeliveryScheduler]] during the
      Render→Lightsail parallel run — a duplicated LINE push **cannot be recalled**. Enforce idempotency in the DB.
---

---
## Session: 2026-08-26 — monday.com adapter for the Daily Pending Case Report

**Date:** 2026-08-26
**Tags:** #session #backend #config

### Summary
Built the read side of the **Daily Pending Case Report** feature: a monday.com GraphQL adapter under a new
[[casereport]] module. Board discovery is complete for both source boards and the design is locked; this
session produced the client, DTOs, pager and a temporary preview endpoint. **264 sources compile, 115 tests
pass.** Not yet wired to a scheduler, snapshot tables, AI, Excel or LINE.

**Boards and groups (verified by API).**
- Cleaning Tickets `3451717331`, group **All Case** `new_group96592__1` (46 items, one page, `cursor: null`).
- Delivery Tickets `1647612496`, group **All Case** `group_title`.
- Reports read All Case **only**. Other groups (`Done Check`, `AOTGA`, `Makro Project`, `Tickets Done 2023`)
  are archives *and* customer buckets mixed on one axis, so group membership cannot drive logic.

**⚠️ Column ids collide across boards with different meanings.** `text` = Main Issue on Cleaning but
**Solution** on Delivery; `status_1` = Issue Level on Cleaning but **Sup Status** on Delivery. Column mapping
must therefore be stored per report, never hardcoded.

**Dead columns on Cleaning Tickets** (empty/null on every one of the 46 rows): `date_mm3b365t` วันส่งอะไหล่,
`dropdown_mknqq9fm` Spare Parts Name, `text_mksf2s9c` QTY. So Part Received / Required Part exist **only** in
the comment threads. `formula_mkyqm73w` Aging (Days) returns null over the API as expected — computed from
`date8` instead.

**Findings that shrink the AI's job.**
- `updates[0]` is the **newest** comment (verified across tickets).
- The `*...*` status marker in comments is **one person's habit** (every instance authored by "Boss"): present
  in the latest comment on only **3/12** tickets, and somewhere in the last three on **8/12**. A hint, not a rule
  — so a staff review step is mandatory.
- **The Delivery "Solution" column is a status-change history**, not prose: each line is `{date} {status value}`.
  Confirmed on item `12874545928` (`status` = อยู่ระหว่างจัดส่งอะไหล่ ↔ report line `24-Aug อยู่ระหว่างจัดส่งอะไหล่`).
  **The daily sync can build this log itself from consecutive snapshots — no AI needed for that column.**
- Project/Branch are reliably available in the contact-center's structured first comment
  (`ติดต่อจาก` = Project, `สาขา` = Branch), which beats the typo-ridden `text6`/`asset_owner3__1` columns.

### Files Modified
- [[MondayApiClient]] (`casereport/adapters/monday/`) — GraphQL client; **treats a 200 with an `errors` array as a failure**, surfaces `extensions.request_id`
- [[MondayApiException]] (`casereport/adapters/monday/`) — new
- [[MondayBoardReader]] (`casereport/adapters/monday/`) — cursor pager; empty `boards` array → explicit "not visible to this token" error
- [[MondayItem]], [[MondayItemPage]], [[MondayUpdate]], [[MondayColumnValue]], [[MondayGroup]], [[MondayCreator]] (`casereport/adapters/monday/dto/`) — new records
- [[MondayPreviewController]] (`casereport/controller/`) — **temporary** `GET /api/v1/case-reports/monday/preview`
- [[application.properties]] — `app.monday.api.{base-url,token,version,page-size,updates-per-item}`

### Decisions Made
- **One generator class per report, shared plumbing.** User's call, and it mirrors the existing
  [[TelemetryAdapter]]/[[TelemetryAdapterRegistry]] precedent. Client, Excel writer, LINE sender and scheduler
  stay shared; `AotPendingReportGenerator` / `MkPendingReportGenerator` / `AllPendingReportGenerator` own their
  own filters and column mapping so a change to one cannot break another.
- **Report scope, not customer.** `case_report_definition` + `case_report_column` + `case_report_filter` +
  `case_report_recipient` + `case_report_run` replace the earlier `monday_customer` idea. Adding a report is an
  INSERT. Yayoi deferred on that basis.
- **SLA is computed, not stored.** `On Hold` in `status` **or** `status_1` → 🟡; else `days >= sla_days` → 🔴;
  else 🟢. `sla_days` is per-definition config — **the real threshold is still unconfirmed, assumed 14**.
- **API version pinned to `2026-07`** (current stable). Unpinned requests silently roll forward each quarter.
- ⚠️ **`mvn` is not on PATH — use `.\mvnw.cmd`.** The `CLAUDE.md` build commands need updating.

### Unresolved / Next Steps
- [ ] **Ask the team the real SLA threshold in days**, and whether AOT and MK differ.
- [ ] Flyway **V34** (latest is V33): snapshot + report-definition tables.
- [ ] Excel generation — `poi-ooxml` is **already** in `pom.xml`, no new dependency.
- [ ] ⛔ **LINE cannot send `.xlsx`.** Today staff attach it manually from the LINE app; a bot cannot. Delivery
      becomes text + the pivot image (both bot-supported) + a link to the file.
- [ ] Branch/airport normalisation table — `ท่าอาศยาน` and `Marko` typos are live on the board today.
- [ ] Split multi-serial tickets into one row each; separators seen are both `/` and `และ`.
- [ ] Strip the `#` prefix from `asset_owner`; normalise `Pudu 1` → `PUDU1`.
- [ ] Remove [[MondayPreviewController]] once the console screen exists.
---
---
## Session: 2026-08-28 — Spare parts linked to the robots they fit (RIMS)

**Date:** 2026-08-28
**Tags:** #session #backend #frontend #database #rims

### Summary
RIMS could hold parts and could hold robots, but nothing connected them. Now a robot's
detail page lists the parts that fit it, each part has its own page with part number,
count and movement history, and Inventory/Admin accounts link the two.

**The relationship is many-to-many** — the user's requirement: "some spare parts are
usable for multiple robots". V28's single `inventory_items.robot_id` could not express
that, and its own comment had already flagged the join table as the known next step.
**V35** adds `inventory_item_robots` and drops the old column.

### The measurement that decided the design
The obvious target for the FK was the `robots` catalogue. Measured first, against
production: **only 14 of the 92 warehouse robots (15%) exist in the catalogue.** The
other 78 — Autoxing, AIRROBO, CVTE, most Gausium variants — are models RAASPAL stocks
but will never put in a customer proposal. Linking parts to the catalogue would have
blocked the whole feature behind creating 78 spec-heavy catalogue entries nobody needs.

So the join table references **`robot_inventory_temp`** (what the warehouse actually
stocks). Three tables hold robot data and they do not line up — worth remembering:

| Table | Rows | What it is |
|---|---|---|
| `robots` | 19 | catalogue of models — specs, pricing; feeds the AI proposal engine |
| `robot_units` | 165 | individual physical robots, serial numbers, deployments, telemetry |
| `robot_inventory_temp` | 92 | RIMS warehouse stock — free-text brand/model + a count |

`inventory_items` was verified **empty in production twice** (before writing the
migration and again before running it), so `DROP COLUMN robot_id` loses nothing.

### Files Modified
- [[V35__link_inventory_items_to_robot_stock.sql]] — join table + drop the old column
- [[InventoryItem]] — `robotStockIds` as an `@ElementCollection`; `@Fetch(SUBSELECT)` is
  load-bearing, not tuning: without it a 200-row parts list is 200 extra queries against
  a pooler that allows 15 connections
- [[InventoryItemRepository]] — `MEMBER OF` filter, so "parts for this robot" is the same
  query as everything else
- [[InventoryItemRequest]] / [[InventoryItemResponse]] — `robotStockIds` in, `robots[]` out
- [[InventoryService]] — set replaced wholesale on save; unknown ids rejected naming the id;
  robot names resolved in one query per page
- [[RobotStockEntry]] — `displayName()` moved onto the entity so parts and robots assemble
  it identically
- [[GlobalExceptionHandler]] — **`AccessDeniedException` → 403.** It previously fell through
  to the `Exception` catch-all, so every `@PreAuthorize` denial returned **500**. Found by
  the view-only test; affects all inventory writes, not just this feature.
- RIMS: [[related-parts]], [[robot-links-field]], [[edit-part-form]], new `/inventory/[id]`
  page, robot detail page, parts table, dashboard low-stock rows, `stock-data`, `stock-actions`
- `InventoryItemRobotLinkTest` — 6 tests; suite now **121**

### Decisions Made
- **Join table, not a second FK** — the requirement is genuinely many-to-many.
- **Target `robot_inventory_temp`, not `robots`** — the 15% measurement; parts belong to what
  the warehouse stocks, not to what sales can propose.
- **Replace the link set wholesale on save** — the form submits what is ticked, so an untick
  must remove; a patch would make it ambiguous.
- **Link from the part, not the robot** — attach "this brush fits these six robots" once,
  rather than visiting six robot pages.
- **Checkbox group over multi-select** — the answer is several robots, and a multi-select hides
  what is already ticked. The filter never hides a ticked robot, or narrowing the search would
  appear to silently unlink one and submitting would then do exactly that.
- **No rename of `robot_inventory_temp`** — considered and dropped. `ALTER TABLE RENAME` is
  metadata-only and safe, but the user was rightly cautious about live data and the rename is
  cosmetic. Revisit when nothing is urgent.

### Unresolved / Next Steps
- [ ] ⚠️ **V35 is NOT applied yet.** An untracked `V34__add_case_report_tables.sql` (439 lines,
  nine tables, the monday.com case-report feature) is sitting in the migration folder. Flyway
  refuses to start with two files claiming v34, so this migration was renumbered to **V35** —
  but starting the app now would apply that V34 first, and it is someone's work in progress
  that was deliberately **not** applied to the shared production database. Decide what happens
  to it before deploying either.
- [ ] The feature is verified by tests only (121 pass, 6 covering it end-to-end through MockMvc)
  and a clean RIMS build. Not yet exercised against production data, for the reason above.
- [ ] Parts are linked to warehouse *models*, so every unit of a model shares one part list.
  Correct today; if a specific serial ever needs its own parts, that is a different link.

---
## Session: 2026-08-28 — Robot lifecycle status and packaging status

**Date:** 2026-08-28
**Tags:** #session #backend #frontend #database

### Summary
P'Pom asked, via the CS team, for the warehouse record to carry two things it could not
express: where a robot sits in its lifecycle beyond "held or on demo", and whether it is
still in its carton. Robot Status becomes four values — New Stock, Demo Unit, Under Repair,
Returned from Customer — and Packaging Status (Box / Unbox) is added as a separate field.

The important property of this change is that **no existing row was rewritten**. `IN_STOCK`
and `DEMO` already meant exactly what P'Pom called New Stock and Demo Unit, so they keep their
stored values and gain labels; only the two genuinely new states are new data. That matters
because local development and production share one Supabase database, and [[V34]] taught us
what a non-additive migration does to a running deployment.

The two new states were added to [[RobotUnitStatus]], which is shared with the fleet domain.
Rather than widen `isWarehouseVisible()` — which guards [[RobotUnitService]] in five places and
would have let a `robot_units` row take a store-room-only state — a second predicate
`isStockRoomStatus()` was added, and only [[RobotStockService]] consults it.

### Files Modified
- [[V37__add_robot_stock_lifecycle]] (`RaasPal-Internal-Ops-backend/src/main/resources/db/migration/V37__add_robot_stock_lifecycle.sql`) — **new.** Widens `status` to `VARCHAR(32)` (`RETURNED_FROM_CUSTOMER` is 22 chars and the column held 20) and adds a nullable `packaging VARCHAR(16)`
- [[RobotUnitStatus]] (`.../robotunit/entity/RobotUnitStatus.java`) — `UNDER_REPAIR`, `RETURNED_FROM_CUSTOMER`, and `isStockRoomStatus()`; appended to the enum so no ordinal shifts
- [[Packaging]] (`.../inventory/entity/Packaging.java`) — **new.** `BOX` | `UNBOX`
- [[RobotStockEntry]] (`.../inventory/entity/RobotStockEntry.java`) — `packaging` field, status widened to 32
- [[RobotStockEntryRequest]] / [[RobotStockEntryResponse]] — carry `packaging`
- [[RobotStockService]] (`.../inventory/service/RobotStockService.java`) — `warehouseStatus()` → `stockRoomStatus()`, now checking `isStockRoomStatus()`; persists packaging on create and update
- [[backend-types]] (`raaspal-rims/lib/backend-types.ts`) — `STATUS_LABELS`, `STATUS_HINTS`, `PACKAGING_LABELS`, `asStockRoomStatus()`, `asPackaging()`
- [[stock-actions]] (`raaspal-rims/lib/stock-actions.ts`) — **bug fixed in passing:** status was read as `=== "DEMO" ? "DEMO" : "IN_STOCK"`, which would have silently flattened both new states to New Stock
- [[badge]] (`raaspal-rims/components/ui/badge.tsx`) — `StatusChip`, `PackagingChip`
- [[robot-stock-form]], [[robot-card]], [[robot-summary]], [[robots/page]] — pickers, chips, and one band per state

### Decisions Made
- **Packaging is not part of the identity index** — the user's explicit call. It stays outside `uq_robot_inventory_temp_identity`, so one value covers the whole quantity on a row and a shelf of four holding two boxed and two unboxed cannot be told apart. The alternative (packaging joins the key, rows split like they already do per status) was offered and declined. Revisit if the warehouse hits the limit.
- **A second predicate, not a wider one** — `isStockRoomStatus()` sits alongside `isWarehouseVisible()` rather than replacing it, because the fleet and the store room stopped asking the same question the moment repairs became a status.
- **Labels, not values** — `IN_STOCK`/`DEMO` keep their stored strings. Renaming them in the database would have been a non-additive change to a shared production table for a purely cosmetic gain.
- **Null packaging is a real answer** — every row predating V37 has none, and defaulting them to `BOX` would invent a fact about a shelf nobody has checked. It renders as "Not recorded".

### Unresolved / Next Steps
- [x] ~~V37 is written but NOT applied.~~ **Wrong when written.** The user applied V37 and deployed the
  backend the same day, while the packaging design was still being discussed, and said so afterwards.
  ⚠️ By then the migration file had been rewritten locally to a `boxed_quantity` count. Flyway
  checksums every applied migration and refuses to start on a mismatch, so that rewrite would have
  broken the next deploy with no local symptom. Restored from `efa58fc` with `git checkout`. The rule
  it cost: once a `V*.sql` is committed, treat it as already applied and add a new migration instead.
- [ ] **The dashboard does not count the new states.** `InventorySummaryResponse` sums `IN_STOCK` and `DEMO` only, so a robot moved to Under Repair drops out of both tiles. It is still in the catalogue and the sidebar counts. Decide whether the dashboard needs a third tile or a "not sellable" total.
- [ ] Verified by `mvnw test` (127 pass) and a clean RIMS build with `tsc --noEmit` at exit 0. **Not yet exercised against real data**, because that needs V37 applied.
- [ ] [[V34__add_case_report_tables]] is still parked as `.txt` and will need renumbering to **V38** or later now that V37 is taken.
---

---
## Session: 2026-08-31 — Weekly performance report (preview only)

**Date:** 2026-08-31
**Tags:** #session #backend #frontend

### Summary
RAASPAL sends customers a **monthly** Gausium performance report; the team asked for a **weekly**
one as well, added as a period filter on the existing preview page rather than as a second report.
The report body is unchanged — same aggregation, same [[MonthlyReportView]], same figures — only
the window it covers moved, so a week's numbers reconcile with the month's by construction.

The month path was left exactly as it was: it reads the stored, indexed `report_month` column.
A week could not reuse it, because a week routinely straddles two months (2026-W36 is 31 August
– 6 September), so the week path is a `start_time` range instead. That range is anchored in the
**business timezone** (`app.reports.business-zone`, Asia/Bangkok) rather than UTC — the same choice
[[PartnerDataService]] already made for its day-range filter, and for the same reason the contract-start
clipping documents: cleaning robots run overnight, so a task at 23:30 Bangkok on the final Sunday is
this week's work even though it is Monday in UTC.

Weeks are **ISO weeks (Mon–Sun)**, chosen by the user from three options. The deciding factor was that
`<input type="week">` emits `YYYY-Www` natively, so the picker, the request parameter and the report
label all speak one string with nothing to translate between them, and no custom date picker is needed.

**Scope was explicitly limited to the preview page.** Sharing and emailing stay monthly-only:
`report_links.report_month` is `VARCHAR(7)` and [[ReportLinkService]] keys one token per robot+month,
so a weekly report has nowhere to be sent yet. The two buttons are **disabled with an explanation**
in weekly mode rather than hidden — a disabled button with a reason teaches, a missing one confuses —
and there is a visible amber note as well as the tooltip, because a tooltip alone is not an answer.

`mvnw test` passes at **134** (was 127; 7 new). `npm run build` is clean. The ISO-week helpers were
additionally checked against 16 known dates, including both year boundaries and the 52-vs-53-week
case, and the TypeScript labels come out byte-identical to the Java ones.

### Files Modified
- [[ReportPeriod]] (`RaasPal-Internal-Ops-backend/.../report/service/ReportPeriod.java`) — **new.** The window one report covers. `ofMonth` is deliberately *tolerant* (an unparseable month matches no stored `report_month`, exactly as before); `ofWeek` is deliberately *strict* and 400s, because a week is resolved into a range and a typo would otherwise return a confident report for a window nobody asked for. Also builds the period label
- [[ReportPreviewService]] (`.../report/service/ReportPreviewService.java`) — `build(sn, month)` kept as-is for its existing callers ([[ReportCacheService]], [[ReportLinkService]], [[ReportEmailService]], [[CustomerReportBundleService]]); new `buildForWeek(sn, week)`; both delegate to one private `build(sn, ReportPeriod)`. `clipToContractStart` generalized from a month to any period start
- [[RobotTaskReportRepository]] (`.../telemetry/repository/RobotTaskReportRepository.java`) — non-paged `findByRobotUnitIdAndStartTimeBetween` for the week window (the paged sibling already existed for the partner API)
- [[ReportPreviewController]] (`.../report/controller/ReportPreviewController.java`) — `month` and `week` both optional, **exactly one** required; both together is a 400 rather than a silent winner
- [[WeeklyReportPeriodTest]] (`RaasPal-Internal-Ops-backend/src/test/.../report/WeeklyReportPeriodTest.java`) — **new, 7 tests.** Mon–Sun inclusivity, the Bangkok-vs-UTC Sunday-night boundary, a week spanning two months, both label forms, an empty week, malformed weeks, and the monthly path unchanged
- [[report-week]] (`robot-recommendation-web-raaspal/lib/report-week.ts`) — **new.** `isoWeekOf`, `previousIsoWeek`, `isoWeekRange`, `weekRangeLabel`; the weekly counterpart to [[report-month]]
- [[ReportPreviewPanel]] (`robot-recommendation-web-raaspal/components/ReportPreviewPanel.tsx`) — Monthly/Weekly toggle, `<input type="week">`, week-aware Gausium sync range and sample-data label, share/email disabled in weekly mode with a note
- [[api]] (`robot-recommendation-web-raaspal/lib/api.ts`) — `reportApi.preview(sn, { month } | { week })`
- [[en.json]] / [[th.json]] (`robot-recommendation-web-raaspal/messages/`) — `recNoData` said "choose a month"; now "a month or week"

### Decisions Made
- **A period filter, not a second report.** The weekly report is the monthly report over a shorter window. Nothing about the layout, the metrics or the wording is weekly-specific, so there is one `MonthlyPerformanceReport` shape, one view, and one aggregation to keep correct.
- **Month keeps its column; week gets a range.** Reusing `report_month` for weeks was impossible (weeks straddle months) and rewriting months as ranges was unnecessary risk to a live path — so the two load differently and share everything after.
- **The week window is Bangkok, not UTC.** Follows the precedent [[PartnerDataService]] set and the reason the contract-start clipping already gives. ⚠️ Note the pre-existing inconsistency this sits beside: `report_month` and the `activeDays` count are both bucketed in **UTC**. Left alone deliberately — changing it would move every historical monthly figure — but it means a task just after midnight Bangkok can land in a different month than the week that contains it. Worth revisiting as its own change.
- **Strict weeks, tolerant months.** Asymmetric on purpose, explained above. Matches the strict-`month` hardening already done on the partner API.
- **Sharing/email disabled, not hidden.** Scoped out this phase, and a weekly link would need `report_month` widened (or a period column) plus a migration.

### Unresolved / Next Steps
- [ ] **Weekly sending is not built.** Needs a period-aware [[ReportLink]] (widen `report_month` to hold `YYYY-Www`, or add a period-type column — either way a migration, **V38 or later**), then [[ReportEmailService]] and the public `/report/{token}` page.
- [ ] **`WEEKLY` cadence is still inert.** `deployments.report_cadence` has accepted `WEEKLY` since `V16`, but [[ReportDeliveryScheduler]] only sends MONTHLY. A weekly automated run also needs `report_sends` idempotency keyed per week, not per month.
- [ ] **The customer bundle is monthly-only.** [[CustomerReportBundleService]] and `/report/customer/[token]` were untouched; a weekly bundle is the natural companion to weekly sending.
- [ ] The report title is period-neutral ("Executive Robot Performance Report : 17 – 23 August 2026"), so nothing there needs changing if weekly is later sent to customers.
---

---
## Session: 2026-08-31 — CM report fits one page, measured rather than guessed

**Date:** 2026-08-31
**Tags:** #session #frontend

### Summary
A printed [[CorrectiveMaintenanceReportView]] was coming out as three pages with the first one
two-thirds empty. Three separate faults, found in that order.

**The blank page.** Every table row carried `print:break-inside-avoid`. That is right for short rows —
it stops a Thai label stranding on one sheet with its value on the next — but the corrective-detail row
is taller than a sheet, and a row that cannot be kept whole gets pushed to a fresh page and then
overflows anyway. The two prose rows now opt out and flow; their labels align to the top, since a
centred label in a page-spanning cell prints halfway down the following sheet.

**Text running off the paper.** A `1fr` grid track will not shrink below its content's min-content
width, so a single unbroken run widened the column past the page edge. Needed `min-w-0` *and*
`overflow-wrap: anywhere` — neither is sufficient alone. Not merely a guard against pasted junk:
**Thai is written without spaces between words**, so real paragraphs offer few break opportunities.

**Still two pages.** A character-count heuristic was tried first and was the wrong instrument — height
comes from where text wraps, and Thai wraps nothing like the Latin text such a guess is calibrated
against. Replaced with a binary search over the font size in a layout effect: the largest size at which
the sheet still fits one A4 page, settled before first paint. The estimate survives as the first-paint
value so the page does not visibly resize under the operator.

### Files Modified
- [[CorrectiveMaintenanceReportView]] (`robot-recommendation-web-raaspal/components/report/CorrectiveMaintenanceReportView.tsx`) — `splittable` rows, wrap fix, measured fit-to-page, screen/print geometry unified

### Decisions Made
- **Screen and print geometry are now identical** — same padding, same type size, no `print:text-*` step-down. A measurement taken against different print padding would describe a page nobody prints. Side benefit: the preview became genuinely WYSIWYG.
- **Re-measure after `document.fonts.ready`** — the Thai faces have different metrics from the fallback, so a measurement taken before they load is about the wrong page.
- **`useLayoutEffect`, not `useEffect`** — an operator who hits Print immediately must not get the pre-measurement size.
- **8px floor.** Past that the report stops being something a customer can read and sign, so an enormous ticket takes a second sheet instead. A readable two-page report beats an unreadable one-page report.

### Unresolved / Next Steps
- [ ] Not yet verified on paper. The Nikon ticket (~3,000 characters) should land around 9–10px; if that reads too small, raising `MIN_BODY_PX` is a one-line change that trades the single sheet away.
- [ ] `minHeight="240px"` on the corrective row does not participate in the fitting pass. Harmless today (long reports exceed it anyway), but it is dead weight on a short report that is close to the page limit.
- [ ] [[COMPANY_FOOTER]] is still transcribed from a scan — พอล vs พาล unverified against the company registration.
---

---
## Session: 2026-08-31 — Packaging follow-through and a photo-picker regression

**Date:** 2026-08-31
**Tags:** #session #frontend #database

### Summary
Follow-on from the lifecycle-status work, and mostly a record of two mistakes and what they cost.

**Packaging stays an enum.** A `1 Box · 2 Unbox` split was designed and half-built as a `boxed_quantity`
count, then reverted: V37 was already applied and deployed, and the count needed a different column.
The display was taken as far as the deployed schema allows — the chip leads with the row's count
(`3 Box`) — but one row still holds one packaging value for the whole shelf. A genuine mixed shelf
needs **V38** adding `boxed_quantity INTEGER`, which is additive and safe against the running backend.

**The photo picker was breaking every edit.** [[RobotImagePicker]] seeded its state from `initialValue`
and posted it back. When photos moved out of the list payload, `initialValue` became a URL for the
image endpoint rather than the image — so `validateImage` rejected it and *every* edit to a robot with
a photo failed, whatever field the operator was changing. It only ever worked when a new photo was
chosen, which is exactly why the earlier photo-change test passed. Now tracks the choice: untouched
renders no input at all, `""` clears, a data URI replaces. [[stock-actions]] had to change with it —
it coerced an absent field to `""`, which the server reads as "delete the photo".

**Half a day was lost to a stale local backend.** `raaspal-rims/.env.local` pointed at
`http://localhost:8080`, where a JVM from earlier in the session was still running pre-V37 code. Spring
accepted the `packaging` field, dropped it silently as unknown, and returned a row without it — so a
save looked successful and the value never appeared. Repointed at the deployed API.

### Files Modified
- [[image-picker]] (`raaspal-rims/components/robot/image-picker.tsx`) — three states: untouched, cleared, chosen
- [[stock-actions]] (`raaspal-rims/lib/stock-actions.ts`) — forwards `imageUrl` only when present; status narrowed against the real list rather than tested for `DEMO`
- [[backend-types]], [[badge]], [[robot-card]], [[robot-summary]], [[robot-stock-form]], [[robots/page]] — four status bands, `StatusChip`, `PackagingChip`
- `raaspal-rims/.env.local` — repointed at the deployed API (gitignored, so local only)

### Decisions Made
- **Absent, empty and set are three different instructions.** Both bugs above are the same mistake: treating "no value" and "empty value" as equivalent when the server reads them as opposites.
- **Reverted rather than migrated forward.** The count design was better, but V37 was deployed; changing an applied migration is not an option and a second migration was not warranted mid-discussion.

### Unresolved / Next Steps
- [ ] **V38 for `boxed_quantity`** if the warehouse hits a genuinely mixed shelf. Code for it was written and reverted; restoring it is small.
- [ ] The duplicate-entry error says `DEMO`, not `Demo Unit` — raw enum names leaking into operator-facing messages now that labels exist.
- [ ] RIMS frontend work is **committed but not deployed**; the deployed RIMS has none of the status or packaging UI.
- [ ] A stale backend was left running on port 8080 against the shared Supabase database. Nothing points at it, but it is live and writable.
---

---
## Session: 2026-09-08 — RE KPI dashboard: monday.com case-ticket sync (backend)

**Date:** 2026-09-08
**Tags:** #session #backend #database #kpi #monday

### Summary
First backend slice of the **RE Team KPI dashboard** (plan: `RaasPal-Ops-frontend/docs/re-kpi-dashboard-plan.md`,
frontend scaffold on `feat/re-kpi-dashboard`). The user's decision on the blocking data-source question:
**monday.com first, Excel later.** Built on a new backend branch of the same name, `feat/re-kpi-dashboard`,
committed locally, **not pushed**.

The Cleaning Tickets (`3451717331`) and Delivery Tickets (`1647612496`) boards are now mirrored into a
`case_ticket` table by a sync, and the CM-case third of the deck — Total CM Cases, SLA within/over, First
Time Fix / repeat — is computed from it at `GET /api/v1/kpi/cm-cases?from=YYYY-MM&to=YYYY-MM`, per month and
per service line, in the `YYYY-MM` period shape the frontend's `lib/kpi/period.ts` already uses.

**What could not be verified here.** No `MONDAY_API_TOKEN` exists on this machine, so the column mapping is
config, not code, and only the ids the 2026-08-26 discovery confirmed are marked verified in
`application.properties`; the rest are marked assumed. `GET /api/v1/kpi/monday/boards/{id}` lists a board's
real column ids/titles/types so the mapping can be corrected via env
(`APP_KPI_MONDAY_BOARDS_0_COLUMNS_CLOSEDATE=…`) without a deploy. Every column is archived verbatim in
`case_ticket.raw_columns`, so a mapping fixed later applies on the next sync.

**Definitions are provisional and say so.** SLA = calendar days open→close vs the board's limit (Cleaning 3;
Delivery 3 metro / 5 upcountry, reusing `app.casereport.metro-provinces`); exactly the limit is within; blank
province takes the upcountry limit; open tickets and closed-without-date are "unknown", never "within".
Repeat = another ticket on the same service line names one of this ticket's serials and opens within
`repeat-window-days` (7) after this one closes. The response carries `provisional: true` and a `definitions`
map so a board number can be traced to its rule. Sign-off against "Case_FTFR_SLA Jan–Jun 2026" still needed.

### Files Modified
- `db/migration/V38__add_case_ticket_sync.sql` — **new, pending, additive.** `case_ticket` (table 1 of the parked V34 design + service_line, ticket_no, issue_level, close_date, is_closed, serials_normalised) and `case_ticket_sync_run`. ⚠️ Applies to the shared prod DB the first time anyone starts the backend on this branch.
- `kpi/config/KpiMondayProperties` — `app.kpi.monday.*`: per-board column mapping, group ids (empty = every group), closed statuses, SLA days; fails startup on a half-configured board
- `kpi/service/CaseTicketMapper` — row → entity; date parsing; serial normalisation (`#`, `/`, `และ`, `Pudu 1` → `PUDU1`); raw-column archive
- `kpi/service/CaseTicketWriter` — transactional merge; `updated` counted only when monday's `updated_at` moved; unseen rows marked `is_present=false`, never deleted, and only on a complete read
- `kpi/service/MondayCaseSyncService` — per-board runs with audit rows; one lock for scheduled and manual runs; a failing board does not stop the other; background `start()` for the console
- `kpi/service/KpiCaseMetricsService` — the aggregation above; ≤24-month range
- `kpi/scheduler/MondayCaseSyncScheduler` — `KPI_MONDAY_SYNC_ENABLED` off-switch, 01:30 Bangkok
- `kpi/controller/KpiController` — `/api/v1/kpi/*`, class-level `@PreAuthorize` ADMIN / RAASPAL_TEAM
- [[MondayBoardReader]] — `describeBoard()`; `readGroup(..., includeUpdates)` with `@include(if:)` so the KPI sync skips comments; empty column list now means **every** column (was none); returns `MondayGroupRead(items, complete)`
- [[MondayColumnValue]] — gains `column { id title }`; new `MondayColumnRef`, `MondayColumnDef`, `MondayBoardSchema`, `MondayGroupRead`
- [[GlobalExceptionHandler]] — `MondayApiException` → **502**, was a bare 500
- `application.properties`, `README.md` — config block, endpoints, env vars
- Tests: `kpi/CaseTicketMapperTest`, `KpiCaseMetricsServiceTest`, `MondayCaseSyncServiceTest` (mocked reader, real H2), `KpiApiSecurityTest`, `MondayCaseSyncSchedulerTest` — 35 new, all pass; full suite green

### Decisions Made
- **Reuse the V34 `case_ticket` shape rather than invent a KPI table.** Same two boards, same rows; the Daily Pending Case Report can build on this mirror as V39+ instead of syncing twice. The other eight V34 tables stay parked.
- **Read every group by default, not just All Case.** Closed tickets move between groups and the KPI needs history; the daily report's All-Case-only rule stays a per-feature choice. Consequence: `is_present=false` now means deleted/archived on monday, and those are excluded from the KPI.
- **`raw_columns` is TEXT, not JSONB.** Nothing queries into it yet, and TEXT is what H2, the JPA mapping and Postgres agree on with no custom type; `ALTER … TYPE jsonb USING raw_columns::jsonb` later is cheap.
- **Absent serial = first-time fix, reported.** A repeat cannot be detected without a serial; counting such tickets as failures would punish a blank cell. `withoutSerial` is in every bucket so the reader knows how soft the rate is.
- **`mvn` is not on PATH and `./mvnw` is not executable on this Mac — use `sh mvnw`.**

### Unresolved / Next Steps
- [ ] **Set `MONDAY_API_TOKEN` and confirm the column mapping** via `GET /api/v1/kpi/monday/boards/3451717331` and `…/1647612496`, especially close-date columns (without one, SLA is all "unknown") and the Delivery serial column.
- [ ] **Confirm the KPI definitions with the RE team** against "Case_FTFR_SLA Jan–Jun 2026" — the deck's 1,270 "KPI cases" out of 1,458 implies an exclusion (issue level?) this module does not apply.
- [ ] Frontend: add a `kpiApi` group to `lib/api.ts` and map `KpiCaseMetricsResponse` onto the `KpiReport` panels for Total CM Cases, SLA and FTFR; the other three KPIs (1st Time Install, PM Complete, CSAT) have no source yet — spreadsheets, per the deck.
- [ ] Update `docs/re-kpi-dashboard-plan.md` status (it still says "no implementation started") and tick 4.2 / 4.3 / 4.5 / 4.6 partially.
- [ ] Excel/spreadsheet ingestion for the non-monday KPIs (POI is already a dependency) — deferred by the user.
- [ ] Remove [[MondayPreviewController]] once the console has the KPI screens; its empty-columns behaviour changed (now returns all columns).
---

---
## Session: 2026-09-08 (afternoon) — KPI formulas from the RE team, three boards live, validated against the deck

**Date:** 2026-09-08
**Tags:** #session #backend #frontend #kpi #monday

### Summary
Continuation of the morning session. The user supplied a monday token (now in the gitignored
`application-local.properties`, never committed) and the **RE team's real formulas**, which replaced the
placeholders. Three boards now sync — Cleaning Tickets `3451717331`, Delivery Tickets `1647612496`,
**Installation Tickets `3109668017`** — and the console's KPI page computes from them. Local stack:
docker Postgres `raaspal-kpi-pg` on 5433 (Jenkins owns 8080, so the API runs on **8081**).

**The formulas (user, verbatim intent):**
- **1st Time Install** — from the installation ticket's TimeLine, the *later* date ("30Sep-8Oct → 8Oct,
  or the only date"); look forward **30 days**; any CM naming the same S/N scores it 0.
- **First Time Fix** — after a CM, another CM on the same S/N within **14 days** scores it 0.
- **SLA** — checked within **7 days** of the CM report, measured to the board's **RE Action** date.
  Neither ticket board has a close-date column, so this is time-to-first-action, not time-to-close.
- **The S/N is the foreign key across all three boards.** Installation Tickets has no cleaning/delivery
  column, so an installation's line is resolved by finding its serial on one of the single-line CM boards.

**Validation — live vs the Jan–Jun 2026 deck, after the full read:**
| | live | deck |
|---|---|---|
| Delivery CM by month | 146·131·136·112·165·126 | 146·131·136·111·165·126 |
| Cleaning CM by month | 184·114·140·81·133·104 | 184·112·139·66·38·104 |
| Delivery CM total | 816 | 815 |
| 1st Time Install cleaning / delivery | 9/15 · 5/6 | 10/17 · 4/6 |
| First Time Fix | 75.7% | 72.3% |
| SLA within | 88.2% | 77.2% |
Delivery is exact. Cleaning Apr (81 vs 66) and May (133 vs 38) are the open gaps — the deck's May cleaning
figure was itself derived (203−165), so it may be the deck that is off. SLA differs because the deck's
"checked" definition is unconfirmed.

**Bugs found by running against the real boards:**
1. **Three Delivery column ids were wrong** — serial is `tags42` (was `asset_owner` = Project), project is
   `asset_owner`, branch-code is `tags2`. The serial error would have matched repeats on project name.
2. **No HTTP timeout on the monday client** — a request parked the sync thread for 29 min on 1.8 s CPU;
   with the shared lock every later run would have been refused until restart, and the nightly scheduler
   would have gone silent. Now 15 s connect / 90 s read, one retry on transport failure only.
3. **Page cap of 50×50 silently truncated Delivery's "DONE-Ticket" archive at 2,500** — delivery came out
   at 280 vs the deck's 815 while cleaning matched. Cap now 500 pages × 100 rows, configurable.
4. **`case_ticket.service_line` NOT NULL would have crashed the installation sync** (V40 makes it nullable);
   `case_ticket_sync_run.service_line` likewise (V39).
5. Run rows left RUNNING by a dead process now flip to FAILED at startup.

**Data facts (counts only — the user set a STRICT rule: never pull ticket rows into a transcript):**
- Cleaning 2,720 rows (serial on 2,554; RE Action on 2,086). Delivery 5,400 (serial 4,750; RE Action 3,431).
  Installation 833 (TimeLine parsed on 796; **serial on only 177** — the board rarely records S/N, so 1st
  Time Install is truly measured for ~a fifth of installs; the rest count as success and are reported as
  `withoutSerial`). 38 tickets in H1 are unclassified (serial never serviced).
- Timeline `text` over the API is `YYYY-MM-DD - YYYY-MM-DD` (verified in monday docs); "30Sep-8Oct" is UI only.

### Files Modified
- `db/migration/V39__add_ticket_type_and_action_dates.sql`, `V40__allow_unclassified_service_line.sql` — additive; **applied to the local docker DB only, not prod**
- `kpi/*` — `TicketType`, action/install dates, keyword + serial-link classification, `unclassifiedTickets`, startup cleanup; `KpiCaseMetricsService` rewritten to the formulas
- `casereport/adapters/monday/*` — timeouts + retry, `describeBoard` with `settings_str`, `listBoards`, configurable page cap, per-page progress log
- `application.properties` — all three boards mapped, every id **verified live**; windows 14/30/7
- Frontend `feat/re-kpi-dashboard` — `lib/api.ts` `kpiApi`, `lib/kpi/api-types.ts`, `lib/kpi/from-api.ts`, `ReKpiReportTab` fetches live; PM Complete and CSAT tiles **removed** rather than shown as constants
- `docs/kpi-local-testing.md` — runbook

### Decisions Made
- **Blank serial = success, reported** rather than failure, for both windows.
- **Unclassifiable = fleet total only**, never guessed into a side; contested serial (both boards) classifies nothing.
- **No action date = SLA unknown**, never a breach.
- **7-day SLA supersedes the vault's 3/5-day note** (that was the daily pending-case report's time-to-close).
- Deck tiles with no source (PM Complete, CSAT) are dropped from the live page, not faked.

**Late addition — status labels changed two things (V41, commit 7191e44).** Reading each status column's
`settings_str` (board config, not rows) showed "Installation Tickets" is the RE team's *job* board — Job Type
has 25 labels and "Installation" is one — and the cleaning board's "Type of case" includes parts shipments.
Each board can now name a category column and the values that count (`include-categories`, `(blank)` for
empty cells); rows outside are archived but not counted, reported as `excludedByCategory`. **A cleaning
filter dropping parts-shipment rows was tried and reverted:** it reproduced the deck's total (639 vs 643)
but only by coincidence — it removed 84 Feb–Mar tickets the deck's own monthly labels keep (unfiltered
Jan/Feb/Mar/Jun match within two), and a May surplus cancelled that. Cleaning and delivery count every row;
the real gap is **May cleaning: deck 38, live 133**. The filter mechanism stays for the installation board. The installation list is provisional: H1 still gives 59
(Plans 26, Mapping & Training 24, Installation 6, DONE 2, Install mapping 1) vs the deck's 23, and no subset
of labels gives 23 — **which Job Types count as an installation is a question for the user/RE team.**

### Unresolved / Next Steps
- [ ] **PM Complete** — sourceable: numerator = "PM Yip-upload" `2957857962` (one row per visit, MA date, S/N tags, groups "Done…"/"RAAS Done"); denominator = contract boards "PM Cleaning" `2048972900` / "PM Delivery" `4129404143` (SN:Robot 1–5, warranty timeline, Supplier Team = Raaspal vs Yip in tsoi/SMC). **Waiting on the user for visits-due-per-robot** (deck's 247.9/274.5 is fractional). "Initial Schedule MA Monthly Plan" is only a document tracker.
- [ ] Installation "Job Type": confirm which labels are installations (see late addition) — the H1 count is 59 vs the deck's 23.
- [ ] Explain cleaning Apr/May gap and the SLA definition with the RE team; CSAT has no monday source.
- [ ] **Rotate the monday token** — it appeared in a screenshot the user pasted.
- [ ] Nothing pushed; backend `feat/re-kpi-dashboard` is 8 commits ahead of main, frontend 1.
---

## Session: 2026-09-09 — KPI routes split, CSAT from the survey workbooks, CSAT off the report

**Date:** 2026-09-09
**Tags:** #session #backend #frontend #kpi #csat

### Summary
Three things. (1) The KPI page's tab bar became four routes under a sidebar dropdown —
`/kpi/report`, `/kpi/utilization`, `/kpi/repeat-cost`, `/kpi/csat` — with the old `?tab=` links
redirecting and keeping their period. (2) CSAT got a real source: the RE team's four monthly survey
workbooks (installation, MA = PM, CM cleaning, CM delivery). The backend reads them as they are —
month sheets only, every figure found by label, Top Box and the mean recomputed from the rating
counts — and **every figure on the deck's CSAT slide reproduces exactly**: Top Box overall 86.2,
install 79.2, PM 91.9, cleaning 70.3, delivery 89.7; CSAT 96.3; response rate 53.6 (1,118 of 2,084).
(3) CSAT left the RE report page: it is a monthly hand tally, and that page is for figures computed
from tickets. It has its own page, which says up front that it is not live and how far it runs.

Definitions settled by the workbooks' own arithmetic, no RE-team question needed: **Top Box = ratings
of 5 over all ratings across the five questions; the overall is pooled** (the average of the four
surveys would be 82.8%); **response-rate denominator = customers contacted** (`# ลูกค้า`), not jobs
done. The parser cross-checks the sheet's own summary cells and reports disagreement — the
installation workbook's March 2026 sheet has a stale Top Box cell (83.3% vs 9 of 11) and a broken
Overall CSAT cell (3.47 over counts that cannot average below 4.6). Counts are used; the RE team
should be told. The mean check tolerates ±0.25 because the sheet averages per-question means while
the KPI pools ratings, which parts by a few hundredths whenever someone skips a question.

Also today: an afternoon lost to a token/backend mismatch. `.env.local` points `BACKEND_PROXY_TARGET`
at the Render production backend; a token minted there is a 401 on local, and the sidebar still says
"signed in". Diagnosed by socket (`lsof -a -p <next-pid> -iTCP`), not latency — latency misled once,
and the seeded dev credentials went to production as a result. Rotate `admin@raaspal.com`.

### Files Modified
- Backend (`feat/re-kpi-dashboard`, uncommitted): `kpi/csat/{CsatStream,CsatWorkbookSource,
  FolderCsatWorkbookSource,CsatWorkbook,CsatWorkbookParser}`, `config/KpiCsatProperties`,
  `dto/KpiCsatResponse`, `service/KpiCsatService`, controller endpoints `GET /kpi/csat`,
  `GET /kpi/csat/source`, `POST /kpi/csat/reload`; `application.properties` `app.kpi.csat.*`;
  tests `CsatWorkbookFixtures`, `CsatWorkbookParserTest` (5), `KpiCsatServiceTest` (10), security
  test (+3); runbook CSAT section. No migration — parsed on request, cached until a file changes.
- Frontend (uncommitted): routes `app/[locale]/kpi/{report,utilization,repeat-cost,csat}/`,
  `lib/kpi/params.ts`, sidebar + mobile submenu, `CsatTab` rewritten on `GET /kpi/csat`,
  `KpiId` loses `csat`, report grid 6 → 5, `PLACEHOLDER_KPIS = ['pmComplete']`, en/th messages.

### Decisions Made
- CSAT source = the four workbooks in one folder (`app.kpi.csat.folder`; local profile points at
  `~/Downloads/csatscorefordashboard`). The team replaces them monthly; S3 later behind the same
  `CsatWorkbookSource` interface. Nothing persisted, nothing scheduled.
- CSAT is not on the report page (user, 2026-09-09): "reserved for real time, not dead data".
- No try-local-then-prod fallback in the proxy: a 401 is not a transport failure, and falling back
  would serve production data while testing local. Proposed instead: a backend badge in the header
  and auto-signout when the token's backend differs from the proxy target — not built yet.

### Unresolved / Next Steps
- [ ] Commit both repos and this vault; the frontend has two days of uncommitted work.
- [ ] Tell the RE team about the installation workbook's March sheet (two stale cells).
- [ ] PM Complete visits-due rule, Job Type 59-vs-23, May cleaning 38-vs-133, SLA 71.8-vs-77.2 — unchanged.
- [ ] Rotate the monday token **and** the seeded app password.
- [ ] Backend badge + auto-signout on backend change.
---

## Session: 2026-09-10 — CSAT: the sheet's Top Box cell, and the deck's way of combining

**Date:** 2026-09-10
**Tags:** #session #backend #frontend #kpi #csat

### Summary
Settled after a long, painful loop. The final rule: **where a cell exists, show the cell; where
the deck had to combine sheets, combine them the deck's way.** One survey in one month = the month
sheet's own Top Box cell (`I17 = AVERAGE(Q10:Q14)`, found by the "Top Box" label — never the
`Detail_` tabs, never recomputed). Totals over months or surveys — which no sheet holds, verified by
scanning all 28,211 numeric cells — are all fives over all ratings, which is how the deck was built.
Every deck figure reproduces exactly: 86.2 / 79.2 / 91.9 / 70.3 / 89.7, response 53.6. Monthly
per-survey figures are the cells verbatim (installation March = 83.3%, the sheet's number).

What went wrong, for the record: I never looked at the formula in I17. I inferred a method
(pooling counts) from whether my output matched the deck, defended it when the user said "just read
the cell", then invented a *second* method (weight monthly cells by responses → 79.5) and defended
that. Three numbers, three methods, all mine; the data never changed. The user's instruction had
been the same throughout: read the cell; calculate only what has no cell. The 0.06pt between the
sheet's AVERAGE-over-questions and the deck's pooling is one March respondent who answered Q1 and
stopped — real, tiny, and nobody's to fix.

Also today: the mean-based "CSAT 96.3%" removed from the tile (Top Box only), and every
cross-check warning removed — the dashboard does not audit the RE team's spreadsheet. Then, on the
user's ask, a row of four small charts under the overall one — installation, PM, CM delivery, CM
cleaning, each with its period total as the headline. The reference line I first drew on them came
straight back off: at that width the bars sit close to their own total, so the rule ran through the
value labels and struck them out, and the total is the headline already.

### Files Modified
- Backend: `kpi/csat/CsatWorkbook` (MonthAggregate: customers, responses, notEvaluated, topBox,
  fives, ratings), `CsatWorkbookParser` (cell by label + the I/N column sums), `KpiCsatService`
  (Tally: one sheet → its cell, several → fives/ratings), DTO docs, tests (5 + 10 + 11 green),
  runbook CSAT section.
- Frontend: `CsatBucket` type (six fields), `CsatTab` (Top Box only, response count beside each
  figure; `StreamCard` — the per-survey chart row, 4-across on a wide screen, 2x2 on a laptop),
  en/th messages (`kpi.csat.bySurvey.*`).
- Then a detail view, on the user's ask, matching what the report page does on a panel click:
  `CsatDetailView` + `lib/kpi/csat.ts` (the survey list, colours and the `?survey=` type, shared by
  the page, the view and the parser). Its subject is provenance — every figure labelled "sheet
  cell" or "pooled", the arithmetic drawn where there is any (76 ÷ 96 = 79.2%), and the average of
  the monthly cells named as what the number is *not* (90.6%). The pool's own bars are pooled
  months rather than cells, so its notes say that; the first draft claimed cells for both and would
  have taught the reader the exact error the view exists to prevent. Backend: `Bucket` gains
  `topBoxFromSheet`, `fives`, `ratings` — counts the tally already had; no figure changes.

### Excel export (same day, after a Q&A on presentation formats)
The user asked which format gets these charts into a deck **editable** — numbers changeable,
colours changeable. That rules out every image, SVG included: SVG + PowerPoint's Convert to Shape
gives editable *shapes*, but the numbers are then a drawing, so 79.2 → 80.1 means retyping a label
and dragging a bar. The answer is a **native PowerPoint chart**, which means handing over data, not
pictures. Built as `GET /kpi/cm-cases/export` and `GET /kpi/csat/export` (`XlsxBook` + two
exporters over the POI already in the pom), with an **Export Excel** button beside the period
selector on the report and CSAT pages.

The layout decisions are the feature — get them wrong and Excel's Insert Chart fights you:
- **Header on row 1, data from row 2, nothing above, nothing merged.** A title row or a banner and
  Excel picks the wrong range and loses the series names. This is why provenance goes on an
  `About` sheet instead of at the top of each table.
- **Months down, series across** — the axis the deck uses.
- **Rates are real percentage cells** (0.792 formatted `0.0%`), so the chart axis is a percentage
  axis rather than one running 0 to 1.
- **Never-surveyed leaves the cell blank, not 0.** A gap charts as "not surveyed"; a zero bar
  charts as "nobody was happy".
- CSAT rows carry `sheet cell` / `pooled` beside every figure — the distinction the feature turns on.

Verified by downloading both files and reading them back: `Period Totals` reproduces the deck
(86.2 / 79.2 / 91.9 / 89.7 / 70.3), and clicking the button in a headless browser lands a real
file on disk. 211 backend tests green.

### The export had no charts in it (same day, second pass)
The user opened the workbook and asked where the charts were — a fair question: the first pass
exported data shaped *for* charting, which still left them inserting a chart per panel every month.
The sheets now carry the panels already drawn as real Excel chart objects: same type, same series
colours, values on the bars, and the dark average rule (a flat line series over a column repeating
the one figure, since a chart cannot hold a bare horizontal line). CSAT's Top Box sheet holds five
— the overall panel, then each survey — and the report one per panel sheet.

**Three POI traps, all of which surface only as Excel's "we found a problem with some content":**
1. `poi-ooxml-lite` **cannot write a stacked chart**: it ships the generated classes but not every
   compiled schema resource, so `<c:overlap>` (without which Excel draws a stacked chart's series
   side by side) throws *Could not locate compiled schema resource … stoverlappercent….xsb*. The
   pom now excludes lite in favour of `poi-ooxml-full`, ~14 MB heavier.
2. A `<c:lineChart>` **must declare `<c:grouping>`** and POI writes none, so every panel carrying
   an average rule produced an invalid file.
3. POI marks a number format **source-linked**, telling the reader to ignore the format code: the
   rate axis rendered 0–1 and the labels printed `0.682`. Both `numFmt`s now say
   `sourceLinked="false"`, and data labels need their own as `dLbls`' first child.
Also: `XDDFLineProperties.setWidth` is in **points**, not EMU.

The lesson worth keeping: `CTChart.validate()` in a test names the offending element, where Excel
names nothing. All three bugs above were found that way rather than by opening the file — and note
`qlmanage` renders the bars but ignores chart number formats, so it cannot confirm label formatting.
Excel itself resisted scripting (`save as picture` timed out on AppleEvents), so the formatting is
schema-verified, not eyeballed.

### Charts matched to the page (same day, third pass)
Opened in Excel the charts were right in kind but not in look, and the four survey charts had no
rule where the page heads each with its total (79.2% for installation). Now: one average column
**per survey** on the Top Box sheet so each chart draws its own rule, labelled at its end as a data
label on the line's last point ("Avg Installation 79.2%" — a chart cannot hold a floating caption);
axis 0–100% in quarters with faint gridlines; gap width 60 (the page's 62% slot).

On the site the rule went back onto the four cards **and the detail view**. The reason it had come
off — the rule struck value labels through, visibly even on the overall panel (March "82%") — is
fixed at the source: labels sit above the rule on the panel's background, so the rule breaks around
them. The row of four cards now waits for `2xl`; at `xl` six labelled bars plus the Avg row left the
labels 3px apart, and as a 2×2 they get 35px.

### Labels vs the rule, resolved properly (same day, fourth pass)
The label-background trick rendered as a rule chopped into dashes between the labels — the user
called it "interrupting", rightly. Replaced with placement: a value label sits on its bar unless
the rule would run through it, then it sits just above the rule (band in axis-%, since the
component does not know the plot's pixel height). The plot also gained 14px of headroom above the
100% line so a label over a full-height bar does not climb into the "Avg" caption. Applies to
every panel, cards and detail view included; verified at 1600px (four across, 15px between
labels) and 1280px (2×2).

### Excel: 100% bars "cut off" (same day, fifth pass)
A value axis pinned at exactly 1.0 puts a 100% bar's label outside the plot, which reads as the bar
being clipped. QuickLook hides it — it ignores the fixed axis and autoscales — so the preview looked
fine and Excel did not. Axis max is now 1.1 with the 0.25 unit, so no tick says 110% and the top just
has room; the same headroom fix as the site's panels.

### Unresolved / Next Steps
- [x] Committed and pushed to `feat/re-kpi-dashboard`: backend `8f1889e`, frontend `252f46f`, plus
  `4b59086` for an unrelated fix that had been sitting uncommitted (`translate="no"` on `<html>`;
  Chrome's translate offer rewrites text nodes and the next route transition then throws
  NotFoundError on removeChild).
- [x] The 8081 backend was restarted (`sh mvnw spring-boot:run -Dspring-boot.run.profiles=local`,
  docker Postgres on 5433) to verify the detail view against real data; it had been an IDE launch
  from `target/classes` since 09:37, three hours older than the parser. March installation now
  reads 83.3% and the warnings box is gone.
- [ ] Optional, RE team's call: Q5 is literally "overall satisfaction"; textbook CSAT would use it alone.
## Session: 2026-09-11 — Daily Pending Case Report: MK sheet, SLA, freeze and daily sync

**Date:** 2026-09-11
**Tags:** #session #backend #frontend #database

### Summary
Took the Daily Pending Case Report from a monday adapter to a report the team can open and check. The parked
case-report migration was renumbered **V38** and applied to Supabase (Flyway validated all 37 prior migrations
with no checksum mismatch). The **MK sheet** now generates from the live delivery board, matching the
`Raw_Delivery` sheet of the 09-Sep-2026 workbook column for column, and is viewable under Reports → Pending cases.

The workbook settled three things that had been assumptions. **Days is inclusive** of the open day — on all five
raw sheets `Open Date + Days` lands one past the "as of" date in the heading. **The SLA column is text**
(`over SLA` / `Within SLA` / `On Hold`), not colour. And **MK, Yayoi and Bonus Suki are one customer** (MK
Restaurant Group), which is why one SLA covers them and why branch prefixes M###, Y### and K### sit together.

The user then found the real problem: monday is edited continuously, so picking a past date returned today's
board under an old heading. Three mechanisms were built. **A — freeze on generate**: rows are stored in
`case_report_run.rows_json` the first time a date is generated and served thereafter. **B — daily sync**: both
boards are snapshotted into `case_ticket`, `case_ticket_update` and `case_ticket_status_history`. **Guard**: a
past date with no frozen run is now *refused* rather than fabricated.

⚠️ **A was first described as fixing back-dating. It does not, and that claim was wrong.** A freezes whatever it
captures at generation time, so generating 09 Sep on 11 Sep froze 11 Sep's board. The user caught it by
noticing both dates showed identical rows (only Days differed). The bad 09 Sep run was discarded and the guard
added so the mistake cannot recur. **Nothing before 2026-09-11 is recoverable** — the first snapshot is that day.

### Files Modified
- [[V38__add_case_report_tables]] (`RaasPal-Internal-Ops-backend/src/main/resources/db/migration/V38__add_case_report_tables.sql`) — renumbered from the parked `V34__...sql.txt`; **applied 2026-09-11**
- [[CaseSource]], [[CaseTicket]], [[CaseTicketUpdate]], [[CaseTicketStatusHistory]] (`casereport/entity/`) — snapshot entities
- [[CaseReportDefinition]], [[CaseReportRun]], [[CaseRunStatus]] (`casereport/entity/`) — report config and frozen runs; `isReplaceable()` is false only for SENT
- [[CaseTicketRepository]], [[CaseTicketUpdateRepository]], [[CaseTicketStatusHistoryRepository]], [[CaseReportDefinitionRepository]], [[CaseReportRunRepository]] (`casereport/repository/`)
- [[SlaStatus]], [[SlaCalculator]] (`casereport/service/`) — `daysOpen()` is the single source for both the Days column and the verdict
- [[CaseReportRow]] (`casereport/dto/`) — field order is the sheet's column order
- [[MkPendingReportGenerator]] (`casereport/service/`) — live delivery-board read, MK/Yayoi/Bonus Suki filter, `#` prefix stripped, branch code joined to name
- [[CaseReportRunService]] (`casereport/service/`) — freeze, discard, and the past-date refusal
- [[CaseReportDefinitionSeeder]] (`casereport/service/`) — creates MK_PENDING on startup; **never updates**, so DB edits survive restarts
- [[CaseTicketSyncService]], [[CaseSyncCoordinator]] (`casereport/service/`) — per-board sync, driven from a separate bean
- [[CaseReportDailyScheduler]] (`casereport/scheduler/`) — snapshot then freeze, 06:15 Bangkok, **off by default**
- [[CaseReportController]] (`casereport/controller/`) — `GET /mk`, `GET|DELETE /mk/run`, `POST /sync`, `GET /sync/status`
- [[application.properties]] — `app.casereport.sync-enabled` / `sync-cron`; metro-province comment enriched
- test `application.properties` — `metro-provinces` and `sync-enabled=false` (this file shadows the main one)
- [[SlaCalculatorTest]] — 13 tests, replaying all 16 rows of the live delivery sheet
- [[CasePendingPanel]] (`robot-recommendation-web-raaspal/components/CasePendingPanel.tsx`) — new read-only review screen
- [[ReportsClient]], `reports/page.tsx`, [[api]], `types/api.ts`, `messages/en.json`, `messages/th.json` — third tab group
- `robot-recommendation-web-raaspal/.env.local` — switched to MODE B (gitignored, local only)

### Decisions Made
- **SLA is 3 days inside the six greater-Bangkok provinces, 5 elsewhere, for MK/Yayoi/Bonus Suki delivery** — the user's statement, confirming V38. Compared with `>`, so a case at exactly the limit is still within SLA. Cleaning is 3 days everywhere, and when both thresholds are equal the province is never consulted.
- **Province comes from `color_mm6mwh74`**, a dropdown on the delivery board only. Verified 2026-09-10: 14 labels, all open tickets filled. The six metro labels happen to be Latin and the eight upcountry ones Thai; both scripts are listed in config so a later `กรุงเทพมหานคร` cannot fall through to the 5-day threshold.
- **A missing province yields no verdict, never a default** — defaulting to 5 would show a late Bangkok case as on time.
- **Column ids live in each generator, not a shared map.** `text` is Solution on delivery but Main Issue on cleaning; `status_1` is Sup Status on delivery but Issue Level on cleaning. A shared mapping is one place to get both boards wrong.
- **Refuse a past date rather than fabricate it.** A live read stamped with an old date is convincing precisely because Days changes with `asOf` — so it is worse than an error.
- **Each board syncs in its own transaction, from a separate coordinator bean.** `@Transactional` on a method called via `this` does nothing; the first two attempts failed with "No EntityManager with actual transaction available". Separate transactions also mean a cleaning failure does not roll back the delivery snapshot.
- **Scheduler off by default**, matching `ReportDeliveryScheduler`.

### Unresolved / Next Steps
- [ ] ⚠️ **Turn the scheduler on** (`CASE_REPORT_SYNC_ENABLED=true`). Every day it stays off is a day that can never be reported on. Until then, someone must generate the report each morning or that date is lost.
- [ ] **The deployed backend has none of this.** Deploying needs `MONDAY_API_TOKEN` set on Render — `application.properties` defaults it to empty, so the endpoint would exist and fail on its first monday call.
- [ ] **Excel export not built.** `poi-ooxml` is already in `pom.xml`. The workbook shows the MK sheet should be one sheet in `Raw_Delivery` layout; an All Case export should mirror the full workbook.
- [ ] **Snapshot-backed reconstruction not built.** The generator still reads monday live. Past dates are answered only by frozen runs. `case_ticket` is updated in place, so a reconstruction could recover status (from history) but not problem text, branch or province as they stood.
- [ ] **No "regenerate from board" control in the UI** — `?refresh=true` works on today's draft but the panel never sends it.
- [ ] Two rows of the 09-Sep file disagree with the SLA rule: M453 โรบินสัน ฉะเชิงเทรา (7 days) and M057 ศรีราชานคร (6 days) both print "Within SLA" though both are upcountry and past 5 days. Pinned as expected divergences in `SlaCalculatorTest`. Worth asking whether a scheduled RE On Site date exempts a case — both rows have one.
- [ ] Header typos `Brucn` and `Solutiom` in the live template — keep verbatim or correct? Unanswered.
- [ ] The earliest sync attempts ran with no transaction, so each `save()` committed individually while the bulk close failed. The resulting rows are correct, but those runs were not atomic.
- [ ] `activity_logs` on monday retains column changes with previous and new values — a possible backfill route for the recent past. Timestamp units unconfirmed (a naive conversion produced year 2536) and retention unknown.
---
---
## Session: 2026-09-11b — Solution column via Haiku, merge with PM planner, PR #6 reviewed and merged

**Date:** 2026-09-11
**Tags:** #session #backend #ai #deployment #config

### Summary
Three things landed on `main` in one PR. **First, the Solution column.** It was blank on seven tickets in eight,
and the earlier session's design — "a status log built from consecutive snapshots, no AI" — was wrong: the
09-Sep workbook's Solution lines are a human *paraphrase of the comment thread*, one dated entry per step,
in a fixed house vocabulary (`อยู่ระหว่าง…`, `รอ…`, `เจ้าหน้าที่เข้าซ่อม`). So [[MkPendingReportGenerator]] now
sends each ticket's comments (oldest first, intake form dropped) to a new [[CaseSolutionAiService]], implemented
by [[ClaudeAiService]] on Haiku (`claude-haiku-4-5-20251001`) and by [[MockAiService]] without a key. A value
somebody typed into the board column still wins. The one rule the model kept breaking — a date range across two
months, `25-11 Sep` — is enforced deterministically in [[SolutionLine]] rather than re-prompted.

**Second, the merge.** The coworker's PM 52-week planner reached `main` as **V39** (with the case-report V38
already on production, so main alone would have failed Flyway validation). `main` was merged into `dev-1`
(`faa74a9`); V36–V39 all present, nothing dropped — the CM report (V27) and the case report (V38) were both
checked file by file after the user asked where the CM code was.

**Third, PR #6 (`dev-1` → `main`), reviewed before merging.** The review found eight things; four were real
defects and were fixed on the branch (`cb40b6f`) before the merge (`cf04b84`):
1. **The Thai metro provinces never matched.** Spring Boot reads `.properties` as ISO-8859-1 (verified in the
   `spring-boot-3.4.5.jar` loader), so `กรุงเทพมหานคร` and the rest loaded as mojibake and a Thai-spelled metro
   province would have taken the 5-day SLA. Not yet triggered — the board's six metro labels are Latin — but
   the config existed precisely for the day that changes. Now `\uXXXX`-escaped in both properties files, with
   [[MetroProvincesPropertyTest]] loading both through Spring's own loader.
2. **Solution dates were UTC.** monday's `created_at` ends in `Z`; a comment posted before 07:00 Bangkok was
   dated the previous day. The generator converts to `Asia/Bangkok`; the sync pins `posted_at` to UTC explicitly.
3. **`25-11-Sep` (hyphen before the month) went through unsplit** — only the space form was recognised.
4. `MockAiService` used `"\s+"`, which since Java 15 is a literal-space pattern, not whitespace.
The other four are logged below as follow-ups, none blocking a manual, single-reviewer rollout. Suite: **200
tests pass** (191 after the merge + 9 new).

The Monday API key was moved out of the local override into env config (`MONDAY_API_TOKEN`), and the V38
numbering collision with the coworker's KPI branch was settled: production keeps the case-report V38; his
branch reworks to an `ALTER` at V40+ after this merge.

### Files Modified
- [[CaseSolutionAiService]] (`ai/service/`) — new interface, `summariseProgress(CaseProgressRequest)`; never throws, empty string for nothing
- [[ClaudeAiService]] (`ai/service/`) — implements it on Haiku; [[MockAiService]] — deterministic dated condensation; regex fix
- [[AiPromptTemplates]] (`ai/prompt/`) — `caseSolutionSystemPrompt()`, transcribed from the workbook's style
- [[CaseProgressRequest]] (`casereport/dto/`) — branch, problem, statuses, asOf, `List<Comment(postedOn, author, body)>`
- [[SolutionLine]] (`casereport/service/`) — cross-month range splitter; now both `25-11 Sep` and `25-11-Sep`; one-day sides print `31-Aug`; impossible first day left as written
- [[MkPendingReportGenerator]] — `solutionFor()`; comment dates in Bangkok; future-dated tickets skipped on a back-dated run
- [[CaseTicketSyncService]] — `posted_at` pinned to UTC; stale "status log" Javadoc corrected (also in [[CaseTicket]], [[CaseTicketStatusHistory]], [[CaseReportRow]], [[CaseReportDailyScheduler]])
- [[application.properties]] + test copy — `app.monday.api.token=${MONDAY_API_TOKEN:}`, `MONDAY_UPDATES_PER_ITEM`, metro provinces unicode-escaped
- `deploy/api.env.example`, `deploy/DEPLOYMENT.md`, `README.md` — monday + case-report settings documented for Lightsail
- [[SolutionLineTest]] (11), [[MetroProvincesPropertyTest]] (4), [[MkPendingReportGeneratorTest]] (1) — new/extended
- Merged from main: [[V39__add_pm_planning_tables]], `pm/` package, `deploy/preview/` stack, monday DTO changes (`MondayItemRef`)

### Decisions Made
- **Solution is a model paraphrase, not a snapshot-derived log.** Verified against the workbook line by line; the earlier claim in this vault was wrong and is superseded. A typed board value always wins over the model.
- **Haiku, not Sonnet, for the Solution line** — one call per ticket per generation, short output, house-style constrained; cost matters more than nuance here.
- **The month-boundary rule lives in code, not the prompt.** A paraphrase is not deterministic; a rule the report cannot break has to be enforced where it can be tested.
- **Fix the review's real defects before merging, defer the structural ones.** AI calls inside `@Transactional`, the concurrent-first-generation race, and discard-of-past-runs are documented on PR #6 rather than merged as workarounds.
- **Merge, not squash** — matching the team's PR #4/#5 history.

### Unresolved / Next Steps
- [x] **Deployed to Lightsail 2026-09-11 10:21 UTC** via `bash deploy/deploy.sh` — up after 60 s; Flyway `Current version of schema "public": 38` → `Migrating … to version "39 - add pm planning tables"` → `now at version v39`; `Started … in 41.8 seconds`; container `(healthy)`; `https://api.raaspal.com/v3/api-docs` 200 and `/actuator/health` 401 (expected — not permitted); 1.6 GiB used of 7.6, no swap. The authenticated case-report check was run with placeholder credentials and so returned 401 — still to be done with a real login, and `MONDAY_API_TOKEN` / `ANTHROPIC_API_KEY` presence in `deploy/api.env` is unverified.
- [ ] Set `MONDAY_API_TOKEN` on Render too while it is still serving.
- [ ] After deploying, regenerate today's MK draft with `?refresh=true` — drafts frozen before the UTC fix may date early-morning comments a day early.
- [ ] PR #6 follow-ups: split `CaseReportRunService.rowsFor` so the Haiku calls run outside the transaction; catch `uq_case_report_run_day` on a concurrent first generation and serve the stored run; consider ADMIN-only `DELETE /mk/run` for past dates.
- [ ] `deploy/api.env.example` trap (pre-existing): bare `KEY=` lines give Spring an *empty string*, not the default — 8 numeric/cron keys crash startup if left bare (`CVTE_KAVA_POLLING_INTERVAL_MS`, `PARTNER_*`, `REPORT_CACHE_*`), 11 silently lose their default. Comment them out instead.
- [ ] Coworker's KPI branch: its V38 → `ALTER TABLE case_ticket ADD COLUMN …` + `CREATE case_ticket_sync_run` at **V40**, its later ones V41–V43; rebuild his local DB.
- [ ] Console commit `cafd93a` on `feat/dailycasereport-page` carries a backend commit message (content is right). Optional amend before its PR.
- [ ] Scheduler still off; Excel export, AOT and ALL_PENDING generators still unbuilt (see previous session).
---
