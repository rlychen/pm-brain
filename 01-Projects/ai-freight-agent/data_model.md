# Data Model

*Part of the AI Freight Routing PRD. See [PRD.md](PRD.md) for strategic overview and document map.*

**Sections:** Supply and Demand Model · Rolling Horizon Planning · Customer and Tenant Entity Model

---

## 1. Supply and Demand Model

### 1.1 Demand — Ocean Shipments

Each ocean shipment request provides:

| Field | Description |
|---|---|
| `shipment_id` | Unique identifier |
| `origin` | City or address (geocoded to nearest port/pickup zone) |
| `destination` | City or address (geocoded to nearest port/delivery zone) |
| `cargo_ready_date` | Earliest pickup date at origin |
| `pickup_window` | [earliest, latest] pickup datetime |
| `required_delivery` | Latest acceptable arrival datetime at destination |
| `weight_kg` | Gross weight |
| `volume_cbm` | Volume in cubic meters |
| `cargo_type` | General, hazmat class, temperature-controlled, OOG |
| `hs_code` | 6-digit Harmonized System commodity code — determines customs inspection risk tier (P1) and tariff lookup. MVP: single code per shipment. **P1:** replace with list of HS codes to support mixed-product FCL bookings; inspection risk = max tier across codes. |
| `incoterm` | EXW, FOB, CIF, DDP, etc. |
| `service_level` | Economy, Standard, Express |
| `carrier_preferences` | Preferred, acceptable, excluded carriers |
| `budget_cap` | Optional hard cost cap per shipment |

### 1.2 Demand — Trucking Shipments

Each trucking shipment request provides:

| Field | Description |
|---|---|
| `shipment_id` | Unique identifier |
| `origin` | Pickup address |
| `destination` | Delivery address |
| `pickup_window` | [earliest, latest] pickup datetime |
| `delivery_window` | [latest_arrival] (one-sided — hard deadline) or [earliest, latest] |
| `weight_kg` | Gross weight |
| `volume_cbm` | Volume |
| `pallet_count` | Number of pallets (for load planning) |
| `cargo_type` | General, hazmat, temperature-controlled |
| `service_level` | Standard, Expedite, Dedicated |
| `carrier_preferences` | Preferred, acceptable, excluded |

### 1.3 Supply — The Graph G(N, A)

The routing network is modeled as a directed graph G(N, A):

**Nodes N** represent physical locations and logical waypoints:
- Origin locations (factories, warehouses, pickup addresses)
- Origin ports (container terminals with sailing schedules)
- Transshipment ports (intermediate hub ports)
- Destination ports
- Inland distribution points (rail ramps, cross-docks, truck terminals)
- Final destinations (warehouses, DCs, delivery addresses)

**Arcs A** represent feasible connections between nodes:
- **Ocean arcs**: carrier service legs between port pairs, with scheduled departure/arrival, capacity, and rate
- **Pre-carriage arcs**: pickup truck legs from origin to origin port
- **Drayage arcs**: port-to-inland-hub truck legs
- **Inland trucking arcs**: point-to-point truck moves
- **Transshipment arcs**: inter-terminal transfers at hub ports (time and cost)

**Container types:** FEU (40'HC, ≈ 76 CBM usable, ≈ 26,500 kg payload) and TEU (20', ≈ 33 CBM usable, ≈ 24,000 kg payload). The optimizer selects the cost-minimizing mix per (commodity, sailing) pair. Minimum container count per shipment: `n_k = max(ceil(volume_cbm / 76), ceil(weight_kg / 26500))`, accounting for both volume and weight limits. Container mix pre-computation is described in the formal model (`model/ocean_fcl_routing.tex`, Section 4.6).

| Container type | Code | TEU slots | Usable volume (CBM) | Payload (kg) | Notes |
|---|---|---|---|---|---|
| 20' standard | TEU | 1 | 33 | 24,000 | — |
| 40' standard | FEU (std) | 2 | 67 | 26,500 | Deferred to P1 |
| 40' High Cube | FEU (HC) | 2 | 76 | 26,500 | MVP FEU type; dominant on TPEB |

**Arc attribute schemas by type:**

*Pre-carriage arcs (origin door → POL):*
| Attribute | Description |
|---|---|
| `cost` | Flat truck rate per move (not per CBM — FCL is a dedicated truck) |
| `transit_time_mean` | Mean days, origin door to terminal gate |
| `transit_time_sigma` | Std dev of transit time |
| `distance_km` | Road distance (informational; used by instance generator) |

*Ocean sailing arcs (POL → POD_arrival):*
| Attribute | Description |
|---|---|
| `etd` | Estimated time of departure from POL |
| `cy_cutoff` | **Latest** datetime a container may arrive at terminal for this sailing. Hard constraint on pre-carriage delivery. Typically ETD − 4 days. |
| `rate_per_feu` | Base ocean freight rate per 40'HC container (USD/FEU) |
| `rate_per_teu` | Base rate per 20' container (USD/TEU). Typically 0.56–0.86 × FEU rate — varies by trade lane (TPEB: ~0.79, FEWB: ~0.56, TAWB: ~0.78). Not 0.5× because TEU costs more per CBM to handle. |
| `baf` | Bunker adjustment factor per container (same fraction applied to FEU and TEU rates) |
| `thc_pol` | Terminal handling charge at origin port per container |
| `thc_pod` | Terminal handling charge at destination port per container |
| `capacity_teu` | Per-sailing slot cap in TEU. 1 FEU = 2 TEU slots; 1 TEU = 1 TEU slot. Model constraint P.2 in `ocean_fcl_routing.tex`. MVP proxy: α × alloc(string, month), α ≈ 0.20 (non-binding placeholder; real value requires carrier data). |
| `service_string` | Carrier service string this sailing belongs to (e.g., "MSC Tiger"). Ties to carrier allocation pool. |
| `alloc_period` | Monthly allocation period this ETD falls in (YYYY-MM). |
| `transit_time_mean` | Mean transit days, POL to POD |
| `transit_time_sigma` | Std dev of transit time |
| `carrier` | Carrier name (e.g., MSC, CMA CGM, COSCO) |
| `vessel` | Vessel name |

*Dwell arcs (POD_arrival → POD_exit):*

Each physical port of discharge is represented by **two nodes**: a POD_arrival node (vessel has arrived, container discharged) and a POD_exit node (customs cleared, ready for inland pickup). These are connected by a dwell arc that makes port unloading and customs clearance an explicit model element rather than a hidden constant.

| Attribute | Description | MVP value |
|---|---|---|
| `unload_mean` | Mean time from vessel arrival to container accessible in terminal | Port-specific constant |
| `clearance_mean` | Mean customs processing time, no exam | Port-specific constant |
| `total_dwell_mean` | `unload_mean + clearance_mean` | USLAX/USLGB: 3.5 days; USSEA: 2.5 days |
| `total_dwell_sigma` | Std dev of total dwell | USLAX/USLGB: 1.5 days; USSEA: 1.0 days |
| `free_days` | Port storage free time before demurrage applies | 5 days (standard) |

**P1 — Commodity-specific dwell model (deferred):** In P1, dwell arc weight will become a function of commodity and importer attributes: HS code inspection risk tier, importer C-TPAT certification status, country of origin, and consignee inspection history. Inspection probability model: `p_exam = clip(λ_base × η_HS × η_origin × (1 − δ_CTPAT), 0, 1)`. Expected clearance time = `t_no_exam + p_exam × t_exam`.

*Inland arcs (POD_exit → destination door):*
| Attribute | Description |
|---|---|
| `cost` | Flat rate per container move (one chassis per FEU for FCL drayage) |
| `transit_time_mean` | Mean days, POD exit to destination door |
| `transit_time_sigma` | Std dev of transit time |
| `mode` | FTL truck (solid arc in diagram) or intermodal rail (dashed arc) |
| `distance_km` | Road/rail distance (informational) |

**Transit time estimation from coordinates:** For all trucking arcs (pre-carriage and inland), transit time is estimated from node lat/lon using: road distance = Haversine distance × road factor (1.25 China, 1.20 US); transit days = road distance / average truck speed (600 km/day China, 800 km/day US). For ocean arcs: sailing distance = Haversine × 1.15 (Trans-Pacific lane factor); transit days = sailing distance / 600 km/day (≈ 13.5 knots average, inclusive of port approach and anchorage). Validation: SHA → USLAX geodesic ≈ 9,200 km → sailing distance ≈ 10,580 km → transit ≈ 17.6 days, within published carrier schedule range of 14–18 days.

