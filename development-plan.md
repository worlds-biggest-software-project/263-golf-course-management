# Golf Course Management — Phased Development Plan

> Project: 263-golf-course-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into an implementation specification. The operational schema is based on **Data Model 3 (Hybrid Relational + JSONB)** — the best fit for a multi-market SaaS serving public, private, semi-private, resort, and municipal facilities across US and international handicap jurisdictions. The AI/analytics phases adopt the **time-partitioned fact-table** pattern from Data Model 4. Audit/temporal needs are met with append-only history tables (price snapshots, handicap history, audit log) drawn from Data Models 1 and 2 rather than full event sourcing, keeping the developer learning curve and operational complexity manageable.

---

## Product Summary

**What it is:** An open, AI-native golf course management platform unifying the tee sheet, online booking, pro-shop and F&B POS, player/member CRM, WHS/GHIN handicap integration, payments, and reporting under operator control — with a public OpenAPI 3.1 surface so courses are never locked into a single marketplace or commerce stack.

**Primary personas:**
- **Public green-fee operator** — tee-time yield, online distribution, walk-in POS.
- **Private/semi-private club manager** — membership billing, member experience, dues.
- **Resort golf director** — coordination with hotel/PMS, multi-course facilities.
- **Charity/corporate event organiser** — tournament registration and scoring.

**Key differentiators:** Operator data ownership (no forced marketplace), OpenAPI-first integration surface, and AI applied to live operational decisions (demand-based pricing, member-retention prediction, course-conditions fusion, predictive maintenance) rather than static reports.

**Deployment model:** Multi-tenant SaaS (single PostgreSQL database, Row-Level Security per `tenant_id`), self-hostable via Docker Compose. A REST/JSON API is the system of record; a server-rendered + React admin web UI and a public booking widget consume it. Players use the booking site; a mobile app is backlog.

**Standards the platform must implement:** WHS Interoperability Standard v1.0, GHIN API (US) and iGolf Connect (international) for handicap; USGA Course Rating/Slope formulas (Slope range 55–155, standard 113); PCI DSS via tokenising gateway (SAQ-A scope — no raw card data); EMV/NFC at POS terminals; OAuth 2.0 (RFC 6749) + Bearer (RFC 6750) + OpenID Connect for auth and partner APIs; OpenAPI 3.1 + JSON Schema 2020-12 for the API; GDPR + CCPA for privacy; Reserve with Google booking feed for distribution.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The system is API-and-integration heavy (GHIN, iGolf, GolfNow, Stripe, weather), and the same language spans backend, admin SPA, and booking widget. JSON-native fits the hybrid JSONB schema and OpenAPI-first goal. |
| API framework | Fastify 5 + `@fastify/swagger` | High throughput for tee-sheet reads; first-class JSON Schema validation per route auto-generates the OpenAPI 3.1 spec required by `standards.md`. |
| Schema/validation | Zod + `zod-to-json-schema` | One Zod schema per resource drives runtime validation, TypeScript types, and the OpenAPI/JSON Schema 2020-12 definitions — no triple maintenance. |
| Database | PostgreSQL 16 | Required for JSONB + GIN indexes (Data Model 3), Row-Level Security multi-tenancy, range partitioning for fact tables (Data Model 4), and `gen_random_uuid()`. |
| Query layer | Drizzle ORM | Typed SQL with first-class JSONB column support and lightweight, reviewable SQL migrations — avoids the heavy abstraction of full ORMs while keeping types. |
| Migrations | drizzle-kit | Generates and versions SQL migrations from the Drizzle schema; raw-SQL escape hatch for partitions, RLS policies, and GIN indexes. |
| Cache / queue | Redis 7 + BullMQ | Async workloads: GHIN/iGolf score posting, marketplace sync, dynamic-price recomputation, webhook delivery, campaign sends, AI scoring jobs. Redis also caches tee-sheet reads. |
| Payments | Stripe (Payments + Terminal) | Stripe Terminal provides EMV/NFC certified hardware; tokenisation keeps PCI scope at SAQ-A (per `standards.md` note). Gateway abstraction lets Braintree be added later. |
| LLM / AI | Vercel AI SDK + OpenAI/Anthropic via provider abstraction | Course-conditions narrative generation and marketing copy. Forecasting/retention use classical ML (see below), reserving LLMs for text generation. |
| ML (forecasting/retention) | Python 3.12 microservice (FastAPI + scikit-learn / LightGBM) | Demand forecasting and member-retention models are tabular ML; Python's ecosystem is the right tool. Exposed as an internal REST service called by BullMQ jobs. |
| Frontend (admin) | React 19 + Vite + TanStack Query + shadcn/ui (Tailwind) | Dense operational UI (tee sheet grid, POS register, dashboards). TanStack Query handles the API caching/invalidation; shadcn for accessible components. |
| Frontend (booking widget) | React 19 embeddable widget + Next.js public site | A standalone booking flow embeddable on course websites, plus an SSR public site for SEO. |
| Auth | OpenID Connect via `openid-client`; partner API via OAuth 2.0 client-credentials | Staff SSO and member login per `standards.md`; partner/marketplace apps use OAuth 2.0 + Bearer tokens. |
| Testing | Vitest (unit/integration) + Supertest (HTTP) + Playwright (E2E) + Testcontainers (real Postgres/Redis) | Vitest is fast and TS-native; Testcontainers gives real-DB integration tests; Playwright covers the booking and POS flows. |
| Code quality | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Standard TS toolchain; type checking gates CI. |
| Package manager | pnpm workspaces (monorepo) | Shared types between API, admin, widget, and SDK; efficient installs. |
| Containerisation | Docker + docker-compose | Self-hostable per README; compose wires api, worker, postgres, redis, ml-service. |
| Key libraries | `luxon` (IANA timezones for tee times), `ioredis`, `pino` (logging), `@fastify/rate-limit`, `pg` partitioning helpers, `ajv` (JSON Schema runtime where Zod is insufficient) | Domain-specific needs: per-facility timezone slot generation, structured logs, rate limiting per partner. |

### Project Structure

