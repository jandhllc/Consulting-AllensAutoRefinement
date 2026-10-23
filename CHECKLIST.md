# Engineering checklist → how this project handles it

Legend: ✅ implemented here · 🟦 provided by the platform (Netlify / Neon) and configured · ➖ N/A, with the reason. File references are relative to the repo root.

## Traffic, edge & delivery
| Item | Status | How / why |
|---|---|---|
| Rate limiting | ✅ | Netlify `config.rateLimit` (per IP + domain) on login (10/min), sell (5/min), other forms (10/min), auth (60/min); plus DB-backed per-IP fixed-window throttle (`request_throttle`, 10-min buckets) in `src/lib/forms.mjs` as backup. |
| Caching | ✅ | Hashed assets `/assets/*` cached 1 yr immutable; inventory/sitemap `Netlify-CDN-Cache-Control: s-maxage, stale-while-revalidate, durable`; browsers always revalidate HTML. |
| Load balancing | 🟦 | Netlify's edge network and function platform distribute requests; nothing to run. |
| Reverse proxies | 🟦 | Netlify edge is the proxy. Client IP read from `context.ip` / `x-nf-client-connection-ip`, never trusted from arbitrary headers first. |
| API gateways | 🟦 | Netlify routing (`config.path`, methods, rate limits) acts as the gateway; versioned under `/api/v1`. |
| CDN | 🟦 | Netlify CDN serves `dist/` and cached function responses. |
| Edge caching | ✅ | Durable cache for inventory pages, featured fragment, sitemap and `/media/*` photos (`src/lib/cache.mjs`). |
| Cache invalidation | ✅ | Tag-based: `Netlify-Cache-Tag: inventory, vehicle-<id>, sitemap`; every staff write calls `purgeCache({ tags })`. Static HTML is invalidated by each atomic deploy. TTL fallback if a purge fails. |
| HTTP/2 / HTTP/3 | 🟦 | Served by Netlify's edge automatically. |
| TLS | 🟦 | Automatic Let's Encrypt certs; HSTS `max-age=2y; includeSubDomains; preload`; `upgrade-insecure-requests` in CSP. |
| DNS | 🟦 | Netlify DNS or your registrar → Netlify; documented in README. |
| WAF | 🟦 / ➖ | Netlify's platform protections + strict input validation here. A managed WAF ruleset is an enterprise add-on — not needed for a small dealer site. |
| DDoS | 🟦 | Absorbed by Netlify's edge network; rate limits reduce function cost exposure. |
| TCP vs UDP | ➖ | Plain HTTPS (TCP; HTTP/3 over QUIC/UDP handled by the edge). No custom transports. |
| gRPC | ➖ | Browser + simple JSON API; gRPC adds tooling with no benefit here. |
| WebSockets | ➖ | No real-time features; Netlify Functions don't hold sockets. |
| Long polling | ➖ | Nothing needs push updates; the portal refreshes on navigation. |
| SSE | ➖ | Same — no live feeds required. |
| Webhooks | ➖ | No inbound integrations yet. If added (e.g. email provider bounces), verify signatures + idempotency keys. |

