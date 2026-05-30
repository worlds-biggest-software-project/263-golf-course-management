# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Golf Course Management · Created: 2026-05-22

## Philosophy

This model combines the referential integrity of relational tables for core operational data with PostgreSQL JSONB columns for variable, tenant-specific, and jurisdiction-dependent fields. The core business objects — facilities, courses, bookings, players, memberships, POS transactions, and tournaments — are defined with relational columns for the fields that are universal across all golf operations. Attributes that vary by facility type (public vs. private vs. resort), jurisdiction (US vs. EU vs. Asia-Pacific), integration partner, or tenant customisation are stored in typed JSONB columns with GIN indexes and documented JSON Schema contracts.

This is the pattern that modern SaaS platforms increasingly adopt when they must serve diverse customer segments with a single codebase. A municipal 9-hole course in Texas and a private 36-hole resort in Scotland share the same `facility` table, but their regulatory reporting fields, membership structures, handicap system integrations, and POS configurations differ substantially. Rather than adding nullable columns or maintaining separate schemas per locale, the hybrid approach stores these variations in well-indexed JSONB fields that can evolve without DDL migrations.

The approach aligns with how the golf API ecosystem actually works: GolfAPI.io, iGolf Connect, and Golfcourseapi.com all return JSON payloads with variable structures depending on the course. GHIN and WHS have different field requirements for US vs. international score submissions. A hybrid model can ingest and store these varying structures natively while keeping the operational query paths relational and fast.

**Best for:** Teams building a multi-market SaaS product that must accommodate diverse facility types, jurisdictions, and integration partners without per-tenant schema forks or excessive ALTER TABLE migrations.

**Trade-offs:**
- (+) Rapid feature development — new tenant-specific fields require no schema migration
- (+) Multi-jurisdiction flexibility — US/GHIN fields, EU/R&A fields, and regional tax configurations coexist cleanly
- (+) JSON Schema validation ensures JSONB data stays structured, not arbitrary
- (+) GIN indexes on JSONB provide efficient containment and path queries
- (+) Smaller table count than fully normalized (~18 tables vs. ~44)
- (+) Maps naturally to JSON API payloads — less transformation between storage and REST responses
- (-) JSONB fields cannot participate in foreign key constraints
- (-) Complex JSONB queries can be slower than indexed relational columns for high-cardinality data
- (-) Developers must understand both SQL and JSONB operators
- (-) Schema discipline required — without JSON Schema validation, JSONB fields can drift into unstructured chaos
- (-) Reporting/BI tools may have limited JSONB support compared to flat relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WHS Interoperability Standard v1.0 | Score posting `handicap_details` JSONB captures WHS-specific fields (PCC adjustment, exceptional score flags); different structures for US GHIN vs. R&A iGolf submissions |
| GHIN (USGA) | GHIN integration fields stored in `player.handicap` JSONB under a `"ghin"` key; score sync status in `score_posting.sync_status` JSONB |
| iGolf Connect (R&A) | International handicap fields stored under `player.handicap["igolf"]`; coexists with GHIN fields on the same player record |
| ISO 3166-1/2 | `country_code` relational column on facility and player; subdivision-specific regulatory config in `facility.config` JSONB |
| ISO 4217 | `currency_code` relational column on all monetary tables; multi-currency display preferences in `facility.config` JSONB |
| PCI DSS | No card data stored; `pos_transaction.payments` JSONB holds tokenised gateway references — no raw card numbers |
| GDPR / CCPA | Privacy consent fields in `player.privacy` JSONB with timestamped consent records per regulation; supports adding new privacy regulations without schema changes |
| OpenAPI 3.1 | JSONB fields documented with JSON Schema definitions in the OpenAPI spec; API responses can return JSONB fields directly |
| JSON Schema (draft 2020-12) | Every JSONB column has a corresponding JSON Schema definition enforced at application level and documented in the API spec |
| EMV / NFC | Terminal capabilities and compliance status in `pos_terminal.device_config` JSONB |

---

## Facility & Course

