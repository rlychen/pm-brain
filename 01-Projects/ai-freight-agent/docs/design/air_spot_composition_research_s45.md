# Air Spot Pricing Composition & Cheap-Spot Access — Research (S45)

**Purpose:** Two precise, sourced questions feeding the air-pricing model design (OPEN-2 composition rule;
OPEN-3 Mechanism A cheap-spot calibration). Builds on `air_pricing_calibration_s45.md` (transpac eastbound
TPE/PVG/HKG → LAX/ORD/SFO; spot ~$5.5/kg, contract ~$4.2/kg, finite increasing-block lane supply curve).
Does NOT re-derive levels. Primary directive: realism, sourced — not confirming a hypothesis.

**Confidence tags:** SOURCED (public URL cited), INFERRED (derived from sourced anchors + reasoning),
MRN (market research needed — no free public figure).

---

## Q1 — Does the per-shipment weight break coexist with lane scarcity in SPOT, and how do they compose?

### What the sourcing actually shows

**(a) Spot is quoted as a single all-in $/kg that already reflects the booking's weight band — the
weight break is *embedded in* the spot rate, not a separate stacked multiplier.** This is the load-bearing
finding. The Freightos Air Index (FAX) — the canonical transacted-spot benchmark — publishes "an indicative
**all-in spot rate** for general cargo priced in US$ per kilogram," and it publishes it **in three weight
breaks: 100–300 kg, 300–1,000 kg, and 1,000–3,000 kg**, sourced from *actual booked digital rates on
WebCargo*. So the real, transacted spot quote is itself weight-band-specific and all-in. The weight break
is not a separate line item layered onto a base; it is the structure of the spot quote itself. A 1,200 kg
HAWB and a 200 kg HAWB on the same lane-day get **different all-in $/kg** because they read off different
weight bands of the same market-clearing curve. SOURCED (FAX methodology).

**(b) The "heavier = cheaper per kg" effect lives in published GCR tariffs AND persists into spot.**
GCR/TACT tariffs (the published street/market rate) carry the classic descending weight-break ladder —
M / N / +45 / +100 / +250 / +300 / +500 / +1000 / +2000 / +5000 kg, rates per kg descending with weight
(e.g. a quoted 250 kg @ $2.35/kg vs 500 kg @ $2.10/kg). SOURCED. Crucially the *same descending-with-weight
shape survives into transacted spot*: FAX's three spot weight bands are ordered the same way (heavier band =
its own, generally lower, all-in rate). So the weight break is **not** purely a contract/tariff artifact —
it is present in spot, just *baked into* the all-in per-band quote rather than applied as an explicit
discount line. The spot/ad-hoc rate is "dynamic pricing… the best applicable rate based on capacity,
relationship and priorities," negotiated per shipment against available capacity — i.e. one number, already
weight-aware. SOURCED.

