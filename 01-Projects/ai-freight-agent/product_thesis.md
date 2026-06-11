# Product Thesis & Moat

**Version:** 0.1 (Draft)
**Date:** 2026-06-03
**Status:** Draft — strategic thesis, not a requirements doc

---

## Purpose

This is the durable view of the product: what we are actually building toward, why it
is defensible, and in what sequence the pieces unlock each other. The PRD is the
near-term requirements cut; this doc is the long arc the PRD is a first step along.

The deck is a *derivative* of this doc — the moat slide pulls from here. If the
capacity/yield thesis keeps falling out of the pitch, it is because it has had no
written home. This is the home. Fix the source, the deck stops dropping it.

Cross-references: [`PRD.md §3.6`](PRD.md) (intelligence stack, two-pronged wedge),
[`PRD.md §5`](PRD.md) (differentiation requirements), [`PRD.md §7`](PRD.md) open
questions 10–12 (pricing engine, portfolio optimizer, end-to-end quoting — these *are*
the upper layers described below), [`appendices/competitive.md §C.8`](appendices/competitive.md)
(moat analysis).

---

## 1. The product is a value gradient, not an app

The durable product is four layers. They are not four features — they are one closed
loop that tightens as you move up. Each layer is independently useful and independently
shippable; each one generates the data the next layer needs to exist.

| Layer | What it decides | What it adds | Defensibility on its own |
|---|---|---|---|
| **L1 — Planner (the wedge)** | Given committed bookings, the min-cost feasible route across modes | Labor automation, speed, portfolio-aware allocation, MILP-certifiable plans | Low — commoditizable; "AI builds it in a weekend" is the *perception* to disarm |
| **L2 — Replan loop** | As conditions change (air: demand arriving over time against finite capacity; ocean: in-transit drift), the new best plan for the flexible portion | The actual cost savings. This is where the dollars are | Medium — requires the L1+sensor closed loop, which incumbents don't have assembled |
| **L3 — Capacity controller** | Across the portfolio, how much sunk/contracted capacity to consume now vs. hold, and how much to buy spot | Shadow-price / bid-price control: forward-looking hedging over a scarce, partly-sunk supply pool | High — needs proprietary buy/sell + outcome history to calibrate; not spun up in a weekend |
| **L4 — Market intelligence** | Forward direction of price and capacity per lane → lock now vs. wait, fix vs. spot | A leading-indicator signal on market movement, fed as a control input to L1–L3 | High *if embedded as action*, weak if sold as a chart (see §5) |

The closed loop that makes L1+L2 one system, not two:

```
estimate (quote)  →  plan  →  sense drift (in-transit transit-time ML)  →  replan
        ▲                                                                     │
        └─────────────────────────── outcome data ────────────────────────────┘
```

The transit-time ML model is not a third feature sitting beside the planner — it is the
**sensor** that triggers replan. Quote-time, plan-time, and in-transit estimation are
the same model applied at three points in one loop. That synergy is the wedge's
value; it is also what makes the wedge a *system* rather than a thin optimizer an
incumbent bolts on.

### The continuity that ties L2 to L3

Replan is the **myopic, local version of the controller's global decision.** Every time
L2 reroutes a flexible shipment, it is implicitly answering "consume contracted capacity
now or hold it?" — greedily, one shipment at a time, with no foresight over future
demand or price. L3 is the same decision made with foresight over the whole portfolio:
*replan with a forward view of capacity scarcity and price.* The controller is not a
different product bolted on top; it is the planner's replan logic, de-myopified through a
shadow price.

This continuity is the spine of the thesis. It is why "one engine, four layers" is a
real architecture and not a roadmap of unrelated bets.

---

## 2. Where the value actually is: replan, not "planners can't plan"

The cost-savings story has been under-represented (it is not yet in the deck) and the
framing has been wrong. The pitch is **not** "planners cannot plan well." Experienced
planners plan routine shipments fine. They struggle only when scale is large and the
network is tight and congested — and even then they get a defensible answer.

The real value is **replan of the flexible portion of the book** as conditions change.
A human cannot continuously re-evaluate a moving pipeline against new bookings, shifting
capacity, and live spot prices. The system can. The savings come from reshuffling the
still-open book as it reveals itself — reassigning a flexible shipment off a scarce slot the
moment an urgent one arrives that needs it, re-consolidating shipments that hadn't both booked
yet — not from out-planning a human on a static snapshot. (For **air** the driver is **demand
arriving over time** against finite capacity; for **ocean** it is in-transit transit-time drift.
The reshuffle is **reactive**, never speculative slot-holding.)

This reframe matters because it is also the bridge to L3: the same drift-response logic,
made forward-looking, *is* the capacity controller.

To keep the claim honest, the proof **decomposes** the human→system gap into **planning value
(L1** — the MILP out-plans a manual spreadsheet even with no replanning) and **replan value
(L2** — replanning the open book beats committing once, same solver). L2 is the thesis headline;
L1 is real, near-term-sellable value (no one has productized MILP planning for the mid/small
market) but a separate claim. The decomposition method is `model/backtest_methodology.md` (four
arms: human heuristic → MILP-no-replan → MILP-replan → clairvoyant floor); for air, the
stochastic driver is **demand arrival**, not transit time.