```sql
-- =====================================================
-- FACILITY & COURSE — Hybrid relational + JSONB
-- =====================================================

CREATE TABLE facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            VARCHAR(200) NOT NULL,
    facility_type   VARCHAR(30) NOT NULL CHECK (facility_type IN ('public','private','semi_private','resort','municipal')),
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL,             -- IANA timezone
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: address, contact, and location (varies by country)
    address         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "line1": "123 Fairway Dr", "line2": null,
    --   "city": "Scottsdale", "state": "AZ", "postal_code": "85255",
    --   "latitude": 33.6312, "longitude": -111.9256
    -- }

    -- JSONB: facility-level configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "default_interval_minutes": 8,
    --   "max_advance_booking_days": 14,
    --   "allow_online_booking": true,
    --   "tax_config": {"green_fee_rate": 0.0875, "food_rate": 0.065, "merchandise_rate": 0.0875},
    --   "operating_hours": {"summer": {"open": "05:30", "close": "20:00"}, "winter": {"open": "07:00", "close": "17:00"}},
    --   "pos_config": {"tip_enabled": true, "receipt_printer": "epson_t88"},
    --   "marketing": {"sms_enabled": true, "email_provider": "sendgrid"}
    -- }

    -- JSONB: external system identifiers
    external_ids    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ghin_club_id": "12345",
    --   "golfnow_course_id": "abc-def",
    --   "google_place_id": "ChIJ...",
    --   "igolf_facility_id": "UK-9876"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_facility_tenant ON facility(tenant_id);
CREATE INDEX idx_facility_country ON facility(country_code);
CREATE INDEX idx_facility_external_ids ON facility USING GIN (external_ids);

CREATE TABLE course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    name            VARCHAR(200) NOT NULL,
    holes_count     SMALLINT NOT NULL CHECK (holes_count IN (9, 18, 27, 36)),
    par             SMALLINT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: course details that vary by data source
    details         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "year_built": 1998, "architect": "Tom Fazio",
    --   "style": "desert", "grass_types": {"fairway": "bermuda", "green": "bentgrass"},
    --   "amenities": ["driving_range", "putting_green", "short_game_area"],
    --   "cart_gps_enabled": true
    -- }

    -- JSONB: tee set configurations (replaces separate course_tee + hole_tee_distance tables)
    tee_sets        JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "name": "Championship", "color": "Black", "gender": "M",
    --     "course_rating": 74.2, "slope_rating": 142, "bogey_rating": 97.8,
    --     "total_yardage": 7245,
    --     "holes": [
    --       {"number": 1, "par": 4, "stroke_index": 7, "yardage": 425},
    --       {"number": 2, "par": 3, "stroke_index": 15, "yardage": 185}
    --     ]
    --   },
    --   {
    --     "name": "Member", "color": "Blue", "gender": "M",
    --     "course_rating": 71.5, "slope_rating": 128, "bogey_rating": 94.1,
    --     "total_yardage": 6580,
    --     "holes": [...]
    --   }
    -- ]

    -- JSONB: external data source identifiers
    external_ids    JSONB NOT NULL DEFAULT '{}',
    -- {"ghin_course_id": "...", "igolf_course_id": "...", "golfapi_club_id": "..."}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_facility ON course(facility_id);
CREATE INDEX idx_course_external_ids ON course USING GIN (external_ids);
```

---

## Tee Sheet & Booking

