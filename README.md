# Allen's Auto Refinement — website + staff portal (Netlify)

**Detailing • Restoration • Protection** — *Reviving the past. Protecting your future ride.*
Virginia Beach, VA · used vehicles for sale · cash offers for used vehicles · detailing, restoration and protection (ceramic coating, PPF, undercoating). Cash business: no financing, no payment calculators, no credit applications.

> **This is a proposal build.** Every business detail that isn't known for certain — address, phone, email, hours, payment methods beyond cash, dealer license, owner story, legal entity — is a clearly marked **PLACEHOLDER** in [`config/business.json`](config/business.json). Fill it in once and it flows to every page, the footer, and the JSON-LD structured data (placeholders are shown on the page as dashed amber chips and are *left out* of structured data so nothing unverified is published to search engines). No reviews, ratings, prices, guarantees, statistics or certifications have been invented.

## Architecture

```
                 ┌────────────────────────── Netlify CDN (edge) ──────────────────────────┐
Browser ──HTTPS──▶  dist/*.html  (prerendered at build: /, /services/*, /sell, /about…)      │
                 │  /assets/*   (fingerprinted, cached 1 year, immutable)                   │
                 │  /inventory, /inventory/:slug, /sitemap.xml  → Functions, edge-cached    │
                 │      (Netlify-CDN-Cache-Control durable + Netlify-Cache-Tag; purged by tag│
                 │       the moment staff change inventory)                                  │
                 │  /media/*    → Function → Netlify Blobs (vehicle photos, immutable)       │
                 │  /api/v1/*   → Functions (forms, auth, staff API) with edge rate limits   │
                 └───────────────┬───────────────────────────────┬──────────────────────────┘
                                 │ HTTP (pooled)                 │
                       Postgres (Netlify DB / Neon)      Netlify Blobs: vehicle-photos,
                       versioned SQL migrations          lead-photos (private), backups
                                 ▲
             Scheduled functions: hourly email DLQ retry + housekeeping; nightly JSON backup
```

* **Static marketing pages** are rendered at build time by `scripts/build.mjs` from the same view code the functions use (`src/views`, an auto-escaping `html` template tag — every interpolation is HTML-escaped by default).
* **Inventory** pages are real crawlable URLs rendered by `netlify/functions/inventory.mjs`; filters/sort are plain query parameters (`/inventory?make=Toyota&body=SUV&max_price=25000&sort=price_asc`) so they work with JavaScript off.
* **Staff portal** `/admin` is a static shell (`public/js/admin.js`) talking to `/api/v1/*`.
* **Database**: Postgres. On Netlify the app uses `@neondatabase/serverless` over HTTP via Neon's pooled endpoint (no TCP connection to manage from cold functions). Any other Postgres URL uses `pg`. Tests and local runs can use PGlite (an in-process Postgres) — no install needed.

### Project layout

```
config/business.json      ← owner fills placeholders here (single source of truth)
config/features.json      ← feature flags (build + runtime; env overrides FEATURE_*)
migrations/NNN_*.sql      ← versioned schema; schema_version table records what ran
netlify.toml              ← build, functions, redirects, security + cache headers (IaC)
netlify/functions/        ← 15 small v2 (ESM) functions, each with `export const config`
src/lib/                  ← db, auth (JWT + rotating refresh), forms, validation, cache, blobs,
                            images (sharp), notify (email w/ retry, breaker, DLQ), backup, log
src/views/                ← layout, components, page templates (shared by build + functions)
public/                   ← css, js (site.js, admin.js, boot.js, ambient.js), fonts, source images (PNG incl. logo.png),
                            icons/ (favicons + site.webmanifest, copied to the publish root at build)
scripts/                  ← build, migrate, seed/clear demo, create-user, restore-backup,
                            local-server (Netlify emulation), smoke
test/                     ← node:test suites (36 tests) against PGlite + in-memory Blobs
docs/                     ← RUNBOOK.md, INCIDENT_RESPONSE.md (with postmortem template)
CHECKLIST.md              ← every engineering-checklist item → how it's handled / why N/A
```

