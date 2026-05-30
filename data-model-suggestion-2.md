# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Golf Course Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to an event store. The current state of any entity — a booking, a membership, a score posting, a POS transaction — is derived by replaying the events that produced it. Read models (materialised views) are projected from the event stream to serve queries efficiently. This is the Command Query Responsibility Segregation (CQRS) pattern.

Event sourcing is standard practice in financial systems, audit-heavy regulatory environments, and any domain where "what happened and when" matters as much as "what is true now." For a golf course management platform, this approach provides a complete audit trail of every booking modification, every price change, every score revision, and every membership status transition — without a separate audit log table bolted on after the fact. The event store *is* the audit trail.

The golf domain has strong temporal requirements: WHS handicap indexes are revised periodically based on the best 8 of the last 20 score differentials; dynamic pricing changes throughout the day; membership statuses transition through renewal cycles; and regulatory bodies (USGA/GHIN, R&A) may audit score submissions. An event-sourced design answers temporal questions natively: "What was this player's handicap on March 15?" or "What price was displayed when this booking was confirmed?" become simple event replay queries rather than complex historical table reconstructions.

**Best for:** Operators who need complete audit trails, regulatory compliance with GHIN/WHS score submission, temporal querying for AI-driven analytics, and the ability to replay/reconstruct any past state.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is permanently recorded
- (+) Temporal queries answered natively — replay to any point in time
- (+) AI/ML training data — rich event streams feed demand forecasting and member retention models
- (+) Debugging and dispute resolution — "show me exactly what happened to this booking"
- (+) Event replay enables schema evolution — new read models can be built from existing events
- (-) Higher complexity — developers must understand event sourcing and eventual consistency
- (-) Read model staleness — materialised views lag behind writes (typically milliseconds, but must be designed for)
- (-) Storage growth — event store grows indefinitely (mitigated by snapshots and partitioning)
- (-) Simple CRUD queries require projections — cannot just SELECT from a table
- (-) Testing requires event fixture setup rather than simple row insertion

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| World Handicap System (WHS) v1.0 | Score events capture all WHS-required fields; handicap index revision is an event projected from the score event stream; historical index at any date is a temporal replay query |
| GHIN (USGA) | `ScorePostedToGHIN` event records the confirmation; GHIN sync failures produce `GHINPostingFailed` events for retry processing |
| PCI DSS | Payment events store only gateway tokens — no card data enters the event store |
| GDPR | `PlayerConsentGranted`/`PlayerConsentRevoked` events provide a timestamped consent trail; `PlayerDataErased` event supports right-to-erasure with crypto-shredding on the related events |
| ISO 4217 | All monetary amounts in events include explicit `currency_code` |
| ISO 3166-1 | Jurisdiction codes follow ISO 3166-1 alpha-2 in facility and player events |
| OpenAPI 3.1 | Read model projections map to REST API resources; event types documented as webhook event schemas |

---

## Event Store — Core Infrastructure

```sql
-- ============================================================
-- EVENT STORE: Immutable append-only event log
-- ============================================================

CREATE TABLE es_aggregate (
    aggregate_id    UUID PRIMARY KEY,
    aggregate_type  VARCHAR(100) NOT NULL,           -- 'Booking', 'Player', 'Membership', 'TeeTimeSlot', etc.
    tenant_id       UUID NOT NULL,
    current_version BIGINT NOT NULL DEFAULT 0,       -- optimistic concurrency control
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_aggregate_type ON es_aggregate(aggregate_type);
CREATE INDEX idx_aggregate_tenant ON es_aggregate(tenant_id);

CREATE TABLE es_event (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id    UUID NOT NULL REFERENCES es_aggregate(aggregate_id),
    aggregate_type  VARCHAR(100) NOT NULL,
    event_type      VARCHAR(200) NOT NULL,           -- e.g. 'BookingCreated', 'BookingCancelled', 'ScorePosted'
    event_version   BIGINT NOT NULL,                 -- sequential per aggregate
    tenant_id       UUID NOT NULL,
    payload         JSONB NOT NULL,                  -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',     -- actor_id, ip_address, correlation_id, causation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)             -- ensures ordering & prevents conflicts
) PARTITION BY RANGE (created_at);

-- Partition by month for manageable storage and efficient temporal queries
CREATE TABLE es_event_2026_01 PARTITION OF es_event FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE es_event_2026_02 PARTITION OF es_event FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE es_event_2026_03 PARTITION OF es_event FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
-- ... additional monthly partitions created by scheduled job

CREATE INDEX idx_event_aggregate ON es_event(aggregate_id, event_version);
CREATE INDEX idx_event_type ON es_event(event_type);
CREATE INDEX idx_event_tenant ON es_event(tenant_id);
CREATE INDEX idx_event_created ON es_event(created_at);

-- Snapshot table for performance: avoids replaying full event history
CREATE TABLE es_snapshot (
    aggregate_id    UUID NOT NULL REFERENCES es_aggregate(aggregate_id),
    aggregate_type  VARCHAR(100) NOT NULL,
    snapshot_version BIGINT NOT NULL,
    state           JSONB NOT NULL,                  -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);
```