```sql
-- =====================================================
-- TEE SHEET & BOOKING
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

    -- JSONB: pricing context at this point in time
    pricing         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "base_green_fee": 95.00, "base_cart_fee": 25.00,
    --   "rate_code": "peak_weekend",
    --   "dynamic_adjustments": [
    --     {"rule": "weather_discount", "adjustment_pct": -15, "reason": "rain_60pct"},
    --     {"rule": "occupancy_boost", "adjustment_pct": -10, "reason": "below_50pct"}
    --   ],
    --   "computed_green_fee": 71.25
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, slot_date, slot_time, starting_hole)
);

CREATE INDEX idx_slot_course_date ON tee_time_slot(course_id, slot_date, status);

CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    tee_time_slot_id UUID NOT NULL REFERENCES tee_time_slot(id),
    booked_by_id    UUID,                              -- player who made the booking
    booking_source  VARCHAR(30) NOT NULL
                    CHECK (booking_source IN ('online','phone','walk_in','marketplace','api','staff')),
    player_count    SMALLINT NOT NULL CHECK (player_count BETWEEN 1 AND 8),
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed'
                    CHECK (status IN ('pending','confirmed','checked_in','completed','cancelled','no_show')),
    total_amount    NUMERIC(10,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    -- JSONB: players in this booking (replaces booking_player junction table)
    players         JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "player_id": "uuid", "name": "John Smith", "position": 1,
    --     "green_fee": 85.00, "cart_fee": 25.00, "is_walking": false,
    --     "checked_in": true, "checked_in_at": "2026-06-15T08:15:00Z"
    --   },
    --   {
    --     "player_id": null, "name": "Guest Player", "position": 2,
    --     "green_fee": 95.00, "cart_fee": 25.00, "is_walking": false,
    --     "checked_in": false
    --   }
    -- ]

    -- JSONB: add-on services
    add_ons         JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "cart", "quantity": 2, "unit_price": 25.00},
    --   {"type": "range_balls", "quantity": 1, "unit_price": 8.00}
    -- ]

    -- JSONB: marketplace/channel metadata
    channel_meta    JSONB NOT NULL DEFAULT '{}',
    -- {"marketplace": "golfnow", "confirmation_ref": "GN-2026-X9Y2", "commission_pct": 12.0}

    notes           TEXT,
    cancelled_at    TIMESTAMPTZ,
    checked_in_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_booking_slot ON booking(tee_time_slot_id);
CREATE INDEX idx_booking_facility_date ON booking(facility_id, created_at);
CREATE INDEX idx_booking_players ON booking USING GIN (players);
CREATE INDEX idx_booking_channel ON booking USING GIN (channel_meta);
```

---

## Player & Membership

```sql
-- =====================================================
-- PLAYER & MEMBERSHIP
-- =====================================================

CREATE TABLE player (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(200),
    phone           VARCHAR(30),
    country_code    CHAR(2),                           -- ISO 3166-1
    player_type     VARCHAR(20) NOT NULL DEFAULT 'guest'
                    CHECK (player_type IN ('member','guest','visitor','staff','pro')),
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: full contact/address details (varies by jurisdiction)
    profile         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "date_of_birth": "1985-03-22", "gender": "M",
    --   "address": {"line1": "...", "city": "...", "state": "...", "postal_code": "..."},
    --   "emergency_contact": {"name": "Jane Smith", "phone": "+1-555-0199"},
    --   "preferences": {"preferred_tee": "Blue", "walking_preference": true, "shirt_size": "L"},
    --   "tags": ["frequent_player", "tournament_active", "lesson_interest"]
    -- }

    -- JSONB: handicap integration data (supports GHIN + iGolf simultaneously)
    handicap        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ghin": {"number": "1234567", "index": 12.4, "last_sync": "2026-06-01T10:00:00Z"},
    --   "igolf": {"id": "UK-98765", "index": 12.8, "last_sync": "2026-05-28T14:00:00Z"},
    --   "whs_id": "WHS-GLOBAL-123",
    --   "active_system": "ghin",
    --   "low_index": 10.2,
    --   "trend": "improving"
    -- }

    -- JSONB: privacy and consent (GDPR, CCPA, etc.)
    privacy         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "marketing_opt_in": true, "marketing_opt_in_at": "2026-01-15T09:00:00Z",
    --   "gdpr_consent": true, "gdpr_consent_at": "2026-01-15T09:00:00Z",
    --   "ccpa_opt_out_sale": false,
    --   "data_retention_agreed": true
    -- }

    -- JSONB: external system links
    integrations    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "stripe_customer_id": "cus_ABC123",
    --   "mailchimp_id": "mc-xyz",
    --   "golfgenius_player_id": "gg-456"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_player_tenant ON player(tenant_id);
CREATE INDEX idx_player_email ON player(email);
CREATE INDEX idx_player_name ON player(last_name, first_name);
CREATE INDEX idx_player_handicap ON player USING GIN (handicap);
CREATE INDEX idx_player_integrations ON player USING GIN (integrations);

CREATE TABLE membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id       UUID NOT NULL REFERENCES player(id),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    member_number   VARCHAR(30),
    plan_name       VARCHAR(100) NOT NULL,
    plan_type       VARCHAR(30) NOT NULL
                    CHECK (plan_type IN ('full_golf','limited_golf','social','junior','senior','corporate','family','trial')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('pending','active','suspended','expired','cancelled')),
    start_date      DATE NOT NULL,
    end_date        DATE,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    -- JSONB: billing and financial terms (varies by plan and facility)
    billing         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "frequency": "monthly", "amount": 450.00,
    --   "initiation_fee": 5000.00, "initiation_paid": true,
    --   "billing_day": 1,
    --   "balance_due": 0.00,
    --   "payment_method_token": "pm_ABC123",
    --   "auto_renew": true, "renewal_date": "2027-01-01"
    -- }

    -- JSONB: plan entitlements (varies significantly by facility)
    entitlements    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "includes_cart": true, "includes_range": true,
    --   "guest_passes_per_year": 12, "guest_passes_used": 3,
    --   "advance_booking_days": 14, "priority_booking": true,
    --   "locker_assigned": "A-127",
    --   "dining_minimum": {"amount": 200.00, "frequency": "quarterly"},
    --   "club_storage": true
    -- }

    -- JSONB: loyalty tracking
    loyalty         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "points_balance": 4500, "lifetime_points": 28000,
    --   "tier": "gold", "tier_expiry": "2026-12-31",
    --   "points_per_dollar": 1.5
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_player ON membership(player_id);
CREATE INDEX idx_membership_facility ON membership(facility_id, status);
CREATE INDEX idx_membership_entitlements ON membership USING GIN (entitlements);
```

