# Data Model Suggestion 4: Analytics-First / Time-Series Hybrid

> Project: Golf Course Management · Created: 2026-05-22

## Philosophy

This model is designed around a central thesis: the highest-value differentiator in golf course management software is AI-driven analytics — dynamic pricing, demand forecasting, member retention modelling, weather-correlated revenue analysis, and pace-of-play optimisation. Instead of bolting analytics onto an operational schema as an afterthought, this architecture treats time-series data as a first-class citizen alongside the operational relational core.

The operational layer uses a compact normalised relational schema for day-to-day CRUD (bookings, players, memberships, POS). The analytics layer uses time-partitioned fact tables (inspired by dimensional modelling / star schema patterns) that capture every measurable event at the granularity needed for AI/ML training: per-slot pricing snapshots, per-round weather conditions, per-player engagement metrics, and per-day revenue breakdowns. These fact tables are partitioned by time (monthly or daily) for efficient range queries and automatic archiving, and they are designed to be consumed directly by machine learning pipelines without complex ETL.

This pattern is standard in industries where real-time pricing and demand forecasting drive revenue — airlines, hotels, ride-sharing — but is not yet common in golf course management. The AI-native opportunity identified in the project research (demand-based tee-time pricing, member retention prediction, predictive maintenance) requires exactly this kind of data infrastructure: dense, time-indexed, multi-dimensional fact tables that correlate bookings with weather, pricing decisions with occupancy outcomes, and member activity with retention events.

**Best for:** Teams prioritising AI-driven revenue optimisation, demand forecasting, and data-informed operational decisions. Ideal when the platform's competitive advantage comes from analytics rather than just operational efficiency.

**Trade-offs:**

- (+) Purpose-built for AI/ML training — fact tables map directly to feature vectors without ETL
- (+) Time-partitioned storage enables efficient range queries and automatic archiving
- (+) Revenue and demand analytics are first-class queries, not complex JOINs across operational tables
- (+) Weather-revenue-occupancy correlation is a native query pattern, not a custom report
- (+) Supports real-time dynamic pricing with sub-second price computation from pre-aggregated data
- (+) Historical data can be compressed and archived per partition without affecting operational queries
- (-) Higher storage footprint — fact tables duplicate some operational data for query performance
- (-) Requires disciplined event emission — operational writes must also emit fact records
- (-) Two mental models — developers must understand both the operational OLTP layer and the analytical OLAP layer
- (-) Fact tables need periodic maintenance (partition creation, compression, archiving)
- (-) More complex than a single normalised schema for teams not doing analytics

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WHS Interoperability Standard v1.0 | Score differentials and handicap revisions stored as time-series facts for trend analysis; WHS calculation fields on operational score tables |
| GHIN (USGA) | GHIN sync events captured in `fact_integration_event` for reliability monitoring; handicap index history in `fact_handicap_revision` |
| USGA Course Rating (NCRDB) | Course Rating and Slope Rating stored on operational `course_tee` table; used in score differential calculations that feed the handicap fact table |
| ISO 3166-1 | Country codes on facility and player dimensions; used for geographic segmentation in analytics |
| ISO 4217 | Currency codes on all monetary fact columns; enables multi-currency revenue aggregation |
| PCI DSS | No card data in operational or analytical tables; payment facts reference tokenised gateway IDs only |
| OpenAPI 3.1 | Operational tables map to REST API resources; analytics endpoints serve aggregated fact data |
| TimescaleDB / PostgreSQL Partitioning | Fact tables use native PostgreSQL range partitioning by month; compatible with TimescaleDB hypertables for automatic partition management |

---

## Operational Layer — Compact Relational Core

The operational layer handles day-to-day CRUD. It is intentionally compact — detailed analytics live in the fact tables.