## Event Type Catalogue

The following event types form the domain vocabulary. Each event's `payload` JSONB follows a documented schema.

### Booking Aggregate Events

```sql
-- Example: BookingCreated event payload
-- {
--   "tee_time_slot_id": "uuid",
--   "course_id": "uuid",
--   "facility_id": "uuid",
--   "slot_date": "2026-06-15",
--   "slot_time": "08:24",
--   "booked_by_player_id": "uuid",
--   "booking_source": "online",
--   "channel_id": "uuid | null",
--   "group_size": 4,
--   "players": [
--     {"player_id": "uuid", "player_name": "John Smith", "player_type_id": "uuid", "green_fee": 85.00, "cart_fee": 25.00}
--   ],
--   "total_green_fee": 340.00,
--   "total_cart_fee": 100.00,
--   "currency_code": "USD",
--   "confirmation_code": "GC-2026-A7X9",
--   "pricing_rule_id": "uuid",
--   "price_at_booking": 85.00
-- }

-- BookingCreated         — initial reservation
-- BookingPlayerAdded     — player added to existing group
-- BookingPlayerRemoved   — player removed from group
-- BookingRescheduled     — moved to different tee time
-- BookingCheckedIn       — group arrived at starter
-- BookingPlayerCheckedIn — individual player checked in
-- BookingCompleted       — round finished
-- BookingCancelled       — cancellation with reason
-- BookingNoShow          — group did not appear
-- BookingRefundIssued    — refund processed
```

### Player Aggregate Events

```sql
-- PlayerRegistered        — new player created
-- PlayerProfileUpdated    — name, email, phone changed
-- PlayerHandicapUpdated   — WHS index revised (captures old and new index)
-- PlayerGHINLinked        — GHIN number associated
-- PlayerConsentGranted    — GDPR/CCPA consent recorded
-- PlayerConsentRevoked    — consent withdrawn
-- PlayerDataErased        — right-to-erasure executed
-- PlayerMerged            — duplicate records consolidated

-- Example: PlayerHandicapUpdated payload
-- {
--   "ghin_number": "1234567",
--   "previous_index": 12.4,
--   "new_index": 11.8,
--   "revision_date": "2026-06-01",
--   "differentials_used": [10.2, 11.5, 12.0, 11.1, 10.8, 12.3, 11.7, 10.9],
--   "source": "ghin_sync"
-- }
```

### Membership Aggregate Events

```sql
-- MembershipCreated       — new membership started
-- MembershipRenewed       — renewal processed
-- MembershipSuspended     — put on hold
-- MembershipReactivated   — resumed from suspension
-- MembershipCancelled     — terminated
-- MembershipDuesCharged   — billing event
-- MembershipDuesPaymentReceived — payment confirmed
-- MembershipDuesPaymentFailed   — payment declined
-- MembershipTypeChanged   — upgrade/downgrade
-- MembershipTransferred   — ownership change

-- Example: MembershipDuesCharged payload
-- {
--   "player_id": "uuid",
--   "facility_id": "uuid",
--   "amount": 450.00,
--   "currency_code": "USD",
--   "billing_period_start": "2026-07-01",
--   "billing_period_end": "2026-07-31",
--   "billing_cycle": "monthly",
--   "membership_type": "Full Playing"
-- }
```

### Score Posting Aggregate Events