---

## Handicap & Scoring

```sql
-- =====================================================
-- HANDICAP & SCORING
-- =====================================================

CREATE TABLE score_posting (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id           UUID NOT NULL REFERENCES player(id),
    course_id           UUID NOT NULL REFERENCES course(id),
    played_at           DATE NOT NULL,
    holes_played        SMALLINT NOT NULL CHECK (holes_played IN (9, 18)),
    adjusted_gross_score SMALLINT NOT NULL,
    score_differential  NUMERIC(5,1),
    score_type          VARCHAR(20) NOT NULL
                        CHECK (score_type IN ('home','away','tournament','internet','penalty','combined_9')),

    -- JSONB: tee set snapshot at time of play (denormalised from course.tee_sets)
    tee_set_snapshot    JSONB NOT NULL,
    -- {
    --   "name": "Blue", "color": "Blue", "gender": "M",
    --   "course_rating": 71.5, "slope_rating": 128, "bogey_rating": 94.1
    -- }

    -- JSONB: hole-by-hole detail (replaces separate hole_score table)
    hole_scores         JSONB,
    -- [
    --   {"hole": 1, "par": 4, "strokes": 5, "adjusted": 5, "putts": 2, "fir": true, "gir": false},
    --   {"hole": 2, "par": 3, "strokes": 3, "adjusted": 3, "putts": 1, "fir": null, "gir": true}
    -- ]

    -- JSONB: WHS calculation details
    handicap_details    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "pcc_adjustment": 0.0, "exceptional_score": false,
    --   "is_nine_hole": false, "combined_with": null
    -- }

    -- JSONB: sync status with external systems
    sync_status         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ghin": {"posted": true, "posted_at": "2026-06-15T12:00:00Z", "score_id": "G-12345"},
    --   "igolf": {"posted": false, "last_attempt": null, "error": null}
    -- }

    source              VARCHAR(20) NOT NULL DEFAULT 'manual'
                        CHECK (source IN ('manual','ghin_sync','app','tournament_system')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_score_player_date ON score_posting(player_id, played_at DESC);
CREATE INDEX idx_score_course ON score_posting(course_id, played_at);
CREATE INDEX idx_score_sync ON score_posting USING GIN (sync_status);
```