```
golf-course-management/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── docker-compose.yml                # api, worker, postgres, redis, ml-service
├── docker-compose.test.yml
├── .env.example
├── tsconfig.base.json
├── packages/
│   ├── shared/                       # cross-cutting types & Zod schemas (the source of truth)
│   │   └── src/
│   │       ├── schemas/              # Zod: facility, course, booking, player, score, pos, tournament, pricing
│   │       ├── types/                # inferred TS types + JSONB sub-schemas
│   │       └── errors/               # AppError taxonomy
│   ├── db/                           # Drizzle schema + migrations + RLS policies
│   │   ├── src/schema/              # one file per domain table group
│   │   ├── migrations/              # generated SQL + hand-written (partitions, RLS, GIN)
│   │   └── src/client.ts            # tenant-scoped connection helper
│   ├── handicap/                     # WHS calculation engine + GHIN/iGolf adapters (pure logic, testable)
│   ├── pricing/                      # dynamic pricing rule engine
│   └── sdk/                          # generated TS client from OpenAPI (for widget & partners)
├── services/
│   ├── api/                          # Fastify app
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/             # auth, rls-context, rate-limit, swagger, error-handler
│   │       ├── routes/             # /facilities /courses /tee-sheet /bookings /players /scores /pos /tournaments /pricing /reports /webhooks /partner
│   │       ├── services/           # business logic per domain (BookingService, etc.)
│   │       └── integrations/       # stripe, ghin, igolf, golfnow, weather, google-reserve
│   ├── worker/                       # BullMQ processors
│   │   └── src/jobs/               # postScore, syncMarketplace, recomputePricing, deliverWebhook, sendCampaign, runForecast
│   └── ml-service/                   # Python FastAPI: forecasting + retention + maintenance
│       ├── app/
│       │   ├── main.py
│       │   ├── demand_forecast.py
│       │   ├── retention.py
│       │   └── maintenance.py
│       ├── pyproject.toml
│       └── tests/
├── apps/
│   ├── admin/                        # React + Vite operator console
│   └── booking/                      # Next.js public site + embeddable widget
└── tests/
    ├── integration/                  # Testcontainers-backed API tests
    ├── e2e/                          # Playwright (booking flow, POS, tee-sheet)
    └── fixtures/                     # seed data: sample course, tees, rate codes, players
```

---

## Phase 1: Foundation — Monorepo, Database Core, Multi-Tenancy

### Purpose
Establish the runnable skeleton: pnpm monorepo, Dockerised Postgres/Redis, the Fastify server booting with health checks, the Drizzle schema for the facility/course/tenant core, Row-Level Security multi-tenancy, and the auto-generated OpenAPI document. After this phase a developer can run `docker compose up`, hit `/health`, see an empty but valid `/openapi.json`, and run migrations that create the foundational tables with tenant isolation enforced.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap

