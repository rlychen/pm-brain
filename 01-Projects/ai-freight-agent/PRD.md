# Product Requirements Document
## AI Multimodal Freight Routing Agent

**Version:** 0.1 (Draft)  
**Date:** 2026-05-07  
**Status:** In Review — Phase 0

---

## 1. Executive Summary

This system is an AI-native multimodal freight routing engine with an agentic layer built on top. Given a set of shipment requests with supply/demand constraints, the system recommends optimal end-to-end routes across ocean and trucking modes. It is designed as a **services-as-software product (Path A)**: the customer submits shipment data, the system returns optimized routing recommendations, a human approves, and the customer executes.

The system is built on a proven architectural pattern — deterministic optimization and ML substrate that planning systems call into — extended with a modern agentic layer from day one. It is designed to be materially better than v1 by: (1) being agentic-native rather than retrofitted, (2) jointly optimizing across all mode transitions rather than planning legs sequentially, (3) applying probabilistic modeling of uncertainty rather than point estimates, and (4) continuously re-planning at increasing resolution as shipments progress.

**Commercial model:** Services-as-software. Per-shipment or per-decision pricing. Customers are mid-market freight forwarders and shippers.

---

## 2. Problem Statement

Moving freight from origin to destination across multiple modes (ocean, trucking) requires solving a hard combinatorial optimization problem under uncertainty: selecting routes, carriers, schedules, and mode transitions that minimize cost, meet delivery windows, and remain robust to disruption. Today this is done by:

- Human planners using spreadsheets and email
- Legacy TMS systems that batch-plan in waves, use point estimates for transit times, and plan each mode leg sequentially rather than jointly
- Freight forwarders who have relationships and intuition but no systematic optimization

The result: suboptimal routing, reactive exception handling, and no ability to rapidly evaluate alternatives. The research estimate is 2–5% of procurement spend in routing leakage — at scale, this is material.

AI-native routing that jointly optimizes across modes, models uncertainty, and operates continuously rather than in batch waves is a genuine step-change. No current TMS does this.

---

## 3. User Personas, Requirements, and Tool Mapping

This section is the primary requirements document. Each persona has a role description, the questions and tasks they want to perform, and a mapping to the specific MCP tool that answers each question. The tool inventory derived here drives the components built in Phase 2 and exposed in Phase 3.

**Design principle:** The majority of use cases must be answerable by a deterministic or model-backed tool call — not by the LLM reasoning from scratch or writing code dynamically. The agent's job is to route the user's question to the right tool, interpret the result, and present it clearly. Tools provide consistency, speed, and auditability. LLM inference fills gaps where judgment is needed.

**Priority definitions:**
- **P0** — Must-have for prototype. The ~20% of question types that cover ~80% of daily operational value. System is not useful without these.
- **P1** — High value, built in Phase 5 iteration. Covers the next layer of operational depth.
- **P2** — Important long-term, deferred. Specialized, lower-frequency, or dependent on data sources not yet integrated.

---

### 3.1 Persona: Shipper — Logistics Manager

**Role:** Employed by a manufacturer, retailer, or importer. Owns outbound and/or inbound freight from origin factories or suppliers to destination warehouses or stores. Responsible for on-time delivery, freight cost, and carrier relationships. Interacts with one or more freight forwarders to execute shipments.

**Volume:** 10–500 active shipments at any time. The logistics manager thinks in terms of purchase orders (POs), not containers. Each PO has a delivery due date and a cargo volume (CBM). Depending on CBM, a PO may fill one or more full containers (FCL) or share a container with other cargo (LCL). A "shipment" in this context is typically one PO or a group of POs moving together — the system must handle both FCL and LCL demand, though LCL consolidation optimization is deferred (see note below and Open Question 8).

**KPIs they own:** On-time in full (OTIF), freight cost per unit, transit time vs. committed, carrier reliability.

**What they want to avoid:** Surprises. Late arrivals, unexpected cost overruns, being the last to know about a disruption.

#### Questions and Tasks

| # | Question / Task | Priority | MCP Tool | Underlying Models/Components |
|---|---|---|---|---|
| S1 | Route this shipment: here is my origin, destination, cargo, and delivery deadline. Show me my options. | **P0** | `route_shipment` | Ocean Optimizer, Trucking Optimizer, Graph Generator, Transit Time Models, Rules Engine |
| S2 | Show me the cheapest option, the fastest option, and the most reliable option for this shipment — side by side. | **P0** | `route_shipment` (multi-objective output) | Same as S1 + Probabilistic Transit Model |
| S3 | Where is my shipment right now? | **P0** | `track_shipment` | AIS Adapter, Shipment State Store, Road Tracking Adapter |
| S4 | Will my shipment arrive on time? What is the probability it makes my delivery window? | **P0** | `check_on_time_risk` | Probabilistic Transit Model, Shipment State, Rolling Horizon ETA |
| S5 | My shipment is delayed / there's a disruption. What are my options? | **P0** | `reroute_shipment` | Rolling Horizon Controller, Ocean Optimizer, Trucking Optimizer |
| S6 | How long does it typically take to ship from [origin] to [destination] right now? | **P1** | `lane_transit_estimate` | Transit Time Model (historical + live signal), Lane Analytics |
| S7 | How much will it cost to ship this cargo from [origin] to [destination] by [deadline]? | **P1** | `route_shipment` (cost summary) | Rate Engine, Ocean Optimizer, Trucking Optimizer |
| S8 | Should I ship this by ocean or air given my delivery window? | **P1** | `mode_compare` | Ocean Optimizer, Air Optimizer (future), Transit Time Models, Rate Engine |
| S9 | I have 30 purchase orders this month. Give me a routing plan. | **P1** | `route_batch` | Batch planner wrapping Ocean + Trucking Optimizers |
| S10 | What happened to shipment [ID]? Give me the full history of events and decisions. | **P1** | `shipment_audit_trail` | Shipment Event Log, Decision Log |
| S11 | Generate booking instructions I can send to my freight forwarder. | **P1** | `generate_booking_instruction` | Route Formatter, Carrier/Port Reference Data |
| S12 | Which of my active shipments are currently at risk of being late? | **P1** | `portfolio_risk_scan` | Probabilistic Transit Model, Shipment State, Threshold Rules |
| S13 | How much have I spent on freight this month / quarter by lane, carrier, and mode? | **P2** | `freight_spend_analytics` | Analytics Engine, Shipment Cost Store |
| S14 | Which carriers are most reliable for my lanes? | **P2** | `carrier_scorecard` | Carrier Performance Analytics |
| S15 | What is my freight cost vs. market rate for my key lanes? | **P2** | `rate_benchmark` | Rate Benchmark Engine, Market Rate Feed |
| S16 | My freight forwarder quoted me $X for this lane. Is that reasonable? | **P2** | `rate_benchmark` | Rate Benchmark Engine |
| S17 | How much would I save if I consolidated these LCL shipments into an FCL? | **P2** | `consolidation_evaluate` | Consolidation Model, Rate Engine |
| S18 | What is the CO2 footprint of each routing option for this shipment? | **P2** | `emissions_estimate` | Emissions Model (mode + distance + cargo weight) |
| S19 | How does this route look if I route around the Red Sea? Show me the cost and time delta. | **P1** | `what_if_scenario` | Ocean Optimizer with lane avoidance constraint |
| S20 | What is my exposure if the Port of Los Angeles has a congestion event? | **P2** | `disruption_exposure` | Portfolio-level lane/port dependency scan |

**Note — LCL (Less-than-Container Load):** When a PO's volume is too small to justify a full container, it moves as LCL — consolidated with other cargo into a shared container by a freight forwarder or NVOCC. Routing LCL shipments requires a consolidation optimizer that decides which LCL shipments to group together and onto which sailings. This is a distinct problem from FCL routing and requires its own MILP formulation (bin-packing × routing). Deferred to a future phase — see Section 11 (Components) and Open Question 8.

---

### 3.2 Persona: Freight Forwarder — Operations Planner

**Role:** Works at a freight forwarder. Directly plans and executes shipments on behalf of multiple shipper clients. Manages a high-volume queue of shipments across multiple lanes, carriers, and modes simultaneously. Responsible for: booking carriers, meeting vessel cutoffs, resolving exceptions, managing carrier allocation commitments, coordinating trucking.

**Volume:** 100–500 shipments per day in queue. Queries are portfolio-centric — they think across the full book of business, not individual shipments.

**KPIs they own:** Carrier acceptance rate, cutoff compliance, exception resolution time, carrier allocation utilization, rollover rate.

**What they want to avoid:** Missing a vessel cutoff, overshooting carrier allocation caps, learning about a disruption after it's too late to recover.

#### Questions and Tasks

