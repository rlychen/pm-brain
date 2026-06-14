# BUILD STATUS — AI Multimodal Freight Routing Agent

**Last refreshed:** 2026-06-14 (Session 36 — F1 Slice A + B1–B4 BUILT; standing review agents run; **B5 found
region→region intractable at the proof cell** → next workstream is the NETWORK LAYER + geographic graph-gen. 205
passed, ruff clean.)

**How to use this doc.** The canonical dashboard of the full plan: what is built, what is left. **Read it first on
session start.** **Refreshed FULLY at every sign-off — a full rewrite, never an append; delete stale lines and keep it
clean.** Detail lives in the pointers at the bottom + SESSION_LOG.md (last entry only — it's large) / CONTEXT.md
(RESUME HERE).

---

## Current position

- **Strategy:** go vertical on **AIR** to the **replan-savings proof** (the load-bearing number in `product_thesis.md`).
- **Phase:** 2 (Component Builds), air slice. F1 (independent network-supply + region→region) is **mostly built**;
  one tractability blocker (B5) stands between it and the replay sweep.
- **Just finished (Session 36):** F1 **Slice A** (independent network-supply draw) + the 3 **standing review agents**
  + F1 **Slice B B1–B4** (region→region routing) + determinism/round-trip fixes + a forwarder-level backstop config.
- **▶ Next (S37), ordered:** **(1) build `FreightNet`** (the network layer — name LOCKED, data model + service; DB
  tables of physical freight nodes — airports / ocean ports / rail / ICD / CFS / border / FTZ / barge, with full address
  + type + lat/lon + city + country + codes). **(2) graph-gen review (sim vs real)** → geographic on-the-fly propagation
  over FreightNet, which **also fixes B5**. **(3) F1 Slice C** (spot CW-cap + two-sided price + route-based fallback). Then 2c.
- **Quality:** **205 passed, ruff clean** (2026-06-14).

---

## The B5 blocker (open)

Region→region as built is **intractable at the proof cell**: real HiGHS solve times — static n=20 = 2.6s; arrival
n=10/days=7 = ~60s; **arrival n=15/days=7 > 5 min** (214 MAWB binaries, avg 103 arcs/subgraph). The replay sweep solves
this instance thousands of times (× seeds × (κ,α,λ) × arms), so this is fatal. **Root cause:** every HAWB gets all 9
airport pairs (hardcoded full region set) × boardable-days × ~15 offers/day. **The 720→168h backstop did NOT fix it**
(graph size unchanged — backstop is a backward/deadline prune; the blowup is forward). **Cost-based dominance pruning
REJECTED** (strands HAWBs under tight supply). **The fix = geographic candidate selection** (the network-layer +
graph-gen workstream): give each HAWB only the airports geographically near its door — safe, distance-based, ~4× cut.

---

## Open items awaiting user

- (resolved S36) Network-layer name = **`FreightNet`** (data model + service).
- F1 `[CAL]` to source later (not blocking): κ ladder, α concentration grid + **external anchor for α** (calibration
  agent's top finding — report L2 as a curve across a BTS-FAF-anchored α band, not one peak cell), supply distribution,
  spot two-sided multiplier band, density mix, trucking $/km + km/h + door bbox, slot-density(333)-vs-billing(167) note.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved (PDF compile behind; tex-reconcile deferred) |
| G-Method | **Arrival-only replan methodology** (`model/arrival_only_replan_methodology.md`) | ✓ v0.1 + §13 v4 (S35) |
| G-Method | Backtest (`backtest_methodology.md` v0.5) / Air TT (`air_transit_time.md` v0.3, `s=0`) / 2-FLEX (`flexibility_model.md` v0.3) | ✓ approved |
| G-Method | Scenario IO & replay (`docs/design/scenario_io_and_replay.md` v0.2) / Human heuristic H₀ (spec) | ✓ approved |
| G-Review | Standing review agents (calibration / interface-seam / backtest red-team) | ✓ run **S36** (`docs/critique/14/15/16`); next due **S43** |
| G-Isolation | Air graph + MILP + 2b + generator(2a) + scenario_db/IO + 2-FLEX + λ stream + **Slice A supply + region→region B1–B4** | ✓ passed |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ deferred

| Component | Phase | Status | Notes / pointer |
|---|---|---|---|
| Air graph generator (`src/components/air_graph.py`) | 2 | ◐ | + **region→region (D-A24)**: `Hawb.origin/dest_candidates`, trucking "diamond" chains, `_origin_pol_nodes` dispatch, `check_consolidation_coherence`. Hash-order sorts added (determinism). |
| Air MILP optimizer (`src/components/air_milp.py`) | 2 | ◐ | M1–M6 + **per-tail Δ^post (N7 fix)** + sorted constraint assembly (determinism). **Slice C** adds spot CW-cap + route fallback. |
| Synthetic generator — air (`data/synthetic/air_generator.py`) | 2 | ◐ | + **Slice A** supply draw (κ+α multinomial, `supply` stream) + **region→region** (haversine trucking matrix, region candidate sets, `_facility_gateway` actuals key) + **forwarder backstop config**. `DEMAND_LANES` retired. |
| `tpeb_air_instance.py` topology | 2 | ◐ | + **SFO** 3rd dest gateway (`cx_hkg_sfo` contracted + `br_tpe_sfo`). |
| `scenario_db` (`src/scenario_db.py`) | 2 | ◐ | + `"supply"` RNG stream + `origin/dest_candidates_json` shipments columns. |
| `scenario_io` (`data/synthetic/scenario_io.py`) | 2 | ◐ | + candidate-matrix persist/load (`_load_candidates`). |
| 2-FLEX / 2b transit-time / scenario-IO adapter | 2 | ✓ | unchanged this session |
| **`FreightNet` (network layer)** | 2 | ☐ | **NEXT (S37).** Physical freight-node DB + topology service (data model + "nodes within X km"); prerequisite for real graph-gen. |
| **Geographic graph-gen** (replaces hardcoded candidate set) | 2 | ☐ | over FreightNet; **fixes B5 tractability** |
| **F1 Slice C — spot + route-based fallback (MILP)** | 2 | ☐ | per §13/D-A19 |
| Replay orchestrator 2c / Arms / Scorer + (κ,α,λ) sweep | 2 | ☐ | the proof; after Slice C + N3 ReplayState |
| Ocean FCL / LCL / Trucking; path-TT / rules engine; multimodal stitch; MCP/agent/UI | 2–6 | ☐/⏸ | later stages |

---

## Near-term critical path (the air proof) — ordered

1. ☐ **`FreightNet`** (S37 first) — physical freight-node DB tables + topology service (data model + service).
2. ☐ **Graph-gen review + geographic propagation** — replaces hardcoded candidate airports; **fixes B5**.
3. ☐ **F1 Slice C** — spot per-arc CW-cap `Σ cw_k·x ≤ cap^spot` + two-sided price + `1.5×` route-based fallback.
4. ☐ **Wave 3 — N3 `ReplayState` owner** (clock + per-arm ledger + RNG) before 2c.
5. ☐ **2c replay loop + arms** — M₀ competent single-pass (D-A23) vs M₁ open-book + `M₁'`; conservation + reshuffle +
   disruption-recourse fixtures. **Add (scorer-build notes from S36 review): a hand-computed POSITIVE control + report
   the L2 distribution/CI** (not a §13 change; R1/R2 residue).
6. ☐ **Scorer + (κ,α,λ) sweep** → the thesis number. Loose-corner `|L2|<CI` gate + L2=0-fraction + per-airport
   binding-rate diagnostics; **α reported as a curve (calibration finding)**.

---

## Built & verified (quality state)

- **Test suite last green:** 2026-06-14 (Session 36) — **205 passed**, ruff clean across src/tests/data.
- **Real HiGHS, never mocked.** Cross-process determinism proven hash-seed-independent (region→region exposed + fixed
  several latent set/frozenset hash-order leaks feeding HiGHS sums + the actuals presampler).
- **New tests S36:** `tests/test_network_supply.py` (supply draw), `tests/components/test_air_graph_region.py`
  (region→region routing / dispatch / coherence).

---

## Key locked decisions (pointers, not duplicated)

- **Supply ⟂ demand** (§13 v4) → `project_supply_independent_of_demand`. **Region→region (D-A24)** built.
- **No cost-based dominance pruning** (strands HAWBs under tight supply) → `feedback_no_standalone_cost_pruning`.
- **Network-layer-first then geographic graph-gen** → `project_graph_generation_vision`.
- **Backstop** = forwarder config (`config/forwarder_graph_config.json`, `backstop_buffer_h=168`).
- **Arrival-only proof / deterministic transit / 2-FLEX / determinism pins** — unchanged (see CONTEXT).

---

## Deferred / parked (do not lose)

- **R1/R2 (S36 red-team) — minor, scorer-build notes** (NOT §13 changes): add a positive control + report L2
  distribution/CI. **Calibration: α needs an external anchor** before headlining a cell.
- **Seam audit MATERIAL (deferred):** `load()` partial inverse for arrival scenarios; Δ_k in two columns
  (`soft_deadline_h` vs `effective_deadline_at`). `spot_regime` stream unconsumed until Slice C.
- **N3 ReplayState / N4 disruption sensitivity / N5 L2%-primary / N6 π_hind_locked** — Wave 4 (`docs/critique/13`).
- **N11 carrier deny/blacklist layer**; **F5 `flex_k`/`cw_flex` t=0 wiring** (with 2c).
- **Tex reconcile** (retired predicate-9 still in `.tex`; PDF one compile behind — left unstaged).
- **`_MAX_AIR_LEGS_PROXY=2`** in `compute_fallback_cost` — make graph-derived if a route can exceed 2 air legs.
- ocean refining-ETA/cancellations — Stage 4.

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard — refreshed fully each sign-off |
| `CONTEXT.md` | compressed context + RESUME HERE (S37 = network layer first) |
| `SESSION_LOG.md` | running per-session history (read last entry only) |
| `model/arrival_only_replan_methodology.md` | **governing** proof methodology (§13 v4) |
| `docs/critique/14/15/16-*-s36.md` | S36 standing review (calibration / interface-seam / backtest red-team) |
| `docs/critique/13-integration-and-framework-review.md` | 7-agent review (N1..N18; N3–N6 live for 2c/Wave-4) |
| `config/forwarder_graph_config.json` | forwarder-level graph-gen config (backstop) |
| `EXECUTION_PLAN.md` / `product_thesis.md` / `PRD.md` + appendices | phase framework / thesis / strategic index |
