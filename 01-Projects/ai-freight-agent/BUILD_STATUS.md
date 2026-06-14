# BUILD STATUS — AI Multimodal Freight Routing Agent

**Last refreshed:** 2026-06-14 (Session 37 — graph-gen workstream BUILT (FreightNet + geo candidate selection + fallback
redesign + candidate-path retirement + D-F6 v2 SLA deadlines); **255 passed, ruff clean.** Ran a 4-agent critique
(`docs/critique/17`): **B5 is NOT robustly resolved — REOPENED** (seed-variance tail to ∞); **1 correctness BLOCKING**
(BSA fallback under-bound). Both logged for user review tomorrow; nothing fixed in response yet.)

**How to use this doc.** The canonical dashboard of the full plan: what is built, what is left. **Read it first on
session start.** **Refreshed FULLY at every sign-off — full rewrite, never an append; delete stale lines, keep it
honest.** Detail: SESSION_LOG.md (last entry) / CONTEXT.md (RESUME HERE) / the critique doc pointers below.

---

## Current position

- **Strategy:** go vertical on **AIR** to the **replan-savings proof** (the load-bearing number in `product_thesis.md`).
- **Phase:** 2 (Component Builds), air slice. The graph-gen workstream is built; the replay sweep is blocked on B5
  (tractability) which is **REOPENED** after the S37 critique.
- **Just finished (Session 37):** the full graph-gen workstream — `FreightNet` (network layer), `geo_select`
  (corridor + flight-frontier candidate selection), build-time per-HAWB fallback (`FallbackPolicy`), **full retirement
  of the hardcoded-region candidate path** (generation is door-only; candidates resolve at build), and the **D-F6 v2**
  pre-committed tier×lane SLA deadline methodology change. Then ran a **4-agent critique** (`docs/critique/17`).
- **▶ Next (S38) — triage `docs/critique/17` (user reviews tomorrow), ordered:** (1) **B5 tractability** strategy
  (time-limit+incumbent / solver tuning / smaller n / tighter φ) — REOPENED; (2) **BSA fallback under-bound** fix
  (volume-aware `n_uld_solo`) + dominance invariant test; (3) cheap calibration safety (t_dead floor, `Δ_k<T^abs`
  assert, extend-radius guard, loud empty-seed); (4) doc drift (retract "second-order"; v1-formula scrub); (5) φ
  sensitivity sweep. **Then:** Slice 5 — replay orchestrator / (κ,α,λ) sweep; F1 Slice C.
- **Quality:** **255 passed, ruff clean** (2026-06-14) — but with **2 open BLOCKING findings** (correctness + tractability)
  the suite does not catch.

---

## ⚠ Open BLOCKING (from critique-17 — review tomorrow)

- **BLK-1 — B5 NOT resolved (tractability).** Proof cell n=15/d7, HiGHS threads=1, no time limit: seed0=23s,
  seed1=132s, **seed2 >5min did not finish.** The corridor cut LP size (103→~37 arcs, real) but NOT the
  consolidation/capacity branching hardness (seed-dependent, unbounded). The "21.2s RESOLVED" headline was a lucky
  seed. **B5 is REOPENED.** Fix deferred: HiGHS time-limit + incumbent, MIP-gap/warm-start/heuristic tuning, accept
  n<15, or tighter φ for the sweep.
- **BLK-2 — `air_milp.air_leg_cost_ub` under-bounds BSA per_flight** (`n_uld_solo` counts ULDs by weight only; the real
  cargo is volume-bound, and the MILP bills ULD count by volume too). Fallback can be under-priced → can strand a
  routable HAWB (`feedback_no_standalone_cost_pruning`). Fix: volume-aware `n_uld_solo` + a fallback-dominance test.

---

## Open items awaiting user

- **Review `docs/critique/17` and pick the S38 triage** (see Current position ▶ Next).
- F1/D-F6 `[CAL]` to source later: `_base_transit_h` lane table (calibrate to ~p90 lane achievable transit),
  `sla_offset_h` 12/24/48, `corridor_phi=1.3` (load-bearing — wants a sensitivity sweep), seed_radius/k, κ ladder,
  α grid + external anchor, spot two-sided band, density mix, drayage $/km + km/h + door bbox.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved (PDF behind; tex-reconcile deferred) |