| # | Question / Task | Priority | MCP Tool | Underlying Models/Components |
|---|---|---|---|---|
| F1 | Route all unbooked shipments in my queue. | **P0** | `route_batch` | Batch planner, Ocean + Trucking Optimizers, Rules Engine |
| F2 | Route these shipments — these 10 are urgent (ASAP), route the rest for lowest cost. | **P0** | `route_batch` (priority segmentation) | Batch planner with priority flag |
| F3 | Which shipments have a vessel cutoff in the next 24 hours and aren't booked yet? | **P0** | `cutoff_alert` | Shipment State, Sailing Schedule Store, Cutoff Rules |
| F4 | Which of my active shipments are at risk right now? Rank by urgency. | **P0** | `portfolio_risk_scan` | Probabilistic Transit Model, Shipment State, Risk Scoring |
| F5 | Shipment [ID] was rolled / vessel delayed. Re-route it. | **P0** | `reroute_shipment` | Rolling Horizon Controller, Ocean Optimizer, Trucking Optimizer |
| F6 | What is the optimal carrier for this shipment given our current allocation commitments? | **P0** | `carrier_select` | Rules Engine, Carrier Allocation State, Rate Engine |
| F7 | Which of my shipments are affected by [disruption — port, vessel, carrier, weather]? | **P0** | `disruption_impact_scan` | Shipment State, Graph (port/lane/carrier lookup) |
| F8 | Are we within our carrier allocation caps for this month on this lane? | **P1** | `allocation_check` | Carrier Allocation State, Shipment Booking Store |
| F9 | Should I consolidate these LCL shipments into an FCL? Run the numbers. | **P1** | `consolidation_evaluate` | Consolidation Model, Rate Engine, Sailing Schedule |
| F10 | What ocean sailings are available for [lane] in the next 14 days? | **P1** | `schedule_query` | Sailing Schedule Store, Carrier Service Data |
| F11 | What trucking capacity is available from USLAX to [destination] next week? | **P1** | `trucking_availability` | Road Routing Adapter, Carrier Capacity Data |
| F12 | Plan the drayage and inland trucking for all containers arriving at USLAX next week. | **P1** | `trucking_plan` | Trucking Optimizer, Arrival Schedule, Graph (USLAX node data) |
| F13 | Generate pre-alerts for all shipments departing this week. | **P1** | `generate_pre_alert` | Shipment State, Route Formatter, Carrier/Port Reference |
| F14 | How much of our contracted capacity on [lane] have we used this month? | **P1** | `allocation_check` | Carrier Allocation State |
| F15 | Show me all shipments where the actual route deviated from our routing guide. | **P2** | `routing_compliance_scan` | Routing Guide Store, Shipment Booking Log, Rules Engine |
| F16 | What is the rollover rate for [carrier] on [lane] over the last 90 days? | **P2** | `carrier_scorecard` | Carrier Performance Analytics |
| F17 | Identify which shipments can be consolidated to reduce carrier count this week. | **P2** | `consolidation_evaluate` (portfolio) | Consolidation Model, Batch Optimizer |
| F18 | What are the transit time distributions for [carrier] on [lane] right now? | **P1** | `lane_transit_estimate` | Transit Time Model |
| F19 | This shipper's routing guide says use Carrier X for this lane, but Carrier X has no availability. What are the approved alternatives? | **P0** | `carrier_select` | Rules Engine (routing guide + fallback logic), Carrier Availability |
| F20 | Show me the full G(N,A) for [origin] → [destination] with current arc weights. | **P1** | `graph_visualize` | Graph Generator, Current Arc State |

---

### 3.3 Persona: Freight Forwarder — Business Analyst / Management

**Role:** Owns performance reporting, cost analytics, carrier contract negotiation support, and strategic lane decisions. Not in the daily execution flow. Runs analysis weekly/monthly to identify opportunities and problems.

**Volume:** Lower frequency, higher complexity queries. Think in aggregates — lanes, carriers, time periods — not individual shipments.

**KPIs they own:** Total freight spend, cost per TEU/kg, on-time delivery rate, carrier contract utilization, cost vs. market.

**What they want:** Answers to "where are we leaving money on the table?" and "which carriers are letting us down?"

#### Questions and Tasks

| # | Question / Task | Priority | MCP Tool | Underlying Models/Components |
|---|---|---|---|---|
| B1 | What is our total freight spend this month, broken down by lane, carrier, and mode? | **P0** | `freight_spend_analytics` | Analytics Engine, Shipment Cost Store |
| B2 | Which carriers are underperforming on transit time vs. their committed SLA? | **P0** | `carrier_scorecard` | Carrier Performance Analytics, SLA Benchmark Store |
| B3 | What is our on-time delivery rate by carrier, lane, and mode this quarter? | **P0** | `otd_analytics` | Carrier Performance Analytics |
| B4 | Where are we routing inefficiently vs. market rates? | **P1** | `routing_efficiency_analysis` | Counterfactual Engine, Rate Benchmark, Shipment Cost Store |
| B5 | What is our carrier allocation utilization — are we meeting minimum volume commitments? | **P1** | `allocation_utilization` | Carrier Allocation State, Booking Volume Store |
| B6 | What has the Red Sea rerouting cost us in additional freight expense vs. pre-disruption? | **P1** | `disruption_cost_impact` | Counterfactual Engine, Lane Cost Delta, Shipment Store |
| B7 | If we had routed all our CNSHA→USLAX shipments last quarter using the optimal route in hindsight, how much would we have saved? | **P1** | `counterfactual_analysis` | Counterfactual / Regret Analysis Engine |
| B8 | Show me carrier scorecards for our top 10 carriers by volume. | **P1** | `carrier_scorecard` | Carrier Performance Analytics |
| B9 | Is the CNSHA→USLAX lane getting more expensive over the last 6 months? | **P2** | `lane_trend` | Lane Analytics, Historical Rate Store |
| B10 | How does our freight cost per TEU compare to market benchmarks on our top 5 lanes? | **P2** | `rate_benchmark` | Rate Benchmark Engine |
| B11 | What is our CO2 emissions footprint by lane and mode this quarter? | **P2** | `emissions_analytics` | Emissions Model, Shipment Store |
| B12 | Which of our carrier contracts are underutilized and at risk of minimum commitment penalties? | **P1** | `allocation_utilization` | Carrier Contract Store, Booking Volume |
| B13 | Build a carrier bid analysis: if we switched Carrier X to Carrier Y on [lane], what is the cost and service level impact? | **P2** | `what_if_scenario` (carrier swap) | Ocean Optimizer with carrier constraint, Rate Engine |
| B14 | What percentage of our exceptions last quarter were caused by carrier-side vs. port-side vs. weather events? | **P2** | `exception_root_cause_analytics` | Exception Log, Cause Classification Model |

---

### 3.4 Persona: Compliance Officer / Customs Broker

**Role:** Ensures all shipments comply with trade regulations, carrier contract terms, customs requirements, and internal routing policy. May be embedded at a freight forwarder or at a large shipper. Operates at the intersection of legal, operational, and financial risk.

**Volume:** Reviews shipments at booking time and flags exceptions. Also runs periodic compliance audits.

**KPIs they own:** Customs hold rate, compliance violation rate, freight audit recovery (overbilling caught), denied party clearance rate.

**What they want:** Catch problems before shipments move. Audit trails. Evidence that the system checked the rules.

#### Questions and Tasks

| # | Question / Task | Priority | MCP Tool | Underlying Models/Components |
|---|---|---|---|---|
| C1 | Does this routing comply with current trade regulations for this origin/destination/commodity? | **P0** | `trade_compliance_check` | Regulatory Rules Engine, Sanctions/Embargo DB, Commodity Classification |
| C2 | What documents are required for this shipment (commercial invoice, packing list, COO, phytosanitary, etc.)? | **P0** | `document_requirements` | Trade Lane Rules DB, Commodity Classification, Incoterm Rules |
| C3 | What is the HS code for this commodity and the applicable duty rate on this trade lane? | **P0** | `tariff_lookup` | HS Code DB, Tariff Schedule (HTS, TARIC), FTA eligibility rules |
| C4 | Are there any denied party, OFAC, or sanctions concerns for this shipper, consignee, or notify party? | **P1** | `denied_party_check` | Denied Party Screening DB (OFAC, BIS, EU sanctions lists) |
| C5 | Is there an active embargo or sanction affecting this routing or trade lane? | **P1** | `sanctions_check` | Sanctions/Embargo DB, Trade Lane Lookup |
| C6 | Flag all active shipments that are approaching a potential customs hold situation. | **P1** | `customs_hold_alert` | Shipment State, Compliance Rules, Hold Pattern Recognition |
| C7 | Does the actual freight invoice for this shipment match the planned routing cost? Flag overbilling. | **P1** | `freight_audit` | Planned Cost Store, Invoice Parser, Rate Engine |
| C8 | What are the dangerous goods (IMDG/IATA DGR) requirements for shipping this commodity? | **P1** | `hazmat_requirements` | DG Rules DB (IMDG class, packing group, segregation rules) |
| C9 | Review the routing decisions made this week — were all routing guide rules followed? | **P2** | `routing_compliance_scan` | Routing Guide Store, Shipment Booking Log, Rules Engine |
| C10 | Generate the required customs documentation package for this shipment. | **P2** | `customs_doc_generator` | Document Templates, Shipment Data, Tariff/Origin Rules |
| C11 | Which carrier contracts have clauses that restrict routing flexibility on specific lanes? | **P2** | `contract_compliance` | Carrier Contract Store, Lane/Clause Extractor |
| C12 | Did we correctly classify the country of origin for all shipments this month? | **P2** | `origin_classification_audit` | Origin Rules, Shipment Store, Classification Model |
| C13 | What is the landed cost for this shipment including duties, taxes, and freight? | **P1** | `landed_cost_estimate` | Tariff Lookup, Freight Cost, Duty/VAT Calculator |

