---
tags:
  - ai-freight-agent
  - PRD
  - product
status: In Review — Phase 0
version: '0.1'
last_synced: '2026-05-07'
---
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

**Volume:** 10–500 active shipments at any time. Queries are shipment-centric — they think in terms of individual orders and purchase orders.

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
| `route_shipment` | Given a single shipment (origin, dest, cargo, constraints), return viable routes ranked by cost, transit time, and on-time probability. | Ocean Optimizer, Trucking Optimizer, Graph Generator, Transit Time Models, Rules Engine |
| `route_batch` | Given N shipments with optional priority flags, return an optimized routing plan for the full portfolio. | Batch Planner, Ocean + Trucking Optimizers, Rules Engine |
| `track_shipment` | Given a shipment ID, return current position, milestone trace, and latest ETA. | AIS Adapter, Road Tracking Adapter, Shipment State Store |
| `check_on_time_risk` | Given a shipment ID, return P(arrival ≤ deadline), expected arrival distribution, and risk flags. | Probabilistic Transit Model, Shipment State, Rolling Horizon ETA |
| `portfolio_risk_scan` | Return all active shipments at risk of missing delivery window, ranked by urgency. | Probabilistic Transit Model, Shipment State, Risk Scoring Engine |
| `reroute_shipment` | Given a shipment ID and an exception event, re-plan the remaining legs and return options. | Rolling Horizon Controller, Ocean Optimizer, Trucking Optimizer |
| `carrier_select` | Given a shipment and current allocation state, return the optimal carrier per routing guide + allocation cap + availability constraints. | Rules Engine, Carrier Allocation State, Rate Engine, Carrier Availability |
| `disruption_impact_scan` | Given a disruption event, return all active shipments affected with impact severity. | Shipment State, Graph (lane/port/carrier dependency lookup) |
| `trade_compliance_check` | Check routing against trade regulations, sanctions, embargoes, and carrier restrictions. | Regulatory Rules Engine, Sanctions/Embargo DB, Commodity Classification |
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
| `mode_compare` | Side-by-side comparison of ocean vs. air on cost, transit time, and on-time probability. | Ocean + Air Optimizers, Rate Engine, Transit Time Models |
| `what_if_scenario` | Given a route and parameterized scenario (port closure, carrier unavailable, rate change), return re-optimized route and cost/time delta. | Ocean Optimizer with constraint injection, Rate Engine |
| `allocation_check` | Return current usage vs. committed min/max for a carrier on a lane. | Carrier Allocation State, Booking Volume Store |
| `allocation_utilization` | Return carrier allocation utilization summary across all contracts for a period. | Carrier Allocation State, Booking Volume Store |
| `consolidation_evaluate` | Evaluate whether FCL consolidation is cost-effective for a set of LCL shipments. | Consolidation Model, Rate Engine, Sailing Schedule |
| `schedule_query` | Return available ocean sailings for a lane within a date range. | Sailing Schedule Store, Carrier Service Data |
| `trucking_plan` | Generate an optimal drayage and inland trucking plan for containers arriving at a port. | Trucking Optimizer, Graph (port node data), Road Routing Adapter |
| `trucking_availability` | Return available trucking capacity between two points for a date window. | Carrier Availability, Road Routing Adapter |
| `graph_visualize` | Return the G(N,A) subgraph between two nodes with current arc weights for visualization. | Graph Generator, Current Arc State |
| `routing_efficiency_analysis` | Identify shipments where actual routing cost deviated significantly from optimal. | Counterfactual Engine, Rate Benchmark, Shipment Cost Store |
| `disruption_cost_impact` | Compute total incremental cost to portfolio from a disruption event vs. baseline. | Counterfactual Engine, Lane Cost Delta, Shipment Store |
| `counterfactual_analysis` | For completed shipments, compute regret vs. best available route in hindsight. | Counterfactual / Regret Analysis Engine |
| `landed_cost_estimate` | Total landed cost: freight + duties + taxes + handling. | Tariff Lookup, Freight Cost, Duty/VAT Calculator |
| `denied_party_check` | Screen shipper/consignee against denied party and sanctions lists. | Denied Party Screening DB (OFAC, BIS, EU sanctions) |
| `sanctions_check` | Check whether a trade lane, port, or counterparty is subject to active embargo or sanction. | Sanctions/Embargo DB |
| `customs_hold_alert` | Return active shipments matching patterns associated with likely customs holds. | Shipment State, Compliance Rules, Hold Pattern Model |
| `freight_audit` | Compare actual carrier invoice against planned routing cost; flag discrepancies. | Planned Cost Store, Invoice Data, Rate Engine |
| `hazmat_requirements` | Given a commodity, return IMDG class, packing group, and mode-specific DG rules. | DG Rules DB |
| `shipment_audit_trail` | Return the full event and decision history for a shipment. | Shipment Event Log, Decision Log |
| `generate_pre_alert` | Generate a pre-alert notification for an inbound shipment. | Shipment State, Route Formatter, Carrier/Port Reference |
| `generate_booking_instruction` | Generate a formatted booking instruction for a freight forwarder. | Route Formatter, Carrier/Port Reference Data |