```sql
-- =====================================================
-- OPERATIONAL: FACILITY & COURSE
-- =====================================================

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            VARCHAR(200) NOT NULL,
    facility_type   VARCHAR(30) NOT NULL CHECK (facility_type IN ('public','private','semi_private','resort','municipal')),
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1
    timezone        VARCHAR(50) NOT NULL,             -- IANA timezone
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    config          JSONB NOT NULL DEFAULT '{}',      -- operational config (hours, tax rates, etc.)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_facility_tenant ON facility(tenant_id);

CREATE TABLE course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    name            VARCHAR(200) NOT NULL,
    holes_count     SMALLINT NOT NULL CHECK (holes_count IN (9, 18, 27, 36)),
    par             SMALLINT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_facility ON course(facility_id);

CREATE TABLE course_tee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    tee_name        VARCHAR(50) NOT NULL,
    tee_colour      VARCHAR(20),
    gender          CHAR(1) CHECK (gender IN ('M','F','X')),
    course_rating   NUMERIC(4,1) NOT NULL,            -- USGA Course Rating
    slope_rating    SMALLINT NOT NULL CHECK (slope_rating BETWEEN 55 AND 155),
    bogey_rating    NUMERIC(4,1),
    total_yardage   SMALLINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, tee_name, gender)
);

-- =====================================================
-- OPERATIONAL: TEE SHEET & BOOKING
-- =====================================================

CREATE TABLE tee_time_slot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    slot_date       DATE NOT NULL,
    slot_time       TIME NOT NULL,
    starting_hole   SMALLINT NOT NULL DEFAULT 1,
    max_players     SMALLINT NOT NULL DEFAULT 4,
    booked_players  SMALLINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available','partial','full','blocked','maintenance','tournament')),
    current_price   NUMERIC(10,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, slot_date, slot_time, starting_hole)
);

CREATE INDEX idx_slot_course_date ON tee_time_slot(course_id, slot_date, status);

CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    tee_time_slot_id UUID NOT NULL REFERENCES tee_time_slot(id),
    booked_by_id    UUID,
    booking_source  VARCHAR(30) NOT NULL
                    CHECK (booking_source IN ('online','phone','walk_in','marketplace','api','staff')),
    player_count    SMALLINT NOT NULL CHECK (player_count BETWEEN 1 AND 4),
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed'
                    CHECK (status IN ('pending','confirmed','checked_in','completed','cancelled','no_show')),
    total_amount    NUMERIC(10,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    channel_ref     VARCHAR(100),                    -- marketplace confirmation reference
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_booking_slot ON booking(tee_time_slot_id);
CREATE INDEX idx_booking_facility_date ON booking(facility_id, created_at);

-- =====================================================
-- OPERATIONAL: PLAYER & MEMBERSHIP
-- =====================================================

CREATE TABLE player (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(200),
    phone           VARCHAR(30),
    country_code    CHAR(2),
    ghin_number     VARCHAR(20),
    handicap_index  NUMERIC(4,1),
    player_type     VARCHAR(20) NOT NULL DEFAULT 'guest'
                    CHECK (player_type IN ('member','guest','visitor','staff','pro')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_player_tenant ON player(tenant_id);
CREATE INDEX idx_player_email ON player(email);
CREATE INDEX idx_player_ghin ON player(ghin_number);

CREATE TABLE membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id       UUID NOT NULL REFERENCES player(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    plan_type       VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('pending','active','suspended','expired','cancelled')),
    start_date      DATE NOT NULL,
    end_date        DATE,
    monthly_dues    NUMERIC(10,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_player ON membership(player_id);
CREATE INDEX idx_membership_facility ON membership(facility_id, status);

-- =====================================================
-- OPERATIONAL: SCORE POSTING
-- =====================================================

CREATE TABLE score_posting (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id           UUID NOT NULL REFERENCES player(id),
    course_id           UUID NOT NULL REFERENCES course(id),
    course_tee_id       UUID NOT NULL REFERENCES course_tee(id),
    played_at           DATE NOT NULL,
    holes_played        SMALLINT NOT NULL CHECK (holes_played IN (9, 18)),
    adjusted_gross_score SMALLINT NOT NULL,
    score_differential  NUMERIC(5,1),
    score_type          VARCHAR(20) NOT NULL,
    source              VARCHAR(20) NOT NULL DEFAULT 'manual',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_score_player_date ON score_posting(player_id, played_at DESC);

-- =====================================================
-- OPERATIONAL: POS (minimal — detail in facts)
-- =====================================================

CREATE TABLE pos_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    player_id       UUID REFERENCES player(id),
    transaction_type VARCHAR(20) NOT NULL
                    CHECK (transaction_type IN ('sale','return','void','exchange','member_charge')),
    department      VARCHAR(30) NOT NULL
                    CHECK (department IN ('pro_shop','food','beverage','merchandise','service','rental','lesson','green_fee','cart')),
    total_amount    NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  VARCHAR(20),
    gateway_ref     VARCHAR(200),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pos_facility_date ON pos_transaction(facility_id, created_at);
CREATE INDEX idx_pos_player ON pos_transaction(player_id);
```

---

## Analytics Layer — Time-Partitioned Fact Tables

Every fact table is partitioned by month using PostgreSQL native range partitioning. Partitions are created automatically by a scheduled job. Older partitions can be compressed or moved to cold storage.

### Fact: Booking Events

Captures every booking lifecycle event with the context needed for demand analysis.

