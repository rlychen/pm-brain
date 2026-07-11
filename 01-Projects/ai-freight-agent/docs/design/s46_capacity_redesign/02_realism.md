# S46 Capacity Redesign — Realism & Practicality Review

**Reviewer role:** Realism & Practicality. Stress-tests `01_architecture.md` against how transpacific
eastbound air freight actually works and whether the proposed instances represent a **mid-market**
($50–500M) forwarder's transpac book. **Design review only — no code touched.**

**Inputs:** `01_architecture.md`, `air_pricing_calibration_s45.md`, `air_spot_composition_research_s45.md`,
`air_milp_m4_bsa_schema_options.md`, `forwarder-operations-analysis/{02-network-ops,04-exceptions-replanning}.md`.
One new web check (belly/freighter split, Jun-2026) is cited inline.

Severity legend: **BLOCKING** (a realism error that invalidates a headline metric or the persona claim
if shipped as-is) · **SHOULD-FIX** (materially wrong but the apparatus survives a parameter change) ·
**MINOR** (note / over- or under-modeling, low metric impact).

---

## BLOCKING

### B1. The headline regime (τ<1 network-wide) mislabels *peak* as *normal*
**Problem.** The whole redesign sizes total supply `S = τ·D` and centers the C2 headline sweep at
`τ ∈ {0.7, 0.9, 1.1}`, with the design narrative treating τ<1 (network-wide capacity < demand) as
"the regime where the three metrics are meaningful" (§F, C2 rationale). That is **not the standing
state of a real forwarder**. A forwarder holds BSA *precisely so it is rarely structurally short* on
its committed lanes — the operations record shows roll/off-load is **episodic** (roll rates cited
~5–15% on *hot* lanes, cutoff- and overbooking-driven, not standing demand>supply;
`04-exceptions-replanning.md`), and BSA holders get **priority** while loose/spot freight is bumped
first (`04-exceptions-replanning.md`, Dimerco). Network-wide τ<1 is a **peak-season** condition
(Dec front-load, capacity crunch), not a representative week. Reporting cost-savings / OTP / fallback
headline numbers at τ=0.7 silently reports *peak* economics as the product's everyday value.

**Recommended change.** Re-label the ladder. Make the **normal standing regime τ ≈ 1.0–1.15** (slack-
to-balanced), where shortage arises *locally* from realization noise + a minority of short lanes
(the bucket mix), not from a network-wide deficit. Promote **τ ≤ 0.8 to an explicitly named PEAK /
crunch scenario**, run as a stress arm, not the headline. The three metrics should be reported with
the **balanced regime as the base case** and peak as a labeled sensitivity. (The mechanism work is
fine — this is a framing/headline fix, but if uncorrected every reported number means "peak.")

