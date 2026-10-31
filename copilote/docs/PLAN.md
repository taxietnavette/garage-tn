# Copilote — Implementation Plan (MVP)

Status: **DRAFT — awaiting approval** (no code written yet)
Spec: `docs/SPEC.md` (the build prompt). Decisions & defaults: `docs/DECISIONS.md`.

---

## 1. Architecture overview

```
                ┌───────────────────────┐      ┌──────────────────────────┐
 Driver app     │  apps/driver (Expo)   │      │ apps/console (Next.js)   │  Dispatcher / owner
 iOS + Android  │  outbox (SQLite)      │      │ + /book/[slug] rider page│  Riders (web)
                └──────────┬────────────┘      └────────────┬─────────────┘
                 HTTPS (idempotent REST) + Socket.IO (events, heartbeats)
                           │                                │
                ┌──────────▼────────────────────────────────▼─────────────┐
                │ apps/api  — Fastify + Zod + Socket.IO (Redis adapter)    │
                │  auth · tenancy · bookings · queue · offers · dispatch   │
                │  fares · notifications · privacy · metrics               │
                └──────┬───────────────────┬────────────────────┬─────────┘
                       │ Drizzle            │ BullMQ / pub-sub    │ HTTP (2 s timeout)
              ┌────────▼────────┐  ┌────────▼────────┐   ┌───────▼────────────────┐
              │ PostgreSQL 16 + │  │ Redis 7         │   │ services/intel (Python)│
              │ PostGIS 3       │  │ jobs, pub/sub,  │   │ FastAPI: LightGBM      │
              │ source of truth │  │ rate limits     │   │ forecast + OR-Tools VRP│
              └────────▲────────┘  └────────▲────────┘   └────────────────────────┘
                       │                    │
                ┌──────┴────────────────────┴──────┐
                │ apps/worker — BullMQ processors   │  offer timeouts, lease sweeper,
                │                                   │  scheduled-dispatch release, SMS,
                └───────────────────────────────────┘  forecast refresh/retrain, retention
```

Principles
- **Pure domain cores, thin adapters.** All critical logic (queue/lease state machine, dispatch selection, fare calc, motion/safety lock, voice grammar, forecast baseline) lives in framework-free TS packages with an injected clock. They are exhaustively unit-tested without DB/network. API/worker/apps only wire them to I/O.
- **Postgres is the source of truth** for queue, offers, bookings (transactional, `SELECT … FOR UPDATE` per zone). Redis is for jobs, pub/sub fan-out, rate limiting — losing Redis never loses a queue position.
- **Every write idempotent** (`Idempotency-Key` header / `actionId` in the outbox), every state transition audited in the same transaction.
- **Providers behind interfaces** with dev/console implementations: `NotificationProvider` (Twilio | console), `GeocodingProvider`, `RoutingProvider` (ETA), `LlmProvider` (Claude | null), `FareCalculator`, `ComplianceProvider` (no-op + TODO), `PaymentProvider` (stub), `ForecastProvider` (TS baseline | intel service).

## 2. Repository layout

> Proposed location: this repo's `copilote/` folder **or** a fresh repo — see open question Q1 at the bottom.

