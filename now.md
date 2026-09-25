# NOW — current state, read this first

> **Sections carry their own verified dates; most of this was checked 2026-09-17.**
> This file exists to be *cheap*: it is the one vault file an agent
> should read at the start of every session. [[index]] (626 lines) and [[history]] (2800+)
> are reference material — open them when you need the reasoning behind something, not by default.
>
> Anything here older than ~a week is a hint, not a fact. Re-check before relying on it.

---

## Repos (renamed — older vault entries use the old names)

| Folder | Was called | Stack | HEAD @ 2026-09-22 |
|---|---|---|---|
| `RaasPal-Internal-Ops-backend` | `robot-recommendation-api` | Spring Boot 3.4.5 · Java 21 | `cc3f31e` on `main` |
| `RaasPal-Ops-frontend` | `robot-recommendation-web-raaspal` | Next.js 16 · React 19 · Tailwind v4 | `72e4360` on `main` |
| `RaasPal-RIMS` | `raaspal-rims` | Next.js 16 · inventory console | not checked this session |
| `donation-website` | — | Next.js 16 · React 19 · Tailwind v4 · **pnpm** · "Kindred" item-donation prototype | `fa3a264` on `main` — **PR #1 merged** 2026-09-23 (merge commit, 15 commits); Vercel production deployed, behind Vercel SSO |

Also in the workspace, not code: `Info/` (source workbooks + PDFs), `Plan/`, `Report/`
(**all non-code deliverables go here, not the Desktop**), `AWS/`.

The workspace root is deliberately **not** a git repository.

## What just landed

**PR #3 merged in both repos on 2026-09-17 04:36Z** (backend merge `3749c70`, frontend `e070683`),
bringing the RE KPI dashboard and the PM 52-week planner onto `main` together. Three commits
landed on top: the PM company filter became an *include* list as well as an exclude list, the
frontend now sends whichever list is shorter (header-limit fix), and the CSAT uploader shows an
empty state.

**Merged to `main` and deployed 2026-09-22** (ff to backend `639fed8`, frontend `2a418ea`; perf fix `aa5d0df`/`8fa50b8`): AutoXing performance report, fault poller (V52, off by default), AutoXing registration, and **RE Assignment** (`/re-assignment`, V53; emails and scheduled refresh off). See [[history]] 2026-09-22.

**Merged to `main` 2026-09-24:** **MK spare parts** (backend `8159a9a`, V57 — Lightsail deploy pending; RIMS `1d63105`, Vercel auto-deploys). Staff pages in RIMS under `/mk-stock`; MK's read-only view at RIMS `/mk` behind a PIN. See [[history]] 2026-09-24.

## Build and run

- Backend: **`sh mvnw`**, not `./mvnw`. Local profile: `sh mvnw spring-boot:run -Dspring-boot.run.profiles=local`.
- Frontend dev server runs on **port 3001**, not 3000.
- **Never `npm run build` while `next dev` is live** — it corrupts `.next` and produces lying
  "X is not a function" errors in unrelated files. Type-check with `npx tsc --noEmit`.
  A plain restart does not clear it; Turbopack's cache outlives the process. `rm -rf .next`.
- donation-website: `pnpm install && pnpm dev` (port 3000). Its database is `data/store.json`, git-ignored, seeded on first read and **cached in memory** — to reseed, delete it *and* restart the dev server. See [[history]] 2026-09-23.
- ⚠️ **Backend tests on the Windows machine: set `JAVA_HOME` to JDK 21** (`C:/Program Files/Java/jdk-21.0.10`). The default `java` is 25, where Mockito cannot mock concrete classes and every test class with a `@MockitoBean` on a service fails to load its context — a whole class of errors that looks like your change broke it. (2026-09-25)

## Database

- Highest migration on `main` is **V60** (`V60__widen_customer_report_link_period_key.sql`). **Production is at V60**: V58, V59 and V60 applied 2026-09-25 02:50Z ("Successfully applied 3 migrations", deploy log). Next free number is **V61**.
- **Lightsail runs `d00af7f`** = backend `main` (deployed by the user 2026-09-25 02:49Z; checked on the box). Frontend `main` is `61f159d`, live on `ops.raaspal.com` since 2026-09-25 (new bundle confirmed).
- Logo files: `public/raas-pal-{logo,wordmark}.png` are the brand-blue **website** versions;
  `*-print.png` are the originals and are what the printed reports use. Do not recolour those.
- ⚠️ **The ops email alert has never run in production**: `api.env` has no `OPS_ALERTS_*` and blank
  `MAIL_*` (checked 2026-09-18) — **`MAIL_USERNAME`/`MAIL_PASSWORD`/`MAIL_FROM`/`MAIL_CC` are now set (checked 2026-09-25)**; `OPS_ALERTS_*` not re-checked. "alert pending" on the Contracts page is literal. See [[history]] 2026-09-18. Deploy: ssh in, `git pull`, `cd deploy;
  bash deploy.sh`; health at `https://api.raaspal.com/actuator/health`. `ops.raaspal.com` (Vercel)
  deploys itself on push to `main`; its proxy needs `BACKEND_PROXY_TARGET=https://api.raaspal.com`.
- ⚠️ **Local dev and production share one Supabase database.** Booting locally applies pending
  migrations to prod. Expand and contract: add, deploy, *then* drop.
- ⚠️ **Never edit a committed `V*.sql`.** Flyway checksums it and refuses to start on mismatch —
  breaks the next deploy with no local symptom. Add a new migration.
