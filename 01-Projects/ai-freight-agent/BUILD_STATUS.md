# BUILD STATUS — AI Multimodal Freight Routing Agent

**Last refreshed:** 2026-06-15 (Session 38 — **critique-17 fully triaged + cleared; M₀/M₁′ decomposition
reshaped (Reading B); F1 Slice C, N3 `ReplayState`, and 2c-1 MILP pinning BUILT. 278 passed, ruff clean.**
Both S37 BLOCKING closed: BLK-1 mitigated (600s time-limit + incumbent), BLK-2 dismissed (predicate-8 non-bug).
Now inside 2c — the replay-loop proof machinery.)

**How to use this doc.** The canonical dashboard of the full plan: what is built, what is left. **Read it first on
session start.** **Refreshed FULLY at every sign-off — full rewrite, never an append; delete stale lines, keep it
honest.** Detail: SESSION_LOG.md (last entry) / CONTEXT.md (RESUME HERE) / the critique doc pointers below.

---

## Current position

- **Strategy:** go vertical on **AIR** to the **replan-savings proof** (the load-bearing number in `product_thesis.md`).
- **Phase:** 2 (Component Builds), air slice. The graph-gen workstream + F1 (Slices A/B/C) are built; the proof
  machinery (**2c replay loop**) is now in progress — N3 state owner + 2c-1 pinning done.
- **Just finished (Session 38):** (1) **M₀/M₁′ decomposition reshape (Reading B)** — M₀ greedy / M₁′ single-pass-optimal
  / M₁ open-book; chain `C(H₀)≥C(M₀)≥C(M₁′)≥C(M₁)≥C(π_hind)`; **L1 = C(H₀)−C(M₁′)**, **headline L2 = C(M₁′)−C(M₁)**
  (intra-engine, artifact-free); retired the `C(M₁′)==C(M₀)` leakage placebo; all governing docs reconciled.
  (2) **BLK-1 mitigated** (600s HiGHS time-limit + best incumbent; `mip_gap` reported; real tractability deferred).
  (3) **BLK-2 dismissed** (predicate-8 guarantees routable HAWBs fit 1 ULD → existing bound dominates).
  (4) **critique-17 cheap fixes + doc drift** (MAT-4/MIN-3/MIN-4/MAT-5/MIN-2/MAT-2/MAT-3; MIN-1 φ=1.3 confirmed
  load-bearing). (5) **F1 Slice C** (spot cap C.5d, both billing styles, + generator + persistence). (6) **N3
  `ReplayState`** (clock + declarative-reconcile ledger + `tendered_set`). (7) **2c-1** (`solve(pinned=…)`).
- **▶ Next (S39):** **2c-2** populate `tender_at` (= binding cutoff, D-A1; currently UNSET) → **2c-3** replay-loop
  skeleton (M₁ open-book on `ReplayState`) → **2c-4** M₀ + M₁′ arms → **2c-5** scorer → **2c-6** H₀ + π_hind →
  **2c-7** recourse fixtures → **Stage 3** the (κ,α,λ) sweep = the thesis number.
- **Quality:** **278 passed, ruff clean** (2026-06-15). No open BLOCKING.

---

## Open items awaiting user

- None blocking. (Decisions locked this session: Reading-B decomposition + L1=H₀−M₁′; BLK-1 = time-limit+incumbent;
  spot = both billing styles capped, REAL units, κ-independent `spot_regime` draw; N3 ledger = declarative reconcile.)
- `[CAL]` to source later: `_base_transit_h` lane table (~p90 achievable), `sla_offset_h` 12/24/48, `corridor_phi=1.3`
  (load-bearing — wants the deferred φ sweep), spot cap band (`U(1,3)×1500kg`) + two-sided rate band (`×U(.85,1.18)`),
  seed_radius/k, κ ladder, α grid + external anchor, density mix, drayage $/km + km/h + door bbox.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved (PDF behind; tex-reconcile deferred) |