```sql
CREATE TABLE fact_booking (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_time          TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    course_id           UUID NOT NULL,

    -- Booking dimensions
    booking_id          UUID NOT NULL,
    event_type          VARCHAR(30) NOT NULL
                        CHECK (event_type IN ('created','confirmed','checked_in','completed',
                               'cancelled','no_show','rescheduled')),
    booking_source      VARCHAR(30),
    marketplace_channel VARCHAR(50),

    -- Time dimensions (denormalised for fast aggregation)
    slot_date           DATE NOT NULL,
    slot_time           TIME NOT NULL,
    day_of_week         SMALLINT NOT NULL,              -- 0=Sunday, 6=Saturday
    is_weekend          BOOLEAN NOT NULL,
    is_holiday          BOOLEAN NOT NULL DEFAULT false,

    -- Player dimensions
    player_count        SMALLINT,
    member_count        SMALLINT DEFAULT 0,             -- how many in group are members
    guest_count         SMALLINT DEFAULT 0,

    -- Revenue dimensions
    green_fee_total     NUMERIC(10,2),
    cart_fee_total      NUMERIC(10,2),
    add_on_total        NUMERIC(10,2),
    total_revenue       NUMERIC(10,2),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    price_at_booking    NUMERIC(10,2),                  -- price when booking was made
    price_at_slot_time  NUMERIC(10,2),                  -- price at the time of the slot

    -- Advance booking metric
    days_booked_ahead   SMALLINT,                       -- slot_date - booking_date

    -- Weather at time of play (populated post-round)
    weather_temp_f      SMALLINT,
    weather_wind_mph    SMALLINT,
    weather_condition   VARCHAR(30),
    weather_rain_prob   SMALLINT,

    -- Cancellation context
    cancellation_reason VARCHAR(100),
    hours_before_cancel NUMERIC(6,1)                    -- hours between cancel and slot time
) PARTITION BY RANGE (event_time);

-- Monthly partitions
CREATE TABLE fact_booking_2026_01 PARTITION OF fact_booking FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_booking_2026_02 PARTITION OF fact_booking FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_booking_2026_03 PARTITION OF fact_booking FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_booking_2026_04 PARTITION OF fact_booking FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_booking_2026_05 PARTITION OF fact_booking FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_booking_2026_06 PARTITION OF fact_booking FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_fb_tenant_date ON fact_booking(tenant_id, event_time);
CREATE INDEX idx_fb_facility_slot ON fact_booking(facility_id, slot_date);
CREATE INDEX idx_fb_course_dow ON fact_booking(course_id, day_of_week);
CREATE INDEX idx_fb_source ON fact_booking(booking_source, event_time);
```

### Fact: Pricing Snapshots

Captures the computed price of every tee time slot at regular intervals (e.g., every 15 minutes) plus every time the price changes. This is the training data for the dynamic pricing AI model.

```sql
CREATE TABLE fact_pricing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_time       TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    course_id           UUID NOT NULL,

    -- Slot context
    slot_date           DATE NOT NULL,
    slot_time           TIME NOT NULL,
    day_of_week         SMALLINT NOT NULL,
    is_weekend          BOOLEAN NOT NULL,

    -- Pricing data
    base_price          NUMERIC(10,2) NOT NULL,
    computed_price      NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    price_change_pct    NUMERIC(6,2),                   -- % change from base
    rules_applied       TEXT[],                         -- array of rule names

    -- Context at time of snapshot
    occupancy_pct       NUMERIC(5,1),                   -- % of slots booked for this day
    slots_remaining     SMALLINT,
    hours_until_slot    NUMERIC(6,1),                   -- hours from snapshot to slot_time

    -- Weather forecast at snapshot time
    forecast_temp_f     SMALLINT,
    forecast_wind_mph   SMALLINT,
    forecast_condition  VARCHAR(30),
    forecast_rain_prob  SMALLINT,

    -- Competitor context (if available)
    competitor_avg_price NUMERIC(10,2),                 -- average price at nearby courses

    -- Outcome (populated after the slot passes)
    was_booked          BOOLEAN,                        -- did this slot eventually get booked?
    booked_at_price     NUMERIC(10,2),                  -- price when actually booked
    booking_source      VARCHAR(30)
) PARTITION BY RANGE (snapshot_time);

CREATE TABLE fact_pricing_2026_01 PARTITION OF fact_pricing FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_pricing_2026_02 PARTITION OF fact_pricing FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_pricing_2026_03 PARTITION OF fact_pricing FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_pricing_2026_04 PARTITION OF fact_pricing FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_pricing_2026_05 PARTITION OF fact_pricing FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_pricing_2026_06 PARTITION OF fact_pricing FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_fp_facility_date ON fact_pricing(facility_id, slot_date);
CREATE INDEX idx_fp_course_snapshot ON fact_pricing(course_id, snapshot_time);
```

### Fact: Daily Revenue Aggregation

Pre-aggregated daily revenue by facility, department, and revenue source. This is the primary dashboard and reporting table.

