# Scenario DB — entity-relationship diagram

Mirrors the `SCHEMA` string in `src/scenario_db.py` (the single SQLite↔Postgres seam) as of
Session 31. Three layers: **input** (pre-generated, read-only at replay), **frozen
realizations** (one draw each, shared by all arms), **output** (written by the replay loop,
keyed by run). Attribute lists show PKs/FKs + a few representative columns, not every column —
read the DDL for the full set.

```mermaid
erDiagram
    %% ───────── INPUT: demand + physical substrate + commercial offers ─────────
    SHIPMENTS {
        text id PK
        text origin_locode
        text destination_locode
        real cargo_kg
        int  tier
        real known_at
        real tender_at
        real effective_deadline_at
        real pickup_h
        real customs_cost
    }
    SCHEDULE_LEGS {
        text id PK
        text mode
        text carrier_id
        real etd_h
        real eta_h
    }
    AIR_FLIGHT_LEGS {
        text schedule_leg_id PK,FK
        int  cap_weight_kg
        real cap_volume_cbm
        text cap_uld
    }
    GATEWAYS {
        text code PK
        int  on_airport
        real cartage_h
        real cfs_dwell_h
    }
    HUBS {
        text code PK
        int  is_cfs_h
        real dwell_h
    }
    OFFERS {
        text offer_id PK
        text carrier
        text rate_family
        text origin
        text dest
        text rate_json
        text regime
    }
    OFFER_LEGS {
        text offer_id PK,FK
        int  seq PK
        text schedule_leg_id FK
    }
    BSA_CONTRACTS {
        text id PK
        text settlement_basis
        int  allowance_kg
        int  pivot_kg
    }
    AIR_ULD_ALLOCATIONS {
        text id PK
        text contract_id FK
        text offer_id FK
        text uld_type
        int  ulds_per_departure
    }

    %% ───────── FROZEN REALIZATIONS (one draw; shared by all arms) ─────────
    LEG_ACTUALS {
        text flight_id PK,FK
        real realized_dep_h
        real realized_arrival_h
    }
    COMPONENT_ACTUALS {
        text hawb_id PK,FK
        text arc_type PK
        real realized_delta_h
    }
    SIM_STATE {
        int  id PK
        real sim_clock
    }

    %% ───────── OUTPUT (keyed by run / arm) ─────────
    EXECUTIONS {
        text execution_id PK
        text scenario_id
        text config_hash
        real sim_clock_start
        text executed_at
    }
    RUNS {
        text run_id PK
        text execution_id FK
        text arm
    }
    PLANNING_RUNS {
        text id PK
        text run_id FK
        int  cycle
        text trigger
    }
    ROUTES {
        text id PK
        text run_id FK
        text planning_run_id FK
        text shipment_id FK
        int  cycle
        int  is_current
    }
    ROUTE_LEGS {
        text id PK
        text route_id FK
        int  seq
        text supply_ref_type
        text supply_ref_id
    }
    REALIZED {
        text run_id PK,FK
        text shipment_id PK,FK
        text final_route_id FK
        int  on_time
    }
    CAPACITY_LEDGER {
        text run_id PK,FK
        text arc_id PK
        real sim_clock PK
        int  cap_init
    }
    BOOKING_PROMISE {
        text run_id PK,FK
        text shipment_id PK,FK
        real promised_deadline_at
    }
    FLEX_DENOMINATOR {
        text scenario_id PK
        text shipment_id PK,FK
        int  flex_k
        real cw_k
    }
    METRICS {
        text run_id PK,FK
        real total_cost
        real otp
        real l2_reshuffle
    }

    %% ───────── relationships ─────────
    SCHEDULE_LEGS ||--|| AIR_FLIGHT_LEGS : "air detail (1:1)"
    SCHEDULE_LEGS ||--o| LEG_ACTUALS : "frozen actual"
    SCHEDULE_LEGS ||--o{ OFFER_LEGS : "physical leg of"
    OFFERS ||--|{ OFFER_LEGS : "chains (≥1 leg)"
    OFFERS ||--o{ AIR_ULD_ALLOCATIONS : "BSA positions"
    BSA_CONTRACTS ||--o{ AIR_ULD_ALLOCATIONS : "pools"
    SHIPMENTS ||--o{ COMPONENT_ACTUALS : "ground actuals"

    EXECUTIONS ||--o{ RUNS : "arms (H0/M0/M1/hind)"
    RUNS ||--o{ PLANNING_RUNS : "cycles"
    RUNS ||--o{ ROUTES : "plans"
    RUNS ||--o| METRICS : "scored"
    RUNS ||--o{ CAPACITY_LEDGER : "ledger"
    RUNS ||--o{ BOOKING_PROMISE : "promises"
    RUNS ||--o{ REALIZED : "outcomes"
    PLANNING_RUNS ||--o{ ROUTES : "snapshot in"
    ROUTES ||--o{ ROUTE_LEGS : "legs"
    ROUTES ||--o| REALIZED : "final route"
    SHIPMENTS ||--o{ ROUTES : "planned as"
    SHIPMENTS ||--o{ REALIZED : "realized as"
    SHIPMENTS ||--o{ BOOKING_PROMISE : "promised"
    SHIPMENTS ||--o{ FLEX_DENOMINATOR : "flex denom"
```

## Notes / things the diagram can't show

- **No-FK logical links (intentional).** `GATEWAYS.code` / `HUBS.code` are referenced by
  `SHIPMENTS.origin_locode`/`destination_locode` and `OFFERS.origin`/`dest` **by value, with no
  FK** — gateway/hub configs are scenario-global lookups, not owned by a parent row. Same for
  `carrier_id`/`tenant_id` (plain provenance TEXT, no FK in the single-tenant sim). So GATEWAYS
  and HUBS render as standalone boxes; that's correct.
- **`ROUTE_LEGS.supply_ref` is polymorphic** (`supply_ref_type` + `supply_ref_id`): it points at
  a `schedule_leg` (and in the sim, only that), so no FK — a type-tagged soft reference.
- **`SIM_STATE` is a single pinned row** (`CHECK (id = 1)`); the `visible_shipments` view joins
  it against `SHIPMENTS.known_at` to do clock-gated reveal. The view isn't an entity, so it's
  not in the diagram.
- **`FLEX_DENOMINATOR` is scenario-scoped, not run-scoped** (PK `scenario_id, shipment_id`) — the
  D-F7 arm-invariance requirement. It's the one output-ish table that deliberately does *not*
  hang off `RUNS`.
- **`EXECUTIONS → RUNS`** is the Session-31 normalization: per-execution provenance
  (`scenario_id`/`config_hash`/`sim_clock_start`/`executed_at`) lives once on `EXECUTIONS`; `RUNS`
  is just `(run_id, execution_id, arm)` with `UNIQUE(execution_id, arm)`.
- **Capacity columns are INTEGER** so `CHECK (cap_init = tendered + committed_untendered + free)`
  on `CAPACITY_LEDGER` is exact integer arithmetic.
