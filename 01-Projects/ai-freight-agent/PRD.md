# Product Requirements Document
## AI Multimodal Freight Routing Agent

**Version:** 0.3 (Reorganized)
**Date:** 2026-05-16
**Status:** In Review — Phase 0

---

## 1. Executive Summary

This system is an **AI-native, autonomous multimodal freight routing engine**. Agents plan, validate, and commit routing decisions continuously — without waiting for human approval on every shipment. Humans govern the agents: they set policy, handle escalated exceptions, and audit decisions. On a normal day, operators interact with fewer than 5% of shipments. The other 95% are routed, validated, and committed by agents.

The system is built on a proven optimization and ML substrate — deterministic MILP routing, probabilistic transit time models, rolling horizon re-planning — coordinated by a LangGraph agent layer from day one. It is designed to materially outperform existing TMS platforms by: (1) being agentic-native rather than retrofitted, (2) jointly optimizing across all mode transitions rather than planning legs sequentially, (3) applying probabilistic transit time models rather than point estimates, and (4) continuously re-planning at increasing resolution as shipments progress.

**Commercial model:** Services-as-software. Per-shipment or per-decision pricing. Customers are mid-market freight forwarders and shippers.

**Autonomous operation design:** Agents route autonomously → Compliance Validator reviews → routing committed to dry-run state → auto-commits after configurable window (default 60 min). Operators see exceptions only: infeasible shipments, threshold violations, escalations. Override history is logged as training signal for policy improvement. New customers onboard in Co-pilot mode (agent prepares, human approves); autonomy expands progressively as the agent establishes a performance record.

**Closest architectural peer:** cargo.one (multimodal air + ocean, AI-native OS, MCP-connected, $20M Bessemer, same mid-market forwarder target). Key differentiator: cargo.one does intelligent carrier matching and recommendation. This system does MILP-based joint optimization with formal constraint guarantees (vessel cap, allocation strings, probabilistic transit times, container mix). No published equivalent in the market.

---

## 2. Problem Statement

Moving freight from origin to destination across multiple modes (ocean, trucking) requires solving a hard combinatorial optimization problem under uncertainty: selecting routes, carriers, schedules, and mode transitions that minimize cost, meet delivery windows, and remain robust to disruption. Today this is done by:

- Human planners using spreadsheets and email
- Legacy TMS systems that batch-plan in waves, use point estimates for transit times, and plan each mode leg sequentially rather than jointly
- Freight forwarders who have relationships and intuition but no systematic optimization

The result: suboptimal routing, reactive exception handling, and no ability to rapidly evaluate alternatives. The research estimate is 2–5% of procurement spend in routing leakage — at scale, this is material.

AI-native routing that jointly optimizes across modes, models uncertainty, and operates continuously rather than in batch waves is a genuine step-change. No current TMS does this.

---

## 3. Modes in Scope

### 3.1 Ocean FCL + Trucking (Prototype — In Scope)

**Ocean Full-Container-Load (FCL)** combined with pre-carriage trucking (origin door to port of loading), port drayage (port of discharge to inland), and inland trucking (to final destination).

This combination forces mode-transition handling (the hard part) from day one and covers the dominant volume pattern for cross-border freight. MILP formulation: `model/ocean_fcl_routing.tex`.

**Trucking formal model:** `model/trucking_routing.tex` — three-mode model covering FTL, PTL (Volume LTL), and LTL with carrier-tendering semantics, hard LTL refusal rules (linear-foot, piece dimensions, total weight), contract FTL allocation caps (parallel to ocean string allocation), MABD delivery windows, carrier-lane service availability, FAK class overrides, density-based NMFC 2025 SDS pricing, and tender acceptance probability as a first-class parameter. Anchored in Powell-Sheffi load-planning literature, Caplice procurement work, and the 2025 NMFC overhaul.

**Trade lanes in scope for prototype:**
- Trans-Pacific Eastbound (TPEB): China / Southeast Asia → US West Coast / Gulf
- Far East Westbound (FEWB): China / SE Asia → Europe / Mediterranean

