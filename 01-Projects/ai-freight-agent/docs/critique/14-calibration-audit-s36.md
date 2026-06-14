# Calibration / Assumptions Audit — Session 36 (post Slice-A F1)

**Auditor role:** Assumptions / Calibration Auditor (1 of 3 standing proof-hardening agents).
**Scope:** read-only. Headline result under audit: **L2 = C(M₀) − C(M₁)**, swept over the **(κ, α, λ)** grid.
**Calibrated to:** S36 Slice A — contracted ULD capacity now drawn independently on a `supply` RNG
stream; `capacity_scale` retired; κ (network tightness) + α (Dirichlet lumpiness) introduced;
`total_N = round(n_hawbs·E[SE_k]/κ)`, `E[SE_k] ≈ 0.6632` slots/HAWB (closed form).
**Verdict in one line:** the new supply model is *more* honest than the old circular one, but it has
imported **two new unfalsifiable knobs (α, the κ→regime price map)** and one **silent internal
inconsistency (slot-density 333 vs billing-density 167)** that, if left unpinned, can move L2 by
more than the effect being measured. Slices B and C are unbuilt, so the *dominant* $ lever (spot
pricing) is not yet under the falsifiability design at all.

---

## Method note

Every `[CAL]`, `CALIBRATION NEEDED`, hard-coded band, and uncited magnitude was enumerated across:
`arrival_only_replan_methodology.md` (§10, §12, §13), `flexibility_model.md`, `air_transit_time.md`,
`backtest_methodology.md`, `air_freight_routing.tex`, and `data/synthetic/air_generator.py`. Each is
rated by **headline-sensitivity** = how much L2 (the *difference* C(M₀)−C(M₁)) moves when the knob is
swept within a defensible range, and flagged **unfalsifiable** if its value can produce the desired
answer with no external anchor.

The key discipline question (§13 / D-A10): is the (κ,α,λ) sweep an honest falsifiability design, with
the loose corner gated `|L2| < CI`? **Answer: the *gate* is honest, but it only constrains the
loose corner. Nothing constrains the choice of where the *peak* sits, and several knobs below can
inflate the peak without ever violating the loose-corner null.** That is the core risk.

---

## Ranked findings (highest headline-risk first)

### F1 — α (Dirichlet concentration) is an unfalsifiable L2 amplifier. **[HIGH, UNFALSIFIABLE]**
`generator: GenConfig.alpha`, `_draw_network_supply`. α controls supply *lumpiness*: low α ⇒ ULD
positions pile onto a few flights ⇒ severe local idle-cheap-here / spill-there mismatch ⇒ exactly the
condition that makes reshuffle pay ⇒ **large L2**. High α ⇒ even spread ⇒ L2 → 0.

α is the single most direct manufacture-or-destroy knob in the new model, and it has **no external
anchor whatsoever**. There is no public dataset that says "real forwarder BSA positions are
Dirichlet(α=0.5) lumpy." The methodology lists it as `[CAL]` and says headline is "reported on the
(κ,α) plane" — but reporting a *plane* does not save you if a reviewer asks "which α is real?" and the
answer is "we don't know." The loose-corner gate (high-α + abundant-κ + early-λ) does NOT discipline
this: it only proves L2≈0 in the even-supply corner; it says nothing about whether the *lumpy* corner
where you report your headline reflects reality.

**This is the most dangerous knob introduced this session.** Mitigation is not "sweep it" (already
planned) — it is **anchoring the *range* of α to something observable** and **leading with the L2(α)
curve, not a single α cell**, so the claim becomes "L2 ranges X–Y as supply lumpiness goes from
realistic-even to realistic-concentrated," with the realistic endpoints defended. Without an α anchor,
the honest headline is the *whole curve*, and the peak is uncitable.

Provenance to pin α's range: **BTS FAF** lane-flow concentration + **OpenSky** freighter-frequency
dispersion per lane give a defensible empirical Gini/HHI on how concentrated real capacity is across
flights on a corridor. That maps to a defensible α band. Until that exists, α is adversary-pickable.