#### P2 Tools — Deferred

`rate_benchmark`, `lane_trend`, `emissions_estimate`, `emissions_analytics`, `routing_compliance_scan`, `exception_root_cause_analytics`, `customs_doc_generator`, `contract_compliance`, `origin_classification_audit`, `what_if_carrier_swap`, `capacity_planning`, `disruption_exposure`

---

### 3.6 P0 Priority Summary

The 15 P0 tools cover the daily operational core across all four personas.

- **Routing core (3):** `route_shipment`, `route_batch`, `carrier_select`
- **Tracking and risk core (3):** `track_shipment`, `check_on_time_risk`, `portfolio_risk_scan`
- **Exception management core (2):** `reroute_shipment`, `disruption_impact_scan`
- **Operations workflow core (2):** `cutoff_alert`, routing guide fallback in `carrier_select`
- **Analytics core (3):** `freight_spend_analytics`, `carrier_scorecard`, `otd_analytics`
- **Compliance core (3):** `trade_compliance_check`, `document_requirements`, `tariff_lookup`

---

## 4. Modes in Scope

**Prototype:** Ocean + Trucking (pre-carriage to origin port, drayage from destination port, inland trucking to final destination)

**Deferred:** Air, Rail

The ocean + trucking pair forces mode-transition handling (the hard part) from day one and covers the dominant volume pattern for cross-border freight.

---

## 5. Key Architectural Concept: Rolling Horizon Planning

*First-principles design decision that differentiates this system from all existing TMS platforms.*

See also: [[Rolling Horizon Planning]]

### 5.1 The Problem with Sequential Leg Planning

All existing TMS systems plan each mode leg sequentially: book the ocean leg, estimate the inland leg, execute. The inland leg estimate is made with high uncertainty at booking time (unknown exact arrival time, unknown port clearance delay, no specific carrier committed). By the time the vessel arrives, that estimate is stale but the planning system doesn't re-optimize.

### 5.2 Rolling Horizon Approach

This system maintains a **complete end-to-end plan at all times**, but resolves each leg at different levels of graph resolution depending on how close we are to execution:

- **G_coarse**: Sparse graph for future legs. Arc weights are cost/time envelopes derived from historical data and ML models. Sufficient for the optimization objective but not for execution.
- **G_fine**: Dense graph for the next leg to be executed. Arc weights are actual carrier schedules, confirmed spot rates, and port-specific clearance estimates.

As a shipment advances and uncertainty decreases (e.g., AIS-derived vessel ETA confidence exceeds a threshold), the system fires a **re-plan trigger** and re-solves the next leg on G_fine with real schedules and live rates.

### 5.3 Concrete Example

**At booking time (T=0):**
- Ocean leg: precisely committed — specific carrier, vessel, sailing date, scheduled ETA ±3 days
- Inland leg (USLAX → Phoenix): rough envelope — $400–900, 1–3 days, no specific carrier

**Re-plan trigger fires when:** vessel is 72h from USLAX, AIS-derived ETA confidence > 90%, port clearance window confirmed