```sql
-- ScoreSubmitted          — player submits a score
-- ScoreValidated          — WHS differential calculated
-- ScorePostedToGHIN       — successfully posted to GHIN
-- GHINPostingFailed       — GHIN submission error
-- ScoreRevised            — score corrected by committee
-- ScoreWithdrawn          — score removed (with reason)

-- Example: ScoreSubmitted payload
-- {
--   "player_id": "uuid",
--   "course_id": "uuid",
--   "tee_set_id": "uuid",
--   "played_date": "2026-06-15",
--   "holes_played": 18,
--   "adjusted_gross_score": 82,
--   "score_type": "home",
--   "course_rating": 71.5,
--   "slope_rating": 128,
--   "pcc_adjustment": 0.0,
--   "score_differential": 9.3,
--   "hole_scores": [4,5,3,4,5,4,3,4,5, 4,4,3,5,4,4,3,5,4],
--   "attestor_player_id": "uuid"
-- }
```

### Tee Time Slot Aggregate Events

```sql
-- TeeTimeSlotCreated      — slot generated for tee sheet
-- TeeTimeSlotBlocked      — blocked by staff (maintenance, shotgun start)
-- TeeTimeSlotUnblocked    — block removed
-- TeeTimeSlotPriceChanged — dynamic pricing update
-- TeeTimeSlotCapacityChanged — max players adjusted

-- Example: TeeTimeSlotPriceChanged payload
-- {
--   "course_id": "uuid",
--   "slot_date": "2026-06-15",
--   "slot_time": "10:00",
--   "previous_price": 95.00,
--   "new_price": 75.00,
--   "pricing_rule_id": "uuid",
--   "trigger": "occupancy_below_threshold",
--   "occupancy_pct": 35,
--   "weather_factor": "rain_forecast",
--   "currency_code": "USD"
-- }
```

### POS Transaction Aggregate Events

```sql
-- POSTransactionStarted   — new sale initiated
-- POSLineItemAdded        — product scanned/added
-- POSLineItemRemoved      — item voided from sale
-- POSDiscountApplied      — discount applied
-- POSPaymentReceived      — payment tendered
-- POSTransactionCompleted — sale finalised
-- POSTransactionVoided    — entire transaction voided
-- POSRefundIssued         — return processed

-- POSPaymentReceived payload
-- {
--   "register_id": "uuid",
--   "payment_method": "card_tap",
--   "gateway": "stripe",
--   "gateway_txn_id": "pi_3ABC123",
--   "amount": 47.50,
--   "currency_code": "USD",
--   "tip_amount": 5.00
-- }
```

### Tournament Aggregate Events

```sql
-- TournamentCreated       — event set up
-- TournamentRegistrationOpened  — entries accepted
-- TournamentEntryReceived — player registered
-- TournamentEntryWithdrawn — player withdrew
-- TournamentPairingsGenerated — groups assigned
-- TournamentRoundStarted  — play began
-- TournamentScoreRecorded — live scoring update
-- TournamentRoundCompleted — round finished
-- TournamentResultsPublished — leaderboard finalised
-- TournamentCompleted     — event concluded
```

### Facility & Course Events

```sql
-- FacilityRegistered      — new facility onboarded
-- CourseConditionReported  — daily conditions update
-- WeatherSnapshotRecorded — weather data captured
-- MaintenanceScheduled    — turf/equipment maintenance
-- MaintenanceCompleted    — work finished
```

---

## Read Model Projections (Materialised Views)

Read models are populated by event processors that consume the event stream and maintain denormalised query-optimised tables.