**(c) Lane scarcity is realized as airline revenue-management dynamic pricing, not a posted scarcity line.**
The increasing-marginal-with-fill behavior is real but it is implemented inside the carrier's RM engine, not
exposed to the forwarder as a separate "scarcity premium." Mechanism, from the literature and platform
descriptions:
- Capacity is sold in two tranches: **allotment/BSA** (guaranteed, contracted 6–12 months out, confirmed
  ~2 weeks before departure) and **free-sale spot** (whatever allotment holders don't confirm). SOURCED.
- Free-sale spot is priced by **bid-price / capacity-bucket revenue management**: capacity discretized into
  buckets (one academic example: breakpoints `[50, 100, 200, 500, 750, 1000]` kg for a 1000 kg cap), with
  bid prices set by nonlinear programming over remaining capacity and demand-to-come. As the lane fills and
  departure nears, the bid price the next kg must clear **rises**, and customers' reservation prices rise
  with urgency. SOURCED (mechanism), though the *named buckets/values* are illustrative from one paper, not
  a universal fare-class taxonomy — INFERRED that transpac carriers use bid-price RM of this family.
- Operationally the forwarder sees this as **dynamic re-quoting on the booking platform**: the live
  WebCargo/cargo.one quote for the same lane/weight moves day-to-day (and toward Latest Acceptance Time)
  with remaining capacity. There is no posted "scarcity ×" the forwarder multiplies — the platform just
  returns a higher all-in number when the lane is tight. SOURCED (platform/dynamic-pricing descriptions);
  the exact re-quote magnitude per-fill is MRN.

### Composition rule to model (the deliverable)

**Faithful rule: spot is a single market-clearing all-in $/kg, indexed by weight band and by lane-fill
state — NOT base × weight-break × scarcity stacked as three independent multipliers.**

Concretely, model the spot rate as a function `r_spot(lane, weight_band, fill_state)`:

```
r_spot = R_lane(t)              # lane base spot level (the $5.5/kg anchor), slow-moving
         × wb(weight_band)      # weight-band factor, ≤ 1, descending with HAWB chargeable weight
         × block(fill_state)    # increasing-block scarcity multiplier (1.0 → ~1.2 → ~1.44 → …)
```

This *looks* multiplicative, but the honest reading of the sourcing is:
1. `wb(weight_band)` and `block(fill_state)` are **orthogonal and both real** — they multiply, but they are
   not two separate quoted lines the forwarder negotiates; they are two axes of one all-in number. The model
   may compose them multiplicatively because they act on different objects (one booking's size vs. the
   lane's remaining space), and there is no double-counting.
2. The weight break is applied to the **base lane spot level**, and the block multiplier is applied for the
   lane's current fill — exactly as the prior calibration §4 already prescribes. This research **confirms**
   that prescription against transacted-spot evidence; it does not overturn it.
3. Do **not** add a separate, independently-quoted "GCR ladder line" on top of spot. In spot the ladder is
   *already inside* the all-in per-band rate. Adding it again would double-discount. If the model uses
   continuous weight (not 3 FAX bands), fit `wb()` as a smooth descending step function calibrated so the
   three FAX bands (100–300 / 300–1000 / 1000–3000 kg) line up.

**Net:** stack weight-band × scarcity (two orthogonal axes), single all-in clearing price — not three
independent posted multipliers. Faithful because the transacted benchmark (FAX) is itself an all-in,
weight-band-indexed spot number, and scarcity is an RM-driven movement of that same number.

---

## Q2 — How much CHEAP (base-rate) spot does a mid-market forwarder access on a lane-day vs its own demand?

### What the sourcing actually shows

**(a) On a dense transpac lane, the free-sale spot pool is a *minority* of capacity, and the cheap end of
it is thinner still.** Two layers:
- *Capacity split.* Whole-trade Asia-Pacific→NA widebody-freighter capacity ran ~60–67k t/week in 2025
  (belly adds ~66% of total capacity on top). SOURCED. But the **majority is pre-sold to allotment/BSA
  holders**; only the unconfirmed remainder becomes free-sale spot. SOURCED (two-tranche structure).
- *Lane-level spot share is low on dense lanes.* This is the sharpest number: forwarders procured **less
  than 20% of Hong Kong→US volume via the spot market**, versus **~80% on ex-Vietnam**. SOURCED (Xeneta).
  So on exactly the kind of dense transpac headhaul lane in our network, ~80%+ of volume moves on
  *contract/allotment*, and the spot pool a forwarder draws from is the residual <20% slice. (Global
  all-lane spot share is higher — ~43% Q1-2024 vs ~31% pre-pandemic, ~46% mid-2025, ~half in Q4-2025
  (SOURCED, Xeneta) — but that average is dragged up by thin lanes like Vietnam; the dense transpac lane is
  the low-spot-share end.)
- *Cheap-end thinness.* Within that residual spot pool, the *base* (cheapest-block) portion is the early/
  abundant release; as the lane fills, the RM bid price climbs (Q1c). So "base-rate spot" is a slice of a
  slice: the cheap block of the <20%-of-volume free-sale pool. INFERRED from the two SOURCED facts above.

**(b) A mid-market (non-top-5) forwarder realistically secures *low-thousands* of kg of base-rate spot per
lane-day before climbing the curve — and likely less on a dense, allotment-locked transpac lane.** No public
figure resolves a *named* mid-market forwarder's per-lane-day base-spot access (MRN — commercial/internal).
INFERRED order of magnitude, reasoned:
- A dense transpac origin→gateway lane runs on the order of a few daily widebody departures; per the prior
  calibration the *total* uncommitted spot ceiling is ~10–12k chargeable-kg/lane-week, i.e. **~1.5–2k kg/day**
  of total free-sale spot, of which only the first (cheap) block is base-rate.
- A mid-market forwarder is one of *many* buyers competing for that residual pool and lacks the BSA depth and
  RM-relationship leverage of a top-5 forwarder. Its realistic *base-rate* grab is a fraction of the daily
  free-sale pool — **order of a few hundred to ~1,000 kg/lane-day at base**, after which it is reading higher
  RM bid-price blocks or rolling. INFERRED.
- It does **not** find "lots of base-rate room" on a dense transpac lane precisely because that lane is
  ~80%+ contract-locked (Q2a): the cheap free-sale space is structurally thin there. (On a thin lane like
  ex-Vietnam, base-rate spot would be *abundant* — but that is not our headhaul.)

**(c) Net: when ~500–1,500 kg of one forwarder's demand spills to spot on a dense transpac lane-day, it does
NOT all clear at base — it climbs the curve.** INFERRED, but well-anchored:
- ~500 kg spilling on a *slack* day (early in the booking horizon, lane not yet full) plausibly clears at or
  near base — it is within the first cheap block.
- ~1,000–1,500 kg, or *any* volume late in the horizon / on a firm-to-peak day, **exceeds the base block** of
  a forwarder's realistic cheap access and pushes into rising RM bid-price territory (the §4 1.15–1.25×-per-
  block steps), and at the tight extreme into the fallback tier. This is exactly the regime where tightening
  BSA forces realized cost up — the mechanism binds.
- The decisive realism point: on the dense transpac headhaul, cheap spot is **thin from a single mid-market
  forwarder's vantage**, so a 500–1,500 kg spill is a *meaningful fraction* of its accessible base-rate pool,
  not a rounding error against an abundant cheap supply. The savings/cost-tightness signal therefore *does*
  reach the number — Mechanism A can bind — provided the cheap block is calibrated thin (next section).

---

## What this means for the model

**OPEN-2 (composition rule).** Implement spot as a **single all-in market-clearing $/kg** that is the product
of (i) the slow-moving lane base level `R_lane(t)` (~$5.5/kg anchor), (ii) a **weight-band factor `wb()`
≤ 1, descending with HAWB chargeable weight**, calibrated to align with the three FAX bands
(100–300 / 300–1000 / 1000–3000 kg), and (iii) the **increasing-block scarcity multiplier `block(fill_state)`**
from calibration §4. Weight-band and scarcity are **orthogonal axes of one number**, composed multiplicatively
— NOT three independently-posted lines. Do not also add a separate explicit GCR discount on top of spot:
in transacted spot the weight break is already embedded in the per-band all-in rate, so an extra line would
double-count. (Published GCR tariff *is* a separate explicit ladder, but our pricing object is spot, not the
posted tariff.)

**OPEN-3 Mechanism A (cheap-spot calibration — thin vs abundant).** Calibrate the **cheap (base-rate) block
THIN relative to a single mid-market forwarder's lane-day demand** on the dense transpac headhaul. Justified:
that lane is ~80%+ contract/allotment-locked (HKG→US <20% spot, SOURCED), so the free-sale spot pool is a
minority of capacity and the *base* block of it is thinner still. Concrete handle: size the first (base-rate)
spot block a mid-market forwarder can access at **order of a few hundred to ~1,000 chargeable-kg/lane-day**,
so that a typical 500–1,500 kg BSA-spillover **exceeds the base block on firm/late days and climbs the
1.15–1.25×-per-block curve** (and into fallback at the tight extreme). This is what makes "tighten BSA →
realized cost rises" actually bite. If the cheap block were calibrated abundant (thin-lane behavior),
Mechanism A would NOT bind and the savings signal would wash out — so abundant is the *wrong* default for
this headhaul. Expose the base-block width as a κ-tightness knob (per §4) so thin-lane scenarios can be
modeled separately without changing the transpac default.

---

## Sourced / Inferred / MRN ledger

**SOURCED (public URL):**
- FAX spot rate is "all-in," published in three weight breaks (100–300 / 300–1000 / 1000–3000 kg), from
  actual WebCargo bookings — Freightos.
- GCR/TACT descending weight-break ladder (M/N/+45/+100/+250/+300/+500/+1000/+2000/+5000 kg; per-kg
  descending; e.g. 250 kg vs 500 kg example) — IATA / aviation-professional / vskills.
- Spot/ad-hoc = "dynamic pricing… best applicable rate based on capacity, relationship, priorities,"
  negotiated per shipment — IATA / WebCargo.
- Two-tranche capacity (allotment confirmed ~2 wks pre-departure; remainder → free-sale spot); BSA rate
  more favorable than non-BSA; BSA min-chargeable-weight / pay-or-fly — ScienceDirect / Cornell / Amaruchkul.
- Bid-price / capacity-bucket RM with NLP over remaining capacity; reservation prices rise with urgency;
  illustrative bucket breakpoints — arXiv 2404.04831 / 2405.11000.
- Lane-level spot share: HKG→US **<20%** spot, ex-Vietnam **~80%** — Xeneta.
- Global spot share **~43% Q1-2024 (vs ~31% pre-pandemic), ~46% mid-2025, ~half Q4-2025**, EU→NA 45% Aug-22
  vs 32% Oct-19 — Xeneta.
- Asia-Pac→NA widebody-freighter capacity ~60–67k t/wk (peak 75k); belly ~66% of total capacity — Air Cargo
  News.

**INFERRED (from sourced anchors + reasoning, labeled in text):**
- That weight-band × scarcity compose multiplicatively as two orthogonal axes of one all-in number (no
  double-count) — from FAX all-in-per-band + RM dynamic-pricing facts.
- Transpac carriers use bid-price RM of the family the literature describes (specific bucket values are
  illustrative, not a universal taxonomy).
- Total free-sale spot ceiling ~1.5–2k kg/lane-day (from §4's ~10–12k/wk); mid-market base-rate access
  ~few-hundred–1,000 kg/lane-day; 500–1,500 kg spill climbs the curve on firm/late days.

**MRN (no free public figure):**
- A specific mid-market forwarder's per-lane-day base-rate spot allocation on these exact lanes
  (commercial/internal; confidentiality bars rate cards).
- The exact spot re-quote magnitude per unit of lane-fill (RM curve slope) — carrier-proprietary.
- Lane-resolved (origin→specific US gateway) free-sale spot kg/day and its cheap-block fraction —
  paid/internal data.

---

## Sources

- Freightos Air Index (FAX) — all-in spot, three weight breaks (100–300/300–1000/1000–3000 kg), from booked
  WebCargo rates: https://www.freightos.com/enterprise/terminal/fax-global-freightos-air-index/
- IATA — Air Cargo Tariffs and Rules (GCR/TACT; spot/ad-hoc = dynamic pricing):
  https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/air-cargo-tariffs-and-rules-what-you-need-to-know/
- Aviation Professional — GCR calculation & descending weight breaks (250 vs 500 kg example):
  https://www.aviation-professional.net/2024/01/Calculate-a-General-air-Cargo-Rate-GCR.html
- vskills — classification of air freight rate (M/N/Q, weight breakpoints):
  https://www.vskills.in/certification/tutorial/classification-of-air-freight-rate/
- cargo.one — rate-sheet vs spot, weight breaks / time-to-LAT disparity:
  https://www.cargo.one/mastering-air-freight-rates
- WebCargo — dynamic pricing / real-time rate distribution: https://www.webcargo.co/dynamic-pricing/
- IBS — dynamic pricing in air cargo (free-sale segmentation, evolves to departure):
  https://blog.ibsplc.com/airline-cargo/dynamic-pricing-air-cargo-s-next-frontier
- Air-cargo capacity allocation / two-tranche allotment-vs-spot — ScienceDirect:
  https://www.sciencedirect.com/science/article/abs/pii/S1366554510000797
- Cornell (Huseyin) — Cargo Capacity Management with Allotments and Spot Market Demand:
  https://people.orie.cornell.edu/huseyin/publications/allotments.pdf
- arXiv 2404.04831 — Dynamic Pricing for Air Cargo Revenue Management (bid price, buckets):
  https://arxiv.org/abs/2404.04831
- arXiv 2405.11000 — Data-Driven Revenue Management for Air Cargo: https://arxiv.org/pdf/2405.11000
- Xeneta — air cargo contracts / lane-level spot share (HKG→US <20%, ex-Vietnam ~80%):
  https://www.xeneta.com/blog/air-cargo-contracts-how-long-should-they-last
- Xeneta — spot share ~43% Q1-2024 vs ~31% pre-pandemic / Q4-2025 ~half:
  https://www.xeneta.com/blog/what-the-air-freight-market-looks-like-right-now-and-where-its-heading ;
  https://www.supplychaindive.com/news/air-cargo-contract-behavior-shifts-rates-ecommerce-xeneta/809284/
- Air Cargo News — transpac widebody freighter capacity ~60–67k t/wk; belly ~66%:
  https://www.aircargonews.net/data-news/transpacific-widebody-freighter-capacity-settles-after-market-volatility/1080234.article ;
  https://www.aircargonews.net/airlines/air-cargo-capacity-rises-but-airlines-shift-away-from-the-transpacific/1081169.article
