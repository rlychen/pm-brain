# Critique 09 — Scenario DB architecture (scoped DB-architect review)

**Date:** 2026-06-06 (Session 30). **Reviewer:** scoped DB-architect agent (general-purpose),
review-only. **Scope:** exactly three targets — (1) SQLite↔Postgres parity/drift, (2) the
production↔optimizer adapter seam, (3) schema coherence of `src/scenario_db.py` + the run-identity
model. **Out of scope (excluded by design):** multi-tenant RLS, partitioning, sharding, scaling,
write-concurrency (J3), production auth — single-tenant single-process sim.

**Why this exists.** Run at the schema-first boundary (after `scenario_db.py` was built, before
generator-to-files). The headline: the **output side is sound to build on; the input side is not
ready** — the schema cannot reconstruct the optimizer's inputs. Resume Session 31 by deciding the
input-layer fork (below), then applying the cheap hardening edits, then a round-trip spike.

---

## ▶ FIRST NEXT SESSION — the open decision (input-layer fork, UNRESOLVED)

The replay loop must rebuild `Offer`/`Hawb`/`Gateway`/`Hub`/`RateCatalog` from DB rows. Today it
**cannot** — an entire input layer is missing. How to close it (user was deciding when we stopped):

- **Option A (reviewer + my recommendation) — add the Offer layer + spike-validate.** Add `offers`
  + `offer_legs` + `gateways` + `hubs` tables; promote the six per-HAWB ground scalars onto
  `shipments`; move the fallback constant into `config.json`. Then a **thin round-trip spike** (write
  one scenario → reconstruct `Offer/Hawb/Gateway/Hub/RateCatalog` from rows → `build_air_graph`) to
  prove the schema **before** writing the full generator-to-files. Persists what the MILP consumes;
  `schedule_legs`/`air_flight_legs` stay as the physical substrate underneath. Fastest, lowest-risk.
- **Option B — strict production-shaped + fat adapter.** No `offers` table; reconstruct Offers at
  load time by re-deriving overlapping emission / through-routing from `schedule_legs` + rate tables.
  More faithful to the Postgres production swap, but the adapter is complex and is itself the
  round-trip risk the review flagged.
- **Option C — run the proof in-memory first.** Defer the whole SQLite file layer; first
  replan-savings proof runs on the in-memory `AirInstance`, file persistence added later. **Reopens
  the Session-29 decision** (all data generated to files first, then replayed). Confirmed with user:
  in-memory ⇒ **no input or output files at all**; the proof *number* is still valid (determinism/CRN
  come from the seed, not files); what's given up is the durable/shareable artifact + persisted plan
  history + conservation-ledger audit trail. A "ship faster, add durable substrate after" call, not a
  weaker-number call.

**Key insight driving the fork:** the spec's §9 order (schema → generator → adapter) freezes the
input tables *before* the adapter that consumes them is proven. A missing-column discovery during the
replay-loop build would force a destructive migration of a file format meant to be a permanent
artifact. The round-trip spike (Option A) de-risks that.

---

## Cheap hardening edits — APPLY regardless (output side is otherwise sound)

These were agreed "just apply" but **not yet applied** (we stopped before editing):

1. **Normalize an `executions` table.** `runs` repeats `execution_id`/`scenario_id`/`config_hash`/
   `sim_clock_start`/`executed_at` across all 4 arm rows of one execution — a 4× duplication that can
   drift. Split: `executions(execution_id PK, scenario_id, config_hash, sim_clock_start, executed_at)`
   + `runs(run_id PK, execution_id FK, arm)` with `UNIQUE(execution_id, arm)`. Output tables keep
   keying on `run_id`, unchanged. (MATERIAL, Target 3.)
2. **`UNIQUE(execution_id, arm)`** on runs (even if denormalized) — the DB should assert one row per
   arm per execution, not rely on the writer building `run_id` correctly. (MINOR.)
3. **`is_current` partial unique index:** `CREATE UNIQUE INDEX ... ON routes(run_id, shipment_id)
   WHERE is_current = 1` — enforce the documented "exactly one current plan" invariant; a double-current
   row silently corrupts "final plan" queries. (MINOR.)
4. **Bool `CHECK (col IN (0,1))`** on `has_lithium`/`is_current`/`on_fallback`/`on_time`/`flex_k` —
   closes a Postgres-`BOOLEAN` parity gap cheaply. (MINOR.)
5. **`cap_volume_cbm NOT NULL`** on `air_flight_legs` — matches canonical + the 2D-capacity intent; a
   null volume cap is a latent infeasibility-vs-unbounded bug. (MINOR.)

