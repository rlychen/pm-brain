# Critique 20 — Backtest Methodology Red-Team (Session 45)

**Standing review agent: Backtest Methodology Red-Team.** S45 run; last ran at S36 (critique 16).
The question is **not** "is the code correct" (calibration + interface-seam agents cover that) — it
is **"is the experiment honest?"** Attacked as a hostile reviewer trying to get the paper rejected.
The headline: `L1 = C(H₀)−C(M₁')`, **headline `L2 = C(M₁')−C(M₁)`** (intra-engine), swept over
(κ,α,λ), with the loose-corner `|L2|<CI` gate as the falsifiability null.

**What changed since S36:** all 5 arms + scorer are now BUILT (`src/replay.py`). At S36 every attack
ended "the not-yet-written code must defeat this." This round I ran the real code. **Findings below
are measured, not argued** — `n=12` HAWBs, tier-coupled arrivals, seeds 0–5, κ∈{0.5,1,2,4,8},
default `mip_rel_gap=0.005`, `PYTHONHASHSEED=0`.

---

## The measurement that drives this report

The full chain `C(H₀) ≥ C(M₀) ≥ C(M₁') ≥ C(M₁) ≥ C(π_hind)`, scored via `score_run`, seeds 0–5,
tier-coupled, κ∈{1,2}:

| seed κ | H₀ | M₀ | M₁' | M₁ | π_hind | L1=H₀−M₁' | **L2=M₁'−M₁** | chain |
|---|---|---|---|---|---|---|---|---|
| 0 1 | 35479 | 35479 | 35479 | 35160 | 35160 | 0.0 | +319 | ok |
| 1 1 | 73452 | 41587 | 41587 | 40532 | 40336 | 31865 | +1056 | ok |
| 2 1 | 35091 | 35091 | 35091 | 35091 | 35091 | 0.0 | **0** | ok |
| 2 2 | 35091 | 35349 | 35349 | 35091 | 35091 | **−258** | +258 | ok |
| 3 1 | 72292 | **37222** | **37289** | 36873 | 36873 | 35003 | +415 | **M₀<M₁' by $67** |
| 4 1 | 93425 | 41313 | 41313 | 40705 | 40705 | 52113 | +608 | ok |
| 5 1 | 96831 | 37390 | 37390 | **37528** | 37390 | 59441 | **−137** | **M₁>M₁' and M₁>π_hind** |

Two construction-"guaranteed" inequalities are **violated in the built code**: seed-3 has
`C(M₀) < C(M₁')` (greedy beat the single-pass optimum) and seed-5 has `C(M₁) > C(M₁')` **and**
`C(M₁) > C(π_hind)` (the headline arm did *worse* than its own baseline AND beat the clairvoyant
floor from the wrong side — a negative L2). Both violations are ~$67–$137, sitting **at the
±0.5%-of-objective gap budget** ($176–$187). See R1-new (gap noise floor).

And the κ-sweep — the **primary axis** the whole surface is built on:

| κ | L2 by seed (0..5) | mean | #zero | #neg |
|---|---|---|---|---|
| 0.5 | 319 1056 0 415 608 −137 | 377 | 1 | 1 |
| 1.0 | 319 1056 0 415 608 −137 | 377 | 1 | 1 |
| 2.0 | 319 1056 258 415 608 −137 | 420 | 0 | 1 |
| 8.0 | 319 1056 54 415 608 −137 | 386 | 0 | 1 |

**L2 is essentially κ-invariant.** Contracted ULD positions go 16→4→1 as κ goes 0.5→2→8 (κ is
genuinely wired), yet L2 barely moves, because the spot cap (152,202 kg over 48 arcs, **unchanged
across κ**) is the abundant escape. The loose corner (κ=8) does **not** collapse L2 to ~0.

---

## Verdict summary (ranked by threat to publishability)

