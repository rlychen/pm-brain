# Arrival-Only Replan Methodology — v0.1

**Status: APPROVED (Session 32, 2026-06-09).** **Gate: G-Method — cleared.** This is the **governing
methodology** for the air replan-savings proof; it amends the two prior gated specs —
`air_transit_time.md` (reconciled to v0.3) and `backtest_methodology.md` (reconciled to v0.5) — per the
reconciliation map in §7. Decisions D-A1..D-A4 locked to the recommended defaults (cutoff-only tender;
arrival lateness coupled to tier; ground/customs at the mean; predicate-9 retired for air). Nothing is
sampled to create failure: the realization is **deterministic** and the only stochastic process in the
proof is the **demand-arrival stream**.

> **✅ §14 (S46 capacity redesign) — APPROVED (Session 47).** §14 is the Phase-0 gate deliverable for the
> S46 capacity-redesign build, **approved with D1–D5 locked** (D1 flat+decay / D2 build belly+deferred /
> D3 sweep both settings / D4 both OTP forms / D5 two-number real-vs-penalty cost, kept separate). It
> supersedes the parts of §13 mapped in §14.7 (κ→`τ_ℓ`; decision-clock spot cap; arrival-spread mixture).
> **Phase 1 (generator) may now begin.** §13's unsuperseded content still governs.

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

### 6.1 Mid-shipment recourse anchoring + node-anchored fallback (build requirement, S44)

A disruption can strike while a shipment is **in transit**. The replan must NOT restart from the origin —
the cargo has already moved. This governs **both** the replay/backtest recourse **and** the production air
planning system's disruption handling (same model).

- **Pull-state model.** Each planning run pulls every active shipment's state. **New** shipments plan from
  origin (first-time). An **in-transit** shipment is **not replanned unless a disruption/delay degrades it**
  (otherwise its committed route stands).