## Reliability & resilience
| Item | Status | How / why |
|---|---|---|
| Circuit breakers | ✅ | `CircuitBreaker` around the email provider (opens after `EMAIL_BREAKER_THRESHOLD` failures, cool-down, half-open trial) — `src/lib/retry.mjs`, `src/lib/notify.mjs`. Per warm instance by design. |
| Timeouts | ✅ | Every DB call wrapped in a hard timeout (`DB_TIMEOUT_MS`, default 4 s; Neon fetch also gets an `AbortSignal`); email 3 s; all well under Netlify's 10 s sync limit. |
| Retries | ✅ | Bounded retries with a deadline (`retry()`), client form retry reuses the same idempotency key. |
| Exponential backoff | ✅ | Full-jitter backoff for inline retries and the hourly DLQ (`backoffMs`). |
| Idempotency | ✅ | Client UUID `idempotency_key` with UNIQUE constraint + `ON CONFLICT DO NOTHING` on every lead table; pre-check skips photo uploads on retries; "Accept & add to inventory" guarded by `vehicle_id IS NULL`. |
| Message queues | ➖ / ✅ | No broker needed at this scale; the `failed_notifications` table is a durable work queue drained by a scheduled function. |
| Pub/sub | ➖ | Single consumer per event; no fan-out needed. |
| Event-driven | ✅ (light) | Lead created → background notification (`context.waitUntil`); inventory change → cache-purge event. |
| Distributed transactions | ➖ | One database. Multi-step writes are single atomic SQL statements (data-modifying CTEs) or one transaction. |
| Saga | ➖ | Only cross-system flow is DB + Blobs for photos: compensating delete of blobs if the DB insert fails (`api-admin-photos.mjs`). |
| Dead letter queues | ✅ | `failed_notifications` (pending → sent / dead after 8 attempts); `dead` count logged as `dlq_dead_letters_present` for alerting. |
| Cron | ✅ | Netlify Scheduled Functions: `retry-notifications` (`@hourly`), `nightly-backup` (`0 7 * * *` UTC). |
| Health checks | ✅ | `/api/v1/health` (readiness: DB ping + schema version) and `?probe=live` (liveness). |
| Liveness / readiness | ✅ | As above; 503 when the DB is unreachable. |
| Failover | 🟦 | Netlify serves static pages even if the DB is down; Neon handles storage-layer failover. Inventory serves stale-while-revalidate copies from the CDN during DB blips. |
| Multi-region | ➖ / 🟦 | Static + cached pages are global. Functions/DB run in one region — appropriate for a single-location local business. |
| Disaster recovery (DR) | ✅ | Nightly logical backup to Blobs (independent of the DB provider) + Neon point-in-time restore + `npm run restore-backup`. RPO ≤ 24 h (Neon PITR: minutes), RTO ~30 min. `docs/RUNBOOK.md`. |
| Backups | ✅ | 30 nightly JSON exports retained (`src/lib/backup.mjs`); refresh tokens deliberately excluded. |
| Chaos engineering | ➖ | Overkill for this size. Failure paths (DB timeout, provider 5xx, breaker open, token reuse) are exercised by tests instead. |
| Network partitions | ✅ (bounded) | Timeouts + idempotent retries; CAP choice below. |
| Backpressure | ✅ | Rate limits, body-size caps (64 KB JSON, 5.5 MB sell form, 4 MB photo), DLQ drains ≤25/run and stops when the breaker opens. |

## Data
| Item | Status | How / why |
|---|---|---|
| DB indexing | ✅ | `migrations/001_initial.sql`: unique `slug`, indexes on `status`, `lower(make)`, `price`, `created_at`, partial index for the public listing, lead status/date indexes. |
| Query optimization | ✅ | Public list = one query with `LATERAL` cover photo + `count(*) OVER()`; facets in one query; filters whitelisted. |
| N+1 | ✅ | No per-row queries: photos via `LATERAL`/`json_agg`, notes + authors via one `json_agg` join. |
| Connection pooling | 🟦 / ✅ | Neon pooled endpoint over HTTP (no per-invocation TCP pool); `pg` pool capped at 2 for non-Neon Postgres. |
| Read replicas | ➖ | Read load is absorbed by the CDN; Neon read replicas can be added later without code changes (`DATABASE_URL` per function if ever needed). |
| Sharding | ➖ | Data volume of one dealership fits comfortably in one Postgres. |
| Partitioning | ➖ | Same; `audit_log`/`request_throttle` are pruned instead (hourly job). |
| Replication | 🟦 | Neon replicates storage internally. |
| Leader election | ➖ | No long-running workers; scheduled functions are single-invocation by the platform. |
| CAP | ✅ (decision) | CP for writes (single Postgres primary, strong consistency); AP-ish for reads via CDN stale-while-revalidate. |
| Eventual consistency | ✅ (bounded) | Edge cache may lag a write by the purge propagation time (seconds) or TTL (5 min) if a purge fails. Blobs use `consistency: 'strong'`. |
| Optimistic locking | ✅ | `vehicles.version`; `UPDATE … WHERE id = $1 AND version = $2`; 409 + current record on conflict; UI offers "Load latest". Tested with 3 concurrent writers. |
| Pessimistic locking | ✅ (narrow) | `SELECT … FOR UPDATE` inside the single-statement lead-accept CTE; otherwise avoided. |
| Distributed locks | ➖ | Not needed: conditional updates and unique constraints provide mutual exclusion. |
| Race conditions | ✅ | Atomic increments for login failures/throttle, conditional refresh-token claim, version checks, unique idempotency keys, guarded accept. |
| Deadlocks | ✅ (by design) | Each write touches rows in one statement; no multi-statement lock ordering issues. Postgres would abort and the client sees a 500 + can retry. |
| Migrations | ✅ | `migrations/NNN_*.sql`, `npm run migrate`, auto on production builds; each migration runs in its own transaction. |
| Schema versioning | ✅ | `schema_version` table; reported by `/api/v1/health`. |
| Encryption at rest | 🟦 | Neon and Netlify Blobs encrypt at rest. Passwords bcrypt (cost 12); refresh tokens SHA-256; IPs stored as keyed HMAC. |
| Encryption in transit | 🟦 / ✅ | HTTPS everywhere + HSTS; Neon requires TLS (`sslmode=require`). |

