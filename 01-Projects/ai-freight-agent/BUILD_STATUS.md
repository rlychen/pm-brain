# BUILD STATUS — AI Multimodal Freight Routing Agent

> ⛔ **HARD RULE: TARDINESS PENALTY ALWAYS ON.** Never run any air model (probe/test/calibration/sweep)
> with `tardiness_weight_scale=0`. The quadratic C.10 penalty (calibrated express/standard/deferred) must
> be live 100% of the time — a W=0 run is a BUG, not a result. Full objective always (cost + tardiness +
> fallback). See `CLAUDE.md` top + memory `feedback_tardiness_penalty_always_on`.

**Last refreshed:** 2026-07-11 (Session 54 — **2D SPOT RESERVATION designed, verified, slice S1 BUILT.
386 green.**) Grounded the three spot rate-families (6-agent research: flat/MFB collapse to one channel;
decay is a channel property so the code was already right; reserve-then-assign grounded for spot+coload).
Designed the **2D (weight+volume) spot reservation** that replaces locked D-A1 — a run's spot usage
becomes a committed, non-decaying reservation; swap HAWBs until cutoff; **sunk-cost / free-reserved-
capacity** cost model. Verified across **2 rounds / 7 code-reading agents**; folded all corrections into
spec **REV 3**. User dismissed the one residual (NC-a) as a domain non-issue ⇒ `penalty_frac=1` stands.
**Built slice S1** (schema split `spot_cap`→`spot_wcap`/`spot_vcap`, byte-identical). Branch
`s54-2d-spot-reservation`.
Prior S53: 2026-07-10 (timestamp remodel Phase 0+1). Prior S52: 2026-07-09 (reserve/assign design).

**How to use this doc.** Canonical built/remaining dashboard — read first on session start. Governing
detail: **spec `docs/design/spot_reservation_2d_design_s54.md` (REV 3)** · memory
`project_s54_2d_spot_reservation` (resume anchor) · SESSION_LOG.md S54 entry · CONTEXT.md (RESUME HERE).

---

## Current position

- **Strategy:** go vertical on **AIR** to the replan-savings proof. The proof rides a synthetic generator
  whose supply/demand tightness + capacity-decay + arrival/commit timing must be realistic.
- **Phase:** 2 (Component Builds), air slice — **2D spot reservation: DESIGN APPROVED + REV-3 verified;
  slice S1 (schema) BUILT, 386 green.** This is the S51/S52 commit-timing fix, now concrete: spot the
  plan uses becomes a committed 2D `(weight, volume)` reservation that doesn't decay; both replan arms
  reserve (symmetric floor → M1′ no longer force-dumps); the only M1-vs-M1′ difference is swap fluidity.
- **Next:** slice **S2** (`ReplayState._reserved_spot` + usage ratchet), then S3–S7 (below).

---

## The S54 workstream — 2D spot reservation (headline)

**The fix (spec `spot_reservation_2d_design_s54.md` REV 3, approved-for-build):** replace the single
`tender` (D-A1) with a per-tier commit model. Spot reserves a **2D physical envelope `(r^w, r^v)`**:
- **Reserve early** — a run's spot usage becomes a committed reservation (monotone ratchet), floors the
  booking-curve decay so held space survives to later runs (the timing fix).
- **Assign late** — swap which HAWBs fill it until identity-lock = `min(doc cutoff, ACAS pre-load)`.
- **Cost = sunk-cost / free-reserved-capacity** — the reservation is sunk when booked; reserved space is
  **free at the margin** thereafter, so the optimizer fills it rather than stranding it (kills the L2
  artifact). Total committed spot = `family_cost(peak reserved CW)`. `penalty_frac=1` (v1, no cancel).
- **2D cap** — `C^w` = the live lane-supply value (unchanged); `C^v = C^w/333.33` (LD3 geometry). Honest
  ~2× tighter volume (our 120–240 kg/m³ cargo is volume-bound) → recalibrate operating point via τ.

**Grounding (6 agents):** flat/MFB = one rating channel; decay is a channel property (code already
right); reserve-then-assign real for spot+coload general cargo; ACAS pre-load identity floor on Asia→US;
friction free-before-cutoff / fractional-at-cutoff (v2). Docs: grounding_s54.md, `air_capacity_parameters.csv`.