## Deploy to Netlify

### Option A — GitHub → Netlify (recommended: gives CI, deploy previews, rollbacks)

1. Push this folder to a new GitHub repository (the lockfile is committed; `src/generated/`, `dist/` and `.env` are git-ignored).
2. In Netlify: **Add new site → Import an existing project → GitHub →** pick the repo. Netlify reads `netlify.toml` (build command `npm run build`, publish `dist`, functions `netlify/functions`, Node 20). Don't change these in the UI.
3. **Create the database.** Either:
   * **Netlify DB** (Neon, managed by Netlify): Site → *Extensions / Netlify DB* → create/claim a database. It sets `NETLIFY_DATABASE_URL` (pooled) and `NETLIFY_DATABASE_URL_UNPOOLED` automatically — the app reads them; you don't need `DATABASE_URL`. *(Claim the database into your own Neon account so it isn't deleted after the trial period.)*
   * **Your own Neon project**: create a project at neon.tech, copy the **pooled** connection string (host contains `-pooler`) and set it as `DATABASE_URL`.
4. **Environment variables** (Site configuration → Environment variables; mark secrets as *Secret*):

   | Variable | Required | Notes |
   |---|---|---|
   | `JWT_SECRET` | **yes** | 32+ random chars: `node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"`. Functions refuse to issue sessions without it. |
   | `DATABASE_URL` | if not using Netlify DB | Neon *pooled* string |
   | `SITE_URL` | recommended | Your canonical domain, e.g. `https://www.allensautorefinement.com` (defaults to Netlify's `URL`) |
   | `SEED_DEMO` | optional | `true` on the first deploy to add 3 clearly-labeled demo listings |
   | `RESEND_API_KEY` **or** `SMTP_HOST/PORT/SECURE/USER/PASS` + `NOTIFY_FROM` + `NOTIFY_TO` | optional | Lead email notifications. Unset = disabled (leads still land in the portal). |

5. **Deploy.** Production builds run `npm run build`, which also **applies pending migrations** when a database URL is present (`MIGRATE_ON_BUILD`, default on; deploy previews and branch deploys never migrate — see `netlify.toml`). To run them yourself instead: set `MIGRATE_ON_BUILD=false` and run `DATABASE_URL=<unpooled string> npm run migrate` from your machine.
6. **First login:** open `https://<site>/admin`, sign in as **Shane** with the password that matches the bcrypt hash provided for this project (it is not stored anywhere in this repo). Then: *My account → Change password*, and add other staff under *Staff users* (admin role only). If the password is ever lost: `DATABASE_URL=… NEW_USER_PASSWORD='…' npm run create-user -- --username shane2 --role admin`.
7. **Fill in `config/business.json`** (and `about`, `legal`, `asIsDisclosure`), commit, and Netlify redeploys automatically.
8. **Custom domain**: Domain management → add domain → follow DNS instructions; Netlify provisions TLS (Let's Encrypt) automatically. Then set `SITE_URL` to it and redeploy.
9. Remove the demo listings when real inventory is in: `DATABASE_URL=… npm run clear-demo` (or delete them in the portal).

### Option B — Netlify CLI

```bash
npm i -g netlify-cli          # or prefix commands with npx netlify-cli
netlify login
netlify init                  # create/link the site
netlify env:set JWT_SECRET "$(node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))")"
netlify env:set DATABASE_URL "postgresql://…-pooler…/neondb?sslmode=require"
netlify deploy --build        # draft deploy (unique URL) to check
netlify deploy --build --prod # go live
```

### After deploying

* `npm run smoke -- https://<site>` — read-only check of every page, sitemap, robots, health and the 404 page.
* `https://<site>/api/v1/health` → `{"status":"ok", "db":{"ok":true,"schemaVersion":2}, …}`; `?probe=live` skips the DB.

## CI/CD, previews and rollbacks

* **CI**: `.github/workflows/ci.yml` runs `npm ci` → lint → build → tests → `npm audit` on every PR and push to `main`.
* **Deploy Previews**: once the repo is linked, Netlify builds every PR at its own URL and posts a status check. Previews never run migrations (they share whatever DB they're pointed at — for full isolation, give the `deploy-preview` context its own `DATABASE_URL`, e.g. a Neon branch).
* **Instant rollback**: Netlify → Deploys → pick any earlier deploy → **Publish deploy**. Static files and functions roll back atomically. Database migrations are additive by convention so older code keeps working with a newer schema; see `docs/RUNBOOK.md`.
* **Atomic deploys**: Netlify switches all files + functions at once (effectively blue-green). No partially-updated site is ever served.

## Local development

```bash
nvm use                         # Node 20 (see .nvmrc)
npm ci
cp .env.example .env            # set JWT_SECRET; DATABASE_URL=pglite:./.pglite works with zero setup
npm run build
DATABASE_URL=pglite:./.pglite SEED_DEMO=true npm run migrate
npm run serve:local             # http://localhost:8888 — Netlify-like routing, headers, functions
#   or the real emulator:  netlify dev   (uses your linked site's env; supports Blobs + functions)
npm test                        # 36 tests: build output, SEO, CSP, icons/manifest, forms, auth, CRUD, locking, DLQ, backups
npm run lint
```

Optional local Postgres: `docker compose up -d` then `DATABASE_URL=postgres://aar:aar@localhost:5432/aar`. Docker is **only** for local development — Netlify doesn't run containers.

Create a local staff account (your own password, never committed):
`DATABASE_URL=pglite:./.pglite NEW_USER_PASSWORD='choose-a-long-one' npm run create-user -- --username me --role admin`

## Operations

* **Logs**: every function logs one JSON object per line (`ts, level, msg, fn, requestId, method, path, status, durMs`), secrets redacted; responses carry `X-Request-Id`. View in Netlify → Logs → Functions, or stream to Datadog/New Relic/Axiom/S3 with **Log Drains** (Pro plan+).
* **Monitoring & alerts**: point an uptime monitor (UptimeRobot, Better Stack, etc. — free tiers exist) at `/api/v1/health` (readiness) every 1–5 min; alert on non-200. In your log drain, alert on `level:"error"`, `msg:"backup_failed"`, `msg:"dlq_dead_letters_present"`, `msg:"refresh_token_reuse_detected"`. See `docs/RUNBOOK.md` for suggested SLOs.
* **Backups**: nightly function writes a JSON export of all tables to the `backups` Blobs store and keeps 30. Neon also provides point-in-time restore. Restore steps: `docs/RUNBOOK.md`.
* **Emails**: inline send with 3 s timeout + retry with backoff/jitter + circuit breaker; failures are queued in `failed_notifications` and retried hourly (up to 8 attempts, exponential backoff), then marked `dead`.
* **Feature flags**: `config/features.json` (`showInventory`, `showSellForm`, `showServices`, `showFeaturedOnHome`, `acceptSellPhotos`). Static pages read them at build; functions also honor runtime env overrides, e.g. `FEATURE_SHOW_SELL_FORM=false` (static pages need a redeploy to hide nav links).
* **Demo data**: `npm run seed-demo` / `npm run clear-demo`. Demo rows are `is_demo = true`, show a "Demo listing" badge and banner, are `noindex`, excluded from the sitemap and from Car/Offer structured data.

## Limits & known limitations (honest list)

* **Node 20 is past end-of-life (April 2026).** It's pinned as requested, which forces older-but-supported Netlify SDKs (`@netlify/functions` 5.3, `@netlify/blobs` 10.7 — the newest majors need Node ≥ 22.12). Recommended: change `.nvmrc`, `NODE_VERSION` in `netlify.toml` and `engines` to 22, then let Dependabot upgrade the SDKs (their major-version ignores are in `.github/dependabot.yml`).
* **Not deployed from here.** I could not run `netlify dev` or a real deploy in the build sandbox (no Netlify account/CLI login). Verification used Netlify's own bundler (`@netlify/zip-it-and-ship-it`, which bundled all 15 functions and statically parsed every path, rate limit and schedule), a local runtime emulator (`scripts/local-server.mjs`) and direct invocation tests against PGlite (real Postgres compiled to WASM). `purgeCache`, edge `rateLimit` and real Netlify Blobs only execute on Netlify; they're no-ops/emulated locally.
* **Visual QA wasn't possible in the sandbox** (no headless browser available for this CPU architecture). Markup, CSP compliance, labels and SEO tags are checked by tests; do a manual pass in a browser (and an axe/Lighthouse run) before launch.
* Edge rate limiting via `config.rateLimit` depends on your Netlify plan's rate-limiting availability; the DB-backed per-IP throttle (10-minute windows) always applies as backup.
* The email circuit breaker's state is per warm function instance (resets on cold start) — fine at this scale; the DLQ guarantees nothing is lost.
* Public forms need JavaScript (for the CSRF/time-trap token and photo resizing); a `<noscript>` message points people to phone/email. Inventory browsing and filtering work fully without JavaScript.
* Sell-form uploads: up to 6 photos, resized in the browser to ≤1600 px, total request ≤ 5.5 MB (Netlify's synchronous function body limit is ~6 MB). Staff uploads go one photo per request (≤ 4 MB each).
* The homepage "Featured inventory" block is fetched client-side from `/inventory/featured` (edge-cached) so the static homepage never shows stale cars.

## Legal & compliance — owner action required

* **Have a Virginia-licensed attorney review** `/privacy` and `/terms` (drafted around the VCDPA, Virginia governing law, cash offers being preliminary until inspection, as-is disclosure placeholder, limitation of liability) before launch.
* **Confirm Virginia Motor Vehicle Dealer Board (MVDB) licensing requirements** for buying and selling vehicles (dealer certificate, salesperson licensing, advertising rules such as displaying the dealer name/license, FTC Used Car Rule Buyers Guide on each vehicle offered for sale). Put the license number in `config/business.json → dealerLicense`.
* Fill `asIsDisclosure` with the exact as-is / warranty language your attorney approves.

## Ambient background

Every page (via `layout()`; the admin shell uses a calm variant) carries a fixed, decorative `.ambient` layer. With JS + WebGL and no reduced-motion preference, `public/js/ambient.js` is lazy-loaded after the page has loaded and renders a slow "liquid clearcoat" shader behind the content; otherwise visitors see a static (or CSS-animated) gradient. Mood per page comes from the path (`pageMood()`); sections can override it with `data-mood="neutral|warm|steel|bright"`. Tuning knobs (colours, speeds, fps, resolution) are at the top of `ambient.js`.

## Fonts & images

Bricolage Grotesque and Inter are self-hosted under the SIL Open Font License 1.1 (see `public/fonts/LICENSE.txt`). Site photos are the project's own images (`public/images/*.png`), converted at build to responsive WebP (480/960/full).

**Logo & favicons.** The official circular badge is `public/images/logo.png` (1024², transparent outside the circle). The build turns it into AVIF + WebP at 48/96/192/384 px (header 48 px, footer 96 px, admin sidebar 44 px, admin login 88 px), a 512 px PNG for the JSON-LD `logo`, and a 1200×630 Open Graph/Twitter card (badge on brand navy) used as the default `og:image` — vehicle pages still use their own photo. Favicons and `site.webmanifest` live in `public/icons/`: the build copies them to the publish root (`/favicon.ico`, `/apple-touch-icon.png`, … — modest cache) and to fingerprinted `/assets/icons/*` copies that every `<head>` links to (cached a year, immutable). The shared links come from `iconLinks()` in `src/views/head.mjs` (theme colour `#1b3a5c`). To change the logo, replace those files and redeploy; hashes update automatically. On the first page of a visit the header badge scales in with a light sweep (skipped under reduced motion).