| G-Method | Arrival-only replan methodology (`model/arrival_only_replan_methodology.md`) | ✓ §13 v4 + **S38 Reading-B revisions** (§4/D-A10/D-A11/D-A23) |
| G-Method | **M₀/M₁′/M₁ decomposition (Reading B)** — L1=H₀−M₁′, headline L2=M₁′−M₁ | ✓ locked S38 (all docs reconciled) |
| G-Method | D-F6 v2 SLA deadline / Backtest v0.5 / Air TT v0.3 / 2-FLEX / Scenario IO&replay v0.2 | ✓ approved |
| G-Review | Standing review agents (calibration / seam / red-team) | ✓ S36 (`docs/critique/14/15/16`); next due S43 |
| G-Review | S37 graph-gen critique (`docs/critique/17`) | ✓ **fully triaged & cleared S38** |
| G-Review | N3 `ReplayState` independent design review | ✓ run S38 (3 BLOCKING found in v1 → all fixed before build) |
| G-Isolation | Air graph + MILP (+spot cap +pinning +time-limit) + 2a/2b + scenario_db/IO + 2-FLEX + λ + Slice A + FreightNet + geo_select + **F1 Slice C + N3 ReplayState + 2c-1** | ✓ passed (278) |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ deferred

| Component | Phase | Status | Notes / pointer |
|---|---|---|---|
| `FreightNet` (`src/freightnet.py`) | 2 | ✓ | freight-node ref DB + spatial service; +MIN-4 `start_km≤max_km` guard. |
| `geo_select` (`src/components/geo_select.py`) | 2 | ✓ | corridor + frontier candidate selection; φ=1.3 load-bearing (MIN-1, recorded at the knob); +`seed_radius≤max_radius` guard. |
| Air graph generator (`src/components/air_graph.py`) | 2 | ✓ | geo candidates + `FallbackPolicy`; +MAT-5 empty-seed raise. |
| **Air MILP optimizer (`src/components/air_milp.py`)** | 2 | ✓ | M1–M6 + **C.5d spot cap** (both billing styles) + **`pinned` (2c-1, D-A11)** + **600s time-limit/incumbent (BLK-1)** + `mip_gap`. BLK-2 closed (non-bug). |
| Synthetic generator — air (`data/synthetic/air_generator.py`) | 2 | ✓ | geo build chain; supply draw (κ/α); **spot_regime draw (Slice C: cap + two-sided rate)**; MAT-4 t_dead floor + MIN-3 Δ_k<T^abs assert; D-F6 v2. |
| `flexibility.py` (2-FLEX) | 2 | ✓ | D-F6 v2 `committed_deadline`; v1-drift scrubbed (MIN-2). |
| `scenario_db` / `scenario_io` | 2 | ✓ | + `spot_capacity_ledger` (REAL) table; spot_cap persists via rate_json. |
| **`ReplayState` (`src/replay.py`)** | 2 | ✓ | NEW (N3). Clock (in-mem monotonic, parameterized reveal) + declarative-reconcile ledger (INT contracted + REAL spot) + `tendered_set` pin hook + runs registration. |
| **2c replay loop / arms / scorer** | 2 | ◐ | **2c-1 pinning DONE.** Next: 2c-2 tender_at → 2c-3 M₁ loop → M₀/M₁′ → scorer → H₀/π_hind → recourse fixtures. |
| **Stage 3 — (κ,α,λ) sweep + L1/L2 decomposition** | 3 | ☐ | the thesis number; blocked on 2c. |
| Ocean FCL / LCL / Trucking; path-TT / rules / multimodal stitch; MCP/agent/UI | 2–6 | ☐/⏸ | later stages |

---

## Near-term critical path — ordered

1. ☐ **2c-2 — `tender_at` wiring** (= binding cutoff, D-A1; currently unset in generator/persistence).
2. ☐ **2c-3 — replay-loop skeleton (M₁ open-book)** — `run_replay` on `ReplayState`: reveal → solve(pin tendered) →
   extract per-arc allocation → reconcile → tender at cutoff; record snapshots.