**Verification (2 rounds, 7 agents):** sunk-cost model kills the scored-friction L2-flip; NC-a (monotone
double-charge on service reroutes) **dismissed by user** (cancelled flight voids commitment; air delays
don't strand-with-commitment). Folded: `(1+ε)` on `CW^r`; per-group offset; `family_cost(0):=0`; h0
binding-dim; lexicographic min-reservation tie-break; whole-route atomic cutoff pin.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved; PDF behind — do NOT auto-compile |
| G-Method | Arrival-only replan methodology | ✓ §13 v4 + Reading-B + §14 + §14.1-R + §14.2 |
| **G-Design** | **2D spot reservation (S54, replaces D-A1)** | ✓ **APPROVED for build** — spec REV 3, 7-agent verified, 5 sub + 3 REV decisions locked |
| G-Review | Standing review agents (calibration / seam / red-team) | ⚠ overdue since S45 — the 7 S54 verify agents were design-review, not the 3 standing ones |
| G-Isolation | Air graph + MILP + replay + recourse + generator + F1/Model-Y + S1 schema | ✓ passed (386) |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ deferred

| Component | Status | Notes / pointer |
|---|---|---|
| `FreightNet` / `geo_select` | ✓ | freight-node ref DB + corridor candidates + landmass ground-gate. |
| Air graph generator (`air_graph.py`) | ✓ | geo candidates + `FallbackPolicy`. |
| Air MILP optimizer (`air_milp.py`) | ✓ | M1–M6 + spot cap + soft/hard BSA + `mip_rel_gap` + MFBlink + C.10 PWL. |
| Synthetic generator — air (`air_generator.py`) | ✓ | O-D-lane supply + repositioning + timestamp remodel + **S1 2D spot cap**. |
| `cap_decay.py` (F1) | ✓ | ONLY SPOT decays (grounded S54 = correct); grounded convex curve; BSA firm. |
| `replay.py` — Model-Y + soft-BSA cliff | ✓ | 5 arms; D5 cost split; 3-state OTP; fallback 3-cause. |
| **2D spot reservation (replaces D-A1)** | ◐ | **DESIGN APPROVED + verified; S1 (schema) BUILT. S2–S7 remain.** |
| H0 human baseline (`h0_planner.py`) | ◐ | strands ~29–51% → redesign PARKED; S6 switches spot read to binding dim. |
| **Thesis number (D3 sweep)** | ☐ | BLOCKED on the S54 build (S2–S7) + operating-point recalibration + metric reframe. |
| Tractability (R2) | ◐ | FIX2 shipped S49; FIX1 eta cut + symmetry-breaking = R2. |
| Ocean/LCL/Trucking; path-TT/rules/stitch; MCP/agent/UI | ☐/⏸ | later stages. |

---

## Near-term critical path — the 7 build slices (RESUME HERE = S2)

Each slice built + isolation-tested before the next (hard gate). Tardiness ON in every test; real HiGHS.

1. ✓ **S1 — schema split** `spot_cap`→`spot_wcap`/`spot_vcap` + derived `C^v` + round-trip. **386 green, byte-identical.**
2. ☐ **S2 — reservation state + ratchet:** `ReplayState._reserved_spot` (monotone `(r^w,r^v)`) + post-solve usage ratchet. Behavior-neutral until S3. Tests I1/I4.
3. ☐ **S3 — 2D decay floor:** `CapDecay` decays `spot_vcap` too, floored at the reservation; pre-cutoff envelope / post-cutoff pins branch. Tests I2/I3/I10.
4. ☐ **S4 — 2D MILP caps:** C.5d → raw `Σw ≤ spot_wcap` AND `Σv ≤ spot_vcap`. **Headline volume-breach test lands here.** Tests I5/I6/I7/I13-infeas.
5. ☐ **S5 — cutoff handoff:** reserve→identity, **whole-route atomic pin** at `τ0_a`, reconcile `τ0_a` (arc) vs `tender_at` (shipment). Tests I8.
6. ☐ **S6 — sunk-cost objective:** committed-spot cost (per-group offset, `family_cost(0):=0`, `(1+ε)` on `CW^r`) + ledger 2D-collapse + **h0 binding-dim `min(C^w,C^v·167)`** + lexicographic min-reservation tie-break. Tests I9.
7. ☐ **S7 — arm wiring + M1′ coherence** (S51 reversal) + **recalibrate operating point** (τ ladder / regime bands / density floor; tardiness weights UNCHANGED) + re-run NC1–5. Tests I11/I12/I13.

**S4 & S6 are the load-bearing, delicate slices.** After S7: re-anchor `scripts/d3_sweep.py` + metric reframe → run the sweep = the thesis number.

---

## Built & verified (quality state)

- **Test suite last green:** 2026-07-11 (S54, slice S1) — **386 passed**, S1 byte-identical.
- Ruff: 6 pre-existing E501 in untouched `air_generator.py` arrival-config + `scenario_db.py` (not S54's).
- **Real HiGHS, never mocked;** `mip_rel_gap=0.005` for L2-grade costs. CRN verified (demand ⟂ τ/α/ρ).
- **Perfect-info clairvoyant solve is the honest ceiling** at τ=3: 9% fallback / 60% OTP.

---

## Key locked decisions (pointers, not duplicated)

- **S54 (LOCKED, approved for build):** 2D spot reservation replaces D-A1; reserve-early/assign-late;
  sunk-cost/free-reserved cost; `C^v=C^w/333.33`; `penalty_frac=1` (NC-a dismissed); S51 M1′-dump reversal
  blessed; recalibrate operating point via τ. → spec REV 3, memory `project_s54_2d_spot_reservation`.
- **S54 grounding:** flat/MFB = one channel; decay = channel property (code already right); reserve-then-
  assign grounded (general cargo); identity-lock = min(cutoff, ACAS pre-load). → `reference_air_cargo_spot_family_decay_reserve`.
- **F1 / Model-Y / soft-BSA cliff (S51):** only spot decays; both BSA tiers firm; tendered-only floor (S54
  supersedes for spot with the reservation floor); `dep−48h` cliff. → §14.2.
- **Prior:** S50 §14.1-R O-D-lane tightness + ground-gate · Reading-B (L1=H₀−M₁', L2=M₁'−M₁; S54 downgrades
  the chain to empirical) · supply⟂demand · distress pricing = do NOT model (book early).

---

## Deferred / parked (do not lose)

- **⏸ v2 — SPOT RESERVATION CANCELLATION.** v1 ships no-cancel / `penalty_frac=1`. The grounded extension
  (release a reserved envelope before cutoff with a fractional penalty, `penalty_frac<1`, `r_a`
  releasable) is v2. Also v2: small offload hazard on held spot. Code carries a `TODO(v2)` at the ratchet
  site. → spec §11, grounding_s54.md "Deferred to v2".
- **Operating-point recalibration** (S7): the 2D cap is ~2× tighter on volume → re-derive the S49 τ→regime
  bands + density floor. Tardiness weights UNCHANGED.
- **Metric reframe** — retire `L2_real%`; report service (fallback%/OTP) + unsummed cost + per-tier kg.
- **Standing review agents** overdue since S45 (calibration / seam / red-team — distinct from the S54 design-verify agents).
- **H0 redesign** — strands ~29–51%; user to supply pseudocode. Affects L1 only.
- **FIX1 eta MIR cut + symmetry-breaking** (n≥100 tractability, R2).
- **Agent-created reference files (S52, uncommitted):** `papers/`, `docs/references-air-cargo-two-stage-allotment.md` — triage (vault vs delete).
- **Scratch** in `scripts/_*.err`/`_*.py` — uncommitted. **PDF compiles behind** (user owns compile).

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard |
| **`docs/design/spot_reservation_2d_design_s54.md` (REV 3)** | **the 2D-reservation spec — build follows it slice by slice** |
| **memory `project_s54_2d_spot_reservation`** | **CURRENT resume anchor — design + decisions + slice state** |
| `docs/design/air_cargo_spot_families_grounding_s54.md` | 6-agent grounding (decay + reserve-then-assign, per channel) |
| `air_capacity_parameters.csv` | Tables A–I: all capacity-type params + research verdicts (from code) |
| memory `reference_air_cargo_spot_family_decay_reserve` | grounded Q1/Q2 verdicts + flat/MFB collapse + ACAS |
| `SESSION_LOG.md` (S54 entry) | full S54 record |
| `CONTEXT.md` | compressed context + RESUME HERE (S54) |
| `model/arrival_only_replan_methodology.md` | governing proof methodology (§14 reconciliations pending per spec §11) |
| `scripts/d3_sweep.py` | D3 sweep harness (τ ladder STALE; run after S7) |
| `src/replay.py` / `src/cap_decay.py` / `data/synthetic/air_generator.py` | the build targets for S2–S7 |