### F2 — Slot-density (333) vs billing-density (167) divergence is a silent calibration commitment. **[HIGH, SILENT]**
`generator: _expected_slot_mean` uses physical ULD packing `SE_k = max(w/1500, v/4.5)` → volume binds
below **333.3 kg/cbm**. The density band is `uniform(120, 240)`, so the closed-form assertion
`4.5·240 < 1500` holds and `E[SE_k]=0.6632` is arithmetically correct **for physical slot counting**.

But the **MILP bills chargeable weight on `VOLUMETRIC_DIVISOR = 167.0`** (`air_milp.py:76`), i.e. the
volumetric-weight break-even is **167 kg/cbm**, not 333. Over the band 120–240:
- **120–167 kg/cbm**: volumetric-bound on *both* slot count and billing.
- **167–240 kg/cbm**: still **volume-bound on physical slots** but **actual-weight-bound on billing**.

So for roughly the upper half of the density band, **capacity is rationed by volume while cost is
charged by weight** — the κ knob (which counts physical slots) and the $ a HAWB pays diverge. This is
not a bug per se (slots and billing legitimately use different ULD conventions), but it is a **hidden
calibration commitment**: the *value* of freeing a slot (what L2 monetizes) depends on the billing
density, while the *tightness* (κ) depends on the slot density. Choosing `_DENSITY_HIGH = 240` keeps
`E[SE_k]` closed-form *and* keeps most cargo billing on actual weight; nudging it differently changes
the slot-vs-dollar coupling without touching the headline assertion. **The 240 ceiling is doing
load-bearing work in two unrelated places and is documented as serving only one.**

**What breaks past 333:** the docstring's `assert 4.5·240 < 1500` is a real tripwire — if the density
band is recalibrated up past **333 kg/cbm**, the densest cargo becomes *weight*-bound on slots, `SE_k`
is no longer `v/(density·V_uld)`, and the closed-form `E[SE_k] = E[w]·E[1/density]/V_uld` is wrong;
the assertion fires and the supply draw (hence κ's meaning) silently breaks. Real air cargo routinely
exceeds 333 kg/cbm (machinery, auto parts: 400–600+). So the **120–240 band is itself a calibration
commitment that the proof only works for low-density cargo**, and that commitment is buried in a
docstring assertion, not surfaced as a headline scope caveat. If the calibration note pins density to
real BTS FAF commodity mix, this band will likely break and the closed form must be re-derived as
`E[max(w/W, v/V)]` over a crossing band (an integral, not a product).

### F3 — The κ→regime spot-price map is the dominant $ knob and is **not yet built**, so currently unfalsifiable by omission. **[HIGH, UNFALSIFIABLE / UNBUILT]**
`backtest §7` + memory `reference_air_spot_contract_ratio` correctly identify spot:contract as "the
most controllable lever on the L2 *dollar* figure" and pin it to a two-sided sourced band (soft ~0.85,
peak ~1.18) **tied to κ**. Sourced anchors are good. **But Slice C is unbuilt:** in the current
generator the spot path is just `coload_per_kg ~ uniform(3.5,5.5)` and `flat ~ uniform(3.0,5.0)`
(`_build_rate_catalog`), and the contracted `r_a ~ uniform(2.5,4.0)` — **overlapping static bands with
no κ-tie, no regime, no two-sided multiplier `m`, and no per-arc spot CW-cap**. The §13 D-A19
machinery (`base × m`, `m` two-sided, `Σ cw_k·x ≤ cap^spot`) exists only on paper.

Consequence: **L2 reported in % (as mandated) is the right defense**, but the *dollar* figure is
currently governed by adversary-pickable overlapping uniforms, and even the %-figure depends on the
spot-to-contract *gap*, which the static bands set arbitrarily (a contracted draw of 2.5 vs a spot
draw of 5.5 is a 2.2× gap; 4.0 vs 3.5 is 0.875× — both reachable on the same seed range). The
methodology's κ→regime map is the fix, but **it is the single most important unbuilt piece for L2
credibility**, and until Slice C lands the dollar headline is unfalsifiable by construction. The
inferred parts (normal≈1.0, within-regime spread, the κ→regime map itself) are flagged in memory as
inferred — good — but the *map function* (which κ maps to which regime mean) has no anchor and is a
direct multiplier on every avoided-premium dollar.

