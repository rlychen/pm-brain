# Air-cargo `<demand, capacity, price>` vs time-to-departure — grounded reference

Compiled 2026-07-09 (demand side) and 2026-07-10 (supply side) from seven parallel
research sweeps. Purpose: ground the joint **`<demand arrival, supply capacity
available, supply price>` term structure as a function of time-to-departure** (D−14,
D−7, … D−1) that a peak-season spot pre-buy decision depends on. Sourcing discipline:
every quantitative claim carries a source tag and URL; where a number could not be
verified it is marked **NOT FOUND** rather than filled in. See
`feedback_no_unverified_stats`, `feedback_no_fabricated_mechanisms`.

**Scope link.** This feeds the peak-season change to the air model (do we pre-book
some spot early to protect against overflowing fixed BSA). It does **not** revisit
the routing engine's core; the pre-buy is a hedge layered on top. Companion:
`docs/design/air_cargo_reserve_assign_grounding_s52.md`,
`docs/references-air-cargo-two-stage-allotment.md`.

Source-tag key: **[empirical-real]** measured from real data · **[empirical-industry]**
consulting/vendor data, not peer-reviewed · **[academic-model]** a modeling
assumption, not measured · **[theory-result]** proven analytical result · **[proxy]**
from an adjacent domain · **[trade-press]**.

---

## 0. TL;DR — the three collapse into one driver

The `<demand, capacity, price>` tuple is **not three independent time-series.** It is
one stochastic driver (the booking arrival process) with capacity and price induced
from it:

1. **Demand arrival = the booking curve** — cumulative fraction booked vs D−x
   (§1, §3).
2. **Capacity remaining = total capacity − cumulative bookings** — the *same* curve,
   inverted. Nothing separate to calibrate except the **endpoint** (final load factor)
   and the fine intra-horizon path (proprietary → interpolate) (§4).
3. **Price = bid-price(remaining capacity) + WTP markup** — proven to rise
   monotonically as remaining capacity depletes; **not** a function of calendar time
   (§5).

**Load-bearing result (regime dependence).** The realized price *path over time* is
the net of two proven, opposing forces: capacity depletion pushes opportunity cost
**up**; shrinking time-to-departure pushes it **down** (fewer future arrivals to
displace). Which wins depends on the demand-to-capacity ratio:
- **Tight / peak (high load factor):** close-in **premium** — dynamic per-flight
  repricing to fill + reservation of close-in space for high-yield urgent demand →
  **`p_e < p_l` holds** (industry-grounded).
- **Slack / off-peak:** the whole rate **level drops** (market-tightness effect), but
  **no reliable close-in markdown** — carriers hold yield / withhold capacity rather
  than dump. Theory's "slack → markdown" does **not** survive industry evidence
  (strategic buyers make predictable discounts self-defeating). No dependable gradient
  to exploit either way.

So `p_e < p_l` is **grounded in industry practice for the tight/peak regime** the
model targets — not merely a theory artifact. Off-peak the whole level is simply low,
so the hedge is moot. See §5c for the empirical basis (and why the theoretical
slack-markdown is wrong here).

---

## 1. Headline: what is actually citable about the booking curve

**The distribution's *shape* is well-established; its *exact numbers* are not.** The
literature is near-unanimous on structure and gives us one usable pair of empirical
waypoints; nobody publishes a full per-day histogram.

### 1a. Form (well-supported, consistent across sources)

Air-cargo bookings arrive as a **time-inhomogeneous point process over a short
horizon, with intensity rising toward departure and a heavy tail of very-late
(days-to-hours-before) bookings.** Implemented in the literature as either:
- a **non-homogeneous Poisson process** with intensity `λ(t)` a function of
  days-prior — Eren & Li (2024) [academic-model, data-informed]; Moussawi-Haidar
  (2014) [academic-model]; or
- its **discrete equivalent** — small periods, at most one request each, time-varying
  probability — Levin, Nediak & Topaloglu (2012) [academic-model].

