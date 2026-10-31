# Decisions log

Format: **D-nnn — title** · status · rationale. Defaults chosen without asking (spec §0.4); challenge any of them.

**D-001 — Fastify over NestJS** · proposed
Lighter, faster cold start, first-class Zod via `fastify-type-provider-zod`, easy to share handlers with the worker. Module boundaries enforced by folder conventions + lint rules rather than a DI framework.

**D-002 — Drizzle over Prisma** · proposed
PostGIS geometry/geography columns, partitioned tables, RLS policies and `FOR UPDATE SKIP LOCKED` are first-class or trivially expressible in Drizzle/SQL; Prisma treats them as `Unsupported` and pushes to raw SQL.

**D-003 — Postgres is the source of truth for queue leases; Redis is not** · proposed
Spec suggests Redis for leases. At target scale (≤ 60 drivers × heartbeat/10 s ≈ 6 writes/s per company) Postgres handles it easily, and keeping leases transactional with offers/bookings removes a whole class of split-brain bugs (Redis eviction/restart cannot lose a queue position). Redis is used for BullMQ, pub/sub, Socket.IO adapter and rate limiting. Revisit if > 1 000 concurrent drivers.

**D-004 — Queue position is derived, not stored** · proposed
`position = rank(sort_key)` among non-released entries in the zone. Grace keeps `sort_key`, so restore-to-original-position is automatic and positions can never collide or drift.

**D-005 — Socket.IO over raw ws** · proposed
Built-in reconnection with backoff, acks (used to measure offer delivery latency), rooms, Redis adapter for horizontal scale. Server replays full current state on (re)connect.

**D-006 — Forecast: TS baseline + Python intel service** · proposed
Baseline lives in TS (`forecast-core`) so the product works with no Python service running. LightGBM and OR-Tools both live in one Python FastAPI service (`services/intel`) because OR-Tools VRP is needed for M6 anyway; the ML model is only published if it beats the baseline MAE on a held-out week.

**D-007 — Driver offline storage: expo-sqlite for the outbox, MMKV for prefs** · proposed
Outbox needs durable, ordered, transactional writes; prefs need fast sync reads.

**D-008 — Driver app E2E: Maestro over Detox** · proposed
Works with Expo dev client without native test harness changes; YAML flows are easy to maintain.

**D-009 — Maps: MapLibre GL + OSM-based providers by default** · proposed (pending Q3)
No token required for dev/demo. `GeocodingProvider` (Photon | Google), `RoutingProvider` (haversine×factor | OSRM | Google). Production should use a paid provider respecting usage policies.

**D-010 — Masked rider calls via server-initiated Twilio Voice bridge** · proposed
Twilio Proxy is deprecated; instead the server calls the driver and bridges to the rider with the company number as caller ID on both legs.

**D-011 — Tenant isolation: app-level scoping + Postgres RLS** · proposed
Every repository function requires a `companyId` from the auth context; RLS (`app.company_id` set per transaction) as defense-in-depth. A dedicated test suite hits every route with cross-tenant IDs.

**D-012 — Voice listening & privacy** · proposed
Microphone is opened only (a) during an active offer window or (b) after the large "Parler/Talk" button is pressed while stopped. On-device recognition preferred; only the text transcript (never audio) is sent to Claude, and only when the local grammar fails, with names/phones/addresses redacted. No audio stored. Battery impact is bounded because listening windows are ≤ 15–20 s.

**D-013 — LLM guardrails** · proposed
Claude model from `ANTHROPIC_MODEL` env; strict JSON-schema output; 2 s timeout; any failure ⇒ non-AI fallback (re-prompt / template). LLM output never sets money, ETA, safety state or assignment.

**D-014 — Placeholder fares** · proposed
Seed fare rules are illustrative only and labelled as such; real tariffs are configured by the operator.

**D-015 — Project location** · open (Q1 in PLAN.md)
Drafted under `copilote/` in `garage-tn` to avoid disturbing the existing Garage TN Pages site.