```sql
-- ============================================================
-- READ MODEL: Current Booking State
-- ============================================================

CREATE TABLE rm_booking (
    booking_id      UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    course_id       UUID NOT NULL,
    course_name     VARCHAR(200),
    slot_date       DATE NOT NULL,
    slot_time       TIME NOT NULL,
    booked_by_name  VARCHAR(200),
    booked_by_email VARCHAR(200),
    booking_source  VARCHAR(50),
    channel_name    VARCHAR(100),
    group_size      SMALLINT,
    status          VARCHAR(20),
    confirmation_code VARCHAR(20),
    total_green_fee NUMERIC(10,2),
    total_cart_fee  NUMERIC(10,2),
    currency_code   CHAR(3),
    price_at_booking NUMERIC(10,2),
    booked_at       TIMESTAMPTZ,
    checked_in_at   TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    last_event_version BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_booking_tenant_date ON rm_booking(tenant_id, slot_date);
CREATE INDEX idx_rm_booking_status ON rm_booking(status);

-- ============================================================
-- READ MODEL: Tee Sheet (denormalised for fast display)
-- ============================================================

CREATE TABLE rm_tee_sheet (
    slot_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    course_id       UUID NOT NULL,
    course_name     VARCHAR(200),
    slot_date       DATE NOT NULL,
    slot_time       TIME NOT NULL,
    max_players     SMALLINT,
    booked_players  SMALLINT DEFAULT 0,
    status          VARCHAR(20),
    starting_hole   SMALLINT DEFAULT 1,
    current_price   NUMERIC(10,2),
    currency_code   CHAR(3),
    bookings        JSONB DEFAULT '[]',              -- [{booking_id, player_names, group_size, status}]
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_tee_sheet_date ON rm_tee_sheet(course_id, slot_date, slot_time);

-- ============================================================
-- READ MODEL: Player Profile (denormalised)
-- ============================================================

CREATE TABLE rm_player (
    player_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    full_name       VARCHAR(200),
    email           VARCHAR(200),
    phone           VARCHAR(30),
    ghin_number     VARCHAR(20),
    handicap_index  NUMERIC(4,1),
    handicap_trend  NUMERIC(4,1),                   -- change over last 3 revisions
    membership_status VARCHAR(20),
    membership_type VARCHAR(100),
    loyalty_points  INTEGER DEFAULT 0,
    total_rounds    INTEGER DEFAULT 0,
    total_spend     NUMERIC(12,2) DEFAULT 0,
    last_visit_date DATE,
    at_risk_flag    BOOLEAN DEFAULT false,           -- AI: declining engagement
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_player_tenant ON rm_player(tenant_id);
CREATE INDEX idx_rm_player_email ON rm_player(email);
CREATE INDEX idx_rm_player_ghin ON rm_player(ghin_number);

-- ============================================================
-- READ MODEL: Revenue Dashboard
-- ============================================================

CREATE TABLE rm_revenue_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    facility_id     UUID NOT NULL,
    revenue_date    DATE NOT NULL,
    green_fee_revenue NUMERIC(12,2) DEFAULT 0,
    cart_fee_revenue NUMERIC(12,2) DEFAULT 0,
    pro_shop_revenue NUMERIC(12,2) DEFAULT 0,
    f_and_b_revenue NUMERIC(12,2) DEFAULT 0,
    tournament_revenue NUMERIC(12,2) DEFAULT 0,
    membership_dues_revenue NUMERIC(12,2) DEFAULT 0,
    total_revenue   NUMERIC(12,2) DEFAULT 0,
    rounds_played   INTEGER DEFAULT 0,
    unique_players  INTEGER DEFAULT 0,
    avg_green_fee   NUMERIC(10,2),
    occupancy_pct   NUMERIC(5,1),
    currency_code   CHAR(3) DEFAULT 'USD',
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, facility_id, revenue_date)
);

-- ============================================================
-- READ MODEL: Handicap History (for WHS temporal queries)
-- ============================================================

CREATE TABLE rm_handicap_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id       UUID NOT NULL,
    revision_date   DATE NOT NULL,
    handicap_index  NUMERIC(4,1) NOT NULL,
    differentials   JSONB,                          -- array of the 20 most recent differentials
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_handicap_player_date ON rm_handicap_history(player_id, revision_date);
```

## Example Temporal Query — "What Was the Price When This Booking Was Made?"

```sql
-- Replay: find the tee time price at the moment a booking was created
SELECT
    e.payload->>'new_price' AS price_at_time,
    e.payload->>'pricing_rule_id' AS rule_applied,
    e.payload->>'trigger' AS price_trigger,
    e.created_at AS price_set_at
FROM es_event e
WHERE e.aggregate_id = :tee_time_slot_aggregate_id
  AND e.event_type = 'TeeTimeSlotPriceChanged'
  AND e.created_at <= (
      SELECT created_at FROM es_event
      WHERE aggregate_id = :booking_aggregate_id
        AND event_type = 'BookingCreated'
      LIMIT 1
  )
ORDER BY e.event_version DESC
LIMIT 1;
```

## Example Temporal Query — "Player Handicap on a Specific Date"

```sql
-- Replay: find the handicap index on a specific historical date
SELECT
    e.payload->>'new_index' AS handicap_index,
    e.payload->>'revision_date' AS revision_date,
    e.created_at
FROM es_event e
WHERE e.aggregate_id = :player_aggregate_id
  AND e.event_type = 'PlayerHandicapUpdated'
  AND e.created_at <= '2026-03-15'::timestamptz
ORDER BY e.event_version DESC
LIMIT 1;
```