---

## Point of Sale

```sql
-- =====================================================
-- POINT OF SALE
-- =====================================================

CREATE TABLE pos_terminal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    terminal_name   VARCHAR(100) NOT NULL,
    location        VARCHAR(50) NOT NULL
                    CHECK (location IN ('pro_shop','restaurant','bar','halfway_house','snack_bar','beverage_cart','starter')),
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: device capabilities and config
    device_config   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "serial": "SN-12345", "model": "PAX A920",
    --   "emv_compliant": true, "nfc_enabled": true,
    --   "receipt_printer": true, "barcode_scanner": true
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    sku             VARCHAR(50),
    name            VARCHAR(200) NOT NULL,
    department      VARCHAR(30) NOT NULL
                    CHECK (department IN ('pro_shop','food','beverage','merchandise','service','rental','lesson')),
    unit_price      NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: variable product attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "category": "Apparel > Polo Shirts", "brand": "Titleist",
    --   "cost_price": 32.00, "tax_rate": 0.0875,
    --   "track_inventory": true, "quantity_on_hand": 45, "reorder_point": 10,
    --   "sizes": ["S","M","L","XL"], "colors": ["Navy","White"],
    --   "vendor": "Acme Golf Supply", "vendor_sku": "TIT-POLO-NV"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_facility ON product(facility_id, department);
CREATE INDEX idx_product_sku ON product(sku);
CREATE INDEX idx_product_attributes ON product USING GIN (attributes);

CREATE TABLE pos_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    terminal_id     UUID REFERENCES pos_terminal(id),
    player_id       UUID REFERENCES player(id),
    transaction_type VARCHAR(20) NOT NULL
                    CHECK (transaction_type IN ('sale','return','void','exchange','member_charge')),
    subtotal        NUMERIC(10,2) NOT NULL,
    tax_amount      NUMERIC(10,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(10,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    -- JSONB: line items (replaces pos_transaction_item table)
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"product_id": "uuid", "sku": "TIT-PRO-V1", "name": "Titleist Pro V1 (dozen)",
    --    "quantity": 1, "unit_price": 54.99, "discount": 0, "tax": 4.81, "line_total": 59.80},
    --   {"product_id": "uuid", "sku": "BEER-IPA",  "name": "IPA Draft 16oz",
    --    "quantity": 2, "unit_price": 8.00, "discount": 0, "tax": 1.04, "line_total": 17.04}
    -- ]

    -- JSONB: payment details (replaces separate payment table for simple cases)
    payments        JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"method": "card", "amount": 76.84, "gateway_ref": "pi_3ABC123",
    --    "card_last_four": "4242", "card_brand": "visa", "status": "completed"},
    --   {"method": "member_charge", "amount": 0, "membership_id": "uuid"}
    -- ]

    staff_id        UUID,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pos_facility_date ON pos_transaction(facility_id, created_at);
CREATE INDEX idx_pos_player ON pos_transaction(player_id);
CREATE INDEX idx_pos_line_items ON pos_transaction USING GIN (line_items);
```

---

## Tournament Management

```sql
-- =====================================================
-- TOURNAMENT MANAGEMENT
-- =====================================================

CREATE TABLE tournament (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    course_id       UUID NOT NULL REFERENCES course(id),
    name            VARCHAR(200) NOT NULL,
    tournament_type VARCHAR(30) NOT NULL
                    CHECK (tournament_type IN ('stroke_play','match_play','stableford','scramble','best_ball',
                           'alternate_shot','chapman','shamble','charity','corporate')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','registration_open','registration_closed','in_progress','completed','cancelled')),

    -- JSONB: tournament configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "scoring_format": "net", "rounds": 2,
    --   "max_players": 144, "handicap_allowance": 0.80,
    --   "entry_fee": 75.00, "currency_code": "USD",
    --   "prizes": [{"place": 1, "amount": 500}, {"place": 2, "amount": 250}],
    --   "tee_assignments": {"mens": "Blue", "womens": "Red"},
    --   "shotgun_start": true, "flights": 4,
    --   "golfgenius_event_id": "gg-evt-123"
    -- }

    -- JSONB: entries with pairings (replaces 3 separate tables)
    entries         JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "player_id": "uuid", "name": "John Smith",
    --     "status": "confirmed", "playing_handicap": 12.4,
    --     "entry_fee_paid": true, "payment_id": "uuid",
    --     "team_id": "A",
    --     "rounds": [
    --       {"round": 1, "group": 5, "starting_hole": 1, "tee_time": "08:00",
    --        "gross": 82, "net": 70, "stableford": 36}
    --     ]
    --   }
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tournament_facility ON tournament(facility_id, start_date);
CREATE INDEX idx_tournament_entries ON tournament USING GIN (entries);
```

