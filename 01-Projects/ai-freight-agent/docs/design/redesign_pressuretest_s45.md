# Adversarial Pressure-Test — Air Pricing & Capacity Redesign (S45)

**Role:** adversarial OR-modeling reviewer. **Target:** `air_pricing_capacity_redesign_s45.md` (+ calibration,
+ composition research), the LOCKED increasing-block spot redesign, BEFORE it becomes a formal methodology
amendment. Read-only except this doc. Verdicts: BREAKS / NEEDS-SPEC-FIX / WEAKENS / SURVIVES.

Code facts grounding the verdicts were verified against `src/components/air_milp.py`,
`data/synthetic/air_generator.py`, `src/components/air_graph.py`:
- `_build_spot_cap` (air_milp.py:632–661): co-load arcs cap `Σ cw_k·x ≤ cap`, MAWB arcs cap `Σ_g CW_{a,g} ≤ cap`. Confirmed as the redesign claims.
- `CW_{a,g}` is per **(arc, group)** (air_milp.py:379–389) — NOT per HAWB. The block-sum must group these across arcs into a lane-day pool.
- `_validate_billing` (air_milp.py:1239–1279) is **per-(a,g) scope**; it recomputes flat/MFB closed forms and would NOT catch cross-arc / lane-pool violations.
- Supply draw is genuinely analytic: `total_N = round(n·E[SE_k]/κ)` with `E[SE_k]` closed-form, zero demand draws (air_generator.py:259–304). The `spot_regime` stream is κ-independent.
- MFB is genuinely non-convex (γ binaries + big-M, air_milp.py:695–766) — the BLK-1c structure the new curve must avoid.

---

## Ranked verdicts (worst first)

| # | Attack | Verdict | Severity |
|---|--------|---------|----------|
| 3 | Undiscounted-base premium inverts the weight discount | **BREAKS** | Critical — blocks amendment |
| 2 | Lane-day block-sum vs dual billing path + validators | **NEEDS-SPEC-FIX** | High |
| 6 | Loose-corner null not structurally guaranteed | **NEEDS-SPEC-FIX** | High |
| 5 | n=12 / 400-kg-block lumpiness makes L2 noisy/non-monotone | **WEAKENS** | High |
| 4 | Per-lane-day daily reset rewards cross-day splitting (Jensen) | **NEEDS-SPEC-FIX** | Medium |
| 1 | Supply-from-demand circularity via "thin to beat spill" | **SURVIVES (with one guard)** | Medium |
| 7 | Fallback vestigial at proof scale; L2_fallback unmeasurable | **WEAKENS** | Low–Medium |

---

## Attack 3 — Undiscounted-base premium DISTORTS, and INVERTS the weight discount — **BREAKS**

This is the one that must be fixed before the amendment.

The locked design (§5, OPEN-2 final form) splits spot into two ledgers to stay linear:
- base × weight-break, per booking, on the existing path;
- scarcity premium `Σ baseℓ · (multℓᵢ − 1) · bℓᵢ` on lane fill, **on the UNDISCOUNTED base**, explicitly
  *"dropping the weight-band×scarcity cross-term to stay linear."*

The composition research (`air_spot_composition_research_s45.md`, Q1) says the FAITHFUL, transacted-spot
rule is **multiplicative**: `r = base · wb · block`, the weight-band and scarcity being *two orthogonal axes
of one all-in number*. The redesign drops the `wb·block` cross-term — i.e. it charges every booking the SAME
premium per kg regardless of weight, computed on the full undiscounted base. That is not a harmless
linearization. Worked numbers (base $5.5, light wb=1.00, heavy wb=0.80, block steps 1.0/1.2/1.44/1.73):

| boundary | faithful: heavy-in-Bᵢ vs light-in-Bᵢ₋₁ | redesign: heavy-in-Bᵢ vs light-in-Bᵢ₋₁ |
|---|---|---|
| B0→B1 | 5.28 < 5.50 (heavy cheaper ✓) | 5.50 = 5.50 (tie — discount erased) |
| B1→B2 | 6.34 < 6.60 (heavy cheaper ✓) | **6.82 > 6.60 (INVERTED)** |
| B2→B3 | 7.61 < 7.92 (heavy cheaper ✓) | **8.41 > 7.92 (INVERTED)** |