---

### 3.2 Air Freight (In Scope — Phase 2)

Air freight is in scope for Phase 2 alongside ocean FCL. It is the dominant mode for time-critical shipments (electronics components, pharmaceuticals, e-commerce returns) where speed justifies 4–8× the ocean cost premium. Formal model: `model/air_optimizer.tex` (not yet written).

**Key differences from ocean FCL:**

| Property | Ocean FCL | Air |
|---|---|---|
| Rate unit | Per container (FEU/TEU) | Per chargeable weight kg |
| Chargeable weight | N/A | max(actual_kg, volume_cbm × 167) |
| Capacity unit | TEU slots on vessel | ULD positions on freighter / belly capacity |
| Time unit | Days | Hours |
| Schedule frequency | Weekly per lane (typical) | Daily or multiple daily flights |
| Carrier type | Ocean carrier (MSC, CMA CGM) | Airline, GSSA, integrator (DHL, FedEx) |
| Consolidation | FCL = full container, LCL = shared | Air is always ULD consolidation by GSSA or forwarder |

**Rate structure:** Chargeable weight = max(actual_weight_kg, volumetric_weight_kg) where volumetric_weight_kg = volume_cbm × 167. Rate is quoted per chargeable weight kg. Surcharge stack: FSC (fuel), SSC (security), AMS (customs manifest), and terminal handling charges at origin and destination airport.

**Carrier layer:** The forwarder contracts with a GSSA (General Sales and Service Agent) or directly with airlines. Integrators (DHL Express, FedEx, UPS) operate their own networks and are contracted differently. The optimizer must model all three carrier types as arc options.

**Optimizer design:** Air Optimizer is a separate MILP from ocean, with chargeable weight as the capacity variable rather than TEU slots. Decision variables: airline/routing selection, flight date. Constraints: chargeable weight capacity per flight (proxy for ULD availability), deadline window (in hours), cargo type restrictions.

**Data source:** OpenSky Network historical flight data for Phase 2 (free). Commercial airline schedule feed (IATA, Cirium, or OAG) for Phase 5 production.

**Two capacity/rate modes — both supported from day one:**

The air optimizer operates in two modes simultaneously per lane. A forwarder may have contracted ULD allocations on some carrier/schedule combinations and rely on spot rate cards for others. Both are first-class options in the optimizer.

**Mode 1 — Rate card (spot market):**
```
chargeable_weight = max(actual_kg, volume_m³ × 167)
cost = chargeable_weight × rate_per_kg(weight_break)
     + chargeable_weight × FSC    (fuel surcharge — refreshed monthly)
     + chargeable_weight × SSC    (security surcharge — stable)
     + origin_THC                 (terminal handling, per shipment)
     + destination_THC
     + AMS_filing_fee             (US imports only)
```

IATA weight breaks: N (minimum) → +45 kg → +100 kg → +300 kg → +500 kg → +1000 kg. Rate per kg decreases as weight increases.

**Mode 2 — ULD contracted capacity:**

Analogous to ocean BSA. A forwarder pre-commits ULD positions on a carrier schedule (e.g., "2 LD3s per week on Singapore Airlines PVG → LAX, Fridays") at a contracted all-in rate per kg. The optimizer prefers filling contracted ULDs before going to spot — better rate, guaranteed space.

ULD types and dimensions:
| Type | Usable volume | Payload |
|---|---|---|
| LD3 | 4.5 m³ | 1,587 kg |
| LD7 | 11.1 m³ | 4,626 kg |
| PMC pallet | 7.5 m³ | 6,804 kg |
| AKE (small LD3) | 4.5 m³ | 1,497 kg |

ULD assignment adds a bin-packing layer: shipments must fit within the ULD's physical dimensions and weight limit. Multiple shipments may share a ULD (the optimizer decides). Overflow beyond ULD capacity routes to the spot rate card.