| G-Method | Arrival-only replan methodology (`model/arrival_only_replan_methodology.md`) | ✓ v0.1 + §13 v4 (S35) |
| G-Method | **D-F6 v2 pre-committed tier×lane SLA deadline** (`model/precommitted_sla_deadline_proposal.md`) | ✓ approved S37 |
| G-Method | Backtest (`backtest_methodology.md` v0.5) / Air TT (`air_transit_time.md` v0.3) / 2-FLEX (`flexibility_model.md` v0.3, deadline → D-F6 v2) | ✓ approved |
| G-Method | Scenario IO & replay (`docs/design/scenario_io_and_replay.md` v0.2) | ✓ approved |
| G-Review | Standing review agents (calibration / interface-seam / backtest red-team) | ✓ S36 (`docs/critique/14/15/16`); next due S43 |
| G-Review | **S37 graph-gen critique** (correctness / methodology / seam / red-team) | ✓ run (`docs/critique/17`); 2 BLOCKING open |
| G-Isolation | Air graph + MILP + 2a/2b + scenario_db/IO + 2-FLEX + λ stream + Slice A supply + **FreightNet + geo_select + geo-pipeline** | ✓ passed (255) |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ⚠ done-with-open-blocking · ☐ not started · ⏸ deferred

| Component | Phase | Status | Notes / pointer |
|---|---|---|---|
| **`FreightNet`** (`src/freightnet.py`) | 2 | ✓ | NEW. Freight-node reference DB (`freight_nodes`, all 9 types; airports seeded = 3,274 real from OurAirports CSV asset) + spatial service (`nodes_within_km` / `nearest` / `extend_until_reachable` / `allowed_ids`); `load_freightnet` build-if-missing. |
| **`geo_select`** (`src/components/geo_select.py`) | 2 | ✓ | NEW. Per-door candidate selection: k-NN seeds + detour-corridor ellipse (φ) + exhaustive bidirectional flight-frontier (H_max). Confirmed NOT a dominance prune. |
| Air graph generator (`src/components/air_graph.py`) | 2 | ◐ | + `FallbackPolicy` / `_max_path_cost` / `_hawb_fallback_cost` (fallback added LAST, per-HAWB sized) + `resolve_geo_candidates`/`resolve_geo_subgraphs` + `allowed_air_pairs` air-arc restriction + `Hawb` door coords. |
| Air MILP optimizer (`src/components/air_milp.py`) | 2 | ⚠ | + `air_leg_cost_ub` (fallback pricing) — **BLK-2: under-bounds BSA per_flight (volume).** M1–M6 otherwise intact. Slice C (spot CW-cap) still ☐. |
| Synthetic generator — air (`data/synthetic/air_generator.py`) | 2 | ◐ | **doors-only** generation (region candidate path RETIRED); nominal gateway via FreightNet; `build_geo_air_graph` (the geo build chain); `_base_transit_h` [CAL]; D-F6 v2 `committed_deadline`. |
| `flexibility.py` (2-FLEX) | 2 | ◐ | `derive_deadline`→`committed_deadline` (D-F6 v2); `sla_offset_h` 12/40/120→**12/24/48**; `classify` takes `base_transit_h`, handles born-at-risk slack<0. |
| `scenario_db` (`src/scenario_db.py`) | 2 | ◐ | candidate JSON columns DROPPED; door-coord columns added. |
| `scenario_io` (`data/synthetic/scenario_io.py`) | 2 | ◐ | door persist/load; candidate + persisted `fallback_cost` DROPPED (`SimInputs` lost it). |
| 2b transit-time / `tpeb_air_instance` | 2 | ✓ | tpeb keeps the scalar `fallback_cost` back-compat API (hand-built tests). |
| **F1 Slice C — spot + route-based fallback (MILP)** | 2 | ☐ | per §13/D-A19 |
| Replay orchestrator 2c / Arms / Scorer + (κ,α,λ) sweep | 2 | ☐ | the proof; blocked on B5 (BLK-1) + N3 ReplayState |
| Ocean FCL / LCL / Trucking; path-TT / rules / multimodal stitch; MCP/agent/UI | 2–6 | ☐/⏸ | later stages |

---

## Near-term critical path — ordered