---

### 3.5 Master MCP Tool Inventory

All tools derived from the persona requirements above, with priority, description, and underlying components. These are the tools exposed via the MCP server that the agent calls.

#### P0 Tools — Must-Have for Prototype

| Tool | Description | Underlying Components |
|---|---|---|
| `route_shipment` | Given a single shipment (origin, dest, cargo, constraints), return viable routes ranked by cost, transit time, and on-time probability. Supports single-objective and multi-objective output. | Ocean Optimizer, Trucking Optimizer, Graph Generator, Transit Time Models, Rules Engine |
| `route_batch` | Given N shipments with optional priority flags, return an optimized routing plan for the full portfolio. Supports priority segmentation (urgent vs. economical). | Batch Planner, Ocean + Trucking Optimizers, Rules Engine |
| `track_shipment` | Given a shipment ID, return current position (AIS for ocean, road ETA for trucking), milestone trace, and latest ETA. | AIS Adapter, Road Tracking Adapter, Shipment State Store |
| `check_on_time_risk` | Given a shipment ID, return P(arrival ≤ deadline), expected arrival distribution, and risk flags. | Probabilistic Transit Model, Shipment State, Rolling Horizon ETA |
| `portfolio_risk_scan` | Return all active shipments at risk of missing delivery window, ranked by urgency (time-to-failure × impact). | Probabilistic Transit Model, Shipment State, Risk Scoring Engine |
| `reroute_shipment` | Given a shipment ID and an exception event (rollover, delay, disruption), re-plan the remaining legs and return options. | Rolling Horizon Controller, Ocean Optimizer, Trucking Optimizer |
| `carrier_select` | Given a shipment and current allocation state, return the optimal carrier per routing guide + allocation cap + availability constraints. | Rules Engine, Carrier Allocation State, Rate Engine, Carrier Availability |
| `disruption_impact_scan` | Given a disruption (port, vessel, carrier, weather event), return all active shipments affected with impact severity. | Shipment State, Graph (lane/port/carrier dependency lookup) |
| `trade_compliance_check` | Given a routing and shipment details, check against trade regulations, sanctions, embargoes, and carrier restrictions. Return pass/fail per rule with evidence. | Regulatory Rules Engine, Sanctions/Embargo DB, Commodity Classification |
| `document_requirements` | Given a shipment (origin, dest, commodity, incoterm), return the required documentation set. | Trade Lane Rules DB, Commodity Classification, Incoterm Rules |
| `tariff_lookup` | Given commodity and trade lane, return HS code, applicable duty rate, and FTA eligibility. | HS Code DB, Tariff Schedule |
| `cutoff_alert` | Return all shipments with a vessel booking cutoff within a configurable window that are not yet confirmed booked. | Shipment State, Sailing Schedule Store, Cutoff Rules |
| `freight_spend_analytics` | Return aggregated freight spend sliced by lane, carrier, mode, and time period. | Analytics Engine, Shipment Cost Store |
| `carrier_scorecard` | Return performance metrics for one or more carriers: on-time rate, transit time vs. SLA, rollover rate, acceptance rate. | Carrier Performance Analytics Engine |
| `otd_analytics` | Return on-time delivery rate by carrier, lane, and mode for a specified period. | Carrier Performance Analytics Engine |

#### P1 Tools — Built in Phase 5

| Tool | Description | Underlying Components |
|---|---|---|
| `lane_transit_estimate` | Return the current transit time distribution (mean, std dev, P50/P90) for a lane, mode, and carrier. | Transit Time Model, Lane Analytics |
| `mode_compare` | Given a shipment, return a side-by-side comparison of ocean vs. air (vs. combined) on cost, transit time, and on-time probability. | Ocean + Air Optimizers, Rate Engine, Transit Time Models |
| `what_if_scenario` | Given a route and a parameterized scenario (port closure, carrier unavailable, rate change, lane avoidance), return the re-optimized route and cost/time delta. | Ocean Optimizer with constraint injection, Rate Engine |
| `allocation_check` | Return current usage vs. committed minimum/maximum for a carrier on a lane or across all lanes. | Carrier Allocation State, Booking Volume Store |
| `allocation_utilization` | Return carrier allocation utilization summary across all contracts for a period. | Carrier Allocation State, Booking Volume Store |
| `consolidation_evaluate` | Given a set of LCL shipments on the same lane, evaluate whether FCL consolidation is cost-effective and which sailings support it. | Consolidation Model, Rate Engine, Sailing Schedule |
| `schedule_query` | Return available ocean sailings for a lane within a date range, with carrier, vessel, ETD, ETA, and available capacity indicator. | Sailing Schedule Store, Carrier Service Data |
| `trucking_plan` | Given a set of containers arriving at a port within a window, generate an optimal drayage and inland trucking plan. | Trucking Optimizer, Graph (port node data — BNSF/UP ramps, road distances), Road Routing Adapter |
| `trucking_availability` | Return available trucking capacity (carriers, rates, transit time) between two points for a date window. | Carrier Availability, Road Routing Adapter |
| `graph_visualize` | Return the G(N,A) subgraph between two nodes with current arc weights for visualization. | Graph Generator, Current Arc State |
| `routing_efficiency_analysis` | Identify shipments where actual routing cost deviated significantly from optimal, ranked by opportunity size. | Counterfactual Engine, Rate Benchmark, Shipment Cost Store |
| `disruption_cost_impact` | Given a disruption event and date range, compute the total incremental cost to the portfolio vs. pre-disruption baseline. | Counterfactual Engine, Lane Cost Delta, Shipment Store |
| `counterfactual_analysis` | For completed shipments, compute regret: what would the cost/transit have been under the best available route in hindsight? | Counterfactual / Regret Analysis Engine |
| `landed_cost_estimate` | Given a shipment and routing, return total landed cost: freight + duties + taxes + handling. | Tariff Lookup, Freight Cost, Duty/VAT Calculator |
| `denied_party_check` | Screen a shipper, consignee, or notify party against denied party and sanctions lists. | Denied Party Screening DB (OFAC, BIS, EU sanctions) |
| `sanctions_check` | Check whether a trade lane, port, or counterparty is subject to active embargo or sanction. | Sanctions/Embargo DB |
| `customs_hold_alert` | Return active shipments that match patterns associated with likely customs holds (missing docs, restricted commodities, flagged consignee). | Shipment State, Compliance Rules, Hold Pattern Model |
| `freight_audit` | Compare actual carrier invoice against planned routing cost; flag discrepancies above threshold. | Planned Cost Store, Invoice Data, Rate Engine |
| `hazmat_requirements` | Given a commodity, return IMDG class, packing group, segregation requirements, and mode-specific DG rules. | DG Rules DB |
| `shipment_audit_trail` | Return the full event and decision history for a shipment — every milestone, every routing decision, every override. | Shipment Event Log, Decision Log |
| `generate_pre_alert` | Generate a pre-alert notification for a shipment (to notify consignee, customs broker, or inland carrier of inbound shipment). | Shipment State, Route Formatter, Carrier/Port Reference |
| `generate_booking_instruction` | Generate a formatted booking instruction a shipper can send to a freight forwarder. | Route Formatter, Carrier/Port Reference Data |

#### P2 Tools — Deferred