**String-based carrier allocation capacity:** Ocean carriers sell capacity on *service strings* — fixed port-call loops (e.g., MSC Tiger TPEB: SHA → NGB → SZX → USLAX → USLGB). A freight forwarder's contracted block space agreement (BSA) covers the entire string, not a specific port pair. A booking on SHA→USLAX and a booking on NGB→USLGB on the same string in the same month both draw from the same allocated pool.

The optimizer enforces two capacity constraints per sailing:
1. **Vessel-level cap** (`capacity_teu`) — constraint **P.2** in `ocean_fcl_routing.tex`: maximum TEU slots consumed on a single departure. Prevents the string allocation constraint alone from concentrating all monthly allocation onto one sailing. MVP proxy: `capacity_teu = α × alloc(string, month)`, α ≈ 0.20.
2. **String allocation cap** — constraint **P.3**: the forwarder's remaining contracted block on this string in this monthly period (`rem(s,t) = alloc(s,t) − util(s,t)`). Current utilization (`util`) is an external state input read from the shipment state store before each solve.

Routing decisions for two shipments on the same string in the same month are coupled through P.3 — the optimizer must respect the joint allocation cap even when the shipments use different port pairs. See formal model Sections 8.2–8.3 in `model/ocean_fcl_routing.tex`.

**Graph decomposition for batch solving:** When routing a batch of N shipments, the demand-supply graph can be decomposed before the MILP solve. Two shipments are independent if they share no feasible supply arcs — no common carrier service legs or carrier allocation pools. Independent subsets form disconnected components in the demand-supply intersection graph and are solved separately. This decomposition reduces MILP problem size and enables parallelism. Example: TPEB and FEWB commodities always decompose into independent subproblems.

---

## 2. Rolling Horizon Planning

*This is a first-principles design decision that differentiates this system from all existing TMS platforms.*

### 2.1 The Problem with Sequential Leg Planning

All existing TMS systems plan each mode leg sequentially: book the ocean leg, estimate the inland leg, execute. The inland leg estimate is made with high uncertainty at booking time (unknown exact arrival time, unknown port clearance delay, no specific carrier committed). By the time the vessel arrives, that estimate is stale but the planning system doesn't re-optimize.

### 2.2 Rolling Horizon Approach

This system maintains a **complete end-to-end plan at all times**, but resolves each leg at different levels of graph resolution depending on how close we are to execution:

- **G_coarse**: A sparse graph used for future legs. Arc weights are cost/time envelopes — ranges derived from historical data and ML models. Sufficient for the optimization objective but not for execution.
- **G_fine**: A dense graph used for the next leg to be executed. Arc weights are actual carrier schedules, confirmed spot rates, and port-specific clearance estimates.

As a shipment advances and uncertainty about the next leg decreases (e.g., AIS-derived vessel ETA confidence exceeds a threshold), the system fires a **re-plan trigger** and re-solves the next leg on G_fine with real schedules and live rates.

### 2.3 Concrete Example

**At booking time (T=0):**
- Ocean leg: precisely committed — specific carrier, vessel, sailing date, scheduled ETA ±3 days
- Inland leg (USLAX → Phoenix): rough envelope — $400–900, 1–3 days, no specific carrier

**Re-plan trigger fires when:** vessel is 72h from USLAX, AIS-derived ETA confidence > 90%, port clearance window confirmed

**At T=2 (vessel near USLAX):**
- Ocean leg: confirmed — actual ETA Jul 4 06:00 PST, port clearance est. Jul 4 14:00
- Inland leg: re-solved on G_fine — 3 specific options with actual carrier schedules and spot rates:
  - [A] Direct Truck (OHL) — depart Jul 4 16:00, arrive Jul 5 09:00 — $612 ✓ selected
  - [B] Drayage + Rail — depart Jul 4 20:00, arrive Jul 6 08:00 — $389
  - [C] Expedite (FedEx CC) — depart Jul 4 14:30, arrive Jul 4 22:00 — $1,480

### 2.4 Design Principle

The optimizer holds a full door-to-door plan at all times. Future legs are placeholders on G_coarse — enough to make good booking decisions. As each leg's execution window approaches, it is re-solved on G_fine with real data. This is **Model Predictive Control applied to multimodal freight routing**.

**This is a hard requirement.** The system must maintain and continuously update a full multi-horizon plan. Single-horizon planning (plan once, execute) is not acceptable.

---

## 3. Customer and Tenant Entity Model

*Multi-tenant SaaS architecture. Each Forwarder is a tenant (Organization). See `build_plan.md` for database implementation details.*

### 3.1 Entity Overview

