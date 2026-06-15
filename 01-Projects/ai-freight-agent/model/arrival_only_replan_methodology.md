# Arrival-Only Replan Methodology — v0.1

**Status: APPROVED (Session 32, 2026-06-09).** **Gate: G-Method — cleared.** This is the **governing
methodology** for the air replan-savings proof; it amends the two prior gated specs —
`air_transit_time.md` (reconciled to v0.3) and `backtest_methodology.md` (reconciled to v0.5) — per the
reconciliation map in §7. Decisions D-A1..D-A4 locked to the recommended defaults (cutoff-only tender;
arrival lateness coupled to tier; ground/customs at the mean; predicate-9 retired for air). Nothing is
sampled to create failure: the realization is **deterministic** and the only stochastic process in the
proof is the **demand-arrival stream**.

---

## 1. Thesis (one line)

The only uncertainty is **which tiered HAWBs arrive when**. Replan value = re-solving and **reshuffling
the open (un-tendered) book** as new demand lands under finite capacity. Transit is deterministic; no
failure is injected. The resulting L2 is a **conservative lower bound** — it holds even with perfectly
reliable transit and zero disruptions, so it can't be attacked as manufactured.

## 2. What we removed, and why

- **Per-leg Gaussian transit jitter** (old 2b `sample_route`) — made the *same route* on-time on one
  draw and late on the next: a coin-flip that randomly declares infeasibility. Not the air value driver.
- **A discrete disruption stream as a value source** — same defect: manufactured failure folded into the
  graph. Disruption is something you *react to*, not something you fabricate to inflate the number.
- **Consequence:** the realization is deterministic, and the reliability machinery (`z_tier`, `σ̂`,
  predicate-9 as a *probabilistic* admission) collapses to deterministic deadline feasibility (§3).

## 3. Deterministic realization (replaces the 2b stochastic sampler)

- **Realized arrival `A(r)`** = a single running-clock walk of route `r`: per air leg, the **scheduled
  block** (`arr_utc − dep_utc`; the published block is already carrier-padded to ~P80 on-time); per
  ground/dwell component, a **fixed quantile** `q` of its time (default: the mean — see D-A3). At each
  leg, `dep = max(sched_dep, clock)`. No draws.
- **`leg_actuals` / `component_actuals`** (already built in generator-to-files) are **retained and
  populated deterministically** — a uniform input for the scorer walk and reusable verbatim for ocean,
  where transit *is* stochastic. The RNG sub-streams stay wired (no-ops for air).
- **Predicate-9 → deterministic deadline feasibility** `A(r) ≤ Δ_k`. `z_tier` and `σ̂` are **retired for
  air** (revisit for ocean). Tiers act through **deadline tightness `Δ_k`**, **flexibility** (reshuffle
  headroom), and **priority `W_k`** — never a reliability margin. `route_reliability` survives only as the
  deterministic `Â` calculator.

## 4. The replan engine (the actual proof)

- Demand is revealed over the sim clock via `known_at`, carrying a **service tier**.
- **Irreversibility — tender = commit.** A tendered shipment's route and capacity are locked; the **open
  book** = un-tendered shipments, the only thing reshuffleable. *[D-A1: what forces tender — cutoff-only
  (recommended) vs. earlier firm events.]*
- **Cycle:** reveal new demand → (re)plan → **tender whatever is at its cutoff** → decrement capacity.
- **Arms** (same arrival stream, same frozen promise / control inputs):
  | arm | behavior |
  |---|---|
  | `H₀` | human heuristic — batch-at-cutoff, simple rules (the spreadsheet) |
  | `M₀` | **greedy incremental** — places each newcomer into the best option available when processed, **without** jointly optimizing the cycle's other newcomers; priors pinned (no reshuffle). Myopic baseline. |
  | `M₁'` | **single-pass optimal (pinned replan)** — each cycle the MILP **jointly** optimizes all un-tendered newcomers with priors hard-pinned (`x_{k,a}=1 ∀(k,a)∈S_t`); optimum of the no-reshuffle feasible set. The competent planner we'd actually ship. |
  | `M₁` | **open-book** — full MILP re-optimization of the open book each cycle (reshuffle all un-tendered; pins relaxed). |
  | `π_hind` | all demand known at `t=0`, solved once (the clairvoyant lower bound) |