Two distinct failures:
1. **Magnitude distortion (always):** the premium on the heavy booking is overcharged by a factor `1/wb`
   — here **+25%** (charges `base·(m−1)` instead of `base·wb·(m−1)`). Every heavy booking pays a scarcity
   premium it should not, on every block above B0.
2. **Ordering inversion (cross-block):** under the faithful rule a heavy weight-discounted booking is
   cheaper than a light booking at every block; under the redesign a heavy booking one block up pays a
   HIGHER marginal all-in rate than a light booking one block down. The weight discount is not just
   diluted — it FLIPS SIGN at block boundaries.

Why this is fatal for the thesis, not cosmetic: the entire L2 mechanism is "M₁ keeps the cheap blocks for
the cargo that benefits most." The redesign's objective tells the solver that *heavy* cargo benefits LEAST
from a cheap block (its premium-to-avoid is overstated), so M₁/M₁' will systematically push heavy cargo up
the curve and reserve B0 for light cargo — the exact opposite of the GCR economics the calibration sourced,
and a routing/consolidation artifact baked into the headline. L2 then measures a reshuffle of a *distorted*
objective, not of real scarcity value.

**This contradicts the design's own composition research.** §5 of the redesign recommends the split "because
it preserves both sourced effects" — but the research it cites says the faithful effect is the multiplicative
cross-term, which the split DELETES. The redesign is internally inconsistent here.