**What**: Stand up the pnpm workspace, TypeScript config, linting, formatting, and a CI-runnable test command.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*`, `services/*`, `apps/*`.
- `tsconfig.base.json`: `target ES2023`, `module NodeNext`, `strict true`, `noUncheckedIndexedAccess true`. Each package extends it.
- Root scripts: `lint`, `typecheck`, `test`, `build`, `db:generate`, `db:migrate`, `dev`.
- `.env.example` documents: `DATABASE_URL`, `REDIS_URL`, `STRIPE_SECRET_KEY`, `GHIN_CLIENT_ID/SECRET`, `IGOLF_API_KEY`, `OIDC_ISSUER`, `JWT_SIGNING_KEY`, `WEATHER_API_KEY`, `ML_SERVICE_URL`, `PORT` (default 8080).
- ESLint flat config with `typescript-eslint` recommended-type-checked; Prettier with 100-col print width.

**Testing**:
- `Unit: pnpm typecheck → exit 0 on clean tree`.
- `Unit: ESLint on a file with an unused var → reports no-unused-vars`.
- `Smoke: pnpm build → all packages emit dist/ without error`.

#### 1.2 — Docker Compose environment

**What**: A `docker-compose.yml` bringing up Postgres 16, Redis 7, the API, the worker, and the ml-service.

**Design**:
- `postgres`: image `postgres:16`, env `POSTGRES_DB=gcm`, healthcheck `pg_isready`, named volume.
- `redis`: image `redis:7`, healthcheck `redis-cli ping`.
- `api`: built from `services/api/Dockerfile` (multi-stage: build → slim runtime node:22), depends_on postgres+redis healthy, exposes 8080.
- `worker`: same image, command `node dist/worker.js`.
- `ml-service`: built from `services/ml-service/Dockerfile` (python:3.12-slim + uvicorn), exposes 8000.
- `docker-compose.test.yml` overrides DB name to `gcm_test` and uses ephemeral storage.

**Testing**:
- `Integration: docker compose up → /health returns 200 within 60s`.
- `Integration: ml-service /health returns 200`.

#### 1.3 — Drizzle schema: tenant, facility, course, hole, course_tee (hybrid)

**What**: Define the foundational relational+JSONB tables and generate the first migration.

**Design** (Drizzle schema, hybrid per Data Model 3 — relational columns for universal fields, JSONB for variable config):

```ts
// packages/db/src/schema/facility.ts
export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 200 }).notNull(),
  status: varchar('status', { length: 20 }).notNull().default('active'), // active|suspended
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const facility = pgTable('facility', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  name: varchar('name', { length: 200 }).notNull(),
  facilityType: varchar('facility_type', { length: 30 }).notNull(), // public|private|semi_private|resort|municipal
  countryCode: char('country_code', { length: 2 }).notNull(),        // ISO 3166-1
  timezone: varchar('timezone', { length: 50 }).notNull(),           // IANA
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'), // ISO 4217
  ghinClubId: varchar('ghin_club_id', { length: 20 }),
  // JSONB: address, contact, jurisdiction-specific regulatory config, tax config, multi-currency display
  config: jsonb('config').$type<FacilityConfig>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  tenantIdx: index('idx_facility_tenant').on(t.tenantId),
  configGin: index('idx_facility_config').using('gin', t.config),
}));

export const course = pgTable('course', {
  id: uuid('id').primaryKey().defaultRandom(),
  facilityId: uuid('facility_id').notNull().references(() => facility.id),
  name: varchar('name', { length: 200 }).notNull(),
  holesCount: smallint('holes_count').notNull(), // 9|18|27|36
  par: smallint('par').notNull(),
  isActive: boolean('is_active').notNull().default(true),
  externalIds: jsonb('external_ids').$type<ExternalIds>().notNull().default({}), // {ghin, igolf, golfapi, ncrdb}
  holesConfig: jsonb('holes_config').$type<HoleConfig[]>().notNull().default([]),
  // [{holeNumber, par, strokeIndex, gps:{lat,lng}}]
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const courseTee = pgTable('course_tee', {
  id: uuid('id').primaryKey().defaultRandom(),
  courseId: uuid('course_id').notNull().references(() => course.id),
  teeName: varchar('tee_name', { length: 50 }).notNull(),
  teeColour: varchar('tee_colour', { length: 20 }),
  gender: char('gender', { length: 1 }), // M|F|X
  courseRating: numeric('course_rating', { precision: 4, scale: 1 }).notNull(),
  slopeRating: smallint('slope_rating').notNull(), // CHECK 55..155
  bogeyRating: numeric('bogey_rating', { precision: 4, scale: 1 }),
  // per-hole yardage/metres stored in holesConfig.teeYardages keyed by teeName
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

- Add a hand-written migration adding `CHECK (slope_rating BETWEEN 55 AND 155)` and `CHECK (facility_type IN (...))` constraints that Drizzle does not express natively.
- Corresponding Zod schemas in `packages/shared/src/schemas` mirror each table and define the JSONB sub-shapes (`FacilityConfig`, `HoleConfig`, `ExternalIds`).

**Testing**:
- `Unit: Zod FacilityConfig accepts valid address+tax block; rejects unknown facilityType`.
- `Integration (Testcontainers): run migration → information_schema shows facility, course, course_tee with GIN index on facility.config`.
- `Integration: insert course_tee with slope_rating=200 → DB rejects (check constraint)`.

#### 1.4 — Row-Level Security & tenant context

**What**: Enforce per-tenant data isolation at the database layer and propagate tenant context through requests.

**Design**:
- Hand-written migration enables RLS on every tenant-scoped table and adds policy:
  `CREATE POLICY tenant_isolation ON facility USING (tenant_id = current_setting('app.tenant_id')::uuid);`
- `packages/db/src/client.ts` exposes `withTenant(tenantId, fn)` that opens a transaction and runs `SET LOCAL app.tenant_id = $1` before executing queries.
- Fastify `rls-context` plugin reads the authenticated tenant from the JWT/session and wraps the request's DB access in `withTenant`.

**Testing**:
- `Integration: insert facilities for tenant A and B; query under app.tenant_id=A → only A's rows returned`.
- `Integration: attempt cross-tenant update under tenant A on tenant B row → 0 rows affected`.
- `Unit: withTenant rolls back transaction on thrown error`.

#### 1.5 — Fastify server, health, error taxonomy, OpenAPI

**What**: Boot the API with health checks, a structured error handler, and auto-generated OpenAPI 3.1.

**Design**:
- `GET /health` → `{status:'ok', db:'ok'|'down', redis:'ok'|'down'}` (200 if all ok, 503 otherwise).
- `@fastify/swagger` configured for OpenAPI 3.1; each route registers its Zod schema converted via `zod-to-json-schema`. `GET /openapi.json` serves the spec.
- `AppError` taxonomy in `packages/shared/src/errors`: `ValidationError(400)`, `UnauthorizedError(401)`, `ForbiddenError(403)`, `NotFoundError(404)`, `ConflictError(409)`, `RateLimitError(429)`. Error handler serialises to `{error:{code,message,details}}`.
- `pino` request logging with correlation id (`x-request-id`).

**Testing**:
- `Integration: GET /health with DB up → 200 {db:'ok'}`.
- `Integration: GET /health with DB down → 503`.
- `Integration: GET /openapi.json → valid OpenAPI 3.1 document (openapi:'3.1.0')`.
- `Unit: error handler maps NotFoundError → 404 {error:{code:'NOT_FOUND'}}`.

---

## Phase 2: Identity, Authentication & Authorization

### Purpose
Add staff and player identity, OpenID Connect login, OAuth 2.0 partner credentials, and role-based access control. Every subsequent phase depends on knowing *who* is acting and *what* they may do. After this phase, staff can log in via OIDC, requests carry an authenticated tenant + role, partner apps can obtain Bearer tokens, and routes can be guarded by permission.

### Tasks

#### 2.1 — Staff, roles, permissions (RBAC)

**What**: Model staff accounts and a role→permission matrix per `standards.md` OIDC alignment.

**Design** (relational, from Data Model 1 RBAC section):

```ts
export const staff = pgTable('staff', {
  id: uuid('id').primaryKey().defaultRandom(),
  facilityId: uuid('facility_id').notNull().references(() => facility.id),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  email: varchar('email', { length: 200 }).notNull(),
  role: varchar('role', { length: 30 }).notNull(),
  // owner|general_manager|pro|assistant_pro|starter|ranger|pro_shop|f_and_b|maintenance|admin|bookkeeper
  authProviderId: varchar('auth_provider_id', { length: 200 }), // OIDC subject
  isActive: boolean('is_active').notNull().default(true),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
});

export const rolePermission = pgTable('role_permission', {
  role: varchar('role', { length: 30 }).notNull(),
  permissionKey: varchar('permission_key', { length: 100 }).notNull(), // 'booking.manage','pos.refund','pricing.edit'
}, (t) => ({ pk: primaryKey({ columns: [t.role, t.permissionKey] }) }));
```

- A seeded permission catalogue (`permissionKey` list) and default role→permission grants.
- `requirePermission(key)` Fastify preHandler that checks the actor's role grants `key`.

**Testing**:
- `Unit: requirePermission('pos.refund') with role 'starter' → ForbiddenError`.
- `Unit: requirePermission('pos.refund') with role 'general_manager' → passes`.
- `Integration: seed grants → GM role resolves to expected permission set`.

#### 2.2 — OIDC login for staff & members

**What**: Authorization-code login flow producing a session JWT carrying tenant, facility, actor id, role.

**Design**:
- `GET /auth/login` → redirect to OIDC issuer (`openid-client`).
- `GET /auth/callback` → exchange code, match/create `staff` (or `player`) by `authProviderId`, issue an HTTP-only session cookie + JWT.
- JWT claims: `sub`, `tenantId`, `facilityId`, `actorType('staff'|'player')`, `role`, `permissions[]`, `exp`.
- `auth` plugin verifies JWT, attaches `request.actor`, and feeds tenant to the RLS plugin.

**Testing**:
- `Integration (mocked OIDC): valid callback code → session cookie set, JWT decodes with tenantId`.
- `Integration (mocked OIDC): unknown subject with self-signup enabled → player created`.
- `Unit: expired JWT → UnauthorizedError`.

#### 2.3 — OAuth 2.0 partner credentials (client-credentials + Bearer)

**What**: Let registered partner/marketplace apps obtain Bearer tokens scoped to specific operations.

**Design** (per Data Model 1 `api_partner`):

```ts
export const apiPartner = pgTable('api_partner', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  partnerName: varchar('partner_name', { length: 200 }).notNull(),
  clientId: varchar('client_id', { length: 200 }).notNull().unique(),
  clientSecretHash: varchar('client_secret_hash', { length: 200 }).notNull(),
  scopes: jsonb('scopes').$type<string[]>().notNull().default([]),
  // ['bookings:read','bookings:write','players:read','tee-sheet:read']
  partnerType: varchar('partner_type', { length: 30 }).notNull(),
  rateLimitPerHour: integer('rate_limit_per_hour').notNull().default(1000),
  isActive: boolean('is_active').notNull().default(true),
});
```

- `POST /oauth/token` (grant_type=client_credentials) → validate `clientId`+secret (argon2 hash), issue Bearer JWT with `scopes` claim, RFC 6750 format.
- Partner routes (`/partner/*`) require Bearer and check scope.
- `@fastify/rate-limit` keyed by `clientId` using `rateLimitPerHour`.

**Testing**:
- `Integration: POST /oauth/token valid creds → 200 access_token, token_type:'Bearer'`.
- `Integration: bad secret → 401, no token`.
- `Integration: partner with only bookings:read calls write endpoint → 403`.
- `Integration: exceed rateLimitPerHour → 429`.

---

## Phase 3: Tee Sheet & Booking Core (MVP heart, part 1)

### Purpose
Deliver the operational heart: a per-course tee sheet of time slots and a booking engine that reserves players into slots with overbooking protection. This is the most-used surface in any golf operation. After this phase, staff can generate a tee sheet for a date range, view it, and create/modify/cancel bookings via the API with correct capacity and concurrency handling.

### Tasks

#### 3.1 — Tee sheet configuration & slot generation

**What**: Configure tee-sheet rules per course and generate `tee_time_slot` inventory for date ranges, timezone-aware.

**Design**:

```ts
export const teeSheetConfig = pgTable('tee_sheet_config', {
  id: uuid('id').primaryKey().defaultRandom(),
  courseId: uuid('course_id').notNull().references(() => course.id),
  effectiveDate: date('effective_date').notNull(),
  endDate: date('end_date'),
  firstTeeTime: time('first_tee_time').notNull(),   // 06:00 local
  lastTeeTime: time('last_tee_time').notNull(),     // 18:00 local
  intervalMinutes: smallint('interval_minutes').notNull().default(8),
  maxPlayersPerSlot: smallint('max_players_per_slot').notNull().default(4),
  shotgunStart: boolean('shotgun_start').notNull().default(false),
  crossoverEnabled: boolean('crossover_enabled').notNull().default(false),
});

export const teeTimeSlot = pgTable('tee_time_slot', {
  id: uuid('id').primaryKey().defaultRandom(),
  courseId: uuid('course_id').notNull().references(() => course.id),
  slotDate: date('slot_date').notNull(),
  slotTime: time('slot_time').notNull(),
  startingHole: smallint('starting_hole').notNull().default(1),
  maxPlayers: smallint('max_players').notNull().default(4),
  bookedPlayers: smallint('booked_players').notNull().default(0),
  status: varchar('status', { length: 20 }).notNull().default('available'),
  // available|partial|full|blocked|maintenance|tournament
  blockReason: varchar('block_reason', { length: 200 }),
  currentPrice: numeric('current_price', { precision: 10, scale: 2 }),
}, (t) => ({
  uniq: unique('uq_slot').on(t.courseId, t.slotDate, t.slotTime, t.startingHole),
  dateIdx: index('idx_slot_date').on(t.courseId, t.slotDate, t.status),
}));
```

- `generateSlots(courseId, fromDate, toDate)`: for each date, resolve the applicable `tee_sheet_config`, step from `firstTeeTime` to `lastTeeTime` by `intervalMinutes` in the facility's IANA timezone (luxon), insert idempotently (ON CONFLICT DO NOTHING on the unique key). If `crossoverEnabled`, also create `startingHole=10` slots.
- `POST /courses/:id/tee-sheet/generate` body `{fromDate,toDate}` enqueues a BullMQ job; returns 202 with job id.
- `GET /courses/:id/tee-sheet?date=` returns the day's slots ordered by time.

**Testing**:
- `Unit: generateSlots with 06:00–06:30 @ 8min → slots at 06:00,06:08,06:16,06:24`.
- `Unit: DST boundary date → slot count correct, no duplicate/missing local times`.
- `Integration: generate twice for same range → no duplicate rows (idempotent)`.
- `Integration: crossoverEnabled → both startingHole 1 and 10 slots exist`.

#### 3.2 — Booking creation with concurrency safety

**What**: Reserve 1–4 players into a slot atomically, preventing overbooking under concurrent requests.

**Design** (booking + booking_player from Data Model 1, with JSONB add-ons):

```ts
export const booking = pgTable('booking', {
  id: uuid('id').primaryKey().defaultRandom(),
  facilityId: uuid('facility_id').notNull(),
  teeTimeSlotId: uuid('tee_time_slot_id').notNull().references(() => teeTimeSlot.id),
  bookedById: uuid('booked_by_id').notNull(),
  bookingSource: varchar('booking_source', { length: 30 }).notNull(),
  // online|phone|walk_in|marketplace|api|staff
  marketplaceRef: varchar('marketplace_ref', { length: 100 }),
  playerCount: smallint('player_count').notNull(),
  status: varchar('status', { length: 20 }).notNull().default('confirmed'),
  // pending|confirmed|checked_in|completed|cancelled|no_show
  totalAmount: numeric('total_amount', { precision: 10, scale: 2 }),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'),
  confirmationCode: varchar('confirmation_code', { length: 20 }).notNull(),
  addOns: jsonb('add_ons').$type<AddOn[]>().notNull().default([]),
  // [{type:'cart'|'caddie'|'range_balls'|'rental_clubs', qty, unitPrice}]
  cancelledAt: timestamp('cancelled_at', { withTimezone: true }),
  checkedInAt: timestamp('checked_in_at', { withTimezone: true }),
});

export const bookingPlayer = pgTable('booking_player', {
  id: uuid('id').primaryKey().defaultRandom(),
  bookingId: uuid('booking_id').notNull().references(() => booking.id, { onDelete: 'cascade' }),
  playerId: uuid('player_id'),               // null for guest
  playerName: varchar('player_name', { length: 200 }),
  playerPosition: smallint('player_position').notNull(),
  greenFeeAmount: numeric('green_fee_amount', { precision: 10, scale: 2 }),
  cartFeeAmount: numeric('cart_fee_amount', { precision: 10, scale: 2 }),
  checkedIn: boolean('checked_in').notNull().default(false),
});
```

- `BookingService.create()` runs in a transaction:
  1. `SELECT ... FOR UPDATE` the slot row (pessimistic lock).
  2. Reject if `status IN (blocked,maintenance,tournament)` (→ ConflictError) or `bookedPlayers + playerCount > maxPlayers` (→ ConflictError `SLOT_CAPACITY`).
  3. Insert booking + booking_player rows; increment `bookedPlayers`; set slot `status` to `partial`/`full`.
  4. Generate `confirmationCode` (`GC-<year>-<base32(6)>`).
- State machine for booking status: `pending→confirmed→checked_in→completed`, with `cancelled`/`no_show` terminal; illegal transitions → ConflictError.
- Endpoints: `POST /bookings`, `GET /bookings/:id`, `PATCH /bookings/:id` (reschedule/add player/cancel), `POST /bookings/:id/check-in`.

**Testing**:
- `Integration: book 4 into empty 4-max slot → slot full, booking confirmed`.
- `Integration: two concurrent bookings of 3 into a 4-max slot → one succeeds, one ConflictError SLOT_CAPACITY` (run with real Postgres + parallel transactions).
- `Integration: book a blocked slot → 409`.
- `Unit: status transition completed→pending → ConflictError`.
- `Integration: cancel booking → slot bookedPlayers decremented, status recomputed`.

#### 3.3 — Tee sheet read model & blocking

**What**: A fast denormalised tee-sheet view for the operator grid, plus staff slot blocking.

**Design**:
- `GET /courses/:id/tee-sheet?date=` returns slots with embedded booking summaries: `[{slotTime, status, maxPlayers, bookedPlayers, currentPrice, bookings:[{id, confirmationCode, players:[name], status}]}]`.
- Cache the day payload in Redis keyed `teesheet:{courseId}:{date}`, invalidated on any booking/slot mutation for that course+date.
- `POST /slots/:id/block` body `{reason, status:'blocked'|'maintenance'}` (requires `tee-sheet.manage`); rejects if slot already has active bookings unless `force=true`.

**Testing**:
- `Integration: tee sheet read returns slots with nested confirmed bookings`.
- `Integration: create booking → cached payload invalidated, next read reflects it`.
- `Integration: block slot with active booking, force=false → 409`.

---

## Phase 4: Players, CRM & Membership (MVP heart, part 2)

### Purpose
Add the player/member record that bookings, scores, POS, and tournaments all reference, plus membership plans and billing for private/semi-private clubs. After this phase, staff manage player profiles with privacy consent tracking, link members to plans, and the system can attribute activity to players.

### Tasks

#### 4.1 — Player CRM with privacy consent

**What**: Player profiles with GHIN/WHS identifiers, handicap, and GDPR/CCPA consent in JSONB.

**Design** (hybrid per Data Model 3 — privacy as timestamped JSONB):

```ts
export const player = pgTable('player', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  email: varchar('email', { length: 200 }),
  phone: varchar('phone', { length: 30 }),
  dateOfBirth: date('date_of_birth'),
  gender: char('gender', { length: 1 }),
  countryCode: char('country_code', { length: 2 }),
  playerType: varchar('player_type', { length: 20 }).notNull().default('guest'),
  // member|guest|visitor|staff|pro
  handicap: jsonb('handicap').$type<HandicapBlock>().notNull().default({}),
  // {index, updatedAt, ghin:{number, lastSync}, igolf:{id, lastSync}, whsId}
  privacy: jsonb('privacy').$type<PrivacyBlock>().notNull().default({}),
  // {gdprConsentAt, ccpaOptOut, marketingOptIn, consentHistory:[{regulation,granted,at}]}
  isActive: boolean('is_active').notNull().default(true),
}, (t) => ({
  tenantIdx: index('idx_player_tenant').on(t.tenantId),
  emailIdx: index('idx_player_email').on(t.email),
  handicapGin: index('idx_player_handicap').using('gin', t.handicap),
}));
```

- Endpoints: `POST/GET/PATCH /players`, `GET /players?search=` (name/email/ghin fuzzy), `POST /players/:id/consent`, `POST /players/:id/erase` (GDPR right-to-erasure: anonymise PII, retain anonymised activity for reporting; log to audit).
- `POST /players/merge` consolidates duplicates (reassign bookings/scores/POS to surviving id).

**Testing**:
- `Unit: PrivacyBlock Zod rejects consentHistory entry missing 'regulation'`.
- `Integration: erase player → PII nulled, bookings retained with anonymised player_name, audit entry written`.
- `Integration: search by partial GHIN number → matching players returned`.
- `Integration: merge two players → all child rows repoint to survivor`.

#### 4.2 — Membership plans, memberships, billing

**What**: Define plans, enrol members, and generate dues billing cycles.

**Design** (from Data Model 1 membership tables):

```ts
export const membershipPlan = pgTable('membership_plan', { /* facilityId, name, planType,
  billingFrequency(monthly|quarterly|semi_annual|annual|one_time), baseAmount, currencyCode,
  initiationFee, includesCart, includesRange, guestPassesPerYear, maxAdvanceBookingDays, isActive */ });