The single most-reused arrival-timing input in the field is **Amaruchkul, Cooper &
Gupta (2007), Table 3** — a **six-interval** booking-request probability schedule
concentrating toward departure. Note this is a *numerical-example input*, not a
fitted empirical curve.

### 1b. The two empirical waypoints (the only hard "% booked by day X" found)

From McKinsey cargo-RM analysis of airline booking data [empirical-industry]:
- **< 40% of a flight's cargo capacity is booked at T−14 days** (two weeks out).
- **> 50% is booked in the final 7 days** before departure.
- Post-COVID (2021–22) the curve steepened (booked *faster/earlier* than 2019).

These two points imply a **heavily back-loaded curve**: roughly a third accumulates
over the first two weeks of the horizon, then the majority lands in the last week.

### 1c. Horizon length (spot arrivals)

Spot/free-sale booking requests are modeled as starting **~30 days before departure**
(Hellermann 2006, via Moussawi-Haidar 2014) — some models use a shorter **~10-day**
horizon (Eren & Li 2024). KLM operationally sold non-allotment capacity in roughly
the **last ~30 days** (Slager & Kapteijn 2004 [empirical-real], exact figure
unverified). **Allotment/BSA is a separate, far-earlier process** (contracts set
6–12 months out) — not part of this arrival stream.

### 1d. Late-tail softness (matters for the hedge)

- Bookings placed **7+ days out are 2–3× more likely to cancel** than last-minute
  ones; overall carrier acceptance ~97% (WebCargo/Freightos platform data
  [empirical-real]). Early "reserved" demand is *soft*.
- Cargo **show-up can exceed 100%** via over-tendering; airlines overbook (McKinsey
  [empirical-industry]; Kasilingam 1997). Booked ≠ tendered, in both directions.

### 1e. Cargo vs passenger

Cargo has a **shorter, later-skewed, more volatile/lumpy** booking curve than
passenger (Eren & Li 2024; Kasilingam 1997) — asserted as a defining characteristic,
**no quantified side-by-side comparison found**.

---

## 2. Consolidated evidence table

| # | Claim | Value / time ref | Tag | Source |
|---|---|---|---|---|
| F1 | Discrete inhomogeneous arrival, ≤1 request/period, time-varying prob | whole horizon | academic-model | Levin, Nediak & Topaloglu 2012, *Oper. Res.* 60(2) |
| F2 | Six-interval booking-request probability schedule (canonical reused input) | 6 intervals, concentrating late | academic-model | Amaruchkul, Cooper & Gupta 2007, *Transp. Sci.* 41(4), Table 3 |
| F3 | Spot requests Poisson, start ~30d out, tail to hours before; 1/3 allotment 2/3 spot | 30-day horizon | academic-model | Moussawi-Haidar 2014, *TR-E* 72 |
| F4 | NHPP with λ(t) a function of days-prior; short horizon (10-day baseline) | λ(days-prior) | academic-model, data-informed | Eren & Li 2024, arXiv:2405.11000 |
| F5 | Triangular intensity peaking ~2 days before departure | peak ≈ T−2 | academic-model | derivative of Amaruchkul-style inputs — **exact shape UNVERIFIED** |
| F6 | **<40% booked at T−14; >50% in final week; curve steepened post-COVID** | T−14 <40%, final-7d >50% | empirical-industry | McKinsey, "Ahead of the curve" (2023) |
| F7 | Free-sale sold in ~last 30 days; day-level forecast only ~2–3d out | last ~30d | empirical-real (split) / unverified (2–3d) | Slager & Kapteijn 2004, *JRPM* 3(1) |
| F8 | Show-up can exceed 100% (over-tendering); airlines overbook | at/near departure | empirical-industry | McKinsey; Kasilingam 1997, *EJOR* |
| F9 | Cargo curve shorter/later/lumpier than passenger | qualitative | academic/qual | Eren & Li 2024; Kasilingam 1997 |