ULD allocation data lives in `air_uld_allocations` (see `data_model.md §3.3`). Forwarders define their ULD contracts during onboarding: carrier + ULD type + origin/destination airports + departure days + ULDs per departure + contracted rate. The system tracks `remaining_ulds` against this allocation exactly as `rem(s,t)` works for ocean strings.

**Structural analogy to ocean:**
| Ocean | Air |
|---|---|
| BSA string allocation | ULD contracted allocation |
| Spot rate (load board) | Spot rate card |
| FEU/TEU container mix | ULD type selection + bin-packing |
| rem(s,t) per string/period | remaining_ulds per carrier/schedule/week |

**Capacity (spot mode):** Unconstrained proxy for MVP. Slot constraints added if they bind in practice.

**Open design questions:** (1) Integrator network (DHL Express, FedEx, UPS) — fixed zone-based rate card arc set, or API? (2) Path-level TTM must handle mixed hour/day units when air leg is in hours and trucking legs are in days.

---

### 3.3 Ocean LCL (In Scope — Phase 2)

Ocean LCL (Less-than-Container-Load) is in scope for Phase 2. When a shipment's volume is too small to fill a container, it moves LCL — consolidated with other cargo by the forwarder or an NVOCC into a shared container. Formal model: `model/ocean_lcl_optimizer.tex` (not yet written).

**Key differences from FCL:**

| Property | Ocean FCL | Ocean LCL |
|---|---|---|
| Rate unit | Per container (FEU/TEU flat) | Per CBM or per W/M ton (whichever greater) |
| Capacity constraint | TEU slots on vessel | CBM + kg capacity per container |
| Carrier type | Ocean carrier direct | NVOCC (consolidation forwarder) |
| Consolidation decision | None — full container per shipment | Which shipments share a container |
| Schedule type | Carrier port-pair sailing schedule | NVOCC consolidation schedule (weekly CFS cutoffs) |
| CFS cutoff | CY cutoff (container yard) | CFS cutoff (container freight station — earlier than CY) |

**Optimizer design:** Combined bin-packing × routing MILP. Two coupled decisions:
1. **Assignment:** which LCL shipments are loaded together into a container
2. **Routing:** which NVOCC sailing the container is assigned to

Decision variables: x_{k,c} = 1 if shipment k is in container c; y_{c,s} = 1 if container c is on NVOCC sailing s.
Constraints: container CBM capacity, container weight capacity, CFS cutoff per sailing, delivery deadline per shipment, cargo compatibility (hazmat segregation between co-loaded shipments).
Objective: minimize total cost = Σ NVOCC_rate_per_CBM × CBM_k across all shipments.

**Deconsolidation at destination:** LCL containers arriving at POD must be broken down at a **Container Freight Station (CFS)** before individual shipments can move to their final destinations. The CFS is a new node type in G(N, A) — between POD_exit and the inland delivery nodes. CFS dwell time (typically 1–2 days) must be modeled as a dwell arc.

**Data requirements:** NVOCC consolidation schedules (CFS cutoff dates, weekly departure schedule, POL→POD transit time) and LCL rate tariffs (per CBM by lane). Data source not yet confirmed — this blocks the LCL LaTeX model. Candidate sources: NVOCC rate APIs, carrier tariff filings, synthetic rates calibrated to market ranges.

**Interaction with Destination Leg Planner:** When LCL freight arrives at POD CFS, the Destination Leg Planner takes over for the inland leg. Multiple LCL shipments clearing the same CFS for the same inland destination cluster may be consolidated into a single FTL truck move at this point — the Destination Leg Planner must consider this multi-shipment consolidation opportunity.

---

### 3.4 Destination Leg Planner (In Scope — Phase 2)

When the main leg (ocean or air) is firm_planned and the AIS/flight ETA narrows to within 72 hours of POD, the system triggers precision planning for the POD → final destination leg. This is a separate optimization from the main leg optimizer — smaller problem, shorter horizon, real carrier availability, live rates. Formal model: `model/destination_leg_planner.tex` (not yet written).