---

## Dynamic Pricing

```sql
-- =====================================================
-- DYNAMIC PRICING
-- =====================================================

CREATE TABLE pricing_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    rule_name       VARCHAR(100) NOT NULL,
    priority        SMALLINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: rule conditions and actions (replaces multiple columns)
    rule_definition JSONB NOT NULL,
    -- {
    --   "triggers": ["occupancy", "weather", "advance_days"],
    --   "conditions": [
    --     {"type": "occupancy_below", "threshold": 50},
    --     {"type": "weather_forecast", "condition": "rain", "probability_above": 60}
    --   ],
    --   "action": {
    --     "type": "percent_discount", "value": -15,
    --     "floor_price": 45.00, "ceiling_price": null,
    --     "applies_to": ["green_fee"]
    --   },
    --   "schedule": {
    --     "valid_from": "2026-06-01", "valid_to": "2026-08-31",
    --     "days": ["mon","tue","wed","thu","fri"],
    --     "time_range": {"start": "11:00", "end": "14:00"}
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pricing_rule_facility ON pricing_rule(facility_id, is_active);
```

---

## Course Conditions & Maintenance

```sql
-- =====================================================
-- COURSE CONDITIONS & MAINTENANCE
-- =====================================================

CREATE TABLE course_condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course(id),
    report_date     DATE NOT NULL,
    reported_by     UUID,
    published       BOOLEAN NOT NULL DEFAULT false,

    -- JSONB: condition details (varies by course type and season)
    conditions      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "green_speed": 10.5, "green_status": "excellent",
    --   "fairway_status": "good", "bunker_status": "good",
    --   "rough_height_inches": 2.5,
    --   "cart_path_only": false, "walking_only": false,
    --   "frost_delay": false,
    --   "pin_positions": "front",
    --   "notes": "Aerated greens on holes 10-14, expect slower speeds",
    --   "hole_closures": [],
    --   "weather": {"temp_f": 78, "wind_mph": 12, "condition": "partly_cloudy", "humidity_pct": 45}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_course_date ON course_condition(course_id, report_date DESC);

CREATE TABLE maintenance_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled','in_progress','completed','deferred','cancelled')),
    priority        VARCHAR(10) NOT NULL DEFAULT 'medium'
                    CHECK (priority IN ('low','medium','high','urgent')),
    scheduled_date  DATE,
    completed_date  DATE,

    -- JSONB: task details (varies widely by task type)
    details         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "task_type": "aeration",
    --   "description": "Core aeration on front 9 greens",
    --   "assigned_to": "uuid",
    --   "affected_holes": [1,2,3,4,5,6,7,8,9],
    --   "estimated_hours": 6,
    --   "equipment_needed": ["aerator_toro_procore"],
    --   "materials": [{"name": "top_dressing_sand", "quantity_tons": 2}],
    --   "weather_dependent": true,
    --   "recurring": {"frequency": "quarterly", "next_due": "2026-09-15"}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_maintenance_facility ON maintenance_task(facility_id, status, scheduled_date);
```

---

## Marketing & Engagement