```sql
CREATE TABLE fact_revenue_daily (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    revenue_date        DATE NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',

    -- Revenue by department
    green_fee_revenue   NUMERIC(12,2) NOT NULL DEFAULT 0,
    cart_fee_revenue     NUMERIC(12,2) NOT NULL DEFAULT 0,
    pro_shop_revenue    NUMERIC(12,2) NOT NULL DEFAULT 0,
    food_revenue        NUMERIC(12,2) NOT NULL DEFAULT 0,
    beverage_revenue    NUMERIC(12,2) NOT NULL DEFAULT 0,
    lesson_revenue      NUMERIC(12,2) NOT NULL DEFAULT 0,
    tournament_revenue  NUMERIC(12,2) NOT NULL DEFAULT 0,
    membership_revenue  NUMERIC(12,2) NOT NULL DEFAULT 0,
    other_revenue       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_revenue       NUMERIC(12,2) NOT NULL DEFAULT 0,

    -- Refunds and adjustments
    total_refunds       NUMERIC(12,2) NOT NULL DEFAULT 0,
    net_revenue         NUMERIC(12,2) NOT NULL DEFAULT 0,

    -- Utilisation metrics
    total_slots         INTEGER NOT NULL DEFAULT 0,
    booked_slots        INTEGER NOT NULL DEFAULT 0,
    occupancy_pct       NUMERIC(5,1),
    rounds_played       INTEGER NOT NULL DEFAULT 0,
    unique_players      INTEGER NOT NULL DEFAULT 0,
    member_rounds       INTEGER NOT NULL DEFAULT 0,
    guest_rounds        INTEGER NOT NULL DEFAULT 0,

    -- Yield metrics
    avg_green_fee       NUMERIC(10,2),
    rev_per_available_slot NUMERIC(10,2),               -- RevPAS (like RevPAR in hotels)
    rev_per_round       NUMERIC(10,2),

    -- Channel breakdown
    online_bookings     INTEGER DEFAULT 0,
    phone_bookings      INTEGER DEFAULT 0,
    walkin_bookings     INTEGER DEFAULT 0,
    marketplace_bookings INTEGER DEFAULT 0,
    marketplace_commission NUMERIC(10,2) DEFAULT 0,

    -- Weather summary for the day
    weather_high_f      SMALLINT,
    weather_low_f       SMALLINT,
    weather_condition   VARCHAR(30),                    -- dominant condition
    weather_rain_inches NUMERIC(4,2),
    weather_wind_avg    SMALLINT,

    -- Comparisons (populated by batch job)
    revenue_vs_last_year NUMERIC(6,2),                  -- % change
    occupancy_vs_last_year NUMERIC(6,2),
    revenue_vs_last_week NUMERIC(6,2),

    computed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, facility_id, revenue_date)
) PARTITION BY RANGE (revenue_date);

CREATE TABLE fact_revenue_daily_2026_01 PARTITION OF fact_revenue_daily FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_revenue_daily_2026_02 PARTITION OF fact_revenue_daily FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_revenue_daily_2026_03 PARTITION OF fact_revenue_daily FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_revenue_daily_2026_04 PARTITION OF fact_revenue_daily FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_revenue_daily_2026_05 PARTITION OF fact_revenue_daily FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_revenue_daily_2026_06 PARTITION OF fact_revenue_daily FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_frd_tenant_date ON fact_revenue_daily(tenant_id, revenue_date);
CREATE INDEX idx_frd_facility_date ON fact_revenue_daily(facility_id, revenue_date);
```

### Fact: Player Activity (for retention modelling)

Captures every player touchpoint — rounds played, purchases, lessons, tournament entries — in a single time-series table that feeds the member retention AI model.

```sql
CREATE TABLE fact_player_activity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_time       TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    player_id           UUID NOT NULL,

    -- Activity type
    activity_type       VARCHAR(30) NOT NULL
                        CHECK (activity_type IN ('round_played','pro_shop_purchase','food_purchase',
                               'beverage_purchase','lesson_taken','tournament_entered',
                               'membership_renewed','membership_cancelled','booking_made',
                               'booking_cancelled','loyalty_redeemed','referral_made')),

    -- Monetary value
    amount              NUMERIC(10,2),
    currency_code       CHAR(3) DEFAULT 'USD',

    -- Context
    course_id           UUID,
    booking_id          UUID,
    transaction_id      UUID,
    score_differential  NUMERIC(5,1),                   -- if activity is round_played

    -- Player state at time of activity (snapshot for ML features)
    handicap_at_time    NUMERIC(4,1),
    membership_status   VARCHAR(20),
    loyalty_tier        VARCHAR(20),
    days_since_last_visit INTEGER,
    total_visits_ytd    INTEGER,
    total_spend_ytd     NUMERIC(12,2)
) PARTITION BY RANGE (activity_time);

CREATE TABLE fact_player_activity_2026_01 PARTITION OF fact_player_activity FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_player_activity_2026_02 PARTITION OF fact_player_activity FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_player_activity_2026_03 PARTITION OF fact_player_activity FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_player_activity_2026_04 PARTITION OF fact_player_activity FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_player_activity_2026_05 PARTITION OF fact_player_activity FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_player_activity_2026_06 PARTITION OF fact_player_activity FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_fpa_player_time ON fact_player_activity(player_id, activity_time);
CREATE INDEX idx_fpa_facility_time ON fact_player_activity(facility_id, activity_time);
CREATE INDEX idx_fpa_type_time ON fact_player_activity(activity_type, activity_time);
```