**Decision:** Mode selection (FTL truck / LTL truck / intermodal rail) + carrier + departure timing.

**FTL truck:** Volume fills or nearly fills a trailer (configurable threshold, e.g., ≥ 15 CBM or 80% of trailer cube). Direct, fastest.

**LTL truck:** Volume below FTL threshold. May require routing through a deconsolidation warehouse if FCL/LCL freight must be broken down before individual delivery. Rates per pallet or cwt. LTL case: if multiple LCL shipments arrive at the same POD for the same inland destination cluster, the planner may consolidate them into a single FTL move.

**Intermodal rail:** Origin intermodal ramp → destination intermodal ramp via BNSF or UP, plus drayage at each end. Generally the lowest-cost option for hauls > 800 km. Adds 1–3 days transit vs. FTL. Rail ramp locations and service schedules from BTS FAF + BNSF/UP public data.

**New node types in G(N, A):**
- Intermodal rail ramps (BNSF, UP) — with service days and CFS cutoffs
- Deconsolidation warehouses (CFS) — for LCL and LTL freight breakdown

---

### 3.5 Rail and Other Standalone Modes (Deferred)

Full standalone rail optimization (shipper-to-consignee by rail without the ocean/air main leg) is deferred. Rail is modeled as an arc type within the Destination Leg Planner (§3.4) but not as a primary mode for international freight.

---

## 4. Document Map

The PRD has been decomposed into specialist files by change frequency and concern. This document is the strategic index.

| File | Contents | When to read |
|---|---|---|
| **`PRD.md`** (this file) | Executive summary, problem statement, modes in scope, differentiation requirements, open questions | Strategic framing; always start here |
| **`agent_design.md`** | AI-native design philosophy, autonomy model, confidence tiers, guardrails, deployment modes, routing triggers, agent capabilities, agent architecture (LangGraph, hierarchical pattern, HITL, capability registry) | Agent behavior, trust model, architecture decisions |
| **`data_model.md`** | Supply and demand model, graph G(N,A), arc schemas, container specs, string allocation, rolling horizon planning, customer and tenant entity model (SQL schemas, user roles, shipment lifecycle) | Data structures, database schema, graph formulation |
| **`ui_spec.md`** | Look and feel, color system, typography, screen inventory, persona views, agent action feed, mobile philosophy, wireframes (6 screens), interaction design decisions, agent reasoning transparency | UI/UX design, frontend implementation |
| **`personas_and_tools.md`** | Four user personas (Shipper, Ops Planner, Analyst, Compliance), per-persona Q&A with MCP tool mappings, master MCP tool inventory (P0/P1/P2), P0 priority summary | Tool design, persona requirements, MCP server scope |
| **`build_plan.md`** | Tech stack, multi-tenancy architecture, database design, demand generator, peripheral product components (auth, billing, notifications, admin), agent execution architecture, data sources, components inventory, build sequence, unit testing requirements | Engineering implementation, infrastructure, build order |
| **`appendices/capabilities.md`** | Full agent capability inventory (60+ capabilities across 9 categories) | Capability completeness review |
| **`appendices/competitive.md`** | Differentiation opportunities (7 gaps in all TMS platforms), competitive landscape (16 companies), industry-validated patterns, market gaps, adversarial position assessment, moat analysis, attack scenarios | Competitive positioning, feature prioritization |
| **`model/ocean_fcl_routing.tex`** | Formal MILP formulation for Ocean FCL routing — Draft v2 | Mathematical model review, solver implementation |
| **`Research.md`** | Deep competitive research: 33 sites, 14 companies, May 2026. Full company profiles, sites visited. | Competitive intelligence detail |
| **`docs/freight_concepts.md`** | Freight domain glossary: HBL/MBL, container lifecycle, booking flow, B/L types, trucking instructions, intermodal rail, ULD management, surcharges, customs | Domain knowledge reference for engineering and product team |
| **`docs/taiwan_market.md`** | Taiwan market analysis: TAM $15–20M / SAM $1.5–5M / SOM $300K–1M, top 20 forwarders, TMS adoption, Dimerco deep dive, design partner sequencing | Taiwan go-to-market and design partner planning |
| **`docs/us_market.md`** | US market analysis: TAM $75–160M / SAM $25–50M / SOM $2–8M, major US forwarders, TMS landscape, regulatory complexity (ISF/AMS/PGA), go-to-market approach | US go-to-market, sales motion, conference channels |