| Tool | Description |
|---|---|
| `rate_benchmark` | Market rate benchmark for a lane, mode, and period vs. internal actuals. |
| `lane_trend` | Historical rate and transit time trend for a lane over a specified period. |
| `emissions_estimate` | CO2 estimate per route option (mode + distance + vessel/truck type). |
| `emissions_analytics` | Aggregate emissions by lane, mode, carrier for a period. |
| `routing_compliance_scan` | Scan a time period's bookings for deviations from routing guide rules. |
| `exception_root_cause_analytics` | Classify exceptions by root cause (carrier, port, weather, documentation) and aggregate. |
| `customs_doc_generator` | Generate a full customs documentation package for a shipment. |
| `contract_compliance` | Extract and check carrier contract clauses that constrain routing decisions. |
| `origin_classification_audit` | Audit country-of-origin classification across a period's shipments. |
| `what_if_carrier_swap` | Model cost and service impact of switching a carrier on a lane across historical volume. |
| `capacity_planning` | Pre-booking capacity recommendation given demand forecast and current market rates. |
| `disruption_exposure` | Portfolio-level scan of dependency on a specific port, lane, or carrier. |

---

### 3.6 P0 Priority Summary

The 15 P0 tools cover the daily operational core across all four personas. Building only these tools delivers a working system for freight forwarder operations planners and shippers. Everything else is additive.

**Routing core (3 tools):** `route_shipment`, `route_batch`, `carrier_select`

**Tracking and risk core (3 tools):** `track_shipment`, `check_on_time_risk`, `portfolio_risk_scan`

**Exception management core (2 tools):** `reroute_shipment`, `disruption_impact_scan`

**Operations workflow core (2 tools):** `cutoff_alert`, `routing guide + fallback in `carrier_select``

**Analytics core (3 tools):** `freight_spend_analytics`, `carrier_scorecard`, `otd_analytics`

**Compliance core (3 tools):** `trade_compliance_check`, `document_requirements`, `tariff_lookup`

---

## 4. Modes in Scope

**Prototype:** Ocean + Trucking (pre-carriage to origin port, drayage from destination port, inland trucking to final destination)

**Deferred:** Air, Rail

The ocean + trucking pair forces mode-transition handling (the hard part) from day one and covers the dominant volume pattern for cross-border freight.

---

## 5. Key Architectural Concept: Rolling Horizon Planning

*This is a first-principles design decision that differentiates this system from all existing TMS platforms.*

### 5.1 The Problem with Sequential Leg Planning

All existing TMS systems plan each mode leg sequentially: book the ocean leg, estimate the inland leg, execute. The inland leg estimate is made with high uncertainty at booking time (unknown exact arrival time, unknown port clearance delay, no specific carrier committed). By the time the vessel arrives, that estimate is stale but the planning system doesn't re-optimize.

### 5.2 Rolling Horizon Approach

This system maintains a **complete end-to-end plan at all times**, but resolves each leg at different levels of graph resolution depending on how close we are to execution:

- **G_coarse**: A sparse graph used for future legs. Arc weights are cost/time envelopes — ranges derived from historical data and ML models. Sufficient for the optimization objective but not for execution.
- **G_fine**: A dense graph used for the next leg to be executed. Arc weights are actual carrier schedules, confirmed spot rates, and port-specific clearance estimates.

As a shipment advances and uncertainty about the next leg decreases (e.g., AIS-derived vessel ETA confidence exceeds a threshold), the system fires a **re-plan trigger** and re-solves the next leg on G_fine with real schedules and live rates.

### 5.3 Concrete Example

**At booking time (T=0):**
- Ocean leg: precisely committed — specific carrier, vessel, sailing date, scheduled ETA ±3 days
- Inland leg (USLAX → Phoenix): rough envelope — $400–900, 1–3 days, no specific carrier

**Re-plan trigger fires when:** vessel is 72h from USLAX, AIS-derived ETA confidence > 90%, port clearance window confirmed

**At T=2 (vessel near USLAX):**
- Ocean leg: confirmed — actual ETA Jul 4 06:00 PST, port clearance est. Jul 4 14:00
- Inland leg: re-solved on G_fine — 3 specific options with actual carrier schedules and spot rates:
  - [A] Direct Truck (OHL) — depart Jul 4 16:00, arrive Jul 5 09:00 — $612 ✓ selected
  - [B] Drayage + Rail — depart Jul 4 20:00, arrive Jul 6 08:00 — $389
  - [C] Expedite (FedEx CC) — depart Jul 4 14:30, arrive Jul 4 22:00 — $1,480

See diagram: `docs/rolling_horizon_planning.drawio`

### 5.4 Design Principle

The optimizer holds a full door-to-door plan at all times. Future legs are placeholders on G_coarse — enough to make good booking decisions. As each leg's execution window approaches, it is re-solved on G_fine with real data. This is **Model Predictive Control applied to multimodal freight routing**.

**This is a hard requirement.** The system must maintain and continuously update a full multi-horizon plan. Single-horizon planning (plan once, execute) is not acceptable.

---

## 6. Supply and Demand Model

### 6.1 Demand — Ocean Shipments

Each ocean shipment request provides:

| Field | Description |
|---|---|
| `shipment_id` | Unique identifier |
| `origin` | City or address (geocoded to nearest port/pickup zone) |
| `destination` | City or address (geocoded to nearest port/delivery zone) |
| `cargo_ready_date` | Earliest pickup date at origin |
| `pickup_window` | [earliest, latest] pickup datetime |
| `required_delivery` | Latest acceptable arrival datetime at destination |
| `weight_kg` | Gross weight |
| `volume_cbm` | Volume in cubic meters |
| `cargo_type` | General, hazmat class, temperature-controlled, OOG |
| `hs_code` | 6-digit Harmonized System commodity code — determines customs inspection risk tier (P1) and tariff lookup. MVP: single code per shipment. **P1:** replace with list of HS codes to support mixed-product FCL bookings; inspection risk = max tier across codes. |
| `incoterm` | EXW, FOB, CIF, DDP, etc. |
| `service_level` | Economy, Standard, Express |
| `carrier_preferences` | Preferred, acceptable, excluded carriers |
| `budget_cap` | Optional hard cost cap per shipment |

### 6.2 Demand — Trucking Shipments

Each trucking shipment request provides:

| Field | Description |
|---|---|
| `shipment_id` | Unique identifier |
| `origin` | Pickup address |
| `destination` | Delivery address |
| `pickup_window` | [earliest, latest] pickup datetime |
| `delivery_window` | [latest_arrival] (one-sided — hard deadline) or [earliest, latest] |
| `weight_kg` | Gross weight |
| `volume_cbm` | Volume |
| `pallet_count` | Number of pallets (for load planning) |
| `cargo_type` | General, hazmat, temperature-controlled |
| `service_level` | Standard, Expedite, Dedicated |
| `carrier_preferences` | Preferred, acceptable, excluded |

### 6.3 Supply — The Graph G(N, A)

The routing network is modeled as a directed graph G(N, A):

**Nodes N** represent physical locations and logical waypoints:
- Origin locations (factories, warehouses, pickup addresses)
- Origin ports (container terminals with sailing schedules)
- Transshipment ports (intermediate hub ports)
- Destination ports
- Inland distribution points (rail ramps, cross-docks, truck terminals)
- Final destinations (warehouses, DCs, delivery addresses)

**Arcs A** represent feasible connections between nodes:
- **Ocean arcs**: carrier service legs between port pairs, with scheduled departure/arrival, capacity, and rate
- **Pre-carriage arcs**: pickup truck legs from origin to origin port
- **Drayage arcs**: port-to-inland-hub truck legs
- **Inland trucking arcs**: point-to-point truck moves
- **Transshipment arcs**: inter-terminal transfers at hub ports (time and cost)

**Container types:** FEU (40'HC, ≈ 76 CBM usable, ≈ 26,500 kg payload) and TEU (20', ≈ 33 CBM usable, ≈ 24,000 kg payload). The optimizer selects the cost-minimizing mix per (commodity, sailing) pair. Minimum container count per shipment: `n_k = max(ceil(volume_cbm / 76), ceil(weight_kg / 26500))`, accounting for both volume and weight limits. Container mix pre-computation is described in the formal model (`model/ocean_fcl_routing.tex`, Section 4.6).

| Container type | Code | TEU slots | Usable volume (CBM) | Payload (kg) | Notes |
|---|---|---|---|---|---|
| 20' standard | TEU | 1 | 33 | 24,000 | — |
| 40' standard | FEU (std) | 2 | 67 | 26,500 | Deferred to P1 |
| 40' High Cube | FEU (HC) | 2 | 76 | 26,500 | MVP FEU type; dominant on TPEB |

**Arc attribute schemas by type:**

*Pre-carriage arcs (origin door → POL):*
| Attribute | Description |
|---|---|
| `cost` | Flat truck rate per move (not per CBM — FCL is a dedicated truck) |
| `transit_time_mean` | Mean days, origin door to terminal gate |
| `transit_time_sigma` | Std dev of transit time |
| `distance_km` | Road distance (informational; used by instance generator) |

