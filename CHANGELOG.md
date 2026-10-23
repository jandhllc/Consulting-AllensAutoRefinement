# Changelog
All notable changes to this project are documented here. Versions follow [Semantic Versioning](https://semver.org/).
API endpoints are versioned separately under `/api/v1`; breaking API changes will ship as `/api/v2` alongside v1.

## [1.2.0] — 2026-10-22
### Added
- Official Allen's Auto Refinement badge logo (`public/images/logo.png`) replaces the drawn "A" mark: header nav (48 px, 44 px on phones, wordmark beside it from 600 px up), footer (96 px), admin login (88 px) and admin sidebar (44 px); alt text "Allen's Auto Refinement logo". Built to AVIF + WebP at 48/96/192/384 px via the existing sharp pipeline (cached between builds).
- Favicons + web app manifest (`public/icons/`): `/favicon.ico` (16/32/48), SVG, 16/32/48 PNG, 180 px apple-touch-icon, 192/512 Android icons and `/site.webmanifest` at the publish root, plus fingerprinted copies under `/assets/icons/`.
- Shared `iconLinks()` head block (`src/views/head.mjs`) on every document — prerendered pages, 404/500, thank-you, admin shell and function-rendered inventory pages: SVG / 32 / 16 / `.ico` icon links, apple-touch-icon, manifest and `theme-color` `#1b3a5c` (replaces the old `#0b0b0d`).
- Page-load intro: on the first page of a visit the header badge scales in followed by a soft light sweep (also on hover/focus); the admin login badge does the same. Disabled under `prefers-reduced-motion` and forced colours; CSS only, no layout shift.
- Default `og:image` / `twitter:image` is now a 1200×630 logo card (absolute URL from `SITE_URL`) with width/height/alt; vehicle pages keep their own photo. AutoDealer JSON-LD `logo` is a 512 px PNG `ImageObject`; `image` lists the logo then the hero photo.
- Tests (36): every page — public, admin, 404/500, thank-you, function-rendered `/inventory`, `/inventory/:slug` and the function 404 — has the icon links, manifest and a single theme-color, and every linked file exists; manifest icons exist; `favicon.ico` is a real 3-size ICO; header/footer badge markup + alt; OG/JSON-LD logo URLs; cache-header rules.
### Changed
- `netlify.toml`: `/favicon.ico` and `/site.webmanifest` cached 1 day (+ explicit `Content-Type`), other unhashed root icons 1 week — never immutable; hashed logo/icon copies fall under the existing `/assets/*` 1-year immutable rule. CSP unchanged (`manifest-src 'self'` and `img-src 'self'` already cover it).
- Local server serves `.ico`, `.webmanifest` and `.avif` with correct MIME types; smoke test also checks `/favicon.ico` and `/site.webmanifest`.
### Removed
- Build-generated placeholder `favicon.svg` and the hero-photo Open Graph JPEG.

## [1.1.0] — 2026-10-22
### Added
- Ambient "liquid clearcoat" background on every page via the shared layout (`ambientLayer()` in `src/views/layout.mjs`; admin shell gets a calm variant): fixed full-viewport WebGL fragment shader (`public/js/ambient.js`, external ES module, fingerprinted) — midnight/graphite paint with domain-warped flow (~45 s drift), faint iridescent metallic flakes, two broad showroom-light reflections gliding across (~38 s / ~55 s), flowing aero contour lines, cursor-following glow (fine pointers only), fine film grain (CSS-scrolled noise tile).
- Per-page and per-section moods with ~1.4 s crossfades driven by IntersectionObserver (`data-mood`: warm near Services/Restoration, steel blue near Inventory/Sell, bright near Protection; page default from the path).
- Extras: light-beam dust motes in the home hero (2D canvas, runs only while the hero is visible), hydrophobic water-bead shimmer on the Protection cards, animated gradient seams between sections and above the footer, soft scrims behind text-heavy blocks.
- Test coverage: every prerendered page and function-rendered inventory page includes the layer and points at the built module; module is never loaded eagerly (33 tests).
### Performance & accessibility
- Module is lazy-loaded by `site.js` / `admin.js` after `load` + idle (does not affect LCP); renders at ½ CSS-pixel resolution (admin ⅓·, ~6 fps), capped at 30 fps, pauses on hidden tabs, stops on WebGL context loss; falls back to an animated CSS gradient without WebGL and to a static gradient with JS off, `prefers-reduced-motion`, Save-Data, forced colors and print. `pointer-events: none`, `position: fixed`, `contain: strict` — no layout shift.
- Shader output is soft-capped at ~12 % sRGB so all site text colours keep ≥ 4.5:1 contrast on top of it.
- No CSP change was needed (WebGL, CSSOM style writes and the `data:` grain tile are already allowed by the existing policy).

## [1.0.0] — 2026-10-22
### Added
- Netlify architecture: prerendered static marketing pages (`/`, `/services`, `/services/detailing|restoration|protection`, `/sell`, `/about`, `/contact`, `/privacy`, `/terms`, `/accessibility`, `404`, `500`) with unique meta, canonical, Open Graph/Twitter and JSON-LD.
- Server-rendered, edge-cached `/inventory` (query-string filters/sort/pagination) and `/inventory/:slug` (gallery, specs, Car + Offer JSON-LD, inquiry/viewing form); tag-based cache purge on staff edits.
- Dynamic `/sitemap.xml` (static pages + every published available/pending vehicle) and `robots.txt`.
- Staff portal at `/admin`: JWT access + rotating refresh tokens, roles, lockout, password change, staff users; dashboard KPIs; inventory CRUD with optimistic locking, statuses, mark sold, photos (Netlify Blobs, sharp re-encode), CSV export; cash-offer leads with "Accept & add to inventory"; buyer inquiries; service quotes; notes.
- Public forms with CSRF double-submit + Origin check, honeypot, time-trap, zod validation, idempotency keys, edge + DB rate limits; sell form with multi-step UI and photo upload.
- Postgres via Neon serverless driver; versioned migrations + `schema_version`; Shane seeded from the provided bcrypt hash.
- Email notifications (Resend or SMTP) with timeouts, backoff + jitter retries, circuit breaker, dead-letter queue and hourly scheduled retry; nightly JSON backups to Blobs (30 retained).
- Health check, structured JSON logs with request IDs, feature flags, CI workflow, Dependabot, docs (README, CHECKLIST, RUNBOOK, incident response + postmortem template).
- Demo inventory (3 vehicles, `is_demo`, labeled "Demo listing", noindex, excluded from sitemap/structured data) with `npm run seed-demo` / `npm run clear-demo`.