| # | Attack | Verdict | What settles it |
|---|---|---|---|
| **A1 (new)** | **κ does not drive L2** — primary sweep axis is inert at proof scale; spot is the abundant escape (R6 redux) | **LANDS — hardest** | A κ-cell where contracted capacity genuinely binds the headline routes (cap the spot escape or shrink it with κ); show ∂L2/∂κ > 0 |
| **A2 (new)** | **Gap-difference noise floor** — every cost is a 0.5%-gap-stopped objective; L2 ($137–$1056) overlaps the ±0.5% budget ($176–$187); produces **negative L2** and **chain violations** on real seeds | **LANDS** | Solve the headline cell to a *tight* gap (≤1e-4) or net the residual gap as a reported L2 noise floor; chain must hold OPTIMAL-vs-OPTIMAL |
| R1 (S36) | L2=0-fraction permutation artifact, no kill threshold | **STILL LANDS** | Zero-fraction modest here, but mean is carried by 1–2 high seeds (seed-1 $1056); pre-register zero-ceiling + distinct-reshuffle floor — **still ungated** |
| R2 (S36) | Loose-corner null near-vacuous / no teeth | **NEW FORM — LANDS** | Worse than S36: the loose corner doesn't even produce L2≈0 (κ=8 mean $386). The null gate would **fail** or pass only via a huge CI. κ-invariance breaks the gate's premise |
| **A3 (new)** | **Daily-cadence drift** — H₀ commits on a 24h grid (0,24,48,72,96), never at a cutoff (40,41,64,66,88,113); D-A14 locks **batch-at-cutoff** | **LANDS (spec/impl drift)** | Reconcile: either implement batch-at-cutoff or amend D-A14; the grid commits H₀ with *less* info ⇒ inflates L1 |
| **A4 (new)** | **L1≤0 timing asymmetry** — H₀ late-commit beats M₁' early-commit (seed-2 L1=−$258); §4 still writes the chain as guaranteed | **PARTIALLY — DEFENDED-by-disclosure** | Code docstring + memory disclose it; §4 chain text still reads "guaranteed". Honest *if* reported as found |
| R3 (S36) | Conservation unproven on binding corner | **PARTIALLY DEFEATED-by-build** | Ledger reconcile + tests exist and pass; but `tendered=0` audit-only, the atomic d*→d*+1 move-journal across two arcs is still the next slice, untested |
| A5 (new) | **Disruption-inertness** of the headline | **DEFENDED-by-build** | Verified: `disruptions=[]` ≡ `None` byte-identical; `_RECOURSE_W` only stamped on `to_replan` ⇒ empty with no disruption; refresh returns `[]` when `affected` empty |
| R4 (S36) | Supply frozen across arms; re-screen capacity leak | **DEFEATED-by-build** | Capacity vector drawn once in generation, read-only; ledger only subtracts. Determinism tests green |
| R5/R6 (S36) | L2=reshuffle asserted not decomposed; fallback-avoidance dominates | **STILL LANDS** | `metrics.l2_reshuffle`/`l2_fallback_avoidance` are written **NULL** (scorer line 980); the 3-way split is deferred to Stage 3 — headline still on raw L2 |
| R7 (S36) | π_hind floor sound iff single-time-scalar survives region→region | **PARTIALLY** | Floor violated on seed-5 ($137) — but via gap (A2), not time-scalar. Re-verify walk≡scalar on multi-O/D still owed |
| R8 (S36) | M₀ hobbled / M₁' not pinned | **NEW FORM** | M₀ is fair by design, but seed-3 `M₀<M₁'` shows the *pinning* is gap-leaky, not hobbled — folds into A2 |
| R9 (S36) | Leakage tripwires written for fixed-lane world | **STILL OPEN** | Airport-choice lookahead tripwire still not in the suite |

**The three that threaten publishability of the L2 number: A1 (κ inert), A2 (gap noise floor),
R1/R2 (the null has no teeth and the headline rides 1–2 seeds).** A1 is the single biggest threat.

---

## A1 (NEW) — κ, the primary sweep axis, does not drive L2. **LANDS — hardest**

The entire deliverable is `L2 savings(κ,α,λ)` as a surface, with the loose-κ corner as the
falsifiability null. **Measured: L2 is κ-invariant.** Tightening contracted capacity 16→1 ULD
positions (κ 0.5→8) leaves the L2 vector across seeds essentially unchanged (mean $377→$386). The
reason is structural and matches R6 in a sharper form: the **spot cap is abundant and κ-independent**
(152,202 kg over 48 arcs, unchanged across κ; the whole 12-HAWB book is 7,453 kg). When contracted
ULDs shrink, cargo spills onto spot, which is always there. So the cost the optimizer minimizes — and
the reshuffle value M₁ extracts — lives almost entirely in **spot/consolidation timing**, not in
contracted-slot contention. The canonical thesis mechanism ("free the cheap contracted slot for the
urgent HAWB by bumping DEFERRED") barely fires because there is no contracted scarcity that bites.

A reviewer asks: "your headline is a savings-vs-tightness surface; show me ∂L2/∂κ > 0." On this
instance it is ≈ 0. A flat surface is not a `savings(congestion)` band — it is a constant with a
κ label. **This is the loose-corner null's premise (`|L2|<CI` because nothing binds) failing in the
other direction: nothing binds *anywhere*, so L2 is a constant the κ axis cannot explain.**

**What settles it.** Either (a) make the spot escape scale *with* κ (tighten/cap spot as contracted
tightens, so network tightness actually bites the headline routes), or (b) move the headline to a
forwarder-scale instance where per-airport binding-rate is structural (D-A18 already requires
reporting per-airport binding-rate — report it, and show it is non-trivial at the headline cell), and
(c) **gate the headline on a demonstrated ∂L2/∂κ > 0** with a CI — the convexity-as-falsifiable-
hypothesis DoD (§7) is currently untestable because the surface is flat.

## A2 (NEW) — Gap-difference noise floor. **LANDS**

Every cost in the chain is a `mip_rel_gap=0.005`-stopped objective (`MilpParams.mip_rel_gap=0.005`,
air_milp.py:199; `score_run`'s final cost solve uses default params). **L2 is a *difference* of two
such objectives.** Measured L2 magnitudes are $0–$1056; 0.5% of the objective is **$176–$187**. So the
signal sits *inside* the per-objective gap budget on most seeds.

Direct consequences observed on real seeds (not constructed):
- **seed-5: L2 = −$137** (M₁ scored *higher* than M₁'), and `C(M₁)=37528 > C(π_hind)=37390` — the
  regret floor violated. M₁'s solve carried `mip_gap` up to 4.4e-3 across all 12 cycles; M₁' and
  π_hind ran near-0. A negative L2 from the headline arm doing "worse" than its own no-reshuffle
  baseline is **exactly** the gap-of-difference signature, not a real anti-reshuffle effect.
- **seed-3: C(M₀)=37222 < C(M₁')=37289** by $67 — greedy "beat" the single-pass optimum. M₁' carried
  gap 1.9e-3. The construction-guaranteed `C(M₀) ≥ C(M₁')` is violated by gap leakage, not logic.

The methodology already flags (BLK-1) that time-limited incumbents can transiently violate the chain.
But these are **gap-stopped (not time-limited)** solves on a tiny instance — the chain breaks under
the *intended* operating point. The S38 framing "headline L2 is intra-engine, structurally
artifact-free" is true about *code paths* but says nothing about *numerical* artifact: the same solver
stopped at 0.5% twice still differences two ±0.5% objectives.

**What settles it.** At proof scale the instance is small enough to solve **tight** (`mip_rel_gap`
≤1e-4, or to proven optimality) — do it for the headline cell so the chain holds OPTIMAL-vs-OPTIMAL
per draw, and report any residual as an explicit L2 noise floor. A headline L2 of $300–$400 with a
$180 per-objective gap budget is not yet a number.

## A3 (NEW) — H₀ daily-cadence drift from locked D-A14. **LANDS (spec/impl drift)**

D-A14 (LOCKED, methodology §12): "**batch-at-cutoff `H₀` is the headline baseline**." The S44
implementation runs H₀ on `_daily_times` — a 24h grid anchored at the first arrival. Measured on
seed-3: grid = {0, 24, 48, 72, 96}; cutoffs = {40, 41, 64, 66, 88, 113}. **No grid point is a
cutoff.** A shipment with cutoff 64 commits at the daily run at 48 — 16h of arrival information
earlier than batch-at-cutoff would. The code docstring says "commits a shipment at the last daily run
before its cutoff," which is honest about the *implementation* but **contradicts the methodology's
locked decision**. Direction: daily-grid H₀ commits with *less* info than D-A14 specifies ⇒ a more
pessimistic (L1-inflating) human baseline than the locked spec. Because D-A14 itself was the
"competent human stages to cutoff" fix to an earlier L2-inflating on-arrival H₀, this drift partially
re-opens that fix. It does not touch the headline L2 (M₁'−M₁), but it moves L1 and the Total story.

**What settles it.** Reconcile spec and impl: either implement batch-at-cutoff (plan at the distinct
cutoff times, as the machine arms already do via `_event_times`) or formally amend D-A14 to "daily
cadence + short-fuse inserts" with the L1 direction documented. The anchor choice (first-arrival)
should be shown not to be L2/L1-favorable.

## A4 (NEW) — L1 ≤ 0 timing asymmetry. **PARTIALLY — DEFENDED-by-disclosure**

Measured seed-2 κ=2: **L1 = H₀ − M₁' = −$258** (H₀ cheaper than the "competent" single-pass arm),
the exact mirror of L2=+$258 the user banked. Mechanism: H₀'s daily/late commit can see more arrivals
than M₁''s commit-at-reveal, so on some draws H₀'s late-commit luck beats M₁''s disciplined early
pinning. The `run_replay` docstring discloses this ("the H0 rung is NOT guaranteed … L1 … may be ≤ 0
at tight cells — a finding, not a bug"). **But methodology §4 still writes the chain
`C(H₀) ≥ C(M₀) ≥ C(M₁') …` as "guaranteed by feasible-set nesting,"** which is false for the H₀ rung
(H₀ is not a feasible-set superset of M₁'; it commits on a different clock). This is honest *only if*
L1 is reported as measured (and can be negative), and the value claim relocated to the within-cycle
rung `C(M₀)−C(M₁')`. The asymmetry is not a "built-to-win in the other direction" artifact — it is a
real consequence of the cadence choice — but it interacts with A3: fixing the cadence drift will move
L1's sign behavior, so resolve A3 first.

## A5 (NEW) — Disruption-inertness of the headline. **DEFENDED-by-build**

The 2c-7 recourse machinery (`_disrupted_sim`, `_refresh_active_shipments`, `_RECOURSE_W`) is threaded
loop-wide. Verified inert on the no-disruption headline: (1) `disruptions=[]` ≡ `disruptions=None`
produces byte-identical cycles + routes; (2) `_refresh_active_shipments` returns `[]` the moment
`affected` (delays ∪ cancels) is empty, *before* any route projection; (3) `_RECOURSE_W` tardiness
weights are stamped only on `to_replan` shipments via `replace(..., tardiness_weight=...)`, so with no
disruption every HAWB keeps W=0 and the C.10 penalty term is structurally absent; (4) `sim_t = sim`
(same object) when no disruption is realized. The "recourse is a capability, not a value source" claim
holds in code: the headline path never enters the recourse branch. **Defended.** (Caveat: recovery-to-
fallback raises `NotImplementedError` — slice 3 — so a disruption that strands cargo crashes rather
than scoring; fine for the inert headline, a gap for the recourse sensitivity study.)

## R1–R9 updated verdicts (S36 attacks vs the built code)

- **R1 (L2=0 fraction, no kill threshold) — STILL LANDS.** Zero-fraction is modest at this scale
  (1/6 seeds at κ=1), but the *mean is carried by 1–2 seeds* (seed-1 = $1056 vs others $0–$608). The
  D-A23 zero-fraction diagnostic is computed nowhere in the scorer; no ceiling/floor gate exists.
  "Report L2 conditional on a contention event" still owed.
- **R2 (loose-corner null vacuous) — NEW FORM, LANDS HARDER.** At S36 the worry was the gate passes
  trivially. Measured: the loose corner (κ=8) does **not** give L2≈0 (mean $386) — so the gate either
  *fails* (L2 ≫ CI at the supposed null) or passes only behind a wide CI. The premise "nothing binds
  at the loose corner ⇒ L2→0" is false because spot is abundant *everywhere* (A1). The null is not
  just toothless, it is mis-specified.
- **R3 (conservation on binding corner) — PARTIALLY DEFEATED-by-build.** `ReplayState` ledger with
  declarative reconcile + over-commit guards exists; `test_ledger_conserves_and_never_over_commits`
  passes. But `tendered` is recorded as 0 (audit-only this slice), and the atomic two-arc d*→d*+1
  move-journal (slot conserved *as it moves*) is still the explicitly-deferred conservation-fixture
  slice. The binding-corner identity is asserted on the per-arc reconcile, not the cross-arc move.
- **R4 (supply frozen / re-screen leak) — DEFEATED-by-build.** Capacity drawn once in generation,
  read-only; ledger only subtracts from the frozen vector. Determinism tests green.
- **R5/R6 (L2=reshuffle, fallback-avoidance) — STILL LANDS.** `metrics.l2_reshuffle` and
  `l2_fallback_avoidance` are written **NULL** (replay.py:980); the D-A12 3-way split is deferred to
  the Stage-3 sweep. The headline is still on raw L2, so fallback-avoidance can ride it (folds into A1
  — when contracted binds, the failure mode is a fallback/spot premium, not a reshuffle).
- **R7 (π_hind floor / time-scalar) — PARTIALLY.** Floor violated on seed-5, but via the gap (A2),
  not a time-scalar drift. Re-verify walk≡scalar on a region→region multi-O/D instance (N7) still owed.
- **R8 (M₀ fairness / M₁' pin) — NEW FORM.** M₀ is fair by design (greedy newcomer placement). But
  seed-3's `M₀<M₁'` shows the *pinning equality* is gap-leaky — folds into A2, not a hobbling.
- **R9 (lookahead tripwires post-D-A24) — STILL OPEN.** No airport-choice lookahead tripwire in the
  suite; the prior no-lookahead certification remains lapsed for region→region.

---

## Minimum set before the (κ,α,λ) number is publishable (ranked)

1. **A1 — make κ drive L2, or move the headline.** Cap/shrink the spot escape with κ, OR go to a
   forwarder-scale instance with a structural per-airport binding-rate; gate on a demonstrated
   ∂L2/∂κ > 0 with a CI. Without this the surface is flat and the convexity DoD is untestable.
2. **A2 — kill the gap noise floor.** Solve the headline cell tight (≤1e-4 / proven optimal); the
   chain must hold OPTIMAL-vs-OPTIMAL per draw (seed-3 and seed-5 violations must vanish); report any
   residual as an explicit L2 noise floor. L2 ≈ $300–$400 against a $180 gap budget is not a number.
3. **R1/R2 — the null and the zero-fraction.** Pre-register the zero-fraction ceiling + distinct-
   reshuffle floor; re-derive a discriminating null now that the loose corner is not L2≈0 (a
   constructed positive control with a hand-computed L2*>0 the engine must hit).
4. **A3 — reconcile H₀ cadence with D-A14** (batch-at-cutoff vs daily grid); document the L1 direction.
5. **R5/R6 — build the L2_reshuffle/L2_fallback split** and gate the headline on `L2_reshuffle` with a
   separated CI and the ≥50% reshuffle-share floor; the metrics columns are currently NULL.
6. **R3 — the two-arc atomic move-journal**; R7 — walk≡scalar on multi-O/D; R9 — airport-choice
   lookahead tripwire.

## One-paragraph bottom line

The build defeats the S36 *architecture* attacks (supply-freeze R4, disruption-inertness A5, the
ledger half of R3) — those are real wins. But running the real code surfaces two findings that
threaten the number more than anything at S36: **κ, the primary sweep axis, does not move L2 at proof
scale** (spot is the abundant escape, so contracted tightness never bites the headline — A1), and
**L2 is a difference of two 0.5%-gap-stopped objectives whose signal ($137–$1056) overlaps the gap
budget ($176–$187)**, producing negative L2 and chain violations on real seeds 3 and 5 (A2). Together
these mean the loose-corner null is not just toothless (S36 R2) but *mis-specified* — the corner
doesn't produce L2≈0 — and the headline mean is carried by 1–2 lucky seeds (R1). None are fatal to the
thesis, but **as built, the κ surface is flat and the L2 signal is inside the solver's own noise
floor; the number is not yet publishable.** Fix A1 (make κ bind, or move to forwarder scale) and A2
(solve tight) before any L2 leaves the building.