export const membership = pgTable('membership', { /* playerId, membershipPlanId, facilityId,
  memberNumber, status(pending|active|suspended|expired|cancelled), startDate, endDate, renewalDate,
  billingDay(1..28), balanceDue */ });

export const membershipBilling = pgTable('membership_billing', { /* membershipId, billingPeriodStart,
  billingPeriodEnd, amount, currencyCode, status(pending|invoiced|paid|overdue|waived),
  invoiceNumber, paidAt, paymentId */ });
```

- A scheduled BullMQ job (`generateDues`, daily) creates the next `membership_billing` row for memberships whose `billingDay` matches and whose current period is closed; sets status `invoiced`.
- `maxAdvanceBookingDays` from the member's plan is enforced in `BookingService` (members book further ahead than guests).

**Testing**:
- `Unit: generateDues for monthly plan, billingDay=15, today=15 → one billing row for the coming month`.
- `Unit: generateDues idempotent — running twice same day → no duplicate billing row`.
- `Integration: member with plan.maxAdvanceBookingDays=14 books 20 days out → rejected; guest limited to 7`.

#### 4.3 — Loyalty points

**What**: Points accrual per spend and per round with tiers.

**Design**: `loyalty_program` (pointsPerDollar, pointsPerRound) and `player_loyalty` (pointsBalance, lifetimePoints, tier standard|silver|gold|platinum). Accrual hooks fire on booking completion and POS sale completion (Phase 5).

**Testing**:
- `Unit: 1 pt/$ on $85 round → 85 points accrued`.
- `Unit: lifetimePoints crossing tier threshold → tier upgraded`.

---

## Phase 5: Point of Sale & Payments

### Purpose
Add pro-shop and F&B POS with inventory and PCI-compliant payment processing via Stripe. This monetises the operation and, with Phase 3, completes the core revenue surfaces. After this phase, staff ring up sales at EMV terminals, take card/cash/member-charge payments, and bookings can be paid online — all without the platform touching raw card data.

### Tasks

#### 5.1 — Products, categories, inventory, terminals

**What**: Catalogue of products by department with inventory tracking and EMV terminal registry.

**Design** (from Data Model 1 POS section): `pos_terminal` (location pro_shop|restaurant|bar|halfway_house|snack_bar|beverage_cart|starter, deviceSerial, emvCompliant), `product_category` (department pro_shop|food|beverage|merchandise|service|rental|lesson, parentCategoryId for hierarchy), `product` (sku, unitPrice, costPrice, taxRate, trackInventory, quantityOnHand, reorderPoint, isSerialized).
- Endpoints: CRUD for products/categories/terminals; `GET /products/low-stock` (quantityOnHand ≤ reorderPoint).

**Testing**:
- `Integration: sale of trackInventory product → quantityOnHand decremented`.
- `Integration: product below reorderPoint appears in /products/low-stock`.

#### 5.2 — POS transactions & tendering

**What**: Build a sale from line items, apply discounts/tax, and tender one or more payments.

**Design** (`pos_transaction`, `pos_transaction_item`, `payment` from Data Model 1):
- `POST /pos/transactions` opens a sale; `POST /pos/transactions/:id/items` adds lines (recomputes subtotal/tax); `POST /pos/transactions/:id/discount`; `POST /pos/transactions/:id/payments` tenders.
- Transaction is finalised only when `sum(payments) >= total`; status `sale→completed`. `member_charge` payment method posts to `membership.balanceDue` instead of card.
- `payment.gatewayRef` stores the Stripe PaymentIntent id; `cardLastFour`/`cardBrand` are display-only metadata returned by Stripe — **no PAN stored** (SAQ-A scope per `standards.md`).

**Testing**:
- `Unit: 2 items @ tax 8% → subtotal/tax/total computed correctly`.
- `Integration: tender exact card payment → transaction completed`.
- `Integration: member_charge tender → membership.balanceDue increased, no Stripe call`.
- `Integration: partial payment → transaction stays open until fully tendered`.

#### 5.3 — Stripe payment integration (gateway abstraction)

**What**: A `PaymentGateway` interface implemented by Stripe for card-present (Terminal) and online (PaymentIntents) flows.

**Design**:

```ts
interface PaymentGateway {
  createPaymentIntent(p: { amount: number; currency: string; metadata: Record<string,string> }): Promise<{ id: string; clientSecret: string }>;
  capture(intentId: string): Promise<{ status: 'succeeded'|'failed'; cardLast4?: string; brand?: string }>;
  refund(intentId: string, amount?: number): Promise<{ id: string; status: string }>;
}
```

- `StripeGateway` implements the above. Webhook `POST /webhooks/stripe` (signature-verified) updates `payment.status` on `payment_intent.succeeded`/`.payment_failed`/`charge.refunded`.
- Online booking payment: `POST /bookings/:id/pay` creates an intent; on webhook success, booking moves `pending→confirmed`.

**Testing**:
- `Integration (mocked Stripe): createPaymentIntent → returns clientSecret`.
- `Integration (mocked Stripe): valid webhook signature payment_intent.succeeded → payment.status='completed'`.
- `Integration: webhook with bad signature → 400, no state change`.
- `Integration: refund → payment.refundAmount set, status='refunded'`.

---

## Phase 6: Handicap & Scoring (WHS/GHIN/iGolf)

### Purpose
Implement the WHS calculation engine and integrate with GHIN (US) and iGolf Connect (international) for score posting and handicap retrieval — a table-stakes feature and regulatory requirement per `standards.md`. After this phase, players post scores, the platform computes WHS differentials and handicap indexes, and posts to the appropriate governing body.

### Tasks

#### 6.1 — WHS calculation engine (pure logic)

**What**: A dependency-free `packages/handicap` library computing score differentials, net double bogey adjustment, and Handicap Index.

**Design**:
- `scoreDifferential(ags, courseRating, slope, pcc=0) = (113 / slope) * (ags - courseRating - pcc)` rounded to 1 dp.
- `netDoubleBogey(par, strokeIndex, courseHandicap)` caps each hole for AGS computation.
- `handicapIndex(differentials: number[])`: take best 8 of most recent 20 (fewer-score table for <20), average, apply 0.96, then soft/hard caps against `lowHandicapIndex`.
- `courseHandicap(index, slope, courseRating, par)` and `playingHandicap(courseHandicap, allowance)`.
- All constants per `standards.md` (Slope 55–155, standard 113).

```ts
export interface DifferentialInput { ags: number; courseRating: number; slope: number; pcc?: number; }
export function scoreDifferential(i: DifferentialInput): number;
export function handicapIndex(recentDifferentials: number[], lowIndex?: number): number;
```

**Testing**:
- `Unit: AGS 90, CR 71.5, slope 128 → differential 16.3` (fixture-verified against USGA worked example).
- `Unit: 20 differentials → best-8 average × 0.96, correct to 1 dp`.
- `Unit: fewer-than-20 table (e.g. 5 scores → best 1)`.
- `Unit: soft cap applies when index rises >3.0 above low index`.
- `Unit: net double bogey caps a blow-up hole correctly`.

#### 6.2 — Score posting storage

**What**: Persist score postings and hole scores aligned to GHIN fields.

**Design** (`score_posting`, `hole_score`, `handicap_history` from Data Model 1):
- `score_posting`: playerId, courseId, courseTeeId, playedAt, holesPlayed(9|18), adjustedGrossScore, scoreType(home|away|tournament|internet|penalty|combined_9), scoreDifferential, pccAdjustment, ghinScoreId, postedToGhin, source.
- On insert, compute differential via `packages/handicap`, recompute and append `handicap_history`, update `player.handicap.index`.

**Testing**:
- `Integration: post 18-hole score → differential stored, handicap_history row added, player.handicap.index updated`.
- `Integration: post combined two 9-hole scores → single 18-hole differential`.

#### 6.3 — GHIN & iGolf adapters

**What**: Adapters that post scores to and pull handicaps from the correct governing body per jurisdiction.

**Design**:
- `HandicapProvider` interface: `postScore(score): Promise<{externalScoreId}>`, `getGolfer(id): Promise<{index, ...}>`.
- `GhinProvider` (OAuth 2.0, SwaggerHub schema) and `IgolfProvider` (API key). Selection by `player.handicap.ghin` vs `.igolf` presence and facility country.
- Posting runs in a BullMQ `postScore` job with retry/backoff; on success set `postedToGhin=true`, store `ghinScoreId`; on terminal failure record an error and flag for staff.
- WHS Interoperability Standard v1.0 field mapping documented in adapter.

**Testing**:
- `Integration (mocked GHIN): postScore → externalScoreId stored, postedToGhin=true`.
- `Integration (mocked GHIN): 500 then 200 → job retries, eventually succeeds`.
- `Integration (mocked GHIN): persistent failure → score flagged, staff-visible error`.
- `Unit: provider selection picks iGolf for non-US facility with igolf id`.

---

## Phase 7: Dynamic Pricing & Revenue Management

### Purpose
Add rate codes and a dynamic-pricing rule engine that adjusts tee-time prices by occupancy, time, day, season, and (later, AI) demand — migrating yield management from airlines to tee times per `research.md`. After this phase, slot prices reflect rules in real time, with an audit trail of every computed price.

### Tasks

#### 7.1 — Rate codes & base pricing

**What**: Base green/cart fees with rate types and applicability windows.

**Design** (`rate_code` from Data Model 1): code, rateType(standard|twilight|super_twilight|senior|junior|military|replay|member_guest), baseGreenFee, baseCartFee, effectiveFrom/To, appliesWeekday/Weekend/Holiday. Resolver picks the applicable rate for a slot given date/time/player type.

**Testing**:
- `Unit: Saturday slot with weekend=true and weekday-only code → weekday code excluded`.
- `Unit: twilight rate selected after configured twilight start time`.

#### 7.2 — Dynamic pricing rule engine

**What**: Evaluate ordered rules against slot context to produce a final price with floor/ceiling.

**Design** (`dynamic_price_rule` + `price_snapshot` from Data Model 1; rule conditions as JSONB per Data Model 3):

```ts
interface PriceContext { basePrice: number; occupancyPct: number; advanceDays: number;
  dayOfWeek: number; timeOfDay: string; season: string; weatherFactor?: number; demandScore?: number; }