- **Anchor = head of the last departed arc.** When an in-transit shipment IS replanned, recovery starts from
  the **head node of the last arc the cargo has departed** — equivalently, the last arc whose departure ≤ the
  decision clock. Two cases, one rule:
  - *Next arc not yet departed:* the cargo waits at the head of the previously-departed arc → replan from there.
  - *Current arc already departed (airborne):* the flight has left the tail and **will arrive at the head with
    certainty** (delayed or not) → replan from that head, available at the arc's (delayed) ETA.
  The tail is never the anchor (a departed arc's tail is behind the cargo). Departed arcs are immutable;
  not-yet-departed arcs are re-routable.
- **Node-anchored fallback (feasibility guarantee).** Emit a fallback arc from the **anchor node** to the
  destination door — NOT the origin→dest fallback, which an in-transit cargo can no longer take (its origin
  outflow is spent on the flown prefix, so the origin-anchored fallback is structurally unusable). It **arrives
  at T^abs**; cost = `1.5 × longest-UB(anchor→dest)`, or **`trivial = 1.0` if no feasible real path** from the
  anchor (then it is the only exit — the cargo takes it and lands at T^abs). Its cost is **added to** the
  already-billed flown prefix (the prefix is flown/sunk; the fallback does not replace it).

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

**Status: APPROVED (Session 35); PARTIALLY SUPERSEDED by §14 (S47).** Governing for the F1 build.
Supersedes D-A10, D-A12 (fallback), D-A18–D-A21; adds D-A23, D-A24. Three critique rounds folded (v1
circular-supply → v2 → v3 → v4); final convergence verdict APPROVE-WITH-MINOR-EDITS, edits applied.

> **⚠ Reconciled with §14 (S46 capacity redesign, approved S47).** Three items below are superseded —
> read §14 for the live versions: **(1) D-A18** — the κ-scalar is generalized to per-lane `τ_ℓ`
> (§14.1); the analytic-demand / supply-independence / CRN discipline carries over verbatim. **(2)
> D-A19** — the spot per-arc CW cap is now **decision-clock-dependent** (the booking-curve decay
> `K_a(t)`, §14.2); under the locked LEAN path the flat per-arc cap is **retained**, not replaced by an
> increasing-block curve (that is deferred to v2). **(3) D-A7 / §10** — the binary tier-coupled-arrival
> flag is replaced by the per-tier lead-time bucket mixture (§14.3). Everything else in §13 (region→region
> routing D-A24, the five arms + chain, fallback at 1.5× worst-spot-route, the frozen-capacity-vector
> invariant) stands unchanged. See §14.7 for the full map.

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

## 14. S46 capacity redesign — temporal capacity, arrival spread, and the three metrics (APPROVED v1.0, Session 47)

**Status: APPROVED (Session 47).** **Gate: G-Method (S46 capacity amendment) — cleared.** This is the
methodology amendment for the S46 capacity-redesign build; D1–D5 are locked (§14.6). Phase 1 (generator)
may begin against it. It supersedes the parts of §13 named in §14.7, and was **de-risked by a kill-shot** (5 seeds,
W on and off — `scripts/killshot_s46.py`, S46) that proved the mechanism below makes **capacity bind**
at proof scale (the two replan arms route different chargeable-kg per capacity tier — the quantity S45
measured as $0). Every functional form in §14.2–§14.3 is the form the kill-shot ran and verified
feasible (70/70 HAWBs, all 5 seeds). Sourcing tags: **SOURCED** (public + URL), **INFERRED** (derived
from a sourced anchor), **MRN** (market-research-needed — flagged, not fabricated).

### 14.0 Why (the S45 problem this fixes)

S45's L2 decomposition (`docs/design/l2_decomposition_s45.md`) proved the headline `L2 = C(M₁') − C(M₁)`
was **100% consolidation reshuffle / 0% capacity** and **κ-inert**: both arms routed byte-identical
total spot kg, used zero contracted, touched zero fallback. Root cause was structural — spot was
flat-priced and effectively unbounded (~48× over-supply) and contracted was *more expensive* than spot,
so the optimizer rode spot for everything and the capacity dimension never bound. Finiteness alone does
**not** fix this (red-team `05_redteam.md` F1/F2: both cost-minimizers fill cheap-first and spill the
**same** kg to a flat fallback ⇒ `L2_capacity ≈ 0`). The fix needs three real-world mechanisms that turn
contention from a low-probability accident into a **designed-in, deterministic** event:

1. **Per-lane finite, tiered capacity** with contracted re-anchored **below** spot (§14.1).
2. **Capacity that decays toward departure** (the booking curve) so the spot escape **closes** exactly
   when a late, captive high-value shipment needs it (§14.2). *This is the load-bearing mechanism.*
3. **An arrival spread with a within-tier late tail** so some EXPRESS shipments reliably hit the
   depleted book (§14.3).

The mechanism: a late captive EXPRESS finds spot decayed shut and the only cheap seat is a contracted
position an early DEFERRED is squatting on. **M₁** (open book) bumps the still-untendered DEFERRED to a
later flight and seats the EXPRESS; **M₁'** has the DEFERRED pinned and spills the EXPRESS to fallback.
The arms then route **different total kg per tier** — the quantity F1/F2 require and S45 never produced.

### 14.1 Region-to-region tightness `τ_R` — generalizes κ (supersedes §13 D-A18 κ-scalar)

*(Decision, S47: tightness is set at the **region-to-region** granularity, NOT per airport-lane. A
HAWB's lane (origin airport × dest gateway) is an optimizer decision under D-A24, so the generator
cannot assign demand to a lane. But a HAWB's **region** is a demand attribute — fixed by its door — so
`τ_R = capacity/demand` per region-pair is well-defined with no geometric `q_ℓ` guesswork. The optimizer
still routes within the region-pair; that within-region substitution is the value source.)*

The single global tightness dial `τ = Σ capacity / Σ demand` over the network, with `D = n_hawbs ·
E[cw_k]` the **analytic** expected demand (closed-form, zero demand draws ⇒ CRN preserved). `τ < 1` ⇒
demand exceeds supply. **Regions: one East-Asia origin region × three US dest regions (LAX / SFO / ORD)
⇒ 3 region-pairs indexed by dest gateway.** `τ` maps down deterministically: total `S = τ·D` → per-region
`S_R = τ_R · D_R`, `D_R = n · q_R · E[cw_k]` → split across tiers by the composition vector (§14.4) →
spread ULD positions over the region's flights via `Multinomial(N_R, Dirichlet(α))`.

- **`q_R`** = analytic geometric demand share to region R = the fraction of the dest door box whose
  **nearest** gateway is R (the `_nominal_gateway` rule), integrated on a deterministic haversine grid.
  Reads zero demand draws ⇒ CRN-safe, like `E[cw_k]`. (Geometry favors ORD — the dest box's eastern
  majority is nearest Chicago.)
- **`τ_R`** is drawn per region in **short / balanced / slack** buckets (bands `[0.6,0.85] / [0.95,1.05] /
  [1.2,1.6]`, INFERRED), then rescaled so the demand-weighted mean returns `τ` exactly
  (`Σ_R q_R·τ_R = τ`). The bucket→region assignment is seed-fixed (reproducible); the within-band draw +
  the demand/supply realization carry the randomness — so the *distribution* of short/balanced/slack is
  controllable but realized per-region tightness still fluctuates (the mismatch that is the value source
  is preserved). A HAWB's dest region is fixed by its door, but the optimizer may still seat it via a
  *different* gateway (e.g. an LA-region HAWB via SFO) — the within-region substitution.
- **κ is subsumed**, not discarded: it is the special case `τ_R = const` for all R, contracted-tier-only.
  Keep the uniform-`τ` arm for continuity with S38–S45.
- **α keeps its role**, now governing the spread of `N_R` positions *across flights within a region*
  (between-flight lumpiness); `τ_R` is the between-region mean. Orthogonal, both retained.
- **Supply-independence (§13 D-A18) holds verbatim:** `τ_R` sets a *mean over analytic expected demand*;
  no supply quantity reads the realized book. CRN gate (vary `τ`/α ⇒ demand byte-identical) unchanged.

**Built S47 (slice 1a, isolation-tested):** `_expected_cw_mean` (`E[cw_k]`≈552 kg), `_size_total_supply`
(`S=τ·D`), `_dest_region_shares` (`q_R`), `_draw_region_tightness` (`τ_R`, normalized to `τ`),
`_size_region_supply` (`S_R`, sums to `τ·D`). **Still to wire (next slice):** add `tau` / `region_mix` to
`GenConfig`; route `S_R` into the per-flight position draw (`_draw_network_supply` per region) +
composition split; choose the RNG sub-stream for the `τ_R` draw (a dedicated `region_tightness` stream
for cleanest CRN — add to `RNG_STREAMS`).

---

### 14.1-R (S50) — Region redefinition: door-distance/SLA candidacy + O-D-lane tightness + ground gate

*(Status: **APPROVED v1.0 (S50)** — D-A25…D-A30 approved one-by-one by the user. Supersedes the
single-airport region model of §14.1 above. Motivated by a 4-agent investigation, S50 —
origin/destination arc + region-sizing audit. Two defects found; the first is already fixed
(commit `7c08539`), the second is what this revision addresses.)*

**Why revise.** The §14.1 model set each destination *region* = one airport (LAX / SFO / ORD) over a
single East-Asia origin region, with `q_R` a nearest-*airport* partition of one door bounding box. The
audit surfaced two problems:

1. **Correctness (FIXED, standalone — commit `7c08539`).** Truck-drayage candidacy was pure great-circle
   k-NN (1500 km, no land/water test), so **every HAWB got a priced truck arc across the Taiwan Strait**
   (mainland doors → TPE and vice-versa) — a physically impossible route the MILP used freely. Fixed by a
   **ground-connectivity gate**: `freightnet.ground_group(coord)` classifies a point's landmass, and
   `geo_select` restricts a door's truck-drayage **seeds** to its own landmass (air legs between hubs are
   untouched — cargo *flies* TPE→HKG→US, it does not truck). Verified: 0 cross-landmass truck arcs
   (was 100 %), 0 % structural fallback, feasible + deterministic.
2. **Region incoherence (THIS revision).** LAX and SFO are ~560 km apart (both West Coast, road-connected,
   ~7 h drive) — one *substitutable* market. Splitting them into "short" (LAX) and "slack" (SFO) made the
   3-way short/balanced/slack dial collapse to ≈1.5-way (SFO's "slack" capacity just absorbed LAX's
   overflow → an effective single mildly-short West-Coast region at S/D ≈ 0.82), and there was **no East
   Coast** at all (the door box stopped at Chicago). The intended three-distinct-tightness-regions
   experiment was not realized geographically.

**Candidacy is DOOR-CENTRIC, not clustering (D-A25).** Which airports a door may truck to is decided by
**distance/time from that door**, never by grouping airports together. Airport-to-airport clustering is
wrong: two airports each 500 mi from a door but in *opposite* directions are 1000 mi apart, yet **both are
valid candidates** for that door — a cluster keyed on airport-airport distance would wrongly split them.
So a door↔airport truck arc is a candidate iff **(a)** same ground-connectivity group (landmass — the
committed `ground_group` gate, the hard physical rule that blocks the Taiwan-Strait crossing) **and (b)**
the door is within *reach* of the airport. This is measured per door (a radius / time budget around the
door), never as a static region. Config-driven — the reach knob lives in `GeoSelectConfig`.

**Reach = distance now, committed-SLA time budget next (D-A26).** The current reach knob is a distance
radius: `GeoSelectConfig.seed_radius_km = 1500 km`, applied to **both** origin and destination doors (plus
`corridor_phi=1.3` / `max_air_legs=3` on hubs/hops). The principled successor is a **committed-SLA time
budget**: a route is a candidate iff its **end-to-end transit ≤ the committed SLA `Δ_k`** (D-F6 v2,
`Δ_k = ready + base_transit + sla_offset`, which the model already carries). Example: East-China → Phoenix
SLA = 5 days ⇒ *any* end-to-end route delivering within 5 days is valid, regardless of which gateway it
uses. This unifies candidacy with the existing deadline and retires the inert 1500 km first-hit seed rule
(`extend_until_reachable(min_count=1)` froze the radius and made `seed_k` inert — audit finding). Near-term
build surfaces the radius to config; the SLA-time frontier is the successor (needs route-time propagation
at graph-gen).

**Regions are a SUPPLY-INDEXING partition, decoupled from candidacy (D-A25b).** The origin/dest regions
below exist only to index *tightness/supply* (§ below). They are **not** a candidacy boundary — a door's
routable airports come from D-A25's door-reach set, which the geography (ground-gate + reach) keeps local
without any region enforcement. A HAWB's tightness *lane* is a demand attribute (its nearest origin
cluster × nearest dest cluster); the optimizer's realized routing may differ (within-dest gateway
substitution — an LA door via SFO — is a value source).

**Destination regions — three metro clusters (D-A27, expanded-metros decision, S50).** All member
airports already exist in FreightNet (no DB work); each needs hand-authored flights in the schedule
substrate (below). Final membership is a confirmation item (each added airport = added flights).

| Region | Role slot | Proposed member airports |
|---|---|---|
| **West Coast** | (tightness swept) | LAX, SFO, OAK, ONT, SNA, BUR, SJC |
| **Midwest** | (tightness swept) | ORD, DTW, MDW |
| **East Coast** | (tightness swept) | JFK, EWR, LGA, BOS |

The `_DEST_BBOX` widens east (to ~−71 lon) so demand doors are drawn across all three metros. Which
metro is "short/balanced/slack" is the swept `region_mix`, no longer pinned to LAX-short.

**Origin regions — three, separated by landmass + door-reach (D-A28, S50).** Taiwan is a separate landmass
(the ground-gate blocks any TPE↔mainland drayage). HKG and PVG are the same landmass but ~1255 km apart —
beyond the door-reach radius (D-A26) — so a PRD door never reaches PVG and vice-versa; they serve disjoint
door populations, hence distinct supply-indexing regions. No unrealistic mainland cross-drayage.

| Origin region | Landmass | Proposed member airports | Door box |
|---|---|---|---|
| **Taiwan** | TW (island) | TPE | Taiwan bbox |
| **PRD** (Pearl River Delta) | EA-mainland | HKG (+CAN) | PRD bbox |
| **East China** | EA-mainland | PVG (+SHA) | Yangtze-delta bbox |

Origin doors are drawn from **per-region boxes** (no single straddling box), so a door's landmass is
unambiguous and the ground gate is exact. Tightness is keyed on the **(origin × dest) lane** with origin
the dominant axis (D-A29 below) — NOT on the dest region alone. The three origin regions are therefore the
*primary* scarcity axis (a door's origin region is inescapable under the ground-gate), which is exactly
what the sourced research found.

**Schedule substrate additions.** The timetable is hardcoded offer literals (`tpeb_air_instance.py`); the
audit confirmed adding gateways is localized code surgery, not config. Needed: (a) `Gateway` literals +
hand-authored `_direct`/`_thru` flights reaching each **new** dest gateway (East-Coast JFK/EWR/BOS,
Midwest DTW/MDW, West OAK/ONT/…), ≥2 cheap options per lane to preserve the B5 "≥2 options" invariant;
(b) per-gateway promised transit hours in `_DEST_BASE_TRANSIT_H` (CALIBRATION — grounded, not guessed);
(c) origin flights from all three origin regions; (d) widen `_DEST_BBOX`. Belly/freighter split, decay,
composition, hard-BSA, cost-split, tardiness-always-on, and CRN discipline all carry over unchanged.

**O-D lane tightness (D-A29).** Tightness is re-keyed from dest-region `S_R` to **(origin-region ×
dest-region) LANE** `S_ij` — 3×3 = **9 lanes** — with **origin the dominant axis** (SOURCED: same US
dest, different Asian origins → different rates the same week; Baltic BAI30-HongKong vs BAI80-Shanghai
move on different tracks). Origin and dest doors draw independently so the geometric lane share
**factorizes** `q_ij = q^O_i · q^D_j` (both pure box geometry, **zero demand draws** — CRN-safe). The
global dial maps down **separably**: `τ_ij = τ·(u_i/ū)·(v_j/v̄)`, origin multipliers `u_i` from **WIDE**
short/balanced/slack bands, dest multipliers `v_j` from **NARROW** bands — the band-width asymmetry
(`Var(u) ≫ Var(v)`) *is* origin dominance. `Σ_ij q_ij·τ_ij = τ` exactly (proven + probe-validated);
κ is the special case `u=v=1`.

**Lane cap decomposition + freighter repositioning (D-A30).** Each `S_ij` carves into three pools —
**frozen first, freighter the remainder**: `S_ij = BSA + belly + freighter`. BSA (integer ULDs, soft/hard)
and belly (the D2 `×0.4` thinning) are **lane-frozen** (contracted on a named sector; belly is a
passenger-schedule byproduct — both SOURCED). The **freighter/spot pool is the sole repositionable
capacity**: within each dest region it conserves the freighter budget `G_j` and redistributes across
**origins** toward **expected residual demand** `R_ij = max(0, D_ij − BSA − belly)` (analytic — zero demand
draws), dialed by `reposition_rho ∈ [0,1]`: `F_ij = (1−ρ)·F⁰_ij + ρ·G_j·(R_ij/ΣR)`. **Supply⊥demand
(D-A18) preserved**: every input is analytic/geometry and repositioning draws no RNG, so ρ/τ/α sweeps leave
the `demand` stream byte-identical; realized mismatch still emerges (freighters sit on *expected* residual,
realized doors/timing scatter). Generation-time, frozen across arms (D-A16); the §14.2 booking curve acts
on top. `ρ` defaults to **0 (null / negative-control = static lane draw)**, headline swept `{0, 0.5, 1.0}`.
**Full design + grounding + proofs: `docs/design/s50_lane_tightness_freighter_repositioning.md`.**

**Sizing re-key (lane-keyed; the arithmetic is unchanged, only the grouping key; region path only, legacy
κ path untouched).** NEW: `_origin_region_shares` (q^O_i, mirror of the dest partition), `_draw_lane_tightness`
(u/v on the existing `region_tightness` stream), `_reposition_freighter_spot`, `_expected_residual`; config
`reposition_rho=0.0` + `origin_mix`; WIDE/NARROW band tables. RE-KEY `off.dest` →
`lane(off)=(region_of(off.origin), region_of(off.dest))` at `_size_lane_supply`, `_draw_lane_network_supply`,
`_build_region_rate_catalog`, `_split_contracted` (hard-BSA on the tightest lane first); add airport→region
maps for BOTH axes + the region-name→airports cluster maps (`_DEST_REGION_GATEWAYS`/`_DEFAULT_REGION_MIX`
→ clusters). **No new RNG stream** (repositioning deterministic). `_expected_cw_mean`, `_size_total_supply`,
`_spread_positions`, `_is_belly`/belly thinning, decay, all MILP capacity gates, and every CRN gate are
unchanged.

**LEAN build (first, for the proof).** `_origin_region_shares` + factorized `q_ij`; separable
origin-dominant `τ_ij` (WIDE origin / NARROW dest bands); lane-keyed three-pool carve; `_reposition_freighter_spot`
with `reposition_rho` swept `{0, 0.5, 1.0}`, per-dest-region conservation, generation-time frozen. Reuse the
existing belly `×0.4`, soft/hard BSA, CapDecay, cost split, CRN gates. **Deferred to v2 (FULL):** ANC
cross-dest repositioning; an explicit passenger-schedule belly proxy; yield-weighted (not residual-only)
repositioning; the SLA-time-budget candidacy frontier (D-A26 successor).

**Open calibration items (resolve after the redesigned generator runs).** (i) the door-reach knob —
`seed_radius_km` now, committed-SLA time budget later (D-A26); (ii) per-gateway transit hours (grounded,
not guessed); (iii) final metro membership (each airport = flights to author); (iv) **`reposition_rho`
headline value** (MRN — swept meanwhile); (v) **origin vs dest tightness band widths** (the origin-dominance
lever; INFERRED — calibrate to BAI30/BAI80 + WorldACD origin dispersion); (vi) **the operating point** —
tune the `τ` ladder + `origin_mix`/`region_mix` so a *normal* cell lands at realistic fallback/OTP rather
than today's 43–67 % (becomes tractable once tightness is correctly origin-lane-keyed — the West-Coast
split no longer forces a permanent short region, and the biggest-scarcity axis is now the right one). The
two-cost split, both OTP forms, and provisional-L1 caveats (§14.5, H0 parked) are unchanged.

**Risks to gate at build (from the design doc §6).** R1 — over-repositioning (`ρ→1`) can shrink expected
mismatch; L2 must survive on *realized* (count+timing) mismatch — if it collapses at the headline `ρ`,
report the ρ-sensitivity honestly (the null is a finding). R2 — a `demand`-draw leak in `F_ij` would break
CRN (extend the CRN gate to cover `reposition_rho`). R3 — `q_ij=q^O·q^D` holds only while doors draw
independently (assert it). R4 — assert `Σ_ij S_ij` invariant pre/post reposition (global τ untouched).
R5 — few flights/lane at proof scale ⇒ lumpy; report per-lane binding-rate, not a network average.

---

### 14.2 Time-decaying capacity — the booking curve (the binding mechanism)

For an arc `a` on a flight departing at absolute time `T_a`, viewed at decision clock `t`, **available-
to-book** capacity is the time-zero amplitude `C0_a` (drawn in generation, frozen across arms per §13
D-A16) scaled by a **booking curve** `φ_a(·)` and floored at firm holdings:

**Booking curve (D-T1, CONVEX; GROUNDED S51).** Supersedes the S46/S49 LINEAR form. `τ` = time before the
**cargo cutoff** (`τ = cutoff_a − t`, in days; cutoff ≈ 3–6h before STD is the grounded `τ=0`):
```
  φ_a(τ) = A_cut,a + (1 − A_cut,a) · (1 − e^(−λ_a·τ))