- **Cost chain (guaranteed by feasible-set nesting + optimality):**
  `C(H₀) ≥ C(M₀) ≥ C(M₁') ≥ C(M₁) ≥ C(π_hind)` *when solves reach optimality*. `M₀ ≥ M₁'` because greedy is
  ≥ the optimum of the same pinned set; `M₁' ≥ M₁` because `M₁` optimizes a superset (pins relaxed).
  *(BLK-1 decision S38: each solve is capped at a 600s wall-clock limit and returns the best incumbent —
  NOT solved to optimality. So the chain holds per-draw only for cells that finish; on time-limited cells a
  truncated incumbent can transiently violate it. Accepted for now; real tractability — warm-start chaining /
  symmetry breaking / smaller n — is deferred until the full pipeline is built. Track `status`/`mip_gap` per
  solve.)*
- **Decomposition:**
  - **`L1 = C(H₀) − C(M₁')`** (planning value — human → the competent single-pass optimizer we ship).
    Internally splits into **automation** `C(H₀) − C(M₀)` + **within-cycle optimization** `C(M₀) − C(M₁')`;
    `M₀` is an internal ablation rung, not a product-facing endpoint.
  - **`L2 = C(M₁') − C(M₁)`** (replan value — *headline*; cross-cycle reshuffle / open-book recourse).
    Computed **entirely within the MILP engine** (pins on vs off), so the headline subtracts two runs of the
    *same* solver — structurally free of cross-engine / code-path artifact (this replaces the retired
    `C(M₁')==C(M₀)` "leakage" device; see D-A11).
  - `Total = C(H₀) − C(M₁) = L1 + L2`; `regret = C(M₁) − C(π_hind)`.
- **OTP** is now deterministic-given-routing: a shipment is on-time iff its committed route's `A ≤ Δ_k`.
  Cross-arm OTP differences come from **capacity-driven routing choices under the arrival dynamics**, not
  random transit. OTP is still a **population** metric (across shipments/tiers), scored vs the **frozen
  booking promise**.
- `cw_flex` (per-flexible-kg denominator) frozen at `t=0`, arm-invariant — **unchanged** from approved.

## 5. The sweep

**κ (capacity tightness) × λ (arrival lateness relative to cutoffs).** Tighter κ + later λ ⇒ more demand
is unknown when capacity must be committed ⇒ larger premature-commitment cost ⇒ larger L2. *[D-A2:
arrival–tier coupling — do tiers differ in arrival lateness, or only in deadline tightness? Recommended:
couple them — EXPRESS arrives **late and tight**, DEFERRED **early with slack** — that asymmetry is what
makes reshuffle pay.]*

## 6. Disruption recourse — a TESTED CAPABILITY, not a value source

Kept **out of the headline scenario** so the L2 number stays clean. But the recourse path is real code
that must work and not corrupt state, so it is **verified by deterministic fixtures**, not a probability.

- **Capability:** a firm route breaks (flight delay/cancellation) → **unlock the shipment's *remaining*
  (future) legs** → **replan from the cargo's current node and time** → recover, or fall back.
- **Past legs are immutable.** The "unlock" applies only to the un-flown remainder; the running clock
  continues from where the cargo physically sits when the disruption is realized. (A cancellation means
  the cargo never departed → reroute from its current node; a delay means continue from arrival.)
- **The promise holds.** Recovery is measured against the **original frozen `Δ_k`**. A disrupted shipment
  that now lands late is a **miss**; replan's job is to minimize that. **No renegotiation** — letting
  replan reset the promise would let it cheat the OTP metric.
