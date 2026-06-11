# Critique 10 — Proactive review round (5 agents, Session 31)

**Date:** 2026-06-08 (Session 31). **Reviewers:** 5 independent review-only agents — (1) data-model↔sim
consistency, (2) cross-component integration/seams, (3) test-coverage audit, (4) determinism &
reproducibility, (5) metric-computability/scorer pre-mortem. Run at the schema-committed / spike-not-yet-built
boundary, before generator-to-files freezes the file format.

**Headline:** the output side remains sound; the findings cluster on the **input/reconstruction layer**
(same root as critique 09 — the offer abstraction was added but the capacity-and-billing grain and a few
realization-table semantics weren't), plus **two determinism leaks in the MILP** that are the highest-leverage
fixes because they protect the L2 number's validity. Cross-confirmed findings are noted.

---

## ✅ RESOLUTION LOG (Session 31)

- **D1 RESOLVED — capacity stays NOT NULL; generator writes a freighter-max "no-cap" sentinel.**
  Decision (user): keep `cap_weight_kg`/`cap_volume_cbm` NOT NULL; generator-to-files writes the
  Boeing 747-8F max as effectively-uncapped flight capacity until the model is extended to bind
  flight cap. **Constant: `cap_weight_kg = 134200` (kg), `cap_volume_cbm = 858.0` (m³)** — sourced
  747-8F structural payload ~134,200 kg + ~858 m³ total cargo volume (boeing.com / cargolux.com).
  No schema change; constant lands with generator-to-files.
- **D3 RESOLVED — `capacity_ledger.arc_id` grain = `"{offer_id}:{uld_type}"`** (documented in DDL).
- **D4 RESOLVED — delete `contracted_rate_per_kg` from `data_model.md`** (rate lives on
  `bsa_contracts.rate_per_kg`); applied in the data_model reconciliation batch.
- **D2 RESOLVED — `mct_h` added to `hubs`** (feasibility threshold, distinct from `dwell_h`).
- **F1–F6 APPLIED & GREEN:** F1 HiGHS `threads=1`+`random_seed`; F2 empty-book guard +test; F3
  `rate_family` comment restored to canonical RateFamily values; F4 `route_legs.supply_ref_type` →
  `offer|schedule_leg|NULL`; F5 `order_route` unit test; F6 INFEASIBLE structured-status test.
- **S1–S6 APPLIED & GREEN (136 passed, ruff clean):** S1 hash-stable MILP column order (sorted
  variable creation: x/z/cw/eta/pu_mawbs) + `PYTHONHASHSEED=0` documented as the harness/CI pin for
  constraint-row order + two-solves regression test; S2 `leg_actuals` keeps only `realized_block_h`;
  S3 `component_actuals` arc_type→ArcType.value + `hub_code` in PK; S4 `route_legs` air-billing
  payload (`offer_id`/`mawb_group_key`/`billing_json`); S5 `uld_types` reference table + `offers.cap_kg`
  single home for C.5c (caps off `air_uld_allocations`); S6 time-origin invariant (sim-hour 0 = UTC epoch).
- **S7 DEFERRED to generator-to-files** — generator HAWB id adopts `det_id` when it writes `shipments.id`
  (the in-memory generator id is internal; no churn now).
- **DOC RECONCILIATION APPLIED:** `data_model.md` — bsa_contracts gains `rate_per_kg`/`pivot_kg`;
  `air_uld_allocations` drops `contracted_rate_per_kg`/`pivot_kg` (D4) + annotated production-vs-sim
  grain; added `offers`/`offer_legs`/`gateways`/`hubs`/`uld_types` (D-D2); §5 rate-on-offer note (D-D3).
  `scenario_io_and_replay.md §2.1` gains the hash-independent-build + `PYTHONHASHSEED=0` requirement.
- **ROUND-TRIP SPIKE DONE & GREEN (139 passed).** `data/synthetic/scenario_io.py` (`persist`/`load`)
  + `tests/test_scenario_io_spike.py`. Proves `solve(persist→load(inst)) == solve(inst)` (status /
  total_cost / routes / fallback / mawbs) on the full generated TPEB instance — through-offers, BSA
  per_uld_pivot, all four rate families, special cargo. Schema additions it required: `cargo_caps` on
  `air_flight_legs`; `prep_time_h`/`lambda_disp_h` on `shipments`; `deadline_abs_h`→`backstop_deadline_at`
  (decision a). **Bug the spike caught (its whole point):** `flight_id` is a flight NUMBER, not unique
  per segment — a multi-stop flight (CV33 HKG→ANC→ORD) reused it across both legs, collapsing them on
  reload. Fixed: schedule-leg PK keys per-segment `{flight_id}:{origin}-{dest}`; real flight number
  lives in `air_flight_legs.flight_no`. This is exactly the missing-column-vs-permanent-file risk the
  spike-before-freeze ordering was meant to retire.
- **NEXT: generator-to-files** (extend the generator to write a full scenario via `persist`, pre-sample
  ALL leg/component actuals through 2b into `leg_actuals`/`component_actuals`, with the G1–G4 generator
  determinism items from this review). Then 2-FLEX demand population, then the 2c replay loop.

---

## ▶ DIRECTIONAL DECISIONS NEEDED (cannot fix unilaterally)

These are coupled — they all flow from "what grain is capacity modeled at in the proof."

- **D1 — Capacity model scope.** The MILP's ONLY capacity coupling is the BSA ULD allotment
  (`air_milp.py` C.5 per-contract ULD positions). There is **no per-flight weight/volume constraint** —
  `air_flight_legs.cap_weight_kg`/`cap_volume_cbm` have no MILP consumer AND no in-memory producer
  (`Leg`/`Offer` carry no caps). But this session's hardening made `cap_volume_cbm NOT NULL`, so
  generator-to-files would have to invent a value for a column nothing sources. (Agents 1-B1, 3, 4 all hit this.)
  - **Rec: declare capacity ULD-slot-only for the proof** — make `cap_weight_kg`/`cap_volume_cbm` **nullable**
    + document as not-yet-modeled physical metadata. Adding a flight-capacity constraint is scope expansion
    we haven't planned. (Reverts part of hardening #5; the NOT-NULL was premature.)
- **D2 — MCT (minimum connection time) has no persisted home.** The §4 running-clock walk's connection-miss
  cascade — the whole reason realized arrival is *not* a naive leg-sum — needs `clock + MCT > dep(outbound)`.
  Schema has cutoffs (`*_cutoff_h`, `offers.cutoff_utc_h`) and hub *dwell* (`hubs.dwell_h`) but no MCT
  threshold. `MCT_h` is a named `[CAL]` param in `air_transit_time.md §7` with nowhere to live. (Agent 5 BLOCKING.)
  - **Rec: add `mct_h` to `hubs`.** Dwell (time consumed) ≠ MCT (feasibility threshold); conflating them is wrong.
- **D3 — `capacity_ledger.arc_id` grain is undefined** (free TEXT). Conservation is per-arc-per-step, but
  MILP capacity is per `(offer-arc, uld_type)`. (Agents 1-B2, 2-m1, 5-CaveatB.)
  - **Rec: define `arc_id = "{offer_id}:{uld_type}"`** (matches the allocation grain, round-trips against
    `air_uld_allocations`), consistent with D1. Add a test that every ledgered arc_id resolves to an allocation row.
- **D4 — Delete `contracted_rate_per_kg` from `data_model.md` canonical?** The rate now lives on
  `bsa_contracts.rate_per_kg`. (Agent 1-M3.) Canonical-model deletion → needs sign-off. **Rec: delete + annotate.**

---

## MECHANICAL FIXES (unambiguous; apply on greenlight)

- **F1 — Set HiGHS `threads=1` + `random_seed`** in `air_milp.py:238` (only `output_flag` is set today). The
  §2.1-mandated reproducibility pin is **absent** → solver runs at machine-dependent defaults → phantom L2.
  One line. **Highest leverage.** (Agent 4-B1, cross-confirmed Agent 3-m2.) No ε tie-break (settled policy).
- **F2 — Empty-book guard in `air_milp.solve`.** Confirmed LIVE: `solve(ag, [], rates)` raises
  `AttributeError: 'int' object has no attribute 'bounds'` at `air_milp.py:769` (`sum([])→0`). The orchestrator
  WILL hand an empty `visible_shipments` on early cycles. Return `AirSolution(status="OPTIMAL", total_cost=0.0)`
  when `not air_graph.subgraphs`. +test. (Agents 2-B1, 3-M2.)
- **F3 — Fix `offers.rate_family` DDL comment.** Says `flat|mfb|coload|per_uld_pivot` (introduced this session
  trimming a ruff line) but `RateFamily` values are `flat_rate|min_flat_breaks|coload_per_kg|per_uld_pivot`; the
  adapter would `RateFamily("flat")→ValueError`. Restore canonical values verbatim. (Agent 2-B2.)
- **F4 — `route_legs.supply_ref_type`: add `offer`, drop `ondemand_arc`/`rate_card_lane`** (no backing tables;
  carried from critique 09). (Agents 1-M6, 5-#7.)
- **F5 — Add direct `order_route` unit test** (shuffled chain / singleton / disconnected-fallback branch);
  currently only tested indirectly. (Agent 3-M3.)
- **F6 — Add an INFEASIBLE-status test** for `air_milp` (or document the branch as structurally unreachable).
  (Agent 3-M1.)

---

## SCHEMA / BUILD-ORDER FIXES (clearer-cut; confirm as a batch)

- **S1 — MILP hash-stable build order** (Agent 4-B2). `ArcId`/`HawbId`/`GroupKey` are `str`; variable creation,
  constraint build, and objective `sum(terms)` iterate frozensets/dicts → `PYTHONHASHSEED`-dependent column/row
  order and float-accumulation order → solver may return different equally-optimal incumbents across processes.
  Fix: iterate `sorted(...)` at the build sites (`air_milp.py:245`, the `z`/`mawbs` build, `members`, objective
  terms). Pairs with F1. Needs an experiment to measure divergence magnitude, but the fix is cheap and right.
- **S2 — `leg_actuals`: drop/relabel `realized_dep_h` + `realized_arrival_h`; keep `realized_block_h`** (the only
  truly frozen quantity). Realized departure/arrival are walk-derived (depend on upstream clock, path/policy-
  dependent) — persisting them invites the naive-sum bug §4 forbids. (Agents 1-M1, 4-M2.)
- **S3 — `component_actuals`: align `arc_type` to canonical `ArcType.value` strings** (`"customs"`≠`customs_dwell`,
  `"delivery"`≠`final_delivery`) AND widen the PK — a HAWB has origin+dest CFS dwell and per-hub dwell, so
  `PK (hawb_id, arc_type)` is too coarse; add `hub_code`/side discriminator or make hub-dwell a deterministic
  function of `(hawb_id, hub, seed)`. (Agents 1-M2, 2-M4.)
- **S4 — `route_legs` air-billing payload** so the scorer can recompute realized BSA cost: add `offer_id`,
  `mawb_group_key`, and a small `billing_json` (realized η ULD counts, CW, pivot/over, rate_family) — or a
  dedicated billing table. Without it `metrics.total_cost` is not reconstructable from rows, and the L2
  reshuffle/fallback split can't be computed. (Agent 1-B3, ties to Agent 5 L2 gap.)
- **S5 — ULD physical caps: one source of truth + `uld_types` reference table.** `RateCatalog.uld_types` is a
  GLOBAL dict, but the schema stores caps per-allocation; `Offer.uld_max_*` (predicate-8 fit) vs
  `RateCatalog.uld_types` (C.5b) vs `air_uld_allocations.uld_w/v` are three homes that can disagree. Add a
  `uld_types(code PK, w_kg, v_cbm)` table; document `Offer.uld_max_*` as the deliberately-coarser fit screen.
  Also: `BsaContract.cap` (per-arc C.5c) has **no column** anywhere — latent round-trip failure (generator
  doesn't set it today; the spike must assert). (Agents 1-lossless, 2-M3.)
- **S6 — Time-origin contract.** Offers/legs are absolute UTC-hours; DB columns are sim-hours. Today they align
  only because the fixture's epoch is 0. generator-to-files must normalize (subtract epoch) or assert epoch==0,
  stated as an invariant. (Agent 2-M1.)
- **S7 — Generator HAWB id adopts `det_id`.** Generator emits `gen-{seed}-{i}`; `det_id` is
  `gen-{seed}-hawb-{i}`. generator-to-files should use the canonical factory for `shipments.id`. (Agent 2-det_id.)

---

## DOC RECONCILIATION (data_model.md drift — confirmed unfixed)

The Session-31 hardening was applied to `scenario_db.py` but **not** mirrored in `data_model.md`, so canonical
and schema-module now disagree on the entire air-supply representation. The "one-module Postgres swap" claim is
currently false. (Agent 1-M3/M4/M5.)
- D-D1: `air_uld_allocations` still has the old per-lane-window shape (`departure_days TEXT[] NOT NULL`,
  `contracted_rate_per_kg`, `effective_from/to`) — update to the per-offer model.
- D-D2: no `offers`/`offer_legs`/`gateways`/`hubs` tables in `data_model.md` — promote them.
- D-D3: `data_model.md §5` + `scenario_io_and_replay.md:118` still reference `spot_rate_quotes`/`spot_rate_snapshots`
  — reflect the rate-on-offer fold; add a `rate_json` shape-per-family note.

---

## DEFERRED COVERAGE DEBT (ships WITH the unbuilt component, not "later")

- Binding-capacity + mid-horizon-tender **conservation fixture** (no-double-spend; cross-step `tendered`
  monotonicity + same-step free-return) → **2c orchestrator**. "Single most important missing test." (Agents 3, 5.)
- **Two-solves → identical `route_legs`** reproducibility test (depends on F1) → replay-loop/scenario writer.
- **`C_hind ≤ M1`** per-draw test on a binding-capacity instance → hindsight solve + scorer.
- **Predicate-9** (tier-reliability admission) wiring — `route_reliability` producer exists, no graph-side
  consumer; needs admit-at-z=0/reject-at-high-z_tier test → graph-gen. (Agents 2, 3-M4.)
- **OTP against the FROZEN promise:** scorer must `JOIN booking_promise` (never live `shipments`); pytest
  `realized.on_time == (realized_arrival_h <= booking_promise.promised_deadline_at)`. (Agent 5.)
- **L2 split inputs:** leg-cost reconciliation invariant (`Σ route_legs.act_cost (final) == realized.realized_cost`)
  + per-HAWB feasible-route catalog to size `C^fallback = 2× max feasible real route` → scorer. (Agent 5-L2.)
- **Scorer reads frozen actuals only, never a live RNG** — the central CRN invariant; add a test asserting no
  `random.Random` on the replay/score path + scorer-run-twice byte-identical `realized`. (Agent 4-CRN.)

---

## DEFERRED TO generator-to-files (determinism — Agent 4; must land WITH that module)

These are NOT yet fixed and are easy to lose — the in-memory Stage-2a generator predates the file
layer the spec replaces, so they bite the moment generator-to-files is built on top of it:
- **G1 (M-1) — generator uses one shared `random.Random(seed)`** (`air_generator.py:242`), not the
  named sub-streams. generator-to-files must draw demand/leg-actuals/component-actuals/spot from
  their own `rng_stream(seed, …)` so moving one (κ,λ) axis doesn't shift another stream's draws (CRN).
  **`RNG_STREAMS` has no `"rates"` member** — add one (or reuse `spot_regime`) since `rng_stream`
  raises on unknown names.
- **G2 (M-3) — pre-sample in a canonical sorted order** (by `flight_id`, by `(hawb_id, arc_type)`),
  one draw advanced from the dedicated stream, so frozen actuals are enumeration-order-independent.
- **G3 (m-3) — write `cap_uld` JSON with `sort_keys=True`** (the column comment says sort_keys; the
  writer must honor it). Same for any JSON the generator emits.
- **G4 (S7) — adopt `det_id` for `shipments.id`** (generator emits `gen-{seed}-{i}`; PK should be
  `gen-{seed}-hawb-{i}`).

## DEFERRED TO the scorer (Agent 5; beyond the L2-split debt already listed)

- **SC1 — `route_legs.act_*` are optional/denormalized**, not the authoritative arrival source. The
  scorer computes realized arrival A from `leg_actuals` + `component_actuals` via the §4 walk and
  writes `realized.realized_arrival_h`; document this so no one trusts `act_arrival_h`.
- **SC2 — empty route must be a scorer ERROR, not an instant on-time arrival** (Agent 2 m2):
  `sample_route([])` returns `(ready_h, on_time)`; the scorer must assert non-empty final routes.

## MINOR / acknowledged — judgment calls, NOT changing now (logged so they're not "missed")

- **`flex_denominator.cw_k` stays REAL** (Agent 1 m1). It's a reporting denominator (`L2/cw_flex`),
  not part of the integer conservation identity, so REAL is correct; integer-kg applies to the ledger.
- **`effective_deadline_at` (OTP Δ_k) vs `soft_deadline_h` (C.10 tardiness knee) are intentionally
  distinct** (Agent 1 m3) — one is the on-time bound, the other the PWL penalty breakpoint. Keep both.
- **`ulds_per_departure` == `N_{a,u}`** holds because each offer materializes one departure (Agent 1
  lossless). The round-trip spike asserts it rather than a rename.
- **`rate_json` shape-per-family** (Agent 1 M5): `flat_rate`→`{m,min_chg}`, `min_flat_breaks`→
  `{breaks:[[threshold,rate],…]}`, `coload_per_kg`→`{m}`, `per_uld_pivot`→NULL. The spike's persist/load
  adapter is the executable spec for this; documented here as the contract.

## What's verified SOUND (no action)
- Output lineage `executions→runs→planning_runs→routes→route_legs`; conservation CHECK (exact INTEGER);
  run-identity; reveal view + parameterized `visible_shipments(t)`; `flex_denominator` scenario-scoped (D-F7).
- `scenario_db` determinism pins implemented correctly (FK pragma every connection, no wall-clock/random
  defaults, `det_id`/`rng_stream`, JSON sort_keys, fresh-DB regen). The `executed_at`/`uuid4` provenance carve-out
  is the sanctioned exception, correctly scoped to `executions`.
- MILP **value-bound** test discipline is exemplary (every binding case pinned to a hand-computed total).
- `order_route` tie-break is deterministic; no timing assertions anywhere (CLAUDE.md-compliant).
- RateCatalog is reconstructable for the M1–M3 families currently exercised (holes: global `uld_types`,
  `BsaContract.cap`, `ulds_per_departure`==`N_{a,u}` naming — all sidestepped by today's generator; spike must assert).

---

## Recommended sequence
1. Resolve **D1–D4** (directional).
2. Apply **F1–F6** (mechanical, esp. F1 determinism pin + F2 empty-book — both on the live path).
3. Apply **S1–S7** schema/build-order fixes (S1 pairs with F1; S2–S5 are the input-layer grain fixes the spike
   must validate).
4. Reconcile **data_model.md** (D-D1–D-D3).
5. THEN the round-trip spike — now covering a **through-offer** HAWB (≥2 legs under one arc), the three
   reconstruction holes (S5), and the billing payload (S4) — before generator-to-files freezes the format.
6. Deferred-debt tests land with their components (2c / scorer / graph-gen predicate-9).