### F4 — The fallback dominance factor (2.0 here vs 1.5 in §13) is a live spec/code drift that conditions L2_fallback-avoidance. **[MED, SILENT DRIFT]**
`generator: _FALLBACK_DOMINANCE_FACTOR = 2.0`, `worst_real = 2·per_leg + ground`. §13/D-A19 amended
the fallback to **`1.5 × worst-spot-route`** with **graph-derived `max-air-legs`** (not a literal 2
legs). The code still has `_FALLBACK_DOMINANCE_FACTOR = 2.0` and a **hard-coded `2.0·per_leg`** (two
air legs). Under D-A24's region→region expansion, routes can have >2 air legs, so the hard-coded 2 can
**under-bound** the worst route and the fallback may *not* dominate — exactly the failure §13 calls
out. The fallback level directly sets `L2_fallback-avoidance` (one of the three L2 components), and a
2.0 vs 1.5 factor is a 33% swing on every capacity-rescue dollar. This is pre-Slice-C code that will
be rewritten, but flag it: **the headline split L2_reshuffle / L2_fallback-avoidance is sensitive to a
factor currently inconsistent with its own governing spec.**

### F5 — Book-lead B mean/spread (48 ± 24h) is the λ axis's calibration and is silently fixed in shape. **[MED]**
`ArrivalConfig.book_lead_mean_h = 48.0`, `book_lead_spread_h = 24.0`, `lambda_compress` swept.
λ is swept (good — it compresses the mean toward the cutoff). **But the *spread* (24h) is fixed and
the *shape* is uniform**, both `[CAL]`. The spread sets how much of the book overlaps the cutoff at a
given λ, which is precisely the "how much is unknown when capacity commits" quantity L2 monetizes. A
wider spread at fixed λ-mean manufactures more late arrivals ⇒ more premature commitment ⇒ more L2,
**without moving the swept λ axis**. So λ is honestly swept but its companion (spread) is silently
pinned and is itself a back-door L2 lever. The D-A9 decision to draw lateness *tier-independently* in
the headline is the right falsifiability move (removes the "built to win" arrival asymmetry); the
residual risk is the un-swept spread. Provenance: BTS FAF seasonality + booking-curve shape (no free
source gives booking lead-time distributions; this is genuinely hard to anchor — flag as inferred and
sensitivity-check the spread).

### F6 — Tier-coupled book-lead means (12/48/96h) are a load-bearing empirical claim, correctly demoted to upper-bracket. **[MED, HONESTLY HANDLED]**
`book_lead_coupled_h = (12, 48, 96)`, gated behind `tier_coupled_arrival` (default False). §12/D-A7
already flags this as "a load-bearing empirical claim `[CAL]`, reported as an upper bracket only," and
D-A9 makes the tier-*independent* draw the headline. **This is the model of how a `[CAL]` should be
handled** — the favorable assumption is quarantined out of the headline. No action beyond sourcing the
band if the upper bracket is ever published. Provenance: same booking-curve gap as F5; inferred.

### F7 — Tier mix 20/40/40 swept; SLA offsets and θ_flex silently fixed. **[MED]**
`tier_mix = (0.20, 0.40, 0.40)` is a config and sweepable (D-F5) — good. But `sla_offset_h(tier)`,
`z_tier` (retired for air), `w_sp`, and `θ_flex` (the flex-separation threshold) are `[CAL]` and
**fixed**. These set `cw_flex` (the per-flexible-kg denominator) and which HAWBs count as flexible —
i.e. the denominator of the L2/cw_flex rate and the size of the reshufflable set. `flexibility_model.md`
correctly makes *sandbagging* (`shrink sla_offset + raise θ_flex`) the band's primary sensitivity, so
the *direction* is disciplined. Residual risk: the *headline* `sla_offset` levels (the 90/80/70 OTP
anchors are user-stated, not sourced) set the baseline flexible fraction, and only the pessimistic
edge is swept — the optimistic edge (generous SLA ⇒ more flexible ⇒ bigger reshuffle set) is not
bracketed. Provenance: tier SLA offsets are standard-product structure (defensible *ordering*); levels
need design-partner data, no free source.

