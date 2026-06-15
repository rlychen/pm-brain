# Scenario IO & Replay Harness

**Status:** v0.2 — **APPROVED (Session 29, 2026-06-06).** Critique round 1 folded (§2.1 determinism block + B4/M1/M3/M4); B1 **downgraded per user** from "tie-break engineering" to a plain solver-reproducibility pin (whichever optimum the deterministic solver returns is locked in under non-anticipativity — that's correct policy, not a phantom). Data-architecture spec for the simulation. Companion
to `synthetic_data_contract.md` (the generation output contract) and `data_model.md` (the canonical
table schemas). Realizes the locked decisions: pre-generate + sim-clock reveal, `external_id`
idempotency, provenance tags, route-versioning (immutable per-cycle plan snapshots), orchestrator-
driven capacity decrement, the conservation identity (`backtest_methodology.md §6`).

> **Why this exists.** The generator currently returns an in-memory `AirInstance`. The user requires
> **all data generated to files first, then replayed forward in time** — for reproducibility,
> auditability, and recording each shipment's multiple route plans. This spec defines the file/table
> schemas and the replay-loop contract. No code until approved.

---

## 1. Principle — generate-all-first, then deterministic replay

**All stochastic draws happen once, at generation, and are written to the scenario.** The replay loop
consumes the scenario and produces *decisions*, but **draws nothing new**. Consequences:
- **Reproducibility:** `(seed, config)` → byte-identical scenario → identical outputs. Re-runnable
  forever; a scenario file is a permanent artifact.
- **CRN for free:** every policy arm (`H₀/M₀/M₁'/M₁/π_hind`) reads the *same* scenario, so a measured
  delta is policy difference, not sampling noise — no separate CRN plumbing.
- **Plan-on-estimate / score-on-actual** is clean: planners read the published schedule; the scorer
  reads the **pre-generated actuals**; π_hind is simply *allowed* to read the actuals at `t=0`.

What is **pre-generated** (input): supply, demand + arrival timing, **transit actuals** (one draw per
leg/component, frozen), spot regime. What is **computed during replay** (output): the plans/decisions
(deterministic given inputs + policy) and their realized outcomes.

---

## 2. Storage — SQLite, one file per scenario

Decision (user): **SQLite single-file DB per scenario** (`scenario.db`). It realizes the
`data_model.md` schema directly — tables, the sim-clock reveal view, transactional capacity decrement,
the route-versioning tables — in one portable, reproducible file via Python's stdlib `sqlite3`.
**Postgres-swappable:** same SQL DDL/queries; production swaps the connection. Dialect adaptations to
honor: (a) types — SQLite uses `TEXT` for the `UUID`/`TIMESTAMPTZ`/`JSONB` of the Postgres DDL;
(b) the reveal view uses a `sim_state` table, not Postgres `current_setting` (§5); (c) single-writer
(fine for the single-process sim; production concurrency is a separate J3 concern).

**Code seam:** all DB access goes through one thin `scenario_db` module (open/migrate/read/write), so
the SQLite→Postgres swap is one module, not scattered SQL.

---

## 2.1 Determinism requirements (mandatory — reproducibility is engineered, not assumed)

"Byte-identical from `(seed, config)`" only holds if these are enforced. Each is a `scenario_db` /
generator DoD item.

- **No wall-clock or random defaults in the DB.** Forbid `DEFAULT NOW()` / `gen_random_uuid()`
  (the `data_model.md` Postgres DDL uses these — they must NOT carry into `scenario.db`). All ids and
  timestamps are supplied **explicitly and deterministically** by the writer. (B2/B3)
- **Deterministic ids.** Primary keys are deterministic (`gen-{seed}-{table}-{seq}` or content hash),
  not random UUIDs — the same `external_id` discipline the contract §7 already mandates. (B3)
- **Named, independent RNG sub-streams.** One frozen RNG library — **stdlib `random.Random`** (already
  used in `air_transit_time.py`) — with a **separate stream per draw class**, each seeded
  deterministically as `Random(f"{seed}:{stream}")` for `stream ∈ {demand, leg_actuals,
  component_actuals, spot_regime}`. This keeps sub-streams stable when one axis of the `(κ,λ)` grid
  changes (CRN across the grid would otherwise silently break if a single shared stream shifted when
  `n_hawbs` changed). Do not switch RNG libraries later (stdlib ≠ NumPy streams). (B3)
- **Version ordering by an explicit integer, not `created_at`.** Route-versioning lineage uses the
  explicit monotonic `cycle` (and a per-shipment `version` if needed), **not** `created_at` — multiple
  cycles can share a `sim_clock`, so timestamp ordering collides. `created_at` is set to the
  `sim_clock` value (not ingestion time). (B2)
- **Solver reproducibility (a config pin — NOT tie-break engineering).** Pin HiGHS `threads=1` +
  fixed `random_seed` so a re-solve of the same instance is bit-identical (else parallel MIP timing can
  return different equally-optimal solutions → a re-run gives a different total, breaking
  reproducibility). No tie-break canonicalization is needed: whichever optimum the deterministic solver
  returns is **locked in under non-anticipativity** (you don't retroactively re-pick a different optimum
  when new demand arrives — that would be prescience). This also makes `M₁'`/`M₁` return the **identical
  pre-divergence plan by construction** (same MILP engine on the identical `t=0` instance), so they diverge only
  on genuine replanning = the real L2. DoD: two solves of the same instance → identical `route_legs`.
  - **Hash-independent model build (Session 31).** Arc/HAWB/group ids are strings, so iterating
    sets/dicts to create solver variables and constraints is `PYTHONHASHSEED`-dependent *across
    processes* (within one process it is stable, so the 4 arms in one run are already consistent —
    the headline L2 is safe today). The MILP canonicalizes **column order** by `sorted()` at the
    variable-creation sites (`air_milp.solve`); for full cross-process byte-identity (constraint-row
    order too) the **harness entrypoint and CI must set `PYTHONHASHSEED=0`**. Together with the seed
    pin this makes a re-run on another machine/process reproduce identical `route_legs`.
- **Capacity as integers; the conservation identity is exact.** Store capacity quantities as
  **INTEGER** (ULD slots; chargeable weight as integer kg) so `cap_init = tendered +
  committed_untendered + free` is exact integer arithmetic (no float `==`). Costs/ETAs use SQLite
  native `REAL` (IEEE-754 exact round-trip), **not `TEXT`**; any cost equality uses `abs(residual) < ε`
  with a stated ε. (M2)
- **Idempotent / fresh-DB generation.** Generation writes into a **fresh** `scenario.db`
  (delete-and-recreate on regeneration) so "regeneration is byte-identical" holds; the contract's
  `ON CONFLICT` upsert path-parity is exercised separately as a production concern. (M5)
- **`PRAGMA foreign_keys = ON`** on every connection (off by default in SQLite — else silent orphan
  rows in the route-versioning lineage). (m1)
- **JSON written with `sort_keys=True`** (e.g. `corpus`/`I_t_snapshot`) so dict ordering can't leak
  into byte-identity. (m3)

---

## 3. Scenario layout & identity

```
data/synthetic/scenarios/{scenario_id}/
    config.json        # seed + every knob (the reproducibility key, human-readable)
    scenario.db        # SQLite: inputs (pre-generated) + outputs (written by replay)
```
- `scenario_id = s{seed}_k{kappa}_l{lambda}_h{horizon}_n{n_hawbs}` (+ a short hash of the full config
  for any remaining knobs). Human-legible *and* unique.
- `config.json` is the canonical reproducibility key: `{seed, n_hawbs, kappa (capacity_scale), lambda
  (arrival_lateness), tier_mix, dispersion_s, regime, horizon_days, cadence, T_dead_prob, …}`.
  Regenerating from the same `config.json` must reproduce `scenario.db`'s **input** tables byte-for-row.
- Outputs for multiple arms live in the same `scenario.db` (a `runs` registry keys them, §7), so one
  file holds the whole experiment for that scenario.

---

## 4. Input tables (pre-generated; written by the generator)

Supply + demand tables are the `data_model.md` schemas (referenced, not duplicated here), every row
tagged provenance (`real_network` vs `synthetic`) and carrying `external_id = gen-{seed}-{seq}` where
the contract requires idempotency:

| Table | Source | Notes |
|---|---|---|
| `schedule_legs` (+ `air_flight_legs`) | `data_model.md §1.3` | flights: ETD/ETA, cutoff, 2D capacity |
| `bsa_contracts`, `air_uld_allocations` | `data_model.md` | contracted blocks (fixed supply) |
| `spot_rate_snapshots` / `spot_rate_quotes` | contract §5 | **κ-regime-realized** spot ratio (§ memory `reference_air_spot_contract_ratio`); static within a scenario |
| `shipments` (HAWBs) | `data_model.md §3` | + `tier`, `Δ_k`, `T_dead`, **`known_at`** (arrival), **`tender_at`** (irreversible-lock time) |

**New simulation-only tables (the frozen realizations — this spec adds them):**
- `leg_actuals(flight_id, realized_block_h, realized_arrival_h)` — one draw per flight leg (2b),
  shared by all arms. **Generated for EVERY flight leg in the schedule, not just originally-planned
  ones** (B4): the methodology's recourse rolls/replans a shipment onto flights it wasn't first
  planned on (`air_transit_time.md §4`), and the scorer must read a *frozen* actual for those too —
  drawing one at score time would re-introduce randomness into the "draws-nothing" replay and break
  CRN. The schedule is finite and static (D-T5), so pre-sampling all legs is cheap.
- `component_actuals(hawb_id, arc_type, realized_delta_h)` — per-HAWB ground/customs realizations
  (own entry), keyed so **any feasible path the HAWB could be replanned onto is covered** (or made a
  deterministic function of `(hawb_id, arc_type, seed)` resolvable without a new draw). Together with
  `leg_actuals` these *are* the realized timeline the scorer walks (§6).
- `sim_state(sim_clock)` — the replay clock (§5).

Provenance: real topology/schedule structure where available; synthetic commercial params + actuals.

**Net-new columns to promote into `data_model.md §3`:** `shipments.known_at`, `shipments.tender_at`,
and the tier fields (`tier`, `Δ_k`, `T_dead`) are introduced by the methodology/2-FLEX and are not yet
in the canonical `shipments` DDL. Promote them (same way the contract promotes `schedule_legs`) before
the schema-first build, or the DDL writes a table the canonical model doesn't define.

---

## 5. Reveal mechanism (SQLite form)

The replay loop never reads `shipments` directly during a run — it reads a sim-clock view:
```sql
CREATE VIEW visible_shipments AS
  SELECT s.* FROM shipments s, sim_state c
  WHERE s.known_at <= c.sim_clock;
```
Advancing `UPDATE sim_state SET sim_clock = :t` "reveals" more HAWBs. (This replaces the contract's
Postgres `current_setting('app.sim_clock')`; identical effect, SQLite-portable.) The information set
`I_t` = `visible_shipments` at clock `t` + current capacity state — the policy's *sole* input channel
(the no-lookahead contract, `backtest §5`).

**Cross-arm isolation (M4).** `sim_state` is a single global row, but four arms share one
`scenario.db`. Contract: **arms run strictly sequentially in a single process**, and the per-arm
mutable state is isolated — `sim_state` is reset (or, cleaner, **pass `sim_clock` as a query
parameter** rather than mutating a global), and `capacity_ledger` is keyed by `run_id` so arms' capacity
state never collides. `shipments`/schedule (immutable inputs) are shared read-only — no cross-arm leak
there. State the sequential-execution contract explicitly; don't leave it implicit in "single-writer."

---

## 6. Replay loop contract (deterministic; draws nothing)

Per scenario, per policy arm, walk the clock in steps (cadence from `config`):
```
for t in steps(horizon, cadence):
    UPDATE sim_state SET sim_clock = t
    I_t  ← visible_shipments(t)  +  capacity_ledger(t)        # only known_at ≤ t
    plans ← policy.plan(I_t, published_schedule_estimates)    # H₀ heuristic | M₀ greedy | MILP (M₁'/M₁)
    for shipment in plans:                                    # RECORD EVERY plan this cycle
        write planning_run + route + route_legs (immutable snapshot, §7)
    firm_up_and_tender(bookings with tender_at ≤ t)           # capacity decrement; conservation identity
score: per shipment, REPLAY the running-clock walk of air_transit_time §4 with the FROZEN
       per-leg/component actuals substituted in (max(dep, clock) per leg; connection-made check;
       roll/replan recourse) → realized arrival + on_time + cost. NOT a naive sum of leg arrivals.
```
**Scorer = deterministic replay of the §4 walk (M1).** End-to-end arrival is *not* readable straight
off `leg_actuals`: whether a shipment makes a connection depends on its *upstream* realized arrival vs
this flight's realized departure (the running clock). The scorer reuses `air_transit_time.sample_route`
semantics with the frozen draws plugged in (so it's a deterministic replay, not a new sample); the
connection-made check + recourse are the orchestrator's per `air_transit_time.md §4`.

Determinism: the MILP and the heuristic are deterministic **under the §2.1 solver pins + tie-break**;
inputs are fixed; so the run is reproducible. `M₀` (greedy) and `M₁'` (single-pass MILP) pin priors —
they re-plan only newcomers, never disturbing prior commitments; `M₁` re-plans the open book each
step → naturally produces *multiple* plan snapshots per shipment. `π_hind` is one solve at `t=0` over
the full demand + `leg_actuals` (physical-feasible only, no tender lock — `backtest §3`).

---

## 7. Output tables (written by replay; keyed by run)

- `runs(run_id, scenario_id, arm, config_hash, created_at)` — registry; one row per (scenario × arm).
- **`planning_runs` / `routes` / `route_legs`** (route-versioning, `data_model.md`) — **the per-shipment
  plan history.** Each cycle that touches a shipment writes a complete **immutable snapshot**:
  `planning_runs(run_id, shipment_id, cycle, sim_clock, trigger)`; `routes(version immutable, created_at
  = lineage)`; `route_legs(leg, supply_ref ∈ {schedule_leg, ondemand_arc, rate_card_lane, NULL},
  resolution ∈ {ABSTRACT, CONCRETE}, firmness ∈ {PLANNED, FIRM, EXECUTED}, est_*, act_*)`. Past legs
  carry actuals; future legs are replanned at higher resolution. **This is your "record each shipment's
  multiple route plans."**
- `realized(run_id, shipment_id, final_route_id, realized_arrival_h, on_time, realized_cost, fallback)`.
- `capacity_ledger(run_id, arc_id, sim_clock, tendered, committed_untendered, free)` — the conservation
  identity audit trail (`backtest §6`); asserted exact (integer, §2.1) `cap_init = tendered +
  committed_untendered + free` per arc per step. Keyed by `run_id` (no cross-arm collision, §5/M4).
- **`booking_promise(run_id, shipment_id, promised_Δ_k, z_tier, tier)` (M3)** — written **once at
  tender, never updated**; the frozen-promise / frozen-`z_tier` invariants (`backtest §6`) assert
  replan never alters these (vs the live values in `shipments`).
- **`flex_denominator(scenario_id, shipment_id, flex_k, cw_k)` (M3)** — `cw_flex` is `t=0` and
  **arm-invariant**, so it is **scenario-scoped, NOT `run_id`-scoped** (keying per-run would let arms
  diverge and defeat the D-F7 arm-invariance assertion).
- `metrics(run_id, total_cost, otp, L1, L2_reshuffle, L2_fallback_avoidance, fallback_count, …)`.
- `corpus` — the L3/L4 exhaust (decision, `I_t` snapshot, realized-outcome). **Flat JSONL for the
  first version** (`backtest §9` already blesses this; minimal-design — SQLite rows are a later
  promotion), `sort_keys=True`.

**Indexes (m2):** `planning_runs`/`routes` on `(run_id, shipment_id, cycle)`; `route_legs` on
`(route_id, seq)`; `shipments` on `known_at` (the reveal view). Reconstructs "shipment k at cycle c"
and "final plan" without table scans.

---

## 8. Reproducibility & CRN (the guarantees)

- `(seed, config) → scenario_id`; regeneration is byte-identical on input tables; replay is
  deterministic → identical outputs. A `scenario.db` is a permanent, shareable artifact.
- **CRN:** all arms read the same input tables (same arrivals + same `leg_actuals`) → the L1/L2 deltas
  are policy-only. Paired-CI replications = different `seed`s (different scenario files), same `config`.
- A regeneration check (re-run the generator, diff the input tables) is a DoD item.

---

## 9. Components to build (and order)

1. **`scenario_db` module** — open/migrate (the DDL) / typed read+write helpers / the reveal view +
   `sim_state`. The single SQLite↔Postgres seam.
2. **Generator → files** — extend 2a to write the input tables (supply, demand) + **pre-sample the
   transit actuals** (`leg_actuals`/`component_actuals`, via 2b) into `scenario.db`; write `config.json`.
3. **2-FLEX into the schema** — tiers/`Δ_k`/`T_dead` populate `shipments`; `TierSpec` is the shared
   source (resolves the no-code-home drift the seam audit flagged).
4. **Replay loop (2c)** — the §6 loop: reveal → plan → record snapshots → tender/decrement → score;
   writes §7 outputs. Owns the connection-check + recourse, conservation identity, both tripwires.

Order: **schema first** (§1 `scenario_db` DDL) — it's the contract everything writes into — then
generator-to-files, then 2-FLEX populates demand, then the replay loop + arms.

---

## 10. Definition of Done & open items

- [ ] `scenario_db` creates a `scenario.db` with the full schema; round-trips every table typed.
- [ ] Generator writes a complete scenario from `(seed, config)`; **regeneration is byte-identical**
      on input tables (reproducibility pytest).
- [ ] `visible_shipments` reveals exactly `known_at ≤ sim_clock`; no future rows leak (the demand
      tripwire reads this).
- [ ] Transit actuals pre-sampled + frozen; all arms read identical `leg_actuals` (CRN pytest).
- [ ] Replay records **every** plan cycle per shipment as an immutable route-versioning snapshot.
- [ ] `capacity_ledger` conservation identity (exact integer) holds per arc per step.
- [ ] One `scenario.db` holds multiple arms' runs keyed by `run_id`; arms run sequentially.
- [ ] **Solver determinism:** two solves of one instance → identical `route_legs` (§2.1, B1).
- [ ] **Regeneration byte-identical** on input tables; **all arms read identical `leg_actuals`**
      including rolled-onto flights (B4).
- [ ] Frozen `booking_promise` + scenario-scoped `flex_denominator` immutability invariants assert.

**Named fixture (designed here, not a checkbox) — the binding-capacity + mid-horizon-tender scenario.**
`backtest §6` calls this "the single most important missing test … never exercised." Define it as a
first-class scenario: a lane with **genuinely scarce** cheap capacity (e.g. 1 contracted ULD slot) and
a flight that **tenders/locks at `tender_at = t₁` while later HAWBs still arrive at `known_at > t₁`** —
so a HAWB physically commits mid-horizon and a subsequent arrival contends for the just-locked slot.
Asserts: the conservation identity each step; `M₁` cannot reshuffle a tendered shipment; a "freed" slot
equals an actual ledger return. This fixture ships with the replay loop, not "later."

**Open items (not blocking):** `cadence` default (orchestrator-design memory: 3/day) — set in
`config`; Parquet export of `metrics` for cross-scenario analysis (later). (`corpus`→JSONL and the
data_model column promotion are resolved above.)
