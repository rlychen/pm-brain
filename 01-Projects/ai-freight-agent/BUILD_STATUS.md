# BUILD STATUS — AI Multimodal Freight Routing Agent

**Last refreshed:** 2026-06-13 (Session 35 — F1 REDESIGN: independent network-supply model + region→region routing; `arrival_only_replan_methodology.md` §13 APPROVED v4. No code.)

**How to use this doc.** The canonical dashboard of the full plan: what is built, what is
left. **Read it first on session start.** **Refreshed FULLY at every sign-off — a full
rewrite, never an append; delete stale lines and keep it clean.** Detailed plans/reasoning live
in the pointers at the bottom and in SESSION_LOG.md (read only the last entry — it's large) /
CONTEXT.md (RESUME HERE).

---

## Current position

- **Strategy:** go vertical on **AIR** to the **replan-savings proof** (the load-bearing number in
  `product_thesis.md`); the substrate it forces into existence is reused by every mode.
- **Phase:** 2 (Component Builds), air slice — about to start **F1** against the **redesigned** proof
  methodology (`arrival_only_replan_methodology.md` §13 v4, APPROVED S35).
- **Just finished (Session 35 — no code):** a user-requested pre-F1 critique turned into a **ground-up redesign
  of the proof's supply/demand model** (4 critique rounds; §13 written v1→v4, approved). Two reversals: the
  original F1 (weight-only `cap_a = peak_demand/κ`) **can't ration volume-limited cargo** AND was **circular**
  (built supply from the demand it generated). New model: **supply generated independently of demand across the
  network** (integer ULD positions; κ=network tightness + α placement; mismatch IS the value source) +
  **region→region routing LOCKED (D-A24)**.
- **▶ Next (F1, redesigned):** build the independent network-supply draw (κ+α integer multinomial on a new
  `supply` RNG stream) + CRN test FIRST (fail-fast), then region→region multi-O/D subgraphs + per-airport
  trucking matrix, then the spot CW-cap + route-based fallback MILP changes. **The old "continuous κ" F1 and the
  S34 Wave-2 `[CAL]` knobs are SUPERSEDED.**
- **Honest calendar to the air proof:** ~13–18 working sessions remaining (F1 is now bigger — region→region is a
  scope expansion; then 2c replay + arms + scorer + Stage 3).
- **Quality:** **193 passed, ruff clean** (2026-06-11; unchanged S35 — no code touched).

---

## Open items awaiting user

- **None blocking.** §13 v4 approved as governing. F1 `[CAL]` to source later (not blocking the build): κ ladder,
  α concentration grid, the supply distribution, the spot two-sided multiplier band, the density mix, the
  per-airport trucking matrix values.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved (PDF one compile behind; stale on retired predicate-9 — tex-reconcile deferred) |
| G-Method | **Arrival-only replan methodology** (`model/arrival_only_replan_methodology.md`) | ✓ approved — **v0.1 (S32) + §13 v4 capacity/supply refinement (S35)** |
| G-Method | Backtest methodology (`model/backtest_methodology.md` v0.5) | ✓ approved |
| G-Method | Air transit-time (`model/air_transit_time.md` v0.3) | ✓ approved (air deterministic `s=0`) |
| G-Method | Flexibility model 2-FLEX (`model/flexibility_model.md` v0.3) | ✓ approved |
| G-Method | Scenario IO & replay (`docs/design/scenario_io_and_replay.md` v0.2 +S31 hash-pin) | ✓ approved |
| G-Method | Human heuristic H₀ (`model/human_planning_heuristic.md`) | ✓ spec approved (not built) |
| G-Isolation | Air graph + MILP + 2b + generator(2a) + scenario_db/IO + 2-FLEX + λ arrival stream | ✓ passed (pre-redesign) |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## §13 v4 — the governing F1 design (APPROVED S35)

- **Supply ⟂ demand.** Per-flight contracted capacity = **integer ULD positions**, drawn from a NEW `supply` RNG
  sub-stream, never reading demand. **κ** = network tightness (`total_N = round(E[Σ SE_k]/κ)`, `E[Σ SE_k]` =
  analytic no-consolidation slot mean `max(w/1500,v/4.5)` — closed-form, zero demand coupling). **α** = Dirichlet
  concentration; per-flight ~ `Multinomial(total_N, Dirichlet(α))`. Per-lane tightness EMERGES. κ a coarse
  integer ladder at proof scale.
- **D-A24 (LOCKED) — region→region routing.** Origin/dest airport + lane + flight = optimizer decisions via a
  per-airport trucking matrix + multi-O/D subgraphs + tractability re-check. Fixed-lane `DEMAND_LANES` retired.
- **Contracted gate = existing 2D C.5/C.5b** (feed integer `N_f`); the original volume defect dissolves. D-A20/21
  (SE/cap_a/suppress-C.5b) WITHDRAWN. CW billing only; assert `BsaContract.cap=={}`.
- **3 supply sources/flight:** contracted (C.5b) / spot (NEW per-arc `Σ cw_k·x ≤ cap^spot`, two-sided price
  base×m) / fallback (1.5× graph-derived worst-spot-route; amends D-A12).
- **Falsifiability:** loose corner of (κ,α,λ) sweep gated `|L2|<CI`; regret floor = self-check only. **M₀ (D-A23)**
  = competent single-pass baseline (deterministic within-cycle consolidation, pins priors). Report L2=0 fraction +
  per-airport binding-rate + post-consolidation occupancy.

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ deferred

| Component | Phase | Status | Notes / pointer |
|---|---|---|---|
| Air graph generator (`src/components/air_graph.py`) | 2 | ◐ | built + S34 fixes; **F1 will add region→region multi-O/D subgraphs + airport-pair-specific arc keys (D-A24)** |
| Air MILP optimizer (`src/components/air_milp.py`) | 2 | ◐ | M1–M6 built; **F1 adds spot per-arc CW-cap constraint + route-based fallback; contracted gate REUSES C.5/C.5b (feed integer N_f)** |
| Synthetic generator — air (`data/synthetic/air_generator.py`) | 2 | ◐ | built; **F1 rewrites supply path: independent integer network draw (κ+α multinomial, new `supply` stream); region-O→D demand + per-airport trucking matrix; density mix; capped two-sided spot; retire `capacity_scale`/`n_uld`-as-κ** |
| `scenario_db` (`src/scenario_db.py`) | 2 | ◐ | built; **F1 adds `"supply"` to `RNG_STREAMS`** |
| 2-FLEX (`src/components/flexibility.py`) | 2 | ✓ | Tier/derive_deadline/classify/cw_flex. t=0 wiring deferred (with 2c) |
| Daily substrate (`tpeb_air_instance.py`) | 2 | ◐ | `build_tpeb_daily`; will extend to the region→region airport grid |
| Air transit-time 2b (`src/components/air_transit_time.py`) | 2 | ✓ | deterministic for air (`s=0`); stochastic path kept for ocean |
| Scenario-IO adapter (`data/synthetic/scenario_io.py`) | 2 | ✓ | persist/load/persist_actuals |
| **F1 — independent network-supply + region→region** | 2 | ☐ | **NEXT.** §13 v4. Supersedes the old "continuous κ" F1 + S34 Wave-2 fold. |
| Replay orchestrator 2c | 2 | ☐ | after F1 + the N3 `ReplayState` owner. M₀ greedy / M₁ open-book / `M₁'` / π_hind; recourse fixtures |
| Arms: H₀ / M₀ / M₁ / M₁' / π_hind | 2 | ☐ | M₀ = competent single-pass (D-A23); batch-cutoff H₀; **N6 π_hind_locked** |
| Scorer + Replan-savings backtest (Stage 3) | 2 | ☐ | (κ,α,λ) sweep + loose-corner `|L2|<CI` gate; reshuffle decomposition; **the proof** |
| Ocean FCL / LCL / Trucking optimizers | 2 | ☐ | models drafted, not approved; Stage 4 (ocean = asymmetry test) |
| Path-level TT / rules engine; generic graph gen (2.1) | 2 | ⏸ | Stage 4 |
| Multimodal stitching | 3 | ☐ | after all modes pass isolation |
| MCP server / Agent loop / UI surfaces / L3–L4 | 4–6 | ☐ | UI = the ~+56K LOC cliff, last |

---

## Near-term critical path (the air proof) — ordered

0. ✓ **§13 v4 redesign + approval (S35):** supply ⟂ demand, integer network draw, region→region locked.
1. ☐ **F1 slice A — independent network-supply draw** (FIRST ACTION S36, fail-fast): add `"supply"` to
   `RNG_STREAMS`; κ+α integer multinomial draw of per-flight `N_f` off the new stream; `E[Σ SE_k]` analytic mean;
   retire `capacity_scale`/`n_uld`-as-κ. **CRN test: vary κ/α ⇒ demand draw byte-identical** (hard gate).
2. ☐ **F1 slice B — region→region (D-A24):** per-airport trucking-cost matrix on the HAWB; multi-O/D subgraph
   construction (airport-pair-specific arc IDs); consolidation-coherence invariant (every MAWB arc = one
   airport-pair flight, riders reach tail via priced trucking); tractability re-check.
3. ☐ **F1 slice C — spot + fallback MILP:** new per-arc spot CW-cap constraint `Σ cw_k·x ≤ cap^spot`; two-sided
   spot pricing (base×m); route-based fallback (1.5× graph-derived worst-spot-route); assert `BsaContract.cap=={}`;
   reuse C.5/C.5b for the contracted gate.
4. ☐ **Wave 3 — N3 `ReplayState` owner** (clock + per-arm capacity ledger + RNG sub-streams) before 2c.
5. ☐ **2c replay loop + arms** — M₀ competent single-pass (D-A23, deterministic tie-break) vs M₁ open-book + `M₁'`
   control (`C(M₁')==C(M₀)`); conservation + reshuffle fixture + 3 disruption-recourse fixtures.
6. ☐ **Wave 4 — claim-framing folds** — N4 disruption sensitivity; N5 `L2%` primary; N6 π_hind_locked; loose-corner
   `|L2|<CI` gate; L2=0-fraction + per-airport binding-rate + post-consolidation occupancy diagnostics.
7. ☐ **Wave 5 — e2e tests** (`docs/design/e2e_test_plan.md`): Tier-1 identities + Tier-2 use-case matrix (some
   cases need the region→region generator first).
8. ☐ **Scorer + Stage 3 outputs** → resolves the thesis number.

---

## Built & verified (quality state)

- **Test suite last green:** 2026-06-11 (Session 34) — **193 passed**, ruff clean across src/tests/data. **No code
  touched S35** (methodology-only).
- **Real HiGHS, never mocked.** Determinism proven hash-seed-independent cross-process.
- **Built components:** `air_graph.py`, `air_milp.py`, `air_transit_time.py`, `scenario_db.py`, `flexibility.py`,
  `data/synthetic/{air_generator,tpeb_air_instance,scenario_io}.py`. **F1 will modify air_graph / air_milp /
  air_generator / scenario_db / tpeb_air_instance per §13 v4.**

---

## Key locked decisions (pointers, not duplicated)

- **Supply ⟂ demand (S35, governing F1 design):** §13 v4 of `arrival_only_replan_methodology.md`; memory
  `project_supply_independent_of_demand`. Supersedes the old "continuous κ = peak_demand/κ on `BsaContract.cap`".
- **Input layer = Option A** (S31): one `scenario.db` holds inputs+outputs. → `docs/design/scenario_db_erd.md`.
- **Arrival-only proof (governing)** = the ONLY stochastic process is **demand arrival**; transit DETERMINISTIC.
  → `model/arrival_only_replan_methodology.md` (§10 arrival, §12 D-A9..D-A16, **§13 v4 capacity/supply**).
- **Fallback (S35):** `1.5 × graph-derived worst-spot-route` (`1.5·[top_spot·CW·max_air_legs + ground]`), dominant
  + well-conditioned; replaces the S34 `2× worst-route` / old $1M (amends D-A12).
- **Determinism** = HiGHS `threads=1`+`random_seed` + sorted column order; proven hash-seed-independent.
- **2-FLEX** = `TierSpec` single source; `Δ_k = A_k^min + sla_offset(tier)`. → `model/flexibility_model.md` v0.3.

---

## Deferred / parked (do not lose)

- **N3 `ReplayState` owner** (clock + per-arm ledger + RNG) before 2c. **N4 disruption sensitivity / N5 L2%-primary
  / N6 π_hind_locked** — Wave 4. → `docs/critique/13`.
- **N11 carrier deny/blacklist layer** — graph enforces allow-set only; deny cascade is a later slice.
- **F5 `flex_k`/`cw_flex` t=0 wiring** — with 2c.
- **Tex reconcile** — retired predicate-9 still in `air_freight_routing.tex`; PDF one compile behind.
- **Forwarder scale-up stress test** (after the proof passes; also where the integer-κ ladder becomes a smooth
  (κ,α) plane) — §11 of the methodology; **disruption-recourse fixtures (3)** (built WITH 2c).
- **Standing review agents** (calibration / interface-seam / backtest red-team): last full run S29; **next due
  ~S36** (now overdue — run alongside F1).
- **F1 `[CAL]` to source** (pre-Stage-3): κ ladder, α grid, supply distribution, spot multiplier band, density mix,
  per-airport trucking matrix values.
- Single-consignee direct-delivery bypass; leg `ac_type`/`lithium_ok`/`embargoed_cargo` persisted columns; `mct_h`
  real source; `model/capacity_manager.md` stub (L3).
- ocean refining-ETA/cancellations — Stage 4 (stochastic transit apparatus revives).
- **The S34 Wave-2 fold (F1 continuous-κ / F4 / F3 / D-A17) is RETIRED** — superseded by §13 v4. Some critique-13
  Wave-0/1 *code* fixes already landed (S34) and stand; the Wave-2 *design* is gone.

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard — refreshed fully each sign-off |
| `CONTEXT.md` | compressed context + RESUME HERE |
| `SESSION_LOG.md` | running per-session history (read last entry only) |
| `model/arrival_only_replan_methodology.md` | **governing** proof methodology — v0.1 (§10 arrival, §12 D-A9..16) + **§13 v4 (S35) independent network-supply + region→region** |
| `docs/critique/13-integration-and-framework-review.md` | 7-agent integration/framework review (S34) — N1..N18 (N3–N6 still live for 2c/Wave-4) |
| `docs/design/e2e_test_plan.md` | two-tier e2e test plan (Wave 5) |
| `docs/design/scenario_io_and_replay.md` | SQLite scenario IO + deterministic replay |
| `EXECUTION_PLAN.md` | canonical phase/gate framework (whole product) |
| `product_thesis.md` | four-layer thesis + the load-bearing replan-savings claim |
| `model/backtest_methodology.md` / `flexibility_model.md` / `air_transit_time.md` / `human_planning_heuristic.md` | proof method / 2-FLEX / 2b / H₀ specs |
| `PRD.md` + appendices | strategic index, capabilities, competitive |