- **Deterministic fixtures (built with the 2c loop):**
  1. **Absorbable delay** — small slip, connection still made ⇒ **no replan fires** (the no-op guard).
  2. **Connection-breaking delay on a firm shipment** ⇒ unlock future legs ⇒ reroute from current ⇒
     recover or fallback.
  3. **Cancellation of a firm-carrying flight** ⇒ cargo never departed ⇒ reroute from current node.
  - **Invariants asserted:** no capacity double-spend; ledger conservation through the unlock; frozen
    promise unchanged; two-solves-identical determinism. (Joins the existing 2c fixture set — binding-
    capacity / mid-tender conservation.)
- **Optional, later:** a separate, clearly-labeled **sensitivity study** that *measures* disruption-
  recovery value (its own arm) — never folded into the headline L2.

## 7. Reconciliation into the gated specs (on approval)

- **`air_transit_time.md` v0.2 → v0.3:** replace stochastic `sample_route`-as-realization with the
  deterministic arrival walk (§3); mark `σ̂` / `z_tier` ocean-only; keep `route_reliability` as the
  deterministic `Â`. The DoD's "single-walk realization reproduces a connection slip" item is reframed
  as the **deterministic** connection check (clock + MCT ≤ dep).
- **`backtest_methodology.md` v0.4 → v0.5:** realization = deterministic; OTP deterministic-given-routing;
  redefine `M₀` (incremental-greedy) vs `M₁` (open-book re-opt); add §recourse capability + the three
  fixtures + the "promise holds" rule; frame L2 as a conservative lower bound; drop the "one draw per
  leg" language.

## 8. Decisions — LOCKED (Session 32)

- **D-A1 — irreversibility lock:** **tender-at-cutoff is the only lock.** No earlier firm/hold events in
  MVP (a held/confirmed slot is a later capability).
- **D-A2 — arrival–tier coupling:** **lateness is coupled to tier** — EXPRESS arrives late + tight,
  DEFERRED early + slack. The asymmetry is what makes reshuffle pay; the demand generator encodes it.
- **D-A3 — ground/customs realization quantile:** **the mean** (unbiased); air leg = scheduled block.
  Realization is deterministic (`s = 0`).
- **D-A4 — predicate-9 for air:** **retired** → deterministic deadline feasibility `A ≤ Δ_k`.
  `z_tier` / `σ̂` revive only for ocean.

## 9. Unchanged (already built / approved — no churn)

`scenario_db` schema; generator-to-files plumbing (persist/load, CRN, determinism); 2-FLEX `TierSpec`;
route-versioning; capacity ledger + conservation identity; the `leg_actuals` / `component_actuals` tables
(now deterministic, reused for ocean).

## 10. The demand-arrival process (the λ stream) — LOCKED (Session 32)

This is the **only stochastic primitive**; everything else is deterministic. It complements `flexibility_model.md`
v0.3 (which owns tiers / `Δ_k` / slack / `flex` / `TierSpec`) by defining the timing the flex model left open
(`known_at` and its tier coupling).

