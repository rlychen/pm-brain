# Critique 21 — Simulation design v2, six-agent verification round (S56)

**Object under review:** `docs/design/simulation_design_s56.md` **v2** (the self-contained draft).
**Round design:** 5 no-code reviewers + 1 code-as-inventory miner. The five reviewers were barred from
the source tree — ground truths were (1) physical/commercial reality (sourced or NOT FOUND), (2)
mathematics and simulation methodology, (3) the document's internal consistency. Rationale: the code is
known defective; agreement with it is evidence of nothing (the structural failure of critiques 1–20 was
checking artifacts against artifacts). The miner had the inverted mandate: read the code purely as an
INVENTORY and classify each mechanism as specified / deleted / silently omitted — forbidden from
correctness opinions.

**Disposition:** all findings folded into design v3 (same file). Key outcomes: v2's default world
failed its own gates (τ floor ≈ 2.0 vs band [1.0, 1.3] — found independently by agents 1, 3, 4); the
one-ULD fleet passed the gate built to catch it (M5); a new nonsense instance (Monday-batch) passed
everything; the L2 CI gate was circular; the warm-up bound was wrong (memory composes); cost keying
had a hole (arms paid to strand); the tier-ladder gate was mathematically false; 28 silent omissions.

Reports below are verbatim.

---
---

## Agent 1 — Supply arithmetic (no-code)

I read the full design doc, plus the S50 lane-tightness grounding and `air_capacity_parameters.csv` as
sources. All arithmetic below uses only the spec's stated draws. Note on convention: §2 derives "market
τ ≈ 1.11" from a 90% load factor, i.e. **τ = supply/demand**; §5's "low μ starves large shipments" is
consistent with that (low τ = tight). I use that convention and give the inverse where it matters — the
findings hold under either reading.

**Common arithmetic (per week, expectations).** Book: 120 shipments × 500 kg = 60,000 kg. Volume:
E[1/ρ] for U(150,250) = ln(5/3)/100 = 0.005108, so E[v] = 2.554 m³/shipment → **306.5 m³/wk = 68.1
LD3-volumes**. Chargeable: E[max(1,167/ρ)] = 1.009 → **~60,540 CW-kg/wk**. Volume binds (weight needs
only 40 LD3-weights) — §2's claim checks out. Supply positions: BSA = 9 lanes × 7 × E[floor(ν+U′)] =
**63ν**; spot = 0.40 × 182 legs = 72.8 accessed flights × E[floor(μ_ℓ+U)] = **72.8μ** (E[floor(x+U)] = x
exactly — the draws are mean-exact, good design). Total P = 63ν + 72.8μ positions; capacity =
P × (1500 kg, 4.5 m³).

**F1 — CRITICAL: the τ band is unreachable everywhere on the sweep grid; the world is 2–6×
oversupplied.**

| μ | P (ν=1) | kg / m³ | τ_v = P/68.1 | P (ν=2) | kg / m³ | τ_v |
|---|---|---|---|---|---|---|
| 1.0 | 135.8 | 203,700 / 611 | **1.99** | 198.8 | 298,200 / 895 | **2.92** |
| 2.5 | 245.0 | 367,500 / 1,103 | **3.60** | 308.0 | 462,000 / 1,386 | **4.52** |
| 4.0 | 354.2 | 531,300 / 1,594 | **5.20** | 417.2 | 625,800 / 1,877 | **6.13** |

τ_w = P/40 runs 3.4–10.4 (weight rows dead, as promised). The band [1.0, 1.3] requires **P ∈ [68.1,
88.5] positions/wk**; the loosest possible corner of the design (ν=1, μ=1, all floors binding) already
yields P = 135.8, i.e. τ_v = 2.0. No μ lands in band; worse, μ only moves τ *away* from tight (adding
supply), so the entire sweep lives in the slack regime — scarcity never binds, capacity-driven L2 ≈ 0
by construction, and M3's "default cell in band" gate will fail on the first run. Applying booking-curve
decay doesn't rescue it (spot at φ(5–7 d) ≈ 0.55–0.65 nominal still gives effective τ_v ≈ 1.6 at the
loosest corner). Root cause: the 9-lane × daily grid with integral ≥1-LD3 blocks sets a supply *floor*
(63 BSA + 73 spot positions) that alone exceeds the whole 68-position book. **Corrected rule:**
BSA_positions + φ·182·μ_default must be ≤ ~88. Two consistent fixes — (a) keep the 120/wk book: BSA ≈
36 positions/wk (≈4 dep/lane-week × 1 position) + φ ≈ 0.125 (≈23 accessed flights), default μ = 2 →
P = 82, τ_v = 1.20, with μ=1 → 0.87 (deliberately tight) and μ=4 → 1.88; or (b) keep the supply shape
and grow the book ~4× (≈490–500 shipments/wk for τ_v = 1.15 at ν=2, μ=2.5) — tractability impact
unassessed against §9.

**F2 — CRITICAL: ν ∈ [1,3] is arithmetically incompatible with the 55% BSA target, and starves the
mechanism under test.** BSA CW capacity = 63ν × ~900 = 56,700ν CW-kg vs a 60,540 CW-kg book: coverage
= **94% of the book at ν = 1** (187% at ν=2). Hitting 55% needs mean **ν ≈ 0.59** — below the stated
range floor — or ~4 departures/lane-week at 1 position (≈37 positions = 56%). As written,
marginal-cost-zero BSA absorbs essentially the whole book, so spot carries ~nothing — and the spot
decay/reservation apparatus (the S54 design this sim exists to exercise, §0/§11 Q5) sees no traffic.
L2's commit-timing content dies with it.

**F3 — HIGH: block granularity vs the shipment tail — "born dead" is a design consequence, not a
generator bug.** In any in-band configuration the median flight is ~1 LD3 (4.5 m³) and the largest
~2–3 positions (9–13.5 m³). A shipment needs w ≤ 9ρ–13.5ρ to fit one flight: at the truncation cap of
5,000 kg (25–33 m³) nothing fits; even shipments above ~1,400–2,700 kg (roughly **2–7% of the book**,
P(w>1400) ≈ 7%, P(w>3375) ≈ 1.4%) can exceed every envelope. M1's gate says "≥1 itinerary for every
shipment else generator bug" — it will fire on spec-consistent draws. Already true as-designed at μ=1:
max flight ≈ 5 positions = 22.5 m³ < a 5 t shipment's 25–33 m³. The spec must pick one: allow splitting
a shipment across flights, lower the weight cap to ~2,500–3,000 kg (rejection-resample; state it), or
guarantee multi-position flights per lane per few days and scope M1 accordingly. Related minor:
truncating at [30, 5000] drops the lognormal mean to ~476–485 kg (not 500) — state the truncation
method and quote the truncated mean, since τ is calibrated on it.

**F4 — MEDIUM: μ_ℓ = μ·s_ℓ is not well-defined as written.** Three defects. (a) *Floor censoring:*
with S50 bands, s_ℓ = u_i·v_j ranges ≈ [0.54, 1.74] pre-normalization; at μ = 1 every lane with
s_ℓ < 1 has m_f = max(1, floor(μ_ℓ+U)) ≡ 1 — the realized accessed-flight mean exceeds μ
(normalization violated), lane heterogeneity collapses at the tight end of the sweep, and U_f is inert
there but active at μ = 2.5 (weakens the CRN story). Pathwise monotonicity survives (max∘floor is
monotone), but the dial is biased at low μ. Rule to pin: normalize s_ℓ flight-weighted over the
accessed set *pre-floor*, and report realized mean m̄(μ) per cell rather than asserting it equals μ.
(b) *Feeder legs have no lane:* TPE→HKG and PVG→HKG are among the 26 legs and inside the φ = 0.40
pool, but s_ℓ is indexed by the 9 O-D region lanes — μ_ℓ is undefined for them. (c) *BSA lane-vs-leg:*
"1 departure/day/lane" for TPE/PVG-origin lanes routed via HKG is a two-leg path; R2 keys capacity on
flights, but the spec never says whether the lane's BSA reserves feeder-leg positions too.

**F5 — MEDIUM: take-or-pay economics are incoherent at the stated pivot and sizing.** The carried-over
"current design" pivot (CSV Table A) is 1,200 kg/position at $4.2/kg — but an LD3 cubes out at
max(4.5ρ, 751.5) CW ≤ **1,125 kg even at ρ = 250**, ~900 kg typical. The pivot is unattainable for
this cargo mix: ≥6–25% structural dead-freight at 100% cube fill. On top of that, sizing: at ν=2 the
book can physically cover at most 60.5k/151k ≈ 40% of BSA CW → ≥60% of the $635k/wk BSA bill is dead
freight; even ν=1 best-case (book routed BSA-first) covers ~80% of pivot. No forwarder signs that.
Corrected: pivot ≤ ~850–900 kg CW/position, and BSA sized so expected fill ≈ 85–95% — with the F2
correction (≈36 positions carrying the 55% share = 33.3k CW on 30.6k of pivot) utilization lands
~109%, i.e. a healthy allotment.