---

## 5. Differentiation Requirements

These capabilities do not exist in any current TMS platform. They are explicit design requirements, not nice-to-haves. Detailed treatment in [`appendices/competitive.md §B`](appendices/competitive.md).

### 5.1 Continuous Re-Optimization (Rolling Horizon)

The system must maintain and continuously update a full multi-horizon plan. Batch-wave planning is not acceptable. Formalized in `data_model.md §2`.

### 5.2 Counterfactual / Regret Analysis

After each shipment completes, the system must compute:
- The set of routes that were feasible at booking time (given constraints known at T=0)
- The actual outcome for each route (using realized transit times, not estimates)
- Regret = `|cost(chosen_route) - cost(best_feasible_route_in_hindsight)|`

This data must be stored per shipment and aggregable by lane, carrier, time period, and cargo type. Used for model validation, systematic bias detection, and training data generation.

### 5.3 Learned Constraint Inference

When a human planner overrides a system routing recommendation, the system must log:
- What was recommended (route, carrier, cost, transit time)
- What was chosen instead
- Timestamp and shipment context (lane, cargo type, service level, date)
- Override reason (if provided by planner)

These override signals are the primary input for constraint learning. Over time, systematic overrides on a lane or carrier reveal implicit constraints not yet modeled.

### 5.4 Simultaneous Multi-Echelon Joint Optimization

All existing TMS platforms plan mode legs sequentially. This system must model the full door-to-door journey as a **single optimization problem** on the unified graph G(N, A). The ocean leg, transshipment, drayage, and inland trucking leg are all decision variables in one formulation — not planned one at a time.

### 5.5 Spot Capacity as Supply

Spot market capacity (broker capacity, load board postings) is modeled as a set of arcs in G(N, A) alongside contracted capacity. Spot arcs carry rate distributions rather than fixed rates. The optimizer treats spot as just another supply option — not a fallback of last resort.

### 5.6 Probabilistic Planning

Transit time on each arc is represented as a distribution (mean + variance, or parametric fit from historical data), not a point estimate. The optimization objective must support:
- Expected cost / expected transit time
- Probability of on-time delivery given a delivery window (P(arrival ≤ deadline))
- Risk-adjusted objectives (e.g., minimize cost subject to P(on-time) ≥ 0.95)

### 5.7 Fully Autonomous Decision Chains (End Goal)

The prototype implements decision-support (human approves). The end goal is a fully autonomous planning and exception-management chain: the system detects a disruption, re-optimizes affected shipments, selects the best alternative, books it (via carrier API), and notifies stakeholders — without human intervention. A compliance/validation agent checks planning decisions before execution.

### 5.8 MILP-Grounded Optimization (vs. Intelligent Matching)

The closest market peer (cargo.one) does intelligent carrier selection and recommendation — pattern matching against historical data with ML ranking. This system produces recommendations that are **provably feasible** with respect to hard constraints (allocation caps, vessel slots, deadline windows, container capacity) and **certifiably optimal** within the formulation via MILP. The agent can explain not just what it chose, but why no cheaper feasible route exists. This distinction matters for operator trust and audit: a recommendation backed by an MILP certificate is not the same as a recommendation backed by similarity to prior decisions.

---

## 5.9 Business Case — Why MILP Matters Even on "Routine" Routes

A common critique of MILP-based routing is that 80–95% of FCL shipments are "routine" — a short list of viable carriers, no capacity crunch, trivially optimizable by a human planner. The critique frames MILP as overpowered for easy problems and therefore of limited practical value.

