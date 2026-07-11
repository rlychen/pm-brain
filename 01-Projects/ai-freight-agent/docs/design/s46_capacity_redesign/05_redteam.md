# S46 Capacity Redesign — Adversarial Red-Team

**Reviewer role:** Adversarial Red-Team. One job: find the way this design **fails to demonstrate
the capacity value it claims** — the escape valve that lets capacity NOT bind despite demand>supply,
the mechanism that reproduces the S45 inertness (κ inert, `L2` 100% consolidation / 0% capacity,
contracted never used, fallback never touched, undetected for sessions because nobody checked whether
the arms differed in tier totals).

**Inputs:** `01_architecture.md`, `02_realism.md`, `03_metrics.md`, `04_tractability.md`,
`l2_decomposition_s45.md`, plus live-code probes of `air_milp.py` / `air_graph.py` (cited inline).
**Design review only — no code changed.** Default posture: *if it is not proven that capacity binds
differently across arms, assume it does not.*

**Severity:** **FATAL** = reproduces S45 inertness (the redesign's headline `L2_capacity` ≈ 0 or
`∂L2/∂τ` flat) · **SERIOUS** = a headline metric is wrong/uninterpretable but the apparatus survives
a design change · **WATCH** = latent risk / overconfidence to pre-register against.

---

## The core argument (read this first)

S45's lesson, stated structurally: **`L2_capacity ≠ 0` requires M₁ and M₁′ to route *different total
chargeable-kg per (lane, capacity-tier)*.** In S45 they routed byte-identical totals (spot kg Δ=0,
contracted=0, fallback=0); L2 was pure consolidation reshuffle, and the block curve — a function of
*total lane kg*, the quantity held equal — amplified it by exactly $0.

The redesign changes the *prices and finiteness* of the tiers. It does **not** add any mechanism that
forces the two arms to differ in per-tier totals. Whether they do is left to the replay dynamics to
produce — exactly the unexamined hope that failed in S45. Worse, the redesign installs a **new** flat
unbounded escape (fallback) in the same structural slot the old one (cheap unbounded spot) occupied.
Below I show why the most likely outcome is `L2_capacity ≈ 0` again, and the one test that settles it
before any build.

---

## FATAL findings

### F1 — The unlimited flat fallback is the S45 escape valve reborn one price-tier up
**Mechanism.** Live code (`air_graph.fallback_arc` / `_hawb_fallback_cost`, `air_milp.air_leg_cost_ub`):
fallback is **one arc per HAWB**, cost **fixed at build time** (longest-cost path × margin), with **no
aggregate capacity** — confirmed uncapped. So once a lane's contracted + spot blocks exhaust, every
overflow HAWB exits at a per-HAWB *flat* price, unboundedly. This is structurally identical to S45's
"~150k kg of flat unbounded spot": a flat, unlimited tier that absorbs all tightness at a fixed
marginal price.

Now the killer step. On a short lane, **total spill kg = total lane demand − total cheap capacity**.
That is a property of the *lane* (capacity vs demand), **not of the arm**. Both M₁ and M₁′ are
cost-minimizers facing the same finite cheap capacity; both fill cheap-first and spill the remainder.
If the same *total* kg spills in both arms, and fallback is flat-priced, then **fallback cost is
identical between arms** → its contribution to `L2_capacity` is **0**. M₁'s open-book foresight only
re-shuffles *which* HAWB sits in cheap capacity vs fallback — but with flat within-tier pricing, who-
sits-where is free. This is precisely the S45 mechanism (there: flat spot; here: flat fallback).

**The only way fallback contributes to L2** is if M₁ spills *less total kg* than M₁′ — i.e. foresight
reduces the network-wide overflow via cross-lane / cross-day rebalancing (F2). The block curve and the
$4.2 contract re-anchor do **nothing** for this, because both arms ride the identical deterministic
cost waterfall.