```
Convex / back-loaded (marginal fill largest near the cutoff): availability holds near 1 for weeks then
falls steeply in the final ~1–2 weeks. `A_cut,a` = **cutoff bookable fraction** (residual free space when
cargo closes); `λ_a` = per-day fill rate. **Per-flight draw (D-T2b), on the `cap_decay` supply-side stream
— never reads the demand book (D-A18):** `λ_a ~ U(0.10, 0.16)/day`, and `A_cut,a ~ Beta(·)` **right-skewed,
deck-differentiated** — freighter `Beta(1.3, 8.7)` (mean 0.13) **<** belly `Beta(1.8, 6.2)` (mean 0.225).

**Why these values (GROUNDED — S51 research, two independent agents; supersedes the S49 fit-to-target):**
the S49 gentle `φ_min ~ U(0.30,0.50)` linear curve was reverse-engineered to hit a fabricated 2–8% fallback
target — not grounded. Independent research established: (i) bookings are heavily **back-loaded** — McKinsey
*Ahead of the curve* (2023): **<40% of a flight is booked 2 weeks out**, the bulk filling in the final
week — which **rejects the linear form** and fixes `λ` (~0.10–0.16/day puts most depletion in the last
1–2 weeks); (ii) the **cutoff bookable fraction is low and right-skewed**, triangulated from **dynamic
(volume) load factors ~57–65%** at departure (CLIVE/Xeneta) minus unusable trim — with mass near 0 for
flights that **cube out** before cutoff; (iii) our lanes are **transpacific HEADHAUL** (Asia→US, the tight
import direction — we model no backhaul), so `A_cut` sits at the tight end, **~0.15 central**, not the
direction-blended ~0.30; (iv) **freighters run hotter than belly** (freighter dynamic LF ~75–82% vs belly
~50–58%; IATA freighter CLF 63% vs belly 42%) → lower `A_cut`. Sources + evidence tables:
`docs/design/decay_model_research_s51.md`. **Tags:** shape SOURCED (McKinsey + load-factor curves); `A_cut`
central INFERRED (no public per-flight residual-at-cutoff dataset — the irreducible ±0.07 uncertainty);
the deck ordering SOURCED. Code: the `DecayParams` default. Consequence: **higher, regime-dependent fallback
than the S49 fit — that is the honest grounded result; the 2–8% "normal" target is retired (§14.5).**

**The decay model — ONLY SPOT DECAYS (F1, S51; supersedes the S46 "free pool" model).** The earlier
model decayed the soft-BSA *free* positions continuously (like spot). **F1 corrected that: a held
booking does not decay out from under you** (grounded — a soft BSA/allotment is the forwarder's
*reserved* space, firm until a release deadline; §14.2 sources, Amaruchkul 2018). Three tiers, three
rules:

| tier | rule (available at clock `t`) |
|---|---|
| **Spot** (float kg) | `avail_a(t) = booked_a + max(0, C0_a − booked_a) · φ_a(t)`, `booked_a = Σ cw_k` over cargo pinned (TENDERED) on `a`. The **only** pool that decays (the convex booking curve above). |
| **Soft-BSA** (`per_flight`, integer ULD positions) | **FIRM (no decay) until the release cliff `dep − 48h`** (item-1 below), then capped to the used-at-cliff positions. |
| **Hard-BSA** (`equalized`, take-or-pay) | **does NOT decay** — fully reserved take-or-pay space, the stable reshuffle anchor (D-T2a). |

**Soft-BSA release cliff (item-1, S51).** A soft BSA is a held block, firm until the forwarder's
**release deadline `dep − 48h`** (SOURCED: Amaruchkul 2018 — the penalty-free release cutoff, ~48h
before departure; the season-level minimum-utilization clause ~60–70% is v2). The keep/release
decision is **deferred to the last minute** — a planning cadence point is added at each soft-BSA
flight's `dep − 48h`; on that run the positions the plan **uses** are **locked** (kept at the BSA rate
through tender), the rest **released** to the carrier (gone — a BSA flight has no spot arc to receive
them; released ≠ our spot). One-time per flight; capped thereafter. Locking what is *used* protects
every committed booking (tendered cargo is among the used ⇒ never released). Min-utilization,
tier/peak-dependence, and partial-release are v2.

**The decay floor is tendered-only — Model Y (item-2, S51; supersedes the freeze-from-placement
floor).** Only **committed** (tendered/locked) capacity is protected from spot decay; an un-booked
*plan* is not. This fixes an artifact: the old floor protected M₁''s *placed-but-untendered* cargo,
handing the rigid arm the OTP benefit of early booking it never paid for.

| arm | pinned set fed to the spot floor |
|---|---|
| `M₁`, `M₁'`, `π_hind` | **tendered only** (`state.tendered_set()`) |
| `M₀` | tendered ∪ placed visible priors (greedy, no reshuffle recourse) |
| `H₀` | all placed routes (daily human, likewise floored) |