### F8 — Cargo weight triangular(50,1200,300) and density uniform(120,240) jointly set E[SE_k] and the whole supply scale. **[MED→HIGH coupling]**
`_WEIGHT_LOW/HIGH/MODE = 50/1200/300`, `_DENSITY_LOW/HIGH = 120/240`. These feed `E[SE_k]=0.6632`
directly (`E[w]=516.67`, `E[1/density]=0.005776`). **`total_N = round(n_hawbs·0.6632/κ)`**, so at
n=20, κ=1 ⇒ total_N=13 positions over "dozens of flights" — genuinely tight, ~10 as §13 claims. But
**every one of these five numbers moves `E[SE_k]` and therefore the absolute supply level at a given
κ.** Raise the weight mode and total_N rises (looser supply at fixed κ-label); narrow the density band
upward and `E[1/density]` falls (tighter). So the (weight, density) bands are a **hidden recalibration
of what κ=1 means** — the κ label is only as anchored as these two distributions. This is the
mechanism by which "κ=1 is unit-tight" can silently become "κ=1 is actually loose" (§13 acknowledges
the no-consolidation upper-bound bias makes κ=1 *looser* than unit-tight after consolidation, which is
conservative — but the *distributional* choice of E[SE_k] is a separate, un-acknowledged lever).
**Report realized post-consolidation occupancy per κ-cell (§13 already requires this)** — that is the
honest check that converts the κ-label into an observable tightness. Provenance: BTS FAF commodity
weight + density mix per lane (real, free) — this is sourceable and should be, because it
simultaneously fixes F2, F8, and the κ-label meaning.

### F9 — `n_hawbs` fixed-N (D-A8) is an L2-deflating simplification, acknowledged. **[LOW]**
`n_hawbs=20/30`. §12/M-B9 already says to report the `known_at` distribution and state fixed-N as an
L2-deflating simplification. Honestly handled; flag only that the *scale* (20–30) is where the integer
κ ladder lives, and the smooth (κ,α) plane is a forwarder-scale claim not testable at proof scale
(§11). No manufacture risk; if anything conservative.

### F10 — Static .tex calibration constants (`V_ref=$10k`, `w_p`, customs/handling costs). **[LOW for L2]**
`air_freight_routing.tex`: `V^ref = $10,000 (CALIBRATION NEEDED)`, `w_p (CALIBRATION NEEDED)`. These
feed the C.10 tardiness penalty `W_k`, which D-A12 **excludes from realized_cost** (objective-steering,
not cash). So they steer routing but do not enter C(π) directly. On the headline (`tardiness_weight_scale
= 0.0` default ⇒ W_k=0), they are **inert**. Low L2-sensitivity *as long as the headline keeps W_k=0*;
becomes MED if the cost–OTP frontier (α-sweep) is ever folded into the headline. The fixed
pickup/delivery/customs costs (120/150/200, cartage/cfs) enter `ground` in both real routes and the
fallback, so they partly cancel in L2 (a difference); low net sensitivity.

---

## Honest accounting: swept vs silently-fixed