---

## 3. Proposed calibration for the synthetic generator (NEEDS SIGN-OFF)

**Not yet committed — this is a modeling proposal.** The honest position: we can pin
the *form* and *two waypoints* to citable sources, but the fine shape is an
assumption. Recommended approach that maximizes citability:

**Use Amaruchkul (2007)'s six-interval structure as the form, calibrated to
McKinsey's two waypoints.** Concretely, over a 30-day spot horizon, choose interval
masses of cumulative booked fraction `F` hitting:
- `F(T−14) ≈ 0.35–0.40`
- `F(T−7) < 0.50` (so the final week carries > 0.50)
- `F(0) = 1.0`, with the last-week mass concentrated late (peak near T−2, per the
  unverified F5 — flagged as assumption).

This gives a defensible curve: **~38% over the first ~16 days, a modest week T−14→T−7,
then the majority in the final 7 days.** Both anchors are citable (form = Amaruchkul
2007; waypoints = McKinsey 2023); the intra-last-week allocation is a labeled
assumption, not a claim.

**Two knobs to expose, both grounded:**
- **Horizon length** (10 vs 30 days) — literature spans both; make it a parameter,
  default 30d (the more-cited spot horizon).
- **Late-tail softness** — early bookings 2–3× more cancel-prone (F1d). If the
  generator models bookings, apply an attrition/cancel rate rising with lead time;
  this is what makes early pre-committed volume genuinely uncertain (the φ risk).

**Unit of the curve:** demand-arrival per **region O-D pair** (routing-independent),
NOT per lane — lane load is endogenous to the router. Aggregate substitutable BSA
into a capacity pool before computing overflow.

---

## 4. Supply capacity available vs time-to-departure

**Key structural point: capacity-remaining is the booking curve inverted.** "Capacity
sold" *is* "cumulative bookings," so there is no independent depletion series to
calibrate. Two things *are* independent and must be set separately: the **endpoint**
(total capacity / final load factor) and the **fine intra-horizon path** (proprietary).

### 4a. Shape — the same two McKinsey anchors, inverted

- **~40% of capacity sold by D−14; the majority of the remainder loads inside the
  final week (D−7→D−0).** [empirical-industry] McKinsey 2023 (same source as F6).
  Strongly back-loaded, convex, last-week-dominated.