*Ocean sailing arcs (POL → POD_arrival):*
| Attribute | Description |
|---|---|
| `etd` | Estimated time of departure from POL |
| `cy_cutoff` | **Latest** datetime a container may arrive at terminal for this sailing. Hard constraint on pre-carriage delivery. Typically ETD − 4 days. |
| `rate_per_feu` | Base ocean freight rate per 40'HC container (USD/FEU) |
| `rate_per_teu` | Base rate per 20' container (USD/TEU). Typically 0.63–0.83 × FEU rate — not 0.5× because TEU costs more per CBM to handle. |
| `baf` | Bunker adjustment factor per container (same fraction applied to FEU and TEU rates) |
| `thc_pol` | Terminal handling charge at origin port per container |
| `thc_pod` | Terminal handling charge at destination port per container |
| `capacity_teu` | Max TEU slots on this sailing (vessel-level cap). 1 FEU = 2 TEU slots; 1 TEU = 1 TEU slot. |
| `service_string` | Carrier service string this sailing belongs to (e.g., "MSC Tiger"). Ties to carrier allocation pool. |
| `alloc_period` | Monthly allocation period this ETD falls in (YYYY-MM). |
| `transit_time_mean` | Mean transit days, POL to POD |
| `transit_time_sigma` | Std dev of transit time |
| `carrier` | Carrier name (e.g., MSC, CMA CGM, COSCO) |
| `vessel` | Vessel name |

*Dwell arcs (POD_arrival → POD_exit):*

Each physical port of discharge (POL) is represented by **two nodes**: a POD_arrival node (vessel has arrived, container discharged) and a POD_exit node (customs cleared, ready for inland pickup). These are connected by a dwell arc that makes port unloading and customs clearance an explicit model element rather than a hidden constant.

| Attribute | Description | MVP value |
|---|---|---|
| `unload_mean` | Mean time from vessel arrival to container accessible in terminal | Port-specific constant |
| `clearance_mean` | Mean customs processing time, no exam | Port-specific constant |
| `total_dwell_mean` | `unload_mean + clearance_mean` | USLAX/USLGB: 3.5 days; USSEA: 2.5 days |
| `total_dwell_sigma` | Std dev of total dwell | USLAX/USLGB: 1.5 days; USSEA: 1.0 days |
| `free_days` | Port storage free time before demurrage applies | 5 days (standard) |

**P1 — Commodity-specific dwell model (deferred):** In P1, dwell arc weight will become a function of commodity and importer attributes: HS code inspection risk tier, importer C-TPAT certification status, country of origin, and consignee inspection history. Inspection probability model: `p_exam = clip(λ_base × ρ_HS × ρ_origin × (1 − δ_CTPAT), 0, 1)`. Expected clearance time = `t_no_exam + p_exam × t_exam`. This allows the model to differentiate dwell time by commodity type and shipper trustworthiness.

*Inland arcs (POD_exit → destination door):*
| Attribute | Description |
|---|---|
| `cost` | Flat rate per container move (one chassis per FEU for FCL drayage) |
| `transit_time_mean` | Mean days, POD exit to destination door |
| `transit_time_sigma` | Std dev of transit time |
| `mode` | FTL truck (solid arc in diagram) or intermodal rail (dashed arc) |
| `distance_km` | Road/rail distance (informational) |

**Transit time estimation from coordinates:** For all trucking arcs (pre-carriage and inland), transit time is estimated from node lat/lon using: road distance = Haversine distance × road factor (1.25 China, 1.20 US); transit days = road distance / average truck speed (600 km/day China, 800 km/day US). For ocean arcs: sailing distance = Haversine × 1.15 (Trans-Pacific lane factor); transit days = sailing distance / 600 km/day (≈ 14 knots average, inclusive of port approach and anchorage). Validation: SHA → USLAX geodesic ≈ 9,200 km → sailing distance ≈ 10,580 km → transit ≈ 17.6 days, within published carrier schedule range of 14–18 days.

**String-based carrier allocation capacity:** Ocean carriers sell capacity on *service strings* — fixed port-call loops (e.g., MSC Tiger TPEB: SHA → NGB → SZX → USLAX → USLGB). A freight forwarder's contracted block space agreement (BSA) covers the entire string, not a specific port pair. A booking on SHA→USLAX and a booking on NGB→USLGB on the same string in the same month both draw from the same allocated pool.

The optimizer tracks two capacity constraints per sailing:
1. **Vessel-level cap** (`capacity_teu`): maximum TEU slots on a specific sailing. Shared across all forwarders.
2. **String allocation cap**: the forwarder's remaining contracted block on this string in this monthly period (`rem(s,t) = alloc(s,t) − util(s,t)`). Current utilization (`util`) is an external state input read from the shipment state store before each solve.

This means routing decisions for two shipments on the same string in the same month are coupled — the optimizer must respect the joint allocation cap even when the shipments use different port pairs. See formal model Section 4.5 and constraint P.3 in `model/ocean_fcl_routing.tex`.

**P1 — Time-phased capacity release (deferred):** The MVP commits all remaining `rem(s,t)` in a single routing run. In P1, a demand forecast will drive capacity release in tranches across sequential routing batches, reserving space for future urgent shipments. See Open Question 9.

**Graph decomposition for batch solving:** When routing a batch of N shipments, the demand-supply graph can be decomposed before the MILP solve. Two shipments are independent if they share no feasible supply arcs — no common carrier service legs or carrier allocation pools. Independent subsets form disconnected components in the demand-supply intersection graph and are solved separately. This decomposition reduces MILP problem size and enables parallelism. The batch planner builds the commodity-supply bipartite graph, finds connected components, dispatches each independently, and merges results. Shipments that share a sailing or draw from the same allocation pool remain in the same component and are optimized jointly. Example: TPEB and FEWB commodities always decompose into independent subproblems (different strings, different ports, no shared supply). See formal model Section 9 in `model/ocean_fcl_routing.tex`.

---

## 7. Agent Capabilities

### 7.1 Core Routing and Planning

- Route a single shipment: return all viable options given constraints
- Lowest-cost routing with feasibility check against delivery window
- Fastest routing (minimize expected transit time)
- Reliability-optimized routing (maximize on-time probability using transit time distributions)
- Multi-objective: return Pareto frontier of cost vs. time vs. reliability
- Mode selection: ocean vs. trucking vs. combined multimodal
- Carrier selection within mode against routing guide and preference rules
- FCL vs. LCL decision (consolidation economics)
- Direct vs. transshipment routing
- Cargo-ready-to-vessel-cutoff feasibility check
- Rolling horizon re-plan trigger evaluation and execution

### 7.2 Constraint and Rule Handling

- Hard time windows: pickup window, latest-arrival delivery window
- Soft time windows: prefer within range, penalize violations in objective
- Service level tiers: Economy, Standard, Express
- Carrier preference / blacklist / allocation cap enforcement
- Port or lane avoidance (e.g., Red Sea, Panama congestion)
- Weight and volume constraints per leg and per vessel service
- Commodity restrictions: hazmat class, temperature-controlled, OOG
- Trade lane regulatory constraints
- Budget cap per shipment or per lane
- Dangerous goods and temperature segregation (cannot co-load)

### 7.3 Batch and Fleet Operations (Forwarder Ops)

- Route all unbooked shipments in a portfolio simultaneously
- Demand-supply graph decomposition: partition the batch into independent subproblems (shipments sharing no feasible supply arcs) before solving; each partition dispatched to the optimizer separately and results merged
- Priority segmentation: "route these N urgently, rest economical"
- Volume consolidation: identify which shipments can merge into one container
- Carrier allocation compliance: flag shipments violating contracted allocation caps
- Exception queue: surface shipments requiring human decision, ranked by urgency and impact
- Bulk re-routing: identify all active shipments affected by a specific disruption
- Portfolio status: how many shipments are at-risk vs. on-track right now?

### 7.4 Scenario Analysis and What-If

- What if I shift origin port?
- What if I accept N days more transit time — how much do I save?
- What if I upgrade from LCL to FCL?
- What if carrier X is unavailable on this lane?
- What if I split this shipment across two vessels?
- Red Sea avoidance: route via Cape of Good Hope, show full cost/time delta
- Air vs. ocean: full cost and time comparison for same shipment
- Service level upgrade cost: what does 5 days faster cost?
- Tariff/duty change impact: how does a new duty rate affect landed cost by route?
- Capacity constraint scenario: model what happens if a major port goes down

### 7.5 Disruption and Exception Management

- Detect predicted delay: weather, port congestion, vessel rollover, anchorage wait
- Alert with ranked recommended actions (rebook, reroute, notify customer)
- Autonomous rerouting recommendation on carrier failure or missed cutoff
- Port strike / closure contingency routing
- Vessel schedule change: recalculate all impacted shipments and options
- Customs hold detection and recommended next steps
- Missed pickup window recovery options ranked by cost and time impact
- Proactive risk scoring: which booked shipments are most at risk this week?
- Rolling horizon re-plan on disruption: re-solve fine graph with updated constraints