**F6 — Shipments-per-flight and M5 (answering Q2): M5 passes everywhere; consolidation is real but
thin at the corners.** Median flight cap / median shipment CW (250 kg): worst corner (μ=1, ν=1) =
900/250 = **3.6** (clears K=3, barely); (μ=1, ν=2) = 7.2; (μ=2.5, ν=2) ≈ 9; (μ=4) ≈ 14. Caveat: by
*mean* CW (504 kg) the worst corner is 1.79 — the median hides the heavy tail (this is F3 in disguise;
the gate as written passes). Loading: spread evenly, 120 shipments over ~111 accessed flights ≈
1.1/flight; a concentrating optimizer packing to capacity uses ~38 / ~25 / ~18 flights at μ = 1 / 2.5 /
4 (ν=2) → ~3.2 / ~4.9 / ~6.6 shipments per used flight. The H0-scatter-vs-M1-concentrate gap exists,
but in the as-designed slack world it is driven purely by cost jointness, never scarcity (see F1).

**F7 — LOW: the demand back-door (Q5) is a single, acceptable leak — if it is frozen.** The only place
supply reads demand is ν "sized so BSA covers ≈55% of the *default* book" — a one-time,
expectation-based scale-setting, not per-draw coupling; `n_hawbs` genuinely appears nowhere in the
capacity path, and τ remains a live output. It does not recreate the τ-inertness disease *provided* ν
(or BSA positions/wk) is pinned as a literal number at sign-off, not implemented as a rule that reads
the demand config — otherwise Q1's open book-size decision (40 vs 120/wk) would silently rescale
supply and re-infect. Same discipline for the τ-band requirement: it is correctly a calibration
*check* on an output, not a generator input.

**F8 — LOW: pin the τ convention in §5.** §2's "market τ ≈ 1.11" is supply/demand; the band [1.0,1.3]
and "τ_v is the binding one" only cohere under that reading (and then M3's monotonicity in μ is
*increasing*). One sentence fixes it; as written a builder could implement demand/supply and invert
every gate.

**Bottom line.** The supply model's architecture is right — exogenous, integral, book-blind,
mean-exact draws, CRN-frozen — but its constants are mutually inconsistent by ~2–6×: the daily 9-lane
BSA grid plus φ = 0.40 with ≥1-LD3 floors cannot coexist with a 120/wk × 500 kg book, the τ_v ∈
[1.0,1.3] band, the 55% contracted target, and M1's no-born-dead gate. One coherent repair set: BSA ≈
36 positions/wk (4/lane-week), pivot ≈ 850–900 kg, φ ≈ 0.125, default μ = 2, weight cap ~3,000 kg or
explicit splitting — or scale the book ×4 and re-check §9 tractability. Also resolve the feeder-leg
lane assignment before build.

---
---

## Agent 2 — Demand model & timeline (no-code)

All reading done — the design doc plus the three permitted grounding docs. Findings below, ranked;
each with the corrected rule or constant.

**F1 (blocking) — `base_transit(lane)` is never defined, and it is the load-bearing constant of the
entire timeline.**
§3 uses it in `deadline`, §6's cool-down depends on its max, and the commit_backstop path set depends
on it — yet v2, which claims to "pin every distribution/constant v1 left implicit," gives no formula,
no per-lane values, no max. The ~112h max is only recoverable by *back-solving* the doc's own ≈28d
cool-down (664h − 456h gap − 96h offset = 112h); it appears nowhere as a stated constant. Also
unpinned: the in-ground-chain range (only the out-chain gets "~7–31h, median ~10h") and the
flight-visibility horizon (named in the warm-up formula, no value). Corrected rule: `base_transit(lane)
= median ground_out + scheduled block time (incl. feeder connection where applicable) + median
ground_in + W`, with the wait allowance `W` stated explicitly per lane, tier-agnostic, and a table of
the 9 lane values (or the FreightNet-derived formula) in §3. Whether the express tier is even
serviceable (F2) is entirely a function of `W`.

**F2 (blocking) — express is structurally born-tardy under the stated distributions, and the M1 gate
then rejects valid worlds.**
Construction: express gap ~U(0,24)h, sla_offset 12h. Accessible departures per lane ≈ 1 BSA/day +
φ·spot ≈ 2/day ⇒ inter-departure spacing ~12h (worse on narrow-`s_ℓ` lanes, up to 24h). The on-time
departure window per shipment has width `W + 6h` (deadline slack minus cutoff); if `W` is small, a
shipment whose `ready + ground_out` lands just after a cutoff waits ~12–24h against 12h of slack ⇒
**zero deadline-feasible paths**. That is not a generator bug, but M1's gate ("≥1 eligible itinerary
for every shipment else born dead = generator bug") fires anyway — the gate is unsatisfiable for a
material express fraction. Layer on the daily cadence: mean plan delay ~12h equals the entire express
offset, so even eligible express shipments miss their flights operationally; express OTP becomes a
cadence-phase artifact, not an output of any arm's decisions. Standard tier (offset 24h ≈ max wait on
thin lanes) is borderline for the same reason. Corrected rule — pick one: (a) `sla_offset(E) ≥
cadence(24h) + spacing/2 ≈ 36h` (rescale S/D to preserve ordering), (b) embed `W ≥` inter-departure
spacing in base_transit, or (c) restrict M1's gate to conditions (i)+(ii) for express and let (iii) be
a reported rate. The doc must choose explicitly.

**F3 (blocking) — the commit rule is not total, and fallback is an absorbing state reachable by pure
horizon artifact.**
Three undefined states: (a) *unplaced at backstop* — commits to what? (b) *standing on a tardy flight*
whose cutoff > backstop — min(cutoff, backstop) forces firmness while standing on nothing
deadline-feasible; (c) *empty deadline-feasible path set* (the F2 cohort, plus late cool-down shipments
with truncated schedules) — backstop is a max over ∅, undefined. Separately: deferred gap runs to
456h = 19d before ready; if the flight-visibility horizon is shorter than book→dep lead (~27d) and §0
step 3 "routes the movable book" with fallback as the only feasible arc, a just-booked deferred
shipment falls back and **retires permanently** (§0 step 6) purely because its flights aren't visible
yet. Corrected rules: (i) the MILP may leave a shipment *unplaced* (holding pattern) — fallback is only
chargeable/absorbing at backstop, never before; (ii) at backstop an unplaced shipment commits to
fallback; (iii) if the deadline-feasible path set is empty at generation, define `commit_backstop :=
latest cutoff over paths meeting deadline + T_max` (see F4), and flag the shipment; (iv) pin the
visibility horizon ≥ max book→dep lead (~27d) or state the defer-don't-fallback rule.

**F4 (major) — cool-down truncates exactly the tardy region; maximum tardiness is an unpinned
constant.**
The arithmetic holds as far as it goes: cool-down = max(deadline − book_at) = 456 + 112 + 96 = 664h =
**27.7d ≈ 28d** ✓; warm-up = max book→dep lead ≈ 664 − (min flight + min ground_in) ≈ 664 − 23 ≈ 641h
= **26.7d ≈ 27d** ✓; correct generated span = 27 + 21 + 28 = **76d** (the diagram's "~70–76d" lower
end has no derivation from these constants — it should just say ~76d). But `max(deadline − book_at)`
bounds only *on-time* flights. The tardiness penalty (a CLAUDE.md hard rule — always on) means tardy
arcs must exist in the MILP, and those depart *after* deadline − flight − ground_in. A shipment booked
on the last scored day therefore has its tardy-fly options truncated by the generation edge and is
pushed to fallback asymmetrically — precisely the M6 bookend leak the design says it detects, built in
by construction. Corrected rule: pin `T_max` = maximum modeled tardiness (cap tardy arcs at `deadline
+ T_max`), then cool-down = max(deadline − book_at) + T_max ≈ 28d + T_max. Without a pinned T_max the
MILP arc set is undefined anyway.

**F5 (major) — `known_at ≡ book_at` contradicts the project's own grounding and the doc is silent
about it.**
The demand grounding is explicit: bookings 7+ days out are 2–3× likelier to cancel
(WebCargo/Freightos, F1d), show-up exceeds 100% via over-tendering (booked ≠ tendered in both
directions, F8), and its §3 *recommends* exposing an attrition-with-lead-time knob as "what makes
early pre-committed volume genuinely uncertain." The design deletes the second information event
entirely — weight/volume/ready/deadline certain at booking, no cancel, no reconciliation. Acceptable
for v1 (one info event keeps arms comparable and the machine small), but the bias is directional and
interacts with the headline mechanism: with no demand evaporation, `penalty_frac = 1` monotone
reservation is never punished, so the sim structurally *overstates* the value of reserving early and
understates one grounded source of replan value (revision of the existing book, not just arrival of
new bookings). Corrected text: add one row/caveat to §3 and an entry in §11 or the v2-deferred list —
"no booking revision/cancellation event (contradicts grounding F1d/F8 deliberately); biases toward
early reservation; v2 = lead-time-rising attrition hazard, same cluster as `penalty_frac < 1` (a
cancelled shipment stranding a reserved envelope is exactly when fractional release matters)."