This framing is wrong. It conflates the difficulty of the individual routing problem with the value of the overall system. The value of this product is not concentrated in the hard 5% — it is distributed across all five dimensions below:

**1. Autonomous operation requires a certifiable output**

A human planner who picks "the obvious carrier" on a routine shipment can be overridden, second-guessed, or blamed. An agent that commits routing autonomously needs a stronger foundation: a proof that the selected route is feasible and optimal given all constraints. MILP provides exactly this — an optimality certificate that justifies autonomous commitment without human review. You cannot safely build a 95%-autonomous routing agent on top of a heuristic, because a heuristic can't tell you when it might be wrong. MILP can.

**2. Portfolio-aware allocation is invisible on a per-shipment basis**

Even on an individually "easy" shipment, the MILP sees something a human planner routing that shipment in isolation cannot: that shipments #47–52 this week are all competing for the same BSA string allocation. Shipment #47 looks trivial — three viable carriers, pick cheapest. MILP routes it knowing that allocation will be needed for #51, which has a harder deadline. A planner routing #47 does not know #51 exists yet. This cross-shipment constraint awareness is not replicable without a portfolio optimizer.

**3. "Routine" is not a stable condition — it's a forecast**

A lane is routine 80% of the year. Then Golden Week hits, or USLAX backs up, or a vessel is blanked, or a carrier cancels a string. Suddenly every shipment on that lane is a hard constrained problem. The forwarder with an optimizer handles the disruption. The forwarder using manual planning or a heuristic falls apart. The system you build for disruption runs idle during calm periods — that is acceptable. It is insurance with operational upside.

**4. Labor automation is the primary ROI driver, not route quality**

A 500-shipment/month forwarder has planners spending 20,000+ minutes per month on routing decisions. The per-shipment labor cost at $60K planner salary is approximately $5–8 per shipment. If our system handles 95% of those decisions autonomously at under $1 per routing decision in compute, the ROI is immediate and independent of whether any individual route is "hard" or "easy." Easy routes automated at scale are as valuable as hard routes optimized — because they free planner time for exceptions that genuinely require judgment.

**5. Speed wins slots**

For time-sensitive bookings, the 2-second optimizer wins the vessel slot that the 30-minute manual planning process misses. On peak-season Fridays when TPEB slots fill quickly, the forwarder who confirms first gets the space. Speed is a real competitive advantage even on lanes where the routing decision itself is obvious.

**The correct analogy:** A high-performance vehicle is not only valuable on winding mountain roads. On any road, it is faster, more consistent, and — critically — it enables autonomous driving. The driver can let go of the wheel on a straight road precisely because the system is trustworthy enough to handle it. The value of autonomous capability is not limited to the roads that require manual attention.

**Honest scope of MILP advantage by scenario:**

| Scenario | MILP advantage |
|---|---|
| Routine FCL, unconstrained capacity | Labor automation, speed, consistency, portfolio awareness |
| Routine FCL, near-full BSA allocation | Portfolio constraint awareness is critical — human planners miss this |
| Peak season / capacity crunch | Full MILP advantage: joint optimization over constrained allocation |
| Multi-leg with binding joint constraints | Maximum MILP advantage: joint optimization provably outperforms sequential planning |
| Disruption / vessel delay / re-planning | Rolling horizon re-optimization is only reliable with MILP underneath |

---

## 6. Market Opportunity

*Estimates from first principles — primary market research not yet done. Treat as directional, not authoritative. Numbers sourced from adversarial commercial critique, May 2026.*

### 6.1 TAM / SAM / SOM — Methodology

*All estimates derived from first principles. Primary market research not yet done. Treat as directional.*

**Consistent methodology across all markets:**

| Step | Rate | Basis |
|---|---|---|
| Software spend as % of forwarding revenue | 1.5–2% | Industry benchmark for freight tech spend |
| Routing/optimization as % of software | 15–20% | Estimated share across ~8 software categories |
| Tier 2 forwarder ACV | $30–50K/year | Benchmarked against comparable TMS optimization modules |
| Tier 2 definition | $50M–$500M annual revenue | Target buyer profile — see §6.2 |

