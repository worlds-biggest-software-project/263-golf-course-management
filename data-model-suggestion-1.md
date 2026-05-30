# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Golf Course Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form relational design. Every domain concept — courses, holes, tee sheets, bookings, members, POS transactions, tournaments, and handicaps — gets its own table with explicit foreign keys enforcing referential integrity. Junction tables handle all many-to-many relationships (player-to-tournament, booking-to-add-on, member-to-membership-plan). Reference data (jurisdictions, currencies, tee colours, scoring formats) lives in dedicated lookup tables aligned with industry standards.

The approach mirrors how mature golf management platforms like foreUP and Lightspeed Golf (Chronogolf) structure their API resources: courses, customers, bookings, payments, and player types as distinct, well-defined entities. It also aligns with the GHIN API's resource model (golfers, scores, courses, clubs) and the WHS Interoperability Standard's data exchange requirements.

This model is best suited for a platform that prioritises data integrity, complex cross-entity reporting (revenue by course by time period, member lifetime value, tournament history), and straightforward integration with external APIs (GHIN, iGolf, GolfNow) that expect clean, normalised resource representations.

**Best for:** Teams building a full-featured golf management platform where data integrity, regulatory compliance (PCI DSS, GDPR), and complex reporting are paramount.

**Trade-offs:**
- Pro: Strong referential integrity — cascading deletes and updates are predictable
- Pro: Standard SQL queries work naturally; no JSONB extraction or event replay needed
- Pro: Maps cleanly to REST/OpenAPI resource definitions for partner APIs
- Pro: Straightforward for BI tools and reporting dashboards
- Con: High table count (~55-65 tables) increases migration complexity
- Con: Schema changes for jurisdiction-specific fields require ALTER TABLE and migrations
- Con: Many JOIN operations for cross-cutting queries (e.g. "show me all activity for member X")
- Con: Audit trail requires a separate audit table or trigger-based logging

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WHS Interoperability Standard v1.0 | `handicap_index` table stores Handicap Index per the WHS calculation model; `score_posting` aligns with WHS score submission fields |
| GHIN API (USGA) | `ghin_number` stored on player profiles; score posting fields (`adjusted_gross_score`, `score_type`, `holes_played`, `played_at`) match GHIN API schema |
| USGA Course Rating (NCRDB) | `course_tee` table stores `course_rating`, `slope_rating`, `bogey_rating` per USGA formulas |
| ISO 3166-1/2 | `jurisdiction` lookup table uses ISO country and subdivision codes for multi-region support |
| ISO 4217 | `currency_code` on all monetary fields uses ISO currency codes |
| PCI DSS | No raw card data stored; `payment` table references tokenised payment method IDs from payment gateway |
| OAuth 2.0 (RFC 6749) | `api_partner` and `api_token` tables support partner API authentication flows |
| OpenAPI 3.1 | Table structure maps 1:1 to OpenAPI resource definitions |
| EMV / NFC | `pos_terminal` table tracks EMV device identifiers and compliance status |

---

## Core Entities — Facility & Course