---

## Target 1 — SQLite↔Postgres parity / drift

- **MATERIAL — `departure_days TEXT[]` dropped silently.** `data_model.md:317` has it NOT NULL;
  `scenario_db.air_uld_allocations` omits it. Arguably correct in the sim (allocations key to a
  concrete `schedule_leg_id`, each departure already materialized), but it's **undocumented drift**:
  the "one-module swap" claim is false here — Postgres reintroduces a NOT NULL column the SQLite
  writer never populates. Fix: comment it as collapsed-into-materialized-legs; Postgres adapter
  synthesizes it from the leg set.
- **MATERIAL — `air_uld_allocations` diverged structurally, not just dialectally.** Postgres:
  `origin_airport/destination_airport/effective_from/effective_to/contracted_rate_per_kg/departure_days`;
  SQLite replaced these with `schedule_leg_id` + `uld_w_kg/uld_v_cbm` + `remaining_ulds`. A real model
  change (per-lane-window → per-concrete-leg). **`contracted_rate_per_kg` has no home in SQLite** — the
  rate now lives on the `bsa_contracts` parent (`rate_per_kg`/`overage_rate_per_kg`/`allowance_kg`).
  Confirm the move and either delete `contracted_rate_per_kg` from canonical or annotate it.
- **MATERIAL — polymorphic `supply_ref` enum points at non-existent tables.** `route_legs.supply_ref_type`
  allows `schedule_leg|ondemand_arc|rate_card_lane`, but the sim has **no** `trucking_ondemand_arcs` or
  `rate_card_lanes` tables — two of three values resolve to nothing. Drop the unused values from the
  sim's allowed set (and document) or add the tables. (No FK is expected on a polymorphic ref.)