### Fact: Weather Observations

Hourly weather snapshots per facility, captured from weather APIs. Correlated with booking and revenue facts for demand modelling.

```sql
CREATE TABLE fact_weather (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    observation_time    TIMESTAMPTZ NOT NULL,
    facility_id         UUID NOT NULL,

    -- Current conditions
    temperature_f       SMALLINT NOT NULL,
    feels_like_f        SMALLINT,
    humidity_pct        SMALLINT,
    wind_speed_mph      SMALLINT,
    wind_gust_mph       SMALLINT,
    wind_direction      VARCHAR(5),                     -- N, NE, E, SE, etc.
    condition_code      VARCHAR(30) NOT NULL,            -- clear, cloudy, rain, thunderstorm, etc.
    precipitation_in    NUMERIC(4,2) DEFAULT 0,
    uv_index            SMALLINT,
    visibility_miles    NUMERIC(4,1),

    -- Forecast for next 6 hours (captured at observation time)
    forecast_6hr        JSONB,
    -- {
    --   "temp_high": 85, "temp_low": 78,
    --   "rain_probability": 40, "condition": "scattered_showers",
    --   "wind_sustained": 15, "wind_gust": 25
    -- }

    -- Data source
    weather_source      VARCHAR(30) DEFAULT 'openweathermap'
) PARTITION BY RANGE (observation_time);

CREATE TABLE fact_weather_2026_01 PARTITION OF fact_weather FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_weather_2026_02 PARTITION OF fact_weather FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_weather_2026_03 PARTITION OF fact_weather FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_weather_2026_04 PARTITION OF fact_weather FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_weather_2026_05 PARTITION OF fact_weather FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_weather_2026_06 PARTITION OF fact_weather FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_fw_facility_time ON fact_weather(facility_id, observation_time);
```

### Fact: Handicap Revisions

Captures every handicap index change for trend analysis, exceptional score detection, and player development tracking.

```sql
CREATE TABLE fact_handicap_revision (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    revision_time       TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    player_id           UUID NOT NULL,

    -- Handicap data
    previous_index      NUMERIC(4,1),
    new_index           NUMERIC(4,1) NOT NULL,
    index_change        NUMERIC(4,1),                   -- new - previous
    low_index           NUMERIC(4,1),
    scores_in_record    SMALLINT,                       -- total scores in WHS record
    differentials_used  SMALLINT,                       -- best N of 20

    -- Source
    calculation_source  VARCHAR(20) NOT NULL
                        CHECK (calculation_source IN ('ghin','whs','igolf','system','manual')),

    -- Trend indicators (computed)
    trend_3_month       NUMERIC(4,1),                   -- index change over 3 months
    trend_12_month      NUMERIC(4,1),                   -- index change over 12 months
    is_improving        BOOLEAN
) PARTITION BY RANGE (revision_time);

CREATE TABLE fact_handicap_2026_01 PARTITION OF fact_handicap_revision FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_handicap_2026_02 PARTITION OF fact_handicap_revision FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_handicap_2026_03 PARTITION OF fact_handicap_revision FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_handicap_2026_04 PARTITION OF fact_handicap_revision FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_handicap_2026_05 PARTITION OF fact_handicap_revision FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_handicap_2026_06 PARTITION OF fact_handicap_revision FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_fhr_player_time ON fact_handicap_revision(player_id, revision_time);
```

### Fact: Integration Events

Tracks every interaction with external systems (GHIN, iGolf, GolfNow, Google Reserve, payment gateways) for reliability monitoring and SLA tracking.