## Runtime & scaling
| Item | Status | How / why |
|---|---|---|
| Memory leaks | ✅ (low risk) | Stateless functions; only module-level caches are the DB driver and breaker. Object URLs revoked in the browser. |
| GC | 🟦 | Node's GC; short-lived invocations. |
| Thread safety | ➖ | Node is single-threaded per instance; concurrency safety is handled at the DB layer (see race conditions). |
| Autoscaling | 🟦 | Functions scale per request automatically. |
| Horizontal scaling | 🟦 | Same; no shared in-memory state required for correctness. |
| Vertical scaling | 🟦 / ➖ | Neon compute autoscaling; function memory is fixed by the plan. |
| Cold starts | ✅ | 15 small single-purpose functions (~0.4–1 MB zipped); `sharp` only bundled into the two image functions; PGlite never bundled; HTTP DB driver avoids TCP handshakes. |
| Serverless limits | ✅ | Sync timeout 10 s (all work bounded ≤ ~6 s), ~6 MB request body (client-side resize, one staff photo per request), scheduled function 30 s (DLQ batch 25). |
| Latency | ✅ | Static/edge-cached HTML; responsive WebP; preloaded self-hosted fonts; deferred JS; ambient WebGL background lazy-loaded after `load`/idle, 30 fps cap, reduced resolution, paused on hidden tabs. |
| Throughput | 🟦 | Platform-scaled; CDN absorbs read bursts. |
| p99 / tail latency | ✅ (mitigated) | Durable edge cache + stale-while-revalidate hides cold starts/DB latency for public pages; `durMs` logged per request for percentile tracking in a log drain. |
| Clock skew | ✅ | JWT verification allows 30 s leeway; time-trap and lockouts use server/DB time (`now()`), not client time. |
| Cost optimization | ✅ | Static pages cost nothing per view; CDN caching avoids function/DB hits; no paid services required; Neon scales to zero. |