```sql
-- =====================================================
-- FACILITY & COURSE MANAGEMENT
-- =====================================================

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,               -- multi-tenant isolation
    name            VARCHAR(200) NOT NULL,
    facility_type   VARCHAR(30) NOT NULL CHECK (facility_type IN ('public','private','semi_private','resort','municipal')),
    address_line1   VARCHAR(200),
    address_line2   VARCHAR(200),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,            -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL,         -- IANA timezone
    phone           VARCHAR(30),
    email           VARCHAR(200),
    website_url     VARCHAR(500),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    ghin_club_id    VARCHAR(20),                  -- USGA GHIN club identifier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_facility_tenant ON facility(tenant_id);
CREATE INDEX idx_facility_country ON facility(country_code);

CREATE TABLE course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    name            VARCHAR(200) NOT NULL,        -- e.g. "Championship Course", "Executive 9"
    holes_count     SMALLINT NOT NULL CHECK (holes_count IN (9, 18, 27, 36)),
    par             SMALLINT NOT NULL,
    year_built      SMALLINT,
    architect       VARCHAR(200),
    course_style    VARCHAR(30) CHECK (course_style IN ('links','parkland','desert','mountain','tropical','executive')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    ghin_course_id  VARCHAR(20),                  -- GHIN course identifier
    igolf_course_id VARCHAR(20),                  -- iGolf Connect course identifier
    golfapi_club_id VARCHAR(20),                  -- GolfAPI.io club ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_facility ON course(facility_id);

CREATE TABLE hole (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    hole_number     SMALLINT NOT NULL CHECK (hole_number BETWEEN 1 AND 36),
    par             SMALLINT NOT NULL CHECK (par BETWEEN 3 AND 6),
    stroke_index    SMALLINT NOT NULL CHECK (stroke_index BETWEEN 1 AND 18),
    latitude        NUMERIC(10,7),                -- green centre GPS
    longitude       NUMERIC(10,7),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, hole_number)
);

CREATE TABLE course_tee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    tee_name        VARCHAR(50) NOT NULL,         -- e.g. "Championship", "Member", "Forward"
    tee_colour      VARCHAR(20),                  -- e.g. "Black", "Blue", "White", "Red"
    gender          CHAR(1) CHECK (gender IN ('M','F','X')),
    course_rating   NUMERIC(4,1) NOT NULL,        -- USGA Course Rating
    slope_rating    SMALLINT NOT NULL CHECK (slope_rating BETWEEN 55 AND 155),  -- USGA Slope Rating
    bogey_rating    NUMERIC(4,1),
    total_yardage   SMALLINT,
    total_metres    SMALLINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, tee_name, gender)
);

CREATE TABLE hole_tee_distance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hole_id         UUID NOT NULL REFERENCES hole(id),
    course_tee_id   UUID NOT NULL REFERENCES course_tee(id),
    yardage         SMALLINT,
    metres          SMALLINT,
    UNIQUE (hole_id, course_tee_id)
);
```

---

## Tee Sheet & Booking

```sql
-- =====================================================
-- TEE SHEET & BOOKING
-- =====================================================

CREATE TABLE tee_sheet_config (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES course(id),
    effective_date      DATE NOT NULL,
    end_date            DATE,                          -- NULL = open-ended
    first_tee_time      TIME NOT NULL,                 -- e.g. 06:00
    last_tee_time       TIME NOT NULL,                 -- e.g. 18:00
    interval_minutes    SMALLINT NOT NULL DEFAULT 8,   -- typical 7-10 minutes
    max_players_per_slot SMALLINT NOT NULL DEFAULT 4,
    shotgun_start       BOOLEAN NOT NULL DEFAULT false,
    crossover_enabled   BOOLEAN NOT NULL DEFAULT false, -- 1st and 10th tee starts
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tee_sheet_config_course ON tee_sheet_config(course_id, effective_date);

CREATE TABLE tee_time_slot (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES course(id),
    slot_date           DATE NOT NULL,
    slot_time           TIME NOT NULL,
    starting_hole       SMALLINT NOT NULL DEFAULT 1,
    max_players         SMALLINT NOT NULL DEFAULT 4,
    booked_players      SMALLINT NOT NULL DEFAULT 0,
    status              VARCHAR(20) NOT NULL DEFAULT 'available'
                        CHECK (status IN ('available','partial','full','blocked','maintenance','tournament')),
    block_reason        VARCHAR(200),
    rate_code_id        UUID,                          -- links to pricing
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, slot_date, slot_time, starting_hole)
);

CREATE INDEX idx_tee_time_slot_date ON tee_time_slot(course_id, slot_date, status);

CREATE TABLE booking (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    tee_time_slot_id    UUID NOT NULL REFERENCES tee_time_slot(id),
    booked_by_id        UUID NOT NULL,                 -- references player or staff
    booking_source      VARCHAR(30) NOT NULL
                        CHECK (booking_source IN ('online','phone','walk_in','marketplace','api','staff')),
    marketplace_ref     VARCHAR(100),                  -- GolfNow confirmation, etc.
    player_count        SMALLINT NOT NULL CHECK (player_count BETWEEN 1 AND 4),
    status              VARCHAR(20) NOT NULL DEFAULT 'confirmed'
                        CHECK (status IN ('pending','confirmed','checked_in','completed','cancelled','no_show')),
    total_amount        NUMERIC(10,2),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    notes               TEXT,
    cancelled_at        TIMESTAMPTZ,
    cancellation_reason VARCHAR(200),
    checked_in_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_booking_slot ON booking(tee_time_slot_id);
CREATE INDEX idx_booking_booked_by ON booking(booked_by_id);
CREATE INDEX idx_booking_facility_date ON booking(facility_id, created_at);

CREATE TABLE booking_player (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id          UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    player_id           UUID REFERENCES player(id),    -- NULL for anonymous guests
    player_name         VARCHAR(200),                  -- for guest players without profiles
    player_position     SMALLINT NOT NULL,              -- 1-4 position in group
    green_fee_amount    NUMERIC(10,2),
    cart_fee_amount     NUMERIC(10,2),
    is_walking          BOOLEAN NOT NULL DEFAULT false,
    checked_in          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_booking_player_booking ON booking_player(booking_id);
CREATE INDEX idx_booking_player_player ON booking_player(player_id);

CREATE TABLE booking_add_on (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id          UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    add_on_type         VARCHAR(30) NOT NULL
                        CHECK (add_on_type IN ('cart','caddie','range_balls','lesson','rental_clubs','locker','gps')),
    quantity            SMALLINT NOT NULL DEFAULT 1,
    unit_price          NUMERIC(10,2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Player & Membership

```sql
-- =====================================================
-- PLAYER & MEMBERSHIP
-- =====================================================