**Required fix (pick one, both restore correctness):**
- **(A) Make the premium variable weight-aware by splitting the block-fill var per weight band.** Index the
  lane-fill var as `bℓᵢ,β` (block i, weight-band β), with premium coefficient `baseℓ · wbβ · (multℓᵢ − 1)`.
  Still convex, still no binaries (bands are a static partition of bookings; coefficients are constants).
  Block-sum becomes `Σᵢ Σβ bℓᵢ,β = lane chargeable weight`, with per-band supply `Σᵢ bℓᵢ,β = band-β lane
  weight`. This is the linear, exact realization of `base·wb·block` and removes the inversion entirely. Cost:
  ~(#bands) × more continuous columns per lane — trivial at proof scale (3 FAX bands).
- **(B) If the cross-term truly must be dropped for tractability, drop it the OTHER way:** charge the
  premium on the **discounted** base actually billed, `baseℓ · wbβ · (multℓᵢ − 1)`, by carrying the per-band
  fill anyway — which is just (A). There is no linear way to keep ONE block-fill var AND the cross-term; you
  must disaggregate by band. So (A) is mandatory if both effects are kept.

**Spec language required:** OPEN-2 must be re-resolved. The "premium-above-base on the UNDISCOUNTED base"
form is REJECTED. Either adopt the per-band block-fill formulation (A), or — if the team accepts losing the
weight break on spot entirely — state explicitly that spot drops the GCR break (the "block ladder is the
whole spot price" alternative OPEN-2 names), which at least does not INVERT the discount, only removes it.
The current middle option is the one form that is both more complex AND wrong.

---

## Attack 2 — Lane-day block-sum vs dual billing path & validators — **NEEDS-SPEC-FIX**

The block-sum equality `Σᵢ bℓᵢ = Σ_{a∈Aℓ}(spot chargeable weight on a)` reuses the per-arc chargeable-weight
expressions, but the redesign understates three interactions:

1. **Double-count risk on the MAWB path.** `CW_{a,g}` is per **(arc, group)** (air_milp.py:379–389). A
   lane-day pools *many arcs × many groups*. Summing all `CW_{a,g}` into one BLOCK-SUM is only correct if
   every gram of MAWB chargeable weight on the lane appears in exactly one `CW_{a,g}` term — true ONLY if a
   HAWB rides at most one MAWB arc on that lane-day. The graph permits a HAWB to appear as a *candidate*
   rider on multiple arcs (selection is by `x`/`z`), and `CW_{a,g}` is bounded but its value is the routed
   CW. The redesign must state the BLOCK-SUM reads the *realized routed* CW (gated by `z_{a,g}`/`x`), not the
   candidate CW upper bound, or it will over-fill the lane and over-charge premium. **Spec fix:** define the
   RHS as the realized per-arc spot chargeable weight already used by `_build_spot_cap`, explicitly gated by
   the routing variables, and add a note that a HAWB's chargeable weight enters exactly one lane-day pool.

2. **Co-load + MAWB mixed on one lane-day.** A lane-day pool sums BOTH co-load `Σ cw_k·x` AND MAWB `Σ_g
   CW_{a,g}` arcs. These are *different billing objects* (per-HAWB vs density-mixed group CW). Mixing them in
   one block-sum is dimensionally fine (both are chargeable kg) but means a co-load kg and a MAWB kg compete
   for the same cheap block. That is the intended pooling — but the premium then attaches to MAWB *group* CW,
   which has already absorbed density mixing. Combined with Attack 3, the per-band fix (3A) must define the
   weight band of a MAWB *group* (a consolidation of many HAWBs) — there is no single weight band for a
   mixed group. **Spec fix:** specify how a consolidated MAWB group maps to a weight band (e.g. group total
   CW → band), and confirm that is consistent with how the per-booking base+discount path bands the same
   cargo. If they disagree, double-count returns.

3. **Validator scope gap.** `_validate_billing` is per-(a,g) (air_milp.py:1239–1279). It will NOT validate
   the lane-pool premium — there is no cross-arc invariant in it. Removing the per-arc cap removes a `≤` row
   the validator implicitly relied on for spot. **Spec fix (mandatory per project unit-testing gate):** add a
   lane-pool billing validator that recomputes `Σᵢ rate·b*` from the realized routed chargeable weight per
   lane-day and asserts it equals the objective's premium contribution, AND asserts `Σᵢ b*ℓᵢ` equals the
   realized lane chargeable weight (conservation). Without it, the convex-PWL has no correctness check and
   silently mis-bills are invisible — exactly the failure mode the existing validator exists to catch.

The convex-PWL exactness claim itself (no binaries, cheap-first by convexity in a minimization) is
**CORRECT** for a single homogeneous fill var — verified: increasing marginal rates in a min make filling an
expensive block before a cheaper one strictly dominated, so no ordering binary is needed. The single-HAWB
multi-block case is fine (a 1,200-kg HAWB at 400-kg blocks fills B0→B2, avg $6.67/kg) BECAUSE block-fill is
continuous and the HAWB's `x` is the same binary across the lane — the LP splits its weight across blocks
with no integrality issue. So tractability SURVIVES; it is the *accounting* (1–3 above) that needs the spec
fix.

---

## Attack 6 — Loose-corner null not structurally guaranteed — **NEEDS-SPEC-FIX**

The design ASSERTS (§2, §8) "at the loose corner every arm fills B0 and L2→0." That holds for the *spot*
channel only. But L2 = C(M₁′) − C(M₁) also captures **ULD-consolidation reshuffle value**, which a parallel
agent is measuring and which is **capacity-independent**: M₁ can re-pack which HAWBs share a MAWB / which
ride contracted ULDs even when spot is abundant and free. If that reshuffle saves money at the loose corner
(better density mixing, fewer under-pivot ULDs), L2 does NOT go to 0 there — and the block redesign does
nothing to suppress it, because it only touches the spot premium.

So the design does NOT *structurally* guarantee the null it claims to restore. It guarantees the *spot
price-of-spill* component vanishes at the loose corner; it does not guarantee the *consolidation reshuffle*
component vanishes. If the empirical loose-corner L2 stays > CI after this redesign, that is NOT evidence the
redesign failed — it is the consolidation channel, which is a separate value source.

**Spec fix:** the success criterion in §8 ("loose corner satisfies |L2| < CI") must be **decomposed**, not
asserted. Either (a) split L2 into L2_spot + L2_consol and pre-register the null on L2_spot only, OR (b)
prove (separately) that at the loose corner the consolidation reshuffle is also zero (e.g. contracted ULDs
abundant ⇒ no pivot pressure ⇒ density mixing free in both arms). Do not write "L2→0 at loose corner" into
the methodology as a structural guarantee of THIS redesign; it is a guarantee about one of L2's components.

---

## Attack 5 — n=12 / 400-kg-block lumpiness — **WEAKENS**

With ~700 kg/day/lane demand and the per-lane-day widths [400,400,400,300] (prompt), a single 1,200-kg HAWB
alone spans B0–B2. The "supply curve" is then not a smooth marginal-price object — it is dominated by integer
HAWB-size effects. A 400-kg chunk moving B0→B1 swings premium by $440; that is a large, lumpy step relative
to the ~$386 flat-model L2 and the ~$180 gap-noise floor. Consequences:
- **L2 will be a step function of κ, not a smooth monotone curve.** Whether a given HAWB's weight crosses a
  block boundary depends on integer book composition, so ∂L2/∂κ > 0 may hold *on average* but be non-monotone
  cell-to-cell. The §8 success criterion "∂L2/∂κ > 0 with a CI" is at risk of being satisfied only after
  heavy seed-averaging.
- **Continuous block-fill var hides, does not remove, the lumpiness.** `bℓᵢ` is continuous, but the *demand*
  feeding it is integer HAWBs whose `x` is binary; the LP can fractionally fill blocks but the routing
  decision that determines WHICH cargo is on the lane is integer. So the smoothness of `b` is cosmetic.

**Why WEAKENS not BREAKS:** the design is not wrong, but its "supply curve" framing oversells smoothness at
n=12. It needs forwarder scale (or seed-averaging across many books) to read as a curve. **Spec language:**
state explicitly that at proof scale (n≈7–12/day) L2(κ) is a *seed-averaged step response*, report it with
the κ-ladder CI over seeds, and do NOT claim per-cell monotonicity. Consider widening the proof to more
HAWBs/day if a smooth ∂L2/∂κ is required for the headline, or pre-register the test as a monotone trend over
the averaged ladder (Jonckheere-type), not per-cell.

---

## Attack 4 — Per-lane-day daily reset & Jensen cross-day shifting — **NEEDS-SPEC-FIX**

Convex per-period costs reward *spreading* load across periods (Jensen): two 350-kg days up the curve cost
less than one 700-kg day, all else equal. With a daily-reset pool (OPEN-1 recommends per-lane-day), the
optimizer is incentivized to split a corridor's cargo across days to stay in cheap blocks — **even when
same-flight/same-day consolidation is physically cheaper** (one MAWB, one ULD, pivot economics). Two specific
artifacts:
1. **Spurious cross-day smoothing in M₁.** M₁ sees the open book and can move cargo to a lighter day to
   dodge the convex premium. If deadlines/flight schedules permit the move it is legitimate; if the model
   does NOT bind cargo to a day by its actual deadline/flight, M₁ harvests a Jensen saving that does not
   exist physically — inflating L2 = C(M₁′) − C(M₁) artificially (M₁′ is frozen and cannot smooth).
2. **Weekly scarcity mis-stated.** A daily reset with ceiling ≈ weekly/7 lets each day refill the cheap
   block, so a corridor that is tight on a *weekly* basis looks loose daily. This *under*-states weekly
   scarcity — the opposite of the over-statement the redesign cites as its reason to reject the weekly pool.

**Spec fix:** before per-lane-day is locked, state the binding constraint that prevents non-physical cross-day
shifting — i.e. each HAWB's spot fill is keyed to the day of the flight it is actually routed on (its `x`/`z`
already pins the arc, and the arc carries `#d{day}`), so the block-fill var must be indexed `bℓ,day,ᵢ` and a
HAWB can only contribute to its routed day's pool. The redesign's lane key `(origin_region, dest, day)` does
this IF the day is the routed-arc day, not a free choice. Make that explicit: **the day in the lane key is
the routed flight's day, not a decision variable.** Then Jensen cannot be gamed (cargo can only move days by
re-routing to a different flight, which is real and deadline-constrained). Also document that per-lane-day
under-states weekly scarcity vs the sourced weekly ceiling — and justify it on replay-cadence grounds, not by
pretending it matches the weekly calibration (it does not).