```sql
CREATE TABLE fact_integration_event (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_time          TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID,

    -- Integration target
    integration_name    VARCHAR(50) NOT NULL,            -- 'ghin', 'igolf', 'golfnow', 'stripe', etc.
    operation           VARCHAR(50) NOT NULL,            -- 'score_post', 'booking_sync', 'payment_charge'
    direction           VARCHAR(10) NOT NULL CHECK (direction IN ('inbound','outbound')),

    -- Result
    status              VARCHAR(20) NOT NULL CHECK (status IN ('success','failure','timeout','rate_limited')),
    response_time_ms    INTEGER,
    error_code          VARCHAR(50),
    error_message       TEXT,

    -- Context
    entity_type         VARCHAR(50),                    -- 'score', 'booking', 'payment'
    entity_id           UUID,
    retry_count         SMALLINT DEFAULT 0
) PARTITION BY RANGE (event_time);

CREATE TABLE fact_integration_2026_01 PARTITION OF fact_integration_event FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE fact_integration_2026_02 PARTITION OF fact_integration_event FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE fact_integration_2026_03 PARTITION OF fact_integration_event FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE fact_integration_2026_04 PARTITION OF fact_integration_event FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE fact_integration_2026_05 PARTITION OF fact_integration_event FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE fact_integration_2026_06 PARTITION OF fact_integration_event FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_fie_integration ON fact_integration_event(integration_name, event_time);
CREATE INDEX idx_fie_status ON fact_integration_event(status, event_time);
```

---

## AI/ML Materialised Views

Pre-computed views that serve as direct feature inputs for machine learning models.

```sql
-- =====================================================
-- ML VIEW: Dynamic Pricing Feature Vector
-- Refreshed every 15 minutes by cron
-- =====================================================

CREATE MATERIALIZED VIEW mv_pricing_features AS
SELECT
    fp.facility_id,
    fp.course_id,
    fp.slot_date,
    fp.slot_time,
    fp.day_of_week,
    fp.is_weekend,

    -- Current state
    fp.occupancy_pct,
    fp.slots_remaining,
    fp.hours_until_slot,
    fp.base_price,
    fp.computed_price,

    -- Weather
    fp.forecast_temp_f,
    fp.forecast_wind_mph,
    fp.forecast_rain_prob,

    -- Historical same-day-of-week averages (last 8 weeks)
    hist.avg_occupancy_same_dow,
    hist.avg_revenue_same_dow,
    hist.avg_price_same_dow,
    hist.booking_rate_same_dow,

    -- Competitor pricing
    fp.competitor_avg_price,

    -- Outcome label for training
    fp.was_booked,
    fp.booked_at_price

FROM fact_pricing fp
LEFT JOIN LATERAL (
    SELECT
        AVG(frd.occupancy_pct) AS avg_occupancy_same_dow,
        AVG(frd.net_revenue) AS avg_revenue_same_dow,
        AVG(frd.avg_green_fee) AS avg_price_same_dow,
        AVG(frd.booked_slots::numeric / NULLIF(frd.total_slots, 0)) AS booking_rate_same_dow
    FROM fact_revenue_daily frd
    WHERE frd.facility_id = fp.facility_id
      AND EXTRACT(DOW FROM frd.revenue_date) = fp.day_of_week
      AND frd.revenue_date BETWEEN fp.slot_date - INTERVAL '56 days' AND fp.slot_date - INTERVAL '1 day'
) hist ON true
WHERE fp.snapshot_time >= now() - INTERVAL '24 hours';

CREATE UNIQUE INDEX idx_mv_pricing ON mv_pricing_features(facility_id, course_id, slot_date, slot_time);

-- =====================================================
-- ML VIEW: Member Retention Risk Score Inputs
-- Refreshed daily
-- =====================================================

CREATE MATERIALIZED VIEW mv_retention_features AS
SELECT
    p.id AS player_id,
    p.tenant_id,
    m.facility_id,
    m.plan_type,
    m.status AS membership_status,
    m.start_date AS membership_start,
    m.monthly_dues,

    -- Engagement metrics (last 90 days)
    COUNT(CASE WHEN fpa.activity_type = 'round_played' AND fpa.activity_time >= now() - INTERVAL '90 days' THEN 1 END) AS rounds_90d,
    COUNT(CASE WHEN fpa.activity_type = 'round_played' AND fpa.activity_time >= now() - INTERVAL '30 days' THEN 1 END) AS rounds_30d,
    SUM(CASE WHEN fpa.activity_time >= now() - INTERVAL '90 days' THEN fpa.amount ELSE 0 END) AS spend_90d,
    SUM(CASE WHEN fpa.activity_time >= now() - INTERVAL '30 days' THEN fpa.amount ELSE 0 END) AS spend_30d,

    -- Frequency trend
    COUNT(CASE WHEN fpa.activity_type = 'round_played' AND fpa.activity_time >= now() - INTERVAL '30 days' THEN 1 END)::numeric /
    NULLIF(COUNT(CASE WHEN fpa.activity_type = 'round_played' AND fpa.activity_time BETWEEN now() - INTERVAL '90 days' AND now() - INTERVAL '60 days' THEN 1 END), 0) AS frequency_trend_ratio,

    -- Days since last activity
    EXTRACT(DAY FROM now() - MAX(fpa.activity_time))::INTEGER AS days_since_last_activity,

    -- Activity diversity (how many different activity types in last 90 days)
    COUNT(DISTINCT CASE WHEN fpa.activity_time >= now() - INTERVAL '90 days' THEN fpa.activity_type END) AS activity_type_diversity,

    -- Tournament participation
    COUNT(CASE WHEN fpa.activity_type = 'tournament_entered' AND fpa.activity_time >= now() - INTERVAL '365 days' THEN 1 END) AS tournaments_12m,

    -- Handicap trend
    hr.trend_3_month AS handicap_trend_3m,
    hr.new_index AS current_handicap,

    -- Membership tenure
    EXTRACT(DAY FROM now() - m.start_date)::INTEGER AS membership_tenure_days

FROM player p
JOIN membership m ON m.player_id = p.id AND m.status = 'active'
LEFT JOIN fact_player_activity fpa ON fpa.player_id = p.id
LEFT JOIN LATERAL (
    SELECT new_index, trend_3_month
    FROM fact_handicap_revision
    WHERE player_id = p.id
    ORDER BY revision_time DESC
    LIMIT 1
) hr ON true
GROUP BY p.id, p.tenant_id, m.facility_id, m.plan_type, m.status, m.start_date, m.monthly_dues,
         hr.trend_3_month, hr.new_index;

CREATE UNIQUE INDEX idx_mv_retention ON mv_retention_features(player_id, facility_id);
```