**M₁' rigidity recourse (item-3, S51).** M₁' still hard-pins its placed priors (that *is* single-pass),
but its floor is now tendered-only — so a placed, un-tendered prior on a spot arc that decays below its
load makes the pinned solve INFEASIBLE. M₁' cannot reshuffle (that is M₁'s value), so the honest
recourse is to **dump** the invalidated prior to the uncapped fallback (`_repair_frozen_infeasible`);
a fail-fast guard raises if that ever leaves the solve infeasible (it can't: tendered capacity is
floored, so exhausting the movable priors is always feasible). Consequence: under a filling market,
**M₁ adapts (reshuffles off decaying arcs) while M₁' eats fallback** — flexibility becomes a *service*
asset, not just a cost one, and the "M₁ OTP always < M₁'" artifact is gone.

**Implication for the value mechanism (D-T2a):** hard-BSA never decays and soft-BSA is firm until its
cliff, so the stable reservoir the late EXPRESS bumps into is the contracted block; only **spot**
depletes. Hard + soft BSA = the reservoir; spot = the depleting market.

**Determinism + the optimizer:** `φ_a` is a deterministic function of `(cutoff_a − t)` and the
generation-time draw; the soft-BSA cliff reads the deterministic solve + schedule (no RNG, no demand
draw ⇒ CRN-safe). The MILP needs **no constraint change** — the decayed spot value feeds the per-cycle
`spot_cap`, and the cliff-capped soft-BSA `allotment` feeds C.5/C.5b, which read them directly. The
ledger stays on static caps; the *planning view* (spot decay + soft-BSA cliff) is what changes.