1. ⚠ **Triage `docs/critique/17`** (user review tomorrow) — BLK-1 (B5 tractability), BLK-2 (BSA UB), then MATERIAL/MINOR.
2. ☐ **B5 tractability strategy** (REOPENED) — time-limit+incumbent / solver tuning / n / φ.
3. ☐ **F1 Slice C** — spot per-arc CW-cap + two-sided price + route-based fallback.
4. ☐ **Wave 3 — N3 `ReplayState` owner** (clock + per-arm ledger + RNG) before 2c.
5. ☐ **2c replay loop + arms** — M₀ competent single-pass (D-A23) vs M₁ open-book + `M₁'`. (+ scorer-build notes:
   positive control, report L2 distribution/CI; **born-at-risk fraction diagnostic** from critique-17 MAT-3.)
6. ☐ **Scorer + (κ,α,λ) sweep** → the thesis number. Loose-corner `|L2|<CI` gate; α as a curve.

---

## Built & verified (quality state)

- **Test suite last green:** 2026-06-14 (S37) — **255 passed**, ruff clean across src/tests/data.
- **Real HiGHS, never mocked.** Cross-process (PYTHONHASHSEED) determinism of the full generate→geo-build→solve path
  proven (`test_determinism.py`).
- **New tests S37:** `test_freightnet.py`, `components/test_geo_select.py`, `components/test_geo_select_integration.py`,
  `components/test_fallback_policy.py`, `test_geo_pipeline.py` (+ rewired generated-instance tests to the geo path).
- **Caveat:** 255 green does NOT clear the 2 BLOCKING (BLK-1 tractability is a solve-time tail; BLK-2 is a cost-bound
  gap that surfaces only on specific BSA/volume instances) — see critique-17.

---

## Key locked decisions (pointers, not duplicated)

- **D-F6 v2:** Δ_k = ready + base_transit(lane) + sla_offset(tier) — pre-committed, graph-free → `precommitted_sla_deadline_proposal.md`.
- **Build-time geographic candidate selection** (corridor + frontier) replaces the hardcoded region; candidate path retired → `project_graph_generation_vision`.
- **No cost-based dominance pruning** (geo_select honors this — exhaustive within corridor) → `feedback_no_standalone_cost_pruning`.
- **Supply ⟂ demand** (§13 v4) → `project_supply_independent_of_demand`.
- **Fallback** = per-HAWB max-UB-path × 1.5 (build-time), trivial when stranded; arrives at T^abs (max tardiness).

---

## Deferred / parked (do not lose)

- **critique-17 MATERIAL/MINOR** (review tomorrow): proof-neutrality "second-order" retract; born-at-risk p90 oversold +
  fraction diagnostic; t_dead floor; empty-seed loud; φ load-bearing (sensitivity sweep); doc drift (v1 formula in
  flexibility_model §0/§3/§6 + generator docstrings); `committed_deadline` invariant assert; extend-radius guard.
- **N3 ReplayState / N4 disruption sensitivity / N5 L2%-primary / N6 π_hind** — Wave 4 (`docs/critique/13`).
- **N11 carrier deny/blacklist**; **F5 `flex_k`/`cw_flex` t=0 wiring** (with 2c).
- **Seam (S36):** `load()` partial inverse for arrival scenarios; `spot_regime` stream unconsumed until Slice C.
- **Tex reconcile** (retired predicate-9 in `.tex`; PDF one compile behind — `model/air_freight_routing.pdf` left unstaged).
- ocean refining-ETA/cancellations — Stage 4.

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard — refreshed fully each sign-off |
| `CONTEXT.md` | compressed context + RESUME HERE |
| `SESSION_LOG.md` | running per-session history (read last entry) |
| `docs/critique/17-graph-gen-session-review-s37.md` | **S37 4-agent critique — review tomorrow (2 BLOCKING open)** |
| `model/precommitted_sla_deadline_proposal.md` | D-F6 v2 deadline methodology (APPROVED S37) |
| `model/arrival_only_replan_methodology.md` | governing proof methodology (§13 v4) |
| `model/flexibility_model.md` | 2-FLEX + deadline (D-F6 v2) |
| `docs/critique/13-integration-and-framework-review.md` | 7-agent review (N1..N18; N3–N6 live for 2c/Wave-4) |
| `config/forwarder_graph_config.json` | forwarder graph-gen config (backstop + geo + drayage knobs) |
| `EXECUTION_PLAN.md` / `product_thesis.md` / `PRD.md` + appendices | phase framework / thesis / strategic index |
