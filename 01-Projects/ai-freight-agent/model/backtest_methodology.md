# Backtest Methodology — The Replan-Savings Proof (Air)

**Status:** v0.5 — **APPROVED (Session 32, 2026-06-09)** — see the v0.5 amendment note below;
governed by `model/arrival_only_replan_methodology.md`. **Gate: G-Method.** Authorizes
Stage 3 (the thesis proof) and, per the gate-ordering rule (a method spec is approved **before** the
components it constrains), is written and approved **before** the transit-time
model (2b) and orchestrator (2c) — its information-set / no-lookahead / capacity-integrity /
frozen-promise requirements are **build constraints** on those components, not after-the-fact checks.

Extends `product_thesis.md §2`; operationalizes `PRD.md §5.2` (counterfactual / regret) and
`§5.6` (probabilistic planning). The human baseline policy is specified in
`model/human_planning_heuristic.md`.

> **v0.5 amendment (Session 32, user-driven) — arrival-only realization; transit deterministic.**
> Governed by `model/arrival_only_replan_methodology.md` (approved S32). The v0.2 reframe is now taken
> to its conclusion: the **only** stochastic process is the demand-arrival stream. **Realization is
> deterministic** (scheduled block + ground at the mean) — the v0.4 "one draw per leg → the actual"
> language in §2/§4/§6 is superseded; **OTP is deterministic-given-routing** (still a population metric
> over shipments/tiers, scored vs the frozen promise, but no draws). `π_hind` knows the full demand
> stream at `t=0` (transit is deterministic, so there are no "realized actuals" to also reveal).
> **`z_tier` / `σ̂` / predicate-9 retire for air** → deterministic deadline feasibility `A ≤ Δ_k`
> (`air_transit_time.md` v0.3); they revive for ocean. **Arms (sharpened):** `M₀` = incremental-greedy
> (slot new arrivals into remaining capacity, don't disturb prior soft commitments); `M₁` =
> re-optimize the open (un-tendered) book each cycle — same arrival stream. **L2 = `C(M₀)−C(M₁)` is a
> conservative lower bound** (it holds with perfectly reliable transit + zero disruptions, so it cannot
> be attacked as manufactured). **Disruption recourse is a TESTED CAPABILITY, not a value source** —
> kept out of the headline scenario, verified by three deterministic 2c fixtures (absorbable-delay
> no-op / connection-break unlock-and-reroute / cancellation reroute-from-current); recourse =
> replan-from-current-position, past legs immutable, **promise holds** (disrupted-late = miss, no
> renegotiation). See `arrival_only_replan_methodology.md §6`. CIs come from `R` seeded demand-arrival
> replications (the only remaining randomness).
>
> **Critique-11 hardening (Session 32) — `arrival_only_replan_methodology.md §12`, D-A9..D-A16.** Items that
> touch THIS doc: (C4/D-A12) `realized_cost` / `C(π)` **excludes the C.10 tardiness penalty** (objective-steering,
> not a cash outflow) and the headline is gated on `L2_reshuffle` (3-way split, retire the $1M fallback);
> (C7/D-A14) the **headline `H₀` is batch-at-cutoff** (§0/§3), on-arrival `H₀` (the §4 "known limitation") is the
> upper bracket only; (M-B1/D-A15) "conservative lower bound" scoped to **transit reliability only**; (M-B2/D-A16)
> BSA pacing `cap_a`/`A_c` join `W_k`/`z_tier` as **control inputs frozen + bit-identical across arms**; (C3/D-A11)
> add the **`M₁'` pinned-replan control arm** (`C(M₁')==C(M₀)`) to net tie-break leakage out of `L2`; (C2/D-A10)
> pre-registered null + a required negative-control cell. Sound-no-change: lookahead, double-spend, denominator.
>
> **v0.2 reframe (kept).** Uncertainty for air is **demand arrival**, not transit time. Air
> flights run roughly on time; the replan value comes from HAWBs arriving over the sim clock and
> the ability to **reshuffle the not-yet-tendered book** (re-consolidate MAWBs, re-allocate
> scarce capacity). Transit is low-variance (sampled for OTP, not the value driver). Transit
> uncertainty is an **ocean** driver (Stage 4).
>
> **v0.4 changes (Session 29).** Two structural corrections, user-driven. (A) **No per-route
> Monte-Carlo.** Each shipment is realized **once** (one draw per leg → the actual); **OTP is a
> population-over-time** on-time fraction (binary per shipment vs. the SLA deadline), not a per-route
> probability. CIs come from `R` seeded **horizon replications**, not per-route draws. (B) **OTP is
> *controlled* at graph-gen + the per-shipment penalty, not by a chance constraint:** a deterministic
> tier-reliability admission filter (`air_transit_time.md §5`) is the primary lever; `W_k` is a
> prioritization control input, **frozen during the proof**. Recourse (roll vs. replan) is the
> orchestrator's policy, not a transit function. In-MILP quantile binding / chance constraints are
> **not in MVP**. These supersede the v0.3 "N end-to-end MC draws per route" language in §2/§6/§8.
>
> **v0.3 changes (Session 28).** (1) **Four-arm decomposition** — added a human-heuristic
> baseline `H₀` and a MILP-no-replan middle arm `M₀` so the cost delta splits cleanly into
> **planning value (L1)** and **replan value (L2)**; without `M₀` the human-vs-product number
> conflates the two. (2) **Cost–OTP frontier via a dollarized α-lever** (`min α·cost$ +
> (1−α)·lateness$`), traced at the peak cell. (3) **OTP scored vs. a promise frozen at booking**
> (ungameable). (4) Canonical example corrected to a **reactive** (not anticipatory) reshuffle —
> anticipatory slot-holding would be lookahead. (5) Smaller credibility fixes folded in
> (next-flight-on-contract option, convexity as a falsifiable hypothesis, physical-tender lock,
> paired CIs, literature bracket demoted to a sanity check).

---

## 0. Notation

| Symbol | Meaning |
|---|---|
| `H₀` | **Human-heuristic baseline.** Simple, spreadsheet-executable planner (`human_planning_heuristic.md`): per-cutoff greedy consolidation, FCFS on cheap/contracted capacity, break-only recourse, no proactive re-optimization. |
| `M₀` | **MILP, incremental-greedy (no open-book replan).** Slots each new arrival into remaining capacity; does **not** re-optimize prior soft (un-tendered) commitments. Same commitment timing as `H₀`, solved with the air MILP. Isolates solver quality. |
| `M₁` | **MILP, rolling replan (the product, L2).** Re-optimizes the **open (not-yet-tendered)** book each sim step — reshuffles wherever permitted. |
| `π_hind` | Offline clairvoyant — full demand stream known at `t=0` (transit is deterministic), solved once subject to physical feasibility only (no info/tender-lock). Regret floor (§3). |
| `α` | Objective tradeoff lever `∈ [0,1]`: `min α·cost$ + (1−α)·lateness$` (both dollars). Swept to trace the cost–OTP curve (§8). |
| `L1` | **Planning value** = `C(H₀) − C(M₀)` — the solver out-plans the spreadsheet on a static snapshot. Real, near-term-sellable value (§4). |
| `L2` | **Replan value — the thesis headline** = `C(M₀) − C(M₁)`. |
| `Total` | Customer-facing value = `C(H₀) − C(M₁)` = `L1 + L2`. |
| `C(π)` | Realized cost of `π` over a sim run (operating cost: freight + consolidation + spot/recovery), evaluated on realized actuals. |
| `C_hind` | Clairvoyant cost. Regret floor. |
| `Reg(π)` | `C(π) − C_hind` (≥ 0). |
| `I_t` | **Information set** at sim-clock `t`: rows with `known_at ≤ t` — chiefly which HAWBs have arrived + current capacity state. First-class object (§5). |
| `κ` | **Capacity tightness** — committed-demand-to-capacity ratio. Primary congestion axis (§7). |
| `λ` | **Arrival lateness** — how much of the book is still unknown when cutoffs hit. Second axis (§7). |
| `R` | Seeded **horizon replications** for confidence intervals on deltas — each realizes a whole **demand-arrival** stream once (the only remaining randomness; transit is deterministic). *Not* per-route Monte-Carlo. Bumped at the peak cell (§8). |
| `z_tier` | (ocean-only, retired for air per v0.5) per-tier reliability safety multiplier for the graph-gen OTP admission filter. For air, predicate-9 → deterministic `A ≤ Δ_k`. |
| `cw_flex` | Chargeable weight of the flexible book — per-flexible-kg denominator (§6); frozen at `t=0`, arm-invariant; a conservative lower-bound rate, not an attribution (`flexibility_model.md`). |

---

## 1. What is being measured

`product_thesis.md §2`: the value is **replan of the flexible portion of the book** — for air,
as **demand arrives over time against finite capacity**. Operationally:

> Over a sim clock, HAWBs arrive sequentially. Compare a human-heuristic planner and a
> MILP-no-replan planner (both commit per cutoff, fix only breakage) against a MILP rolling-replan
> planner that reshuffles the not-yet-tendered book. **Decompose** the cost gap into planning
> value (L1) and replan value (L2), report L2 as a `savings(congestion)` **band** over a 2-D
> sweep, plus a cost–OTP frontier.

The mechanism (canonical case — **reactive, not anticipatory**):

> One cheap contracted ULD slot ($1,000; else spot $3,000). A **flexible** HAWB arrives early and
> a greedy planner books it into the cheap slot (cheapest feasible *now* — no foresight). Later,
> before the flexible HAWB has physically tendered, an **urgent** HAWB arrives needing that slot.
> `M₀`/`H₀` do not revisit the earlier booking → urgent goes spot → $4,000. `M₁` re-optimizes the
> still-reshufflable book → moves the flexible HAWB to a later cheap flight (it has slack), frees
> the slot for the urgent one → $2,000.

The win is **reactive reshuffle of the not-yet-tendered book**, never speculative slot-holding:
`M₁` does *not* leave the slot empty on Day 1 anticipating the urgent HAWB (that would require
knowing the future and would trip the §5 lookahead tripwire). It books greedily like everyone
else and *reverses* the reversible booking when the urgent HAWB actually appears. All arms share
the **same physical-tender lock** (§2); the only difference is that `M₁` uses the revision freedom
and `M₀`/`H₀` do not.

L2 scales convexly with `κ`: abundant capacity → any order works → L2 ≈ 0; tight capacity →
commitment order dominates → L2 rises. Convexity is a **hypothesis to test, not assume** (§7).

---

## 2. The experiment

- **Unit of analysis = the whole simulation, not a shipment.** One result = one full multi-day
  horizon (e.g., ~30 days of arrivals, batched planning, the entire book). Every metric — cost,
  OTP, L1/L2, savings% — is a **portfolio/horizon aggregate** (total cost over the book; fraction
  of all shipments on-time). There is **no per-shipment improvement claim**; the value is a
  property of planning the whole moving book, not any single routing.
- **Stochastic process = the HAWB arrival stream.** Pre-generated with future-dated `known_at`,
  revealed as `known_at ≤ sim_clock` (the `synthetic_data_contract.md` reveal mechanism). The
  arrival schedule (timing + weight/volume/deadline/lane/service-tier per HAWB) is the realization.
- **Transit: low-variance, single-draw realization.** Policies **plan on the estimate** (published
  ETA); the sim **scores on a single realized actual per leg** (one draw per leg, walked into one
  end-to-end arrival — `air_transit_time.md §4`). **No per-route Monte-Carlo:** each shipment is
  realized **once**; OTP is the on-time fraction over the **shipment population** in the period (§6).
  Not the value driver, but genuinely realized so OTP is a real number and the occasional connection
  slip fires as a break-trigger. *(Air transit is low-variance and static through time; a refining
  ETA forecast + cancellations are an ocean/Stage-4 driver.)*
- **Capacity options must include the realistic middle.** The alternative to a held contracted
  slot is **not** only premium spot at ~3×. Include **roll-to-next-flight-on-contract** (late but
  cheap — just a later-departure arc in the existing air graph) so the no-replan failure mode is
  "late but cheap," not always "on-time but 3×." Without it the baseline's worst case is
  one-sided and L2 is exaggerated.
- **Physical-tender lock.** A HAWB becomes irreversible at **physical tender** (CFS receipt,
  ~24–48h pre-flight), not at the notional flight cutoff or a soft firm flag. All arms share this
  identical, externally-imposed lock clock — so the only behavioral difference between `M₀` and
  `M₁` is whether they revise the pre-tender set, never an asymmetric commitment deadline.
- **Instance.** Air slice, seeded from `data/synthetic/tpeb_air_instance.py`, scaled by the
  generator (2a). Start at **15–30 HAWBs** with a tractability checkpoint before scale-up.
- **Common random numbers.** Per `(κ, λ)` cell, `R` seeded horizon replications; **CRN spans both
  the arrival stream and the per-leg transit draws** — paired across all arms, so the measured delta
  is policy difference, not sampling noise.
- **Plan-on-estimate, score-on-actual.** A policy plans using `I_t` + ETAs; it is scored on
  realized actuals. The estimate→actual progression is what break-triggers and `M₁` recourse react
  to; the dominant value driver remains demand arrival.

---

## 3. The policy set

Four arms, scored on identical CRN streams. The two MILP arms (`M₀`, `M₁`) use the existing air
MILP (`src/components/air_milp.py`); `H₀` uses the heuristic in `human_planning_heuristic.md`.

| Arm | Solver | Commitment / recourse | Role |
|---|---|---|---|
| **H₀** | spreadsheet heuristic | per-cutoff greedy batch; break-only recourse; no proactive re-opt | Realistic human baseline |
| **M₀** | air MILP | **same as H₀** (per-cutoff batch, break-only) | MILP-no-replan — isolates solver quality |
| **M₁** | air MILP | re-optimizes the not-yet-tendered book each step | The product (L2) |
| **π_hind** | air MILP | full demand + realized actual ETAs at `t=0`; physical-feasible only, no tender lock | Regret floor `C_hind` |

**Why four arms (the L1/L2 decomposition).** Comparing only `H₀` vs `M₁` fuses two effects: the
MILP plans better than a spreadsheet even with zero replanning (**L1**), and the MILP replans
while the human does not (**L2**). The thesis claim is specifically L2 ("not out-planning a human
on a static snapshot"). `M₀` is the middle arm that splits them:

- **L1 = C(H₀) − C(M₀)** — solver vs. spreadsheet on the *same* commitment behavior. Clean
  (only the planner differs).
- **L2 = C(M₀) − C(M₁)** — replan vs. no-replan with the *same* solver. Clean (only recourse
  differs). **This is the headline.**
- **Total = C(H₀) − C(M₁) = L1 + L2** — the customer-facing number, reported as L1+L2, never
  mislabeled as replan value.

The decomposition is exact and additive. It also *tests* (rather than assumes) the claim that
"the main difference is replanning": if true, `M₀ ≈ H₀` and L2 carries the total; if L1 is large,
that is itself important to know.

**π_hind** uses the point-in-time replay machinery validated in the S2 spike (closed-form
hindsight on a 3-HAWB/5-step instance; regret matches by hand; no double-spend). 3c productionizes it.

**π_hind's constraint set (must be stated, or the floor is invalid).** π_hind solves **once** with
**perfect foresight of both the full demand stream AND the realized transit actuals** (the
final-sample per-leg ETAs, not estimates) and the single static spot price (cheapest realized, if
spot is ever made intra-sim time-varying — L3 era). It is subject to **physical feasibility only**
(capacity, MCT, deadlines) and is **free of the information constraint and the
sequential-commitment / physical-tender lock** (with foresight there is no reason to commit early).
We give the clairvoyant every advantage, even unrealistic ones — that is the point of a lower bound.
Consequently `Reg(M₁) = C(M₁) − C_hind` mixes **two irreducible gaps** — information (M₁ can't see
the future) *and* commitment-structure (M₁ must commit sequentially, π_hind needn't) — and **neither
is recoverable headroom** (§6). If π_hind were left subject to the tender lock it would not be a true
clairvoyant and M₁ could beat it on some draw (`Reg < 0`), silently breaking the
`C_hind ≤ M₁ ≤ M₀ ≤ H₀` chain.

---

## 4. Why the decomposition is honest — and why BOTH layers are real value

**L2 is confound-free by construction.** `M₀` and `M₁` share the same solver, the same arrival
stream through the same `I_t`, the same near-deterministic transit, the same physical-tender lock,
**and the same frozen control inputs (`z_tier`, `W_k`)**. The *only* difference is that `M₁`
re-optimizes the not-yet-tendered set. So `C(M₀) − C(M₁)` is the value of recourse over
sequentially-revealed demand — by construction, not by argument. The v0.1 "you measured your own
forecast error" objection does not arise (no forecaster asymmetry; the uncertainty is *which HAWBs
have arrived*, seen identically by both).

**Re-screening is *part of* L2, declared not hidden.** `M₁` re-runs graph generation (incl. the
predicate-9 tier-reliability screen) at each replan step on the current schedule/capacity state, so
its admitted route set legitimately refreshes; `M₀` screens once at commit. This *is* recourse value
(re-optimizing includes re-screening against fresher state) and is **defined into L2 on purpose** —
not silently bundled. `z_tier` itself is frozen per shipment (§6), so the refresh changes the
*admissible set as schedules move*, never the *tier stringency*. A reviewer asking "is L2 re-solving
or re-screening?" gets: both, by definition of replanning the open book.

**L1 is real value, not a throwaway "commodity."** It is tempting to dismiss L1 as "an AI can
build a planner in a weekend." That conflates *technical replicability* with *commercial reality*:
no one has productized MILP planning for the mid/small market — only the largest forwarders/BCOs
have it, built internally. That gap is a genuine near-term wedge with real sellable value. The
decomposition exists to **attribute** the number honestly (so L2 is not credited with L1's work),
**not** to diminish planning. Frame it as: **L1 = the planning wedge** (unbuilt for this segment,
sellable now, weaker long-run defensibility) and **L2 = the replan moat** (harder, compounds with
the estimate-vs-actual data corpus). Both go in the story; the split keeps it credible.

**Known limitation — H₀ commitment timing (deferred).** The current `H₀`/`M₀` commit close to
on-arrival, which is the *pessimistic* (L2-inflating) edge: a diligent human stages flexible cargo
and commits at cutoff, by which time some urgent demand has already arrived, shrinking the gap. This
is accepted **for now** — current `H₀` is treated as the conservative upper bracket on the human
baseline, and a more realistic batch-at-cutoff `H₀` is a deferred refinement. When built, report L2 at
the cutoff-commit number with on-arrival as the upper bracket. (Flagged so the headline isn't read as
final until the fair-human variant lands.)

**Remaining honesty obligations** are handled by the band (§7), not a further decomposition:
simulation tends to *overstate* (clean labels, well-behaved arrivals) and a single calm-quarter
partner *understates*; the band brackets the truth.

---

## 5. Information set & no-lookahead — build constraint → 2c

The orchestrator receives a **first-class, timestamped `I_t` object** containing only rows with
`known_at ≤ sim_clock`. No-lookahead is enforced by the object's contract (a policy cannot read a
HAWB that hasn't "arrived"), not by view discipline. `I_t` is the policy's sole channel to state.

**Demand lookahead tripwire (gated DoD).** Inject a future-only HAWB (`known_at > t`) that, if
leaked, would change the plan, and assert the plan at `t` is **bit-identical** to the run without
it. A leak is indistinguishable from a real saving (replan "magically" pre-positions for a
shipment it shouldn't know about), so this is non-negotiable. Implemented as a **pytest assertion**.

**Schedule lookahead — the channel the demand tripwire cannot catch.** `flex_k`/`Δ_k`/`A_k^min` are
frozen at `t=0` over the *full* schedule (`flexibility_model.md §2.3`). This is lookahead-**safe only
because air schedules are static** (`air_transit_time.md` D-T5: a published flight's existence is
knowable in advance — real knowledge, not leakage). State this assumption explicitly: **no-lookahead
rests on the static-schedule property for schedule fields, and the `I_t` contract for demand.** Add a
**second tripwire**: inject a future-only *flight* (or a cancellation at `t' > t`) and assert the plan
at `t` is bit-identical — the demand tripwire perturbs only demand rows and would miss a schedule
leak. **Stage-4/ocean breaks the static-schedule assumption** (blank sailings, re-timing) and requires
re-deriving the no-lookahead argument for schedule fields.

---

## 6. Metrics, capacity integrity & regret accounting — build constraint → 2c / 3c

**No capacity double-spend across re-solves (correctness crux) — stated as a conservation law, not a
decrement-ordering rule.** A double-spend is a *phantom saving indistinguishable from a real one*
(M₁ books into a slot it doesn't truly hold → lower cost → looks like reshuffle value). Decrement-
ordering ("decrement a tendered slot before the next run sees it") is necessary but not checkable; the
invariant is the **per-arc, per-step conservation identity**, asserted as a pytest and holding
identically in structure for `M₀` and `M₁`:

> For every capacity arc `a` and sim-step `t`:
> `cap_initial(a) = tendered(a,t) + committed_untendered(a,t) + free(a,t)`,
> with `tendered` monotone non-decreasing and irreversible; `committed_untendered` reversible by `M₁`
> only, and reversing it must **return the slot to `free` in the same step** (no slot in two ledgers);
> `M₁` may never reshuffle a `tendered` shipment; a "freed" slot must equal an actual ledger return,
> not an intention. `M₀` and `M₁` see the *identical* capacity state at the same sim-time.

**This invariant has never been exercised** — the Stage-0.5 spikes had no binding capacity and no
mid-horizon tender (they re-solve from full capacity each step). So 3c **must** add a **binding-
capacity + mid-horizon-tender test instance** (a flight locks while later HAWBs are still arriving)
and assert the identity each step. This is the single most important missing test in the proof.

**Fallback cost must be sized, and fallback-avoidance reported separately from reshuffle.** The
per-HAWB fallback cost is **`C^fallback(k) = 2 × (most expensive *feasible* real route for k)`**
(feasible = passes the pre-filter/deadline even if capacity may preclude it), computed from the
realized catalog — *not* the legacy flat `$1,000,000` (≈7× the model's own sizing rule; one
differential fallback at $1M ≈ 1000× a real reshuffle, so it would make L2 fallback-avoidance, not
reshuffle — the Stage-0.5 caveat). Then **report fallback counts per arm** and **split the headline**:
`L2_reshuffle` (freight-dollar savings from reshuffling — the thesis number) vs.
`L2_fallback-avoidance` (`C^fallback × Δ(fallback count)` — a real but distinct capacity-rescue value,
sized realistically). *(Implemented in the 3c harness, where per-HAWB route costs + fallback
accounting live; the static generator's instance constant is annotated as a placeholder.)*

**OTP is binary per shipment, a population-over-time metric in aggregate.** Per shipment, on-time is
the **binary** event `A ≤ Δ_k` (the effective deadline `Δ_k = min(contractual T_dead, tier T_SLA)`,
exogenous — the *same* `Δ_k` the air model's C.10 penalizes against). The reported **OTP%**
is the on-time fraction over the **whole shipment population** in the period (weekly→monthly→
quarterly) — never a per-shipment probability. The 90/80/70%-tier promises are portfolio targets hit
by *control* (below), not per-shipment chance constraints.

**OTP is controlled at graph-gen + the penalty, not by a chance constraint** (`air_transit_time.md
§5`). (1) The graph-gen **tier-reliability filter** admits only routes meeting a deterministic
reliability margin for the tier (`Â(r)+z_tier·σ̂(r) ≤ deadline`) — the primary control, tuned
closed-loop against measured OTP. (2) The per-shipment penalty `W_k` is a **prioritization control
input** (who wins scarce reliable capacity under contention), default a fixed per-tier ratio. **Both
control inputs are frozen during the proof** — the penalty especially: a live
OTP-to-date→penalty→OTP loop would confound L2 with tuning the very metric we report. The
prioritization lever is real and valuable but is **demonstrated as a separate capability**, never
live inside the L2 measurement.

**Frozen-at-booking invariants (pytest).** (a) **Frozen promise:** each HAWB's `Δ_k` promise
is set at booking firm-up, stored immutably, never recomputed on replan (else `M₁` games OTP by
re-promising downward). *(Because OTP is a population fraction, the residual gaming surface is not
per-shipment re-promising but **tier-mix shift** — `M₁` improving aggregate OTP by prioritizing
easy shipments and dumping hard ones to fallback; defended by the per-tier dominance check, §8.)* (b) **Frozen `z_tier`:** the tier reliability floor a shipment was *admitted*
under is frozen per shipment at booking — even though the *admitted route set* legitimately changes
as schedules/capacity evolve, and even if the *global* tier calibration drifts mid-period. Both
asserted bit-identical across re-solves.

**Reading the OTP deltas.**
- **Clean L2 OTP claim — `M₀` vs `M₁` only.** Same initial plan, same frozen promise; `M₀` lets
  realized OTP slip (break-only recourse), `M₁` defends it by replanning. Pure recourse — L1-free.
- **Commercial lens — `H₀` vs `M₁`** (and realized-vs-SLA across all arms): the L1+L2 "does the
  planner keep its SLA" story.
- **Report OTP and mean-tardiness separately, per tier and aggregate, at the peak cell.** The
  quadratic penalty minimizes mean tardiness, which is *correlated with but not identical to* the
  binary on-time count; reporting both is the falsification check for that metric/objective mismatch
  (the optimizer must not reduce mean tardiness while worsening the on-time fraction).

**Cost metrics — report percent AND per-flexible-kg** (user D-2):
- `savings%` per layer, e.g. `L2% = (C(M₀) − C(M₁)) / C(M₀)`; likewise `L1%`, `Total%`.
- **Per-flexible-kg**: `L2 / cw_flex`. Across the `λ` sweep the book composition changes, so
  percent and per-flexible-kg are *not* affine reparametrizations — both carry information; report
  both. Absolute-$ is dropped (book-size-dependent). **Caveat (`flexibility_model.md` D-F8):
  `cw_flex` is a sum of per-HAWB marginals while `L2` is a portfolio interaction, so `L2/cw_flex` is
  a conservative lower-bound *rate*, not an attribution.** The value-attributed companion is
  `L2 / (mass actually reshuffled onto/off a binding-capacity arc)` (the 2-FLEX ex-post diagnostic).
  `cw_flex` is frozen at `t=0` and identical across arms (D-F7).
- **Paired-CRN confidence interval on every published delta.** The whole payoff of CRN is a
  variance-reduced CI on the *difference*; report it (don't quote a bare point). Replication count
  per cell is set to hit a target CI half-width, not fixed at "many."

**Regret invariants** (S2 machinery):
- `Reg(π) ≥ 0` for every arm and draw.
- `C_hind ≤ C(M₁) ≤ C(M₀) ≤ C(H₀)` in expectation. Per-draw `C_hind ≤ ·` violations are hard
  bugs; per-draw `C(M₁) ≤ C(M₀)` violations on adversarial arrival orders are allowed and reported.
- **`Reg(M₁) = C(M₁) − C_hind` is partly irreducible** — the clairvoyant exploits information no
  online policy can have. Do **not** market this gap as "recoverable headroom"; it bounds the gap,
  it is not opportunity left on the table.

---

## 7. The deliverable: `L2 savings(congestion)` as a BAND over a 2-D sweep

**2-D sweep — both axes demand/timing:**
- **`κ` capacity tightness** (primary). **`λ` arrival lateness** (how much of the book is unknown
  when cutoffs hit; the demand-side analog of "drift severity").

Report L2 savings over the `(κ, λ)` grid as a **surface/heatmap**, with the peak a **named,
defensible regime** (e.g., "TPEB peak season + last-minute booking surge against a tight
allocation"). **Pre-register the grid and the peak-regime name before running**, so the regime is
not chosen post-hoc to maximize the number; arrival distributions and capacity ratios are reviewed
artifacts with documented provenance (lane structure from BTS FAF).

**Spot-vs-contract gap = a regime-mixture tied to κ, not a free scalar (the dominant $ knob).** The
$ size of every avoided premium scales with `spot/contract`, so it is the most controllable lever on
the L2 *dollar* figure — hence report L2 in **% as the headline**, and pin the ratio to real,
two-sided data rather than the canonical 3×. Sourced air-cargo anchors (2023–2026): soft market
~**0.85** (spot ~15% *below* contract), peak ~**1.18** (spot ~16–21% *above*); spot reacts ~2–3× faster
than contract, so the ratio swings across 1.0 by regime. **Model:** draw `spot/contract ~ LogNormal`
within a regime, regime **tied to the swept κ** (loose κ → soft → <1; tight κ → peak → >1) — so the
gap is pinned to the axis we already sweep, with sourced anchors, not adversary-pickable. Sourced: the
0.85/1.18 anchors + two-sidedness ([WorldACD/avitrader Nov-2024](https://avitrader.com/2024/11/29/air-cargo-rates-climb-as-peak-season-strengthens/),
[FreightWaves Aug-2023](https://www.freightwaves.com/news/wait-for-airfreight-recovery-could-extend-deep-into-2024),
[Supply Chain Dive/Xeneta](https://www.supplychaindive.com/news/air-cargo-industry-spot-rates-peak-season-xeneta-june/720975/)).
Inferred (flag as such): normal≈1.0, the within-regime spread, the κ→regime map; lane-level monthly
precision needs licensed TAC/Xeneta. *(Generator's static overlapping rate bands are the placeholder;
this regime model is implemented with the 3a κ-sweep where regime is driven.)*

**Convexity is a falsifiable hypothesis, not a given.** Define the test up front — fit `L2%(κ)`
and report the curvature/second-difference sign with CIs across replications. **If the surface is
not convex, report it as found** (DoD clause). Assuming convexity is the question-begging move a
sharp reviewer probes.

**Report a BAND, not a curve.** At each cell:
1. **Adversarial arrival ordering → the floor.** Worst-case sequence (large flexible shipments
   grab scarce capacity first; urgent ones arrive last).
2. **Sandbagged flexibility → pessimistic curve.** Perturb the *inputs* to 2-FLEX's derivation, never
   the derived label (`flexibility_model.md` D-F4): **shrink `sla_offset_h` + raise `θ_flex`** (fewer
   HAWBs clear the ≥2-separated-options test), with optional ε classifier-error noise on the *live*
   reshuffle set (not on the frozen `cw_flex` denominator). The flexible fraction is our biggest
   assumption; if savings survive, the number is robust. Tie the *locked* transition to physical tender
   (§2), not a notional flag, so the reshufflable set is realistically bounded.
3. **Literature-prior sanity check (not a gated DoD).** A one-paragraph note that the simulated
   peak sits within the order-of-magnitude range reported for online stochastic combinatorial
   optimization / network revenue management (bid-price). Competitive-ratio bounds are loose, so
   this is a sanity sentence, not a bracket the peak must fall "inside."

---

## 8. The cost–OTP frontier (dollarized α-lever)

Cost and OTP are a **frontier, not two scalars** (`PRD.md §5.6`) — two independent numbers are
gameable. The lever is a single **dollarized** tradeoff (user D-1/D-5, option a):

> `min  α·cost$ + (1−α)·lateness$`,  `α ∈ {0.1, 0.2, …, 0.9}`

where `lateness$` is a real dollar cost of being late (per-hour-late or SLA-breach charge), so the
objective is all-dollars and `α` weights operating-cost vs. lateness-cost on comparable scales —
calibratable by intuition. This is the existing M5 knob reparametrized (`W = (1−α)/α`), so **no
optimizer change** is needed.

**Tracing and the dominance claim.** Each point is a **whole-simulation** outcome: for an arm at a
given `α`, run the full horizon, then report **total operating cost$** over the book and
**aggregate OTP** (on-time fraction over the shipment population; each shipment realized **once**,
one draw per leg). Sweeping `α` traces the arm's portfolio cost–OTP curve. The claim: **`M₁`'s curve
dominates `M₀`'s** (lower cost at matched OTP *and* higher OTP at matched cost) — read L2 savings off
the gap. `H₀` is a **single point** (a human does not tune a penalty; its service buffer is the crude
analog); show `M₁` dominates it (the L1+L2 customer claim).

**Fixed tier ratios across the α-sweep.** The per-shipment penalty carries differential tier
weights, but `α` is the *global* cost-vs-lateness scaler. Hold the per-tier penalty **ratio fixed and
pre-registered** across the α-sweep (α moves the aggregate; the tier OTP spread is set by the
graph-gen filter, not α), and check `M₁` dominates `M₀` **per tier**, not only in aggregate — else a
tier-mix shift could masquerade as dominance.

**Frontier points carry CIs on both axes.** Each frontier point is a population outcome (cost\$ and
on-time fraction) whose *level* has sampling variance set by book size × `R`; a frontier *level* is
not a delta, so the §6 "CI on every delta" rule does not cover it. Report a CI on **both** cost and
OTP at each point, and claim dominance only where the CI boxes separate.

In buyer terms this curve is just a **knob**: "squeeze cost" vs "protect service" — a forwarder
selling a 95% SLA picks the point that holds 95%+ at max savings. That is the only thing the
frontier needs to communicate.

- **Scope:** trace the full `α`-curve at the **peak `(κ, λ)` cell only** (with `R` bumped there for
  a tighter CI); a single representative `α` elsewhere for the band.
- **Note:** at portfolio scale (hundreds of shipments/month) the curve is smooth — the
  integer-program non-convexity that can hide a Pareto point in a *single* shipment's choice set
  averages out across the book, so a weighted-sum `α` sweep is sufficient. (Only if a deck ever
  needs the cheapest cost to hit an exact OTP target would an ε-constraint pass add anything.)

**OTP evaluation — one realized timeline per shipment, legs not independent (build constraint → 2b).**
Each shipment is realized **once**: one draw per leg, walked on a running clock
(`air_transit_time.md §4`), so leg correlation and **connection-miss cascades** survive (a late leg
blows a tight connection). **2b emits per-leg realized actuals**; the **orchestrator** does the
connection-made check + policy recourse (`H₀/M₀` roll, `M₁` replan-from-current) — *not* 2b. `OTP =`
on-time shipments / population over the period; cost is realized operating cost over the same book.
CIs on every delta come from `R` seeded horizon replications (CRN-paired across arms). There is **no
per-route Monte-Carlo** and no Bernoulli connection coin — the single realized timeline carries the
cascade.

---

## 9. Persist the L3 / L4 corpus — at near-zero cost

The backtest emits exactly the moat exhaust the upper layers need: timestamped **(decision,
known-forward-state `I_t`, realized-outcome)** triples. For the proof, persist them as **flat
JSONL** with the right fields (decision / `I_t` snapshot / realized outcome), tagged
`ingestion_source='backtest'`. **Production-table-schema conformance (`route_legs` est+act, etc.)
is deferred** — a north star, not a Stage-3 blocker (consistent with plan §2a's in-memory hedge).
Not building L3 — just not discarding its training data.

---

## 10. Decisions (all RESOLVED)

- **D-1 / D-5 — cost–OTP frontier:** dollarized α-lever `min α·cost$ + (1−α)·lateness$`, sweep
  α=0.1…0.9 (option a — lateness in dollars, intuitive to calibrate). No optimizer change. §8.
- **D-2 — cost metric:** report **percent AND per-flexible-kg** (both informative across the λ
  sweep); drop absolute-$. §6.
- **D-3 — estimate/actual + OTP:** plan-on-ETA; score on a **single realized actual per leg** (one
  end-to-end running-clock walk per shipment, **no per-route Monte-Carlo**); OTP = **population-over-
  time** on-time fraction (binary per shipment: `A ≤ Δ_k`); CIs via `R` horizon replications.
  OTP **controlled** at graph-gen + penalty, **not** chance constraints. §6, §8, `air_transit_time.md`.
- **D-4 — baseline / decomposition:** four arms `H₀ / M₀ / M₁ / π_hind`; report L1 = H₀−M₀,
  **L2 = M₀−M₁ (headline)**, Total = H₀−M₁. §3–§4.
- **D-6 — OTP promise frozen at booking**, immutable, pytest invariant. §6.

---

## 11. Definition of Done (Stage 3 gate)

- [ ] **`H₀`** human heuristic implemented per `human_planning_heuristic.md` (fair, non-strawman).
- [ ] **`M₀`** MILP-no-replan arm (same commitment timing as `H₀`).
- [ ] **`M₁`** MILP rolling replan wrapping the air MILP in the orchestrator (2c).
- [ ] **L1 / L2 / Total decomposition** reported, L2 the headline (§3–§4).
- [ ] **Demand lookahead tripwire** pytest green — no future arrival leaks (§5).
- [ ] **Schedule lookahead tripwire** pytest green — inject future-only flight/cancellation, plan
      bit-identical; static-schedule assumption stated (§5).
- [ ] **No-capacity-double-spend** as the **per-arc/per-step conservation identity** (§6), asserted on
      a **binding-capacity + mid-horizon-tender** instance (the never-yet-exercised path).
- [ ] **`C^fallback` sized** = 2× worst feasible real route per HAWB (not the $1M placeholder);
      **fallback counts reported per arm**; headline split `L2_reshuffle` vs `L2_fallback-avoidance` (§6).
- [ ] **π_hind constraint set** = physical-feasible only, full demand + realized actual ETAs, no
      tender lock; `C_hind ≤ M₁` asserted **per draw on a binding-capacity instance** (§3, §6).
- [ ] **Spot:contract = κ-tied regime mixture** (sourced two-sided anchors), not a fixed gap; L2
      reported in % as headline (§7).
- [ ] **Frozen-promise** invariant asserted (§6).
- [ ] **Frozen-`z_tier`** per-shipment invariant asserted (§6).
- [ ] **Control inputs frozen during the proof** — no live OTP-to-date→penalty rebalancing (§6).
- [ ] **Re-screening policy stated:** predicate-9 re-screen per replan step for `M₁` (once-at-commit
      for `M₀`) is declared **part of** the L2 definition, not silently bundled (§4); `z_tier` frozen.
- [ ] **Regret invariants** asserted; `C_hind` from S2; `Reg(M₁)` flagged partly-irreducible (§6).
- [ ] **2-D `(κ, λ)` surface**, pre-registered grid + named peak; **convexity tested, reported as
      found** (§7).
- [ ] **Band**: adversarial-arrival floor + sandbagged-flexibility; literature note as sanity
      check (§7).
- [ ] **OTP** = population-over-period on-time fraction; one realized timeline per shipment (2b emits
      per-leg actuals; orchestrator does connection-check + recourse); CIs via `R` replications (§8).
- [ ] **Cost–OTP frontier** at the peak cell via α-sweep, **fixed pre-registered tier ratios**; `M₁`
      dominates `M₀` at matched-OTP AND matched-cost, **per tier and aggregate**; `M₁` curve dominates
      the `H₀` point (§8).
- [ ] Savings reported as **percent and per-flexible-kg**, with paired-CRN CIs (§6).
- [ ] **L3/L4 triples** persisted as flat JSONL (§9).
- [ ] A written **method + caveats note** resolving `product_thesis.md`'s `[TODO: quantify]` for
      air — L2 as a band, decomposed from L1, bracketed by literature, stress-tested against an
      adversarial arrival order and sandbagged flexibility.