### 7.6 Tracking and Visibility

- Where is this shipment right now? (AIS position on ocean legs)
- Current ETA prediction (ML-based, not just carrier schedule)
- Full milestone trace: cargo ready → picked up → departed → transshipment → arrived → customs cleared → delivered
- Is this shipment on track or at risk? (vs. committed delivery window)
- How many days remain to delivery?
- Remaining legs with mode transitions
- All shipments at risk: portfolio-level exception view
- Filter active shipments by mode, carrier, lane, risk status

### 7.7 Analytics and Performance

- Cost breakdown by lane, carrier, mode, time period
- Transit time performance vs. committed SLA by carrier and lane
- On-time delivery rate by carrier, lane, mode
- Carrier scorecard: cost, reliability, rollover rate, on-time delivery
- Lane performance trends: is this lane getting more expensive or slower?
- Route explanation / audit trail: why was this specific route chosen?
- Savings attribution: how much did optimization save vs. default routing?
- Counterfactual / regret analysis: what would this shipment have cost if we had chosen differently? (see Section 10.2)
- Carrier volume commitment utilization: am I meeting minimum contracted volumes?
- Emissions estimate: CO₂ per route option

### 7.8 Advisory and Decision Support

- Is this quote from my forwarder reasonable vs. market?
- What are the most reliable carriers for this lane?
- Am I using my carrier allocations efficiently?
- What is my exposure if [port / lane / carrier] goes down?
- Should I pre-book capacity given current demand signals?
- What is the market benchmark rate for this lane?

---

## 8. Agent Architecture

### 8.1 Framework: LangGraph

**Decision: LangGraph, not direct Anthropic SDK.**

This reverses the initial CLAUDE.md decision. The research is unambiguous for a system with 6+ agent personas:

| Criterion | LangGraph | Direct Anthropic SDK |
|---|---|---|
| Behavioral control | High — explicit graph state, conditional edges | High — but you build everything |
| Decision logging | LangSmith — best-in-class, zero-build | Build it yourself |
| MCP integration | `langchain-mcp-adapters` — production-tested | Native Anthropic SDK |
| Model swappability | Yes — model-agnostic | No — Claude-only |
| Planner-validator pattern | Native supervisor with conditional edges | Custom build |
| Human-in-the-loop | First-class `interrupt()` + PostgreSQL checkpointer | Build it yourself |
| Debuggability | Time-travel debugging in LangSmith | Build it yourself |
| Production maturity | Tier 1 — thousands of production deployments | Tier 3 — newer |

**Why this matters for this system specifically:**
- We need full logging of every agent decision (CLAUDE.md requirement) — LangSmith provides this with zero custom code
- We have 6+ agent personas with branching control flow — managing this in a hand-rolled loop is fragile
- We need model-agnosticism — if we want to run the Execution Monitor on a cheaper/faster model than the Routing Planner, LangGraph handles this cleanly
- The planner-validator supervisor pattern is a native LangGraph construct; building it from scratch on the Anthropic SDK would reproduce 80% of LangGraph's core

CLAUDE.md tech stack will be updated to reflect this change.

### 8.2 Architecture Pattern: Hierarchical with Hub-and-Spoke Leaves

```
                    ┌─────────────────────┐
                    │  Top-Level Router   │
                    │   (Orchestrator)    │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
  ┌───────────────────────┐       ┌─────────────────────────┐
  │  Planning Supervisor  │       │  Operations Supervisor  │
  └───────────┬───────────┘       └────────────┬────────────┘
              │                                │
       ┌──────┴──────┐                ┌────────┴────────┐
       ▼             ▼                ▼                 ▼
  ┌─────────┐  ┌──────────┐   ┌───────────┐   ┌────────────────┐
  │Routing  │  │Compliance│   │Execution  │   │Market          │
  │Planner  │→ │Validator │   │Monitor    │   │Intelligence    │
  └─────────┘  └──────────┘   │(event-    │   │(scheduled +    │
                               │driven)    │   │on-demand)      │
                               └───────────┘   └────────────────┘
```

The Planning Supervisor runs the planner → validator loop for every routing request. The Operations Supervisor manages the two background agents that run continuously. The Top-Level Router receives incoming requests and dispatches to the appropriate supervisor.

### 8.3 Agent Personas

#### Routing Planner Agent
**Responsibility:** Given a shipment request or batch of requests, call the optimization and ML tools to generate end-to-end route recommendations with supporting rationale.

**Tools available:** Graph generator, ocean optimizer, trucking optimizer, transit time models, rolling horizon controller, rules engine, AIS adapter, road routing adapter, rate lookup

**Output:** Structured route recommendation: `{route: [legs], total_cost, expected_transit_days, p_on_time, constraints_checked: [list], rationale: str}`

**Does not have access to:** Carrier booking APIs (writes), external notification systems

#### Compliance/Validation Agent
**Responsibility:** Independently review the Routing Planner's output against rules, regulations, business policy, and logical consistency. Not a peer reviewer — a skeptical auditor.

**Tools available:** Carrier restriction database, business rules engine, rate benchmark lookup, regulatory constraint checker. **No access to the optimization solver** — the validator cannot re-run the planner's work, only verify it.

**Independence guarantees (anti-sycophancy):**
1. Receives only the planner's structured output — not the planner's conversation history or chain of thought
2. Uses a different system prompt, explicitly framed as "find problems, not confirm correctness"
3. Must address every item on a defined checklist — cannot skip items
4. Consider running a different model from the planner (e.g., different Claude version or temperature setting) to reduce agreement bias

**Validation checklist (mandatory, per-item structured output):**
- Carrier constraint compliance: are all carriers on this route permitted for this lane, cargo type, and date?
- Time window feasibility: can the recommended route physically meet the pickup and delivery windows given the transit time distributions?
- Weight and volume compliance: do cargo specs fit within capacity on each leg?
- Business rule compliance: does this route conform to routing guide, allocation caps, and customer-specific rules?
- Optimization sanity check: is total cost within expected bounds for this lane and service level?
- Regulatory check: are there trade lane restrictions, embargoes, or hazmat rules that apply?

**Output:** `{status: PASS|FAIL|ESCALATE, findings: [{rule: str, status: PASS|FAIL, evidence: str}], summary: str}`

**Three-outcome logic:**
- `PASS`: all checklist items pass — recommendation forwarded to user
- `FAIL`: one or more items fail — recommendation returned to planner for revision (max 3 revision cycles, then auto-ESCALATE)
- `ESCALATE`: validator detects a conflict it cannot resolve (e.g., delivery window requires carrier X but carrier X has an active restriction) — human review required via `interrupt()`

#### Execution Monitor Agent
**Responsibility:** Continuously watches active shipments against their planned itineraries. Detects exceptions (delay, rollover, customs hold, missed pickup), fires rolling horizon re-plan triggers, and generates proactive alerts with recommended actions.

**Operates:** Event-driven (subscribes to AIS updates, EDI 214 events, carrier APIs) + polling for shipments approaching re-plan trigger thresholds. Runs asynchronously — not in the request/response path.

**Tools available:** AIS adapter, shipment state store, rolling horizon controller, routing planner (can request a re-plan), alert dispatch

**Output:** Exception alerts with ranked recommended actions; re-plan triggers with updated state context.

#### Market Intelligence Agent
**Responsibility:** Maintains a live view of market conditions — spot rate indices, carrier capacity signals, port congestion alerts, weather disruptions, geopolitical events. Provides context to the Routing Planner on demand and pushes alerts when conditions change materially.

**Operates:** Scheduled refresh (configurable interval) + on-demand query from Routing Planner.

**Tools available:** Rate index feeds, port congestion APIs, weather/disruption feeds, news monitoring

**Output:** `{market_context: {lane: {spot_rate_range, capacity_signal, disruption_alerts}}}` — queryable by the Routing Planner as a tool call.

#### Future Personas (Deferred)
- **Customer Communication Agent** — translates routing decisions and exceptions into shipper-facing language, manages notification drafts. No write access to booking systems.
- **Scenario / What-If Agent** — stress-tests approved routes against parameterized scenarios. Used in planning mode, not real-time.

### 8.4 Human-in-the-Loop Checkpoints

LangGraph `interrupt()` fires at the following points, pausing execution for human review:

1. **Validator returns ESCALATE** — recommendation has an unresolvable conflict
2. **Planner-validator loop reaches 3 revision cycles without PASS** — auto-escalate
3. **Any booking action** (future, when autonomous execution is enabled) — all writes to carrier systems require explicit human approval until the system is validated at scale
4. **Exception requires rerouting a high-value or time-critical shipment** — configurable threshold (e.g., shipments with &lt;24h delivery window remaining)

State is persisted via PostgreSQL checkpointer — human can resume the workflow after reviewing without losing context.

### 8.5 Logging