### 14.3 Arrival spread — per-tier lead-time bucket mixture (sharpens §10 / D-A7)

The current binary `tier_coupled_arrival` flag offers no within-tier spread (every EXPRESS looks the
same). Replace it with a **per-tier lead-time bucket mixture** over book-lead `B` (hours before the `d*`
cutoff):

| bucket | `B` range (h) | meaning |
|---|---|---|
| early | [96, 144] | planned well ahead |
| medium | [48, 96] | normal |
| late | [12, 48] | short-fuse |
| very-late | [`min_b`=6, 12] | just-in-time (existing prep+dispatch floor) |

Each tier carries a bucket-weight row `π_tier` (a 3×4 matrix — the one load-bearing knob). Draw `bucket ~
Categorical(π_tier)`, then `B ~ U(bucket_range)`. Illustrative (INFERRED; tail magnitudes MRN):

| tier | early | medium | late | very-late |
|---|---:|---:|---:|---:|
| EXPRESS | 0.15 | 0.25 | 0.35 | **0.25** |
| STANDARD | 0.25 | 0.40 | 0.25 | 0.10 |
| DEFERRED | **0.50** | 0.30 | 0.15 | 0.05 |

This gives (i) a genuine early→very-late spread, (ii) **within-tier** spread — EXPRESS carries a designed
heavy late tail (the captive shipment) *and* a small early mass, (iii) DEFERRED skewed early (the natural
bump candidate, arrives long before its cutoff, sits untendered). **CRN discipline:** the categorical +
within-bucket uniform are drawn on the **`demand`** stream in a **fixed draw order and count** (always
draw both; the bucket only selects the range) — so any capacity knob (`τ_ℓ`, φ, H) leaves demand
byte-identical (the hard CRN gate). **No planted scenario (D-T3):** scale `n` so `E[express] ≈ 12–16` ⇒
`P(≥1 very-late express) ≈ 0.95+` by natural draw — realistic arrivals only, no hard-wired captive.