**Fix / pre-registered check.** (a) Make the fallback *price* a poor proxy for the binding story —
report `total_fallback_kg` **per arm** and gate on `Δkg ≠ 0`, not on fallback $ (which is the same flat
price either way). (b) Consider giving fallback a *rising* marginal price or a finite belly-of-the-
next-flight capacity so that *who* spills matters; a flat unbounded escape can never produce a clean
capacity signal. (c) Pre-register: **if `total_fallback_kg(M₁′) = total_fallback_kg(M₁)` at the tight
cell, the redesign has reproduced S45** — stop and redesign, do not ship the L2 number.

### F2 — Strict cost ordering ⇒ both arms run the identical greedy waterfall ⇒ no tier-allocation difference
**Mechanism.** The redesign installs a strict price ladder: hard-BSA marginal $0 (sunk) < soft-BSA
$4.2 < spot block₀ $5.5 … block₃ $9.5 < fallback $13.75. Against a *fixed* per-lane demand, a cost
minimizer fills this ladder bottom-up **deterministically**. M₁ and M₁′ both do this. The per-tier
*totals* are then a function of lane demand and lane capacity — identical for both arms — **unless the
arms place different total demand on the lane in the first place.** That cross-lane/cross-day demand
difference is the *entire* source of any capacity component of L2, and the design neither forces nor
estimates it. It is the same gap S45 fell into: the apparatus *can* produce an arm difference, but
nothing *makes* it.

The arm difference can only arise from M₁′'s **premature commitment** of scarce capacity (it cannot
revisit a tendered HAWB; M₁ can), causing later cargo to spill that foresight would have placed. For
that to bite you need: (i) cheap capacity genuinely contested *across cycles* (early and late HAWBs
fighting for the same finite block), and (ii) M₁′ actually committing the *wrong* HAWB. Neither is
guaranteed; both are plausibly *rare* at the proof scales (see S2/S3). If they don't fire, L2 reverts
to the consolidation artifact.

**Fix / pre-registered check.** Make cross-cycle contention a *designed* condition, not an accident:
size at least one short lane so its cheap capacity is exhausted by **early-cycle** cargo while
**known-better** late-cycle cargo is still arriving (the design must show the arrival stream + capacity
schedule actually create this race). Pre-register the kill-shot (below) as the gate.

---

## SERIOUS findings

### S1 — The soft-BSA pivot floor still dominates at mid-market fill: "contracted never used" survives for the soft half
**Mechanism.** Confirmed in code: soft BSA bills `r_a · max(CW, π·Ση)` (the pivot floor). Re-anchoring
`r_a` to $4.2 only beats $5.5 spot **when the ULD fills past the pivot**. Arithmetic: with π≈1200 kg
(realism M2 caps it there), soft-contracted's effective per-kg = `4.2·π/CW`, which is ≤ $5.5 only when
`CW ≥ 4.2π/5.5 ≈ 0.76π ≈ 917 kg` per ULD (≈61% of LD3). At realism S5's thin depth (~4 ULD/lane-week)
and HAWB mean 517 kg, you need ~2 HAWBs consolidated per position to clear the floor. If consolidation
is thinner than that on a given flight, **soft contracted is *more* expensive than spot even after the
re-anchor — the exact S45 root cause, unfixed, for the soft half.** Since realism B2 argues hard-BSA
share should *drop* to 0.3–0.4 (mid-market risk-aversion), the genuinely-binding tier (hard BSA, sunk
marginal $0) is the *smaller* slice, and the larger soft slice may sit idle again.

**Fix / check.** Before relying on the $4.2 anchor, compute the *effective* per-kg of a representative
soft ULD at *expected realized fill* and confirm it is < spot block₀. If not, contracted won't bind
single-arm and L2_capacity is dead before replay even runs. Either raise `hard_bsa_frac`, or lower the
pivot, or accept that only hard-BSA is the binding tier and size it accordingly.