**Schedule (D-A5).** Per lane, **one departure per day** over `D ≈ 7` days (the flex model's F1, F2, F3…).
Each departure `d` has a CFS cutoff `cutoff(d) = dep(d) − L_cut`; reaching `cutoff(d)` **forces tender** for
`d` (the only irreversibility lock, D-A1). Per-departure cheap/contracted capacity is rationed by **κ**.
*(Build implication: extend the current single-cycle TPEB schedule, deps at 16–46h, to a daily horizon.)*

**Per-HAWB arrival (D-A6, D-A7).** Draw `tier` (20/40/40 mix) and an earliest-feasible/target departure `d*`:
- **`known_at = cutoff(d*) − B`** — anchored to the **cutoff** (D-A6, the real decision point), with **book-lead
  `B` tier-coupled** (D-A7): EXPRESS `B` small (arrives just before the cutoff; only `d*` on-time), STANDARD
  moderate, DEFERRED `B` large (known days ahead; `d*..d*+2` on-time = its slack).
- `ready_k = known_at + prep_time_h`; `Δ_k` tier-derived per the flex model (EXPRESS tight → only `d*`;
  DEFERRED loose → several departures).
- **Fixed `N` HAWBs per scenario** (D-A8), `known_at` drawn, generate-all-first → reveal over the clock.

**The engine.** Early-DEFERRED greedily lands the cheap `d*` slot; late-EXPRESS needs `d*` but it's gone; `M₁`
re-solves at the cutoff cycle and **bumps DEFERRED → `d*+1`** (it has slack) to free the cheap `d*` slot for
EXPRESS. `L2` = the bump. Pure information-timing × scarcity — no injected failure.

**The sweep.** **κ** = per-departure cheap-capacity tightness. **λ** = a global **compression of book-lead `B`
toward the cutoff** (higher λ ⇒ more of the book unknown when cutoffs fire ⇒ static commits more blindly ⇒
larger `L2`). Tier coupling sets the relative ordering; λ is the global shift.

**Decisions — LOCKED:** D-A5 daily departures, `D≈7`. D-A6 `known_at` anchored to `cutoff(d*)`. D-A7 book-lead
tier-coupled (EXPRESS small … DEFERRED large) — **but the HEADLINE cell uses tier-INDEPENDENT lateness, see
D-A9; D-A7 is a load-bearing empirical claim `[CAL]`, reported as an upper bracket only.** D-A8 fixed `N` with
drawn `known_at` (generate-all-first).

## 11. Next-stage gates (user-requested, Session 32)

- **Multi-agent critique of the simulation BEFORE/WITH the build.** Before trusting the replan number, run
  several critique agents over this design (arrival process + replay loop): *is the simulation clean and
  sensible, and does it provide a clear, falsifiable test of the replan thesis?* Fold findings before/while
  building 2c — the project's established design→critique→fold→build cadence.
- **Scale-up stress test AFTER the proof passes.** Once the small TPEB proof is green, stress with **larger
  supply + many HAWB arrivals**: mimic a **small forwarder**, then a **medium forwarder** (scale lanes,
  departures, daily arrival volume). Tractability + does the replan signal hold (or sharpen) at scale.

## 12. Design-review hardening (critique 11, 4-agent) — LOCKED (Session 32)

The 4-agent design review (`docs/critique/11-simulation-design-review.md`) converged on a small set of
load-bearing fixes. Folded here as decisions **D-A9..D-A16** + DoD gates. The classic confounds (lookahead,
forecaster-asymmetry, double-spend, denominator inflation, OTP re-promising) were judged **sound, no change**.

**Decisions — LOCKED:**

- **D-A9 (C1 — the credibility crux) — independent-arrival is the HEADLINE.** Book-lead `B` is a per-tier
  *distribution with overlap* (`[CAL]`), not a deterministic offset. The **headline cell draws lateness
  tier-INDEPENDENTLY** (only `Δ_k` stays tier-coupled); the tier-coupled-favorable arrival (D-A7) is reported
  as an **upper bracket**, never the headline. If `L2` collapses under independent arrival, that is the
  finding. (Removes "we built the sim to win.")
- **D-A10 (C2) — pre-registered null + mandatory negative control.** Null: *thesis unsupported if peak-cell
  `L2` CI straddles 0, or `M₁'≈M₁` across the grid.* A **required abundant-capacity × early-arrival cell** with
  a gated `|L2| < CI` pass condition (a regime where replan must NOT help). **κ is dialed in binding-ness**
  (peak-concurrent-demand / slots), not quantized ULD integers (`max(1, round(2·scale))` is retired as the κ axis).
- **D-A11 (C3 — blocks 2c) — pin-prior-soft + the three pinned-vs-open arms (`M₀ / M₁' / M₁`).** *(Revised S38 —
  retires the `C(M₁')==C(M₀)` invariant; `M₁'` is now a first-class no-reshuffle baseline, not a leakage placebo.)*
  **Two orthogonal axes, do not conflate:**
  1. **Priors (the original "Reading A", KEPT):** un-tendered prior-cycle commitments are pinned via a soft-pin
     primitive `x_{k,a}=1 ∀(k,a)∈S_t` (ground re-derives; a HAWB whose pinned departure goes infeasible falls to
     fallback). **Both `M₀` and `M₁'` pin priors;** only `M₁` relaxes them (open book).
  2. **Newcomer placement (the S38 distinction):** `M₀` places newcomers **greedily/myopically** (one at a time,
     no joint within-cycle optimization); `M₁'` places them at the **joint optimum** of the same pinned set.
  Hence `C(M₀) ≥ C(M₁')` (greedy ≥ optimum) and `C(M₁') ≥ C(M₁)` (`M₁` optimizes a superset) — a *guaranteed
  inequality*, not an empirical hope. The old "net the `M₀−M₁'` gap out as leakage" is **gone**: that gap is the
  real **within-cycle optimization value** (a component of `L1`), and the **headline `L2 = C(M₁') − C(M₁)` is
  intra-engine** (same MILP, pins on vs off), so it carries no cross-code-path artifact for `M₁'` to detect.
- **D-A12 (C4) — the headline means *reshuffle*.** `realized_cost` **excludes** the C.10 quadratic tardiness
  penalty (objective-steering term, not a cash outflow; `C(π)` = freight + consolidation + spot/recovery).
  Report **three** components: `L2_reshuffle` / `L2_fallback_avoidance` / tardiness-penalty-delta. **Gate
  `L2_reshuffle > 0` (separated CI) as the headline**, with a pre-registered reshuffle-share floor (≥50%).
  Retire the `$1M` fallback for `C^fallback = 2× worst-feasible-route`.
- **D-A13 (C6) — one time-scalar source of truth.** `(block, ground-mean, dwell, MCT)` are read **identically**
  by graph-build forward-propagation, the MILP `arr_dest`, and the scorer walk. Invariants: **walk ≡ scalar
  for committed routes**; **`C_hind ≤ C(M₁)` per draw**. (Stops consolidation-drift from contaminating OTP and
  breaking the regret floor.)
- **D-A14 (C7) — batch-at-cutoff `H₀` is the headline baseline** (a competent human stages to the build cutoff);
  on-arrival `H₀` is the upper bracket only. Reconcile the `backtest §0/§3` (batch) vs `§4` (on-arrival) text.
- **D-A15 (M-B1) — scope the "conservative lower bound" claim** to **transit-reliability only** (adding
  disruptions only raises `L2`); it is **NOT** a lower bound w.r.t. the human baseline timing or the arrival
  asymmetry, both set to `L2`-favorable values otherwise.
- **D-A16 (M-B2) — BSA pacing inputs frozen.** `cap_a` / `A_c` are **bit-identical across all four arms** and
  static per departure (extend the "control inputs frozen during the proof" rule). Dynamic pacing is a separate
  capability demo, never live inside `L2`.

**Build WITH 2c (fixtures / instance tasks):**
- **C5 — atomic bump + global conservation.** Conservation is a **per-step identity across all arcs** + a
  **per-shipment move-journal** (a slot is conserved as it moves d*→d*+1, not just per-row). Add the **2-arc
  reshuffle fixture**. Add a **per-slip cost** to a bumped HAWB (extra CFS-dwell day × storage + notification/
  admin, reusing gateway `cfs_dwell`/handling) so M₁ reshuffles only when it pays.
- **M-B4 — cutoff timing.** Snap every `cutoff(d)` to a sim-step grid point (no cutoff strictly inside a step);
  deterministic within-step tender order `(tender_at, tier, shipment_id)`; `cutoff ≤ t` ⇒ tendered before
  `plan()` runs. Simultaneous-cutoff contention fixture.
- **M-B5 — ≥2 cheap options per lane.** Emit the roll-to-next-flight-on-contract option so the no-replan
  failure is "late but cheap" in some cells, not always "spot 3×."

**Deferred to report-time / scale-up:** M-B3 (lead with `L2%`; `L2/cw_flex` peak-cell only — denominator
shifts across the λ grid); M-B6 (`π_hind_locked` to split recoverable vs irreducible regret); M-B7 (power pilot
at the peak cell, size `R` for CI half-width `< L2/4`); M-B8 (pre-register the reporting α / operating point);
M-B9 (report the `known_at` distribution; state fixed-N as an L2-deflating simplification); + scope caveats
(disruption-recovery is the larger real driver — run the §6 recourse sensitivity before claiming the air thesis
*fully* proven; ready-time / customs-hold variance zeroed by the deterministic-headline scope).

## 13. Capacity & pricing refinement — independent network-supply model (APPROVED v4, Session 35, 2026-06-13)

**Status: APPROVED (Session 35).** Governing for the F1 build. Supersedes D-A10, D-A12 (fallback),
D-A18–D-A21; adds D-A23, D-A24. Three critique rounds folded (v1 circular-supply → v2 → v3 → v4); final
convergence verdict APPROVE-WITH-MINOR-EDITS, edits applied.

**Framing (corrects two v0.1 errors).** (1) κ as a single global scalar with contracted-always-cheaper is
too narrow. (2) More fundamentally, **supply must NOT be derived from the realized demand** — that is
circular and erases the supply/demand mismatch the optimizer exists to resolve. A forwarder buys contracted
blocks **ahead**, on forecast and carrier relationships; today's actual cargo does not match. The mismatch
— idle cheap capacity on some lanes, spill on others — **is** the problem, and the source of replan value.
Supply is generated **independently of demand**, across the whole network; the optimizer routes cargo to
wherever the contracted / spot / fallback mix is cheapest.

**The network & demand — D-A24 (LOCKED, S35): region→region routing is committed scope.** The lane grid =
(origin gateways × dest gateways); each lane carries its daily flights. A HAWB is **region-O → region-D**
(e.g. Shanghai-area → LA-area): its **origin airport, dest airport, lane, and flight are optimizer
decisions**, driven by a **per-airport trucking-cost matrix** (cost from the HAWB's true pickup/delivery
point to each candidate origin/dest airport) + air price + capacity. **This is a committed scope expansion**,
not optional — the fixed-lane (`DEMAND_LANES`, single origin/dest per HAWB) model is retired. The build
carries three mandatory pieces: **(1)** a per-airport trucking-cost matrix on the HAWB (replacing the single
`pickup_cost`/`delivery_cost` scalars); **(2)** **multi-origin/multi-dest subgraph construction** — each
HAWB's subgraph spans every candidate origin×dest airport pair in its regions, with airport-pair-specific
arc IDs; **(3)** a **tractability re-check** at instance size before the κ×α sweep is trusted. F1 does not
proceed on fixed lanes.

**Three supply sources per flight.**
| source | capacity | price |
|---|---|---|
| **contracted** (BSA) | **integer ULD positions `N_f`** (take-or-pay), drawn | contracted base rate, per-ULD pivot (C.13) |
| **spot** | **chargeable-weight cap** (drawn) — new per-arc CW-sum constraint | base rate × `m`, `m` ~ two-sided band |
| **fallback** | unlimited | **1.5 × the worst realistic spot route** (route-based, not a single rate) |

**Decisions — APPROVED (supersede D-A10, D-A18, D-A20, D-A21; amend D-A12; the v1/v2 `cap_a`/SE/`peak_demand`
machinery is WITHDRAWN):**

- **D-A18 (rev v4) — supply is an independent INTEGER network draw.** Per-flight contracted capacity =
  **integer ULD positions**, drawn from its **own `supply` RNG sub-stream, never reading the demand draw**
  (new stream — F8; today allotment sizes ride the `rates` stream, which must change so κ/α don't couple to
  rate draws). Knobs: **(a) κ = network tightness** — `total_N = round(E[Σ_k SE_k] / κ)`, where
  **`E[Σ SE_k]` is the analytic mean of standalone slot consumption `SE_k = max(w_k/1500, v_k/4.5)`** over the
  cargo density-mix distribution — a **closed-form constant** (reads zero demand draws → CRN), and the
  explicit **no-consolidation upper bound** on slot demand (consolidation only lowers true slots, so κ is a
  conservative tightness). **(b) concentration α** — per-flight counts ~ `Multinomial(total_N, p)`,
  `p ~ Dirichlet(α)` (low α lumpy → severe local mismatch even at κ=1; high α even). Per-lane/flight tightness
  **emerges** from random supply vs where demand lands. **At proof scale `total_N` is small (~10 over dozens
  of flights), so κ is swept on a COARSE INTEGER LADDER, not a smooth continuum** — the smooth (κ,α) plane is
  a forwarder-scale property (§11 stress test). Headline reported on the (κ,α) plane. **Under D-A24, κ indexes
  NETWORK tightness only** — where the mismatch actually bites is an *emergent* (α × per-airport trucking
  matrix) property, so the tractability re-check must report **per-airport binding-rate**, not a network
  average that hides slack airports. **Also report realized post-consolidation occupancy** alongside each κ
  label (since `E[Σ SE_k]` is the *no-consolidation upper bound*, a cell labeled κ=1 is actually *looser* than
  unit-tight after consolidation — a conservative bias that understates L2, never inflates it). `[CAL]` κ
  ladder, α, the supply distribution.

- **D-A19 (rev v4) — two-sided spot (capped); fallback = 1.5× worst-spot-route.** Each flight carries a
  **spot capacity** (drawn cap) at rate = base × `m`, `m` from the sourced two-sided band (~0.85 soft … ~1.18
  peak; memory `reference_air_spot_contract_ratio`), drawn from the supply stream. The spot cap is a **new
  explicit per-arc constraint `Σ_k cw_k·x_{k,a} ≤ cap^spot_a`** summed over the arc's riders — **NOT** a
  reuse of C.5c (which caps *actual* weight and assumes a per-group CW var that coload spot offers do not
  build); it bills on `cw_k` exactly as coload already does. **Fallback** is unlimited at
  **`1.5 × [top-spot-rate · CW_k · max-air-legs + the HAWB's full ground/trucking chain]`** — 1.5× the *worst
  realistic spot route*, where **`max-air-legs` is graph-derived** (`max` over HAWBs of the air-leg count in
  `A_k`, computed from the built graph — not a literal `2`, which D-A24's expanded grid could under-bound) —
  so it **dominates every real route's total cost** (a single-rate `1.5×` would not
  dominate a circuitous multi-leg+trucking route), while staying well-conditioned (no $40k/$1M). **Amends
  D-A12** (which locked `2× worst-feasible-route` → now `1.5× worst-spot-route`). Feasibility comes from the
  fallback arc, not infinite spot — killing the "dump on cheap infinite spot" trivialization that would zero
  reshuffle value at high α.

- **D-A20 / D-A21 — WITHDRAWN.** Integer ULD positions mean the **existing C.5 / C.5b two-dimensional
  allotment is the contracted-capacity gate as-is**: `Σ w_k x ≤ 1500·η`, `Σ v_k x ≤ 4.5·η`, `η ≤ N_f`. It
  already rations **weight and volume together**, so the original "weight-only kg cap can't ration volume"
  defect (an artifact of the continuous C.5c approach) **dissolves**. No SE constraint, no `cap_a`, no
  suppress-C.5b, no occupancy floor. The κ knob is simply the **drawn integer `N_f`** per flight, replacing
  `n_uld = max(1, round(2·scale))`. **CW stays only in billing** (C.13b), unchanged. **Invariant (F7):**
  `BsaContract.cap == {}` for all contracts in the headline scenario — the weight-only `C.5c-uld` secondary
  cap must never be populated on a contracted arc (asserted in the generator + a test), so the withdrawn
  machinery cannot leak back.

- **D-A23 (rev S38) — `M₁'` is the competent single-pass baseline; `M₀` is the greedy ablation.** *(The
  S35-approved D-A23 made `M₀` the optimal single-pass arm; that role moves to `M₁'`, and `M₀` is demoted to
  the myopic baseline — see D-A11.)* `M₁'` **optimally consolidates each cycle's newly-revealed HAWBs**
  (priors pinned) — the competent no-reshuffle planner we ship. `M₀` places the **same** newcomers
  **greedily/myopically** under a deterministic order `(tender_at, tier, shipment_id)`. `M₁` may reshuffle the
  whole open book. So the headline **`L2 = C(M₁') − C(M₁)`** measures **cross-cycle reshuffling (= replanning)**;
  the within-batch consolidation a naive greedy leaves on the table is the *separate* `C(M₀) − C(M₁')`
  (within-cycle optimization value, a component of `L1`, never folded into the replan headline). **Report the
  fraction of draws with `L2 = 0`** as a diagnostic (if large, the arrival permutation — not recourse — is
  driving the result). *(Sharpens §4 / D-A11.)*

- **D-A10 (rev v4) — dedicated control cell retired; the sweep's loose corner is GATED as the null.** No
  separately-constructed control instance. Instead the **abundant-capacity × even-supply × early-arrival
  corner of the (κ,α,λ) sweep — already computed — carries a pre-registered pass condition: `|L2| < CI`
  there** (a regime where replan must NOT help). That restores falsifiability at **zero extra construction**.
  The regret floor `C(π_hind) ≤ C(M₁)` (D-A13) is retained but **only as a labeled integration self-check** —
  it holds **by construction** (π_hind has the superset full-information feasible set), so it catches *bugs*,
  not a false thesis, and is **not** the falsifiability guard. Directional credibility additionally reads off
  the full (κ,α,λ) sweep shape.

**Unchanged invariants.** D-A16 frozen-across-arms applies to the **drawn integer network capacity vector** —
**bit-identical across `H₀/M₀/M₁/M₁'/π_hind`**, computed once in generation, persisted, read-only (no arm
recomputes it). D-A12 reshuffle decomposition (with the fallback amendment above). CRN **three-stream
separation (demand / supply / rates each own RNG sub-stream)** — varying κ or α must leave the demand draw
**byte-identical** (hard-gated test). **Consolidation coherence (F5, B1=A):** every MAWB-candidate arc is a
**single physical airport-pair flight** (arc IDs airport-pair-specific, never region-level), and every rider
reaches that arc's tail airport via a **priced trucking arc** — asserted by an invariant/test, so region→region
routing can't consolidate cargo that departs different airports onto one MAWB.

**Build implication for F1.** *Generator:* independent integer network-supply draw (κ + α multinomial, **own
`supply` RNG stream**); per-airport trucking matrix + region-O→D demand + multi-O/D subgraphs (B1=A); capped
spot (per-arc CW-sum constraint, rate = base × m) + fallback @ 1.5× worst-spot-route; density-mix cargo;
retire `capacity_scale` / `n_uld`-as-κ; namespace arc keys (airport-pair-specific). *MILP:* **reuse C.5/C.5b
as the contracted gate** (feed the drawn integer `N_f`; assert `BsaContract.cap=={}`); add the **spot
per-arc CW-sum cap** (new constraint, not C.5c); route-based fallback pricing; region routing in graph-gen.
*Tests:* supply independent of demand (vary κ/α ⇒ demand byte-identical); capacity vector frozen across arms;
C.5b binds correctly in 2D; spot CW cap binds; fallback dominates every real route; M₀ deterministic; **loose
corner `|L2|<CI` gate**; consolidation-coherence invariant; tractability at region→region size; static path
byte-identical.
