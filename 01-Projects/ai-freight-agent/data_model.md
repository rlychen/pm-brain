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