- Corroborated by the industry-standard **~30-day booking window** [academic-model]
  (Garg et al. 2024, arXiv:2407.20192, "typical cargo booking window of up to 30 days
  before departure"; KLM free-sale practice, Slager & Kapteijn 2004).

### 4b. Endpoint — final load factor at departure (the best-measured piece)

- Global **dynamic load factor ~55–63%** average (Xeneta/CLIVE, measured weekly):
  55% Jul-2023, 61% Mar-2024, **63% Nov-2024 (30-month high)**, 57% Jan-2025.
  [empirical-real]
- Lane-specific: **~90% on peak transpacific headhaul** (near practical ceiling);
  **<40% on weak backhauls** (directional imbalance). [empirical-industry] Xeneta/CLIVE.
- Caveat: "dynamic load factor" is a market-average utilization across all flights on
  a lane (weight-or-volume, whichever binds), **not** a per-flight time-to-departure
  curve.

### 4c. Effective availability is more back-loaded than gross

Early bookings cancel **2–3× more** than last-minute (F1d, WebCargo/Freightos
[empirical-real]), so the reliable-to-fly remaining capacity is lower than gross
bookings imply — the availability curve nets down for attrition.

### 4d. Close-out timing (advisory, not distributional)

- Peak Asia-outbound space effectively gone **~D−4/D−5**; forwarders advised to book
  **D−7 to D−10**. [empirical-industry] C.H. Robinson, StatTimes.
- Hard physical tender cutoff **~6h pre-departure** (varies by hub); "soft cutoffs"
  (space gone before published cutoff) in peak. [empirical-industry] AA Cargo, Sensio.

### 4e. Who measures the fine curve (all proprietary)

Rotate **Live Capacity** (per-flight slot utilization / "tonnage potential" from
Flightradar24 movement data) and Xeneta/CLIVE **dynamic load factor** (8 key lanes,
weekly) are the closest real remaining-capacity signals — both **paywalled**; neither
publishes a per-flight day-by-day depletion trajectory.

**Net:** shape (convex, last-week-dominated) and endpoints (D−14 ≈ 40% sold,
departure ≈ 55–90% full by lane/season) are data-backed; everything between D−14 and
D−1 is **interpolation, not measured** → interpolate a convex curve between the two
anchors, calibrate the endpoint to lane load factor.

---

## 5. Supply price vs time-to-departure

**Bottom line: no publicly citable spot-price-by-lead-time curve exists, but RM
theory pins the functional FORM firmly — and it is capacity-driven, not calendar-driven.**

### 5a. No measured price-by-lead-time curve exists (verified absence)

- Xeneta segments air rates by **weight break, surcharge, service level** only — no
  temporal/lead-time axis. Finest temporal cut is **contract-validity duration**
  (short <3mo vs long >3mo), *not* booking lead time for a given flight.
  [empirical-industry] https://help.xeneta.com/docs/rate-structure-and-methodology-air
- Freightos Air Index = **7-day rolling all-in spot** per kg, three weight breaks —
  no days-to-departure axis. [empirical-industry]
- WorldACD holds transaction microdata but **publishes no lead-time breakdown**;
  a lead-time cut is a custom/licensed pull. [empirical-industry]
- **Conclusion: the D−14…D−1 price curve for a fixed flight/lane does not exist in any
  free source.** Producing it needs transaction records with both booking timestamp
  and departure timestamp (WorldACD / carrier-or-aggregator booking logs — NDA-gated).

### 5b. RM theory — price = bid-price(remaining capacity) + markup, monotone in capacity [theory-result]

- **Opportunity cost (bid price) strictly increases as remaining capacity is
  consumed** — from concavity of the value function. Talluri & van Ryzin, Prop. 1
  (marginal value `ΔV_j(x)` decreasing in remaining capacity `x`).
  https://www.informs-sim.org/wsc09papers/013.pdf
- **Closed form (Gallego & van Ryzin 1994, Thm. 1):** optimal price `p*(n,t)` strictly
  decreasing in inventory `n`, and for exponential demand `p*(n,t) = ΔV(n,t) + 1/α`
  — **price = marginal value of a unit (bid price) + constant WTP markup.**
  https://business.columbia.edu/sites/default/files-efs/pubfiles/3943/vanryzin_optimal_dynamic_pricing.pdf
- So price is fundamentally a function of **remaining capacity**, which itself
  depletes per §4. Price is *induced* by the capacity curve, not an independent series.

### 5c. Temporal direction — empirically regime-dependent, but NOT a clean slack markdown

**Corrected against industry evidence — theory's slack-markdown does not survive.**
The tight-regime half holds empirically; the slack "markdown" does not. Leading with
what actually happens (industry), theory demoted to a footnote:

- **Tight / high load factor → close-in PREMIUM (grounded).** Dynamic-pricing engines
  reprice each flight's free-sale residual to its own booking curve and to
  proximity-to-departure (IBS Software [vendor-claim]; Lufthansa Cargo "Rapid Rate
  Response" via PROS [vendor/industry-reported]), so a filling flight reprices **up**
  into departure. Carriers also **reserve** close-in residual for structurally
  high-yield urgent demand — AOG, Next-Flight-Out, perishables, pharma — that pays a
  premium for guaranteed space. → `p_e < p_l`. [industry-reported]
- **Slack / low load factor → the whole rate LEVEL drops, but NO reliable close-in
  markdown.** Soft markets fall market-wide (documented **−40–50% YoY** back to 2019
  levels at ~56% load factor; *"$1 is better than flying empty"* — a major digital-
  forwarder exec [industry-reported, FreightWaves]) — but this is a depressed level for
  advance AND spot alike, **not** a dependable D−1 fire sale. Carriers exercise **yield
  discipline** (Etihad Cargo: *"we are not going to go to the bottom on yield"*
  [industry-reported]), accept **fewer** low-yield bookings close to departure, and
  cut / ground / redeploy capacity rather than dump it (Air Cargo Week; The Loadstar
  [industry-reported]). Close-in ad-hoc pricing is volatile in **both** directions
  (~±15%), not a one-way markdown.
- **Discipline erodes only in genuine oversupply** — macro yield dilution (Xeneta's van
  de Wouw: carriers *"chasing market share at the expense of price discipline"*
  [industry-reported]) — but that is a market-wide level drop, **not** per-flight
  close-in dumping.

**Modeling conclusion (grounded):** encode market tightness as a **κ / load-factor
multiplier on the WHOLE rate structure**, plus a **close-in premium that switches on
only in the tight regime**. Do **NOT** encode a deterministic close-in markdown on
slack flights — empirically unsupported and contradicted by capacity-discipline
evidence. For the pre-buy: `p_e < p_l` holds in the tight regime (the target); in
slack there is no dependable gradient to exploit, and the whole level is low anyway,
so overflow is cheap and the hedge is moot.

**Theory footnote (why the folk result fails here).** Myopic-buyer RM models
(Gallego & van Ryzin 1994; arXiv:2404.04831 — price non-increasing toward departure
under homogeneous demand absent rising WTP) predict a slack markdown. It fails in
practice because buyers are **strategic**: a reliable close-in discount trains
shippers to wait, so carriers hold the line. Consistent with the S51 verdict
(`reference_air_cargo_timing_and_pricing`): no reliable close-in markdown → book early.

### 5d. Cargo breaks the clean monotonicity [academic-model / theory-result]

Cargo's **batch / all-or-nothing acceptance** and **2D (weight × volume) capacity**
destroy the clean 1D bid-price monotonicity; Eren & Li (2024, arXiv:2405.11000)
explicitly note "monotonicity of bid prices in capacity and time dimensions [is lost]
due to batch arrivals and the all-or-nothing constraint," and estimate `b(x,t)`
with a neural net rather than assume shape (weight and volume bid prices estimated
separately and summed). So even the theoretical form is **approximate** for
indivisible ULDs.

### 5e. Directional magnitudes that are NOT lead-time curves (do not misuse)

- Express/priority tier premium ~**20–40%/kg** — a **service-tier** premium, not
  lead-time. [proxy, low-tier, unverified]
- Peak-season premium ~**30–50%** transpacific — **seasonal**, not lead-time.
  [proxy, low-tier, unverified]
- The "+50% peak spot premium" — **could not be verified; excluded** (see §8).

**Net for the model:** justified to encode price as `bid-price(remaining capacity) +
markup`, monotone increasing as capacity→0, **active only in the tight regime**; the
per-day price *number* stays a calibrated parameter (markup scale / bid-price
steepness), flagged as assumption and ideally swept. Do not hard-code a fixed % markup.

---

## 6. Peak-season overflow + spot term structure (surrounds the decision)

Grounds the premise "peak is when we overflow fixed BSA" and the `p_e < p_l`
assumption behind buying early.

- **Overflow premise holds.** Nov 2024 demand **+10% YoY vs capacity +2%**; dynamic
  load factor **63%** (30-month high) (Xeneta [empirical-real]). E-commerce pre-buys
  large BSA blocks (Shein ~5,000 t/day, Temu ~4,000 t/day [industry-reported]),
  pushing other shippers to spot/premium.
- **Spot term structure — direction only, no measured curve.** Peak surcharges,
  10+ day lead times, "short-notice booking no longer reliable" (C.H. Robinson
  [industry-reported]) all point to close-in spot being **scarcer and pricier** in
  peak. The one last-minute-discount mechanism (empty-flight distress pricing) by
  construction does **not** fire when flights are full → does not contradict the
  "no reliable close-in markdown in peak" prior.
- **NOT FOUND:** a measured close-in-vs-early spot **price-by-time-to-departure
  curve**. Providers publish spot *levels*, not a rate-by-lead-time term structure.
  A quantified `p_l/p_e` premium requires licensed booking-date-stamped data →
  **market-research needed**, do not fabricate.
- **Excluded:** the widely-repeated "+50% peak spot premium" — could not be verified
  on any primary page. Do not cite.
- **Spot:contract ratio** — no clean peak-vs-soft multiple published; the project's
  own `reference_air_spot_contract_ratio` range (~0.85 soft to ~1.18 peak) remains
  the better internal reference.

---

## 7. Penalty & decision-epoch structure (grounds φ and the trigger time)

Five real epochs before an air departure (carrier tariffs [primary-official]):
1. **Seasonal** — BSA/allotment contract fixes rate + committed/pivot weight
   (6–12 months out).
2. **Days out** — booking placed: "firm commitment" but cancelable with fees.
3. **~24–48h** — allotment **release** (unused space handed back, billed on actual
   usage) — an academic generalization, **NOT a universal rule; do not hard-code 48h**.
4. **3/2/1 business days (or "48 working hours before LAT")** — cancellation-fee
   **cliff**, escalating to 100%.
5. **LAT / cut-off** — physical tender/acceptance (RCS); failure = no-show.

**The unused-reservation penalty φ is instrument-dependent — the key modeling fork:**
- **Hard BSA** → pay full block up to committed/pivot weight even at zero use
  (φ = full rate on unused).
- **Soft BSA** → release penalty-free if you notify by the release date (φ ≈ 0).
- **Ad-hoc/spot no-show** → escalating slice of freight charge: 25→50→100% (CMA CGM);
  40–50% of AWB >300 kg (ITA Airways); flat $300 >250 kg (American). Small shipments
  (≤250–300 kg) frequently exempt. [all primary-official carrier tariffs]

**Signal for the model:** reserved capacity is cheap to hold early, expensive to
abandon late, with a sharp step inside the final ~48h. That is exactly the cost the
pre-buy hedge trades against.

---

## 8. Do-not-fabricate list (verified gaps)

- **No public per-day booking-lead histogram.** Only the two McKinsey waypoints exist
  publicly; finer granularity is proprietary (Rotate, cargo.one, WebCargo). The
  24/48/72h day-of-departure tonnage share = **NOT FOUND**.
- **No measured λ(t) curve from real carrier data.** All coded shapes trace to
  Amaruchkul 2007 Table 3 (an example input). The "triangular, peak T−2" shape is
  **unconfirmed** from any primary source.
- **No public per-day capacity-availability curve** (§4). The intra-horizon depletion
  path is proprietary (Rotate Live Capacity, Xeneta/CLIVE); only two anchors +
  endpoint are public.
- **No measured spot price-by-lead-time term structure** (§5, §6). Providers publish
  spot *levels* and contract-*duration* cuts, never rate-by-lead-time-for-a-flight. A
  quantified `p_l/p_e` premium requires licensed booking-date-stamped microdata →
  **market-research needed**, do not fabricate.
- **No "price rises close-in" law** (§5c). Temporal direction is regime/capacity-driven,
  not calendar-driven; a fixed % close-in markup is unjustified. Encode price via
  `bid-price(remaining capacity)`, active only in the tight regime.
- **No clean peak spot:contract ratio** (§6).
- **No universal allotment release window** — 48h is academic, not a rule (§7).
- **Per-carrier hard-BSA dead-space penalty rates are NDA** — only the soft/hard
  binary is public.
- **"+50% peak spot premium" is unverified** — exclude.

---

## 9. Full citations

**Academic / OR-RM**
- Levin, Y., Nediak, M., Topaloglu, H. (2012). Cargo Capacity Management with
  Allotments and Spot Market Demand. *Operations Research* 60(2):351–365.
  https://pubsonline.informs.org/doi/10.1287/opre.1110.1023 ·
  https://people.orie.cornell.edu/huseyin/publications/allotments.pdf
- Gallego, G., van Ryzin, G. (1994). Optimal Dynamic Pricing of Inventories with
  Stochastic Demand over Finite Horizons. *Management Science* 40(8):999–1020.
  https://business.columbia.edu/sites/default/files-efs/pubfiles/3943/vanryzin_optimal_dynamic_pricing.pdf
- Talluri, K., van Ryzin, G. Revenue Management: Models and Methods (Prop. 1, bid-price
  monotonicity). *Proc. 2009 Winter Simulation Conf.* https://www.informs-sim.org/wsc09papers/013.pdf
  — and *The Theory and Practice of Revenue Management* (2004), §2.5.
- Garg, ... et al. (2024). Time series forecasting with high stakes: A field study of
  the air cargo industry. arXiv:2407.20192. https://arxiv.org/abs/2407.20192
- Dynamic Pricing for Air Cargo Revenue Management (2024) — price non-increasing toward
  departure under homogeneous demand. arXiv:2404.04831. https://arxiv.org/html/2404.04831v1
- Amaruchkul, K., Cooper, W.L., Gupta, D. (2007). Single-Leg Air-Cargo Revenue
  Management. *Transportation Science* 41(4):457–469.
  https://pubsonline.informs.org/doi/10.1287/trsc.1070.0198
- Moussawi-Haidar, L. (2014). Optimal solution for a cargo revenue management problem
  with allotment and spot arrivals. *TR-E* 72:173–191.
  https://www.sciencedirect.com/science/article/abs/pii/S1366554514001732 · OA:
  https://scholarworks.aub.edu.lb/server/api/core/bitstreams/461be5e2-3cff-4aa7-b35a-06bf6c316cbb/content
- Eren, S., Li, ... (2024). Data-Driven Revenue Management for Air Cargo.
  arXiv:2405.11000. https://arxiv.org/abs/2405.11000
- Kasilingam, R.G. (1997). Air cargo revenue management: characteristics and
  complexities. *EJOR*. https://www.sciencedirect.com/science/article/abs/pii/0377221795003290
- Slager, B., Kapteijn, L. (2004). Implementation of cargo revenue management at KLM.
  *J. Revenue & Pricing Mgmt* 3(1):80–90.
  https://link.springer.com/article/10.1057/palgrave.rpm.5170096
- Amaruchkul, K., Lorchirachoonkul, V. (2011). Air-cargo capacity allocation for
  multiple freight forwarders. *TR-E* 47(1):30–40.
  https://ideas.repec.org/a/eee/transe/v47y2011i1p30-40.html

**Industry / market data**
- McKinsey (2023). Ahead of the curve: getting cargo revenue management right as the
  cycle turns. https://www.mckinsey.com/industries/logistics/our-insights/ahead-of-the-curve-getting-cargo-revenue-management-right-as-the-cycle-turns
- Xeneta (2024). 2024 is a "peak season to be proud of."
  https://www.xeneta.com/news/2024-is-a-peak-season-to-be-proud-of-as-air-cargo-industry-continues-to-manage-strong-demand-growth
- Xeneta (2024). Summer provides perfect warm-up act.
  https://www.xeneta.com/news/summer-provides-perfect-warm-up-act-as-the-global-air-cargo-market-heads-towards-hotly-anticipated-q4-peak
- Freightos/WebCargo (2025). Holiday air freight booking strategy.
  https://www.freightos.com/freight-resources/holiday-air-freight-booking-strategy/
- IBS Software (2021). Dynamic pricing – Air cargo's next frontier.
  https://blog.ibsplc.com/airline-cargo/dynamic-pricing-air-cargo-s-next-frontier
- CargoAi. Airfreight eBooking Journey. https://www.cargoai.co/blog/airfreight-ebooking-journey/
- Supply Chain Dive (2024). Air freight industry anticipates surging peak rates.
  https://www.supplychaindive.com/news/air-cargo-industry-spot-rates-peak-season-xeneta-june/720975/
- C.H. Robinson. Peak season air freight shipping tips.
  https://www.chrobinson.com/en-us/resources/blog/peak-season-air-freight-shipping-tips/

**Supply capacity & price data (industry)**
- Xeneta. Air rate structure & methodology (no lead-time axis).
  https://help.xeneta.com/docs/rate-structure-and-methodology-air
- Xeneta / CLIVE. Capacity & load-factor trends (dynamic load factor).
  https://help.xeneta.com/docs/capacity-and-load-factor-trends
- Freightos. Global Freightos Air Index (FAX) — 7-day rolling spot.
  https://www.freightos.com/enterprise/terminal/fax-global-freightos-air-index/
- WorldACD. Market data (transaction microdata; no public lead-time cut).
  https://www.worldacd.com/market-data/
- Rotate. Live Capacity (per-flight slot utilization).
  https://letsrotate.com/news/rotate-launched-live-capacity/
- IndexBox. Transpacific air cargo load factor ~90%.
  https://www.indexbox.io/blog/transpacific-air-cargo-utilisation-hits-maximum-as-semiconductor-demand-surges/

**Close-in pricing behaviour / yield discipline (industry)** — grounds §5c
- FreightWaves. Price war keeps air cargo rates below natural level (soft-market
  level drop; "$1 better than flying empty" — attribution genericized per
  confidentiality rule). https://www.freightwaves.com/news/price-war-keeps-air-cargo-rates-below-natural-level
- Air Cargo News. Etihad Cargo takes a disciplined approach to capacity management
  ("not going to go to the bottom on yield").
  https://www.aircargonews.net/etihad-cargo-takes-a-disciplined-approach-to-capacity-management/1057022.article
- Air Cargo Week. Air capacity recovery brings new booking and routing questions
  (carriers "accepting fewer bookings close to departure").
  https://aircargoweek.com/air-capacity-recovery-brings-new-booking-and-routing-questions/
- The Loadstar. Airlines chase yield as weak demand fails to dent air cargo rates.
  https://theloadstar.com/airlines-chase-yield-as-weak-demand-fails-to-dent-soaring-air-cargo-rates/
- IBS Software. Dynamic pricing – air cargo's next frontier (per-flight residual
  repricing). https://blog.ibsplc.com/airline-cargo/dynamic-pricing-air-cargo-s-next-frontier
- FreightWaves. Lufthansa Cargo offers dynamic spot pricing (Rapid Rate Response, via
  PROS). https://www.freightwaves.com/news/lufthansa-cargo-offers-dynamic-spot-pricing
- The Loadstar. Carriers lose pricing discipline with 'unsustainable' ex-Asia rates
  (discipline erodes in oversupply). https://theloadstar.com/carriers-lose-pricing-discipline-with-unsustainable-ex-asia-freight-rates/

**Mechanics / tariffs (primary-official)**
- CMA CGM Air Cargo, General Conditions of Sale, v2.0 (2024-03-31).
  https://www.cma-cgm.com/assets/public/documents/CCAC_General%20Conditions%20of%20Sale_20240331_V2_0.pdf
- ITA Airways Cargo, Conditions of Carriage (2025-10-26).
  https://www.ita-airways-cargo.com/resource/1671617752000/Conditions_EN
- American Airlines Cargo. Fair booking policy.
  https://www.aacargo.com/about/american-airlines-cargo-implements-fair-booking-policy.html
- IATA, Cargo Services Conference Resolutions Manual, RP 1670 (messaging standard).
  https://www.iata.org/contentassets/b38f5c2910e843bc967f4fff2d4fc53a/rp1670.pdf