**Organization** (tenant) — a freight forwarder. Has many Users, manages many Shippers, has contracted Carriers. This is the top-level tenant boundary. All data is isolated by `tenant_id` (the organization's UUID).

**User** — a human who logs in. Belongs to exactly one Organization. Has one of three Roles:
- `ops_planner` — daily execution; handles exception queue; can override agent decisions
- `analyst` — read-only analytics and reporting; cannot override agent
- `admin` — org settings, user management, routing policy, autonomy mode configuration

**Shipper** — a company that ships goods. A Shipper is a client of a Forwarder. Each Shipper record is owned by one Organization. If two different Forwarders both work with Acme Corp, each has their own Shipper record for Acme — this is correct, because the forwarder's relationship with the shipper (pricing, routing guides, contacts) is specific to that forwarder.

**Carrier** — a shipping line, trucking company, or airline. Carrier master data is shared across tenants (MSC, COSCO, DHL are the same entities regardless of which forwarder is using them). Carrier-specific contract terms (allocations, rates, preferred status) are per-tenant via the TenantCarrier join table.

### 3.2 Core Entity Relationships

```
organizations (tenant)
  ├── users (role: ops_planner | analyst | admin)
  ├── shippers (forwarder's clients, each with routing_guide)
  ├── tenant_carriers (forwarder↔carrier relationship + contract status)
  ├── carrier_allocations (BSA allocation caps per string per period)
  └── shipments
       ├── routes (candidate + committed routing decisions)
       └── bookings (per-leg carrier bookings with status)

carriers (global reference — shared across tenants)
ports (global reference — shared across tenants)
```

### 3.3 Key Table Schemas

```sql
-- Top-level tenant
CREATE TABLE organizations (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                    TEXT NOT NULL,
    slug                    TEXT NOT NULL UNIQUE,        -- used in URLs, stable
    subscription_tier       TEXT,
    autonomy_mode           TEXT DEFAULT 'co_pilot',     -- co_pilot|supervised|autonomous
    confidence_threshold    REAL DEFAULT 0.80,
    dry_run_window_minutes  INT DEFAULT 60,
    is_active               BOOLEAN DEFAULT TRUE,
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Human users within a forwarder org
CREATE TABLE users (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    email                   TEXT NOT NULL UNIQUE,
    full_name               TEXT,
    role                    TEXT NOT NULL,               -- ops_planner|analyst|admin
    auth_provider_user_id   TEXT,                        -- Clerk user ID
    is_active               BOOLEAN DEFAULT TRUE,
    last_login_at           TIMESTAMPTZ,
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Forwarder's shipper clients
CREATE TABLE shippers (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    name                    TEXT NOT NULL,
    country_code            TEXT,
    contact_email           TEXT,
    default_service_level   TEXT DEFAULT 'standard',    -- economy|standard|express
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Global carrier reference (shared across tenants)
CREATE TABLE carriers (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scac_code               TEXT UNIQUE,
    name                    TEXT NOT NULL,
    mode                    TEXT NOT NULL,              -- ocean|air|truck
    is_active               BOOLEAN DEFAULT TRUE
);

-- Forwarder↔carrier relationship with contract status
CREATE TABLE tenant_carriers (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    carrier_id              UUID NOT NULL REFERENCES carriers(id),
    status                  TEXT NOT NULL,              -- preferred|acceptable|blacklisted
    contract_start_date     DATE,
    contract_end_date       DATE,
    UNIQUE(tenant_id, carrier_id)
);

-- Ocean BSA allocation caps per string per period
CREATE TABLE carrier_allocations (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    carrier_id              UUID NOT NULL REFERENCES carriers(id),
    string_code             TEXT NOT NULL,              -- e.g. "MSC Tiger", "CMA AEX-1"
    period_start            DATE NOT NULL,              -- first day of allocation period (monthly)
    period_end              DATE NOT NULL,
    allocated_teu           INT NOT NULL,
    remaining_teu           INT NOT NULL               -- decremented at booking firm-up
);

-- Air ULD contracted capacity per carrier/schedule
-- Analogous to ocean BSA: forwarder pre-commits ULD positions on a carrier schedule
-- at a contracted rate; optimizer fills these before going to spot rate card.
CREATE TABLE air_uld_allocations (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    carrier_id              UUID NOT NULL REFERENCES carriers(id),
    uld_type                TEXT NOT NULL,              -- LD3|LD7|PMC|AKE
    -- LD3: 4.5 m³ / 1,587 kg  LD7: 11.1 m³ / 4,626 kg
    -- PMC: 7.5 m³ / 6,804 kg  AKE: 4.5 m³ / 1,497 kg
    origin_airport          TEXT NOT NULL,              -- IATA airport code
    destination_airport     TEXT NOT NULL,
    departure_days          TEXT[] NOT NULL,            -- e.g. ['MON', 'FRI']
    effective_from          DATE NOT NULL,
    effective_to            DATE NOT NULL,
    ulds_per_departure      INT NOT NULL,               -- how many ULDs contracted per flight
    contracted_rate_per_kg  NUMERIC NOT NULL,           -- all-in contracted rate (includes FSC/SSC)
    remaining_ulds          INT NOT NULL                -- decremented at booking firm-up
);

-- Individual shipment (one per cargo movement request)
CREATE TABLE shipments (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    shipper_id              UUID REFERENCES shippers(id),
    reference_number        TEXT,                       -- forwarder's own reference
    origin_locode           TEXT NOT NULL,
    destination_locode      TEXT NOT NULL,
    cargo_cbm               REAL,
    cargo_kg                REAL,
    cargo_ready_date        DATE,
    delivery_deadline       DATE,
    service_level           TEXT DEFAULT 'standard',
    status                  TEXT DEFAULT 'unrouted',
    -- unrouted|soft_planned|firm_deadline|firm_planned|in_transit|destination_planning|delivered
    -- exception states: escalated|infeasible|cancelled
    ingestion_source        TEXT DEFAULT 'manual',     -- push_api|manual|demand_generator
    external_id             TEXT,                      -- external system's shipment ID (for dedup)
    mode                    TEXT DEFAULT 'ocean_fcl',  -- ocean_fcl|ocean_lcl|air|multimodal
    firm_deadline_at        TIMESTAMPTZ,               -- when soft plan must be firmed (CYC - threshold)
    firmed_at               TIMESTAMPTZ,               -- when route transitioned to firm_planned
    created_at              TIMESTAMPTZ DEFAULT NOW(),
    updated_at              TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (tenant_id, external_id)                    -- dedup push_api ingestion
);

-- Routing decision — one to three candidates (cheapest/fastest/reliable) per shipment
CREATE TABLE routes (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    shipment_id             UUID NOT NULL REFERENCES shipments(id),
    route_type              TEXT,                       -- cheapest|fastest|reliable
    status                  TEXT DEFAULT 'candidate',
    -- candidate|soft|firm
    -- candidate: pre-computed alternative (cheapest/fastest/reliable) not yet selected
    -- soft: agent-selected tentative plan; can be replanned
    -- firm: committed to carrier; this leg is locked
    plan_type               TEXT DEFAULT 'soft',       -- soft|firm (for the selected route)
    total_cost_usd          NUMERIC,
    transit_days_p50        REAL,
    transit_days_sigma      REAL,                       -- path-level sigma from Monte Carlo
    p_on_time               REAL,
    confidence_score        REAL,
    risk_level              TEXT,                       -- low|high
    tier                    SMALLINT,                   -- 1|2|3
    otp_risk_days           REAL,
    legs                    JSONB,                      -- ordered list of legs
    solver_output           JSONB,                      -- full MILP solution
    firm_deadline_at        TIMESTAMPTZ,               -- mirrors shipment.firm_deadline_at; when this route must firm
    firmed_at               TIMESTAMPTZ,               -- when route became firm
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Per-leg carrier booking
CREATE TABLE bookings (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id),
    route_id                UUID NOT NULL REFERENCES routes(id),
    carrier_id              UUID REFERENCES carriers(id),
    booking_reference       TEXT,                       -- carrier-issued reference
    mode                    TEXT NOT NULL,              -- ocean|air|truck
    status                  TEXT DEFAULT 'pending',    -- pending|confirmed|cancelled
    booked_teu              INT,
    vessel_name             TEXT,
    voyage_number           TEXT,
    etd                     TIMESTAMPTZ,
    eta                     TIMESTAMPTZ
);
```

### 3.4 User Roles and Permissions

| Action | ops_planner | analyst | admin |
|---|---|---|---|
| View exception queue | Yes | Yes | Yes |
| Override agent decision | Yes | No | Yes |
| View analytics and reports | Yes | Yes | Yes |
| Configure routing policy | No | No | Yes |
| Manage users | No | No | Yes |
| Configure autonomy mode | No | No | Yes |
| View audit log | Yes | Yes | Yes |
| Generate API keys | No | No | Yes |
| Approve trust graduation | No | No | Yes |

### 3.5 Shipment Lifecycle States

The lifecycle separates **planning states** (tentative, replan-eligible) from **execution states** (committed to carrier, cannot change without penalty). The key design principle: delay firm commitment as long as possible. Keep routes soft until the CYC cutoff forces your hand.

```
unrouted
   │  Agent batch run assigns carrier + sailing (tentative)
   ▼
soft_planned         ← tentative route assigned; agent can replan freely as
   │                    conditions change (new sailings, better consolidation,
   │                    rate shifts). Replanning runs include this shipment.
   │
   │  Rolling Horizon Controller detects CYC cutoff - firm_threshold_days
   ▼
firm_deadline        ← operator alerted; firming required within X days.
   │                    In Autonomous mode: auto-firms after short confirmation
   │                    window. In Supervised/Co-pilot: operator confirms.
   │
   │  Compliance Validator runs final allocation check. Route locked.
   ▼
firm_planned         ← carrier booking placed (manually or via future API).
   │                    Main leg is frozen. Replanning is BLOCKED for this leg.
   │                    Downstream legs (destination trucking) remain soft.
   │
   │  Cargo physically departs origin (first leg underway)
   ▼
in_transit
   │
   │  AIS ETA confidence > threshold (typically 72h from POD arrival)
   ▼
destination_planning ← Destination Leg Planner triggered. POD → final
   │                    destination leg is optimized with precision using
   │                    confirmed ETA. FTL vs. LTL vs. rail decision made here.
   │
   │  Destination leg firm_planned; cargo moves from POD
   ▼
delivered            ← all legs complete; cargo at final destination door
```

**Planning state rules:**
- `unrouted` and `soft_planned` — replanning runs freely include these shipments
- `firm_deadline` — operator alerted; last window to adjust before auto-firm (Autonomous mode) or manual confirm (Co-pilot/Supervised)
- `firm_planned` — no replanning by agent; operator override requires explicit reason; triggers Compliance Validator

**Invariant:** Every shipment always has a valid plan (soft or firm). A shipment never becomes plan-less. The system never silently migrates a soft plan when a CYC expires — all plan changes are deliberate decisions.

**Soft plan = one specific tentative sailing, held deliberately.**

If a shipment is soft-planned to CYC1, the plan is to execute CYC1. There are exactly two ways a soft plan changes:

1. **Active replanning** — the optimizer re-runs (scheduled batch, accumulation trigger, or manual) and produces a better plan on a different sailing. The agent updates the soft plan deliberately. This may happen any number of times while the shipment is soft.
2. **Firm up** — the plan is committed to the current soft-planned sailing. The route becomes `firm_planned`.

If neither happens and the soft-planned sailing's CYC expires, that is an **infeasibility escalation** — the plan became unexecutable because no action was taken. The Rolling Horizon Controller's job is to prevent this by alerting well before it can happen.

**Rolling Horizon Controller behavior (corrected):**
- Monitors each soft-planned shipment's current sailing CYC
- Fires alert when CYC is within the configured threshold → operator or agent must act (firm or actively replan)
- Does NOT automatically migrate the soft plan when CYC passes — migration is always an active optimizer decision
- The infeasible-at-first-routing case (no feasible sailing at T=0) escalates immediately and requires manual handling; it does not recur from CYC expiry if the system is alerting correctly

**Firm-up triggers (per-tenant configurable):**

| Trigger | Default | Configurable |
|---|---|---|
| Hours before current soft-plan's CYC | Alert at 24h | Yes — early-firming tenants: 72h; late-optimizing: 8h |
| Operator explicit action | Manual firm at any time | Always |
| Auto-firm (Autonomous mode) | Auto-firms at CYC alert threshold if no replan issued | Autonomous mode only |

**Scheduled replanning and plan migration:**

The scheduled batch replanning runs (daily or accumulation-triggered) re-run the optimizer over all soft-planned shipments. If a better sailing exists — due to new shipments on the same lane enabling better consolidation, rate shifts, or schedule changes — the optimizer returns a new soft plan. The agent updates the plan. This is deliberate and logged. It is not automatic expiry migration.

Service level does NOT govern firm-up timing. It governs which sailings are feasible (deadline constraint) and tier assignment (risk classification), but not when commitment is required.

**Per-tenant configuration profiles:**
- **Early firmer:** Short CYC alert threshold (72h+). Prioritizes carrier relationship and operational certainty. Often firms immediately after the first batch planning run.
- **Late optimizer:** Long delay before alert (8–12h before CYC). Accumulates shipments, lets the agent replan frequently. May find better consolidation or lower rates. Accepts more uncertainty in exchange for optimization opportunity.

**Exception states** (can occur at any point prior to delivered):
- `escalated` — Tier 3; in human exception queue awaiting decision
- `infeasible` — no feasible route found; in exception queue with earliest achievable date
- `cancelled` — cancelled by operator

### 3.6 Ingestion Modes

Shipments enter the platform through three channels:

| Mode | `ingestion_source` | Description |
|---|---|---|
| Push API | `push_api` | External TMS/ERP/WMS posts shipment via REST. Auth: API key (per-tenant). Idempotent on `external_id`. Validates schema; returns structured error on validation failure. |
| Manual UI | `manual` | Forwarder ops enters via UI form or CSV bulk upload. |
| Demand Generator | `demand_generator` | Dev/test only. Generates synthetic but realistic shipment batches on a schedule per `demand_generator_configs`. Not exposed in production. |

All three modes insert into the same `shipments` table with `status = unrouted`. The routing agent treats them identically.

## 4. Policy Rules and Snapshots

All editable, versionable, auditable policies — carrier rules, embargo rules, lithium acceptance, ULD interchange agreements, service-product catalog, and future rule types — share one generic data-model pattern. Three tables; one row per rule edit (append-only); one snapshot per policy type per solve run; bind snapshots to routing runs for full reproducibility.

### 4.1 Tables

```sql
-- Append-only rule history. Tenant-scoped, RLS-enforced. One row per rule version.
CREATE TABLE policy_rules (
  rule_id         UUID PRIMARY KEY,
  tenant_id       UUID NOT NULL,
  policy_type     TEXT NOT NULL,               -- 'carrier' | 'embargo' | 'lithium' | 'uld_interchange' | 'service_product' | ...
  layer           TEXT NOT NULL,               -- type-specific layer (e.g., 'tenant_blacklist', 'shipper_lane')
  scope           JSONB NOT NULL,              -- type-specific scope filter (e.g., {shipper_id, origin, destination})
  action          JSONB NOT NULL,              -- type-specific action payload (e.g., {deny, allow, prefer})
  effective_from  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  effective_to    TIMESTAMPTZ,                 -- NULL = open-ended; set on supersession or soft-delete
  status          TEXT NOT NULL,               -- 'active' | 'superseded' | 'soft_deleted'
  supersedes      UUID REFERENCES policy_rules(rule_id),
  version         INT NOT NULL,                -- monotonic per (tenant_id, policy_type, scope-hash)
  metadata        JSONB,                       -- {reason, ticket_ref, expires_review_at, ...}
  created_by      UUID NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ix_policy_rules_active
  ON policy_rules (tenant_id, policy_type, status, effective_from, effective_to);

-- One snapshot per (tenant, policy_type) per pre-solve. Dedupe by rule_checksum.
CREATE TABLE policy_snapshots (
  snapshot_id     UUID PRIMARY KEY,
  tenant_id       UUID NOT NULL,
  policy_type     TEXT NOT NULL,
  rule_ids        UUID[] NOT NULL,             -- active rule_ids at snapshot time, sorted
  rule_checksum   TEXT NOT NULL,               -- sha256(sorted rule_ids) for dedupe
  trigger         TEXT NOT NULL,               -- 'pre_solve' | 'scheduled' | 'manual' | 'rule_edit'
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, policy_type, rule_checksum)
);

-- A routing run binds to one snapshot per policy_type it consulted.
CREATE TABLE routing_run_policy_bindings (
  run_id          UUID NOT NULL REFERENCES routing_run(run_id),
  policy_type     TEXT NOT NULL,
  snapshot_id     UUID NOT NULL REFERENCES policy_snapshots(snapshot_id),
  PRIMARY KEY (run_id, policy_type)
);
```

### 4.2 Per-type instantiation

Each policy type defines its `layer` enum, `scope` schema, `action` schema, and how its rules resolver materializes the resolved output consumed by the optimizer.

| `policy_type` | Layer enum | Scope shape | Action shape | Resolved output |
|---|---|---|---|---|
| `carrier` | `tenant_blacklist`, `shipper_lane`, `service_product`, `lane_preference`, `commodity_overlay` | `{shipper_id?, origin?, destination?, commodity_class?, service_product_id?}` | `{deny: [...], allow: [...] \| "ANY", prefer: [...]}` | Per shipment: $(C^{\text{allow}}, C^{\text{deny}}, C^{\text{pref}})$ |
| `embargo` | `tenant`, `regulatory`, `carrier_imposed` | `{carrier?, origin?, destination?, hub?, ac_type?, cargo_type?, commodity?, lithium_pi?}` with date range | `{hard_deny: true}` (embargoes are always hard) | Per flight: active embargo set $E_f$ |
| `lithium` | `tenant`, `regulatory`, `carrier_policy` | `{carrier?, ac_type?, pi_code?, section?}` | `{accept: bool, conditions?: {soc_required, ddr_excluded}}` | Per (carrier, ac_type, PI, section): acceptance matrix |
| `uld_interchange` | `alliance`, `bilateral` | `{carrier_pair: [c1, c2], uld_types: [...]}` | `{interchange_ok: bool}` | Interchange set $\Pi$ |
| `service_product` | `catalog` | `{product_id}` | Full bundle: `{name, mode_allow, carrier_allow, carrier_deny, ac_type_allow, T_SLA, handling_tier, cargo_type_min}` | Catalog $P$; resolved on `sp(k)` lookup |

Type-specific JSON Schemas validate the `scope` and `action` payloads at insert time; rules engine enforces resolved-output invariants (e.g., "deny wins over allow" for carrier).

### 4.3 Lifecycle semantics

- **Edit a rule** → insert new row with `supersedes = old.rule_id`; update old row to `status = 'superseded'`, `effective_to = NOW()`. Both rows preserved.
- **Soft delete (emergency removal)** → update target row to `status = 'soft_deleted'`, `effective_to = NOW()`. Row retained for audit; resolver excludes from active set immediately.
- **Active-rule query** at time T:
  ```sql
  SELECT * FROM policy_rules
  WHERE tenant_id = $1 AND policy_type = $2 AND status = 'active'
    AND effective_from <= $3 AND (effective_to IS NULL OR effective_to > $3)
  ```
- **Bulk operations** (e.g., toggle 5 carrier rules at once) → single transaction; one snapshot generated on next solve.

### 4.4 Snapshot and per-run binding

Pre-solve flow per policy type:
1. Resolver computes active rule_id list for `(tenant_id, policy_type)` at current time.
2. Compute `rule_checksum = sha256(sorted_rule_ids)`.
3. Look up `policy_snapshots` by `(tenant_id, policy_type, rule_checksum)`; insert if missing (UNIQUE constraint handles concurrent inserts).
4. Bind `snapshot_id` to the routing run via `routing_run_policy_bindings`.

A routing run typically binds to one snapshot per policy type it consulted: `(carrier, embargo, lithium, uld_interchange, service_product)` → 5 bindings.

**Reproducibility query** (replay a past run's policy state):
```sql
SELECT pr.* FROM routing_run_policy_bindings rb
  JOIN policy_snapshots ps ON ps.snapshot_id = rb.snapshot_id
  JOIN UNNEST(ps.rule_ids) AS rid ON true
  JOIN policy_rules pr ON pr.rule_id = rid
WHERE rb.run_id = $1
ORDER BY rb.policy_type, pr.layer;
```

**Snapshot immutability:** once created, snapshots are read-only. Rule edits between snapshot creation and solve completion do not affect the in-flight solve. New edits create a new snapshot at the next solve.

### 4.5 UI and operational semantics

- **Policy Editor screen** (per `ui_spec.md`, P0): rules grouped by `(policy_type, layer)`; per-row edit, history view (chase `supersedes` chain), soft-delete with reason. Role-gated: edit requires `policy_admin` role.
- **Emergency override**: "Disable all rules in layer X" → bulk soft-delete in single transaction. Audit-logged. Next solve uses reduced ruleset.
- **Audit trail**: full `policy_rules` history. Per-run binding lets you answer "which rules governed run R, who authored them, when, why?"
- **Review reminders**: `metadata.expires_review_at` surfaces in operator dashboard when due — prevents stale rules from accumulating.

### 4.6 Worked example — carrier policy

Three rule rows for tenant `acme-fwd`:

```json
[
  {
    "rule_id": "rule-tnt-blk-001",
    "tenant_id": "acme-fwd",
    "policy_type": "carrier",
    "layer": "tenant_blacklist",
    "scope": {},
    "action": { "deny": ["KE"], "allow": "ANY", "prefer": [] },
    "effective_from": "2026-05-01T00:00:00Z",
    "effective_to": null,
    "status": "active",
    "version": 1,
    "metadata": {
      "reason": "Payment dispute Q2 2026",
      "ticket_ref": "FWD-1234",
      "expires_review_at": "2026-08-01T00:00:00Z"
    },
    "created_by": "ops-user-23",
    "created_at": "2026-05-01T09:14:00Z"
  },
  {
    "rule_id": "rule-shp-ln-042",
    "tenant_id": "acme-fwd",
    "policy_type": "carrier",
    "layer": "shipper_lane",
    "scope": { "shipper_id": "beta-corp", "origin": "TPE", "destination": "JFK" },
    "action": { "deny": ["AA"], "allow": "ANY", "prefer": ["CX", "BR"] },
    "effective_from": "2026-04-01T00:00:00Z",
    "effective_to": null,
    "status": "active",
    "version": 2,
    "supersedes": "rule-shp-ln-041",
    "metadata": { "reason": "Beta Corp Q2 contract amendment" },
    "created_by": "ops-user-15",
    "created_at": "2026-04-01T11:22:00Z"
  },
  {
    "rule_id": "rule-ln-pref-007",
    "tenant_id": "acme-fwd",
    "policy_type": "carrier",
    "layer": "lane_preference",
    "scope": { "origin": "TPE", "destination": "JFK" },
    "action": { "deny": [], "allow": "ANY", "prefer": ["CV"] },
    "effective_from": "2026-03-15T00:00:00Z",
    "effective_to": null,
    "status": "active",
    "version": 1,
    "metadata": { "reason": "CV contract rate favorable on TPE-JFK Q2-Q4 2026" },
    "created_by": "ops-user-23",
    "created_at": "2026-03-15T16:00:00Z"
  }
]
```

For shipment $k$ from Beta Corp on TPE→JFK, service product `PRM_AIR_EXP`, the rules engine resolves (per air-freight model §6.15, deny-wins):
- $\mathcal{D}_k = \{\text{KE, AA}\}$
- $\mathcal{A}_k = \{\text{CX, BR, CV, LH, EK, KE}\}$ (from service product; tenant_blacklist has allow="ANY")
- $C_k^{\text{allow}} = \mathcal{A}_k \setminus \mathcal{D}_k = \{\text{CX, BR, CV, LH, EK}\}$
- $C_k^{\text{deny}} = \{\text{KE, AA}\}$
- $C_k^{\text{pref}} = \{\text{CX, BR, CV}\} \cap C_k^{\text{allow}} = \{\text{CX, BR, CV}\}$

The snapshot captures `rule_ids = [rule-tnt-blk-001, rule-shp-ln-042, rule-ln-pref-007]`; the routing run binds to that snapshot via `routing_run_policy_bindings(run_id, 'carrier', snapshot_id)`. Replay six months later: same resolved triple, regardless of subsequent edits.

### 4.7 Out of MVP scope

- **Time-windowed rule activation** (peak-season toggles, off-hours rules) — model via `metadata.schedule` JSONB; resolver consults at solve time. Deferred to rules_engine.tex.
- **Conditional rules** ("prefer freighter when commodity is electronics AND value > $50k") — same `metadata.condition` pattern; deferred.
- **ML-learned preference weights** from operator override history (`logs/overrides.jsonl`) — Phase 5 constraint learning; not a JSONB policy but feeds into preference resolution.
- **Cross-policy dependencies** (e.g., lithium rule references carrier rule) — out of MVP; each policy type is independent.

## 5. Spot Rate Snapshots

Spot/free-sale rates are time-varying market data. Captured periodically (typical: hourly for air, daily for ocean) into immutable snapshots; each routing run binds to the snapshot it consulted for reproducibility, the same pattern as Policy Rules in §4.

### 5.1 What is and is not stored

**Stored:**
- Rates per (carrier, lane or flight, ULD/container type, rate type, weight breaks)
- Capture metadata: source, captured_at, tenant scope
- Per-quote validity (when source provides one — live API quotes carry it; published indices typically do not)

**Not stored (out of MVP scope):**
- Reconciliation between snapshot rate and realized booking rate — derivable on demand from `JOIN` of snapshot rate vs. shipment booking record
- Synthetic / fallback / interpolated rates — if no rate exists for an arc at solve time, the arc is excluded from the subgraph (no rate ⇒ no option)
- Forecasted future rates (machine-learned or extrapolated) — deferred

### 5.2 Tables

```sql
CREATE TABLE spot_rate_snapshots (
  snapshot_id          UUID PRIMARY KEY,
  tenant_id            UUID,                    -- NULL = shared baseline (e.g., TACT, public market index); non-null = forwarder-specific (negotiated/net rate API result)
  mode                 TEXT NOT NULL,           -- 'air' | 'ocean' | 'ground'
  source               TEXT NOT NULL,           -- 'webcargo' | 'cargoai' | 'carrier_api:CX' | 'manual' | 'tact_baseline' | 'xeneta' | ...
  captured_at          TIMESTAMPTZ NOT NULL,
  freshness_threshold  INTERVAL NOT NULL DEFAULT '24 hours',  -- max age before solver treats snapshot as stale
  notes                JSONB,                   -- e.g., {api_request_id, ingestion_job_id, schema_version}
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ix_spot_rate_snapshots_recent
  ON spot_rate_snapshots (tenant_id, mode, source, captured_at DESC);

CREATE TABLE spot_rate_quotes (
  quote_id             UUID PRIMARY KEY,
  snapshot_id          UUID NOT NULL REFERENCES spot_rate_snapshots(snapshot_id),
  carrier              TEXT NOT NULL,
  flight_id            TEXT,                    -- non-null for air (specific flight); null for ocean (use lane + service_string)
  lane_origin          TEXT NOT NULL,
  lane_destination     TEXT NOT NULL,
  service_string       TEXT,                    -- ocean: which sailing/service
  uld_type             TEXT,                    -- air: ULD type (PMC, LD3, ...); ocean/ground: NULL or container type
  rate_type            TEXT NOT NULL,           -- 'GCR' | 'SCR' | 'BUC' | 'CR' | 'NEGOTIATED' | 'TACT_BASELINE' | 'OCEAN_FAK' | 'OCEAN_SPOT' | ...
  applies_to           JSONB,                   -- {commodity_codes?, uld_types?, min_qty?, max_qty?, weight_break_min?, weight_break_max?}
  weight_breaks        JSONB,                   -- list of {min_weight, rate_per_kg, flat_rate?} for IATA tiered pricing
  currency             TEXT NOT NULL,
  surcharges_included  BOOLEAN NOT NULL,        -- if false, base rate only; surcharges from separate parameter set
  valid_until          TIMESTAMPTZ,             -- NULL = published baseline (no per-quote expiry); non-null = live API quote with carrier-defined expiry
  raw_response         JSONB                    -- raw payload from source for audit
);

CREATE INDEX ix_spot_rate_quotes_lookup
  ON spot_rate_quotes (snapshot_id, carrier, lane_origin, lane_destination);

CREATE TABLE routing_run_rate_bindings (
  run_id         UUID NOT NULL REFERENCES routing_run(run_id),
  mode           TEXT NOT NULL,
  snapshot_id    UUID NOT NULL REFERENCES spot_rate_snapshots(snapshot_id),
  PRIMARY KEY (run_id, mode)
);
```

### 5.3 Tenant scope and RLS

`tenant_id` is nullable to distinguish two rate classes:

| `tenant_id` | Rate class | Examples |
|---|---|---|
| NULL | Shared baseline | TACT published rates, public market indices |
| Non-null | Forwarder-specific | Negotiated rates via authenticated API (WebCargo, CargoAi, direct carrier portals), confidential IATA Net Rates |

RLS policy on `spot_rate_snapshots` and `spot_rate_quotes`:
```sql
USING (tenant_id IS NULL OR tenant_id = current_setting('app.current_tenant_id')::uuid)
```
A tenant sees their own forwarder-specific rates plus all shared baselines. A forwarder's negotiated rates never leak across tenants.

Source: IATA documents Net Rates as bilaterally agreed and confidential between airline and forwarder ([IATA — Air Cargo Tariffs and Rules](https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/air-cargo-tariffs-and-rules-what-you-need-to-know/)).

### 5.4 Validity semantics

**At solve time, a quote is *usable* if:**
1. `NOW() − snapshot.captured_at ≤ snapshot.freshness_threshold` (our staleness rule)
2. `quote.valid_until IS NULL OR NOW() ≤ quote.valid_until` (carrier's expiry, when provided)

Both checks must pass. Quotes that fail either check are dropped from the candidate set; if no quote remains for a (carrier, flight or sailing, ULD type) tuple, the corresponding arc is excluded from the subgraph (per §5.1 "no rate ⇒ no option").

**Typical `valid_until` values by source (when present):**
- WebCargo / CargoAi live air quotes — typically 48–72h pre-flight, sometimes shorter ([FreightAmigo — Understanding Freight Quote Validity](https://www.freightamigo.com/en/blog/logistics/understanding-freight-quote-validity-and-re-quoting-ensuring-accurate-pricing-for-your-shipments/))
- Direct carrier portal quotes — varies by carrier; often quote-session-bound
- Published rate cards / TACT / market indices — `valid_until` is NULL; refreshed on the source's update cadence (daily or weekly typically), governed by `freshness_threshold` not `valid_until`

### 5.5 Snapshot capture and dedup

Capture jobs run on a schedule per (tenant, mode, source):
- Air via WebCargo / CargoAi: hourly during business hours, less frequent overnight
- Ocean via Xeneta or carrier APIs: daily
- TACT baseline: weekly (slower-changing reference)

Each capture inserts a new `spot_rate_snapshots` row. **No dedup by content hash** (unlike `policy_snapshots`) because rate data churns frequently — even identical-looking snapshots represent distinct moments and the solver may care about which exact data lineage was used.

Retention: keep snapshots for replay window (default: 90 days), then archive or summarize.

### 5.6 Per-run binding and reproducibility

Each routing run binds to one snapshot per mode consulted (one for air, one for ocean, etc.) via `routing_run_rate_bindings`. Pre-solve flow per mode:
1. Resolver picks the most recent snapshot for `(tenant_id, mode)` where `captured_at ≤ NOW()` and `NOW() − captured_at ≤ freshness_threshold`.
2. If no fresh snapshot exists for that mode, the resolver returns an empty rate set; downstream subgraph construction excludes all arcs of that mode.
3. The selected `snapshot_id` is written to `routing_run_rate_bindings`.

**Replay query:**
```sql
SELECT q.*
  FROM routing_run_rate_bindings b
  JOIN spot_rate_quotes q ON q.snapshot_id = b.snapshot_id
WHERE b.run_id = $1
ORDER BY b.mode, q.carrier, q.lane_origin, q.lane_destination;
```

### 5.7 Worked example — air spot snapshot for one flight

Snapshot captured 2026-05-17 14:00 UTC, tenant `acme-fwd`, source `webcargo`:

```json
{
  "snapshot_id": "snap-2026-05-17-14",
  "tenant_id": "acme-fwd",
  "mode": "air",
  "source": "webcargo",
  "captured_at": "2026-05-17T14:00:00Z",
  "freshness_threshold": "PT24H"
}
```

Three rate rows for flight CX880 TPE-HKG-JFK, PMC, departing 2026-05-20:

```json
[
  {
    "carrier": "CX",
    "flight_id": "CX880-20260520",
    "lane_origin": "TPE",
    "lane_destination": "JFK",
    "uld_type": "PMC",
    "rate_type": "GCR",
    "applies_to": { "commodity_codes": ["*"] },
    "weight_breaks": [
      { "min_weight": 0,    "rate_per_kg": 5.20 },
      { "min_weight": 45,   "rate_per_kg": 4.40 },
      { "min_weight": 100,  "rate_per_kg": 3.80 },
      { "min_weight": 300,  "rate_per_kg": 3.20 },
      { "min_weight": 500,  "rate_per_kg": 2.95 },
      { "min_weight": 1000, "rate_per_kg": 2.65 }
    ],
    "currency": "USD",
    "surcharges_included": false,
    "valid_until": "2026-05-19T14:00:00Z"
  },
  {
    "carrier": "CX",
    "flight_id": "CX880-20260520",
    "lane_origin": "TPE",
    "lane_destination": "JFK",
    "uld_type": "PMC",
    "rate_type": "SCR",
    "applies_to": { "commodity_codes": ["0002", "0003"] },   // SCR for electronics commodity codes
    "weight_breaks": [
      { "min_weight": 100, "rate_per_kg": 3.20 },
      { "min_weight": 500, "rate_per_kg": 2.50 }
    ],
    "currency": "USD",
    "surcharges_included": false,
    "valid_until": "2026-05-19T14:00:00Z"
  },
  {
    "carrier": "CX",
    "flight_id": "CX880-20260520",
    "lane_origin": "TPE",
    "lane_destination": "JFK",
    "uld_type": "PMC",
    "rate_type": "BUC",
    "applies_to": { "uld_types": ["PMC"] },
    "weight_breaks": [
      { "min_weight": 0, "flat_rate": 9500.00 }
    ],
    "currency": "USD",
    "surcharges_included": false,
    "valid_until": "2026-05-19T14:00:00Z"
  }
]
```

The optimizer picks the cheapest applicable rate type per shipment: electronics shipment uses the SCR row; non-electronics evaluates GCR weight-tiered vs. BUC flat; cheaper of the two binds. All three rates expire 2026-05-19 14:00 UTC (carrier-defined, 48h validity); a solve at 2026-05-19 13:00 honors them; a solve at 15:00 drops these quotes and either uses a newer snapshot or excludes this flight from A_k.

### 5.8 Out of MVP scope

- **Rate forecasting** — extrapolating future rates from historical snapshots for planning shipments beyond live-quote horizon. P1.
- **Cross-source rate reconciliation** — when WebCargo and direct CX API disagree on a CX rate, which wins? MVP: use the highest-priority source per (mode, carrier) — priority table is a tenant config. P1: confidence-weighted reconciliation.
- **Bid-ask spread modeling** — some sources quote "indicative" rates that diverge from actual booking rates by predictable margins. Track and adjust. P1.
- **Soft BSA reclaim → spot capacity flow** — when a soft BSA goes unused and carrier reclaims for free-sale (covered briefly in `air_freight_routing.tex` §11 deferred), the released capacity should surface in the next spot snapshot. Snapshot ingestion needs to read updated capacity from the source; no model change required, just an ingestion-pipeline awareness.

## 6. Surcharge Catalog

The routing optimizer's freight-cost terms (base rate × chargeable weight, BSA pivot floor, spot weight-break rate) capture *only the base air-freight rate*. Real quotes layer 10–20 additional surcharges per shipment per arc: fuel, security, terminal handling, customs filing, AWB issuance, screening, DGR handling, war risk, peak-season, perishable handling, valuables, overweight, lithium handling. Without modeling these, the optimizer's cost is systematically understated by hundreds to low-thousands USD per shipment.

The Surcharge Catalog is the storage and resolution layer for these. The MILP's `air_freight_routing.tex` §6.7 consumes the resolved per-arc surcharge stack at solve time.

### 6.1 Table

Single table, append-only, versioned via `effective_from` / `effective_to`. Snapshot binding follows the same per-run pattern as §4 (Policy Rules) and §5 (Spot Rate Snapshots).

```sql
CREATE TABLE surcharge_catalog (
  surcharge_id     UUID PRIMARY KEY,
  tenant_id        UUID,                    -- NULL = shared baseline (e.g., IATA-published industry-wide); non-null = tenant-specific contract or override
  surcharge_type   TEXT NOT NULL,           -- enumerated; see §6.2
  scope            JSONB NOT NULL,          -- applicability filter; see §6.3
  calculation      JSONB NOT NULL,          -- basis, rate, currency, floors/ceilings; see §6.4
  effective_from   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  effective_to     TIMESTAMPTZ,             -- NULL = open-ended
  status           TEXT NOT NULL,           -- 'active' | 'superseded' | 'soft_deleted'
  supersedes       UUID REFERENCES surcharge_catalog(surcharge_id),
  version          INT NOT NULL,
  metadata         JSONB,                   -- {reason, source_doc_url, ticket_ref, expires_review_at}
  created_by       UUID NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ix_surcharge_active
  ON surcharge_catalog (tenant_id, surcharge_type, status, effective_from, effective_to);

CREATE TABLE surcharge_snapshots (
  snapshot_id      UUID PRIMARY KEY,
  tenant_id        UUID,
  surcharge_ids    UUID[] NOT NULL,
  rule_checksum    TEXT NOT NULL,
  trigger          TEXT NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, rule_checksum)
);

ALTER TABLE routing_run_policy_bindings
  ADD COLUMN surcharge_snapshot_id UUID REFERENCES surcharge_snapshots(snapshot_id);
-- or use a separate bindings table; keeping it adjacent to other policy bindings
```

### 6.2 Surcharge type enumeration

| `surcharge_type` | What it is | Typical basis | Who charges |
|---|---|---|---|
| `FSC` | Fuel surcharge | per_kg | Airline; refreshed monthly |
| `SSC` | Security surcharge | per_kg | Airline / regulator |
| `THC_O` | Origin terminal handling | per_shipment or per_kg | Airline / ground handler |
| `THC_D` | Destination terminal handling | per_shipment or per_kg | Airline / ground handler |
| `AMS` | US Automated Manifest System filing | per_shipment | CBP-mandated; broker / carrier |
| `ICS2` | EU pre-load entry summary | per_shipment | EU-mandated; carrier |
| `ACI` | Canada Advance Cargo Information | per_shipment | CBSA-mandated; carrier |
| `AWB_FEE` | AWB issuance / processing | per_awb | Airline |
| `SCREENING` | Mandatory screening / X-ray | per_kg or per_shipment | CCSF / RA3 facility |
| `DGR_HANDLING` | Dangerous goods special handling | per_shipment | Airline / CFS |
| `WAR_RISK` | War-risk premium (specific origins/destinations) | percentage_of_freight or per_kg | Airline |
| `PSS` | Peak-season surcharge | per_kg | Airline (seasonal) |
| `GRI` | General rate increase (seasonal adjustment) | per_kg | Airline |
| `COOL_CHAIN` | Perishable / pharma cold-chain handling | per_shipment or per_kg | Airline / CFS |
| `AVI` | Live animal handling | per_shipment | Airline (per IATA LAR) |
| `VAL` | Valuables handling (escort, vaulted CFS) | per_shipment or percentage | Airline / CFS |
| `OVERWEIGHT` | Heavy piece surcharge | per_piece or per_shipment | Airline |
| `LITHIUM_HANDLING` | Lithium battery handling (per PI / per section) | per_shipment | Airline / CFS |

Tenant-specific custom types are allowed (e.g., `TENANT_FEE_X`) but should be rare and documented in `metadata.reason`.

### 6.3 Scope schema

The `scope` JSONB describes which (shipment, flight, arc) combinations a surcharge applies to. All fields are optional; absent fields mean "applies to all":

```json
{
  "carrier": "CX" | ["CX", "BR"] | null,
  "origin_country": "TW" | null,
  "origin_iata": "TPE" | null,
  "destination_country": "US" | null,
  "destination_iata": "JFK" | null,
  "region": "TPEB" | "FEWB" | "TAWB" | null,
  "ac_type": "FREIGHTER" | "PAX_BELLY" | null,
  "cargo_type": "DGR" | "PER" | "VAL" | "AVI" | "GEN" | null,
  "lithium_pi": "PI965" | null,
  "service_product_id": "PRM_AIR_EXP" | null,
  "min_chargeable_weight_kg": 500 | null,
  "applies_on_arc_types": ["air"] | ["ground_origin", "ground_destination"] | null,
  "date_range": { "start": "2026-06-01", "end": "2026-09-30" } | null
}
```

A surcharge applies when **all** specified scope fields match the (shipment, flight, arc) tuple. Multiple surcharges of the same `surcharge_type` may apply if scopes don't conflict (e.g., a tenant-specific FSC adjustment layered on top of an airline base FSC — though most cases use supersession instead of stacking).

### 6.4 Calculation schema

The `calculation` JSONB defines how the surcharge amount is computed once it applies:

```json
{
  "basis": "per_kg" | "per_shipment" | "per_awb" | "per_uld" | "per_cbm" | "per_piece" | "percent_of_freight",
  "rate": 0.42,
  "currency": "USD",
  "min_charge": 25.00 | null,
  "max_charge": 500.00 | null,
  "weight_basis": "chargeable" | "actual" | null,    // for per_kg only
  "percent_of_field": "base_freight" | "total_freight_incl_other_surcharges" | null   // for percent_of_freight only
}
```

Calculation by basis:

| basis | Amount formula |
|---|---|
| `per_kg` | `rate × cw_k` (or `w_k` if `weight_basis = "actual"`) |
| `per_shipment` | `rate` (flat, applied once per shipment per matching arc) |
| `per_awb` | `rate` (flat, per AWB; for forwarders with multiple AWBs per shipment, applies per AWB) |
| `per_uld` | **Flight-level cost**, not per-shipment-per-arc: `rate × Σ_{u,c} z_{f,u}^c` charged once per flight regardless of which shipments share the ULDs. The forwarder pays the airline this fee once per loaded ULD; not allocated to individual shipments. Enters the objective via a separate flight-level term, not via the per-arc `surcharge_cost(k, i, j) · x_{ij}^k` machinery. |
| `per_cbm` | `rate × v_k` |
| `per_piece` | `rate × piece_count_k` |
| `percent_of_freight` | `(rate / 100) × base_freight_k` (or `total_freight_k` per `percent_of_field`) |

`min_charge` and `max_charge` (in `currency`) clip the per-arc amount: `clip(amount, min_charge, max_charge)`. Min/max apply at the per-arc level, not per-snapshot or per-shipment.

### 6.5 Resolution — two cost paths

Surcharges split into two paths based on their calculation basis:

**(A) Per-shipment-per-arc surcharges** — bases `per_kg`, `per_shipment`, `per_awb`, `per_cbm`, `per_piece`, and `percent_of_freight` (when `percent_of_field = base_freight`). Pre-solve, for each (shipment $k$, arc $(i,j) \in A_k$), the resolver computes:
```
S_arc(k, i, j) = { s ∈ surcharge_catalog | scope_match(s, k, i, j) AND s.basis ∈ {per_kg, per_shipment, per_awb, per_cbm, per_piece, percent_of_freight(base)} AND s.status = 'active' AND now ∈ [effective_from, effective_to) }
```
Per-arc surcharge amount:
```
surcharge_cost(k, i, j) = Σ_{s ∈ S_arc(k, i, j)} amount(s, k, i, j)
```
The amount is a constant per (k, arc) after pre-resolution. The objective adds:
```
+ Σ_k Σ_{(i,j) ∈ A_k} surcharge_cost(k, i, j) × x_{ij}^k
```

**(B) Flight-level surcharges** — basis `per_uld`. These are paid once per flight per loaded ULD regardless of which shipments share the ULDs. Pre-solve, for each flight $f$, the resolver computes:
```
S_flight(f) = { s ∈ surcharge_catalog | scope_match_flight(s, f) AND s.basis = 'per_uld' AND s.status = 'active' AND now ∈ [effective_from, effective_to) }
```
Flight-level per-ULD surcharge amount (linear in $z$, independent of $x$):
```
flight_uld_surcharge_cost(f) = Σ_{s ∈ S_flight(f)} s.rate × Σ_{u, c} z_{f,u}^c
```
The objective adds a separate flight-level term:
```
+ Σ_f flight_uld_surcharge_cost(f)
```
This term is linear in the integer variable $z_{f,u}^c$ — no bilinearity, no per-shipment attribution. Operationally: terminal-handling-per-ULD, airline-charged ULD build-up fees, and similar costs depend on how many ULDs were loaded, not which shipments contributed to them.

**Why this split:** treating `per_uld` as per-shipment-per-arc was a bug — multiplying `rate × Σ z` by `x_{ij}^k` would either over-charge (every shipment on the flight pays the full per-ULD fee, n times for n shipments) or introduce a bilinear `x · z` term requiring McCormick linearization. The flight-level treatment is correct accounting and trivially linear.

### 6.6 Worked example — Premium Air shipment TPE-JFK

Shipment $k$: 1{,}200 kg chargeable weight, GEN cargo, service product `PRM_AIR_EXP`, routing TPE → HKG (CX564) → JFK (CX880). Tenant `acme-fwd`. Snapshot at solve time materializes:

```json
[
  {
    "surcharge_type": "FSC",
    "scope": { "carrier": "CX", "ac_type": "FREIGHTER" },
    "calculation": { "basis": "per_kg", "rate": 0.42, "currency": "USD" },
    "applies_to_arcs": ["CX564", "CX880"]
  },
  {
    "surcharge_type": "SSC",
    "scope": { "carrier": "CX" },
    "calculation": { "basis": "per_kg", "rate": 0.18, "currency": "USD" }
  },
  {
    "surcharge_type": "THC_O",
    "scope": { "origin_iata": "TPE" },
    "calculation": { "basis": "per_shipment", "rate": 65.00, "currency": "USD", "min_charge": 25.00 }
  },
  {
    "surcharge_type": "THC_D",
    "scope": { "destination_iata": "JFK" },
    "calculation": { "basis": "per_shipment", "rate": 90.00, "currency": "USD" }
  },
  {
    "surcharge_type": "AMS",
    "scope": { "destination_country": "US" },
    "calculation": { "basis": "per_shipment", "rate": 25.00, "currency": "USD" }
  },
  {
    "surcharge_type": "AWB_FEE",
    "scope": { "carrier": "CX" },
    "calculation": { "basis": "per_awb", "rate": 35.00, "currency": "USD" }
  },
  {
    "surcharge_type": "SCREENING",
    "scope": { "destination_country": "US", "ac_type": "FREIGHTER" },
    "calculation": { "basis": "per_kg", "rate": 0.06, "currency": "USD", "min_charge": 35.00 }
  },
  {
    "surcharge_type": "PSS",
    "scope": { "region": "TPEB", "date_range": { "start": "2026-08-01", "end": "2026-10-15" } },
    "calculation": { "basis": "per_kg", "rate": 0.50, "currency": "USD" }
  }
]
```

Per-arc breakdown for solve on 2026-05-17 (outside PSS date range — PSS doesn't apply):

| Surcharge | Arc TPE→HKG (CX564) | Arc HKG→JFK (CX880) | Total |
|---|---|---|---|
| FSC | 1200 × 0.42 = 504 | 1200 × 0.42 = 504 | 1{,}008 |
| SSC | 1200 × 0.18 = 216 | 1200 × 0.18 = 216 | 432 |
| THC_O | 65 (only on origin arc) | --- | 65 |
| THC_D | --- | 90 (only on destination arc) | 90 |
| AMS | --- | 25 (US destination flight) | 25 |
| AWB_FEE | 35 (issued once at origin) | --- | 35 |
| SCREENING | --- | max(0.06 × 1200, 35) = 72 | 72 |
| PSS | --- (outside date range) | --- | 0 |
| **Total surcharges** | | | **1{,}727 USD** |

The base air freight rate (GCR/BSA pivot) for this shipment runs perhaps $3,500–4,000 — surcharges add ~45% on top. A pre-Task-#2 model that omits the stack would under-quote this shipment by $1.7k. The optimizer chooses among routings with surcharges visible.

### 6.7 Lifecycle, versioning, RLS — same pattern as §4 and §5

Lifecycle (insert / supersede / soft-delete) and the per-run snapshot binding follow the identical pattern documented in §4.3 and §5.6. The reproducibility query is analogous: `routing_run → surcharge_snapshots → surcharge_ids → surcharge_catalog`.

RLS: tenant-scoped rows visible only to that tenant; `tenant_id IS NULL` rows are shared baselines accessible to all tenants. Common shared baselines: IATA-mandated AMS/ICS2/ACI flat fees, carrier-published war-risk surcharges. Common tenant-specific rows: negotiated FSC discounts, tenant-only handling fees.

### 6.8 Out of MVP scope

- **Stacked compound surcharges** — e.g., "FSC = base_FSC × (1 + war_risk_uplift_pct)". MVP treats each surcharge as additive; multiplicative interactions are P1.
- **Auto-ingestion from carrier rate sheets** — manual entry / CSV upload in MVP; carrier-API ingestion is P1.
- **Per-shipper override stacking** — currently, shipper-specific surcharge overrides require explicit catalog rows. P1: a cleaner per-shipper supplement table layered on top of base catalog.

## 7. Currency and FX

The model's canonical settlement currency is **USD**. All optimizer cost computations are in USD. Rates and surcharges stored in the catalog tables (§5, §6) carry an explicit `currency` field and are converted to USD at solve time using the FX snapshot bound to the routing run.

### 7.1 Tables

```sql
CREATE TABLE fx_snapshots (
  snapshot_id     UUID PRIMARY KEY,
  base_currency   TEXT NOT NULL DEFAULT 'USD',
  source          TEXT NOT NULL,            -- 'ecb' | 'oanda' | 'manual' | 'tenant_override'
  captured_at     TIMESTAMPTZ NOT NULL,
  freshness_threshold INTERVAL NOT NULL DEFAULT '24 hours',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE fx_rates (
  rate_id         UUID PRIMARY KEY,
  snapshot_id     UUID NOT NULL REFERENCES fx_snapshots(snapshot_id),
  from_currency   TEXT NOT NULL,            -- e.g., 'TWD', 'EUR', 'JPY'
  to_currency     TEXT NOT NULL,            -- typically 'USD'
  rate            NUMERIC NOT NULL,         -- 1 from_currency = rate × to_currency
  UNIQUE (snapshot_id, from_currency, to_currency)
);

ALTER TABLE routing_run_policy_bindings
  ADD COLUMN fx_snapshot_id UUID REFERENCES fx_snapshots(snapshot_id);
```

### 7.2 Solve-time conversion

For any monetary value $v$ in currency $X$ at solve time:
```
v_usd = v × fx_rates[fx_snapshot.snapshot_id][X → USD].rate
```

If a snapshot is missing a required (from_currency, to_currency) pair, the solve fails fast with a structured error — no inferred or interpolated rates. Snapshots are tenant-agnostic (FX is shared baseline data); a single global FX snapshot per capture window suffices.

### 7.3 Capture cadence

- Daily snapshot from ECB (European Central Bank) reference rates or OANDA (commercial FX)
- Manual override available per tenant for contract-locked rates (e.g., a shipper contract that fixes EUR conversion for the contract year regardless of market)
- Snapshot freshness threshold default 24h; tenants on tight FX exposure can tighten to 4h

### 7.4 Out of MVP scope

- **Forward FX contracts** — tenant hedges currency exposure with forward contracts. Out of routing-model scope; lives in treasury / billing.
- **Multi-leg currency settlement** — different invoice currencies per leg of a multimodal shipment. P1; MVP normalizes all to USD at solve time.
- **FX spread / bid-ask modeling** — using mid-market rate in MVP; spread modeling is treasury concern.