### 14.4 Capacity composition + hard-BSA wiring

Total lane capacity `S_ℓ` splits across tiers by a composition vector. Within contracted, a
`hard_bsa_frac` knob splits soft (`per_flight`) vs hard (`equalized`, sunk allowance `A_c = positions·π`,
drawn from the *analytic* expected block on the `supply` stream — never the realized book). **Hard-BSA is
BUILT-NOT-GENERATED today** (`_build_c13a_equalized`, `over_c`, `allowance_kg`, `r_c` all exist; the
generator never emits `settlement="equalized"`) — the wiring is the fix; the MILP is unchanged. Both
contracted rates re-anchored to **~$4.2/kg** (< base spot **$5.5/kg**, both SOURCED Xeneta NE-Asia→NA
Apr-26) so contracted is worth filling — **the S45 root-cause fix**; without it `ψ_cap` stays 0.

**Deck split — belly vs freighter spot caps (D2, build now; GROUNDED S49).** Flights are tagged `deck ∈
{belly, freighter}`. Belly (passenger-aircraft hold) carries spot/co-load only and **no BSA**; freighter
(main-deck cargo) carries the contracted blocks plus its spot leftover. The one knob is the **belly
spot-cap fraction = 0.4** of a freighter's spot cap (e.g. freighter spot ≈ 3,000 kg / ~2 ULD → belly ≈
1,200–1,500 kg / ~1 ULD). **Derivation:** real per-aircraft *total* cargo payload puts a long-haul
passenger belly at **~0.15** of a transpacific freighter (belly ~12–21 t: 787-9 ≈ 12.8 t, A350-900 ≈
14.7 t, 777-300ER ≈ 21 t with a full pax load; freighter ~100–139 t: 777F ≈ 102 t, 747-400F ≈ 128 t,
747-8F ≈ 139 t). But the *knob is the SPOT cap, not total capacity*: a freighter carves most of its hold
to BSA and leaves only its spot remainder, while belly is **all** spot — so the spot-cap ratio lifts to
`0.15 / ≈0.35 (freighter spot share) ≈ 0.4`. **Sources** (sourced S49 — see memory
`reference_belly_freighter_capacity`): Boeing 777 ACAP (boeing.com), Atlas Air 747-400F sheet, Wikipedia
747-8, ResearchGate SFO–SIN A350/787 belly case study, STAT Times/Cargojet 777-300ER belly, IATA
transpacific-freighter-share report. **Confidence medium**; biggest caveat — belly often flies partly
empty / is pax-weight-limited, so the *effective* peak fraction can sit below 0.4 (the `τ ≤ 0.8` stress
arm captures that). No MILP change — belly just gets a smaller `spot_cap` number on its arcs.

### 14.5 The three metric families (the deliverable measures)

**Family 1 — cost (D5: TWO numbers, kept SEPARATE — never summed into a single "total cash"; the
capacity/consolidation split is RETIRED).** Each arm reports exactly two cost numbers, side by side:
- **`real_cost`** (real money) — everything genuinely paid for capacity: co-load/spot freight +
  flat-rate bucket + min-flat-breaks bucket + BSA per-flight pivot + BSA equalized overage + **sunk
  hard-BSA cost** + ground/dwell/handling + per-MAWB fix + Path-A/Path-B surcharges.
- **`fallback_penalty`** (a MODELING penalty, not real cash) — cost charged on `ArcType.FALLBACK` arcs
  only (the relief-valve markup). Reported as its own number so it can never inflate a real-money figure.

**Do NOT combine them into one total.** They are two different things — real spend vs how hard the plan
leaned on the fake relief valve — and combining hides the trade. (The C.10 quadratic tardiness penalty is
likewise excluded from both — it surfaces in Family 2.)

**Savings — reported as a PAIR at every level (never a single combined L2):**
- **Real-money savings:** `L2_real$ = real_cost(M₁') − real_cost(M₁)`, `L1_real$`, `Total_real$ =
  real_cost(H₀) − real_cost(M₁)`, with percents off the matching real base: `L2_real% = L2_real$ /
  real_cost(M₁')`, `Total_real% = Total_real$ / real_cost(H₀)`.
- **Fallback-penalty reduction (separate, the "fake" diagnostic):** `L2_pen$ = fallback_penalty(M₁') −
  fallback_penalty(M₁)`, `Total_pen$ = fallback_penalty(H₀) − fallback_penalty(M₁)`. Read alongside the
  Family-3 fallback count/% — together they say how much relief-valve the replan avoided.

So the open-book win shows honestly as a pair, e.g. *"real money +$5,880, fallback penalty −$19,250,
EXPRESS now on-time"* — not a single netted "+$13,370 saved." **No `ψ_cap`, no `L2_capacity`, no
consolidation bucket.**

**Sunk-cost note:** the sunk hard-BSA cost is frozen and identical across all arms (D-A16), so it
**cancels in every real-cost difference** — it sets the absolute `real_cost` level only, never a saving
delta. Reported in `real_cost` for honesty, flagged as delta-invariant.

**Family 2 — OTP / tardiness.** Each scored `k` lands in exactly one of three delivery states:

| state | predicate | arrival |
|---|---|---|
| **on_time_real** | `¬fallback ∧ A_k ≤ Δ_k` | walk `A_k` |
| **late_real** | `¬fallback ∧ A_k > Δ_k` | walk `A_k` (real route, missed promise) |
| **fallback** | `on_fallback` | `A_k = T^abs_k` (backstop; always late — assert `T^abs > Δ_k`) |

**On-time def (precise):** on-time iff delivered on a **real (non-backstop) route** within the **frozen**
promise `Δ_k` (`A_k ≤ Δ_k`, strict; not the live replan-mutable deadline, not `T^abs`). `OTP = n_on_time
/ N`. Tardiness `tard_k = max(0, A_k − Δ_k)`; report `mean / median / p95` over the delayed set,
`total_tardiness`, and the cash-equivalent `Σ_k W_k · tard_k` (linear; the C.10 quadratic stays out of
the cash metric). Emit the full state vector per arm.