- Flyway 10 also refuses **out-of-order** versions, which is what makes the local DBs below bite.
- V59/V60 widened `report_month` to `VARCHAR(8)` on `report_links`, `report_sends`, `customer_report_links`: the columns now hold a *period key*, `2026-08` or `2026-W38`.

**Local Docker Postgres `raaspal-kpi-pg` on `localhost:5433`:**

| DB | State |
|---|---|
| `pm_verify` | schema **V42**, 4,352 `pm_visit` rows (synced 2026-09-11) |
| `kpi_local` | stale — **needs rebuilding** before the current backend will boot against it |
| `pm_v38`, `pm_v39` | empty, leftovers |

Main ships V49, so a fresh sync wants a **throwaway database**, not a migration of `pm_verify`.

## PM 52-week planner (newest feature)

Four monday boards, ids in `application.properties` under `app.pm.monday.*`:

| Board | Id | Role |
|---|---|---|
| PM Cleaning | `2048972900` | contract/site master |
| Subitems of PM Cleaning | `2444194682` | the actual visits |
| PM Delivery | `4129404143` | contract/site master |
| Subitems of PM Delivery | `4152679385` | the actual visits |

- The forward schedule **already exists** as subitems — the original spec's "calculate next PM
  date" premise was wrong. The problem was visibility, not calculation.
- Status lives on the **subitem** status column: Cleaning `color_mm1dz8vz`, Delivery `status`.
  Labels → buckets in `PmStatusBucket.fromRaw`; `OVERDUE` is computed at query time, never stored.
- Region/zone are derived from **province**, never from monday's `ภาค` column (12 spellings for
  ~7 regions). See `ProvinceResolver`.
- Company is **derived from the item name** (`PmItemMapper.deriveCompany`) because
  `customer_name_raw` is empty on 64% of rows.

**Data-quality findings worth fixing at source, in monday:**
- **1,080 of ~1,578 PM Delivery visits (68%) have no status at all.** Cleaning has zero blanks.
  That asymmetry is the entire grey/UNPLANNED population on the planner.
- 93 of 149 derived companies have exactly one site — ~60% of the company filter isn't a real chain.
- 49% of PM Delivery contracts have no province; ~1,330 Cleaning visits have no plan date.

## Open items

- [ ] **Rotate the monday API token.** A live `me:write` JWT leaked into an agent transcript via an
      IDE selection. Rotation status unknown. (The token itself is not recorded anywhere here.)
- [ ] `kpi_local` needs rebuilding (see above).
- [ ] Merged local branches `feat/re-kpi-dashboard` still exist in both repos and can be deleted.
- [ ] Wire the `pmComplete` KPI placeholder now that both features are on `main`.
- [ ] **No-data site map** — parked 2026-09-17 by the user. Blocker: no coordinates anywhere in the
      schema (`branch`/`site` are free text). See [[history]] 2026-09-17c for the proposed design.
- [ ] Orphan object in S3 `raaspal-customer-contracts` from the failed first attach — user deletes.
- [ ] ⚠️ AWS account is on the Free Plan — $107.76 credits, ends 2027-02-06 or when spent; production
      and the contract PDFs live in it. Director.
- [ ] `raaspal-api-preview` still up on `0.0.0.0:8081` — and **unhealthy** since ~2026-09-16. Stop it.
- [ ] donation-website — **now deployed to Vercel (2026-09-23), so these are live**: real database (JSON file won't work on Vercel; site likely errors on first load), remove the self-approve "demo" verification button and the demo logins on `/login`, require `SESSION_SECRET` (it falls back to a hard-coded dev value). Added 2026-09-23.
- [ ] **Weekly reports are live (2026-09-25)** — `REPORT_WEEKLY_SCHEDULER_ENABLED=true` on Lightsail, Monday 08:00 Bangkok. **Nothing sends until a robot is set to Weekly**: set Pandora's robot in Tools → Robots. First automatic run Mon 2026-09-28 for 21–27 Sep. See [[history]] 2026-09-25. Added 2026-09-25.

## Working rules that have cost time when ignored

- **No customer data in a transcript.** Schema and column names only, never live ticket rows.
  Aggregate counts are fine; names are not.
- **Small, meaningful commits** — split by unit of meaning, in dependency order. Never one big commit.
- **This vault has one branch, `main`.** Pull before writing, commit, pull again, then push.
- Parallel shell calls that each start with `cd` clobber one another — use `git -C <repo>` instead.
- zsh does not word-split unquoted variables, and *does* glob an unquoted `--include=*.java`. Quote it.

---

## Editing this file without fighting everyone else

`now.md` is **edited in place**, which is the shape git merges worst — unlike [[history]], which is
append-only and merges itself. Two people correcting the same fact on the same day *will* conflict.
Keeping that cheap:

- **One fact per line. Do not reflow or re-wrap** a paragraph you did not change — a reflowed
  paragraph is a conflict on every line of it, instead of one line.
- **Do not reorder sections.** New facts go at the end of the section they belong to.
- **Leave the section headings alone.** They are the anchors git aligns on.
- On a conflict, **keep the value that was actually verified**, not the newer commit, and put how
  it was verified in the line. If both were verified, the later measurement wins.
- If a fact needs a paragraph of reasoning, it does not belong here — put the reasoning in
  [[history]] with a date, and leave one line here pointing at it.

**Verified dates are per-section, not global.** Update the date on the section you touched; do not
re-date the file because you corrected one line in it.