---

## Example Analytics Queries

### Revenue vs. Weather Correlation

```sql
-- How does rain affect revenue? Compare rainy vs. dry days this season
SELECT
    CASE WHEN weather_rain_inches > 0.1 THEN 'Rainy' ELSE 'Dry' END AS day_type,
    COUNT(*) AS days,
    AVG(net_revenue) AS avg_daily_revenue,
    AVG(occupancy_pct) AS avg_occupancy,
    AVG(avg_green_fee) AS avg_price,
    AVG(rounds_played) AS avg_rounds
FROM fact_revenue_daily
WHERE facility_id = :facility_id
  AND revenue_date BETWEEN '2026-04-01' AND '2026-09-30'
GROUP BY CASE WHEN weather_rain_inches > 0.1 THEN 'Rainy' ELSE 'Dry' END;
```

### Dynamic Pricing Effectiveness

```sql
-- Did price reductions actually increase bookings?
SELECT
    CASE
        WHEN price_change_pct < -20 THEN 'Deep discount (>20%)'
        WHEN price_change_pct < -10 THEN 'Moderate discount (10-20%)'
        WHEN price_change_pct < 0 THEN 'Light discount (<10%)'
        WHEN price_change_pct = 0 THEN 'Base price'
        ELSE 'Surcharge'
    END AS price_bucket,
    COUNT(*) AS snapshots,
    AVG(CASE WHEN was_booked THEN 1.0 ELSE 0.0 END) * 100 AS booking_rate_pct,
    AVG(booked_at_price) AS avg_booked_price,
    AVG(occupancy_pct) AS avg_occupancy_at_snapshot
FROM fact_pricing
WHERE facility_id = :facility_id
  AND slot_date BETWEEN '2026-06-01' AND '2026-06-30'
  AND hours_until_slot BETWEEN 0 AND 24        -- only look at last 24h before slot
GROUP BY 1
ORDER BY 1;
```

### Member Retention Early Warning

```sql
-- Members showing declining engagement
SELECT
    player_id,
    membership_status,
    plan_type,
    rounds_30d,
    rounds_90d,
    days_since_last_activity,
    spend_30d,
    frequency_trend_ratio,
    current_handicap,
    membership_tenure_days
FROM mv_retention_features
WHERE facility_id = :facility_id
  AND membership_status = 'active'
  AND (
      days_since_last_activity > 30                     -- no visit in 30 days
      OR frequency_trend_ratio < 0.5                    -- frequency dropped by half
      OR (rounds_90d < 3 AND membership_tenure_days > 180)  -- low play for established member
  )
ORDER BY days_since_last_activity DESC;
```

### Booking Source Channel Performance

```sql
-- Which booking channels drive the most revenue per round?
SELECT
    booking_source,
    marketplace_channel,
    COUNT(*) AS bookings,
    SUM(total_revenue) AS total_revenue,
    AVG(total_revenue / NULLIF(player_count, 0)) AS revenue_per_player,
    AVG(days_booked_ahead) AS avg_advance_days,
    AVG(CASE WHEN event_type = 'cancelled' THEN 1.0 ELSE 0.0 END) * 100 AS cancel_rate_pct
FROM fact_booking
WHERE facility_id = :facility_id
  AND event_type IN ('completed','cancelled')
  AND event_time >= '2026-01-01'
GROUP BY booking_source, marketplace_channel
ORDER BY total_revenue DESC;
```

### Year-over-Year Revenue Trend