```
copilote/
├─ apps/
│  ├─ api/                  Fastify server (REST + Socket.IO)
│  │  ├─ src/modules/{auth,companies,zones,drivers,vehicles,shifts,queue,
│  │  │               bookings,offers,dispatch,fares,notifications,voice,
│  │  │               forecast,contracts,reports,privacy,audit}/
│  │  ├─ src/plugins/       tenancy, auth/rbac, idempotency, rate-limit, request-id, errors
│  │  ├─ src/realtime/      socket gateway, rooms (company, zone, driver, booking)
│  │  └─ test/              Supertest integration (Testcontainers Postgres+Redis)
│  ├─ worker/               BullMQ processors (same modules, no HTTP)
│  ├─ console/              Next.js App Router + Tailwind + shadcn/ui + MapLibre
│  │  └─ app/[locale]/(dash)/{map,bookings,drivers,vehicles,zones,queues,
│  │                          contracts,reports,settings}  + app/book/[slug]
│  └─ driver/               Expo (dev client) + Expo Router
│     └─ src/{app,features/{auth,shift,queue,offer,trip,earnings,coach,settings},
│             copilot/{motion,voice,tts,stt},outbox,sync,store,i18n}
├─ packages/
│  ├─ domain/               shared types, enums, Zod schemas (API contract)
│  ├─ queue-core/           lease/grace state machine + ordering (pure)
│  ├─ dispatch-core/        candidate selection, eligibility, follow-on rules (pure)
│  ├─ fare-core/            FareCalculator default impl (pure)
│  ├─ motion-core/          moving/stopped hysteresis detector (pure)
│  ├─ voice-core/           fr/en fixed grammar, normalization, intent schema (pure)
│  ├─ forecast-core/        seasonal × smoothing baseline, Québec holidays (pure)
│  ├─ db/                   Drizzle schema, migrations, seed, RLS policies
│  ├─ i18n/                 i18next resources fr-CA (default) / en-CA + helpers
│  ├─ sdk/                  typed API + socket client used by console, driver, simulate
│  └─ config/               eslint, tsconfig, prettier, vitest presets
├─ services/intel/          Python 3.12 FastAPI: /forecast/train, /forecast/predict, /vrp/solve
├─ scripts/simulate/        fleet simulator + KPIs; also used for load test (M7)
├─ infra/                   docker-compose.yml, Dockerfiles, OSRM profile (optional)
├─ docs/                    SPEC.md, PLAN.md, DECISIONS.md, PRIVACY.md, SECURITY.md, DEMO.md
├─ .github/workflows/ci.yml lint · typecheck · unit · integration · (M7) e2e
└─ .env.example
```

## 3. Data model (Drizzle, PostgreSQL + PostGIS)

All tenant tables carry `company_id` (FK, indexed) and are covered by RLS (`company_id = current_setting('app.company_id')`) as defense-in-depth behind app-level scoping. IDs are UUIDv7 (time-ordered). Timestamps `timestamptz`, stored UTC; company timezone used only for display / forecast buckets.