**At T=2 (vessel near USLAX):**
- Ocean leg: confirmed — actual ETA Jul 4 06:00 PST, port clearance est. Jul 4 14:00
- Inland leg: re-solved on G_fine — 3 specific options:
  - [A] Direct Truck (OHL) — depart Jul 4 16:00, arrive Jul 5 09:00 — $612 ✓ selected
  - [B] Drayage + Rail — depart Jul 4 20:00, arrive Jul 6 08:00 — $389
  - [C] Expedite (FedEx CC) — depart Jul 4 14:30, arrive Jul 4 22:00 — $1,480

Diagram file: `ai-freight-agent/docs/rolling_horizon_planning.drawio`

### 5.4 Design Principle

The optimizer holds a full door-to-door plan at all times. Future legs are placeholders on G_coarse — enough to make good booking decisions. As each leg's execution window approaches, it is re-solved on G_fine with real data. This is **Model Predictive Control applied to multimodal freight routing**.

**This is a hard requirement.** The system must maintain and continuously update a full multi-horizon plan.

---

## 6. Supply and Demand Model

### 6.1 Demand — Ocean Shipments

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
| `incoterm` | EXW, FOB, CIF, DDP, etc. |
| `service_level` | Economy, Standard, Express |
| `carrier_preferences` | Preferred, acceptable, excluded carriers |
| `budget_cap` | Optional hard cost cap per shipment |

### 6.2 Demand — Trucking Shipments

| Field | Description |
|---|---|
| `shipment_id` | Unique identifier |
| `origin` | Pickup address |
| `destination` | Delivery address |
| `pickup_window` | [earliest, latest] pickup datetime |
| `delivery_window` | Hard deadline or [earliest, latest] |
| `weight_kg` | Gross weight |
| `volume_cbm` | Volume |
| `pallet_count` | Number of pallets |
| `cargo_type` | General, hazmat, temperature-controlled |
| `service_level` | Standard, Expedite, Dedicated |
| `carrier_preferences` | Preferred, acceptable, excluded |

### 6.3 Supply — The Graph G(N, A)

**Nodes N:**
- Origin locations (factories, warehouses, pickup addresses)
- Origin ports (container terminals with sailing schedules)
- Transshipment ports (intermediate hub ports)
- Destination ports
- Inland distribution points (rail ramps, cross-docks, truck terminals)
- Final destinations (warehouses, DCs, delivery addresses)

**Arcs A:**
- **Ocean arcs**: carrier service legs between port pairs, with scheduled departure/arrival, capacity, and rate
- **Pre-carriage arcs**: pickup truck legs from origin to origin port
- **Drayage arcs**: port-to-inland-hub truck legs
- **Inland trucking arcs**: point-to-point truck moves
- **Transshipment arcs**: inter-terminal transfers at hub ports

Each arc carries:
- `transit_time_distribution`: mean + variance (probabilistic, not point estimate)
- `cost`: base rate + fuel surcharge + accessorials
- `capacity`: available slots or load units
- `cutoff_time`: latest departure to meet downstream schedule
- `service_type`: carrier, service name, frequency

---

## 7. Agent Capabilities

### 7.1 Core Routing and Planning
- Route a single shipment: all viable options given constraints
- Lowest-cost, fastest, reliability-optimized routing
- Multi-objective: Pareto frontier of cost vs. time vs. reliability
- Mode selection: ocean vs. trucking vs. combined multimodal
- Carrier selection within mode against routing guide and preference rules
- FCL vs. LCL decision (consolidation economics)
- Direct vs. transshipment routing
- Rolling horizon re-plan trigger evaluation and execution

### 7.2 Constraint and Rule Handling
- Hard and soft time windows (pickup and delivery)
- Service level tiers: Economy, Standard, Express
- Carrier preference / blacklist / allocation cap enforcement
- Port or lane avoidance (e.g., Red Sea, Panama congestion)
- Weight and volume constraints per leg
- Commodity restrictions: hazmat class, temperature-controlled, OOG
- Trade lane regulatory constraints
- Budget cap per shipment or per lane

### 7.3 Batch and Fleet Operations
- Route all unbooked shipments in a portfolio simultaneously
- Priority segmentation: urgent vs. economical
- Volume consolidation: identify which shipments can merge into one container
- Carrier allocation compliance monitoring
- Exception queue: surface shipments requiring human decision, ranked by urgency
- Bulk re-routing on disruption
- Portfolio status: at-risk vs. on-track