## Delivery & operations
| Item | Status | How / why |
|---|---|---|
| CI/CD | ✅ | `.github/workflows/ci.yml` (lint, build, test, audit) + Netlify continuous deployment. |
| Docker | ➖ (dev only) | Netlify doesn't run containers. `docker-compose.yml` offers a local Postgres only. |
| Kubernetes | ➖ | No cluster to run; the platform is serverless. |
| Helm | ➖ | No Kubernetes. |
| Terraform | ➖ (optional) | `netlify.toml` + function configs are the IaC. A Netlify Terraform provider exists if you later want env vars/domains as code. |
| IaC | ✅ | `netlify.toml` (build, functions, redirects, headers, contexts) and in-source `export const config` per function. |
| Service discovery | ➖ | Routes are declared paths; no services to discover. |
| Feature flags | ✅ | `config/features.json` + runtime `FEATURE_*` overrides (`src/lib/site.mjs`). |
| Blue-green | 🟦 | Netlify atomic deploys swap all files + functions at once. |
| Canary | 🟦 / ➖ | Netlify Split Testing can route a % of traffic to a branch deploy; usually unnecessary here. |
| Rolling deploys | ➖ | Atomic deploys make rolling unnecessary. |
| Rollbacks | 🟦 | Deploys → Publish any previous deploy (instant). Migrations are additive so rollbacks stay compatible. |
| Build caching | ✅ | Netlify caches `node_modules`; image variants cached in `node_modules/.cache/aar-images` by source hash (second build: 15/15 cached). |
| Dependency hell | ✅ | Committed lockfile, `npm ci`, `.nvmrc` + `NODE_VERSION=20`, `engines`, Dependabot with grouped updates and major-version holds for SDKs that need Node 22. |
| Semver | ✅ | `package.json` version + `CHANGELOG.md`. |
| API versioning | ✅ | All endpoints under `/api/v1`; health reports `apiVersion`. |
| Monitoring | ✅ / 🟦 | Health endpoint for uptime monitors; Netlify function metrics/logs. |
| Logging | ✅ | Structured JSON lines with redaction (`src/lib/log.mjs`). |
| Tracing | ✅ (light) | Request ID per invocation (`x-nf-request-id` or UUID), returned as `X-Request-Id` and on every log line. Full distributed tracing is N/A for one hop. |
| Metrics | ✅ / 🟦 | `durMs`, `status`, error/lead/backup events in logs → derive metrics in the drain; Netlify function analytics. |
| Alerts | ✅ (documented) | Suggested alert rules in README + `docs/RUNBOOK.md`. |
| SLOs / SLIs | ✅ (documented) | `docs/RUNBOOK.md`: availability 99.5% monthly for public pages; form success ≥ 99%; p95 inventory < 800 ms. |
| Error budgets | ✅ (documented) | 0.5% ≈ 3.6 h/month; freeze non-urgent changes when exhausted. |
| Observability | ✅ | Logs + health + request IDs + audit log table. |
| Production incidents | ✅ | `docs/INCIDENT_RESPONSE.md`. |
| On-call | ✅ (lightweight) | Owner/designated staff receive uptime alerts; escalation list in the incident doc. |
| Postmortems | ✅ | Blameless template in `docs/INCIDENT_RESPONSE.md`. |

## Security
| Item | Status | How / why |
|---|---|---|
| Secrets | ✅ | Only env vars (`JWT_SECRET`, DB URL, SMTP/Resend); `.env.example` only; logs redact secret-like keys; functions refuse to sign tokens without `JWT_SECRET` in production. |
| IAM | ✅ | Roles `admin` / `staff`; admin-only user management; disable revokes sessions immediately; audit log of writes. |
| OAuth | ➖ | Staff use first-party accounts; no third-party sign-in required (could add later). |
| JWT rotation | ✅ | 15-min HS256 access JWT + rotating refresh token (hashed in DB, 30-min idle, 12-h absolute); reuse of a rotated token revokes the family; password change bumps `password_version` and invalidates old JWTs. |
| CORS | ✅ | Same-origin only: no `Access-Control-Allow-*` headers ever, preflights refused (403). |
| CSRF | ✅ | Double-submit token (`aar_csrf` cookie ↔ header/field, timing-safe compare) + strict Origin/Referer check on every state-changing request; cookies `SameSite=Strict`. |
| SQLi | ✅ | Parameterized queries only; dynamic SQL limited to whitelisted column/sort names. |
| XSS | ✅ | Auto-escaping template tag server-side; admin UI builds DOM with `textContent`; strict CSP with no inline scripts/styles; JSON-LD `<`-escaped. |
| SSRF | ✅ | The server never fetches user-supplied URLs; only fixed provider endpoints (Resend). |
| File uploads | ✅ | Magic-byte sniffing, size/count caps, sharp re-encode to WebP (strips EXIF/GPS, defeats polyglots), pixel limit, random keys, lead photos private behind auth. |
| Brute force | ✅ | Edge rate limit + DB throttle + lockout after 5 failures (15 min) + constant-time dummy compare for unknown users. |
| Spam | ✅ | Honeypot + signed time-trap (≥3 s, ≤6 h) with indistinguishable responses for bots. |
| Headers | ✅ | CSP, HSTS preload, X-Frame-Options DENY, nosniff, Referrer-Policy, Permissions-Policy, COOP/CORP — identical for static (netlify.toml) and functions (test enforces no drift). |