| Table | Key columns | Notes |
|---|---|---|
| `company` | id, slug, name, timezone (`America/Toronto`), default_locale (`fr-CA`), settings jsonb | settings (Zod-validated, versioned): `dispatch {strategy, offerTimeoutSec=15, maxOfferAttempts, scheduledLeadMin=15, fallbackNearestMaxEtaMin}`, `queue {heartbeatSec=10, leaseSec=30, graceMin=3}`, `followOn {neverWhileOnTrip=false, maxDistanceM=500, maxMinutesToDropoff=3}`, `motion {movingKmh=8, movingSec=3, stoppedKmh=3, stoppedSec=5}`, `sms {templates…}` |
| `user` | id, company_id, role (owner\|dispatcher\|driver), email?, phone?, password_hash?, locale, disabled_at | one auth identity; drivers log in by phone OTP, staff by email+password |
| `refresh_token` | id, user_id, family_id, hash, expires_at, revoked_at, replaced_by | rotation + reuse detection |
| `otp_challenge` | id, phone_hash, code_hash, expires_at, attempts | rate-limited |
| `zone` | id, company_id, name, name_en?, polygon geometry(Polygon,4326), queue_enabled, priority | GiST index |
| `vehicle` | id, company_id, plate, type (sedan\|minivan\|wav), capacity, features text[] (pets, bike_rack, child_seat…), active | |
| `driver` | id, company_id, user_id, display_name, first_name, phone, language, status, current_vehicle_id?, current_shift_id?, last_location geography(Point), last_seen_at, prefs jsonb (nav app, voice on) | status ∈ offline\|available\|offered\|en_route\|on_trip\|break |
| `shift` | id, company_id, driver_id, vehicle_id, started_at, ended_at, earnings jsonb (trips, gross, tips stub) | |
| `queue_entry` | id, company_id, zone_id, driver_id, sort_key, joined_at, lease_expires_at, grace_expires_at?, state (active\|grace\|released), released_reason?, last_heartbeat_at | **position is derived** = rank of `sort_key` among non-released entries in the zone; grace keeps sort_key ⇒ original position. Unique (driver) where state≠released. |
| `rider` | id, company_id, phone_e164, phone_hash, name?, locale?, sms_opt_out, created_at | basis for Law 25 export/delete |
| `account` | id, company_id, name, kind (corporate\|medical\|adapted), billing_ref, active | |
| `booking` | id, company_id, ref (short code), source (phone\|web\|dispatcher\|contract), rider_id, pickup geography(Point)+pickup_address, dropoff geography(Point)?+dropoff_address?, scheduled_at?, dispatch_at (computed), requirements jsonb {wheelchair, pax, pets…}, account_id?, contract_run_id?, status, assigned_driver_id?, vehicle_id?, fare_estimate_cents, fare_final_cents?, fare_breakdown jsonb, timestamps per status, cancel_reason?, notes | status: `scheduled → dispatching → offered → assigned → arrived → on_board → completed`; terminal `cancelled`, `unfilled`, `no_show` |
| `offer` | id, company_id, booking_id, driver_id, attempt, sent_at, delivered_at?, expires_at, responded_at?, response (accepted\|declined\|timeout\|superseded), channel (voice\|tap), stt_source (grammar\|llm)? | partial unique: one open offer per booking and per driver |
| `driver_action` | id (client actionId, idempotency key), company_id, driver_id, type, booking_id?, client_captured_at, client_sent_at, server_received_at, effective_at, result jsonb, status (applied\|rejected) | idempotent outbox sync log |
| `idempotency_key` | key, company_id, user_id, route, request_hash, response jsonb, expires_at | REST idempotency (24 h) |
| `location_ping` | driver_id, company_id, recorded_at, point geography, speed_mps, heading, accuracy_m, booking_id? | **range-partitioned by day**, 30-day drop job; trip trace source for final fare |
| `demand_forecast` | company_id, zone_id, hour_start, expected_requests, p10, p90, model (baseline\|lgbm), generated_at | |
| `forecast_eval` | company_id, model, holdout_week, mae, created_at | MAE reporting |
| `contract_run` | id, company_id, account_id?, service_date, vehicle_id?, driver_id?, status (proposed\|approved\|rejected\|in_progress\|done), solver_meta jsonb | |
| `contract_run_stop` | id, run_id, booking_id, kind (pickup\|dropoff), seq, eta, window_start, window_end | |
| `notification_log` | id, company_id, booking_id?, channel (sms\|voice), template, locale, to_hash, provider_id, status, sent_at | no raw phone/body stored |
| `audit_log` | id, company_id, actor_type (user\|driver\|system), actor_id, action, entity, entity_id, before jsonb, after jsonb, request_id, at | append-only; written in same tx |

## 4. Core logic design

### 4.1 Queue protection (`queue-core`)
State machine per entry, all transitions functions of `(entry, event, now, settings)`:
- `join` → active, `sort_key = now` (tie-break by id), `lease_expires_at = now + leaseSec`.
- `heartbeat` (every 10 s via socket, REST fallback) → extends lease; if `grace` and `now < grace_expires_at` → **restore to active, same sort_key**.
- sweeper tick (worker, every 2 s, plus on-demand before each dispatch decision): active & `now > lease_expires_at` → grace (`grace_expires_at = lease_expires_at + graceMin`); grace & `now > grace_expires_at` → released(`connectivity`), remember `pending_notice` for next reconnect ("hors ligne 4 min, repris en position X").
- Offer selection walks entries in sort order **skipping** grace entries without removing them.
- Server clock is authoritative for leases; client time never extends a lease.
- Tests: drop < grace (position kept, incl. when others joined/left meanwhile), drop > grace (released + notice + rejoin at tail), reconnect during active offer (offer redelivered with remaining TTL; accept after expiry rejected), duplicate syncs (same actionId ⇒ same result, no double effect), clock skew (±10 min device clock), property-based test (fast-check) over random event sequences: no two active entries share a rank, positions never jump ahead of anyone not released.