**F6 (major) — the arrival process is under-specified to generate.**
"Time-inhomogeneous Poisson, 120/wk, weekly periodicity (weekday weights)" is missing: (a) the weekday
weight *values* (no source exists — the grounding's curves are in time-to-departure space, a different
axis; tag them [A] or drop to homogeneous); (b) whether 120/wk is the normalized mean of the weighted
intensity; (c) **within-day intensity** — uniform over 24h vs business-hours matters materially here,
because express slack is measured in hours and phase against the midnight planning run and −6h cutoffs
(F2) is decided by booking hour; (d) tier, lane, weight, density assignment: presumably i.i.d. marks
independent of arrival time and of each other — state it (the grounded "express books latest, deferred
earliest" is already carried by gap(tier), so independence is defensible, but say so); (e) the actual
`q^O, q^D` lane shares and `s_ℓ` values are "per S50" — not in the doc, violating its own
self-containment charter. Corrected rule: pin all five in §3 as tagged assumptions.

**F7 (minor) — the tier-ladder gate is only structural under a counterfactual the doc doesn't state.**
"Fix a shipment, vary only its tier ⇒ eligible set weakly grows" fails if varying the tier re-draws
gap(tier): ready_at shifts later, so the eligible window *shifts* (early flights drop out) rather than
grows. It is a theorem only when book_at, the gap draw, and ready_at are held fixed and *only*
sla_offset varies (12→24→96 monotone, base_transit tier-agnostic ⇒ deadline monotone ⇒ weak growth).
Corrected rule: define the ladder counterfactual as "hold ready_at fixed; vary sla_offset only."

**F8 (note) — commit rule and arm comparability: mostly well-handled, one thing to state.**
Yes, M1 can defer firmness by hopping to later flights (each hop resets "the flight it is standing
on") up to the backstop, and M1p cannot — but that *is* the estimand, not a confound: the backstop is
exogenous and arm-invariant, cadence is identical, and deferral is priced by the per-arm ratchet (each
hop ratchets reservations on both flights, never released). The S54 grounding's symmetry fix (both
arms reserve; only assignment fluidity differs) is correctly carried into §0. What the doc should
state: the commit rule *binds only M1* — H0/M0/M1p freeze by arm policy long before any cutoff, and
PIH trivially satisfies it — so "commit" is an M1-only constraint, which is fine but should be said.
The remaining well-posedness holes are F3, not comparability. One consistency check that passes:
backstop < ready + ground_out is impossible by construction (any deadline-feasible path's cutoff ≥
ready + ground_out), *provided* the path set is non-empty (F3c).

**Summary of corrected constants.**
Cool-down: 27.7d ≈ 28d ✓ as stated, **plus T_max** (new pinned constant) once tardy arcs are bounded.
Warm-up: ≈ 26.7d ≈ 27d ✓ as stated; also serves visibility-horizon and ratchet warm-state. Generated
span: **76d** (state it; drop the unexplained "~70" lower bound). Implied max base_transit ≈ 112h is
consistent with the bookends but must be *stated*, with the per-lane construction and the wait
allowance W made explicit — W is what decides F2.

---
---

## Agent 3 — Physical realism (no-code, sourced)

**Arithmetic base used throughout:** E[CW] ≈ 505 kg (E[w]=500 × E[max(1,167/ρ)]≈1.009), so book ≈
120/wk × 0.505 t ≈ **61 t/wk ≈ 3.2 kt/yr**. Chargeable per LD3 position ≈ 900 kg (doc §2). All
BSA-magnitude reasoning below is **reasoning from take-or-pay economics, not citation** — the doc's
own [NF] tag is correct; I found no public BSA-size source either.

**1. BSA shape (§4) — NEEDS CHANGE. This is the biggest implausibility, and it is also an internal
contradiction, not just a realism gap.**
Three of the design's own numbers are mutually unsatisfiable:
- 1 dep/day/lane × 9 lanes = 63 contract flights/wk; `n_f = floor(ν + U′)` with ν ∈ [1,3] gives **at
  minimum (ν=1) exactly 1 position/flight = 63 positions/wk ≈ 57 t chargeable = 91% of the book**. At
  ν=2: 126 positions ≈ 113 t = **1.8× the book**.
- The stated target `contracted_share ≈ 0.55` needs ~34 t ≈ 38 positions/wk ⇒ ν ≈ 0.6 — **below the
  [1,3] band and below the ≥1-position structural floor. The 55% target is mathematically unreachable
  under the stated shape.**
- Take-or-pay burn at ν=2 (reasoning, using the project's own r_c=4.2 $/kg, pivot 1200 kg, hard_frac
  0.35 from `air_capacity_parameters.csv`): ~44 hard positions/wk × 1200 × 4.2 ≈ **$220k/wk of
  take-or-pay against a total book freight spend of roughly $280k/wk**. Since BSA is 1.8× the book,
  ≥45% of BSA is dead even at perfect capture, and the origin-dominant lane mix guarantees thin lanes
  hold daily hard positions against <1 t/day of demand. No forwarder survives a quarter of that; real
  practice is to contract the base load *below* forecast and spot the peak — which is exactly what the
  55% target encodes. The 55% target itself is well-supported: Xeneta reports forwarders buying
  "nearly half of their volumes in the spot market" (Xeneta, Jan 2025); the academic allotment
  literature uses 1/3–2/3 splits (Moussawi-Haidar 2014, already in the grounding doc).

**My number: keep the 55% target, fix the shape to it — contracts only on the 3–4 dominant lanes (per
the q_ij mix), ~3–4 contract departures/lane-week, ~35–40 positions/wk total, zero BSA on thin lanes.**
That is also the commercially recognizable shape: a 3 kt/yr forwarder holds a handful of lane-specific
blocks and co-loads the rest; it does not hold a 9-lane daily allotment grid.

**2. Tightness story (§5) — NEEDS CHANGE: τ_v is never defined, and nominally the band is
unreachable.**
Doc's "market τ ≈ 1.11" = 1/0.90, so τ = supply/demand coverage. At defaults (ν=2, μ=2.5): supply ≈
113 t BSA + 73 accessed flights × ~2.5 positions ≈ 164 t spot ≈ **277 t vs a 62 t book → τ_v ≈ 4.5**,
and even the minimum corner (ν=1, μ=1) gives ≈ 123/62 ≈ 2.0 — the [1.0, 1.3] band is unreachable on
nominal capacity. It becomes reachable only if τ is computed on the **decayed-at-book_at,
eligibility-restricted** catalog (spot at typical standard-tier leads retains ~25–40% of nominal), and
only near the ν=1/μ=1 corner — which then forces contracted share to ~91%, re-triggering finding 1.
The doc's own §7 rule ("all capacity gates read the DECAYED catalog") suggests this is the intent, but
M3 never says so. **Fix: define τ_v explicitly (decayed, tier-eligible, per-flight-reachable), and
re-derive the default cell after the BSA cut.** Separately: 90% is verified but is described by the
source as a **near-practical-maximum peak**, not a standing condition (IndexBox/Xeneta); market
average dynamic load factor is ~57–63% (Xeneta). Simulating permanent peak is a legitimate deliberate
regime choice for this proof — but the doc should label it "peak-quarter transpac," not "the market."

**3. Forwarder scale (120/wk ≈ 3.2 kt/yr) — DEFENSIBLE, with the caveat that it's an SME, not
"mid-market" in the league-table sense.**
Top-50 air forwarders bottom out around ~100 kt/yr globally, with "mid-level" defined as 100–500 kt
(Transport Topics 2024; top-50 list). 3.2 kt on one trade (plausibly 20–40% of its air book) puts this
player far outside the top 100 — a regional/SME consolidator. That is a perfectly coherent simulated
desk: ~9 t/day, ~1.9 shipments/day/lane, top lane running a real daily consol, and its spot access
(~66 t/wk at μ=1) is only ~4% of the lane group's free-sale space at 90% LF (~26 legs/day ≈ 2,000+
t/day, 10% free) — one small buyer among hundreds, fine. The doc is right that 40/wk is not a
consolidation desk. The incoherence is only with the 9-lane daily BSA grid (finding 1), not with the
scale itself.