---

### 6.2 Market Sizing — Global, US, Taiwan

| | **Global** | **United States** | **Taiwan** |
|---|---|---|---|
| **Freight forwarding gross revenue** | ~$214B | ~$35–40B (international portion of $127.7B total US market) | ~$6B (2.85% of global) |
| **Total freight software spend** | $3–4B | $500M–800M | ~$100M |
| **TAM** (routing optimization software) | **$450M–$800M** | **$75M–$160M** | **$15–20M** |
| **Tier 2 forwarder count** | ~2,000–3,000 | ~300–500 | ~50–100 |
| **SAM** | **$150M–$250M** | **$25M–$50M** | **$1.5M–$5M** |
| **SOM (5-year)** | **$5M–$20M ARR** | **$2M–$8M ARR** | **$300K–$1M ARR** |
| **Customers to hit SOM** | 100–300 | 50–150 | 10–20 |
| **Competitive density** | Moderate | Moderate + GoFreight incumbency | Low — no routing optimizer present |
| **Detailed analysis** | This document | [`docs/us_market.md`](docs/us_market.md) | [`docs/taiwan_market.md`](docs/taiwan_market.md) |

**US sizing note:** The $127.7B IBISWorld US figure includes domestic freight brokerage (trucking). The international forwarding portion relevant to our product is ~$35–40B. Using only the international segment for the US TAM derivation.

---

### 6.3 Global TAM — Supporting Detail

*Top-down:* Global freight forwarding ~$214B gross revenue (2026). 1.75% software spend = $3.7B total. 17.5% routing/optimization share = **$648M midpoint**. Including 3PL and NVOCC adjacent buyers → upper bound ~$1B.

*Bottom-up:* ~2,000–3,000 Tier 2 forwarders globally. $60K average ACV = **$150M SAM**. TAM (broader addressable) = $450M–$800M. Both approaches converge.

**SAM — $150M–$250M** (Tier 2 mid-market forwarders, English-speaking and European markets)

~2,000–3,000 Tier 2 forwarders globally at $50–75K average ACV.

**SOM — $5M–$20M ARR (5-year)**

Realistic capture: 100–300 Tier 2 forwarders at $30–50K ACV. Upper bound requires CargoWise integration live and strong design-partner references by year 2.

---

### 6.2 Target Customer

**Right buyer: Tier 2 freight forwarder, $50M–$500M annual revenue.**

Large enough to have a dedicated ops team (and therefore a buyer for routing software). Small enough to not have budget for bespoke in-house optimization. Will pay $30–50K/year if the product demonstrably reduces routing labor and carrier cost.

**Wrong buyers:**
- **Tier 1** (Kuehne+Nagel, Expeditors, DSV, CEVA): building in-house (DSV/Tango is proof); procurement cycle >24 months; risk of productizing their own tool against us
- **Tier 3** (<$50M): too price-sensitive; buys GoFreight/WMS-tier products; MILP complexity is not valued at this volume

**ACV: $30–50K/year.** Requires low-touch CS and product-led onboarding — CAC must stay well under $30K or the unit economics break.

---

### 6.3 Commercial Model

**Per-shipment pricing:** Incentive-aligned — customer pays when the system routes a shipment. Eliminates "we're not using it" churn. Complicates revenue forecasting.

**Risk — adverse selection:** Per-plan pricing may cause customers to submit only their hardest cases, inflating per-plan compute cost. Mitigate with a monthly minimum commit (floor + per-shipment above the floor).

**Unit economics to validate in design-partner phase:** Cost per routing decision = MILP solve time × compute cost + LLM call cost. Prototype target: <$1 per routing decision. Determines pricing floor and whether per-shipment model is viable.

---

### 6.4 Integration Prerequisite

