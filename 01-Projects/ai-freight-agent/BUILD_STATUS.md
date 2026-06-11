# BUILD STATUS — AI Multimodal Freight Routing Agent

**Last refreshed:** 2026-06-10 (Session 33 — λ arrival-stream generator + 2-FLEX BUILT; 4-agent build critique run)

**How to use this doc.** The canonical dashboard of the full plan: what is built, what is
left. **Read it first on session start.** **Refreshed FULLY at every sign-off — a full
rewrite, never an append; delete stale lines and keep it clean.** Detailed plans/reasoning live
in the pointers at the bottom and in SESSION_LOG.md (read only the last entry — it's large) /
CONTEXT.md (RESUME HERE).

---

## Current position

- **Strategy:** go vertical on **AIR** to the **replan-savings proof** (the load-bearing number in
  `product_thesis.md`); the substrate it forces into existence is reused by every mode.
- **Phase:** 2 (Component Builds), air slice.
- **Just finished (Session 33):** the **λ arrival-stream generator + 2-FLEX**, in four green slices
  (191 passed, ruff clean): (1) **2-FLEX core** `src/components/flexibility.py` (`Tier` / single-source
  `TierSpec` / `classify` / `cw_flex`, ordering invariants at import); (2) **daily substrate**
  `build_tpeb_daily(D=7)` (tiles TPEB offers at 24h, `build_tpeb_instance` untouched); (3) **λ arrival
  stream** `air_generator.py` Part B (`ArrivalConfig`/`generate_arrival_instance`/`HawbArrival`; `d*`
  cutoff-anchored `known_at`; tier-derived `Δ_k` via new `air_graph.earliest_arrival`; headline lateness
  tier-INDEPENDENT, D-A9); (4) **persistence** (`scenario_io` arrival columns + `write_arrival_scenario`;
  reveal view works). Then ran the **methodology-§11 4-agent BUILD critique → `docs/critique/12`**.
- **Critique-12 result:** soundness core CLEAN in code (D-A9 / D-A13 walk≡scalar / no lookahead / no
  double-spend / CRN). **3 convergent BLOCKING** (F1 κ-axis still the retired integer ULD count; F2 cutoff
  anchor broken; F3 no symmetric null → D-A17) + MATERIAL (F4 only 2/6 lanes capacitated & demand-starved;
  F5 `cw_flex` t=0 persistence gap; F6 DEFERRED slack manufactures flexibility) + MINOR (F7 tractability;
  F8 `t_dead_prob` CRN). **No code fixes applied yet** — paused for user go.
- **▶ Next: (a) rewrite the F1/F2/F3 numeric walkthroughs clearer** (user found them unclear — `docs/critique/12`
  § Numeric walkthroughs), **then (b) fold F1 → F4 → F2 → F3/D-A17** (gates 2c) → build 2c replay loop.
- **Honest calendar to the air proof:** ~12–16 working sessions remaining.
- **Quality:** **191 passed, ruff clean** (2026-06-10). ~5.3K real component LOC.

---

## Open items awaiting user

- **Go-ahead on the critique-12 fold** (the only thing paused). Sequence proposed: rewrite F1/F2/F3 clearer →
  F1 continuous-κ dial (κ = peak-concurrent-demand ÷ contracted slots) → F4 capacitate an origin-diverse lane
  (`FlatRate.cap` is plumbed, unset) + `lane_mix` knob + M-B5 roll option → F2 cutoff `= dep − L_cut` & anchor
  `d*` to the binding contracted leg → F3 → methodology **D-A17** (τ effect-size floor + `cell_role` guard).
  F1/F2 are directional (they change how the load-bearing instance is built) — confirm before implementing.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved (edited S29; PDF not recompiled) |
| G-Method | **Arrival-only replan methodology** (`model/arrival_only_replan_methodology.md` v0.1) | ✓ approved (S32 — governing) |
| G-Method | Backtest methodology (`model/backtest_methodology.md` v0.5) | ✓ approved |
| G-Method | Air transit-time (`model/air_transit_time.md` v0.3) | ✓ approved (air deterministic `s=0`) |
| G-Method | Flexibility model 2-FLEX (`model/flexibility_model.md` v0.3) | ✓ approved |
| G-Method | Scenario IO & replay (`docs/design/scenario_io_and_replay.md` v0.2 +S31 hash-pin) | ✓ approved |
| G-Method | Human heuristic H₀ (`model/human_planning_heuristic.md`) | ✓ spec approved (not built) |
| G-Isolation | Air graph + MILP + transit-time (2b) + generator (2a) + scenario_db + scenario-IO + **2-FLEX + λ arrival stream** | ✓ passed |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ deferred

| Component | Phase | Status | Notes / pointer |
|---|---|---|---|
| Air graph generator (`src/components/air_graph.py`) | 2 | ✓ | construction + integration-validated; `order_route`; **+S33 `earliest_arrival`** (A_k^min edge) |
| Air MILP optimizer (`src/components/air_milp.py`) | 2 | ✓ | M1–M6; solver seed pin, empty-book guard, hash-stable column order |
| Synthetic generator — air 2a (`data/synthetic/air_generator.py`) | 2 | ✓ | `AirInstance` + `write_scenario`; **+S33 Part B λ arrival stream** (`ArrivalConfig`/`generate_arrival_instance`/`write_arrival_scenario`) |
| **2-FLEX (`src/components/flexibility.py`)** | 2 | ✓ | **NEW S33** — Tier/TierSpec/derive_deadline/classify/cw_flex; +23 tests. `flex_k`/`cw_flex` t=0 wiring still deferred (F5) |
| **Daily substrate (`build_tpeb_daily`)** | 2 | ✓ | **NEW S33** — D=7 tiling; +6 tests |
| Air transit-time 2b (`src/components/air_transit_time.py`) | 2 | ✓ | deterministic for air (`s=0`); stochastic path kept for ocean |
| `scenario_db` schema seam (`src/scenario_db.py`) | 2 | ✓ | full schema + reveal view; arrival columns populated S33 |
| Scenario-IO adapter (`data/synthetic/scenario_io.py`) | 2 | ✓ | `persist`(+arrivals)/`load`/`persist_actuals` |
| Replay orchestrator 2c | 2 | ☐ | **BLOCKED on the critique-12 fold (F1/F2/F4/F3-D-A17).** Then M₀ pin-prior-soft + `M₁'` arm + recourse fixtures |
| Arms: H₀ / M₀ / M₁ / M₁' / π_hind | 2 | ☐ | M₀ incremental-greedy vs M₁ open-book re-opt; batch-cutoff H₀ headline |
| Scorer + Replan-savings backtest (Stage 3) | 2 | ☐ | κ×λ band + α-frontier; **the proof** |
| Ocean FCL / LCL / Trucking optimizers | 2 | ☐ | models drafted, not approved; Stage 4 (ocean = asymmetry test) |
| Path-level TT / destination leg / rules engine; generic graph gen (2.1) | 2 | ⏸ | Stage 4 |
| Multimodal stitching | 3 | ☐ | after all modes pass isolation |
| MCP server / Agent loop / UI surfaces / L3–L4 | 4–6 | ☐ | UI = the ~+56K LOC cliff, last |

---

## Near-term critical path (the air proof) — ordered

0. ✓ **λ arrival-stream + 2-FLEX DONE (S33)** — 4 slices, 191 passed; 4-agent build critique run (`docs/critique/12`).
1. ☐ **Clarity rewrite** — rewrite `docs/critique/12` § Numeric walkthroughs F1/F2/F3 clearer/step-by-step
   (user-flagged). First action S34.
2. ☐ **Fold the critique-12 BLOCKING/MATERIAL set (gates 2c), in order F1 → F4 → F2 → F3:**
   - **F1** — replace the integer κ proxy (`n_uld=max(1,round(2·scale))`) with continuous binding-ness
     (κ = peak-concurrent-demand ÷ contracted slots; size BSA `cap_a` continuously; `n_uld` billing-only).
   - **F4** — capacitate ≥1 origin-diverse lane (set `FlatRate.cap`) + add `lane_mix` to `ArrivalConfig` +
     emit the M-B5 roll-to-next-contract option; report L2 by lane.
   - **F2** — derive cutoffs as `dep − L_cut` (L_cut>0); anchor `d*` to the binding contracted leg.
   - **F3** — methodology **D-A17**: pre-registered symmetric null (negative-control `|L2|<CI`; peak-cell CI
     lower bound > τ AND reshuffle-share ≥50%; `L2(κ)` monotone) + `cell_role` guard (`tier_coupled⇒upper_bracket`).
   - **F8** (cheap, anytime) — always-consume the `t_dead` uniform so `t_dead_prob` stays CRN.
3. ☐ **2c replay loop + arms** — M₀ incremental-greedy vs M₁ open-book re-opt + `M₁'` control (`C(M₁')==C(M₀)`);
   one time-scalar SoT (D-A13); global conservation + 2-arc reshuffle fixture + 3 disruption-recourse fixtures;
   **F5** persist frozen t=0 `cw_flex` + D-F7 arm-invariance pytest; **F7** window-prune subgraph to ±2d + gated
   warm-start (behind the `M₁'` invariant).
4. ☐ **Negative control + null + baseline** (per D-A17) — required abundant×early `|L2|<CI` cell; batch-cutoff H₀.
5. ☐ **Scorer + Stage 3 outputs** — running-clock walk from `leg_actuals`+`component_actuals`; OTP vs FROZEN
   `booking_promise`; L1/L2 + `L2_reshuffle` gated headline; → resolves the thesis number.

**Calibration note (pre-Stage-3, gating the headline):** source `L_cut` (F2), DEFERRED `sla_offset` slack (F6),
contracted-vs-spot share & which lanes (F4); + the 2a distribution-provenance note (lane structure vs BTS FAF).

**Post-proof broaden (Stage 4), in order:** Ocean FCL (**the asymmetry test**; G-LaTeX first) → LCL + Trucking →
path-level TT / destination leg / rules engine (+ generic Graph Generator 2.1 gate) → multimodal stitching → MCP →
agent → UI (the ~+56K LOC cliff) → L3/L4.

---

## Built & verified (quality state)

- **Test suite last green:** 2026-06-10 (Session 33) — **191 passed** in ~6s, ruff clean across src/tests/data.
- **Built components:** `air_graph.py` (+`earliest_arrival`), `air_milp.py`, `air_transit_time.py`,
  `scenario_db.py`, `data/synthetic/air_generator.py` (+ Part B λ stream / `write_arrival_scenario`) +
  `tpeb_air_instance.py` (+ `build_tpeb_daily`) + `scenario_io.py` (+ arrival columns) + **`src/components/flexibility.py`**
  (real HiGHS, never mocked).
- **New S33 tests:** `tests/components/test_flexibility.py` (+23), `tests/test_tpeb_daily.py` (+6),
  `tests/test_arrival_stream.py` (+11), `tests/test_arrival_persistence.py` (+4).

---

## Key locked decisions (pointers, not duplicated)

- **Input layer = Option A** (S31): one `scenario.db` holds inputs+outputs; offers carry their rate; `offer_legs`
  chains physical legs; reference tables; per-HAWB ground scalars on `shipments`. → `docs/design/scenario_db_erd.md`.
- **One offer = one rate; one lane → many offers** (sourced). → memory `reference_air_offer_rate_cardinality`.
- **Capacity = ULD-slot-only for the proof** (D1) — MILP binds at the BSA allotment tier. *(NOTE: the κ-as-`n_uld`
  dial is being replaced per critique-12 F1 — continuous binding-ness, not integer ULD count.)*
- **MCT ≠ dwell** (D2); ledger `arc_id` = `{offer_id}:{uld_type}` (D3).
- **Determinism** = HiGHS `threads=1`+`random_seed` + sorted column order + `PYTHONHASHSEED=0` for cross-process
  byte-identity; named RNG sub-streams (G1).
- **Arrival-only proof (S32, governing)** = the ONLY stochastic process is **demand arrival**; transit DETERMINISTIC;
  predicate-9/z_tier/σ̂ retired for air → `A ≤ Δ_k`. Arms M₀ greedy / M₁ open-book / `M₁'` control / batch H₀;
  L2=M₀−M₁ (reshuffle-gated headline). Sweep κ×λ; D-A9 independent-arrival headline; null + negative-control cell.
  → `model/arrival_only_replan_methodology.md` (§10 arrival, §12 D-A9..D-A16). Ocean (Stage 4) revives stochastic transit.
- **2-FLEX built (S33)** = `TierSpec` single source (sla_offset/z_tier/w_sp, `[CAL]` 12-40-120 / 2-1-0.5 / 4-2-1);
  `Δ_k = A_k^min + sla_offset(tier)`; `flex_k` = ≥2 θ-separated, non-dominated, on-time options. → `model/flexibility_model.md` v0.3.

---

## Deferred / parked (do not lose)

- **Critique-12 fold (the live queue)** — F1 continuous-κ, F2 cutoffs+binding-anchor, F3→D-A17, F4 lane capacity+`lane_mix`+M-B5,
  F5 frozen-`cw_flex`+D-F7 pytest, F6 DEFERRED-slack sandbag, F7 tractability, F8 CRN. → `docs/critique/12`.
- **`flex_k`/`cw_flex` t=0 wiring** — compute once at generation over the pre-9 option set *with cost*, persist as a
  scenario-level scalar; the component is built and arm-invariant, the caller is not (F5).
- **Forwarder scale-up stress test** (after the proof passes) — small → medium forwarder; tractability + signal hold. §11.
- **Disruption-recourse fixtures (3)** — absorbable / connection-break / cancellation; built WITH 2c.
- **Single-consignee direct-delivery bypass**; **dest cartage at US gateways**; leg `ac_type`/`lithium_ok`/`embargoed_cargo`
  columns; `corpus` first-class rows; per-MAWB-break hub-dwell attribution; `model/capacity_manager.md` stub (L3).
- **Standing review agents** (calibration / interface-seam / backtest red-team): last full run S29; **next due ~S36**.
- **2a distribution-calibration provenance note** — Stage-3 precondition.
- **Generic Graph Generator (2.1)** deliberately skipped for air; gate retired at Stage 4 (re-triggers if 2c
  re-pruning outgrows air's embedded rules).
- ocean refining-ETA/cancellations — Stage 4 (stochastic transit apparatus revives).

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard — refreshed fully each sign-off |
| `CONTEXT.md` | compressed context + RESUME HERE |
| `SESSION_LOG.md` | running per-session history (read last entry only) |
| `docs/critique/12-simulation-build-review-brief.md` | **4-agent BUILD critique (S33)** — F1..F8 + § Numeric walkthroughs (clarity-TODO) + fold sequence |
| `docs/critique/11-simulation-design-review.md` | 4-agent sim-DESIGN review (S32) → D-A9..D-A16 |
| `docs/design/scenario_io_and_replay.md` | SQLite scenario IO + deterministic replay |
| `EXECUTION_PLAN.md` | canonical phase/gate framework (whole product) |
| `product_thesis.md` | four-layer thesis + the load-bearing replan-savings claim |
| `model/arrival_only_replan_methodology.md` | **governing** proof methodology (v0.1) — arrival-only, deterministic transit |
| `model/backtest_methodology.md` / `flexibility_model.md` / `air_transit_time.md` / `human_planning_heuristic.md` | proof method / 2-FLEX / 2b / H₀ specs |
| `PRD.md` + appendices | strategic index, capabilities, competitive |