### 4.2 Offline actions & clock skew
Driver app outbox (SQLite): `{actionId(UUIDv7), type, bookingId, capturedAt(device), monotonicMs}`. On sync the server computes `effective_at = server_received_at − (client_sent_monotonic − client_captured_monotonic)`, clamped to `[booking.assigned_at, server_received_at]`. Replayed in capture order; each action idempotent via `driver_action.id`. Invalid transitions (e.g. booking cancelled meanwhile) return `rejected` with a localized reason the app speaks/shows.

### 4.3 Dispatch (`dispatch-core` + `dispatch` module)
- Trigger: booking becomes `dispatching` (immediately, or by delayed BullMQ job at `scheduled_at − leadMin`).
- Candidate pipeline: eligibility (status available, vehicle meets requirements: WAV, capacity, features; not in grace; follow-on rules for on-trip drivers; not previously declined/timed-out for this booking) → strategy: **zone FIFO** for the pickup zone (PostGIS `ST_Contains`) → fallback **nearest by ETA** (RoutingProvider; default = haversine × 1.3 road factor / zone speed, OSRM optional).
- One offer at a time: insert `offer` + set driver `offered` in a tx, emit `offer:new` (socket ack measures delivery), schedule BullMQ timeout job (`offerTimeoutSec`, default 15 s). Decline/timeout → next candidate. Accept → `assigned`, driver `en_route`, queue entry released(`assigned`). After `maxOfferAttempts` or no candidates → `unfilled` alert on console (stays re-dispatchable manually).
- Concurrency: per-booking and per-zone advisory locks; offer acceptance is a conditional update (`WHERE response IS NULL AND expires_at > now()`), so late accepts lose cleanly.
- Every transition → `audit_log` in the same transaction. Target < 300 ms decision (measured, histogram in /metrics).