**4. Timing (§3) — DEFENSIBLE.** The leads-before-READY convention reconciles with the
leads-before-DEPARTURE anchor once dwell is added: standard = 1–5 d gap + ~1–2 d (ground ~10 h + daily
cadence + flight wait) ≈ **2–7 d pre-departure** vs the C.H. Robinson 5–7 d ex-Asia anchor — mean
slightly short of the anchor, but the anchor is peak advice and the tier ordering (express latest,
deferred earliest) matches it. Express 0–24 h works mechanically: minimum book→wheels-up = gap + ~10 h
ground + 6 h cutoff ≈ 16–40 h, and at ~2–3 legs/day/lane the shipment catches a day+1 flight —
consistent with real NFO/urgent behavior. The McKinsey waypoints also check out: mass booked >14 d
pre-departure ≈ 10–13% (deferred tail only) < 40% ceiling; express+standard ≈ 75% inside the final
week > 50%.

**5. Demand shape (§3) — DEFENSIBLE.** The lognormal math is internally exact (mean/median = 2 ⇒ σ =
√(2 ln 2) = 1.177 ✓); truncation clips ~0.6% at 5 t, ~3.6% at 30 kg — harmless. A per-HAWB weight
anchor is genuinely **NOT FOUND** (WorldACD holds shipment-count data but publishes no public
average); 250/500 with occasional 2–5 t HAWBs is a recognizable consol book, honestly tagged [A]. Two
minor notes: density U(150,250) means only ~17% of shipments bill volumetric (ρ<167) — real
general-cargo books skew lighter (garments/e-commerce ~100–150 kg/m³), so consider a lower bound of
~120, which only strengthens the volume-binds story; tier mix 20/55/25 is [NF] and 20% express is on
the high side for a consolidator, but it's a labeled knob.

**6. Single biggest remaining implausibility after the BSA fix:** the world is **permanently peak** —
90% market LF, tight-regime pricing, and every accessed flight guaranteeing this SME a full free LD3
(the ≥1-position floor), sustained over the whole 70–76 d horizon with no seasonality. Acceptable as a
declared stress regime for the L2 proof; not acceptable if headline savings numbers get quoted as
"the market" — the doc should say so in one line. (Minor internal nit, flagged in passing: M1's
tier-ladder monotonicity only holds if `ready_at` is held fixed when varying tier — as written,
`gap(tier)` re-draws ready, which can break weak growth of the eligible set.)

**Summary table**

| # | Magnitude | Verdict |
|---|---|---|
| 1 | BSA: 1 dep/day/lane × 9 × ν∈[1,3] | **NEEDS CHANGE** — 0.9–1.8× the book; 55% target unreachable; my number: 3–4 dep/wk on dominant lanes only, ~35–40 positions/wk |
| 2 | τ_v ∈ [1.0,1.3] default + "market τ=1.11" | **NEEDS CHANGE** — τ undefined; nominal τ ≈ 2.0–4.5; define decayed/eligible τ and re-derive after BSA cut; label 90% LF as peak regime |
| 3 | 120 shpt/wk ≈ 3.2 kt/yr, spot on 40% of 26 legs/day | **DEFENSIBLE** (SME/regional; coherent once BSA is cut) |
| 4 | Booking leads before READY (E 0–24h / S 24–120h / D 120–456h) | **DEFENSIBLE** — reconciles with 5–7 d pre-departure anchor and McKinsey waypoints |
| 5 | Lognormal 250/500 [30,5000], ρ~U(150,250), 20/55/25 | **DEFENSIBLE** — weight anchor honestly NOT FOUND; minor: density lower bound ~120; tier mix [NF] |

Sources: Xeneta Jan 2025 (spot share ~half, DLF 62%) · IndexBox/Xeneta transpac 90% = near-practical
max · Transport Topics 2024 airfreight top-50 · top-50 air forwarders 2024 · C.H. Robinson
booking-lead advice · McKinsey booking curve · WorldACD market data (no public shipment-weight
figure) · project grounding: `docs/design/air_cargo_demand_arrival_grounding.md`,
`air_capacity_parameters.csv`.

---
---

## Agent 4 — Gates red-team (no-code)

### (a) Verdicts on the two known nonsense instances

**One-ULD fleet (every flight = exactly 1 LD3): excluded — but not by M5. M5, the gate purpose-built
for this instance, passes it.**
- **M5 arithmetic:** median shipment CW ≈ 255 kg (lognormal median 250 kg; ρ ~ U(150,250) has median
  200 > 167, so the CW multiplier's median is 1.0). One LD3 = 1500 kg / 4.5 m³. kg basis: 1500/255 ≈
  **5.9 ≥ 3 → PASS**. m³ basis: median v ≈ 1.27 m³; 4.5/1.27 ≈ **3.5 ≥ 3 → PASS**. Chargeable basis
  (~900 kg/position per §2): 900/255 ≈ **3.5 ≥ 3 → PASS**. M5 fails to fire under every unit
  interpretation. The median is blind to the tail, and K=3 was calibrated (implicitly) against the
  *mean* 500 kg shipment, not the median 250 kg one. The doc's own rationale text for M5 ("passes
  every other check") is arithmetically wrong as of this draft.
- **What actually kills it: M1 + M2 + M4, via the demand tail.** A shipment exceeds one LD3 by volume
  iff w > 4.5ρ; at ρ=200 that's w > 900 kg, P ≈ 13.8% (Z = ln(3.6)/1.177 = 1.09). Over ~1,260
  shipments (17.1/day × 74d), ~175 shipments have **zero eligible itineraries** → M1 fails, M2
  deficiency > 0, PIH fallback > 0 → M4 fails.
- **Verdict: excluded, three times over — accidentally.** The exclusion rides entirely on the demand
  truncation at 5,000 kg and the fat lognormal tail, not on the anti-one-ULD invariant. If demand were
  ever re-pinned lighter (e.g., the 40/wk small-desk variant with a tighter cap), the one-ULD fleet
  walks back in through a passing M5.

**τ=12 world: excluded by M3 for the default cell — and unreachable inside the pinned sweep.**
M3: 12 ∉ [1.0, 1.3] → default cell fails ✓. Reachability: at μ=4, supply ≈ 63 contract flights ×
E[n_f] + 67 accessed flights × 4 spot positions ≈ 365 positions ≈ 1,640 m³/wk vs demand ≈ 295 m³/wk →
τ_v ≈ 5.6 max. τ=12 needs knobs (φ, ν, book size) the sweep doesn't expose ✓. All other gates pass it
trivially. **M3 is the single point of exclusion**, and "sweep cells may exit the band deliberately"
means an oversupplied cell can still legally exist in the sweep — acceptable only if the headline is
pinned to the default cell (it isn't; see holes).

### The buried lede: the doc's own default world fails its own gates

**M3 is unsatisfiable for every μ ∈ [1,4].** Position floors: contract = 1 dep/day/lane × 9 lanes ×
n_f ≥ 1 → ≥ 63 positions/wk; spot = φ·(24 trunk legs/day)·7 ≈ 67 accessed flights × m_f ≥ 1 → ≥ 67
positions/wk. Floor supply ≈ 130 positions ≈ 585 m³/wk. Demand = 120/wk × mean v ≈ 2.45 m³ ≈ 295
m³/wk. **τ_v(μ=1) ≈ 2.0** — the sweep's *minimum* is already 1.5× above the band ceiling. The band
[1.0,1.3] needs ≈ 74–85 positions/wk; the contract floor alone is 63 and spot access adds 67 more. No
μ reaches it.

**The BSA 55% target is unsatisfiable too.** 63 contract positions × ~900 kg chargeable ≈ 56.7k kg vs
book CW ≈ 57.7k kg/wk → contracted share ≈ **98%**, not 55%, at the ν floor. §4's two sentences (1
dep/day/lane; sized to 55%) contradict each other at 120/wk.

**M1 is unsatisfiable at any μ in the sweep.** Max positions on any flight at μ=4: n_f ≤ 3 plus m_f =
floor(4+U_f) = 4 → **7 positions = 31.5 m³**. Demand truncation allows w=5,000 kg at ρ=150 → v = 33.3
m³ → needs **8 positions**. P(v > 31.5 m³) ≈ 0.05% → ~0.7 shipments per horizon → **roughly half of
all seeds contain a born-dead shipment even at μ=4**. At μ=1 the max flight is 2 positions (9 m³ /
3,000 kg); P(shipment > 2 positions) ≈ 2–3% → ~30 per horizon → **every μ=1 seed fails M1 with
probability ≈ 1**. The "born dead = generator bug" hard gate declares the pinned generator itself a
bug.

**Constructive note:** demand ≈ 240/wk repairs both simultaneously — τ_v(μ≈1.3) ≈ 1.1 (in band), BSA
share ≈ 50% (near target) — and truncating weight at ~2 positions' worth (w ≤ 3,000 kg AND v ≤ 9 m³,
i.e. w ≤ 9ρ) repairs M1 at low μ. The 120/wk pin in Q1 is the odd constant out.

