# Synthetic Data Generation Contract

*Part of the AI Freight Routing PRD. See [PRD.md](PRD.md) for strategic overview and [data_model.md](data_model.md) for the canonical entity model this contract targets.*

**Purpose:** Define the output contract for the supply/demand generator used pre-launch (and as the optimizer test harness) so that synthetic data is *launch-compatible* — swapping synthetic → real customer data is a data-source change, not a rewrite.

**Status:** Spec draft. Two table definitions here (`schedule_legs`, `demand_generator_configs`) are referenced by `data_model.md` but not yet defined there; on approval, promote them into `data_model.md` §1.3 and §3.6 respectively.

---

## 1. Governing principle

The generator is a **data source, not a data format.** It writes valid rows into the *same tables the live system reads*, tagged by provenance. There is no bespoke "sim schema." Swapping to real data is then a source swap.

Three provenance hooks already exist in `data_model.md`:

- `shipments.ingestion_source = 'demand_generator'` (§3.6)
- `spot_rate_snapshots.source` + nullable `tenant_id` (§5.2–5.3)
- Project rule: every row tagged **real-network** vs **synthetic** at point of use

Every generated row sets these. Real network topology (LOCODE/IATA ports, carriers, schedule structure, lane mix) is used wherever possible; only commercial parameters (rates, allocation caps, demand attributes, arrival timing) are synthetic.

---

## 2. Supply classes

Two commercial supply classes, both riding on one physical substrate:

| Class | Meaning | Target table(s) | Rate behavior | Capacity behavior |
|---|---|---|---|---|
| **Fixed (contracted)** | Pre-committed block space at contracted rate | `carrier_allocations` (ocean BSA), `air_uld_allocations` (air ULD) | Fixed (`contracted_rate_per_kg`) | Committed block; `remaining_*` decremented on firm-up |
| **Floating (market/spot)** | Free-sale capacity at time-varying market rate | `spot_rate_quotes` within `spot_rate_snapshots` | Time-varying; perturbed each capture | Free-sale slice of leg capacity |

Both classes draw from the same physical leg: a sailing/flight has a total capacity (`schedule_legs.capacity_total`); the forwarder's BSA block is a slice of it, the remainder is free-sale spot. The generator emits all three layers — the physical substrate (§3), the fixed overlay, and the floating overlay.

---

## 3. New table — `schedule_legs` (physical substrate)

`data_model.md` §1.3 describes ocean-sailing and air-flight arc attributes in prose but gives them no SQL home, yet `spot_rate_quotes.flight_id` references a flight that must exist. For a streaming, reproducible sim the schedule must be persisted (so capacity state is replayable and rolling-horizon firm-up has a referent), not just materialized into the graph at solve time.

```sql
CREATE TABLE schedule_legs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mode            TEXT NOT NULL,            -- 'ocean' | 'air'
    carrier_id      UUID NOT NULL REFERENCES carriers(id),
    service_string  TEXT,                     -- ocean: ties to carrier_allocations.string_code
    flight_id       TEXT,                     -- air: ties to spot_rate_quotes.flight_id
    origin_locode   TEXT NOT NULL,            -- ocean POL / air origin IATA
    dest_locode     TEXT NOT NULL,            -- ocean POD / air dest IATA
    etd             TIMESTAMPTZ NOT NULL,
    eta             TIMESTAMPTZ NOT NULL,
    cy_cutoff       TIMESTAMPTZ,              -- ocean: latest gate-in (§1.3); air: latest acceptance
    capacity_total  NUMERIC NOT NULL,         -- TEU slots (ocean) / kg or ULD positions (air)
    capacity_unit   TEXT NOT NULL,            -- 'teu' | 'kg' | 'uld'
    transit_mean    REAL,
    transit_sigma   REAL,
    vessel          TEXT,                     -- ocean
    provenance      TEXT NOT NULL,            -- 'real_schedule' | 'synthetic'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ix_schedule_legs_lane
  ON schedule_legs (mode, origin_locode, dest_locode, etd);
```

Generation: schedule *topology* (which carriers run which lanes, frequency, transit envelopes) is real-network where available (carrier published schedules, OpenSky for air); `capacity_total` and exact ETDs may be synthetic. `service_string` must match `carrier_allocations.string_code` and `flight_id` must match the `spot_rate_quotes.flight_id` the generator emits, so the fixed and floating overlays bind to a real leg.

---

## 4. New table — `demand_generator_configs` (generator input)

Referenced in `data_model.md` §3.6 but never defined. This is the input contract that drives the arrival process — what makes HAWBs trickle in over time.