---

## Attack 1 — Supply-from-demand circularity ("thin to beat spill") — **SURVIVES (one guard)**

The worry: B0 was sized thin *because* B0 < expected spill (~700 kg/day), which looks like sizing supply from
demand — the §13 v4 / D-A18 violation. Verdict: **does not break**, because the composition research
(`air_spot_composition_research_s45.md`, Q2) anchors the thin B0 to a **sourced structural fact**, NOT to this
instance's realized book: dense transpac headhaul is ~80%+ contract-locked (HKG→US <20% spot, Xeneta), so the
free-sale base block is structurally thin (~few-hundred–1,000 kg/lane-day). That is an *analytic/parametric*
anchor — the same discipline as `E[SE_k]` in `total_N`. The "~700 kg/day spill" is used to *check the
calibration is in the binding regime*, not to set the width per realized demand.

**The guard that must be in the spec (else it DOES re-introduce circularity):** the B0 width must be a fixed
parameter drawn from a distribution whose mean is the SOURCED structural number, computed with **zero reads
of `Σ SE_k` of the realized HAWBs** — identical to how `total_N` uses the closed-form `E[SE_k]`. The danger
is a future "auto-tune B0 so spill clears in B1" convenience: that WOULD couple supply to demand. **Spec
language:** "B0 width is a sourced parametric constant (mid-market base-rate access, dense-headhaul); it is
NEVER set as a function of the realized book's weight. A test mirroring the contracted `total_N` test must
assert: vary the demand seed ⇒ B0 width (and the whole block schedule + ceiling) byte-identical." With that
test in place, supply-independence is preserved. Without it, the "calibrated to beat spill" phrasing is one
refactor away from the circularity §13 killed.