| Knob | Status | Risk if the silent ones bite |
|---|---|---|
| κ (network tightness) | **swept** (coarse integer ladder) | — but its *meaning* rides on F8's distributions |
| α (Dirichlet lumpiness) | **swept** | F1: range is unanchored ⇒ peak uncitable |
| λ (book-lead compression) | **swept** | F5: companion *spread* is fixed |
| M₀ tie-break `(tender_at,tier,id)` | **fixed-deterministic** (correct, D-A11/D-A23) | none — required for C(M₁')==C(M₀) |
| transit `s` | **swept** (s=0 headline, s>0 stress) | none for air headline |
| `E[SE_k]` weight/density bands | **silently fixed** | F2, F8 — recalibrates κ-label + may break closed form |
| spot:contract gap / κ→regime map | **unbuilt** (Slice C) | F3 — dominant $ knob, currently overlapping uniforms |
| fallback factor (2.0 vs spec 1.5) | **silently fixed, drifted** | F4 — sets L2_fallback-avoidance |
| book-lead spread (24h) | **silently fixed** | F5 — back-door λ |
| sla_offset / θ_flex levels | **fixed (sandbag = pessimistic edge only)** | F7 — optimistic edge unbracketed |
| tier mix 20/40/40 | **swept** | low |

**The loose-corner `|L2|<CI` gate (D-A10) is an honest null** — it forces L2≈0 in the
abundant-κ × even-α × early-λ corner, which catches a *generically broken* engine. **But it does not
constrain the peak**, and F1/F3/F5/F8 all inflate the *peak* without touching the loose corner. So the
sweep is *partially* gameable: you cannot fake "replan never helps," but you can inflate "how much
replan helps" via α-range, the spot gap, the λ-spread, and the E[SE_k] band — none of which the
loose-corner gate sees. **Falsifiability of the *magnitude* requires anchoring those four, not just the
gate.**

---

## Calibration-note must-pin-down checklist (free sources only)

| # | Knob (location) | Current placeholder | Defensible real range | Source to pin (free) | L2-sensitivity |
|---|---|---|---|---|---|
| C1 | **α Dirichlet concentration** (`GenConfig.alpha`) | 1.0, swept | map to capacity-dispersion HHI/Gini on real lanes | **BTS FAF** lane flow + **OpenSky** freighter freq dispersion | **HIGH** |
| C2 | **Density band** (`_DENSITY_LOW/HIGH 120/240`) | uniform(120,240) | real commodity densities span ~80–600+ kg/cbm; **crosses 333** | **BTS FAF** commodity mix → density per lane | **HIGH** |
| C3 | **κ→regime spot map + two-sided `m`** (`backtest §7`, Slice C unbuilt) | overlapping uniforms (no κ-tie) | soft 0.85 / normal ~1.0 / peak 1.18 (sourced); map + spread inferred | WorldACD/FreightWaves/Xeneta anchors (have URLs); lane precision needs paid TAC/Xeneta | **HIGH** |
| C4 | **Weight distribution** (`50/1200/300` tri) | triangular | per-lane HAWB weight dist | **BTS FAF** tonnage/shipment | **MED→HIGH** (sets E[SE_k]) |
| C5 | **Fallback factor** (`_FALLBACK_DOMINANCE_FACTOR 2.0`) | 2.0 + hard-coded 2 legs | spec says 1.5 × graph-derived max-air-legs | internal (spec D-A19) — just reconcile | **MED** |
| C6 | **Book-lead mean/spread** (`48 ± 24h`) | uniform(24,72) | air booking lead-time curve | no free source; flag inferred, sensitivity-check spread | **MED** |
| C7 | **Tier-coupled lead (12/48/96)** | upper-bracket only | — | quarantined; source only if bracket published | **MED** (bracketed out) |
| C8 | **SLA offsets / θ_flex** (`TierSpec`) | `[CAL]`; sandbag = pessimistic edge | standard express/std/deferred structure | product structure (ordering defensible); levels need partner | **MED** |
| C9 | **Tier mix** (`20/40/40`) | swept | per-lane service mix | **BTS FAF** + partner | LOW |
| C10 | **V_ref $10k, w_p** (.tex) | CALIBRATION NEEDED | — | inert while W_k=0 on headline | LOW (headline) |

DAT is **not licensed** — no checklist item depends on it. UN/LOCODE + IATA fix topology only (already
real). NOAA AIS is ocean (Stage 4). The two highest-value free sources for *this* proof are **BTS FAF**
(weight/density/flow concentration → fixes C1, C2, C4 simultaneously) and the **already-sourced
WorldACD/Xeneta anchors** (C3).

---

## The single most important thing to pin before the (κ,α,λ) sweep is trusted

**Anchor the range of α (and report L2 as a curve over it, not a peak cell).** α is the only knob
introduced this session that *directly manufactures or destroys L2* — low α creates the local
supply/demand mismatch that *is* the replan value, high α erases it — and it currently has **zero
external anchor**. The (κ,λ) axes are either swept-with-a-sourced-tie (κ→spot once Slice C lands) or
quarantined (λ-spread, tier-coupling). α is swept but its *endpoints are adversary-pickable*, and the
loose-corner null does not constrain the lumpy corner where the headline is reported. Until α's range
is pinned to an observable capacity-dispersion statistic (BTS FAF lane concentration + OpenSky
freighter frequency), the honest headline is "L2 ranges X–Y across the realistic α band," with both
endpoints defended — not a single number. Pin α first; it is the load-bearing assumption of the
entire redesigned supply model.