### (b) New nonsense instance: the **Monday-batch world**

**Construction (fully inside the doc's generator family):** the weekday weights of the
time-inhomogeneous Poisson are **[A] Q1, unpinned — user's call**. Set them to (1,0,0,0,0,0,0): all
~120 weekly bookings land Monday. Everything else at (repaired) defaults with τ_v in band.

**Gate walk:** M1 ✓ (schedule at full breadth; eligibility never reads the arrival process). M2 = 0 ✓
(same weekly totals, same edges; the LP is time-blind beyond eligibility windows). M3 ✓ (τ is a
ratio; batching doesn't move it). M4 ✓ (PIH is clairvoyant; batching is irrelevant; C(arm) ≥ C(PIH)
holds by theorem). M5 ✓ (capacity and shipment-size medians untouched). M6 ✓ (fallback trend vs
book-day is trivially flat with one book-day per week). R1–R7 ✓ (persistence/accounting/feasibility;
none touch arrival staggering).

**Why it is scientifically useless:** Monday's planning run reveals the entire week's book to every
arm at once. M1p solves it jointly once; on every subsequent run M1 has **no new information**, and
the decay curve (floored at its own reservation) means the option set can only *shrink* between runs —
so re-solving cannot beat the first solve. **L2 = C(M1p) − C(M1) ≈ 0 by construction of the arrival
process**, with capacity honestly tight, τ in band, and every gate green. This is the τ=12 world's
ghost: not "nothing binds," but "nothing is ever *revealed between runs*."

**The property that makes it possible — and it is accidental, not stated:** every gate in §7 is a
**static property of the world** (capacity geometry, feasibility, tightness ratios, size ratios). The
estimand L2 is a **dynamic information property** — how much book arrives between planning runs
relative to how long shipments stay uncommitted. The doc *knows* this failure mode: §5 rejects book
size as the sweep dial precisely because "at ~1/cycle M0 ≡ M1p byte-identical, measured; the dial
would modulate whether the estimand exists." That knowledge became a dial-selection argument, never a
gate. Nothing in §7 measures information churn. (The S51 diagnosis — book-lead 1.9d vs. decay window —
was exactly this class of bug, and it would ship again.)

**Missing gate (proposal):** an information-churn invariant, e.g. median over shipments of (# planning
runs strictly between `book_at` and final commit) ≥ 2, **and** median fraction of the movable book
that is not a newcomer at each run ≥ some floor. Deterministic per instance given the arm-free
timeline (`book_at`, `commit_backstop`, cadence) — hard-gateable in `load()`.

**Bonus milder variant, no adversary needed:** the express tier (gap U(0,24)h, offset 12h) likely
commits at its *first* planning run — commit_backstop lands before run 2. For that tier M1 ≡ M1p per
shipment structurally, diluting L2. Same unguarded axis; worth a per-tier churn report.

**Related circularity, worse than the hole:** §6's "gate: CI_lower > 0" on L2. If that is a validity
gate, then any world where replanning genuinely has no value is rejected as *invalid* — the experiment
cannot ever conclude "L2 ≈ 0," which §11 Q5 explicitly names as a live possibility (reservation
stickiness → L2 → 0). Gating on the desired sign of the headline makes the headline unfalsifiable. It
must be a *result*, never a gate.

### (c) Ambiguity list — what a builder must guess from the doc alone

**M1:** (1) path definition absent — no max path length, no min connect time at HKG, no rule on
multi-trunk or destination-side connections; R5's "missed connection" inherits the same undefined
min-connect. (2) `base_transit(lane)` used but never defined — deadlines, M1(iii), M2,
commit_backstop all uncomputable. (3) decayed-at-`book_at` with whose r_f? Presumably r = 0 — guess.
(4) the tier ladder is not structural as written — "vary only its tier" changes ready_at; eligible
sets shift, they don't nest; counterexample given; must be re-specified or it hard-fails honest
instances immediately. (5) which shipments — window-only or all generated? Unstated.
**M2:** (6) "bipartite shipments × flights" contradicts "edges = M1 eligibility" which is over paths —
a 2-leg itinerary consumes two legs jointly; either feeders are uncapacitated (unstated) or M2 must be
a path-flow LP; as written a feeder bottleneck hides behind deficiency = 0. (7) kg + m³ rows with what
coupling? One x_{s,f} with two rows, or two independent flows (which understate deficiency)?
Deficiency unit unstated. (8) which capacity number on a flight's row — different shipments see
different decayed values of the same flight; a row needs one number; genuinely ill-defined.
**M3:** (9) τ_w, τ_v never defined — numerator (nominal vs decayed; accessed vs all), denominator (raw
vs net-of-BSA), window, aggregation all guesses; under total/total the pinned default is ≈2.0, under
spot/residual ≈25. Aggregate-only τ admits a lane-concentration nonsense variant; per-lane τ should at
least be reported.
**M4:** (10) PIH at which gap (scoring 0, presumably — say so); what happens to a failing seed —
regenerate? That conditions the population on clairvoyant feasibility; legitimate only if the
rejection rate is reported and capped; neither is specified.
**M5:** (11) median over which flights? All ~26/day — 60% hold nothing — makes the median 0 and the
gate always-fail; must be accessed-only; unstated. (12) which unit for "flight cap" — kg, m³, or
chargeable? The ratio spans 3.5–5.9 on the same instance; and per (a) K=3 doesn't exclude the
motivating instance under any of them.
**M6:** (13) trend test unspecified (Mann-Kendall? OLS? α?) — tolerable only because report-only.
**R-gates:** (14) R2 has no operational check — needs Σ offer capacities per flight ≤ flight capacity.
(15) R3 references "per current design" — an external document; the 48h-cliff and pooling semantics
can't be built from this doc. (16) warm-up "computed from drawn fields" — the flight-visibility
horizon is not a drawn field and is defined nowhere. (17) scoring semantics: does the scorer re-solve
(re-optimize?) or price a fixed assignment at gap 0? If it re-optimizes, it isn't scoring the arm's
policy, and R4 attaches to the wrong solve. (18) rate-card magnitudes (family rates, MAWB fees,
weight-breaks) are pinned nowhere; C(arm) is not computable as specified.

### (d) Hard/report split — corrections

Deterministic per instance (safe to hard-gate): M1 existence (once paths + base_transit are pinned),
M2 (once the LP is pinned), M4 both parts (given scoring gap 0 — `C(arm) ≥ C(PIH)` is the one true
theorem and the best gate in the doc), M5, M3 monotone-in-μ, R1, R3–R7 as runtime raises. R2 is not
data-checkable as written.
Corrections: (1) the M1 tier ladder as specified is *false*, not flaky — re-specify at fixed ready_at.
(2) M1 existence + M4 as silent regenerate = selection bias — count redraws; fail the cell if
rejection > 10%. (3) M3's band is a calibration precondition, currently unsatisfiable — as a hard gate
it guarantees an undocumented renegotiation of φ/ν/demand at build time; pin the recalibration
procedure in the doc. (4) §6's L2 CI_lower > 0 must be demoted from gate to result. (5) M6 report-only
is correct. (6) Add the missing hard gate: information churn.

### (e) Gate placement — remaining holes to a headline number

(1) The load() gauntlet can be bypassed in-process — require that scoring reads only
persisted-then-loaded instances. (2) Nothing validates the decay computation at runtime — the decayed
catalog is computed during replay, the same seam where `ready = 0` shipped; add a per-run raise 0 ≤
r_f ≤ avail(f,t) ≤ C_f per dimension. (3) M4 placement unstated and not load()-time — it needs a full
PIH solve; if it runs once at instance creation, a later MILP change silently breaks it with zero
instance diff; M4 must re-run per experiment. (4) instance_card contents unspecified — the
lane-concentration variant diffs clean unless per-lane τ is on the card; pin the schema. (5) the
aggregation seam is ungated — which cell yields the headline is never pinned; pre-register headline =
default cell, all cells reported. (6) boundary cost-shifting across arms — an arm that defers boundary
shipments past the window end exports their cost while their service stays in; report per-arm carried
chargeable weight on window flights.

**Bottom line.** The two historical nonsense instances are excluded, but the exclusion map is not the
intended one: the one-ULD fleet passes M5 and dies only on the demand tail; τ=12 dies only on M3.
Worse, the pinned default world itself fails M1 and M3 — the constants and the gates are mutually
inconsistent and will force undocumented recalibration at build time. And the gate set has a
structural blind spot: every gate is static, the estimand is dynamic — the Monday-batch world passes
everything while L2 = 0 by construction, a failure mode the doc itself names in §5 and then leaves
unguarded. Add an information-churn gate, fix the tier-ladder definition, demote the L2 CI to a
result, pin τ's definition and the instance-card schema, and re-derive the default constants.

---
---

## Agent 5 — OR / experiment-design expert (no-code)

**Verdict.** The skeleton is sound: exogenous supply, flight-keyed cost / shipment-keyed service,
CRN-paired μ sweep with pathwise monotonicity, PIH as bound-only. But two claims the design leans on
hardest are wrong as stated — the "warm-up suffices EXACTLY" claim (F1) and the implicit adequacy of
the cost-keying for non-flight costs (F2) — and the inference plan (F3) treats 36 crossed cells as 36
replicates. Q5 should not ship as penalty_frac = 1 alone (F4).

**F1 (blocker) — The finite-memory warm-up argument is false as stated; memory is compositional, not
max().**
The per-flight state audit itself passes: reservation ratchet, sunk envelope cost, decay params die at
departure; commit_backstop, retirement, priors are per-shipment and die within residence; R3 confines
the only genuinely pooled cost (hard-BSA take-or-pay) to a lane-week. There is no hidden pooled
*state variable*. But finite state lifetime ≠ finite memory. A window-start flight departing day 27
accumulates ratchet state over its visibility lifetime [27−H_vis, 27]. The planning runs in that
interval depend on the book then, whose steady-state composition includes shipments up to
max(deadline−book_at) ≈ 28 d old. With generation starting at day 0, the run at day 27−H_vis ≈ 11 is
missing the 17–28-day-old cohort. Those missing shipments cannot ride the day-27 flight directly, but
they displace: their absence loosens competition on flights departing [11, 27], letting present
shipments take earlier/cheaper slots, which under-ratchets the early window flights. So the correct
first-order bound is **warm-up ≥ H_vis + max(deadline − book_at) ≈ 16 + 28 ≈ 44 d**, not max(16, 27)
= 27 d — and even that is first-order only (the displacement chain telescopes at −2H_vis−2·28,
negligibly). The bias is one-sided (looser early-window world) and plausibly arm-asymmetric — M1
exploits slack differentially — so it lands directly on L2 at the leading bookend, which §9 declares
"the science." *Correction:* set warm-up = H_vis + max residence (~44 d; horizon ≈ 93 d, ~20% more
sim), or keep 27 d but demote "EXACTLY" to "first-order" and add an empirical stationarity gate:
cost-rate and reservation-depth trend over flights in the last week of warm-up vs. window weeks (M6's
flight-keyed cousin), per arm.

**F2 (blocker) — Cost keying is unspecified exactly where arms differ: fallback and tardiness
penalties have no flight.**
§6 asserts "every cost term lands on exactly one flight/MAWB," but (i) a fallen-back shipment rides no
flight — if the fallback penalty is excluded from flight-keyed C, an arm is *paid* to strand (its
window cost rate drops); if included, the keying is mixed and undefined; (ii) §0 puts the tardiness
penalty in the solve objective, §6 files tardiness under service — it is never stated whether C(arm)
in L1/L2 includes it. If C is freight-only, L2's gate CI_lower > 0 is uninterpretable without a
service non-inferiority companion, since an arm can buy cost with tardiness. (iii) Hard-BSA
take-or-pay is a lane-week cost, contradicting the one-flight claim — though as a use-independent
constant it cancels in L1/L2 and only shifts levels. *Correction:* define C(arm) = flight-keyed
freight + reservation sunk cost over window flights, **plus** book_at-keyed tardiness and fallback
penalties over window shipments, plus lane-week-keyed BSA deficiency over window weeks. All three keys
are exogenous and arm-invariant. Add a scorer assertion that every cost term carries exactly one key,
and align the window to lane-week contract boundaries (21 d = 3 whole weeks already permits this —
state it).

**F3 (high) — 6×6 crossed seeds are not 36 replicates.**
Cells sharing a supply (or demand) seed are correlated; a naive paired-t over 36 differences
understates variance whenever main effects dominate — which the design's own variance-decomposition
step anticipates. Also missing: a pilot to size σ(L2) (with Q5 live, L2 may be small; 6×6 could be
badly underpowered, making a failed gate ambiguous between "L2 = 0" and "no power"); multiplicity
handling across the μ grid (per-cell CI_lower > 0 gates inflate FWER if the claim is "L2 > 0
somewhere"); and the CI/M6 interaction (a detected bookend leak invalidates the CI — M6 is logically
prior to inference, not a parallel report). *Correction:* CI from a two-way random-effects
decomposition, Var(L̄2) = σ²_S/6 + σ²_D/6 + σ²_SD/36 with Satterthwaite df (or, conservatively, t over
the 6 supply-seed means). Pin the control-variate estimator (β̂ regression on C(PIH), one df spent).
Designate the default cell as the single confirmatory gate; the sweep is estimation (curve +
simultaneous band), not 13 hypothesis tests. Run a 2–3-seed pilot at the default cell to size the
detectable effect and pre-register a minimum effect of interest. Sequence M6 before the CI.

**F4 (high) — Q5: do not ship penalty_frac = 1 alone.**
Incentive structure: under never-release + full sunk peak pricing, incumbency is free at the margin
while moving costs 100% of the old envelope plus the new one. Only the arm that moves things pays this
— the ratchet is a directional tax priced onto exactly the treatment (M1), while H0/M0/M1p never
trigger it. Worse, the tax is time-inconsistent by construction: step 4 ratchets *after* the solve, so
the MILP never prices the externality its own placement creates on future runs — M1's option value is
confiscated by a mechanism invisible to it at decision time. And per your own S54 grounding, reality
is *free change/cancel before cutoff* with a fractional 25–50% no-show only on space held-and-unfilled
at cutoff (AA, Lufthansa, IAG primary sources) — so penalty_frac = 1 with a horizon-peak envelope is
not a conservative simplification of the grounded world; it is a counterfactual friction well above
it, with a known one-sided bias on the estimand. A null L2 under it is unidentifiable (no value vs.
confiscated value), so "test at 1, fix if it bites" buys only one interpretable outcome and forces the
rerun in the other. *Correction:* since penalty_frac < 1 is already specced (S54 v2), run the default
cell as a bracketing pair {1, ~0.35} with 1 retained as the robustness lower bound and the grounded
value as primary; gate the sweep on the primary. Minimal intermediate if release machinery truly must
wait: price the envelope at its value *at cutoff* (identity-lock), not the horizon peak — that already
encodes "free pre-cutoff, pay for held-and-unfilled at cutoff" with zero new mechanism.

**F5 (medium) — M2 is inconsistent with the path structure and has no time convention.**
M1 eligibility is explicitly "paths, not legs," yet M2 is bipartite shipments × flights: multi-leg
paths (TPE→HKG, PVG→HKG feeders) consume two legs jointly, which a bipartite flow cannot express — it
can certify deficiency = 0 on instances that are dynamically infeasible at the feeder. Also,
"decayed-at-book_at" capacity is per-shipment-timestamped: two shipments see different decayed values
of the same flight, so no single capacity vector exists for one LP. *Correction:* LP feasibility over
eligible *path* variables with per-leg kg + m³ rows; pin a conservative snapshot convention (each
flight at the latest book_at among its eligible shipments); state explicitly that M2 is a
necessary-condition screen, not a feasibility certificate.

**F6 (medium) — Gate/sweep conflicts and the ≥1-position floor at the low end.**
With μ_ℓ = μ·s_ℓ and narrow lanes s_ℓ < 1, μ_ℓ < 1 occurs inside the sweep range, activating the ≥1
clamp: the exact mean identity E[floor(x+U)] = x breaks, the realized accessed-flight mean exceeds μ,
and the low end is attenuated lane-selectively. Separately: M1's "≥1 eligible itinerary else generator
bug" and M4's "PIH fallback = 0" can *legitimately* fail in deliberate scarcity cells (a 5 t shipment
needs ~6 positions), yet only M3 carries a sweep-cell exemption. And M5's global medians hide narrow
lanes reduced to 1-position flights, where consolidation is structurally impossible and L2 ≡ 0 — the
exact one-ULD pathology M5 exists to catch, reintroduced lane-locally. *Correction:* apply M1/M4 as
hard gates on the default cell only; in sweep cells, report born-dead and PIH-strand as censoring
outcomes. Report per cell: floored-flight fraction, realized mean positions per lane, eligibility by
size decile. Make M5 per-lane. Otherwise the sweep mechanics are good — pathwise monotone survives
both s_ℓ scaling and the clamp, continuous U_f makes floor ties a.s. impossible, and ~800 jump points
per unit μ make the pathwise staircase effectively smooth at any sensible grid.