interface PriceRule { id: string; triggerType: 'occupancy'|'weather'|'advance_days'|'day_of_week'|'time_of_day'|'season'|'demand_forecast';
  conditions: JsonLogic; adjustmentType: 'percent'|'fixed'; adjustmentValue: number;
  minPriceFloor?: number; maxPriceCeiling?: number; priority: number; }
function computePrice(ctx: PriceContext, rules: PriceRule[]): { price: number; applied: string[] };
```

- Rules evaluated by descending `priority`; each matching rule adjusts the running price; final clamped to floor/ceiling. Every computation writes a `price_snapshot` (basePrice, computedPrice, rulesApplied[], factors) for auditability and AI training data.
- BullMQ `recomputePricing` job recomputes affected slots on booking-pace change, weather update, or rule edit; updates `tee_time_slot.currentPrice` and invalidates the tee-sheet cache.

**Testing**:
- `Unit: occupancy<30% rule → −20% applied`.
- `Unit: two matching rules → applied in priority order, result clamped to ceiling`.
- `Unit: no rule matches → price equals base`.
- `Integration: rule edit → recomputePricing updates slot.currentPrice and writes snapshot`.

---

## Phase 8: Tournaments

### Purpose
Add tournament setup, registration, pairings, and scoring with WHS-correct handicap allowances, linking competition scores back into the handicap pipeline. Serves the charity/corporate organiser persona and closes the gap that forces incumbents to bolt on a separate tournament tool.

### Tasks

#### 8.1 — Tournament setup & registration

**What**: Create tournaments, open registration, and accept entries with entry-time handicaps.

**Design** (`tournament`, `tournament_entry`, `tournament_round` from Data Model 1): tournamentType (stroke_play|match_play|stableford|scramble|best_ball|alternate_shot|chapman|shamble|charity|corporate), scoringFormat(gross|net|both), handicapAllowance, status state machine (draft→registration_open→registration_closed→in_progress→completed). Entry fee paid via the Phase 5 payment flow.

**Testing**:
- `Integration: open registration, add entry with entry fee → tournament_entry registered, payment captured`.
- `Unit: entry into registration_closed tournament → ConflictError`.

#### 8.2 — Pairings & live scoring

**What**: Generate groups/tee-time assignments and record round scores, feeding net/gross/stableford results.

**Design** (`tournament_pairing`, `tournament_pairing_player`, `tournament_score`): pairings assigned to `tee_time_slot`s (reusing Phase 3 inventory, slots flagged `tournament`). `tournament_score` links `score_posting_id` so competition scores post to handicaps. Leaderboard endpoint computes gross/net/stableford from `playing_handicap × allowance`.

**Testing**:
- `Unit: net leaderboard sorts by (gross − playingHandicap)`.
- `Unit: stableford points computed from net strokes vs par per hole`.
- `Integration: submit round score → leaderboard updates, score_posting created and queued for GHIN`.

---

## Phase 9: Reporting & Analytics Foundation (fact tables)

### Purpose
Introduce the time-partitioned fact-table layer (Data Model 4) that powers operator dashboards and is the substrate for the AI phase. After this phase, operators see revenue, rounds, occupancy, and member-activity dashboards, and dense time-indexed data is captured for ML.

### Tasks

#### 9.1 — Fact tables & emission

**What**: Monthly-partitioned fact tables populated as a side effect of operational writes.

**Design** (Data Model 4): `fact_revenue_daily` (green_fee, cart_fee, pro_shop, f_and_b, tournament, membership_dues, total, rounds_played, unique_players, occupancy_pct), `fact_booking_event`, `fact_price_snapshot` (mirrors price_snapshots for ML), `fact_player_engagement` (per-player rounds/spend/last_visit). Range-partition by month; a scheduled job pre-creates next month's partitions. Operational services emit fact rows in the same transaction (booking complete, POS complete, dues paid).

**Testing**:
- `Integration: completed booking + POS sale → fact_revenue_daily row reflects both`.
- `Integration: partition for current month exists; job creates next month's`.
- `Integration: fact emission shares the operational transaction (rollback removes both)`.