### Estimation method (the load-bearing claim)

The replan savings figure is the load-bearing claim of the entire thesis — the arc
(wedge → controller → moat) rests on replan generating measurable, attributable savings on
the flexible portion of a real book. It does **not** require a design partner to estimate.
A partner *validates and anchors*; it does not *originate* the number.

**1. Estimate via calibrated simulation, not a partner.** Run the synthetic backtest: real
topology (available) + synthetic commercial params + the realized uncertainty process, and
measure the gap across a **four-arm decomposition** — `H₀` human heuristic → `M₀` MILP
no-replan → `M₁` MILP replan → `π_hind` clairvoyant floor — reporting **L1 = H₀−M₀ (planning
value), L2 = M₀−M₁ (the replan headline), Total**. For the **air** proof the uncertainty
process is the **HAWB arrival stream** (demand revealed over the sim clock); transit is
low-variance, realized once per shipment so OTP is a real **population-over-time** number (the
on-time fraction over the book, not a per-route probability). (For **ocean**, transit drift — AIS
delay distributions (NOAA), blank-sailing frequency, spot-rate volatility — re-enters as a
first-class driver.) Method + invariants:
[`model/backtest_methodology.md`](model/backtest_methodology.md).

**2. Report a band, not a scalar.** L2 is near-zero when capacity is abundant (any commitment
order works) and is *expected* to rise as capacity tightens — but **convexity is a tested
hypothesis, not an assumption**. Trace `savings(congestion)` over a 2-D sweep (**κ
capacity-tightness × λ arrival-lateness**) as a **band** (adversarial-arrival floor +
sandbagged-flexibility), with a named, pre-registered peak regime. The peak figure is the
load-bearing one: disruption insurance with calm-period upside (ties to [`PRD.md §5.9`](PRD.md)
#3). A single partner measured in a calm quarter hands you the *worst* version of your number.

**3. Cost and OTP are a frontier, not two scalars.** Replan can spend cost to buy OTP or save
cost at constant OTP — two independent numbers are gameable. Trace the cost–OTP curve with a
single **dollarized tradeoff lever** (`min α·cost$ + (1−α)·lateness$`, α swept) and require
`M₁` to **dominate `M₀` at matched-OTP AND matched-cost**; OTP is scored against a
**booking-frozen promise**. In buyer terms it is one knob: squeeze cost vs. protect service.

**Bias bracketing (state honestly).** Simulation tends to *overstate* (clean flexibility
labels, well-behaved arrivals, model friendlier than reality); a single calm-period partner
*understates* (low congestion). The truth is bracketed between them — which is why neither
alone is the answer. Literature priors (online stochastic combinatorial optimization, network
revenue management / bid-price control) bound the plausible range as a sanity check.

**Regime caveat (air is a favorable regime — the proof number is an upper-ish anchor).** Air
is the *best case* for the replan thesis: short cycles, high frequency, abundant cheap
re-routing optionality, volatile capacity. Ocean is the opposite — multi-week, sparse sailings,
and blank-sailings that *remove* options rather than add them — so replan value there could be
structurally lower. The air figure is therefore an **upper-ish anchor, not a representative
one**; the deck must not imply it generalizes unqualified. **Ocean FCL is committed as the
explicit asymmetry test** (Stage 4): proving or bounding the thesis in the unfavorable regime is
what makes the air number credible rather than cherry-picked.

> A defensible, honestly-caveated estimate can go in the deck **before** any partner is
> signed — as a `savings(congestion)` **band** with a cost–OTP frontier, never as a fabricated
> scalar. The design partner later validates and anchors one point on the curve to a real
> network.

---

## 3. Moat mechanics

The honest near-term assessment in [`competitive.md §C.8`](appendices/competitive.md)
holds: weak moat in years 1–2, potentially strong by year 3–4. This section is the
*mechanism* behind that, organized around the value gradient.

### 3.1 The data flywheel (the real moat)

L1–L2 generate proprietary exhaust that nobody else retains:

- **Point-in-time estimate-vs-actual.** What was *known* at each decision moment (ETAs,
  supply on hand, committed orders) vs. what actually happened. Most forwarders overwrite
  estimates with actuals and keep no estimate history — they literally cannot measure
  their own planning quality. That gap is itself a wedge ("you can't currently measure
  this") and a forward-shadow-run pilot.
- **Realized capacity-consumption decisions** under known forward state — the training
  signal that calibrates the L3 controller and the L4 forward signal.

A competitor's fresh AI starts each integration and each forecast from zero. Ours starts
from a corpus that compounds. The model *generates*; the proprietary normalization corpus
and the verification/eval harness are what make the output *trustworthy*. The moat is the
corpus and the harness, not the generation.

### 3.2 Two distinct cold-starts (do not conflate them)

The upper layers are defensible precisely because they require data you do not have on
day one. But the data requirement is the barrier to entry **for us, too** — and it is a
*different* barrier for each layer:

- **L3 controller — portfolio depth.** Bid-price control over sunk-vs-spot allocation only
  bites when the allocation portfolio is deep and lane volume is dense. This is size-gated:
  the forwarders deep enough for it to matter are the large ones, who tend to build
  in-house ([`PRD.md §6.2`](PRD.md) wrong-buyer analysis, DSV/Tango).
- **L4 market intelligence — network density.** The forward price/capacity signal is only
  worth paying for if it *beats the public index forecast* (Xeneta, Freightos FBX,
  Drewry WCI, TAC Index for air). The edge is the leading indicator your own flow gives
  you — booking lead-time compression, quote-to-book conversion shifts, your buy rates
  before they print to the indices. That edge exists only once you have enough
  cross-customer flow to see the market turn early. Different axis (network density), same
  flywheel logic.

Both cold-starts are answered the same way: **the wedge is how you earn the data.** L1–L2
are not the deliverable — they are the instrument that produces the calibration corpus the
upper layers need. That reframes the months spent on the planner: it is not just a product,
it is the data-acquisition mechanism for the defensible layers.

### 3.3 Encoded domain judgment

Every freight-specific constraint the planner already handles — flat-rate breaks,
min-flat-break logic, density mixing, BSA over-pivot signals, DG segregation, MAWB-vs-coload —
is encoded judgment a generic optimizer silently gets wrong because it does not know the
constraint exists until a real rate sheet burns it. This is the "scar tissue" advantage.
**Never sell it as "this was hard to build"** (a buyer can't evaluate that, and it reads as
engineering vanity). Sell the *specific freight reality the engine handles that a generic
optimizer gets wrong*, and let the difficulty be the implication.

---

## 4. Build sequencing

The architecture is "one engine, four layers." The build order is dictated by the data
dependency, not by which layer is most defensible:

1. **L1 planner + transit-time ML** (current phase). Ships first, sells first, and —
   critically — earns the estimate-vs-actual + capacity-consumption corpus.
2. **L2 replan loop.** The closed loop. Where the demonstrable savings come from. Turns
   L1 from "an optimizer" into "a continuously-correct plan."
3. **L3 capacity controller.** Built once the corpus can calibrate a shadow-price engine.
   Maps to [`PRD.md §7`](PRD.md) open questions 10 (pricing engine) and 11 (portfolio
   optimizer). The air model is already seeding this — the capacity manager sends a
   signal to the planner for how much BSA to consume above pivot (see air model BSA
   accumulator / volume-kicker work).
4. **L4 market intelligence.** Built once cross-customer flow density gives a forward edge
   over public indices. Maps to open question 12 (end-to-end quoting / RM).

Each layer is gated by the data the layer below produces. You cannot pull L3 forward; you
can only earn it.

---

## 5. Positioning notes (deck + GTM)

**Investor deck.** Lead and dwell on L1–L2 and the replan savings number. Put L3–L4 in *one*
vision/moat slide near the end, framed through the data flywheel — wedge earns the data →
data builds the controller → controller is the moat. That arc answers three objections at
once: "AI builds it in a weekend," the controller's cold-start, and "why now / why you."
Do **not** let L3–L4 onto the product or GTM slides; the failure mode for a technically deep
founder is pitching the cathedral and burying the first brick.

**Customer deck.** Cut L3–L4 to at most a one-line roadmap mention, or omit. Customers buy
cost savings on *their* flexible shipments, not your hedging architecture. Lead and end with
the replan number.

**Sell the action, not the chart (L4).** A standalone directional rate dashboard makes you
the weakest new entrant against entrenched index incumbents with years of contributed data.
The same signal, embedded as the control input that *automatically retimes bookings and
reshapes plans* inside the loop, is differentiated, sticky, and not directly comparable to a
rate index. Same data, completely different defensibility.

**Pre-loaded answers to the two sharp questions:**

- *"How is your forward signal better than Xeneta?"* — The proprietary leading-indicator flow
  from our own quoting/booking loop (lead-time compression, conversion shifts, buy rates
  before they print). Which is, again, the thing the wedge exists to produce.
- *"Which customer's portfolio is deep enough that the controller pays for itself, and is that
  the same customer you land with the wedge?"* — Land mid-market with the planner; the
  controller's full value lands as we move up-market and accumulate flow. If those are
  different buyers, say so and make it the explicit up-market motion. Do not let the deck
  imply one buyer gets all four layers on day one.

### Open tension (carry forward)

L3 portfolio control is size-gated toward the large buyers who build in-house; the mid-market
wedge buyer ([`PRD.md §6.2`](PRD.md), Tier 2 $50M–$500M) may be too thin for the controller's
full value to land. L4 broadens the addressable buyer back out (any freight buyer faces a
timing/sourcing decision) but its value down-market is capped by a coarse action space and
free-ish public indices. The unresolved question: is the diagnostic/backtest wedge ("you
can't measure your own planning quality") the right *first* surface — the thing that earns the
flow before either upper layer can pay for itself? Flagged, not resolved.

---

## Related memory

`project_two_pronged_wedge.md`, `project_intelligence_layer_positioning.md`,
`project_override_rate_kpi.md`, `reference_rate_api_landscape_2026.md`,
`reference_air_cargo_allotment_contracts.md`.