```sql
-- =====================================================
-- MARKETING & ENGAGEMENT
-- =====================================================

CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    name            VARCHAR(200) NOT NULL,
    campaign_type   VARCHAR(30) NOT NULL
                    CHECK (campaign_type IN ('email','sms','push','in_app','social','retention','winback')),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','scheduled','active','paused','completed')),

    -- JSONB: campaign configuration and metrics
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "target_segment": "members_inactive_30_days",
    --   "scheduled_at": "2026-07-01T09:00:00Z",
    --   "template_id": "summer-special-2026",
    --   "offer": {"type": "discount", "value": 20, "unit": "percent", "valid_until": "2026-07-31"},
    --   "metrics": {"sent": 450, "delivered": 442, "opened": 198, "clicked": 67, "converted": 12},
    --   "ab_test": {"variant_a": "subject_line_1", "variant_b": "subject_line_2", "winner": "a"}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_facility ON campaign(facility_id, status);
```

---

## Staff & Access Control

```sql
-- =====================================================
-- STAFF & ACCESS CONTROL
-- =====================================================

CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facility(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(200) NOT NULL,
    role            VARCHAR(30) NOT NULL
                    CHECK (role IN ('owner','general_manager','pro','assistant_pro','starter','ranger',
                           'pro_shop','f_and_b','maintenance','admin','bookkeeper')),
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: permissions and capabilities
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "booking": ["view","create","modify","cancel"],
    --   "pos": ["sell","refund","void","close_register"],
    --   "members": ["view","create","modify"],
    --   "reports": ["view_revenue","view_utilisation","export"],
    --   "pricing": ["view","modify_rules"],
    --   "tournaments": ["view","create","manage_scoring"]
    -- }

    -- JSONB: auth provider links
    auth            JSONB NOT NULL DEFAULT '{}',
    -- {"provider": "oidc", "subject_id": "auth0|abc123", "last_login": "2026-06-15T08:00:00Z"}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_facility ON staff(facility_id, is_active);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL CHECK (actor_type IN ('staff','player','system','api_partner')),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                             -- {"old": {...}, "new": {...}}
    context         JSONB NOT NULL DEFAULT '{}',       -- {"ip": "...", "user_agent": "...", "api_partner": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Marketplace & API Integration

```sql
-- =====================================================
-- MARKETPLACE & API INTEGRATION
-- =====================================================

CREATE TABLE api_partner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    partner_name    VARCHAR(200) NOT NULL,
    partner_type    VARCHAR(30) NOT NULL
                    CHECK (partner_type IN ('marketplace','booking_engine','handicap','analytics','marketing','custom')),
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: credentials and config (replaces multiple columns)
    credentials     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "client_id": "abc123", "redirect_uri": "https://...",
    --   "scopes": ["bookings:read","bookings:write","players:read"],
    --   "rate_limit_per_hour": 1000
    -- }
    -- Note: client_secret_hash stored separately in vault, NOT in JSONB

    -- JSONB: channel-specific configuration
    channel_config  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "marketplace": "golfnow",
    --   "courses": [{"course_id": "uuid", "external_id": "GN-12345", "commission_pct": 12.0}],
    --   "sync_enabled": true, "last_sync_at": "2026-06-15T06:00:00Z",
    --   "webhooks": [
    --     {"event": "booking.created", "url": "https://...", "active": true, "failures": 0}
    --   ]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_partner_tenant ON api_partner(tenant_id, is_active);
CREATE INDEX idx_api_partner_channel ON api_partner USING GIN (channel_config);
```

---

## Example Queries — JSONB Operations

```sql
-- Find all players with a GHIN number
SELECT id, first_name, last_name, handicap->>'ghin' AS ghin_data
FROM player
WHERE handicap ? 'ghin';

-- Find players whose GHIN handicap index is below 10
SELECT id, first_name, last_name,
       (handicap->'ghin'->>'index')::numeric AS handicap_index
FROM player
WHERE (handicap->'ghin'->>'index')::numeric < 10.0;

-- Find bookings from GolfNow marketplace
SELECT id, created_at, total_amount, channel_meta
FROM booking
WHERE channel_meta @> '{"marketplace": "golfnow"}';

-- Find all tee time slots with weather-based price adjustments
SELECT slot_date, slot_time, current_price,
       pricing->'dynamic_adjustments' AS adjustments