### S2 — Endogenous routing washes out the designed per-lane tightness (the real LLN-style threat)
**Mechanism.** The design reassures that realization noise won't wash out the bucket bands (correct —
at ~20 HAWBs/lane, demand CV ≈ 11%, far below the 0.6-vs-1.2 band gap). **But that is the wrong threat
model.** The actual washout is *endogenous routing*: under D-A24 region→region routing the optimizer
**flows demand toward cheap (slack) lanes and away from short (expensive/fallback-prone) lanes**, which
*equalizes realized utilization* and makes the supply-side bucket assignment cosmetic. A "short" lane
stays short only if its cargo is **geographically captive** (no slack lane reachable from its origin
box). The design sets tightness on the *supply* side per-lane but lets *demand* route freely — so the
binding short lane only exists where cargo can't escape it. The architecture never establishes that any
HAWB cohort is captive to a short lane.

**Fix / check.** Either (a) deliberately make a subset of HAWBs captive (origin/dest boxes that only
reach a short lane's gateways), so designed tightness survives routing; or (b) measure realized
per-lane utilization *after* the solve and confirm the short lanes are still short — don't trust the
*supply* `τ_ℓ`, trust the *realized fill*. If routing equalizes everything, the bucket-mix story is dead.

### S3 — Fixing "contracted never used" by making it "always fully used by BOTH arms" creates no arm difference
**Mechanism.** Contracted is now the cheapest tier and finite. Both M₁ and M₁′ will fill it to
capacity (greedy waterfall, F2). So contracted *utilization* flips from "0 in both arms" (S45) to
"≈100% in both arms" — which is **still no difference between arms**. The S45 verdict required a
*contracted↔spot reshuffle margin that differs across arms*; a tier that both arms saturate identically
provides none. The contracted-vs-spot margin only differs across arms at the *marginal* lane where
contracted is exactly contested across cycles (F2) — a thin, possibly empty, set.

**Fix / check.** Report contracted utilization *per arm* and gate on `Δ ≠ 0`. Saturation-in-both is a
*failure* signature, not success. The interesting regime is partial, cross-cycle-contested contracted.

### S4 — φ_cap / ψ_cap cannot cleanly separate capacity savings from consolidation savings (the confound is structural)
**Mechanism.** Chargeable weight = `max(actual, vol·167)`, and consolidation (density-mixing onto a
MAWB) *changes* the chargeable weight that hits the capacity tiers. So a *consolidation* decision moves
$ in the *contracted/spot/fallback* buckets — which the metrics doc's attribution (a) labels
"capacity." φ_cap therefore mis-credits consolidation-driven savings to capacity. ψ_cap (b) tries to
fix this by counting kg "re-grouped onto a different MAWB at the **same tier**" as consolidation — but a
HAWB moving from spot-block₂ to spot-block₁ is the *same* "spot" tier yet a *different* capacity price;
is that capacity or consolidation? The block index *is* the capacity signal. The taxonomy is ambiguous
exactly where the two channels entangle, so the ψ_cap≥0.30 gate is not well-defined and is potentially
game-able by where you draw the tier boundary.

**Fix / check.** Define the capacity/consolidation split on an **invariant**: hold the MAWB grouping
*fixed* between arms and re-bill (isolates pure tier reallocation = capacity), then hold tier totals
fixed and vary grouping (isolates consolidation). The difference of differences is the cross-term. Only
a grouping-invariant decomposition can claim to separate them; the current $-bucket grouping cannot.

### S5 — The metrics are in tension: a binding-capacity regime is a *peak* regime, so the headline can't be both "everyday" and "capacity-bearing"
**Mechanism.** Capacity binds only at τ<1 (realism B1: that is *peak*, not the standing state). But the
OTP target band is 80–92% *normal* / 70–80% *peak*, and realistic fallback is 2–8% normal / 15–30%
tight. To get a non-trivial `L2_capacity` you must push τ down into the peak band — where, by the
calibration doc's own targets, OTP and fallback are at their *stressed* values. So you **cannot
simultaneously** report (i) everyday-normal economics, (ii) a binding capacity story, and (iii)
in-band normal OTP/fallback. τ moves all three together. The "headline replan value" is then either a
*normal-regime number where capacity barely binds* (L2 reverts toward consolidation) or a *peak number
mislabeled as everyday* (compounds realism B1). The design owes an explicit statement of which regime
the headline `L2%` is quoted at, and that capacity value is intrinsically a peak/crunch phenomenon.

**Fix / check.** Quote `L2_capacity` and `L2_consolidation` *separately by regime*, and state plainly
that the capacity component is a peak-regime value. Do not blend them into one "everyday replan %".

---

## WATCH findings

- **W1 — ψ_cap≥0.30 is an aspiration, not a demonstration.** The metrics doc treats clearing the gate
  as the expected outcome of the fix. S45 was *also* expected to show κ-sensitivity and didn't. The
  gate must be run as a *prove-or-refute* before build (kill-shot), not asserted as a property of the
  design.
- **W2 — Deferred-air (#9) is itself a soft escape valve.** A cheap, slow, capacious arc that the
  optimizer takes "when slack allows" is another way for tightness to leak out without binding the
  contracted/spot tiers. If it is too capacious or too cheap, it absorbs the overflow that should have
  pressured fallback, muting metric 3. Size it tight or fold it into the spot pool (realism M1 agrees).
- **W3 — `S = τ·D` with `D` = analytic expected demand means the *realized* network can be slacker than
  τ implies.** Realized demand fluctuates below its mean on some seeds; with supply pinned to the
  analytic mean, those seeds are *slacker* than the dial says → fewer binding lanes than intended.
  Report realized network τ (realized demand / realized capacity), not just the dial.
- **W4 — Both arms saturating hard-BSA `A_c` (sunk, marginal $0) is mandatory, not optional, for
  both.** Since the allowance is already paid, *every* arm fills it first identically → it contributes
  $0 to L2 by construction. Hard-BSA helps *realism* (sunk-cost dynamics) but is structurally incapable
  of carrying an arm-difference. Don't expect it to move L2.

---

## Kill-shot test — run FIRST, before any build (the check that was missing in S45)

**Question it answers:** *Do M₁ and M₁′ route different total chargeable-kg per capacity tier?* If no,
`L2_capacity = 0` and the redesign has reproduced S45 — regardless of the block curve, the re-anchor,
or hard BSA. This is the single scalar nobody computed in S45.

**Smallest experiment (hours, not a build):** Patch the *existing* `air_generator` / `air_milp` with
the **three minimal changes claimed to make capacity bind** — (1) shrink the spot cap hard (finite,
realism S2's ~1–2k kg base block), (2) re-anchor contracted below spot ($4.2 < $5.5), (3) keep
contracted finite — on **one short lane** of the C0/C1 grid at a tight cell (τ≈0.7). Run the existing
`run_replay` for **M₁′ and M₁** and emit, per arm:

1. **`total_kg` per capacity tier** {contracted (soft / hard), spot-block-i, fallback}.
2. **`total_fallback_kg`** and **contracted utilization**.
3. The deltas M₁′ − M₁ for every tier total.

**Decision rule (pre-registered):**
- **PASS** (capacity binds across arms) **iff** at least one of: `total_fallback_kg(M₁′) ≠
  total_fallback_kg(M₁)`, **or** the contracted/spot tier totals differ between arms — i.e. the arms
  route *different total kg per tier*. Then, and only then, build the full block-curve apparatus.
- **FAIL** (S45 reproduced) **iff** every per-tier total is equal between arms (Δ ≈ 0), even if
  fallback is now non-zero. Equal-totals + flat fallback = zero capacity component. **Stop; redesign.**

**Cheaper static pre-check (minutes, run even before the above):** at realism S5's thin depth, compute
the *effective* per-kg of a representative soft-BSA ULD at *expected fill* (`4.2·max(CW,π)/CW`) and
confirm it is `< $5.5`. If it is not, soft contracted will not bind in *either* arm and S1 has already
killed the soft half before any replay dynamics matter.

**Why this is the missing S45 check:** S45's failure was invisible because the only thing measured was
`L2` (the $ delta), never *whether the arms differed in per-tier kg*. They didn't — total spot kg was
byte-identical. The kill-shot measures that one quantity directly and refuses to let a non-zero `L2`
(which consolidation alone produces) be mistaken for a capacity result.