#### 9.2 — Reporting endpoints & dashboards

**What**: Aggregate endpoints and admin dashboard views.

**Design**: `GET /reports/revenue?from=&to=&groupBy=day|week|month`, `/reports/utilization`, `/reports/members/activity`. Admin SPA renders revenue, occupancy heatmap, and member-activity charts. Cross-tenant queries blocked by RLS.

**Testing**:
- `Integration: revenue report for date range → totals match summed fact rows`.
- `Integration: report request under tenant A excludes tenant B data`.

---

## Phase 10: AI-Native Features

### Purpose
Deliver the differentiators from `research.md`: demand-based pricing, member-retention prediction, course-conditions narrative generation, and predictive maintenance — built on the fact tables and the Python ml-service. After this phase the platform actively makes operational recommendations, not just reports.

### Tasks

#### 10.1 — Demand forecasting → dynamic pricing

**What**: An ML model predicting booking demand per slot, feeding a `demandScore` into the Phase 7 pricing engine.

**Design**: `ml-service` `POST /forecast/demand` takes slot features (historical occupancy, day/time, season, weather forecast, advance days, competitor signal) and returns `demandScore 0–1` (LightGBM regressor trained on `fact_booking_event` + `fact_price_snapshot`). A scheduled BullMQ job scores upcoming slots; `demandScore` enters `PriceContext`, enabling `demand_forecast` rules.

