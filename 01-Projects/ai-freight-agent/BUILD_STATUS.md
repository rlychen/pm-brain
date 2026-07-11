# BUILD STATUS — AI Multimodal Freight Routing Agent

> ⛔ **HARD RULE: TARDINESS PENALTY ALWAYS ON.** Never run any air model (probe/test/calibration/sweep)
> with `tardiness_weight_scale=0`. The quadratic C.10 penalty (calibrated express/standard/deferred) must
> be live 100% of the time — a W=0 run is a BUG, not a result. Full objective always (cost + tardiness +
> fallback). See `CLAUDE.md` top + memory `feedback_tardiness_penalty_always_on`.

**Last refreshed:** 2026-07-10 (Session 53 — **TIMESTAMP REMODEL BUILT (Phase 0+1), 386 green.** Grounded the
⟨book, available, latest-arrival⟩ triplet (8 research agents: booking curve, tier windows via 3 independent
signals, book→available gap, RM timestamp modeling, reserve-vs-assign wiring) and, after 4 plan-checkers + 2
reviewers, built the "grid realism" unit: tier `sla_offset` E/S/D 12/24/**96** (deferred = standard+3d),
`backstop_buffer_h` 168→**264**, `t_dead` ceiling 144→**240**, opt-in **`flight_horizon_days`** (flights 16d /
demand 7d), and random **`book_at`** metadata (own CRN-safe `book_gap` RNG stream). Reverted the S52-era broken
book-curve edit → restored D-A9. **Grounded facts:** base_transit is door-to-door + tier-agnostic; tier = SLA
slack + existing tardiness weight (no new axis); deferred = standard + 3 days (Lufthansa). **Next: Phase 2 —
M1′ coherence (symmetric reserve floor), the S52 structural fix.** Plan of record: `scratchpad/timestamp_remodel_plan.md`
v3. Grounding: `docs/design/air_cargo_demand_arrival_grounding.md`.)
Prior S52: 2026-07-09 (split-`tender` reserve/assign DESIGN + grounding, no build). Prior S51: 2026-07-05.

**How to use this doc.** Canonical built/remaining dashboard — read first on session start. Detail:
**memory `project_s52_reserve_assign_synthesis` (the two tables + 5 decisions)** · SESSION_LOG.md S52 entry ·
CONTEXT.md (RESUME HERE) · `model/commit_timing_reserve_assign_proposal.tex`.

---

## Current position

- **Strategy:** go vertical on **AIR** to the replan-savings proof. The proof rides a synthetic generator
  whose supply/demand tightness + capacity-decay + arrival/commit timing must be realistic.
- **Phase:** 2 (Component Builds), air slice — **grid realism BUILT (S53), 386 green.** The generator now
  carries the grounded ⟨book, available, latest-arrival⟩ triplet: tier = SLA slack (deferred = std+3d) +
  the existing tardiness weight, opt-in 16-day flight horizon (`flight_horizon_days`), and random `book_at`
  metadata (own CRN-safe RNG stream). base_transit confirmed door-to-door + tier-agnostic.
- **Next:** Phase 2 — **M1′ coherence (symmetric reserve floor)**, the S52 structural fix (M1′ commits a route
  early but holds no space → decayed spot infeasibilizes it → self-inflicted fallback dump). Then Phase 3 —
  measure L1/L2/OTP on the realistic grid + coherent arm. **Sweep/proof configs must set `flight_horizon_days=16`**
  (default None keeps the 7-day behavior + tests green).
- **S52 (still pending):** the reserve-space-early / assign-cargo-late design is grounded + written; the
  symmetric-reserve build IS Phase 2 above.

---

## S52 — the design + grounding (the headline workstream)

**The fix (proposal `model/commit_timing_reserve_assign_proposal.tex`, written NOT compiled):** split the
monolithic `tender` (which today pins the route, freezes assignment, AND is the only decay-protection at
once, all at the cutoff) into two events — **reserve** a per-arc spot *quantity* `r_a` early (no HAWB pin;
decay floor `f=max(r_a,b_a)` so reserved space doesn't decay) and **assign** cargo→ULD late (fluid until
cutoff). **Red-team reframe:** make **M1′ coherent FIRST** — today it commits a route early but holds no
space (Model-Y floors only tendered) → decayed spot forces `_repair_frozen_infeasible` to dump to fallback;
the "57% fallback" is a strawman artifact. Both arms must reserve; the ONLY difference is assignment fluidity.

**Grounding (two rounds; VERDICT = real & standard):** reserve-space-early/assign-cargo-late is confirmed
for **spot** (not just BSA) — the **master booking is cargo-agnostic** (IATA Cargo-IMP **FFR** space-request
≠ **FWB** waybill; IAG e-booking needs route+commodity+weight+own AWB #, not shipper/consignee; cargo.one
"book against AWB stock"). Reserved-but-unused space **costs money, cutoff-gated** (AA $300 no-show >250kg;
LH 25%<48h/50%<24h; free to release before cutoff). See memories `project_s52_reserve_assign_synthesis`,
`reference_air_cargo_reserve_assign_practice`; doc `air_cargo_reserve_assign_grounding_s52.md`.

**What the grounding moved:** Change-4 friction → **cutoff-gated fractional no-show** (not flat full-spot
charge) ⇒ reserve likely **releasable, not monotone**; 48h cliff → **calibration range** (~24–96h);
spot-reserve grounding **upgraded** (master booking cargo-agnostic); scope = **general cargo** (special
commodities + security/advance-data force identity early).

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved; PDF behind — do NOT auto-compile |
| G-Method | Arrival-only replan methodology | ✓ §13 v4 + Reading-B + §6/§6.1 + §14 + §14.1-R + §14.2 (S51) |
| G-Design | **Commit-timing reserve-vs-assign proposal (S52)** | ◐ **written + grounded; NOT approved — 5 decisions pending (incl. D-A1 amendment)** |
| G-Review | Standing review agents (calibration / seam / red-team) | ⚠ run S45; **overdue since S52** |
| G-Isolation | Air graph + MILP + replay + recourse + generator + F1/Model-Y/soft-BSA-cliff | ✓ passed (386) |
| G-LaTeX | Ocean FCL / LCL / Trucking models | ☐ drafted, NOT approved |

---

## Component status — whole product

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ deferred

| Component | Status | Notes / pointer |
|---|---|---|
| `FreightNet` / `geo_select` | ✓ | freight-node ref DB + corridor candidates + landmass ground-gate. |
| Air graph generator (`air_graph.py`) | ✓ | geo candidates + `FallbackPolicy`. |
| Air MILP optimizer (`air_milp.py`) | ✓ | M1–M6 + spot cap + soft/hard BSA + `mip_rel_gap` + MFBlink + C.10 PWL. |
| Synthetic generator — air (`air_generator.py`) | ✓ | O-D-lane supply + increment-4 repositioning + fix-B per-origin BSA. |
| `cap_decay.py` (F1, S51) | ✓ | ONLY SPOT decays; grounded convex curve; both BSA tiers firm. |
| `replay.py` — Model-Y + soft-BSA cliff (S51) | ✓ | tendered-only spot floor; repair-frozen + fail-fast; `dep−48h` cliff. |
| `ReplayState`/`run_replay`/arms/scorer | ✓ | 5 arms; D5 cost split; 3-state OTP; fallback 3-cause. |
| **Commit-timing fix (reserve-vs-assign)** | ◐ | **DESIGNED + GROUNDED, NOT BUILT.** Proposal `.tex` written; 5 decisions pending. Build starts with M1′ coherence. |
| H0 human baseline (`h0_planner.py`) | ◐ | strands ~29–51% → redesign PARKED (awaits user pseudocode); affects L1 only. |
| **Thesis number (D3 sweep)** | ☐ | BLOCKED on the timing fix build + metric reframe; `scripts/d3_sweep.py` τ ladder STALE. |
| Tractability (R2) | ◐ | FIX2 shipped S49; FIX1 eta cut + symmetry-breaking = R2. |
| Ocean/LCL/Trucking; path-TT/rules/stitch; MCP/agent/UI | ☐/⏸ | later stages. |

---

## Near-term critical path — ordered (RESUME HERE)

**Do NOT build. First get sign-off on the 5 decisions (memory `project_s52_reserve_assign_synthesis`).**

1. ☐ **Sign off the 5 decisions:** (1) D-A1 amendment (split tender→reserve+assign); (2) **friction model** —
   cutoff-gated fractional no-show magnitude + reserve **monotone vs releasable** (the load-bearing knob);
   (3) estimator (D5-separated + per-tier kg + 3-cause); (4) M1′ symmetric-reserve recoding; (5) methodology
   framing (general-cargo scope; reserve = master-slot quantity; special-commodity/security exceptions).
2. ☐ **One consistent doc pass** (after sign-off): rewrite `air_cargo_reserve_assign_grounding_s52.md`
   (spot+BSA), update memory, refine proposal `.tex` per the decisions.
3. ☐ **Build — M1′ coherence FIRST** (symmetric reserve floor; kill the frozen-route/unheld-space dump),
   then the reserve primitive + friction + book-lead → ~5d. Re-run negative controls (NC1–5) before trusting.
4. ☐ **Re-anchor `scripts/d3_sweep.py` τ ladder** (stale `{0.75,1.05}`) + metric reframe → **run the sweep = the thesis number.**
5. ⏸ **Capacity mix** (contracted 0.89× demand + metro-mismatched) · **R2 scalability** (FIX1 eta cut + symmetry-breaking).

---

## Built & verified (quality state)

- **Test suite last green:** 2026-07-05 (S51) — **386 passed**, ruff clean. **S52 changed NO code** → unchanged.
- **Real HiGHS, never mocked;** `mip_rel_gap=0.005` for L2-grade costs. CRN verified (demand ⟂ τ/α/ρ).
- **Perfect-info clairvoyant solve is the honest ceiling** at τ=3: 9% fallback / 60% OTP.

---

## Key locked decisions (pointers, not duplicated)

- **S52 (PROPOSED, not locked):** split `tender` → reserve-space-early + assign-cargo-late, symmetric arms;
  friction = cutoff-gated fractional no-show; reserve grounded for spot (master booking cargo-agnostic).
  → memory `project_s52_reserve_assign_synthesis`, proposal `.tex`. **D-A1 amendment pending.**
- **F1 / Model-Y / soft-BSA cliff (S51):** only spot decays; both BSA tiers firm; grounded convex curve;
  tendered-only floor; `dep−48h` cliff. → §14.2.
- **Prior:** S50 §14.1-R O-D-lane tightness + ground-gate · Reading-B decomposition (L1=H₀−M₁', L2=M₁'−M₁) ·
  supply⟂demand · D-F6 v2 SLA deadline · distress pricing = do NOT model (book early).

---

## Deferred / parked (do not lose)

- **The 5 open decisions** — the RESUME point (sign-off pending).
- **Agent-created files to TRIAGE** (S52, uncommitted, unreviewed): `papers/`,
  `docs/references-air-cargo-two-stage-allotment.md`, edits to `docs/academic-literature-references.md` +
  `references/air-cargo-allotment-contracts.md`. Likely move to vault or delete (notes→vault, not repo).
- **Workflow-harness bug:** deep-research `WebFetch` has no timeout → a dead host hangs the whole run (stalled
  15+h twice on `scm-en.ecer.com`). Avoid that harness for web research / pre-screen URLs; regular agents fine.
- **Book-lead re-grounding** — lengthen 1.9d → ~5d (Change 6). **Metric reframe** — retire L2_real%.
- **Capacity mix** — contracted undersized (0.89× demand) + metro-mismatched; feeder spot oversized.
- **H0 redesign** — strands ~29–51%; user to supply pseudocode. Affects L1 only.
- **FIX1 eta MIR cut + symmetry-breaking** (n≥100 tractability, R2).
- **Standing review agents** overdue since S52.
- **Old scratch** in `scripts/_*.err`/`_*.py` (S50) — uncommitted. **PDF compiles behind** (user owns compile).

---

## Doc map (where detail lives)

| Doc | Role |
|---|---|
| `BUILD_STATUS.md` (this) | clean built/remaining dashboard |
| **memory `project_s52_reserve_assign_synthesis`** | **CURRENT resume anchor — grounding table + changes table + 5 decisions** |
| **`SESSION_LOG.md` (S52 entry)** | full S52 record — design / grounding / what-moved / decisions |
| `model/commit_timing_reserve_assign_proposal.tex` | the reserve-vs-assign proposal (written, NOT compiled) |
| `docs/design/air_cargo_reserve_assign_grounding_s52.md` | grounding synthesis (BSA-heavy; rewrite w/ spot after sign-off) |
| memories `reference_air_cargo_reserve_assign_practice` / `reference_air_cargo_timing_and_pricing` | grounded practice + timing/pricing |
| `CONTEXT.md` | compressed context + RESUME HERE (S53) |
| `model/arrival_only_replan_methodology.md` | governing proof methodology (§14.2 = S51) |
| `scripts/d3_sweep.py` | D3 sweep harness (τ ladder STALE; metric reframe needed) |
| `src/replay.py` / `src/cap_decay.py` | Model-Y + soft-BSA cliff / F1 spot-only decay |