CREATE TABLE player (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(200),
    phone               VARCHAR(30),
    date_of_birth       DATE,
    gender              CHAR(1) CHECK (gender IN ('M','F','X')),
    address_line1       VARCHAR(200),
    address_line2       VARCHAR(200),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2),                       -- ISO 3166-1
    ghin_number         VARCHAR(20),                   -- USGA GHIN ID
    whs_id              VARCHAR(30),                   -- WHS global player ID
    handicap_index      NUMERIC(4,1),                  -- current WHS Handicap Index
    handicap_updated_at TIMESTAMPTZ,
    player_type         VARCHAR(20) NOT NULL DEFAULT 'guest'
                        CHECK (player_type IN ('member','guest','visitor','staff','pro')),
    marketing_opt_in    BOOLEAN NOT NULL DEFAULT false,
    gdpr_consent_at     TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_player_tenant ON player(tenant_id);
CREATE INDEX idx_player_email ON player(email);
CREATE INDEX idx_player_ghin ON player(ghin_number);
CREATE INDEX idx_player_name ON player(last_name, first_name);

CREATE TABLE membership_plan (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    name                VARCHAR(100) NOT NULL,         -- e.g. "Full Golf", "Social", "Junior"
    description         TEXT,
    plan_type           VARCHAR(30) NOT NULL
                        CHECK (plan_type IN ('full_golf','limited_golf','social','junior','senior','corporate','family','trial')),
    billing_frequency   VARCHAR(20) NOT NULL
                        CHECK (billing_frequency IN ('monthly','quarterly','semi_annual','annual','one_time')),
    base_amount         NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    initiation_fee      NUMERIC(10,2) DEFAULT 0,
    includes_cart       BOOLEAN NOT NULL DEFAULT false,
    includes_range      BOOLEAN NOT NULL DEFAULT false,
    guest_passes_per_year SMALLINT DEFAULT 0,
    max_advance_booking_days SMALLINT DEFAULT 7,       -- how far ahead members can book
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE membership (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id           UUID NOT NULL REFERENCES player(id),
    membership_plan_id  UUID NOT NULL REFERENCES membership_plan(id),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    member_number       VARCHAR(30),                   -- club-assigned member number
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('pending','active','suspended','expired','cancelled')),
    start_date          DATE NOT NULL,
    end_date            DATE,                          -- NULL for open-ended
    renewal_date        DATE,
    billing_day         SMALLINT CHECK (billing_day BETWEEN 1 AND 28),
    balance_due         NUMERIC(10,2) NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_player ON membership(player_id);
CREATE INDEX idx_membership_facility ON membership(facility_id, status);

CREATE TABLE membership_billing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    membership_id       UUID NOT NULL REFERENCES membership(id),
    billing_period_start DATE NOT NULL,
    billing_period_end  DATE NOT NULL,
    amount              NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','invoiced','paid','overdue','waived')),
    invoice_number      VARCHAR(50),
    paid_at             TIMESTAMPTZ,
    payment_id          UUID,                          -- references payment table
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_billing_membership ON membership_billing(membership_id);

CREATE TABLE loyalty_program (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    name                VARCHAR(100) NOT NULL,
    points_per_dollar   NUMERIC(6,2) NOT NULL DEFAULT 1.0,
    points_per_round    INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE player_loyalty (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id           UUID NOT NULL REFERENCES player(id),
    loyalty_program_id  UUID NOT NULL REFERENCES loyalty_program(id),
    points_balance      INTEGER NOT NULL DEFAULT 0,
    lifetime_points     INTEGER NOT NULL DEFAULT 0,
    tier                VARCHAR(20) DEFAULT 'standard'
                        CHECK (tier IN ('standard','silver','gold','platinum')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (player_id, loyalty_program_id)
);
```

---

## Handicap & Scoring

```sql
-- =====================================================
-- HANDICAP & SCORING (WHS/GHIN aligned)
-- =====================================================

CREATE TABLE score_posting (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id               UUID NOT NULL REFERENCES player(id),
    course_id               UUID NOT NULL REFERENCES course(id),
    course_tee_id           UUID NOT NULL REFERENCES course_tee(id),
    played_at               DATE NOT NULL,
    holes_played            SMALLINT NOT NULL CHECK (holes_played IN (9, 18)),
    adjusted_gross_score    SMALLINT NOT NULL,
    score_type              VARCHAR(20) NOT NULL
                            CHECK (score_type IN ('home','away','tournament','internet','penalty','combined_9')),
    score_differential      NUMERIC(5,1),               -- (113 / Slope) * (AGS - CR - PCC)
    pcc_adjustment          NUMERIC(3,1) DEFAULT 0,     -- Playing Conditions Calculation
    is_nine_hole            BOOLEAN NOT NULL DEFAULT false,
    exceptional_score       BOOLEAN NOT NULL DEFAULT false,
    ghin_score_id           VARCHAR(30),                -- GHIN API score reference
    posted_to_ghin          BOOLEAN NOT NULL DEFAULT false,
    posted_to_ghin_at       TIMESTAMPTZ,
    source                  VARCHAR(20) NOT NULL DEFAULT 'manual'
                            CHECK (source IN ('manual','ghin_sync','app','tournament_system')),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_score_posting_player ON score_posting(player_id, played_at DESC);
CREATE INDEX idx_score_posting_course ON score_posting(course_id, played_at);

CREATE TABLE hole_score (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    score_posting_id    UUID NOT NULL REFERENCES score_posting(id) ON DELETE CASCADE,
    hole_id             UUID NOT NULL REFERENCES hole(id),
    strokes             SMALLINT NOT NULL,
    adjusted_strokes    SMALLINT NOT NULL,              -- after net double bogey cap
    putts               SMALLINT,
    fairway_hit         BOOLEAN,
    gir                 BOOLEAN,                        -- green in regulation
    penalty_strokes     SMALLINT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hole_score_posting ON hole_score(score_posting_id);

CREATE TABLE handicap_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id           UUID NOT NULL REFERENCES player(id),
    effective_date      DATE NOT NULL,
    handicap_index      NUMERIC(4,1) NOT NULL,
    low_handicap_index  NUMERIC(4,1),                  -- for soft/hard cap calculations
    calculation_source  VARCHAR(20) NOT NULL
                        CHECK (calculation_source IN ('ghin','whs','manual','system')),
    scores_used         SMALLINT,                      -- number of best differentials used
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_handicap_history_player ON handicap_history(player_id, effective_date DESC);
```

---

## Point of Sale & Inventory

```sql
-- =====================================================
-- POINT OF SALE & INVENTORY
-- =====================================================

CREATE TABLE pos_terminal (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    terminal_name       VARCHAR(100) NOT NULL,
    location            VARCHAR(50) NOT NULL
                        CHECK (location IN ('pro_shop','restaurant','bar','halfway_house','snack_bar','beverage_cart','starter')),
    device_serial       VARCHAR(100),
    emv_compliant       BOOLEAN NOT NULL DEFAULT true,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product_category (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    name                VARCHAR(100) NOT NULL,
    parent_category_id  UUID REFERENCES product_category(id),
    department          VARCHAR(30) NOT NULL
                        CHECK (department IN ('pro_shop','food','beverage','merchandise','service','rental','lesson')),
    sort_order          SMALLINT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    category_id         UUID NOT NULL REFERENCES product_category(id),
    sku                 VARCHAR(50),
    name                VARCHAR(200) NOT NULL,
    description         TEXT,
    unit_price          NUMERIC(10,2) NOT NULL,
    cost_price          NUMERIC(10,2),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    tax_rate            NUMERIC(5,3) DEFAULT 0,
    track_inventory     BOOLEAN NOT NULL DEFAULT true,
    quantity_on_hand    INTEGER DEFAULT 0,
    reorder_point       INTEGER,
    is_serialized       BOOLEAN NOT NULL DEFAULT false,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_facility ON product(facility_id, category_id);
CREATE INDEX idx_product_sku ON product(sku);

CREATE TABLE pos_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    terminal_id         UUID REFERENCES pos_terminal(id),
    player_id           UUID REFERENCES player(id),    -- NULL for anonymous walk-in
    membership_id       UUID REFERENCES membership(id),-- for member charge
    transaction_type    VARCHAR(20) NOT NULL
                        CHECK (transaction_type IN ('sale','return','void','exchange','member_charge')),
    subtotal            NUMERIC(10,2) NOT NULL,
    tax_amount          NUMERIC(10,2) NOT NULL DEFAULT 0,
    discount_amount     NUMERIC(10,2) NOT NULL DEFAULT 0,
    total_amount        NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    staff_id            UUID,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pos_transaction_facility ON pos_transaction(facility_id, created_at);
CREATE INDEX idx_pos_transaction_player ON pos_transaction(player_id);

CREATE TABLE pos_transaction_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id      UUID NOT NULL REFERENCES pos_transaction(id) ON DELETE CASCADE,
    product_id          UUID NOT NULL REFERENCES product(id),
    quantity            NUMERIC(10,3) NOT NULL,
    unit_price          NUMERIC(10,2) NOT NULL,
    discount_amount     NUMERIC(10,2) DEFAULT 0,
    tax_amount          NUMERIC(10,2) DEFAULT 0,
    line_total          NUMERIC(10,2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pos_item_transaction ON pos_transaction_item(transaction_id);

CREATE TABLE payment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    pos_transaction_id  UUID REFERENCES pos_transaction(id),
    booking_id          UUID REFERENCES booking(id),
    membership_billing_id UUID REFERENCES membership_billing(id),
    payment_method      VARCHAR(20) NOT NULL
                        CHECK (payment_method IN ('card','cash','member_charge','gift_card','mobile_pay','check','comp')),
    amount              NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    gateway_ref         VARCHAR(200),                  -- Stripe/Braintree payment ID (PCI DSS)
    card_last_four      CHAR(4),
    card_brand          VARCHAR(20),
    status              VARCHAR(20) NOT NULL DEFAULT 'completed'
                        CHECK (status IN ('pending','completed','failed','refunded','partially_refunded')),
    refund_amount       NUMERIC(10,2) DEFAULT 0,
    processed_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_facility ON payment(facility_id, processed_at);
CREATE INDEX idx_payment_pos ON payment(pos_transaction_id);
CREATE INDEX idx_payment_booking ON payment(booking_id);
```

---

## Tournament Management

```sql
-- =====================================================
-- TOURNAMENT MANAGEMENT
-- =====================================================

CREATE TABLE tournament (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    course_id           UUID NOT NULL REFERENCES course(id),
    name                VARCHAR(200) NOT NULL,
    description         TEXT,
    tournament_type     VARCHAR(30) NOT NULL
                        CHECK (tournament_type IN ('stroke_play','match_play','stableford','scramble','best_ball',
                               'alternate_shot','chapman','shamble','charity','corporate')),
    scoring_format      VARCHAR(20) NOT NULL DEFAULT 'gross'
                        CHECK (scoring_format IN ('gross','net','both')),
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    rounds_count        SMALLINT NOT NULL DEFAULT 1,
    max_players         SMALLINT,
    entry_fee           NUMERIC(10,2) DEFAULT 0,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','registration_open','registration_closed','in_progress','completed','cancelled')),
    handicap_allowance  NUMERIC(4,2) DEFAULT 1.0,      -- e.g. 0.80 = 80% of handicap
    golfgenius_event_id VARCHAR(50),                   -- Golf Genius integration
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tournament_facility ON tournament(facility_id, start_date);

CREATE TABLE tournament_entry (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_id       UUID NOT NULL REFERENCES tournament(id),
    player_id           UUID NOT NULL REFERENCES player(id),
    entry_status        VARCHAR(20) NOT NULL DEFAULT 'registered'
                        CHECK (entry_status IN ('registered','confirmed','withdrawn','disqualified','waitlisted')),
    playing_handicap    NUMERIC(4,1),                  -- handicap at time of entry
    team_id             UUID,                          -- for team formats
    entry_fee_paid      BOOLEAN NOT NULL DEFAULT false,
    payment_id          UUID REFERENCES payment(id),
    registered_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tournament_id, player_id)
);

CREATE INDEX idx_tournament_entry_tournament ON tournament_entry(tournament_id);
CREATE INDEX idx_tournament_entry_player ON tournament_entry(player_id);

CREATE TABLE tournament_round (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_id       UUID NOT NULL REFERENCES tournament(id),
    round_number        SMALLINT NOT NULL,
    round_date          DATE NOT NULL,
    course_tee_id       UUID NOT NULL REFERENCES course_tee(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                        CHECK (status IN ('scheduled','in_progress','completed','cancelled')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tournament_id, round_number)
);

CREATE TABLE tournament_pairing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_round_id UUID NOT NULL REFERENCES tournament_round(id),
    tee_time_slot_id    UUID REFERENCES tee_time_slot(id),
    group_number        SMALLINT NOT NULL,
    starting_hole       SMALLINT NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tournament_pairing_player (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_pairing_id UUID NOT NULL REFERENCES tournament_pairing(id),
    tournament_entry_id UUID NOT NULL REFERENCES tournament_entry(id),
    position_in_group   SMALLINT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tournament_score (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_entry_id UUID NOT NULL REFERENCES tournament_entry(id),
    tournament_round_id UUID NOT NULL REFERENCES tournament_round(id),
    gross_total         SMALLINT,
    net_total           SMALLINT,
    stableford_points   SMALLINT,
    score_status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (score_status IN ('pending','partial','submitted','verified','disqualified')),
    score_posting_id    UUID REFERENCES score_posting(id), -- link to handicap score
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tournament_entry_id, tournament_round_id)
);
```

---

## Dynamic Pricing & Revenue

```sql
-- =====================================================
-- DYNAMIC PRICING & REVENUE MANAGEMENT
-- =====================================================

CREATE TABLE rate_code (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    code                VARCHAR(30) NOT NULL,
    name                VARCHAR(100) NOT NULL,         -- e.g. "Peak Weekend", "Twilight", "Senior"
    rate_type           VARCHAR(20) NOT NULL
                        CHECK (rate_type IN ('standard','twilight','super_twilight','senior','junior','military','replay','member_guest')),
    base_green_fee      NUMERIC(10,2) NOT NULL,
    base_cart_fee       NUMERIC(10,2) DEFAULT 0,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    applies_weekday     BOOLEAN NOT NULL DEFAULT true,
    applies_weekend     BOOLEAN NOT NULL DEFAULT true,
    applies_holiday     BOOLEAN NOT NULL DEFAULT true,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_code_facility ON rate_code(facility_id, effective_from);

CREATE TABLE dynamic_price_rule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    rule_name           VARCHAR(100) NOT NULL,
    trigger_type        VARCHAR(30) NOT NULL
                        CHECK (trigger_type IN ('occupancy','weather','advance_days','day_of_week','time_of_day','season','demand_forecast')),
    adjustment_type     VARCHAR(10) NOT NULL
                        CHECK (adjustment_type IN ('percent','fixed')),
    adjustment_value    NUMERIC(10,2) NOT NULL,        -- positive = surcharge, negative = discount
    min_price_floor     NUMERIC(10,2),                 -- never go below this
    max_price_ceiling   NUMERIC(10,2),                 -- never exceed this
    priority            SMALLINT NOT NULL DEFAULT 0,   -- higher = evaluated first
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE price_snapshot (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tee_time_slot_id    UUID NOT NULL REFERENCES tee_time_slot(id),
    computed_price      NUMERIC(10,2) NOT NULL,
    base_price          NUMERIC(10,2) NOT NULL,
    rules_applied       TEXT[],                        -- array of rule IDs that contributed
    weather_factor      NUMERIC(4,2),
    occupancy_factor    NUMERIC(4,2),
    computed_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_snapshot_slot ON price_snapshot(tee_time_slot_id, computed_at DESC);
```

---

## Marketing & Engagement

```sql
-- =====================================================
-- MARKETING & ENGAGEMENT
-- =====================================================

CREATE TABLE marketing_campaign (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    name                VARCHAR(200) NOT NULL,
    campaign_type       VARCHAR(30) NOT NULL
                        CHECK (campaign_type IN ('email','sms','push','in_app','social','retention','winback')),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','scheduled','active','paused','completed')),
    target_segment      VARCHAR(100),
    scheduled_at        TIMESTAMPTZ,
    sent_count          INTEGER DEFAULT 0,
    open_count          INTEGER DEFAULT 0,
    click_count         INTEGER DEFAULT 0,
    conversion_count    INTEGER DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE campaign_recipient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES marketing_campaign(id),
    player_id           UUID NOT NULL REFERENCES player(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','sent','delivered','opened','clicked','bounced','unsubscribed')),
    sent_at             TIMESTAMPTZ,
    opened_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_recipient_campaign ON campaign_recipient(campaign_id);
```

---

## Course Conditions & Maintenance

```sql
-- =====================================================
-- COURSE CONDITIONS & MAINTENANCE
-- =====================================================

CREATE TABLE course_condition_report (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES course(id),
    report_date         DATE NOT NULL,
    reported_by         UUID,                          -- staff member
    green_speed         NUMERIC(4,1),                  -- stimpmeter reading
    fairway_status      VARCHAR(20) CHECK (fairway_status IN ('excellent','good','fair','poor','closed')),
    green_status        VARCHAR(20) CHECK (green_status IN ('excellent','good','fair','poor','closed')),
    bunker_status       VARCHAR(20) CHECK (bunker_status IN ('excellent','good','fair','poor','closed')),
    cart_path_only      BOOLEAN NOT NULL DEFAULT false,
    walking_only        BOOLEAN NOT NULL DEFAULT false,
    notes               TEXT,
    weather_temp_f      SMALLINT,
    weather_wind_mph    SMALLINT,
    weather_condition   VARCHAR(30),
    published           BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_report ON course_condition_report(course_id, report_date DESC);

CREATE TABLE maintenance_task (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    task_type           VARCHAR(30) NOT NULL
                        CHECK (task_type IN ('mowing','aeration','irrigation','fertilization','pest_control',
                               'equipment_service','bunker','cart_path','tree_work','general')),
    description         TEXT NOT NULL,
    assigned_to         UUID,                          -- staff member
    priority            VARCHAR(10) NOT NULL DEFAULT 'medium'
                        CHECK (priority IN ('low','medium','high','urgent')),
    status              VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                        CHECK (status IN ('scheduled','in_progress','completed','deferred','cancelled')),
    scheduled_date      DATE,
    completed_date      DATE,
    affected_holes      SMALLINT[],                    -- array of hole numbers affected
    estimated_duration  INTERVAL,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_maintenance_task_facility ON maintenance_task(facility_id, status, scheduled_date);
```

---

## API & Integration

```sql
-- =====================================================
-- API PARTNER & INTEGRATION
-- =====================================================

CREATE TABLE api_partner (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    partner_name        VARCHAR(200) NOT NULL,
    client_id           VARCHAR(200) NOT NULL UNIQUE,
    client_secret_hash  VARCHAR(200) NOT NULL,
    redirect_uri        VARCHAR(500),
    scopes              TEXT[],                        -- e.g. {'bookings:read','bookings:write','players:read'}
    partner_type        VARCHAR(30) NOT NULL
                        CHECK (partner_type IN ('marketplace','booking_engine','handicap','analytics','marketing','custom')),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    rate_limit_per_hour INTEGER DEFAULT 1000,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_webhook (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_partner_id      UUID NOT NULL REFERENCES api_partner(id),
    event_type          VARCHAR(50) NOT NULL,          -- e.g. 'booking.created', 'score.posted'
    target_url          VARCHAR(500) NOT NULL,
    secret_hash         VARCHAR(200),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_triggered_at   TIMESTAMPTZ,
    failure_count       INTEGER DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE marketplace_listing (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    marketplace         VARCHAR(30) NOT NULL
                        CHECK (marketplace IN ('golfnow','supreme_golf','teeoff','google_reserve','custom')),
    external_course_id  VARCHAR(100),
    listing_status      VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (listing_status IN ('active','paused','pending','rejected')),
    commission_rate     NUMERIC(5,2),                  -- percentage
    sync_enabled        BOOLEAN NOT NULL DEFAULT true,
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_marketplace_listing ON marketplace_listing(facility_id, marketplace);
```

---

## Staff & Access Control

```sql
-- =====================================================
-- STAFF & RBAC
-- =====================================================

CREATE TABLE staff (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id         UUID NOT NULL REFERENCES facility(id),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(200) NOT NULL,
    phone               VARCHAR(30),
    role                VARCHAR(30) NOT NULL
                        CHECK (role IN ('owner','general_manager','pro','assistant_pro','starter','ranger',
                               'pro_shop','f_and_b','maintenance','admin','bookkeeper')),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    auth_provider_id    VARCHAR(200),                  -- OpenID Connect subject
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permission_key      VARCHAR(100) NOT NULL UNIQUE,  -- e.g. 'booking.manage', 'pos.refund'
    description         VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role_permission (
    role                VARCHAR(30) NOT NULL,
    permission_id       UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role, permission_id)
);

CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    actor_id            UUID,                          -- staff or system
    actor_type          VARCHAR(20) NOT NULL CHECK (actor_type IN ('staff','player','system','api_partner')),
    action              VARCHAR(50) NOT NULL,          -- e.g. 'booking.create', 'payment.refund'
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    old_values          JSONB,
    new_values          JSONB,
    ip_address          INET,
    user_agent          VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Facility & Course | 4 | facility, course, hole, course_tee + hole_tee_distance |
| Tee Sheet & Booking | 5 | config, slots, bookings, players-in-booking, add-ons |
| Player & Membership | 6 | player, plans, memberships, billing, loyalty program, player loyalty |
| Handicap & Scoring | 3 | score posting, hole scores, handicap history |
| Point of Sale | 6 | terminals, categories, products, transactions, line items, payments |
| Tournament | 6 | tournament, entries, rounds, pairings, pairing players, scores |
| Dynamic Pricing | 3 | rate codes, rules, price snapshots |
| Marketing | 2 | campaigns, recipients |
| Course Conditions | 2 | condition reports, maintenance tasks |
| API & Integration | 3 | partners, webhooks, marketplace listings |
| Staff & Access | 4 | staff, permissions, role_permission, audit log |
| **Total** | **44** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — supports distributed systems, multi-tenant data merges, and API resource identification without exposing sequential IDs.

2. **Multi-tenant via `tenant_id` column** — uses PostgreSQL Row-Level Security for data isolation rather than separate schemas, reducing operational complexity for SaaS deployment.

3. **Separate `tee_time_slot` from `booking`** — the tee sheet is a time-slot inventory that exists independently of bookings, allowing blocked/maintenance slots and marketplace availability to be managed separately from customer reservations.

4. **GHIN and WHS fields on player and score tables** — direct alignment with USGA/R&A API field names so score posting and handicap sync require minimal transformation.

5. **No raw card data stored** — `payment.gateway_ref` holds tokenised references from Stripe/Braintree, keeping the platform at PCI DSS SAQ-A scope rather than handling card data directly.

6. **Audit log with JSONB diff** — `old_values`/`new_values` JSONB columns capture before/after state for any auditable action, providing compliance trail without event sourcing complexity.

7. **Dynamic pricing as rules + snapshots** — `dynamic_price_rule` defines the pricing logic, `price_snapshot` records the computed price at a point in time for auditability and AI training data.

8. **Tournament scoring links to handicap posting** — `tournament_score.score_posting_id` connects competition results directly to the WHS/GHIN score posting pipeline.

9. **Marketplace listing as a separate entity** — each distribution channel (GolfNow, Supreme Golf, Google Reserve) gets its own configuration record with commission rates and sync status, avoiding hardcoded integrations.

10. **ISO standard codes for all reference data** — ISO 3166 for countries, ISO 4217 for currencies, IANA for timezones — no proprietary code sets.