3. ☐ **2c-4 — M₀ (greedy) + M₁′ (single-pass, pin priors)** arms.
4. ☐ **2c-5 — scorer** — deterministic `air_transit_time §4` replay with FROZEN actuals → realized arrival/OTP/cost/fallback.
5. ☐ **2c-6 — H₀ heuristic + π_hind** (degenerate single-cycle, no tender lock).
6. ☐ **2c-7 — recourse fixtures** (methodology §6: absorbable / connection-break / cancellation).
7. ☐ **Stage 3 — (κ,α,λ) sweep** + L1/L2 decomposition; loose-corner `|L2|<CI` gate; born-at-risk fraction diagnostic.

---

## Built & verified (quality state)

- **Test suite last green:** 2026-06-15 (S38) — **278 passed** (was 255), ruff clean across src/tests/data.
- **Real HiGHS, never mocked.** Cross-process determinism of the full generate→geo-build→solve path proven.
- **New tests S38:** `test_replay.py` (13 — clock/reveal/RNG + ledger reconcile/conservation/two-arm isolation);
  +spot-cap (coload & flat) + spot-regime (escape-hatch + κ-stability) + pinning + time-limit-contract tests.
- **No open BLOCKING.** (BLK-1 mitigated by the cap; BLK-2 was a non-bug.) The cost chain holds per-draw only for
  OPTIMAL solves — a TIME_LIMIT solve can transiently violate it (a tractability artifact, documented, not a bug).

---

## Key locked decisions (pointers, not duplicated)

- **Reading B decomposition (S38):** M₀ greedy / M₁′ single-pass-optimal / M₁ open-book; L1=H₀−M₁′ (M₀ internal
  ablation), headline L2=M₁′−M₁ (intra-engine) → `arrival_only_replan_methodology.md §4` + D-A11/D-A23.
- **BLK-1 (S38):** 600s HiGHS time-limit + best incumbent (`MilpParams.time_limit_s`); real fix deferred.
- **F1 Slice C:** spot = both coload & flat/MFB capped on CW (uncapped = infinite escape hatch, D-A19); cap drawn
  `U(1,3)×1500kg`, two-sided rate `base×U(.85,1.18)`, both on κ-independent `spot_regime` stream.
- **N3 ledger:** declarative per-cycle reconcile; two units/tables (INT contracted, REAL spot ε-conservation).
- **D-F6 v2 / supply⟂demand / no cost-dominance pruning / geographic candidate selection** — unchanged (S35–S37).

---

## Deferred / parked (do not lose)

- **MIN-1 φ sensitivity sweep** — confirm the corridor isn't pruning load-bearing routes; needs L2, so **gated on 2c**.
- **Recourse fixtures** (methodology §6) — built WITH the 2c loop (2c-7), out of the headline scenario.
- **N4 disruption-sensitivity arm / N5 L2%-primary / N6 π_hind floor** — Wave 4 (`docs/critique/13`).
- **N11 carrier deny/blacklist**; **F5 `flex_k`/`cw_flex` t=0 wiring** (with 2c); `load()` partial-inverse for arrivals.
- **Tex reconcile** (PDF one compile behind — `model/air_freight_routing.pdf` left unstaged; do NOT auto-compile).
- ocean refining-ETA/cancellations — Stage 4.
- **Memory to add S39:** Reading-B decomposition + L1=H₀−M₁′ (new `project_*` record).

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard — refreshed fully each sign-off |
| `CONTEXT.md` | compressed context + RESUME HERE |
| `SESSION_LOG.md` | running per-session history (read last entry) |
| `model/arrival_only_replan_methodology.md` | governing proof methodology (§13 v4 + S38 Reading-B revisions) |
| `model/backtest_methodology.md` / `precommitted_sla_deadline_proposal.md` / `flexibility_model.md` | arms/decomposition / D-F6 v2 / 2-FLEX |
| `src/replay.py` | N3 `ReplayState` — the 2c state owner |
| `docs/critique/17-graph-gen-session-review-s37.md` | S37 critique (fully triaged S38) |
| `docs/critique/13-integration-and-framework-review.md` | 7-agent review (N3 done; N4–N6 live for Wave-4) |
| `EXECUTION_PLAN.md` / `product_thesis.md` / `PRD.md` + appendices | phase framework / thesis / strategic index |