All agent interactions are logged via LangSmith. Every tool call, every state transition, every agent input and output is captured with timestamp. This satisfies the CLAUDE.md requirement for agent interaction logging.

Additionally, the agent interaction log (`logs/agent_interactions.jsonl`) captures the user-facing query/response pairs for capability extension analysis — separate from the internal LangSmith trace.

### 8.6 Production Failure Mode Mitigations

| Failure Mode | Mitigation |
|---|---|
| Infinite planner-validator loop | Max 3 revision cycles, then auto-ESCALATE |
| Sycophantic validator | Independent tool access, explicit skeptic framing, PASS rate monitoring |
| Context loss on agent handoff | Typed Pydantic schemas at every boundary; pass only what downstream needs |
| Concurrent booking conflicts | Optimistic locking on carrier capacity state; serialize writes through a single stateful node |
| Latency cascade | Execution Monitor and Market Intelligence run async — never in the request path |
| Model version drift | Version-pin model + system prompt pairs; test planner-validator pair as a unit before deployment |
| Validation theater | Monitor PASS rate in production; >90% PASS rate without findings triggers review |

---

## 9. Data Sources

All data sources are either real network topology or synthetic commercial parameters. Data provenance is documented at the point of use in code.

| Source | Type | Use | License |
|---|---|---|---|
| UN/LOCODE | Real topology | Port and location reference | Free |
| IATA codes | Real topology | Airport reference | Free |
| NOAA AIS (historical) | Real signal | Ocean vessel tracking, transit time training | Free |
| AIS live feed (TBD) | Real signal | Active shipment tracking | Paid (future) |
| Google Maps Routes API | Real signal | Road transit time estimation | Pay-per-use |
| OSRM | Real signal | Road routing (free alternative) | Free |
| OpenSky Network | Real signal | Air freight schedules (historical) | Free |
| Ocean carrier schedules | Real topology | Sailing schedule graph construction | Public / scraped |
| BTS Freight Analysis Framework | Real topology | Trucking lane structure | Free |
| Synthetic rates | Synthetic | Commercial rate parameters for optimization | N/A |
| DAT | — | **NOT licensed. Do not use without explicit license.** | — |
| USLAX / port authority data | Real signal | Terminal throughput, berth schedules, typical clearance windows by terminal | Free / public |
| BNSF / UP intermodal ramp data | Real topology | Inland ramp locations, service days, cutoff times | Public |
| Google Maps Distance Matrix | Real signal | Road distance and historical transit time between inland nodes | Pay-per-use |
| NOAA / NWS weather | Real signal | Weather disruption risk by port and lane | Free |

---

## 10. Differentiation Requirements

These capabilities do not exist in any current TMS platform. They are explicit design requirements, not nice-to-haves.

### 10.1 Continuous Re-Optimization (Rolling Horizon)

Requirement stated in Section 5. The system must maintain and continuously update a full multi-horizon plan. Batch-wave planning is not acceptable.

### 10.2 Counterfactual / Regret Analysis

After each shipment completes, the system must compute:
- The set of routes that were feasible at booking time (given constraints known at T=0)
- The actual outcome for each route (using realized transit times, not estimates)
- Regret = `|cost(chosen_route) - cost(best_feasible_route_in_hindsight)|`

This data must be stored per shipment and aggregable by lane, carrier, time period, and cargo type. Used for model validation, systematic bias detection, and training data generation.

### 10.3 Learned Constraint Inference

When a human planner overrides a system routing recommendation, the system must log:
- What was recommended (route, carrier, cost, transit time)
- What was chosen instead
- Timestamp and shipment context (lane, cargo type, service level, date)
- Override reason (if provided by planner)

These override signals are the primary input for constraint learning. Over time, systematic overrides on a lane or carrier reveal implicit constraints not yet modeled. The logged data feeds into both rule extraction (explicit constraint updates) and model retraining (implicit preference learning).

### 10.4 Simultaneous Multi-Echelon Joint Optimization

All existing TMS platforms plan mode legs sequentially. This system must model the full door-to-door journey as a **single optimization problem** on the unified graph G(N, A). The ocean leg, transshipment, drayage, and inland trucking leg are all decision variables in one formulation — not planned one at a time.

The rolling horizon architecture (Section 5) implements this: G_coarse enables joint optimization at booking time with uncertainty; G_fine enables high-resolution optimization at execution time.

### 10.5 Spot Capacity as Supply

Spot market capacity (broker capacity, load board postings) is modeled as a set of arcs in G(N, A) alongside contracted capacity. Spot arcs carry rate distributions rather than fixed rates. The optimizer treats spot as just another supply option — not a fallback of last resort.

### 10.6 Probabilistic Planning

Transit time on each arc is represented as a distribution (mean + variance, or parametric fit from historical data), not a point estimate. The optimization objective must support:
- Expected cost / expected transit time
- Probability of on-time delivery given a delivery window (P(arrival ≤ deadline))
- Risk-adjusted objectives (e.g., minimize cost subject to P(on-time) ≥ 0.95)

This is the formal foundation for reliability-optimized routing (Section 7.1).

### 10.7 Fully Autonomous Decision Chains (End Goal)

The prototype implements decision-support (human approves). The end goal is a fully autonomous planning and exception-management chain: the system detects a disruption, re-optimizes affected shipments, selects the best alternative, books it (via carrier API), and notifies stakeholders — without human intervention. A compliance/validation agent checks planning decisions before execution.

---

## 11. Components Inventory

Each component is independently buildable and testable. No stitching until each component passes isolation tests.

| Component | Description | Mode(s) |
|---|---|---|
| Graph Generator | Constructs G(N, A) from network data sources. Each node is enriched with public real-world data: port nodes get terminal throughput, typical customs clearance windows, and anchorage wait distributions; city/inland nodes get intermodal ramp locations (e.g. BNSF, UP), road distance matrices, and historical road transit time distributions. Arc weights are derived from this node-level enrichment, not left as abstract estimates. | Ocean + Trucking |
| Ocean Transit Time Model | ML model: distribution over transit time per ocean arc | Ocean |
| Trucking Transit Time Model | ML model: distribution over transit time per trucking arc | Trucking |
| Ocean Optimizer | MILP: Binary Multi-Commodity Network Flow. Selects optimal ocean route for a batch of commodities given demand, vessel capacity, and string-based carrier allocation constraints. Builds commodity-specific subgraphs (only arcs reachable on a complete origin→destination path), pre-computes optimal FEU/TEU container mix per (commodity, sailing), decomposes independent subproblems before solving, and returns structured infeasibility reports for commodities with no feasible path. Formal model: `model/ocean_fcl_routing.tex`. | Ocean |
| Trucking Optimizer | MILP: selects optimal trucking route given demand and constraints | Trucking |
| Multimodal Stitching Layer | Assembles mode-specific solutions into a coherent end-to-end plan | Both |
| Rolling Horizon Controller | Monitors shipment state, fires re-plan triggers, manages G_coarse/G_fine resolution | Both |
| Rules Engine | Evaluates routing guide compliance, carrier restrictions, business rules | Both |
| AIS Tracking Adapter | Ingests AIS data, maps vessel positions to shipment milestones, produces ETA updates | Ocean |
| Road Routing Adapter | Calls Google Maps / OSRM for road transit time and distance | Trucking |
| MCP Server | Exposes all components as MCP tools callable by the agent layer | Both |
| Planning Agent | LLM agent that orchestrates tools to answer routing queries | Both |
| Validation Agent | Reviews planning decisions before surfacing to user | Both |
| Execution Monitor Agent | Watches active shipments, detects exceptions, triggers re-plans | Both |
| Agent Interaction Logger | Logs all agent queries and responses with timestamp | Both |
| LCL Consolidation Optimizer | MILP: given a set of LCL shipments on the same or nearby lanes, determine optimal grouping into shared containers and assignment to sailings. Combines bin-packing (CBM/weight fit) with routing optimization. Distinct from FCL ocean optimizer — requires NVOCC consolidation schedules and LCL rate data. | Ocean (deferred) |
| Instance Generator | Produces synthetic but geographically realistic problem instances for solver testing and model validation. Uses UN/LOCODE port coordinates, GeoNames city geocoding, public sailing schedule data, and Haversine-based transit time estimation. Outputs structured JSON instances matching the model parameter schema. Required before any solver benchmarking. Build is a joint session — do not implement independently. | Ocean + Trucking |
| Customs Inspection Model | **P1.** Commodity-specific dwell time model for POD nodes. Computes inspection probability as a function of HS code risk tier, importer C-TPAT status, country of origin, and consignee inspection history. Feeds into dwell arc weights in the Ocean Optimizer. Deferred from MVP — MVP uses fixed port-level dwell constants (USLAX/USLGB: 3.5 days; USSEA: 2.5 days). | Ocean (P1) |

---