**Family 3 — fallback / no-feasible-route incidence.** `fallback% = fallback_count / N`, per arm, with a
three-cause split: **(a) structural-infeasible** (no real path in the subgraph — a floor every arm
shares), **(b) capacity-exhaustion roll** (a real path exists but contracted+spot is exhausted — the
arm-varying component the redesign moves), **(c) disruption-induced** (≡ 0 in the clean headline).
`fallback_count(M₁') − fallback_count(M₁)` = fallback avoidance by replanning.

**Calibration targets (SOURCED where possible, see `03_metrics.md` §4 for URLs):**

| metric | target band | regime | confidence |
|---|---|---|---|
| carrier leg-level OTP (DAP, 6-h window) | **62.7%** | normal-to-stressed | SOURCED (industry 2025); a leg grain, but see the door-OTP RE-ANCHOR below |
| **door-to-door forwarder OTP vs `Δ_k`** (what we score) — **RE-ANCHORED S49** | **~0.50–0.65 blended, tier-differentiated** (express highest, deferred lowest) | by regime | GROUNDED to reality — industry DAP 62.7%, 80% = "good", **no published per-tier door figure** |
| roll / fallback incidence — **a CAPACITY signal (S49)** | **~2–8% normal / 15–30% tight-peak** | by `τ` | INFERRED; fallback fires when no real-flight slot exists (capacity-forced, not a cost choice — bump-test confirmed) |
| replan-savings % | low-to-high single-digit % of freight spend | — | INFERRED (3%→10% expedite-gap proxy); direct figure **MRN** |
| fallback price multiple (our 2.5× base) | 2–4× band; 2.5× sits low-realistic ⇒ conservative | — | SOURCED |

> **RE-ANCHOR (S49, user-approved).** The prior door-OTP target **80–92% was a phantom** (INFERRED/MRN) — it
> was flagged as "the biggest calibration risk," and the penalty-on n=70 calibration confirms OTP saturates
> ~0.5–0.66 no matter the supply, because it's bound by on-time-capable flight capacity at commit time, not
> by incentive. Grounded reality: industry DAP **62.7%**, 80% = "good", no per-tier door figure published.
> So the target is **~0.50–0.65 blended, tier-differentiated** (express penalized hardest → highest OTP per
> the money-back-guarantee norm; deferred lowest). The proof reports OTP as the **honest cost↔service
> tradeoff** (next paragraph), not a number forced to 0.9. Single-seed (grid); the D3 sweep confirms it.

**The headline = a cost↔service tradeoff PAIR (S49, user-approved), not a single forced OTP.** The arms
trade off: **M₁' (commits early) → higher OTP / higher real cost**; **M₁ (open-book) → lower real cost /
lower OTP** (it keeps cargo flexible for cost, so some shipments miss the early on-time flights). Report
`L2_real% = real_cost(M₁')−real_cost(M₁)` (the replan saving) **beside** the OTP/fallback delta — the value
is "open-book saves real money at a stated service cost," an honest pair. Forcing OTP to 0.9 would fight
both reality and the model's correct behavior.

### 14.6 Decisions for one-pass approval — D1–D5