Note the interaction with OPEN-3 Mechanism B: under B the ceiling mean depends on κ. κ is a knob, not demand,
so that is allowed (same as `total_N`'s κ) — but the same byte-identical-under-demand-seed test must cover
the ceiling draw too.

---

## Attack 7 — Fallback vestigial at proof scale — **WEAKENS**

Ceiling ~11,250 kg/lane-week (~1,600 kg/lane-day) vs ~700 kg/day spill ⇒ the lane rarely fills past B1–B2,
and the 2.5× fallback essentially never triggers at proof scale. Then:
- The L2_fallback component the methodology wants to split out (avoided-fallback premium) is **unmeasurable**
  in the proof — it will read ~0 not because the design is right but because the regime never reaches it.
- The "single-humped / saturating" L2(κ) shape (§8) loses its right half: without fallback ever binding, L2
  is monotone-increasing only, and the saturation story is untestable.

**Why WEAKENS not BREAKS:** the fallback is still correct as a backstop (No-Standalone-Cost-Pruning), it is
just inert at this scale. But the methodology must not claim to *measure* L2_fallback on this surface.
**Spec fix:** either (a) tighten the per-lane-day ceiling so the tight corner actually reaches fallback at
n≈7–12/day (e.g. ceiling closer to the ~700–1,000 kg/day spill so B3/fallback bind under tight κ), making
L2_fallback observable; or (b) explicitly mark L2_fallback as NOT measured at proof scale and defer it,
rather than reporting a ~0 that looks like a result. Option (a) also helps Attack 5 (more block crossings →
the curve reads less lumpy relative to the signal) and Attack 6 (clearer tight-corner signal).

---

## Bottom line

**NOT safe to write into the methodology amendment as-is.** One BREAKS (Attack 3) and three NEEDS-SPEC-FIX
(2, 6, 4) must be resolved first. The convex-PWL / no-binary / BLK-1c-safe core is sound (verified against the
MFB structure) — the failures are in pricing composition and accounting, not tractability.

**Minimum changes before amendment:**
1. **Attack 3 (must):** REJECT the undiscounted-base premium. Adopt per-weight-band block-fill vars `bℓᵢ,β`
   with premium `base·wbβ·(multᵢ−1)` (still linear, no binaries), OR explicitly drop the weight break on spot
   entirely. The current OPEN-2 split is wrong.
2. **Attack 2 (must):** define BLOCK-SUM on *realized routed* CW gated by `x`/`z`, specify MAWB-group→weight-
   band mapping, and ADD a lane-pool billing validator (project unit-test gate).
3. **Attack 6 (must):** decompose the loose-corner null into L2_spot + L2_consol; pre-register on L2_spot, do
   not claim the redesign structurally zeroes total L2 at the loose corner.
4. **Attack 4 (must):** make the lane-key day = routed flight's day (not a decision var); document that
   per-lane-day under-states weekly scarcity.
5. **Attack 1 (guard):** add the byte-identical-under-demand-seed test for B0/schedule/ceiling.
6. **Attacks 5, 7 (should):** report L2(κ) as a seed-averaged step response (no per-cell monotonicity claim);
   either tighten the ceiling so fallback binds at proof scale or defer L2_fallback explicitly.