### 4.4 Safety lock (`motion-core`, driver app)
Hysteresis detector fed by GPS speed (+ accelerometer magnitude variance to smooth GPS dropouts/tunnels): moving if > 8 km/h sustained 3 s; stopped if < 3 km/h sustained 5 s (configurable from company settings). While `moving`: tap UI disabled except glanceable info; any gated control shows "Sécuritaire à l'arrêt / Safe when stopped"; offers voice-only. Follow-on offers only when stopped or within `maxDistanceM`/`maxMinutesToDropoff` of drop-off, never as a modal over navigation (audio + slim banner), and suppressed entirely when `neverWhileOnTrip`. The server also enforces follow-on eligibility (doesn't trust the client).

### 4.5 Fares (`fare-core`)
`FareCalculator.estimate(booking, route)` / `.final(trace, waitSegments)`. Rules: flag drop, per-km, per-minute waiting (speed < threshold), minimum, flat zone-to-zone matrix, airport flat/surcharge; integer cents, rounding rule configurable. Final fare from GPS trace (distance via filtered pings: accuracy filter, outlier/jump rejection). Dev seed uses **placeholder** rates clearly marked — real Québec rates are set by the operator / CTQ, not invented here.

### 4.6 Voice (`voice-core` + driver app)
- TTS: expo-speech in driver language; offer phrase built from i18n templates with ICU formatting for numbers/money/times.
- STT: on-device recognizer (expo-speech-recognition with contextual strings). Normalize (lowercase, strip accents/punctuation) → fixed grammar match (fr/en, incl. `prends-la`, `passe`, `répète`, `arrivé`, `client à bord`, `course terminée`, `pause`, `combien j'ai fait aujourd'hui`, `appelle le client`, `en retard`).
- No match → `POST /voice/intent` → Claude with JSON-schema output `{intent, confidence}`, 2 s timeout, PII-redacted logs. `unknown`/confidence < 0.8 → re-prompt once → treat as timeout. **Never auto-accept on ambiguity.**
- Listening window only during an active offer or after a large "Parler / Talk" button press when stopped. No audio persisted.

### 4.7 Notifications
Event-driven (booking.confirmed, driver.assigned, driver.near [2 min ETA], driver.arrived, trip.completed → receipt link) → BullMQ → `NotificationProvider` (Twilio SMS | console). Locale: rider preference, else guess (+1 area codes 418/438/450/514/579/581/819/873/367/354/263 ⇒ fr-CA; else en-CA). Templates per company, facts (ETA, price, plate) always template-filled; Claude only optional rephrase of the "running late" free text, with fallback to template. "Call the rider": server-initiated Twilio Voice bridge (both legs see the company number) — numbers never exposed to the driver.

### 4.8 Forecast & coach (M6)
Baseline in `forecast-core`: seasonal mean by zone × weekday × hour over trailing 8 weeks, holiday mapped to Sunday profile, multiplied by an exponentially-smoothed recent trend ratio (α configurable). ML: LightGBM in `services/intel` (features: hour, dow, holiday flags (computed Québec calendar), lags 1 w/2 w, rolling trend, weather stub=null). Champion/challenger on held-out last week; the lower-MAE model is published to `demand_forecast`. Nightly retrain, hourly refresh. Coach card (stopped only) + voice on request; console heatmap = forecast vs live available drivers per zone + "under-supplied next hour" list.

### 4.9 Contract batching (M6)
`services/intel /vrp/solve` with OR-Tools (pickup-delivery + time windows + capacity dims seats/wheelchair + max ride-time ⇒ detour %). Proposed runs stored `proposed`; dispatcher approves/edits/rejects; nothing auto-assigns. Pure-TS greedy insertion heuristic as fallback if the service is down (also used in tests as a reference).

## 5. API surface (v1, abbreviated)
REST under `/v1`, Zod-validated, OpenAPI generated (`@fastify/swagger`).
- Auth: `POST /auth/otp/request`, `/auth/otp/verify`, `/auth/login`, `/auth/refresh`, `/auth/logout`
- Driver: `POST /driver/shift/start|end`, `POST /driver/status`, `POST /driver/queue/join|leave`, `GET /driver/me`, `POST /driver/actions:sync` (batch outbox), `GET /driver/earnings?range=`, `GET /driver/coach`
- Offers: `POST /offers/:id/respond`
- Console: CRUD `zones|drivers|vehicles|accounts`, `GET/POST/PATCH /bookings`, `POST /bookings/:id/assign|cancel|redispatch`, `GET /queues`, `GET/PUT /settings`, `GET /reports/*`, `GET /forecast`, `POST /contracts/runs:propose`, `POST /contracts/runs/:id/approve|reject`
- Public rider: `GET /public/:slug`, `POST /public/:slug/otp`, `POST /public/:slug/bookings`, `GET /public/bookings/:token` (status), `GET /public/receipts/:token`
- Privacy: `GET /privacy/riders/:id/export`, `DELETE /privacy/riders/:id` (owner role)
- Ops: `GET /healthz`, `GET /readyz`, `GET /metrics` (Prometheus)
Socket.IO namespaces: `/driver` (heartbeat, location, offer:new/offer:cancelled, booking:update, queue:position, notice) and `/console` (driver:update, booking:update, queue:update, alert). Reconnect: exponential backoff 1 s→30 s with jitter; server replays current state on connect (no event-history dependence).

## 6. Seed data (Sherbrooke, QC)
Company "Taxi Démo Sherbrooke" (`demo-sherbrooke`), tz America/Toronto, fr-CA.
Zones: **Centre-ville**, **Fleurimont / CHUS**, **Rock Forest / Carrefour de l'Estrie** (hand-drawn approximate polygons). 20 vehicles (14 sedans, 4 minivans, 2 WAV), 20 drivers (mixed fr/en), owner + dispatcher logins, 2 corporate/medical accounts, placeholder fare rules, airport (YSC) flat-rate example. 2 weeks of synthetic trip history drawn from a realistic intraday/weekly profile + noise (Poisson), addresses sampled from a bundled list of real Sherbrooke street addresses/POIs (no geocoding needed offline).

## 7. Testing strategy
- **Unit (Vitest)**: every `*-core` package ≥ 90 % branch coverage; fast-check property tests for queue & dispatch.
- **Integration (Vitest + Supertest + Testcontainers postgis/redis)**: auth/RBAC, **tenant isolation suite** (every route with a foreign-tenant id must 404), idempotency, dispatch end-to-end with fake clock, offer timeouts via BullMQ.
- **Console E2E (Playwright)** M3/M7. **Driver smoke (Maestro)** M4/M7 (Maestro chosen over Detox: works with Expo dev client, YAML flows, simpler CI).
- **Python (pytest)** for intel service; MAE regression thresholds.
- CI: GitHub Actions — pnpm cache, turbo, lint + typecheck + unit on every push; integration with service containers; E2E on main/PR label.

## 8. Milestones & checklist

### M1 — Foundation
- [ ] pnpm + Turborepo monorepo, strict TS, ESLint/Prettier, shared configs
- [ ] Docker Compose: postgis, redis, api, worker, console (+ `intel`, `osrm` profiles later)
- [ ] `packages/db`: Drizzle schema for all tables above, migrations, RLS policies, partitioned `location_ping`
- [ ] Seed script (Sherbrooke fleet + 2 weeks history)
- [ ] Auth: phone OTP (console provider in dev), email/password, JWT access + rotating refresh, RBAC, tenancy plugin
- [ ] `packages/i18n` fr-CA/en-CA wired into api errors, console shell, driver shell
- [ ] Observability base: pino structured logs, request IDs, `/metrics`, Sentry env-gated
- [ ] `.env.example` fully documented; CI workflow (lint, typecheck, test)
- [ ] Tests: auth, RBAC, tenant isolation baseline

### M2 — Core dispatch
- [ ] `queue-core` + tests (all 5.2 cases + property tests)
- [ ] `dispatch-core` + tests (FIFO, nearest-ETA fallback, requirements, exclusions)
- [ ] `fare-core` + tests (estimate, final from trace, zone flat, airport, minimum)
- [ ] Zones / bookings / offers / queue / shifts APIs; idempotency; audit log
- [ ] Worker: lease sweeper, offer timeouts, scheduled-dispatch release
- [ ] Socket.IO gateway (driver + console), heartbeats, state replay on reconnect
- [ ] `driver/actions:sync` with clock-skew handling
- [ ] Integration suite for 5.1 / 5.2 with fake clock

### M3 — Dispatcher console
- [ ] Next.js shell, auth, fr/en switch, night/day theme
- [ ] Live map (MapLibre): drivers by status, bookings, zones
- [ ] Phone booking form with address autocomplete + fare estimate; list/filter; assign/reassign/cancel
- [ ] Drivers & vehicles CRUD; per-zone queue view with grace countdown
- [ ] Zone polygon editor
- [ ] Settings: fares, offer timeout, grace, follow-on, SMS templates
- [ ] Reports: trips/hour, unfilled, acceptance rate, avg pickup ETA
- [ ] Playwright smoke

### M4 — Driver app (tap)
- [ ] Expo dev-client app, Expo Router, Zustand + React Query, i18n, ≥ 56 px targets, auto night mode
- [ ] Login OTP, language; start shift / vehicle; home (status, zone, position, connection + grace countdown)
- [ ] Offer screen, trip lifecycle, nav hand-off (Google Maps / Waze / Apple Maps)
- [ ] SQLite outbox + idempotent sync; socket reconnect with backoff
- [ ] Adaptive background location (4 s on trip / 10–15 s idle / off when off-shift)
- [ ] Earnings (today/week, trips, payout stub)
- [ ] Maestro smoke flow

### M5 — Copilot
- [ ] `motion-core` + tests; safety-lock overlay + gating
- [ ] `voice-core` grammar + tests; TTS offers; STT; Claude fallback (2 s, JSON schema, redaction) + tests
- [ ] Voice commands (arrived, on board, complete, pause, earnings, call rider, running late)
- [ ] Follow-on rules (client + server enforcement) + tests
- [ ] Rider SMS notifications (templates, locale guess) + masked call bridge
- [ ] Rider booking page `/book/[slug]` + live status page
- [ ] Privacy endpoints (export/delete), `PRIVACY.md` draft

### M6 — Intelligence
- [ ] `forecast-core` baseline + Québec holidays + tests
- [ ] `services/intel`: LightGBM train/predict, held-out-week MAE vs baseline
- [ ] Nightly retrain + hourly refresh jobs
- [ ] Coach card + voice; console heatmap + under-supplied list
- [ ] OR-Tools VRP batching, TS heuristic fallback, proposal review/approve UI

### M7 — Hardening
- [ ] `scripts/simulate` (N drivers on real roads via OSRM or recorded routes, forecast-driven bookings, random connectivity drops, KPIs)
- [ ] Load test: 60 drivers / 300 bookings/h; report p95 offer delivery & dispatch latency
- [ ] Full Playwright + Maestro E2E in CI
- [ ] `SECURITY.md` checklist review (authz, rate limits, headers, secrets, deps audit)
- [ ] README (≤ 10 commands to demo), `DEMO.md` script matching Definition of Done

## 9. Risks / honest caveats
- **Hands-free STT in a noisy cab** is the riskiest UX piece; on-device recognizers vary by device/OS. Mitigation: tiny grammar, contextual hints, confirmation read-back ("Course acceptée"), never auto-accept on ambiguity. Real-car testing needed — cannot be proven in CI.
- **iOS background audio/listening** limits: we only listen inside offer windows (app foreground or with an active audio session from navigation hand-off may be interrupted). Hand-off to Waze/Google Maps backgrounds our app; offers then arrive via TTS while backgrounded + local notification. Needs device validation in M5.
- **Driver app builds** (iOS/Android binaries, store accounts) cannot be produced in this cloud container; I'll build/test JS + Maestro flows against Android emulator in CI where feasible, and document local `eas build` steps.
- **Maps/geocoding** in production need a paid provider (Google/Mapbox); MVP defaults to OSM-based providers with usage-policy caveats.
- **Regulatory**: fares/meter/Revenu Québec SRM are explicitly out of scope; placeholders only.

## 10. Open questions for approval
- **Q1 (irreversible-ish): where should Copilote live?** This repo (`taxietnavette/garage-tn`) is currently the *Garage TN* GitHub-Pages PWA. Options: (a) keep everything under `copilote/` in this repo (current draft location — no impact on the Pages site), or (b) a new dedicated repo (cleaner CI/history; you'd create it on GitHub and I'd move the files). **Recommendation: (b)**, fall back to (a) if you prefer to start now.
- Q2: Seed city — Sherbrooke (default) or Trois-Rivières?
- Q3: OK to use OSM-based map tiles/geocoding (MapLibre + Photon/OSRM) for the MVP, with Google/Mapbox as pluggable production providers?