**Testing**:
- `Unit (Python): model.predict on fixture features → score in [0,1]`.
- `Integration: high-demand forecast + demand rule → slot price increases within ceiling`.
- `Integration (mocked ml-service): forecast job populates demandScore on upcoming slots`.

#### 10.2 — Member-retention (churn) prediction

**What**: Flag at-risk members from declining visits/spend and trigger retention campaigns.

**Design**: `ml-service` `POST /retention/score` over `fact_player_engagement` (visit frequency trend, spend trend, days since last visit, tenure) returns `churnRisk 0–1`. A scheduled job sets `player`/read-model `at_risk_flag`; high-risk members enqueue a Phase 11 retention campaign.

**Testing**:
- `Unit (Python): member with declining visits → higher churnRisk than steady member`.
- `Integration: scoring job flags at-risk members and enqueues winback campaign`.

#### 10.3 — Course-conditions narrative & predictive maintenance

**What**: Fuse staff reports + weather into guest-facing condition updates (LLM), and predict maintenance needs from usage/weather.

**Design**: `course_condition_report` and `maintenance_task` tables (Data Model 1). Conditions endpoint pulls latest report + weather API, calls the LLM (Vercel AI SDK) with a templated prompt to produce a guest-facing summary; staff approve before publishing. Maintenance model (ml-service) predicts task urgency from rounds-played per hole + weather stress.

LLM prompt template (structure):
```
System: You write concise, guest-facing golf course condition updates. Factual, friendly, <80 words.
User: Course: {courseName}. Date: {date}. Greens: {greenStatus} (stimp {greenSpeed}).
Fairways: {fairwayStatus}. Bunkers: {bunkerStatus}. Cart policy: {cartPolicy}.
Weather: {tempF}°F, wind {windMph}mph, {condition}. Staff notes: {notes}.
```

**Testing**:
- `Integration (mocked LLM): condition report → draft summary generated, unpublished until approved`.
- `Unit (Python): hole with high rounds + heat stress → elevated maintenance urgency`.

---

## Phase 11: Marketing, Marketplace Distribution & Public Booking

### Purpose
Add outbound marketing/retention campaigns, third-party distribution (GolfNow, Reserve with Google), and the public booking website/widget — completing the customer-acquisition loop and the operator's independence from any single marketplace.