```sql
-- Monthly revenue comparison: this year vs. last year
SELECT
    EXTRACT(MONTH FROM revenue_date) AS month,
    SUM(CASE WHEN EXTRACT(YEAR FROM revenue_date) = 2026 THEN net_revenue END) AS revenue_2026,
    SUM(CASE WHEN EXTRACT(YEAR FROM revenue_date) = 2025 THEN net_revenue END) AS revenue_2025,
    ROUND(
        (SUM(CASE WHEN EXTRACT(YEAR FROM revenue_date) = 2026 THEN net_revenue END) -
         SUM(CASE WHEN EXTRACT(YEAR FROM revenue_date) = 2025 THEN net_revenue END)) /
        NULLIF(SUM(CASE WHEN EXTRACT(YEAR FROM revenue_date) = 2025 THEN net_revenue END), 0) * 100,
        1
    ) AS yoy_change_pct
FROM fact_revenue_daily
WHERE facility_id = :facility_id
  AND revenue_date >= '2025-01-01'
GROUP BY EXTRACT(MONTH FROM revenue_date)
ORDER BY 1;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| **Operational Layer** | | |
| Facility & Course | 3 | facility, course, course_tee |
| Tee Sheet & Booking | 2 | tee_time_slot, booking |
| Player & Membership | 2 | player, membership |
| Scoring | 1 | score_posting |
| POS | 1 | pos_transaction |
| Operational subtotal | **9** | Compact OLTP core |
| **Analytics Layer** | | |
| Fact: Booking Events | 1 | Partitioned by month |
| Fact: Pricing Snapshots | 1 | Partitioned by month; 15-min interval snapshots |
| Fact: Revenue Daily | 1 | Partitioned by month; one row per facility per day |
| Fact: Player Activity | 1 | Partitioned by month; all player touchpoints |
| Fact: Weather | 1 | Partitioned by month; hourly observations |
| Fact: Handicap Revisions | 1 | Partitioned by month |
| Fact: Integration Events | 1 | Partitioned by month; external API health |
| Analytics subtotal | **7** | Each with monthly partitions |
| **Materialised Views** | 2 | mv_pricing_features, mv_retention_features |
| **Total** | **18** | 9 operational + 7 fact tables + 2 materialised views |

---

## Key Design Decisions

1. **Two-layer architecture: OLTP + OLAP** — operational tables serve application CRUD with low latency; fact tables serve analytics and AI/ML with high throughput on range queries. Neither layer compromises for the other's requirements.

2. **Denormalised fact tables** — fact tables intentionally duplicate dimensional data (day_of_week, is_weekend, weather) to eliminate JOINs during analytical queries. A booking fact row contains everything needed to answer "what happened" without touching the operational layer.

3. **Monthly range partitioning on all fact tables** — PostgreSQL native partitioning enables partition pruning (queries on a date range only scan relevant partitions), automatic archiving (detach old partitions), and per-partition compression. Compatible with TimescaleDB hypertables for automatic partition management.

4. **Pricing snapshots as ML training data** — `fact_pricing` captures the price of every slot at regular intervals along with the context (weather, occupancy, competitor prices) and the outcome (was it booked?). This is a labelled training dataset for a dynamic pricing model — the `was_booked` and `booked_at_price` columns are the target variables.

5. **Player activity as a unified engagement stream** — rather than querying across bookings, POS, tournaments, and memberships to assess player engagement, `fact_player_activity` provides a single time-ordered stream of all interactions. The `days_since_last_visit` and `total_spend_ytd` snapshots are pre-computed ML features.

6. **Weather as a first-class dimension** — weather is the single largest external factor affecting golf course revenue. Hourly weather observations in `fact_weather` and weather context on booking/revenue/pricing facts enable the demand forecasting model to learn weather-revenue correlations automatically.

7. **RevPAS metric (Revenue Per Available Slot)** — analogous to the hotel industry's RevPAR, this metric in `fact_revenue_daily` provides a single number that captures both pricing power and utilisation. It is the key performance indicator for yield management.

8. **Materialised views as ML feature stores** — `mv_pricing_features` and `mv_retention_features` serve as direct inputs to prediction models. They are refreshed on a schedule (15 minutes for pricing, daily for retention) and indexed for fast lookup during inference.

9. **Integration event monitoring** — `fact_integration_event` tracks every API call to GHIN, iGolf, GolfNow, and payment gateways. This enables SLA monitoring ("GHIN API has been failing for 2 hours"), retry analytics, and integration health dashboards — often overlooked in golf software that silently drops sync failures.

10. **Compact operational layer** — the OLTP layer has only 9 tables, significantly fewer than the normalised model (44 tables). Detailed breakdowns (hole-by-hole scores, line items, tournament pairings) that are rarely queried in real-time are captured in the fact tables rather than normalised operational tables, keeping the application layer simple and fast.