### 7.4 Scenario Analysis and What-If
- Origin port shift, transit time vs. cost tradeoff, LCL→FCL upgrade
- Carrier unavailability, shipment splitting
- Red Sea avoidance: route via Cape of Good Hope, full cost/time delta
- Air vs. ocean full comparison
- Service level upgrade cost, tariff/duty change impact
- Capacity constraint scenario: model port downtime

### 7.5 Disruption and Exception Management
- Detect predicted delay: weather, port congestion, vessel rollover, anchorage wait
- Alert with ranked recommended actions
- Autonomous rerouting recommendation on carrier failure or missed cutoff
- Port strike / closure contingency routing
- Vessel schedule change: recalculate all impacted shipments
- Proactive risk scoring: which booked shipments are most at risk this week?
- Rolling horizon re-plan on disruption

### 7.6 Tracking and Visibility
- AIS position on ocean legs
- ML-based ETA prediction (not just carrier schedule)
- Full milestone trace: cargo ready → picked up → departed → transshipment → arrived → customs cleared → delivered
- On-track vs. at-risk status vs. committed delivery window
- Portfolio exception view with filters

### 7.7 Analytics and Performance
- Cost breakdown by lane, carrier, mode, time period
- Transit time performance vs. SLA by carrier and lane
- On-time delivery rate
- Carrier scorecard: cost, reliability, rollover rate, on-time delivery
- Route explanation / audit trail: why was this specific route chosen?
- Savings attribution: how much did optimization save vs. default routing?
- Counterfactual / regret analysis
- Carrier volume commitment utilization
- Emissions estimate: CO₂ per route option

---

## 8. Agent Architecture

### 8.1 Framework: LangGraph

**Decision: LangGraph, not direct Anthropic SDK.**

| Criterion | LangGraph | Direct Anthropic SDK |
|---|---|---|
| Decision logging | LangSmith — best-in-class, zero-build | Build it yourself |
| MCP integration | `langchain-mcp-adapters` — production-tested | Native Anthropic SDK |
| Model swappability | Yes — model-agnostic | No — Claude-only |
| Planner-validator pattern | Native supervisor with conditional edges | Custom build |
| Human-in-the-loop | First-class `interrupt()` + PostgreSQL checkpointer | Build it yourself |
| Debuggability | Time-travel debugging in LangSmith | Build it yourself |

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

### 8.3 Agent Personas

**Routing Planner Agent** — orchestrates optimization and ML tools to generate end-to-end route recommendations. Output: `{route, total_cost, expected_transit_days, p_on_time, constraints_checked, rationale}`

**Compliance/Validation Agent** — independently reviews planner output as a skeptical auditor. Separate system prompt, no shared planner history, no access to the optimization solver. Mandatory per-item checklist covering: carrier constraint compliance, time window feasibility, weight/volume compliance, business rule compliance, optimization sanity, regulatory check. Output: `{status: PASS|FAIL|ESCALATE, findings: [{rule, status, evidence}], summary}`

**Execution Monitor Agent** — event-driven, watches active shipments, fires rolling horizon re-plan triggers, generates proactive alerts. Runs async, never in the request path.

**Market Intelligence Agent** — scheduled refresh + on-demand queries. Maintains live view of spot rates, port congestion, disruptions. Queryable by Routing Planner as a tool call.

### 8.4 Human-in-the-Loop Checkpoints

LangGraph `interrupt()` fires at:
1. Validator returns ESCALATE
2. Planner-validator loop reaches 3 revision cycles without PASS
3. Any booking action (future autonomous mode)
4. Exception requires rerouting a high-value or time-critical shipment (<24h window)

State persisted via PostgreSQL checkpointer.

### 8.5 Production Failure Mode Mitigations

| Failure Mode | Mitigation |
|---|---|
| Infinite planner-validator loop | Max 3 revision cycles, then auto-ESCALATE |
| Sycophantic validator | Independent tool access, skeptic framing, PASS rate monitoring |
| Context loss on agent handoff | Typed Pydantic schemas at every boundary |
| Concurrent booking conflicts | Optimistic locking on carrier capacity state |
| Latency cascade | Execution Monitor and Market Intelligence run async |
| Validation theater | Monitor PASS rate; >90% PASS without findings triggers review |

---

## 9. Data Sources