### Tasks

#### 11.1 — Marketing campaigns

**What**: Segmented email/SMS campaigns with delivery tracking, including AI-triggered winback.

**Design** (`marketing_campaign`, `campaign_recipient` from Data Model 1): campaignType (email|sms|push|in_app|social|retention|winback), segment resolution against player attributes/fact tables, send via BullMQ `sendCampaign` job through an email/SMS provider (e.g. SendGrid/Twilio behind an interface). Respect `privacy.marketingOptIn` and CCPA/GDPR.

**Testing**:
- `Integration: campaign to opted-in segment → only consented players receive; opted-out excluded`.
- `Integration (mocked provider): send → recipient status transitions sent→delivered`.

#### 11.2 — Marketplace & Reserve with Google distribution

**What**: Sync availability/prices to external channels and ingest external bookings.

**Design** (`marketplace_listing`, `api_webhook` from Data Model 1): per-channel listing config with commission rate and sync status. BullMQ `syncMarketplace` pushes slot availability/price; inbound bookings arrive via `/partner/bookings` (OAuth from Phase 2) tagged `bookingSource='marketplace'` with `marketplaceRef`. Reserve with Google booking feed generated per its spec; webhooks fire `booking.created`/`score.posted` to subscribed partners with HMAC signatures.

**Testing**:
- `Integration (mocked GolfNow): availability sync → external feed reflects open slots`.
- `Integration: inbound marketplace booking via partner API → booking created with marketplaceRef, decrements slot`.
- `Integration: webhook delivery → signed payload POSTed; failure increments failure_count and retries`.

#### 11.3 — Public booking site & embeddable widget

**What**: Player-facing booking flow (Next.js site + embeddable React widget) consuming the generated SDK.

**Design**: Browse course → pick date → see priced slots (Phase 7) → select players/add-ons → pay (Phase 5 online intent) → confirmation. Widget is a script-embeddable bundle calling the public API with a facility public key. SEO-friendly SSR pages for course/availability.

**Testing**:
- `E2E (Playwright): open booking site → select slot → pay (test card) → confirmation code shown, booking confirmed in DB`.
- `E2E: attempt to book a full slot → UI shows unavailable, no booking created`.

---

## Phase 12: Hardening — Compliance, Audit, OpenAPI Publication, Ops

### Purpose
Make the platform production-ready: complete audit logging, finalise PCI/GDPR/CCPA posture, publish the public OpenAPI spec and SDK, and add observability and rate limiting across the surface. After this phase the system is deployable for real operators.

### Tasks

#### 12.1 — Audit log & compliance

**What**: System-wide audit trail and compliance endpoints.

**Design** (`audit_log` with JSONB diff from Data Model 1): every mutating service writes `{actorId, actorType, action, entityType, entityId, oldValues, newValues, ip}`. GDPR endpoints: data export (portability) and the erase flow from 4.1; CCPA opt-out honoured in marketing. PCI: confirm no PAN anywhere, document SAQ-A scope.

**Testing**:
- `Integration: update a booking → audit_log row with old/new JSONB diff`.
- `Integration: data export for player → JSON bundle of all their records`.
- `Test: grep codebase/schema for card-number storage → none (SAQ-A assertion)`.

#### 12.2 — OpenAPI publication & SDK generation

**What**: Publish the OpenAPI 3.1 spec and generate the TypeScript SDK used by the widget and partners.

**Design**: `GET /openapi.json` complete and validated in CI (`@redocly/cli lint`). `packages/sdk` generated from the spec (openapi-typescript + a thin fetch client). A Redoc-rendered developer portal page is served at `/docs`.

**Testing**:
- `CI: redocly lint openapi.json → no errors`.
- `Integration: generated SDK call to GET /facilities → typed response matches runtime`.

#### 12.3 — Observability, rate limiting, deploy

**What**: Logging/metrics, global rate limiting, and production Docker/compose.

**Design**: pino structured logs shipped to stdout; `/metrics` Prometheus endpoint (request latency, queue depth, GHIN post success rate). `@fastify/rate-limit` global + per-partner. Production `Dockerfile` multi-stage; healthchecks for k8s/compose; graceful shutdown draining BullMQ.

**Testing**:
- `Integration: /metrics exposes request and queue metrics`.
- `Integration: exceed global rate limit → 429`.
- `Smoke: production image boots, /health 200, graceful SIGTERM drains jobs`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB core, RLS, OpenAPI)   ─── required by everything
    │
Phase 2: Identity / Auth / RBAC                          ─── requires 1
    │
Phase 3: Tee Sheet & Booking ───┐                        ─── requires 2
Phase 4: Players / CRM / Member ─┤ (4 can parallel 3)    ─── requires 2
    │                            │
    ├── Phase 5: POS & Payments  ─── requires 3,4
    ├── Phase 6: Handicap/Scoring ── requires 4 (+3 for tournaments later)  ── can parallel 5
    └── Phase 7: Dynamic Pricing ─── requires 3 (rate codes) ── can parallel 5,6
         │
Phase 8: Tournaments            ─── requires 3,5,6
Phase 9: Reporting / Fact tables ── requires 3,4,5 (+7 snapshots)
    │
Phase 10: AI-Native Features    ─── requires 7,9 (+ ml-service)
    │
Phase 11: Marketing / Marketplace / Public Booking ── requires 5,7,10 (winback uses 10.2)
    │
Phase 12: Hardening / Compliance / OpenAPI / Ops ── requires all; runs partly continuously
```

**Parallelism opportunities:**
- **Phases 3 and 4** can be developed concurrently once Phase 2 lands (different table groups; booking-player linkage is the only join).
- **Phases 5, 6, and 7** can proceed in parallel after 3+4 (POS, handicap engine, and pricing engine are independent modules).
- The **Python ml-service** (used in Phase 10) can be scaffolded any time after Phase 9 produces fact data.
- **Phase 12** tasks (audit logging, OpenAPI hygiene) are best applied incrementally from Phase 3 onward rather than deferred entirely.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass (`pnpm test`), including Testcontainers-backed real-Postgres tests for any data-layer work.
3. ESLint and Prettier pass (`pnpm lint`).
4. Type checking passes (`pnpm typecheck`, `tsc --noEmit`).
5. Docker build succeeds and `docker compose up` reaches healthy for all affected services.
6. The phase's feature works end-to-end (demonstrated by an integration or Playwright E2E test).
7. New configuration/env vars added to `.env.example` and documented.
8. New API endpoints appear in the generated `/openapi.json` and pass `redocly lint`.
9. Drizzle migrations generated, reviewed, and applied cleanly on a fresh database (including hand-written RLS policies, CHECK constraints, GIN indexes, and partitions).
10. RLS verified for any new tenant-scoped table (cross-tenant access returns no rows).
11. For payment/handicap/privacy work: compliance assertions hold (no PAN stored; consent respected; WHS formulas match worked examples).
12. Audit logging added for any new mutating endpoints.
```