```sql
CREATE TABLE demand_generator_configs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id         UUID NOT NULL REFERENCES organizations(id),
    arrival_process   TEXT NOT NULL,          -- 'poisson' | 'nhpp' | 'replay'
    lambda_per_day    REAL,                   -- base arrival rate (homogeneous Poisson)
    seasonality       JSONB,                  -- {peak_multipliers, day_of_week_curve} for NHPP
    od_distribution   JSONB,                  -- lane mix, BTS FAF-derived weights
    attr_distribution JSONB,                  -- weight/volume/density/deadline/value sampling params
    mode_mix          JSONB,                  -- {ocean_fcl, ocean_lcl, air, multimodal} probabilities
    seed              BIGINT NOT NULL,        -- determinism
    horizon_days      INT NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

A parallel `supply_generator_config` (fixed-block caps, spot-rate volatility, capture cadence per mode) is the analog input for the supply side; spec deferred until the demand path is built and exercised.

---

## 5. Output stream → table mapping

| Stream | Class | Target table(s) | Provenance | Streaming semantics |
|---|---|---|---|---|
| HAWBs / shipment requests | demand | `shipments` (+ `shippers` seed) | synthetic attrs, real O/D | Pre-generated with future-dated `created_at`; revealed by sim clock (§6) |
| Fixed ocean capacity | fixed | `carrier_allocations` | synthetic caps, real strings | Static per period; `remaining_teu` decremented on firm-up (§8) |
| Fixed air capacity | fixed | `air_uld_allocations` | synthetic caps, real carriers | Static per `effective_from/to`; `remaining_ulds` decremented |
| Floating/market rates | floating | `spot_rate_snapshots` + `spot_rate_quotes` | synthetic rates, real lanes | Periodic capture (air hourly / ocean daily, §5.5); each tick = new snapshot, perturbed `weight_breaks` |
| Physical schedules + capacity | substrate | `schedule_legs` (§3) | real topology, synthetic capacity | Generated once per horizon |
| Surcharge stack | cost-side | `surcharge_catalog` (+ snapshots) | synthetic rates, real types | Mostly static; FSC/PSS refreshed periodically |
| FX | cost-side | `fx_snapshots` + `fx_rates` | synthetic or real ECB | Daily snapshot |
| Reference seed | substrate | `organizations`, `users`, `shippers`, `carriers`, `ports`, `tenant_carriers` | real topology, synthetic tenant identities | Seeded once, fixed seed |

Demand field mapping (`data_model.md` §1.1–1.2 → `shipments` columns) is 1:1; the generator samples `cargo_cbm`, `cargo_kg`, density, `cargo_ready_date`, `delivery_deadline`, `service_level`, and `mode` from `attr_distribution`.

---

## 6. Streaming reveal mechanism (decision: pre-generate + clock view)

All demand rows are generated up front with **future-dated `created_at`** spanning the horizon. A sim-clock view exposes only already-arrived rows:

```sql
CREATE VIEW visible_shipments AS
  SELECT * FROM shipments
  WHERE created_at <= current_setting('app.sim_clock')::timestamptz;
```

The optimizer/orchestrator reads `visible_shipments`, never `shipments` directly, during simulation. Advancing `app.sim_clock` "reveals" more HAWBs.

Rationale: this is fully reproducible and replay-friendly — the entire scenario is fixed at generation time by `seed`, and any solve can be replayed at any clock value. The alternative (a live process inserting rows as the clock advances) is closer to real ingestion-path concurrency but non-deterministic; it is **not** chosen for the harness. If ingestion-path concurrency needs testing later, that is a separate integration test, not the default sim mode.

---

## 7. Idempotency — `external_id` on every generated shipment

The generator sets a deterministic `external_id` per shipment (e.g. `gen-{seed}-{seq}`). This is mandatory, for two reasons:

1. **Path parity.** Real `push_api` ingestion (§3.6) is an idempotent upsert on `UNIQUE (tenant_id, external_id)` — `INSERT ... ON CONFLICT (tenant_id, external_id) DO NOTHING`. Generated rows must flow through the *same* upsert path, or the sim tests a code path that does not exist in production.
2. **Safe replays.** In Postgres, `NULL` values are distinct under a UNIQUE constraint, so `external_id = NULL` rows bypass dedup entirely — re-running the generator or restarting mid-scenario would silently duplicate every shipment. A real `external_id` makes the constraint fire and replays idempotent.

---

## 8. Capacity decrement ownership (decision: orchestrator-driven)

The generator sets only the *initial* state: `carrier_allocations.allocated_teu = remaining_teu` and `air_uld_allocations.ulds_per_departure = remaining_ulds` (fully unutilized at horizon start). It does **not** pre-bake utilization.

`remaining_*` is decremented at booking firm-up, driven by the **sim clock through the orchestrator** — the same rolling-horizon firm-up logic (`data_model.md` §3.5) the live system uses. This keeps capacity state a product of the system under test, so the harness actually exercises firm-up against live capacity rather than a static snapshot.

---

## 9. Open items / cross-references

- Promote `schedule_legs` (§3) into `data_model.md` §1.3 and `demand_generator_configs` (§4) into §3.6 on approval.
- `supply_generator_config` (§4) spec deferred until the demand path is built and verified.
- Surcharge and FX generation reuse the existing snapshot/binding pattern (`data_model.md` §6–7) — no new mechanism, just synthetic rows.
- Reproducibility chain: `seed` → fixed scenario → any solve replayable via existing `routing_run_*_bindings` (policy, rate, FX) plus the clock view.