**F7 (low-medium) — Boundary-discretion mass is large relative to the 21 d window, and §9's shrink
clause is dangerous.**
Deferred shipments have flight-choice spans up to ~2 weeks; with two boundaries, the share of window
shipments with cross-boundary discretion is O(2·E[span]/21) — substantial for the 25% deferred mix.
Under stationarity this is variance, not bias (in/out flows balance in expectation; flight membership
itself is exogenous, so there is no endogenous-membership bias in the rate), but finite-sample
imbalance can be material at 3 weeks. *Correction:* add a per-arm flow-balance diagnostic — net
chargeable weight booked-in-window-but-flown-out minus booked-out-but-flown-in, expected ≈ 0 — and
report cost per window flight *and* per window CW. Amend §9: the window may shrink only to whole weeks
and never below max flight-choice span (~2 weeks).

**F8 (low) — Minor internal inconsistencies.**
(a) Truncating LogN(median 250, σ = 1.18) at 5,000 kg removes ~0.5% of shipments but ~8–9% of expected
weight: realized mean ≈ 460, not 500; specify truncation method and state that τ targets use the
realized mean. (b) "Warm-up computed from drawn fields" makes horizon geometry seed-dependent; use
distributional maxima. (c) H0's "cheapest standalone route" is ill-defined under joint MAWB/ULD costs
— the doc claims self-containment but neither defines nor cites the H0 policy. (d) §9's tractability
envelope was measured on the defective world; the rewrite deliberately tightens τ_v, which typically
hardens the MILPs — treat as a planning estimate, not a bound. (e) M4's `C(arm) ≥ C(PIH)` gate
false-fails if PIH's incumbent sits above optimum; the ≤1e-6 scoring gap covers this — note the
direction. (f) Design choices justified by measurements on the defective code ("M0 ≡ M1p at ~1/cycle,
measured") should be re-verified on the new world before being load-bearing.

**What is right and should be kept:** exogenous supply with n_hawbs nowhere in the capacity path;
flight-keyed cost rate over an exogenous, arm-invariant flight set (no endogenous-membership bias);
book_at-keyed paired service set; E[floor(μ+U)] = μ giving exact mean-linearity of the supply dial;
CRN freezing with byte-identical demand; planning gap 0.005 as policy vs. scoring gap ~0 as
measurement; rejecting flight-count and book-size as tightness dials for the stated reasons;
whole-week window under weekly periodicity; and routing out-of-window shipments so window flights
carry full load.

---
---

## Agent 6 — Completeness miner (code as inventory; forbidden from correctness opinions)

Inventory complete. I read the design doc and all 13 source files in full (~9,200 lines). Below is the
coverage table — every distinct mechanism/rule/parameter-family the current simulator contains,
classified against the design doc. SILENTLY OMITTED rows first.

### SILENTLY OMITTED (design neither specifies nor mentions; builder forced to invent a decision)

| # | Mechanism (code source) | Decision the builder must invent |
|---|---|---|
| O1 | Spot rate system: family assignment per arc (flat_rate / min_flat_breaks / coload_per_kg), spot base $5.5 × regime multiplier U(0.85,1.18), contract rate $4.2, min_chg U(80,150), 4-break MFB ladders (`air_generator.py`) | What each accessed flight's price is and which billing family it rates under — §0 says "the family rate" but no family assignment or level is given |
| O2 | MilpParams scalars: MAWB fixed fee $50 (with the >0 no-phantom invariant), dunnage ε=0.05, C.10 PWL α-grid (0,.25,.5,.75,1), 600s wall-clock backstop (`air_milp.py`) | Values for every objective scalar |
| O3 | Hard backstop T^abs (`backstop_buffer_h`=264h): the fallback arrival time, the C.10 PWL span (T^abs−Δ), the predicate-7 horizon prune (`air_graph.py`, `air_generator.py`) | When a fallen-back shipment "arrives" (for tardiness/OTP) and what time bound prunes far-future flights — the design's only backstop is `commit_backstop`, a commitment time, not an arrival |
| O4 | Fallback-arc cost sizing: longest upper-bound path × margin 1.5, `air_leg_cost_ub` worst-family bound, trivial $1 when no real path (`air_graph.FallbackPolicy`) | How "a large penalty" (§0) is quantified so fallback dominates every real route without wrecking the MIP gap |
| O5 | Cost-category decomposition + D5 separation: freight_spot/contracted/coload/ground/mawb_fix/surcharge vs fallback_penalty vs tardiness_penalty; real_cost basis of L1/L2 (`air_milp._cost_breakdown`, `replay.RunScore`) | Whether fallback and tardiness penalties are inside C(arm) for the §6 cost-rate estimand, or reported separately |
| O6 | Scoring state machinery: 3-state OTP (on_time_real / late_real / fallback), fallback 3-cause split (structural / capacity_roll / disruption via `_has_real_path`), tardiness mean/median/p95 (`replay.score_run`) | What per-shipment outcome states and diagnostics the scorer emits |
| O7 | `booking_promise` freeze: score against the Δ_k frozen at commit, never the live row (`replay.py`, `scenario_db.py`) | Whether a frozen-promise table exists (possibly moot with an exogenous deadline — still a call) |
| O8 | Cargo-class heterogeneity: mix GEN/PER/DGR/VAL/HUM = .60/.20/.10/.05/.05, temperature, lithium (30% of DGR), consolidation group g(k)=(class,temp) with VAL/HUM/AVI singletons, capability predicates 2–5 (`air_generator.py`, `air_graph.py`) | Whether the new demand carries cargo classes at all — this silently determines the whole MAWB-group partition and who can consolidate with whom |
| O9 | prep_time 2h (CO* = cutoff − prep) and dispatch lead λ_disp 4h (origin-POL predicate 6) (`air_graph.py`) | Whether cutoff admission subtracts prep/dispatch leads — §7 M1(ii) uses `ready + ground_out ≤ cutoff` instead, leaving the leads undefined |
| O10 | Born-dead prevention at generation: `latest_ready` clamp of the reveal time (F2b) (`air_graph.latest_ready`, `_gen_arrivals`) | What the generator does when a drawn shipment has no eligible itinerary — §7 M1 declares it "a generator bug" but gives no mechanism (clamp / redraw / reject) |
| O11 | Biting shipper deadline t_dead: `t_dead_prob`, offset U(112, 240)h floor/cap, Δ = min(t_dead, T_SLA) (`air_generator.py`, `flexibility.committed_deadline`) | Whether within-tier deadline heterogeneity survives — not in the §3 timeline and not in §10's deletion list |
| O12 | base_transit(lane) values: 84/96/112h by dest metro, keyed on nominal gateway (`air_generator._DEST_BASE_TRANSIT_H`) | The magnitudes (or derivation rule) of the lane capability estimate the §3 deadline formula multiplies through every SLA |
| O13 | Ground-chain scalars & costs: per-gateway cartage h/$, CFS dwell h/$ (6–8h, $75–98), customs 12h/$200, drayage 60 km/h and $0.8/km (`tpeb_air_instance._gateways`, `air_graph.py`) | Every in-chain hour and every ground dollar the objective bills — §3 pins only the out-chain 7–31h transit shape |
| O14 | Hub transit dwell: CFS-H deconsol P5 (HKG 6h/$260, re-grouping allowed) vs tech-stop connection P6 (ANC 2h/$0), which airports are CFS-H, per-HAWB dwell-arc dedup, mct_h (`air_graph.py`, `tpeb_air_instance.py`) | How a connection at HKG (implied by §3's feeders) physically works — dwell time, cost, and whether re-consolidation is allowed there |
| O15 | Offer/arc structure: one-offer-one-arc, multi-leg through offers, overlapping emission, interline, multi-stop single flight number, max_air_legs=3 hop budget (`air_graph.py`, `geo_select.py`) | How ~26 legs/day compose into bookable priced products and how many air legs a path may chain — §7 M1 says "paths, not flights" without bounding them |
| O16 | Geographic candidate selection: k-NN seeds (k=3, 1500 km, extend-until-reachable), corridor ellipse φ=1.3, `ground_group` landmass truck gate, per-HAWB allowed air pairs (`geo_select.py`, `freightnet.ground_group`) | How a door's coordinates map to candidate airports and admissible flights — §3 keeps FreightNet but not the selection rule |
| O17 | Transit-time realization: deterministic s=0 walk vs stochastic (sd_air 1.5h, ground 10%, customs 25%), pre-sampled frozen `leg_actuals`/`component_actuals` (CRN) (`air_transit_time.py`, `air_generator.presample_actuals`) | Whether realized transit equals schedule, and whether frozen-actuals tables exist — the design never states the realization model |
| O18 | Disruption injection + §6 recourse: delay/cancel events with `realized_at`, three-zone replan gate, flown-prefix unlock, pin release, re-tender, recourse tardiness re-stamp (`replay.Disruption`, `_refresh_active_shipments`, `_flown_prefix`) | Whether the disruption/recourse subsystem exists in the rebuild — R5 specifies only the miss physics, not injection or recovery |
| O19 | Frozen-arm infeasibility recourse: M1p dump-largest-cw-priors-to-fallback repair, and the fail-fast invariant behind it (`replay._repair_frozen_infeasible`) | What a pinned-but-INFEASIBLE M1p/M0 solve does (the reservation floor may or may not make this unreachable — needs a decision either way) |
| O20 | H0 capacity handling & billing: how "cheapest standalone route" respects finite capacity, `reroute_to_next_supply` contention roll, billing by one all-pinned MILP solve (`h0_planner.py`, `replay._plan_cycle_h0`) | The §0 one-liner leaves open what H0 does when its choices over-subscribe an arc and how its book is costed |
| O21 | Short-fuse arrivals vs the daily grid: the guarantee that every [reveal, cutoff] window contains a run (verified ≥25.3h on the OLD generator; H0 legacy ad-hoc inserts) (`replay._hourly_times`, `_daily_times`) | What happens to a shipment whose window straddles no daily run under the NEW demand (E-tier gap U(0,24)h) — fall through to fallback or ad-hoc book |
| O22 | Capacity audit ledger: integer ULD + float spot ledgers, declarative per-cycle reconcile, tendered-monotone rule, conservation CHECK / ε=1e-6 (`replay.ReplayState`, `scenario_db.py`) | Whether a conservation audit trail exists at all — §7's gate list does not include it |
| O23 | Output/telemetry schema: executions/runs/planning_runs, versioned `routes` (is_current, soft|firm) + `route_legs`, `CycleRecord`, det_id/config.json provenance (`scenario_db.py`, `replay.py`) | How runs, plan snapshots, and provenance are persisted and versioned |
| O24 | Flight departure-time-of-day pattern: the US-outbound bank (~dep 36–39h+offset), feeder timing that clears CFS-H dwell + cutoffs (`tpeb_air_instance.py`) | Clock-time placement of the 26 legs/day — it interacts directly with the daily planning hour x and the dep−6h cutoffs |
| O25 | CRN stream registry beyond supply: named streams for rates, decay jitter, book-gap, spot regime, region tightness, actuals; canonical sorted draw order (`scenario_db.RNG_STREAMS`) | Which non-supply draws live on which stream so sweeping one knob leaves the others byte-identical — §4 pins only U_f |
| O26 | cw_flex / flex_k reshuffle denominator: θ_flex=24h separation, non-dominance frontier, arm-invariant `flex_denominator` table (`flexibility.py`, `scenario_db.py`) | Whether the flexible-mass reporting denominator survives the rebuild |
| O27 | Surcharge machinery: Path-A per-arc (per_kg/per_shipment/per_cbm), Path-B per-ULD σ — supported by the MILP, populated to zero by the generator (`air_milp.py`) | Whether the new world carries surcharges (zero, drawn, or dropped) |
| O28 | z_tier reliability margin / predicate-9 tier filter (inert at s=0) (`flexibility.TierSpec`) | Keep-or-drop of the tier reliability margin, coupled to the O17 realization decision |

### SPECIFIED

| Mechanism | Doc § |
|---|---|
| Discrete-event replay: reveal, plan, ratchet, commit, retire | §0, R7 |
| Daily cadence at hour x ∈ {6,12,18,24}, identical for all arms | §0 |
| Five arms and what each may move | §0 |
| L1/L2 estimand, empirical per-draw (no nesting theorem except ≥ PIH) | §0 |
| Booking-curve decay: convex A(τ), A_cut ~ Beta by deck, λ ~ U(0.10,0.16)/day, cutoff anchor, spot-only | §0 |
| 2D spot reservation: envelope buy, element-wise-max ratchet, never released (penalty_frac=1), free at margin, sunk cost on peak reserved CW, identity mutable until cutoff | §0 (+§11 Q5) |
| Decay floored at the arm's committed reservation; BSA firm | §0, §4 |
| Spot capacity = integer LD3 positions × (1500 kg, 4.5 m³), per flight, 2D | §4, R2 |
| LD3 physics, cw = max(w, 167v), density U(150,250), volume binds | §2, §3 |
| Tardiness penalty in objective; per-tier weights re-measured | §0, §10 |
| MILP consolidation formulation (flow, MAWB linkage, C.4 density mixing, C.5b/C.13 BSA, MFB linearization + MFBlink cut, billing validation) | §10 "survives" (via tex) |
| Soft-BSA 48h release cliff; hard-BSA equalized take-or-pay per lane-week, never horizon-pooled | §4, R3 |
| BSA sizing: 1 dep/day/lane, n_f = floor(ν+U′), ν∈[1,3], ≈55% contracted target | §4, Q3/Q4 |
| Spot access φ=0.40; m_f = floor(μ_ℓ+U_f), U_f CRN-frozen, ≥1-position floor, s_ℓ lane scaling; supply never reads the book | §4 |
| Tightness dial: μ ∈ [1,4] sweep, τ_w/τ_v reported, default τ_v ∈ [1.0,1.3] | §5 |
| Demand: Poisson 120/wk + weekday weights, lognormal weight (250/500, [30,5000]), v=w/ρ, tier mix 20/55/25, lane mix q_ij=q^O·q^D | §3 |
| Timeline: book_at, ready = book+gap(tier), deadline = ready+base_transit+offset, endogenous cutoff dep−6h, commit_backstop over paths | §3 |
| Commit at the standing flight's cutoff, no later than commit_backstop | §0, §3 |
| Network: 3 origins / 3 dest metros / 9 lanes, feeders, ~26 legs/day, ~15% belly, weekly tiling; doors from gateway bounding boxes; out-chain 7–31h shape | §3 |
| Horizon ~70–76d, 21d scored window, warm-up/cool-down from drawn fields; out-of-window shipments routed but unscored | §6 |
| Cost keyed on flight departure date; service keyed on book_at | §6 |
| Gap policy 0.005 planning / 0 scoring (assert <1e-6) | §6 |
| Seeds 6×6 supply×demand, PIH control variate, paired-CI gate | §6 |
| Gates M1–M6; decayed-catalog reads; gates in load/persist/solve/score; instance_card.md | §7 |
| R1 byte-exact persistence; R4 scorer refuses non-OPTIMAL; R5 miss expressible; R6 no departed-cutoff flight | §8 |
| Solver determinism, threads=1 per solve, parallel cells | §9 |
| FreightNet topology survives | §10 |

### DELETED

| Mechanism | Doc § |
|---|---|
| d*/target_offer_id, tender_at, book-lead buckets/means/λ_compress, tier_coupled_arrival (the whole cutoff-anchored reveal machinery) | §3, §10 |
| Supply generator: κ/α Dirichlet-multinomial position spread, E[SE_k]/E[cw] demand-derived sizing (τ·n·E[cw]), spot_regime kg-cap draw U(1,3)×1500, per-lane S_ℓ=τ_ℓ·D_ℓ split, per-arc spot kg caps | §1, §4, §10 (D3/D4/D7/D8/D9) |
| Freighter repositioning ρ (reposition_rho residual-chasing redistribution) | §4/§10 (supply rewrite; no ρ term in m_f) |
| Belly spot-cap thinning 0.4× | superseded by §4 uniform positions (belly survives only via decay A_cut, §0) |
| 7-day demand window + `flight_horizon_days` opt-in split horizon | §6/§10 (horizon shape replaced) |
| n_hawbs as the instance-size knob | §3/§5 (120/wk; book-size dial explicitly rejected) |
| Legacy cadence split (H0 daily + short-fuse inserts vs machine per-event; `per_arrival`) | §0 (uniform daily cadence) |
| H0 batch supply-first BFD greedy-consolidate (fill-well gate 0.8, two passes) | §0 (replaced by cheapest-standalone — but see O20) |
| ~12 tests codifying old defects (`fallback_count>0`, `otp<1.0`, spot cap bands) | §10 |

**Key observation for the caller:** the omissions cluster in four zones — (1) **the dollar side**
(O1–O5, O12–O13: virtually no cost magnitude in the doc besides LD3 geometry), (2) **the
graph/physical layer between doors and flights** (O14–O16, O24: hubs, through-products, candidate
selection, clock times), (3) **the realization/recourse layer** (O17–O19, O3: what actually happens
after commit, including where a fallback shipment "arrives"), and (4) **demand heterogeneity**
(O8–O11: cargo classes, prep/dispatch, t_dead). Files read: `src/replay.py`, `src/cap_decay.py`,
`src/h0_planner.py`, `src/components/air_milp.py`, `src/components/air_graph.py`,
`src/components/air_transit_time.py`, `src/components/flexibility.py`, `src/components/geo_select.py`,
`src/freightnet.py`, `src/scenario_db.py`, `data/synthetic/air_generator.py`,
`data/synthetic/tpeb_air_instance.py`, `data/synthetic/scenario_io.py`.