## 12. Build Sequence

Phases are gates. Each phase requires explicit approval before the next begins.

**Phase 0 — PRD** ← CURRENT  
**Phase 1 — Formal Models (LaTeX)**  
One model per component. Each approved individually before code starts.

**Phase 2 — Component Builds**  
Order: Graph Generator → Transit Time Models → Mode Optimizers → Rules Engine → Adapters → Stitching Layer → Rolling Horizon Controller

**Phase 3 — MCP Server**  
Expose all verified components as tools.

**Phase 4 — Agent Layer**  
Planning Agent → Validation Agent → Execution Monitor

**Phase 5 — Integration and End-to-End Testing**

**Phase 6 — Iterate**  
Add air mode, improve models, extend agent capabilities.

---

## 13. Open Questions and Future Decisions

1. **Decision-support vs. autonomous execution**: Prototype is decision-support (human approves). Define the trigger conditions and safety checks required before moving to autonomous execution. *(Key commercial and liability decision — defer until prototype is validated with design partners.)*

2. **Design partner selection**: Who are the first 2–3 customers? Freight forwarder vs. shipper? What data do they bring? What lanes do we start with?

3. **Live AIS feed**: NOAA historical is sufficient for model training. For production tracking we need a live feed. Evaluate MarineTraffic, VesselFinder, SpireGlobal on cost vs. coverage vs. API quality.

4. **Carrier booking APIs**: When we move toward autonomous execution, we need carrier API integrations for booking. Who are the first ocean carriers? (MSC, CMA CGM, COSCO are the volume leaders — start with one.) Which trucking carriers for drayage and inland?

5. **Pricing model**: Per-shipment, per-decision, or monthly subscription with volume tiers? What is the per-shipment cost floor given compute (MILP solve + LLM call per routing decision)?

6. **Emissions optimization**: Carbon as a routing objective requires accurate emissions factors per mode, carrier, and vessel. Data source TBD.

7. **Multi-agent framework**: LangGraph (decided — see Section 8). LangSmith for observability, PostgreSQL checkpointer for HITL state persistence.

8. **LCL consolidation optimizer**: LCL routing requires a consolidation layer that groups LCL shipments into containers before routing. This is a combined bin-packing + routing MILP — distinct from the FCL ocean optimizer. Requires NVOCC consolidation schedules and LCL rate data. Design and scope TBD for a future phase.

9. **Time-phased carrier capacity release**: The MVP commits all of `rem(s,t)` (remaining contracted block on a string in a period) in a single routing run. A production forwarder cannot do this — future urgent shipments need reserved headroom. The correct approach is to release capacity in tranches across sequential routing batches, driven by a demand forecast model. This requires: (a) a rolling demand forecast by trade lane and period, (b) a capacity allocation policy (hold-back fraction), (c) integration with the routing batch scheduler. Deferred to P1; the string allocation constraint structure is already in place to support it.

---

## Appendix A: Full Agent Capability Inventory

*Complete list of 60+ capabilities across 8 categories. In-scope items are implemented in the prototype. Deferred items are documented for future phases.*

### A.1 Core Routing and Planning (In Scope)
- Route single shipment: all viable options
- Lowest-cost routing
- Fastest routing
- Reliability-optimized routing (probabilistic)
- Multi-objective Pareto frontier (cost / time / reliability)
- Mode selection: ocean vs. trucking vs. combined
- Carrier selection within mode
- FCL vs. LCL consolidation decision
- Direct vs. transshipment optimization
- Cargo-ready-to-cutoff feasibility check
- Multi-stop / relay routing

### A.2 Constraint Handling (In Scope)
- Hard and soft time windows (pickup and delivery)
- Service level tiers
- Carrier preference / blacklist / allocation caps
- Port / lane avoidance
- Weight and volume constraints
- Commodity restrictions (hazmat, temperature, OOG)
- Trade lane regulatory constraints
- Budget caps
- Dangerous goods / temperature segregation

### A.3 Batch Fleet Operations (In Scope)
- Route full portfolio simultaneously
- Priority segmentation
- Volume consolidation identification
- Carrier allocation compliance monitoring
- Exception queue with urgency ranking
- Bulk re-routing on disruption
- Portfolio risk status

### A.4 Scenario Analysis (In Scope)
- Origin port shift
- Transit time vs. cost tradeoff
- LCL vs. FCL upgrade
- Carrier unavailability
- Shipment splitting
- Red Sea avoidance / Cape of Good Hope routing
- Air vs. ocean comparison
- Service level upgrade cost
- Tariff / duty change impact
- Port closure contingency

### A.5 Disruption and Exception Management (In Scope)
- Predicted delay detection
- Ranked recommended actions
- Rerouting on carrier failure
- Port strike contingency
- Vessel schedule change impact
- Customs hold handling
- Missed pickup recovery
- Proactive risk scoring

### A.6 Tracking and Visibility (In Scope — Simplified)
- Real-time position (AIS)
- ML-based ETA prediction
- Full milestone trace
- On-track vs. at-risk status
- Remaining legs and mode transitions
- Portfolio exception view

### A.7 Analytics (In Scope)
- Cost breakdown by dimension
- Transit time vs. SLA performance
- On-time delivery rate
- Carrier scorecard
- Lane performance trends
- Route explanation / audit trail
- Savings attribution
- Counterfactual / regret analysis
- Carrier volume commitment utilization
- Emissions estimate per route

### A.8 Advisory (In Scope)
- Forwarder quote reasonableness check
- Carrier reliability by lane
- Allocation efficiency
- Disruption exposure assessment
- Capacity pre-booking signal
- Market rate benchmark

### A.9 Deferred Capabilities (Future Phases)
- 3D load building (weight/cube/pallet bin-packing for trucking)
- Dangerous goods and temperature segregation routing (constraint modeling in scope; physical co-load planning deferred)
- Backhaul and continuous move optimization (driver trip chaining)
- Carbon / emissions as optimization objective (emissions estimation in scope; optimization deferred)
- Freight audit (actual invoice vs. planned cost matching)
- Vendor routing guide compliance (inbound supplier shipment rules)
- Air mode (Phase 6)
- Rail mode (future)
- Autonomous booking execution (requires carrier API integrations)

---

## Appendix B: Differentiation Opportunities — Gaps in All Existing TMS Platforms

The following seven capabilities are absent from every major TMS platform (Blue Yonder, Oracle TM, SAP TM, Manhattan Associates, MercuryGate) and represent genuine differentiation opportunities. Each has been elevated to an explicit design requirement in Section 10.

1. **Continuous re-optimization** — existing TMS runs batch planning waves. We design for rolling horizon re-optimization. *(Requirement: Section 10.1)*

2. **Learned constraint inference** — no existing system learns implicit preferences from planner override history. We log all overrides for constraint learning. *(Requirement: Section 10.3)*

3. **Simultaneous multi-echelon joint optimization** — existing systems plan legs sequentially. We model the full door-to-door journey as a single MILP on the unified graph. *(Requirement: Section 10.4)*

4. **Market-responsive spot capacity** — existing systems use static rate tables. We model spot as live supply arcs alongside contracted capacity. *(Requirement: Section 10.5)*

5. **Counterfactual / regret analysis** — no existing system computes post-hoc regret over routing decisions. We store and analyze the gap between chosen route and best-available-in-hindsight. *(Requirement: Section 10.2)*

6. **Fully autonomous end-to-end decision chains** — all current AI features are assistive (recommend, alert). End goal is a planning agent checked by a compliance agent that can execute autonomously. *(Requirement: Section 10.7)*

7. **Probabilistic planning** — all existing optimization uses point estimates for transit time. We model transit time distributions and support probability-of-on-time-delivery as an optimization objective. *(Requirement: Section 10.6)*

---

## Appendix C: Competitive Landscape Summary

*Based on TMS platform research (Blue Yonder, Oracle TM, SAP TM, Manhattan Associates, MercuryGate) and AI-native freight platform research (project44, Pando Pi, Portcast, Shipsy, and others). Research conducted May 2026.*

**What the best TMS platforms do well:**
- Multi-mode route optimization with complex constraint handling (Oracle TM, Blue Yonder)
- Deep carrier tendering and routing guide management
- Exception detection and re-tender workflows
- Analytics and freight spend reporting
- Ocean + air native depth (Oracle TM strongest)

**What AI-native platforms add:**
- Leading AI-native freight forwarders (2026): AI customs audit agents, container optimization agents, digital routing guides with rule logic, natural language search, AIS-powered ocean intelligence via MCP
- Pando Pi (2025): Autonomous freight procurement, dispatch planning, and payment — "AI Teams for Logistics"
- project44 / FourKites: Predictive ETA, exception prioritization, cross-modal visibility

**What none of them do** — see Appendix B. The window for a well-designed, reliable, explainable agentic routing system is open.