### B2. 0.70 contracted is a *large-forwarder, lane-aggregate* number applied to a *mid-market* book
**Problem.** The 0.70 contracted / 0.22 spot split is INFERRED from the SOURCED "HKG→US <20% spot"
Xeneta fact. But that figure is a **lane-aggregate procurement mix dominated by deep-BSA majors** —
it is not a mid-market forwarder's own book. A mid-market ($50–500M) forwarder has **shallower BSA
depth** and structurally leans **more on spot and co-load via neutral master loaders** (a mid-market
staple — `02-network-ops.md` A8: NAC/ECU co-load 15–30% cheaper per kg; A1: "use commitment first to
avoid min-charge waste"). Holding **70% of its own volume as committed BSA overstates mid-market
contract depth** — that's a top-5-forwarder posture. This is load-bearing: the contracted share is
exactly the knob that decides whether the contracted tier binds (the entire S45 fix).

**Recommended change.** Set the **mid-market default to ~0.50–0.60 contracted / 0.30–0.40 spot+co-load**,
keep 0.70 only as a "large-forwarder / deep-BSA" variant. Within contracted, the 50/50 hard/soft split
is also high on hard — mid-market forwarders are *risk-averse to take-or-pay* (hard-BSA under-use →
empty-ULD charges, `02-network-ops.md` A1); a **hard_bsa_frac ≈ 0.3–0.4** is more faithful. Treat both
as the persona-defining realism knobs (already flagged in H.3 — but the issue is persona fidelity, not
just a value).

---

## SHOULD-FIX

### S1. Belly ≈66% overstates belly for transpac eastbound
Global belly is **~55%** of international air-cargo capacity (IATA, Jan-2025), and **transpac eastbound
is freighter-leaning** — Asia→NA freighter share runs *above* the global average (long-haul + freighter
e-commerce surge), even as belly grows (~+16.8% YoY Dec-25). The design's **"belly ≈66%"** misreads the
calibration doc's ambiguous "belly adds ~66% of total capacity on top." **Fix:** set belly to
**~45–55%** with an explicit *freighter-majority* note for this lane, so the freighter/ULD/BSA tier is
not artificially thinned. (Source: [Air Cargo News](https://www.aircargonews.net/airlines/air-cargo-capacity-rises-but-airlines-shift-away-from-the-transpacific/1081169.article), [Statista belly/freighter share](https://www.statista.com/statistics/1170554/air-cargo-capacity-share-aircraft-type-worldwide/).)

### S2. The cheap (base) spot block contradicts the design's *own* sourced research
The block ladder puts **5,000 kg at base rate** (44% of the ~11.25k ceiling at 1.00×). But
`air_spot_composition_research_s45.md` Q2 concludes the **base block is THIN** for a mid-market
forwarder on a dense, ~80%-contract-locked lane — "a slice of a slice," realistic base-rate access
**~few-hundred to ~1,000 kg/lane-day**, and explicitly: *"if the cheap block were calibrated abundant,
Mechanism A would NOT bind and the savings signal would wash out."* A 5,000 kg base block is exactly
that abundant-cheap calibration the research warns against — it risks **re-creating the S45 inert
failure** (spot stays cheap-and-deep). **Fix:** narrow the base block to **~1,000–2,000 kg** and let
the marginal price rise faster (front-load the ladder toward the expensive blocks), OR model the
forwarder accessing only a fraction of the lane's base block.

### S3. Fixed contract<spot ($4.2<$5.5) is regime-blind; in the slack arm real spot dips *below* contract
$4.2 contract < $5.5 spot is right for **normal-to-firm** markets, but in a **slack** market spot falls
**below** contract (~0.85× contract in soft regimes — memory `reference_air_spot_contract_ratio`). The
design holds the ordering fixed across the τ sweep, so the slack arm (τ=1.1) is unrealistic — and worse,
in a genuinely soft market **riding spot over contract is the *correct* behavior** (and would reproduce
S45-style "contracted never used" *legitimately*). **Fix:** make the spot base **regime-dependent**
(lower in slack), so the contract↔spot crossover is a function of τ, not a hardcoded ordering. The
"contracted binds" story should hold in tight/firm and *invert* in slack — that's the realistic test.

### S4. HAWB density band 120–240 kg/cbm under-represents volumetric (low-density) cargo
Air cargo's defining economic feature is **low-density cargo billed on volumetric weight** (chargeable =
max(actual, vol·167)). A density band of **120–240 with most mass above the 167 break** means chargeable
≈ actual for most HAWBs, so the volumetric/density-mixing dimension (C.5b-v) **rarely binds** — yet
density-mixing is the heart of real air consolidation. **Fix:** widen density **down to ~60–100** at the
low end (e-commerce, apparel, foam-packed goods are routinely <120 kg/cbm), so a meaningful fraction of
HAWBs bill volumetric and ULD volume actually competes with weight in the build. Mode/heavy-tail is fine;
the *floor* is too high.

### S5. Contracted depth at C2 scale (~4 ULD/lane-week) is below a real mid-market BSA
At 120 HAWBs / 6 lanes / τ=0.7, contracted resolves to **~5.8k kg ≈ 4 LD3 positions per lane-week** —
below the sourced mid-market BSA depth (**~1–4 ULD per *flight* × several flights ≈ 10–20 ULD/lane-week**,
`air_pricing_calibration_s45.md`). The instance is **demand-light relative to real BSA depth**: absolute
lane capacities are a scaled-down toy, not a real lane. **Acceptable for a correctness proof**, but
**label it** and flag that `∂metrics/∂τ` measured here may not transfer to real-volume books (the scale
tension the design already acknowledges in §F — make it explicit in the C2 caveat).

---

## MINOR

- **M1. Deferred-air as a named 0.05 composition tier is mild over-modeling.** On transpac eastbound,
  deferred/economy is largely an *express-carrier* product, marginal for a BSA/spot consolidation book.
  Fold it into a *slow spot arc* (the design already says "no new constraint") rather than a distinct
  supply product + composition slot. Keeps the minimal-design ethos; still gives the OTP cost lever.
- **M2. Pivot `U(1000,1500)` top end is aggressive.** Pivot = ULD chargeable *capacity* (1,500) means you
  always pay for a full ULD however empty — punitive even for hard allotments. Cap nearer **1,000–1,200**.
- **M3. Single ULD type (LD3).** Real consolidation picks AKE/PMC/PAG by cargo profile
  (`02-network-ops.md` A2). Accepted simplification; note it.
- **M4. Off-load priority not modeled.** BSA cargo is protected; spot/loose freight rolls first
  (`04-exceptions-replanning.md`). If the OTP metric proves sensitive, roll **spot-tendered before
  contracted** on a short lane. Likely already approximated by the optimizer's assignment, so low priority.
- **M5. MCT / connection banks absent.** Fine for direct headhaul (HKG→LAX nonstop); flag if dest
  gateways are reached via an interior hub (e.g., ORD via a connect) — missed-connection is the single
  most common air operational pain (`04-exceptions-replanning.md`).
- **M6. Screening / DG segregation grouping omitted.** Already an accepted scope cut
  (`project_air_screening_decision`); note only that consolidation feasibility is *optimistic* vs reality
  (real builds are constrained by DG/screening/temp compatibility — `02-network-ops.md` P5).
- **M7. Charter defer to v2 — agree.** Well-reasoned. Fallback at 2.5× base is the correct
  feasible-but-expensive backstop; charter's distinct "cheaper-than-rolling-a-large-block" value is niche.

---

## What's realistic and well-grounded — DO NOT TOUCH

- **Rate *levels*.** Spot $5.5/kg (Xeneta NE-Asia→NA Apr-26 $5.54), contract ~$4.2 as a *firm-market*
  level, ~0.76× ratio — well-sourced and correctly anchored.
- **Chargeable-weight convention & ULD geometry.** LD3 1,500 kg / 4.5 cbm, 167 kg/cbm (IATA 1:6),
  chargeable = max(actual, vol·167) — SOURCED and correct.
- **Increasing-block convex-PWL spot curve + 1.15–1.25×/block step.** Faithful to airline bid-price RM
  (the composition research confirms the mechanism); the no-binary PWL is the right modeling choice.
- **Soft (per_flight pivot) vs hard (equalized take-or-pay `A_c` sunk) BSA.** Matches ops reality —
  hard-BSA under-use → empty-ULD charges; "use commitment first" (`02-network-ops.md` A1). Correct.
- **Fallback at 2.5× base + no-standalone-cost-pruning.** Correct relief-valve economics.
- **Supply-independence discipline** (analytic `E[cw_k]`, CRN gate, `spot_supply` stream) — methodologically
  sound; keep verbatim. The κ→per-lane `τ_ℓ` generalization (keep α as within-lane noise) is clean.
- **HAWB weight = triangular(50, 1200, 300), mean ~517 kg.** Reasonable for consolidated air; only the
  *density* floor (S4) needs widening, not the weight shape.
</content>
</invoke>