| Source | Type | Use | License |
|---|---|---|---|
| UN/LOCODE | Real topology | Port and location reference | Free |
| IATA codes | Real topology | Airport reference | Free |
| NOAA AIS (historical) | Real signal | Ocean vessel tracking, transit time training | Free |
| Google Maps Routes API | Real signal | Road transit time estimation | Pay-per-use |
| OSRM | Real signal | Road routing (free alternative) | Free |
| OpenSky Network | Real signal | Air freight schedules (historical) | Free |
| Ocean carrier schedules | Real topology | Sailing schedule graph construction | Public |
| BTS Freight Analysis Framework | Real topology | Trucking lane structure | Free |
| Synthetic rates | Synthetic | Commercial rate parameters | N/A |
| DAT | — | **NOT licensed. Do not use.** | — |
| USLAX / port authority data | Real signal | Terminal throughput, clearance windows | Free / public |
| BNSF / UP intermodal ramp data | Real topology | Inland ramp locations, service days | Public |
| NOAA / NWS weather | Real signal | Weather disruption risk by port and lane | Free |

---

## 10. Differentiation Requirements

Seven capabilities absent from every major TMS platform — explicit design requirements, not nice-to-haves.

1. **Continuous re-optimization (Rolling Horizon)** — batch-wave planning is not acceptable. (→ Section 5)
2. **Counterfactual / regret analysis** — post-shipment regret = `|cost(chosen) - cost(best_in_hindsight)|`. (→ Section 10.2)
3. **Learned constraint inference** — log all planner overrides as the primary input for constraint learning. (→ Section 10.3)
4. **Simultaneous multi-echelon joint optimization** — full door-to-door journey as one MILP on G(N,A). (→ Section 10.4)
5. **Spot capacity as supply** — spot arcs with rate distributions alongside contracted capacity. (→ Section 10.5)
6. **Probabilistic planning** — transit time distributions, not point estimates; P(on-time ≤ deadline) as optimization objective. (→ Section 10.6)
7. **Fully autonomous decision chains (end goal)** — prototype is decision-support; end goal is autonomous plan + execute. (→ Section 10.7)

---

## 11. Components Inventory

Each component is independently buildable and testable. No stitching until each component passes isolation tests.

| Component | Description | Mode(s) |
|---|---|---|
| Graph Generator | Constructs G(N, A) from network data. Nodes enriched with real public data: port nodes get terminal throughput, customs clearance windows, anchorage wait distributions; inland nodes get intermodal ramp locations (BNSF, UP), road distance matrices, historical road transit distributions. | Ocean + Trucking |
| Ocean Transit Time Model | ML model: distribution over transit time per ocean arc | Ocean |
| Trucking Transit Time Model | ML model: distribution over transit time per trucking arc | Trucking |
| Ocean Optimizer | MILP: selects optimal ocean route given demand and constraints | Ocean |
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

---

## 12. Build Sequence

**Phase 0 — PRD** ← CURRENT  
**Phase 1 — Formal Models (LaTeX)** — one model per component, each approved individually  
**Phase 2 — Component Builds** — Graph Generator → Transit Time Models → Mode Optimizers → Rules Engine → Adapters → Stitching Layer → Rolling Horizon Controller  
**Phase 3 — MCP Server** — expose all verified components as tools  
**Phase 4 — Agent Layer** — Planning Agent → Validation Agent → Execution Monitor  
**Phase 5 — Integration and End-to-End Testing**  
**Phase 6 — Iterate** — air mode, improved models, extended agent capabilities

---

## 13. Open Questions

1. **Decision-support vs. autonomous execution** — define trigger conditions and safety checks required before moving to autonomous execution.
2. **Design partner selection** — who are the first 2–3 customers? What lanes do we start with?
3. **Live AIS feed** — evaluate MarineTraffic, VesselFinder, SpireGlobal on cost vs. coverage vs. API quality.
4. **Carrier booking APIs** — when moving toward autonomous execution, which ocean carriers first? Which trucking carriers for drayage?
5. **Pricing model** — per-shipment, per-decision, or monthly subscription? What is the per-shipment cost floor given MILP solve + LLM call?
6. **Emissions optimization** — carbon as a routing objective requires accurate emissions factors per mode, carrier, and vessel. Data source TBD.
