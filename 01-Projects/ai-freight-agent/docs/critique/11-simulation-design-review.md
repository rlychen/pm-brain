# Critique 11 — Simulation Design Review (4-agent, pre-2c-build)

**Session 32, 2026-06-09.** Four adversarial agents reviewed the arrival-only replan-savings
simulation **design** (not code — 2c/arms aren't built yet) across four lenses: (1) simulation
soundness / confounds, (2) falsifiability / clear test, (3) OR mechanism rigor, (4) operational
realism. Governing docs reviewed: `arrival_only_replan_methodology.md`, `backtest_methodology.md`
v0.5, `flexibility_model.md` v0.3, `air_transit_time.md` v0.3, `scenario_io_and_replay.md`,
`air_freight_routing.tex`, the synthetic generator, `docs/forwarder-operations-analysis/`.

**Headline:** the design is unusually well-armored against the *classic* backtest confounds
(lookahead, forecaster-asymmetry, double-spend-as-identity, denominator inflation, OTP re-promising
— all four agents said so). The risks that remain are structural to the **arrival-only +
deterministic-transit + tier-coupled-arrival** choices. Findings below are ordered by how many
independent agents hit them (convergence = signal) and by severity.

---

## A. Convergent load-bearing findings (≥2 agents)

### C1 — The tier-coupled arrival (EXPRESS-late / DEFERRED-early) is *engineered to produce the result*, asserted not sourced, and possibly reversed in reality. **[BLOCKING]** (Soundness #3, Falsifiability B1, Realism B1)
The L2 mechanism (§10: early-DEFERRED grabs the cheap slot → late-EXPRESS needs it → M₁ bumps) is
hard-coded into the demand generator via D-A2/D-A7. So a positive L2 partly means "the sim does what
we told it to." Three problems compound: (a) **no sourcing** — the realism agent searched the DITL
research and found no booking-lead-time-by-tier signal; real EXPRESS (pharma, auto, contracted lanes)
often books *with notice*, and the canonical last-minute booking is offloadable spot backfill (closer
to DEFERRED), so the direction may be partly reversed; (b) **λ is entangled** — compressing λ
preferentially hides EXPRESS, so "L2 rises with λ" is partly tautological; (c) it makes the headline
non-comparable to a neutral world.
**Fix (consensus):** make book-lead `B` a per-tier *distribution with overlap* (not a deterministic
offset), `[CAL]`-gated like slack-hours. Add an **independent-arrival control** (lateness tier-*independent*,
only `Δ_k` tier-coupled) and **make independent-arrival the headline cell**, with coupled-favorable
shown as an upper bracket. Flag D-A7 as a *load-bearing empirical claim*. If L2 collapses under
independent arrival, that is the real finding.

### C2 — No pre-registered null and no required negative-control cell → the test cannot demonstrably fail. **[BLOCKING]** (Falsifiability B1, Soundness #1)
"L2 is a conservative lower bound" is one-sided; nothing names the L2 that would falsify the thesis.
The κ knob can't even reach the genuinely-slack regime: `n_uld = max(1, round(2·capacity_scale))`
(`air_generator.py:165`) is integer-quantized with a floor of 1, so "L2≈0 when capacity is abundant"
is *asserted, never demonstrated*.
**Fix:** pre-register (a) a **null + rejection rule** (e.g. "thesis unsupported if peak-cell L2 CI
straddles 0, or M₀≈M₁ across the grid"); (b) a **mandatory abundant-capacity × early-arrival
negative-control cell** with a gated `|L2| < CI` pass condition (a regime where replanning *should not*
help — a tripwire on the metric itself); (c) make κ continuous in *binding-ness* (ratio of peak
concurrent demand to slots), not quantized ULD integers.

### C3 — "M₀ incremental-greedy" is under-specified; two non-equivalent readings, one of which collapses M₀→M₁, and the MILP has no primitive to express the intended one. **[BLOCKING]** (Mechanism B1, Soundness #2)
Docs define M₀ two incompatible ways: "don't disturb prior soft commitments" (Reading A: pin prior
soft `x_{k,a}`) vs "re-solve the visible book greedily each cycle" (Reading B — but a cost-min MILP over
the full visible set *will* re-route prior soft HAWBs = it replans = collapses to M₁). The canonical
mechanism only works under Reading A, and the model has **no constraint to pin soft un-tendered arcs**
(C.12 only locks *tendered* arcs). Without a precise coded definition, L2 is whatever the implementer
built, and the "same solver, pure recourse" claim is asserted not tested.
**Fix:** lock **Reading A**; add a **soft-pin constraint primitive** `x_{k,a}=1 ∀(k,a)∈S_t` for the
M₀ arm, pinning the *departure/MAWB-arc choice* (ground re-derives); specify behavior when a pinned
departure becomes infeasible. **Add a pinned-replan control arm `M₁'`** (M₁'s full-scope solve but with
priors pinned = M₀'s feasible set): assert `C(M₁') == C(M₀)` per draw — any gap is pure
formulation/tie-break leakage that must be netted out of L2. This is the single most important missing
arm.

### C4 — L2 can be dominated by fallback-avoidance and avoided-tardiness-penalty, not reshuffle — i.e. the *wrong* thesis. **[BLOCKING]** (Falsifiability B3, Mechanism M6, Realism m3)
The reshuffle/fallback split exists but is *reporting*, not a *gate*; nothing forces the reshuffle
component to carry the headline. Worse, the realized cost may include the C.10 **quadratic tardiness
penalty `W_k·τ²`**, which is an *objective-steering term, not a cash outflow* — if it leaks into
`realized_cost`, L2 is dominated by avoided-penalty, not avoided-spend. Fallback is still `$1M` in the
instance (`tpeb_air_instance.py`).
**Fix:** (a) confirm the scorer's `realized_cost` **excludes** the C.10 penalty (per §0, `C(π)` =
freight+consolidation+spot/recovery only); (b) report **three** components — `L2_reshuffle` (freight-$
on HAWBs on real routes in both arms), `L2_fallback_avoidance` (`C^fallback·Δcount`), and the
tardiness-penalty delta *separately*; (c) **gate `L2_reshuffle > 0` (separated CI) as the headline**,
pre-register a reshuffle-share floor (e.g. ≥50%); (d) make the `C^fallback = 2× worst-feasible` override
actually fire (retire the $1M).

### C5 — The DEFERRED-bump is the value mechanism, but (a) the ledger can't enforce the cross-arc atomic move, and (b) the bump is treated as costless. **[BLOCKING mechanism + MATERIAL realism]** (Mechanism B2, Realism M2)
The bump is a simultaneous {release slot on arc(d*), claim slot on arc(d*+1)} across **two** ledger
rows; the conservation `CHECK` is **per-row**, so a phantom-free state (released but not yet
re-claimed) is undetectable — the exact "double-spend = phantom saving" failure, invisible because each
row individually balances. Separately, a real bump carries re-handling, an extra CFS-dwell day,
re-screen risk, and a customer notification — none modeled, so every reshuffle's *net* value is
inflated.
**Fix:** (a) make conservation a **global per-step identity across all arcs** plus a **per-shipment
move-journal** (a HAWB's slot is conserved as it moves, not just per-arc); add a **2-arc reshuffle
fixture** (current binding-capacity fixture only tests single-slot contention). (b) Attach a per-slip
cost to a bumped HAWB (extra dwell-day × storage + fixed notification/admin, reusing the gateway
`cfs_dwell`/handling fields) so M₁ reshuffles only when it pays.

### C6 — Time-scalar inconsistency: graph-build forward-propagation, the MILP's `arr_dest` scalar, and the scorer's running-clock walk must share ONE scalar table — else OTP is contaminated and the regret floor breaks (`Reg<0`). **[BLOCKING]** (Mechanism B3, Mechanism M5)
Time feasibility is enforced at graph-build on the scheduled block; the MILP optimizes against
precomputed `arr_dest(k,a)`; the scorer computes `A` by a running-clock walk. For a *solo* HAWB in the
deterministic regime these coincide — but **not once MAWB consolidation couples HAWBs** (a HAWB's
realized arrival depends on its MAWB's routing; `A_k^min` is the *solo* fastest). So a HAWB can be
admitted + penalty-free in the plan yet *miss* in the score from consolidation alone — contaminating
the "M₀-vs-M₁ pure-recourse" OTP claim with zero transit randomness. And if the walk ever beats
`arr_dest` (e.g. graph-build pads ground to a quantile but the walk uses the mean), M₁ can beat
`π_hind` → `Reg<0`, breaking the floor.
**Fix:** a **single source of truth** for every time scalar (block, ground-mean, dwell, MCT) read
identically by graph-build, `arr_dest`, and the scorer walk. Assert **walk ≡ scalar for committed
routes** (pytest) and **`C_hind ≤ C(M₁)` per draw** tied to the shared table. Where consolidation drift
is unavoidable, concede OTP is not pure-recourse and route the drift to a declared bucket.

### C7 — The H₀ human baseline is stated as batch-at-cutoff but implemented/penalized as on-arrival → a strawman that inflates L1/L2; compounds with C1. **[MATERIAL]** (Realism M3, Falsifiability M4)
`backtest §0/§3` define H₀ as batch-at-cutoff; `§4` admits the code commits on-arrival (the
"pessimistic / L2-inflating edge"). A competent human *stages flexible cargo and commits at the build
cutoff* — that's the normal job (DITL P3), and by cutoff much of the late-EXPRESS demand has already
arrived, capturing much of what M₁ captures. With C1's favorable coupling, on-arrival H₀ is *doubly*
favorable.
**Fix:** make **batch-at-cutoff H₀ the headline baseline**; report on-arrival H₀ only as the upper
bracket; reconcile the §0/§3-vs-§4 contradiction before running.

---

## B. Single-agent material findings

- **M-B1 — "Conservative lower bound" is internally contradicted.** It's a lower bound *w.r.t. transit
  reliability* but the design's own admissions (on-arrival H₀, favorable coupling, κ-tightness,
  spot:contract gap) push the *other* way. **Scope the claim precisely:** lower bound w.r.t. transit
  disruption only; *not* w.r.t. the human baseline timing or arrival asymmetry. (Falsifiability M4)
- **M-B2 — `cap_a` / `A_c` (BSA pacing inputs) must be frozen across all arms.** They are exogenous
  per-solve inputs; a dynamic pacing controller "that sees the whole period" is a schedule-lookahead
  channel and, if per-arm, a confound. **Extend the "control inputs frozen during the proof" rule to
  `cap_a`/`A_c`; assert bit-identical across arms.** (Mechanism M7)
- **M-B3 — `L2/cw_flex` denominator shifts meaning across the λ grid** (λ changes which HAWBs are
  `flex_k`), so the surface isn't cross-cell comparable. **Lead with `L2%`; demote `L2/cw_flex` to the
  peak-cell deep-dive**, or freeze `cw_flex` at a single reference cell. (Soundness #5, Falsifiability M5)
- **M-B4 — Cutoff/tender timing granularity + ordering.** Snap every `cutoff(d)` to a sim-step grid
  point (no cutoff strictly inside a step); deterministic within-step tender order `(tender_at, tier,
  shipment_id)`; a HAWB with `cutoff ≤ t` is tendered *before* `plan()` runs. Add a simultaneous-cutoff
  contention fixture. (Mechanism M4)
- **M-B5 — Single BSA arc per lane ⇒ binary "cheap-or-3×-spot" worst case.** The methodology's own
  "roll-to-next-flight-on-contract" (late-but-cheap) option isn't in the instance. **Emit ≥2
  contracted/cheap options per lane** so the no-replan failure isn't always "spot 3×." (Realism M5)
- **M-B6 — π_hind may be over-granted.** Add a **`π_hind_locked`** (demand-clairvoyant but
  tender-locked) so regret splits into M₁'s recoverable online-suboptimality vs the irreducible
  commitment gap. (Falsifiability B2)
- **M-B7 — Signal-to-noise unvalidated** on a 15–30 HAWB instance where one bump ≈ the whole signal.
  **Run a pilot at the peak cell; size R for CI half-width < L2/4; gate "peak-cell L2 CI excludes 0."**
  May argue for moving the headline to the §11 forwarder-scale instance. (Falsifiability M7)
- **M-B8 — Pre-register the reporting α** (the cost–OTP operating point, e.g. M₁'s OTP hits the named
  SLA) so the headline scalar isn't post-hoc cherry-picked; the full α-curve stays as dominance
  evidence. (Falsifiability M6)
- **M-B9 — Fixed-N removes arrival-*count* uncertainty (deflates L2) and `known_at` anchored to cutoffs
  synchronizes arrivals into waves (inflates L2).** Report the `known_at` distribution; state fixed-N as
  an explicit (L2-deflating) simplification alongside the already-bracketed on-arrival-H₀ (L2-inflating)
  one — the honesty bracket is currently one-sided. (Soundness #4)

## C. Minor / scope caveats

- Solver-seed sensitivity at marginal cells; a lexicographic *min-reshuffle* secondary objective so
  M₀=M₁ ties are meaningful not tie-break artifacts. (Mechanism #9)
- ANC tech-stop / through-offer: state the lock is the **origin cutoff** of the chosen MAWB-arc.
  (Mechanism #8)
- Per-tier fallback-*composition* check (not just per-tier OTP) to close a residual gaming channel.
  (Falsifiability #8)
- Headline scenarios should assert `disruption_event_count == 0` so the recourse path provably never
  fires in the headline number. (Falsifiability #9)
- `ready`/`prep_time` and customs-dwell are fixed constants — ready-time variance and CBP/PGA holds
  (real replan triggers) are zeroed; consistent with the deterministic-headline scope, state as caveats.
  (Realism m1/m2)
- **Scope caveat (state sharply):** deterministic transit means the headline omits irregular-ops /
  disruption-recovery — per the DITL research the *larger* real replan driver. The L2 cell is the
  *arrival-timing component* under perfectly reliable transit; run the §6 disruption-recovery
  sensitivity arm before claiming the air replan thesis is fully proven. (Realism M4)

## D. What's sound (4-agent consensus — do not churn)

Deterministic realization kills the manufactured-failure attack; the L1/L2/M₀-middle-arm decomposition
is clean and additive; no-lookahead via the `I_t` contract + demand & schedule tripwires; conservation
as a stored integer CHECK identity (form is right; C5/C6 are the gaps); frozen-promise / frozen-`z_tier`
+ per-tier dominance vs OTP gaming; `flex_k` derived + arm-invariant `cw_flex`; generate-all-first +
CRN-for-free + HiGHS determinism pins; spot:contract as a κ-tied sourced two-sided regime (avoids the
3× trap); tiers anchored to the standard express/standard/deferred product.

---

## E. Recommended disposition (for user triage)

**Fold into the methodology as new decisions/DoD gates BEFORE building 2c (cheap, load-bearing):**
C1 (independent-arrival control + headline), C2 (null + negative-control cell), C3 (lock Reading A +
soft-pin primitive + `M₁'` control arm), C4 (penalty out of cost + 3-way split + reshuffle-gated
headline), C6 (single time-scalar table), C7 (batch-at-cutoff H₀ headline). Plus M-B1 (scope the
lower-bound claim) and M-B2 (`cap_a`/`A_c` frozen) — both one-line rule extensions.

**Fold as fixtures/build-tasks WITH 2c:** C5 (global conservation + 2-arc fixture + bump cost), M-B4
(cutoff grid + tender order), M-B5 (≥2 cheap options/lane).

**Defer (report-time / scale-up):** M-B3 (L2% headline), M-B6 (π_hind_locked), M-B7 (power pilot —
do at first peak-cell run), M-B8 (α pre-reg), M-B9 (fixed-N caveat), and the §C scope caveats.