| # | decision | options | **recommendation** |
|---|---|---|---|
| **D1** | spot pricing model | (a) flat-finite per-arc cap **+ decay** (kill-shot-proven; zero MILP change) vs (b) increasing-block convex lane curve (architecture `01` type 4) | **(a) flat-finite + decay.** The kill-shot proved capacity binds with flat caps + the booking curve; the block curve prices total-kg-per-lane (held equal by both arms — S45 showed it amplifies L2 by **$0**). **Defer the block curve to v2** as a defensibility refinement, not the κ-fix. |
| **D2** | belly/freighter split + deferred-air topology | build now vs defer | **defer to v2.** Schedule-substrate realism; the three metrics work without it. Keeps the LEAN path minimal. |
| **D3** | realism knobs (composition / τ regime) | the `01`/`02` defaults vs the realism-review corrections | **adopt the `02_realism` corrections:** mid-market composition **~0.55 contracted / 0.30–0.40 spot+co-load**, **`hard_bsa_frac ≈ 0.35`**; **τ base = 1.0–1.15** (balanced standing regime — `τ<1` network-wide is **PEAK**, not normal), **τ ≤ 0.8 a labeled stress arm**. Widen the density floor to ~60–100 kg/cbm so volumetric cargo actually competes. |
| **D4** | OTP metric | strict binary OTP vs richer reporting | **report all three delivery states; lead with `fallback%` + `tardiness-p95`** (the truer service metrics — strict binary OTP under-credits open-book's spread-mild-lateness behavior, backwards from shipper value). **Grace-band OTP secondary** (decide the band width in Phase 3). |
| **D5** | cost reporting | capacity-vs-consolidation attribution (ψ_cap etc.) vs a simple objective split | **LOCKED (user, S47): two-number real-vs-penalty split.** Report `real_cost` (all capacity paid + sunk) and `fallback_penalty` (FALLBACK-arc cost) per arm; `C = real + penalty`. **The capacity/consolidation split + `ψ_cap` are RETIRED** (confounded by weight-break rating; user rejected). |

**DECISIONS LOCKED WITH THE USER (Session 47) — these override the recommendation column above:**

- **D1 = (a) flat-finite + decay.** The increasing-block spot curve is **deferred to v2**. Capacity
  binds via the booking-curve decay (§14.2); zero MILP constraint change.
- **D2 = belly/freighter split BUILT (S49); deferred-air arc DROPPED (S49, grounded).** **Phase 1 grew**
  to include the deck-split schedule-substrate rework (tag flights `deck ∈ {belly, freighter}`; belly =
  thinner spot/co-load caps, freighter = ULD/BSA). No new MILP constraint — belly just gets a smaller
  `spot_cap` number. **GROUNDED S49:** belly spot cap = **0.4 ×** freighter spot cap (derivation + sources
  in §14.4 / memory `reference_belly_freighter_capacity`). **The deferred-air arc is DROPPED** (was
  "build now" in the S47 lock): research (memory `reference_deferred_air_capacity`) found deferred/economy
  air is **not a separate capacity pool** — it is the lowest-priority commercial tier sold on the **same**
  space-available spot/leftover space, so a separate-capacity deferred arc would **double-count** the very
  pool the belly split rations. The deferred *behavior* (slow, cheap routing for slack cargo) already
  emerges from the shared spot pool × DEFERRED's slack deadline × cost-minimization; and the deferred
  *discount* is shipper-revenue-side, not forwarder-cost-side, so it is out of scope for the cost-side
  L2 objective. True charter (#7) and any deferred-product modeling **remain v2.**
- **D3 = SWEEP BOTH SETTINGS on all three knobs** (not a single pick). The three become reported axes:
  **composition ∈ {0.55, 0.70} contracted**, **`hard_bsa_frac` ∈ {0.35, 0.50}**, **τ across the bands
  RE-ANCHORED S49** (below). The user accepts the larger instance count to see
  how each knob plays out. Each cell still labels its regime (normal vs peak) so no number is
  mis-presented. *(Implication for §14.1 / the C0–C2 ladder: the headline sweep is a small factorial over
  these knobs × τ; the Tractability phase must re-confirm the budget at this expanded count — and the
  penalty-on model is ~2× slower, so this re-confirm is load-bearing.)*
  > **τ BANDS RE-ANCHORED (S49, penalty-on calibration; supersedes "normal 1.0–1.15 / stress ≤0.8").**
  > With the gentle decay (§14.2) and the penalty ON, the realistic regimes shifted UP: **normal ≈ τ
  > 1.5–2.0** (fallback ~2–11%, OTP ~0.48–0.59), **peak/stress ≈ τ 1.0** (fallback ~40%+, OTP ~0.26),
  > **loose ≈ τ 3.0** (fallback ~1%, OTP ~0.66). "τ 1.0–1.15 = balanced normal" was wrong — at τ=1.0 the
  > supply⟂demand mismatch + decay + ULD granularity leave only ~40% of nominal capacity usable at commit,
  > i.e. deep-peak. Sweep **τ ∈ {1.0 (peak), 1.5, 2.0 (normal), 3.0 (loose)}**. Single-seed; the sweep
  > confirms across seeds.
- **D4 = REPORT BOTH (a) and (b).** Report the single strict-binary OTP number (a) **and** the rich
  three-state view (b) — `on_time_real / late_real / fallback`, fallback%, tardiness-p95, grace-band
  OTP. Lead the *narrative* with fallback% + p95 (the truer service signal), but surface binary OTP
  alongside so the two can be compared.
- **D5 = TWO numbers, kept SEPARATE (never summed).** Per arm report `real_cost` (co-load/spot + flat +
  MFB + BSA pivot + BSA overage + sunk hard-BSA + ground/handling + surcharges) **and** `fallback_penalty`
  (FALLBACK-arc cost) **side by side — do NOT combine into a total**. Savings reported as a pair: real-money
  (`L2_real$`/`L2_real%`, `Total_real$`/`Total_real%`) and fallback-penalty reduction (`L2_pen$`) separately.
  C.10 tardiness stays out of both (→ Family 2). **The capacity-vs-consolidation attribution and `ψ_cap`
  are RETIRED** — the dollar split was confounded by IATA weight-break rating. Sunk hard-BSA cost cancels
  in all real deltas (D-A16). All other current metrics retained.

**Savings reporting (user, S47) — REAL and PENALTY kept separate; each percent normalized by its OWN
real base.** Never a single combined "L2". Report:
- **`L2_real$ = real_cost(M₁') − real_cost(M₁)`** + **`L2_real% = L2_real$ / real_cost(M₁')`** — the
  real-money replan saving as a fraction of the single-pass real cost.
- **`Total_real$ = real_cost(H₀) − real_cost(M₁)`** + **`Total_real% = Total_real$ / real_cost(H₀)`** —
  the full automation+replan real-money saving as a fraction of human real spend.
- **`L2_pen$` / `Total_pen$`** — the fallback-penalty reduction (the "fake" relief-valve diagnostic),
  reported as its own dollars beside the Family-3 fallback count/%. **Never folded into the real-money %.**

*(Rationale: the open-book arm often spends MORE real money to avoid the fallback relief valve — netting
them into one number would hide that the real cost rose while the fake penalty fell. Keep the pair.
Supersedes the combined-`C` wording earlier in this draft.)*

**Build scope this amendment gates (post-D1–D3):** flat spot + decay; **belly/freighter + deferred-air
in-build**; the three realism knobs swept on both settings; D4/D5 to be locked. Defer only the
increasing-block curve and true charter to v2.

### 14.7 What this supersedes / amends in §13

- **D-A18** — κ-scalar generalized to per-lane `τ_ℓ` (§14.1). Analytic-expected-demand discipline,
  `supply`-stream independence, CRN gate, and the frozen-capacity-vector-across-arms invariant all hold
  verbatim.
- **D-A19** — the spot per-arc CW cap becomes **decision-clock-dependent**: `K_a(t) = booked_a + max(0,
  C0_a − booked_a)·φ_a(t)` (§14.2). Fallback unchanged (1.5× worst-spot-route, route-based). Under the
  LEAN path (D1a) the **flat per-arc cap is retained** (not replaced by the block curve) — so C.5d
  stands and only its *value* decays per cycle.
- **D-A7 / §10** — the binary tier-coupled-arrival flag is replaced by the per-tier lead-time bucket
  mixture (§14.3); `tier_coupled_arrival=True` survives as the all-mass-in-one-bucket special case.
- **Unchanged:** D-A16 (frozen capacity vector — now includes `C0_a` and the per-flight φ-jitter draw),
  D-A12 (reshuffle decomposition + C.10 excluded from cash), D-A11/D-A23 (the five arms + chain), D-A24
  (region→region routing), the three-stream CRN separation (now four: `demand` / `supply` / `rates` /
  `cap_decay`).

### 14.8 Definition-of-done for Phase 0 (this gate)

This amendment is **approved** when the user has signed off on: (1) the corrected decay model (§14.2),
(2) the three metric families + the door-OTP target band + the `ψ_cap ≥ 0.30` gate (§14.5), (3) the
arrival-spread mixture (§14.3), and (4) **D1–D5** (§14.6). On approval: reconcile §13 per §14.7, set the
top-of-file pending banner to APPROVED, and start Phase 1 (generator). The kill-shot
(`scripts/killshot_s46.py`) is SCRATCH — Phase 1 rebuilds its monkeypatched hook into tested components.