## Example: Rebuilding a Read Model from Scratch

```sql
-- If the rm_revenue_daily projection is corrupted or a new projection
-- is needed, rebuild from the event stream:
--
-- 1. TRUNCATE rm_revenue_daily;
-- 2. Process all events of types:
--    BookingCreated, BookingCancelled, BookingRefundIssued,
--    POSTransactionCompleted, POSTransactionVoided, POSRefundIssued,
--    MembershipDuesPaymentReceived
-- 3. For each event, update the appropriate revenue_date row
--
-- This is the key advantage: new read models (e.g. rm_channel_performance,
-- rm_weather_revenue_correlation) can be added at any time by replaying
-- existing events without modifying the write side.
```

---

## Reference Data Tables (Non-Event-Sourced)

Some data is static reference data that does not benefit from event sourcing:

```sql
CREATE TABLE ref_course (
    id              UUID PRIMARY KEY,
    facility_id     UUID NOT NULL,
    name            VARCHAR(200) NOT NULL,
    holes           SMALLINT NOT NULL,
    par             SMALLINT NOT NULL,
    ncrdb_course_id VARCHAR(50),
    tee_sets        JSONB NOT NULL DEFAULT '[]',
    -- [{name, color, gender, course_rating, slope_rating, bogey_rating, yardage}]
    holes_config    JSONB NOT NULL DEFAULT '[]',
    -- [{hole_number, par, stroke_index, tee_yardages: {Blue: 425, White: 400}}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_product (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    sku             VARCHAR(50),
    name            VARCHAR(200) NOT NULL,
    category        VARCHAR(100),
    unit_price      NUMERIC(10,2) NOT NULL,
    cost_price      NUMERIC(10,2),
    tax_rate_pct    NUMERIC(5,2) DEFAULT 0,
    currency_code   CHAR(3) DEFAULT 'USD',
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_pricing_rule (
    id              UUID PRIMARY KEY,
    facility_id     UUID NOT NULL,
    name            VARCHAR(200) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    conditions      JSONB NOT NULL,                 -- [{type, operator, value}]
    action          JSONB NOT NULL,                 -- {type, fee_type, amount}
    is_active       BOOLEAN DEFAULT true,
    valid_from      DATE,
    valid_until     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_player_type (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(100) NOT NULL,
    is_member       BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_distribution_channel (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL,
    commission_pct  NUMERIC(5,2),
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store Infrastructure | 3 | es_aggregate, es_event (partitioned), es_snapshot |
| Read Models | 5 | rm_booking, rm_tee_sheet, rm_player, rm_revenue_daily, rm_handicap_history |
| Reference Data | 5 | ref_course, ref_product, ref_pricing_rule, ref_player_type, ref_distribution_channel |
| **Total** | **~13** | Plus monthly partitions of es_event |

---

## Key Design Decisions

1. **Single event store table (partitioned by month)** — all domain events live in `es_event` with aggregate type and event type as discriminators. Monthly partitioning keeps query performance manageable and enables time-based archiving.

2. **JSONB payloads with documented schemas** — event payloads are stored as JSONB, giving flexibility for schema evolution (new fields can be added without breaking existing events). Each event type has a documented JSON Schema that new code must comply with.

3. **Optimistic concurrency via aggregate versioning** — `es_aggregate.current_version` and `es_event.event_version` provide a unique sequence per aggregate. Concurrent writes to the same aggregate fail with a version conflict and must retry.

4. **Snapshots for performance** — after every N events (e.g., 100), a snapshot of the aggregate state is stored. Replay starts from the latest snapshot rather than from event #1, keeping rebuild times bounded.

5. **Read models are disposable and rebuildable** — if a read model is corrupted or a new analytical view is needed, it can be (re)built by replaying the event stream. This is the primary advantage for AI/ML use cases: new training datasets can be created from historical events without modifying the write path.

6. **Reference data stays relational** — course configurations, products, and pricing rules are relatively static and do not need event sourcing. They are stored in conventional tables referenced by events via UUID.

7. **Crypto-shredding for GDPR compliance** — player events can be encrypted with a per-player key. When erasure is requested, the key is destroyed, rendering the event payloads unreadable while preserving the event sequence for aggregate integrity.

8. **Event metadata captures audit context** — every event includes `metadata` with actor ID, IP address, correlation ID, and causation ID, providing full traceability without a separate audit log.