- **MINOR — bool-as-INTEGER unconstrained** (see hardening #4). **MINOR — `cap_volume_cbm` nullable**
  vs canonical NOT NULL (see hardening #5).
- **Verified fine (not problems):** `JSONB→TEXT` with `sort_keys=True`; no `gen_random_uuid()`/`NOW()`
  defaults anywhere (`det_id` + explicit `created_at = sim_clock`); `NUMERIC→REAL` cost/ETA + `→INTEGER`
  capacity (the conservation CHECK depends on it); `PRAGMA foreign_keys = ON` on every connection;
  type-affinity laxity is inert for a generator writing typed Python values via parameterized inserts.

## Target 2 — production↔optimizer adapter seam (the weak area)

The round-trip from rows back to optimizer inputs is **not lossless today**. The gap is an entire
missing layer, not adapter glue.

- **BLOCKING — `Gateway`/`Hub` configs have no table.** `air_graph.Gateway` (code, on_airport,
  cartage_h, cartage_cost, cfs_dwell_h, cfs_handling_cost) and `air_graph.Hub` (code, is_cfs_h,
  dwell_h, dwell_cost) are mandatory `build_air_graph` inputs, hardcoded today in
  `tpeb_air_instance._gateways()/_hubs()`. Absent from both `scenario_db.py` and `data_model.md`. A
  file-first replay **cannot reconstruct the graph** without them. Fix: add `gateways` + `hubs` tables
  (or one `ground_config` table with a `kind` discriminator). **Single most concrete missing piece.**
- **BLOCKING — ground/dwell/fallback arcs + per-HAWB ground scalars not persisted.** Pickup, cartage,
  CFS dwell, customs, hub dwell, fallback are first-class `Arc`s the MILP routes + bills, with per-HAWB
  scalars (`pickup_h/pickup_cost/delivery_h/delivery_cost/customs_dwell_h/customs_cost` on `Hawb`).
  `schedule_legs` holds only air; the six ground scalars are **not columns on `shipments`**. Fix:
  promote the six scalars onto `shipments` (matches their per-HAWB nature); persist gateway/hub dwell
  via the config tables. Fallback arcs are derivable from `Hawb.deadline_abs_h` + a tenant fallback
  constant — no table needed, but **the fallback constant must live in `config.json`**.
- **MATERIAL — rate tables don't cover every `RateFamily`, and the join key is wrong.** `RateCatalog`
  keys all families by `ArcId = "AIR:{offer_id}"`; `spot_rate_quotes` keys by `schedule_leg_id` and
  enumerates only `flat_rate|min_flat_breaks|coload_per_kg` — **`per_uld_pivot` absent**. Three
  problems: (1) an `Offer` is *not* a `schedule_leg` — overlapping emission means one offer → 1..n legs
  (`Offer.legs`); a through-offer spans two legs, so `spot_rate_quotes.schedule_leg_id` can't identify
  the priced segment, and there is **no `offers` table** carrying `offer_id`/origin/dest/rate_family/
  cutoff/uld_max. (2) `BsaContract.arcs`/`allotment`/`pivot`/`r_a` have no clean reconstruction path —
  `air_uld_allocations.ulds_per_departure` is per-leg not per-offer-arc; `pivot_kg` sits on the contract
  not the arc. (3) `FlatRate.cap` (C.5c per-offer weight ceiling) has no column. Fix: add an **`offers`
  table** + **`offer_legs` junction** (`offer_id, seq, schedule_leg_id`) so overlapping emission
  round-trips; re-key `spot_rate_quotes` to `offer_id`; add `per_uld_pivot` / fold the BSA-arc mapping.
- **Net:** production tables as written cannot losslessly reconstruct optimizer inputs. Missing:
  the `Offer` abstraction (offer + offer→leg junction), `Gateway`/`Hub` config tables, per-HAWB ground
  scalars on `shipments`.

## Target 3 — schema coherence + run-identity model

- **MATERIAL — normalize `executions`** (hardening #1). **MINOR — `UNIQUE(execution_id, arm)`**
  (#2). **MINOR — `is_current` partial unique index** (#3).
- **Verified sound:** lineage `runs→planning_runs→routes→route_legs` is clean (FK targets precede
  referents; `UNIQUE(run_id, cycle)`, `UNIQUE(run_id, shipment_id, cycle)`, `UNIQUE(route_id, seq)`;
  versioned by explicit integer `cycle`, not `created_at`). Conservation `CHECK` is exact integer and
  correctly placed. `flex_denominator` scoped to `scenario_id` (not `run_id`) is right per D-F7.
  `booking_promise (run_id, shipment_id)` write-once is coherent. `sim_state CHECK(id=1)` + reveal view
  + parameterized `visible_shipments(t)` cross-arm form are correct. **Index coverage is adequate** —
  `realized`/`capacity_ledger`/`booking_promise`/`metrics` are all `run_id`-prefixed by their PKs; no
  missing index.

## Net assessment (reviewer)

Output side is sound — build on it with the two cheap hardening edits. Input side is **not ready**:
missing the `Offer` abstraction, `Gateway`/`Hub` tables, and per-HAWB ground scalars; `spot_rate_quotes`
mis-keyed and omits `per_uld_pivot`. The "SQLite→Postgres is one module" claim is overstated by the
`departure_days`/`air_uld_allocations` drift (documentation/adapter problem, not a blocker).

**Resolve first:** the missing supply-and-ground persistence layer (Target 2's two BLOCKINGs) — add
`offers` + `offer_legs`, `gateways`/`hubs`, promote the six ground scalars — **before** generator-to-files,
then a thin round-trip spike to prove the schema. The §9 ordering freezes these tables before the
adapter that reads them is built, so a late discovery forces a destructive migration of a "permanent"
file.

---

## Decisions LOCKED this session (Session 30), for continuity

- **data_model.md §3 column promotion DONE:** `shipments` gained `tier` / `known_at` / `tender_at` /
  `effective_deadline_at` (Δ_k) / `backstop_deadline_at` (T_dead). (Demand `tier` distinct from
  `routes.tier` echo and `booking_promise.tier` frozen promise.)
- **`scenario_db.py` BUILT & green** (18 tables + reveal view; §2.1 determinism pins; integer
  conservation CHECK; generic typed helpers; `det_id`/`rng_stream` factories). 128 tests pass, ruff clean.
- **Run-identity = Decision (b): execution history retained.** Re-running a scenario APPENDS a new
  execution (4 fresh arm-runs), never overwrites. `runs.execution_id` groups the arms;
  `run_id = {execution_id}:{arm}` globally unique; `executed_at` (wall-clock ISO-8601) + `execution_id`
  are the deliberately-real-world provenance fields, written explicitly by the harness (never a DB
  default), excluded from input-table reproducibility. `new_execution(scenario_id)` mints both.
  Output tables key on `run_id` → each execution isolated; file grows per execution by design.
  **(NOTE: the reviewer recommends normalizing this into an `executions` table — hardening #1.)**
- **Storage model confirmed:** ONE `scenario.db` per scenario holds BOTH pre-generated inputs AND
  replay outputs; only sidecars are `config.json` (repro key) + `corpus.jsonl` (L3/L4 exhaust). The sim
  runs *off* SQLite (reads inputs via reveal view, writes outputs back to the same file), not off loose
  files.