FROM tee_time_slot
WHERE pricing->'dynamic_adjustments' @> '[{"rule": "weather_discount"}]';

-- Search products by brand using GIN index
SELECT id, name, unit_price, attributes->>'brand' AS brand
FROM product
WHERE attributes @> '{"brand": "Titleist"}';

-- Find members with dining minimum entitlements
SELECT p.first_name, p.last_name, m.plan_name,
       m.entitlements->'dining_minimum' AS dining_min
FROM membership m
JOIN player p ON p.id = m.player_id
WHERE m.entitlements ? 'dining_minimum'
  AND m.status = 'active';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Facility & Course | 2 | facility, course (tee sets + holes embedded in course JSONB) |
| Tee Sheet & Booking | 2 | tee_time_slot, booking (players + add-ons in JSONB) |
| Player & Membership | 2 | player, membership (billing + entitlements + loyalty in JSONB) |
| Handicap & Scoring | 1 | score_posting (hole scores + sync status in JSONB) |
| Point of Sale | 3 | pos_terminal, product, pos_transaction (line items + payments in JSONB) |
| Tournament | 1 | tournament (entries + pairings + scores in JSONB) |
| Dynamic Pricing | 1 | pricing_rule (conditions + actions in JSONB) |
| Course Conditions | 2 | course_condition, maintenance_task |
| Marketing | 1 | campaign (config + metrics in JSONB) |
| Staff & Access | 2 | staff (permissions in JSONB), audit_log |
| API & Integration | 1 | api_partner (credentials + channel config in JSONB) |
| **Total** | **18** | vs. 44 tables in the normalized model |

---

## Key Design Decisions

1. **18 tables instead of 44** — by embedding variable-structure data (players in bookings, line items in transactions, tee sets in courses, entitlements in memberships) as JSONB, the schema eliminates ~26 junction and detail tables. This dramatically simplifies migrations, reduces JOIN complexity, and accelerates development.

2. **GIN indexes on every JSONB column** — PostgreSQL GIN indexes support `@>` (containment), `?` (key existence), and `?|` / `?&` (key set) operators, enabling efficient queries into JSONB without full table scans. This is essential for queries like "find all bookings from GolfNow" or "find players with GHIN numbers."

3. **Relational columns for query-critical fields** — `status`, `slot_date`, `slot_time`, `facility_id`, `player_type`, `country_code`, and other high-cardinality filter fields remain relational columns with B-tree indexes. Only variable or secondary data goes into JSONB.

4. **Dual handicap system support via JSONB** — `player.handicap` stores both GHIN (US) and iGolf (international) data under separate keys, enabling a player who belongs to clubs in multiple countries to maintain both handicap records without separate tables or nullable column sprawl.

5. **JSON Schema contracts enforced at application layer** — every JSONB column has a documented JSON Schema (referenced in the OpenAPI spec). Application middleware validates incoming JSONB against the schema before write. This prevents the "JSONB chaos" anti-pattern where unstructured data accumulates.

6. **Tournament entries as JSONB array** — a 144-player tournament stores all entries, pairings, and scores in a single JSONB column. This is a deliberate trade-off: it sacrifices per-player relational querying for dramatically simpler tournament CRUD and the ability to version/snapshot the entire tournament state atomically.

7. **POS transaction line items embedded** — a typical pro shop sale has 1-5 line items. Embedding them as JSONB eliminates a JOIN for every receipt display while still supporting GIN-indexed product lookups across transactions.

8. **Privacy consent as JSONB** — as new privacy regulations emerge (beyond GDPR and CCPA), new consent fields can be added to the `player.privacy` JSONB without schema migrations — critical for a platform operating across multiple jurisdictions.

9. **Pricing rules as self-contained JSONB documents** — each rule is a complete specification (conditions, actions, schedule) in a single JSONB column. This makes rules portable, versionable, and easy to clone/modify without touching multiple related tables.

10. **External IDs consolidated into single JSONB columns** — rather than adding a new VARCHAR column for every integration partner (GHIN, iGolf, GolfNow, Google, Golf Genius), external identifiers live in `external_ids` JSONB. Adding a new integration requires zero schema changes.