Pure routing SaaS without TMS write-back will not be purchased by mid-market forwarders. CargoWise integration is a **sales prerequisite**, not a Phase 5 nice-to-have.

- CargoWise partner program: 4–12 weeks for approval
- Per-customer CargoWise integration: 6–9 months

Design partners must be identified and CargoWise partner enrollment started before GTM begins. CSV/manual data entry is the interim fallback for early design partners only. See `build_plan.md` for integration sequencing.

---

### 6.5 Competitive Window

**18–24 months** before the competitive environment materially hardens. After that window, WiseTech's routing optimization module will be live in CargoWise, and cargo.one will have had time to add joint optimization on top of their carrier data foundation. The differential between MILP and heuristic matching narrows once competitors have the data and the talent.

Implication: design partner relationships and CargoWise integration must be underway before month 12. See [`appendices/competitive.md §C.9`](appendices/competitive.md) for the three specific attack scenarios.

---

## 7. Open Questions and Future Decisions

1. **Decision-support vs. autonomous execution**: Prototype is decision-support (human approves). Define the trigger conditions and safety checks required before moving to autonomous execution. *(Key commercial and liability decision — defer until prototype is validated with design partners.)*

2. **Design partner selection**: Who are the first 2–3 customers? Freight forwarder vs. shipper? What data do they bring? What lanes do we start with?

3. **Live AIS feed**: NOAA historical is sufficient for model training. For production tracking we need a live feed. Evaluate MarineTraffic, VesselFinder, SpireGlobal on cost vs. coverage vs. API quality.

4. **Carrier booking APIs**: When we move toward autonomous execution, we need carrier API integrations for booking. Who are the first ocean carriers? (MSC, CMA CGM, COSCO are the volume leaders — start with one.) Which trucking carriers for drayage and inland?

5. **Pricing model**: Per-shipment, per-decision, or monthly subscription with volume tiers? What is the per-shipment cost floor given compute (MILP solve + LLM call per routing decision)?

6. **Emissions optimization**: Carbon as a routing objective requires accurate emissions factors per mode, carrier, and vessel. Data source TBD.

7. **Multi-agent framework**: LangGraph (decided — see `agent_design.md §3`). LangSmith for observability, PostgreSQL checkpointer for HITL state persistence.

8. **LCL consolidation optimizer**: LCL routing requires a consolidation layer that groups LCL shipments into containers before routing. This is a combined bin-packing + routing MILP — distinct from the FCL ocean optimizer. Requires NVOCC consolidation schedules and LCL rate data. See §3.3 above.

9. **Time-phased carrier capacity release**: The MVP commits all of `rem(s,t)` (remaining contracted block on a string in a period) in a single routing run. A production forwarder cannot do this — future urgent shipments need reserved headroom. The correct approach is to release capacity in tranches across sequential routing batches, driven by a demand forecast model. Deferred to P1; the string allocation constraint structure is already in place to support it.

10. **Pricing engine**: The routing MILP minimizes carrier cost — it produces cost, not price. Pricing (what the forwarder charges the shipper) is out of scope for v1. A future pricing engine would sit above the routing layer: take the optimized carrier cost as input, apply markup rules (cost-plus, market-rate benchmarking, competitive positioning), and produce a quoted price.

11. **Portfolio / capacity allocation optimization**: When the forwarder has constrained BSA allocation across competing shipments, accepting and routing all shipments at minimum cost is not always the right objective — some shipments have higher margin than others. A portfolio optimizer would jointly decide which shipments to accept, how to allocate scarce capacity across them, and how to route each, with the objective of maximizing total margin subject to allocation constraints. Requires the pricing engine. Deferred.

12. **End-to-end quoting and profit maximization**: If the product evolves to quoting directly to shippers, the system must jointly optimize routing cost and quote price. This is a revenue management problem layered on top of routing — the objective shifts from min(cost) to max(margin). Requires a pricing model, demand model (price elasticity), competitive rate benchmark, and portfolio optimizer. Treat as a distinct future product phase, not an extension of the routing optimizer.
