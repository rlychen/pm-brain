# User Personas, Requirements, and Tool Mapping

*Part of the AI Freight Routing PRD. See [PRD.md](PRD.md) for strategic overview and document map.*

This section is the primary requirements document. Each persona has a role description, the questions and tasks they want to perform, and a mapping to the specific MCP tool that answers each question. The tool inventory derived here drives the components built in Phase 2 and exposed in Phase 3.

**Design principle:** The majority of use cases must be answerable by a deterministic or model-backed tool call — not by the LLM reasoning from scratch or writing code dynamically. The agent's job is to route the user's question to the right tool, interpret the result, and present it clearly. Tools provide consistency, speed, and auditability. LLM inference fills gaps where judgment is needed.

**Priority definitions:**
- **P0** — Must-have for prototype. The ~20% of question types that cover ~80% of daily operational value. System is not useful without these.
- **P1** — High value, built in Phase 5 iteration. Covers the next layer of operational depth.
- **P2** — Important long-term, deferred. Specialized, lower-frequency, or dependent on data sources not yet integrated.

---

## Persona: Shipper — Logistics Manager

**Role:** Employed by a manufacturer, retailer, or importer. Owns outbound and/or inbound freight from origin factories or suppliers to destination warehouses or stores. Responsible for on-time delivery, freight cost, and carrier relationships. Interacts with one or more freight forwarders to execute shipments.

**Volume:** 10–500 active shipments at any time. The logistics manager thinks in terms of purchase orders (POs), not containers. Each PO has a delivery due date and a cargo volume (CBM). Depending on CBM, a PO may fill one or more full containers (FCL) or share a container with other cargo (LCL). A "shipment" in this context is typically one PO or a group of POs moving together — the system must handle both FCL and LCL demand, though LCL consolidation optimization is deferred.

**KPIs they own:** On-time in full (OTIF), freight cost per unit, transit time vs. committed, carrier reliability.

**What they want to avoid:** Surprises. Late arrivals, unexpected cost overruns, being the last to know about a disruption.

### Questions and Tasks

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

**Note — LCL (Less-than-Container Load):** When a PO's volume is too small to justify a full container, it moves as LCL — consolidated with other cargo into a shared container by a freight forwarder or NVOCC. Routing LCL shipments requires a consolidation optimizer that decides which LCL shipments to group together and onto which sailings. This is a distinct problem from FCL routing and requires its own MILP formulation (bin-packing × routing). Deferred to a future phase — see `build_plan.md` (Components) and Open Question 8 in PRD.md.

---

## Persona: Freight Forwarder — Operations Planner

**Role:** Works at a freight forwarder. Does not manually plan shipments — the AI agent does. Instead, governs agent behavior: sets routing policy, resolves escalations the agent cannot handle, reviews audit trails, and adjusts guardrails when override patterns reveal policy gaps.

**Volume:** 100–500 shipments per day in queue. On a normal day, directly interacts with fewer than 5% of shipments (exceptions only). Queries are portfolio-centric — they think across the full book of business, not individual shipments.

**KPIs they own:** Carrier acceptance rate, cutoff compliance, exception resolution time, carrier allocation utilization, rollover rate.

**What they want to avoid:** Missing a vessel cutoff, overshooting carrier allocation caps, learning about a disruption after it's too late to recover.

### Questions and Tasks

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

## Persona: Freight Forwarder — Business Analyst / Management

**Role:** Owns performance reporting, cost analytics, carrier contract negotiation support, and strategic lane decisions. Not in the daily execution flow. Runs analysis weekly/monthly to identify opportunities and problems.

**Volume:** Lower frequency, higher complexity queries. Think in aggregates — lanes, carriers, time periods — not individual shipments.

**KPIs they own:** Total freight spend, cost per TEU/kg, on-time delivery rate, carrier contract utilization, cost vs. market.

**What they want:** Answers to "where are we leaving money on the table?" and "which carriers are letting us down?"

### Questions and Tasks

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

## Persona: Compliance Officer / Customs Broker

**Role:** Ensures all shipments comply with trade regulations, carrier contract terms, customs requirements, and internal routing policy. May be embedded at a freight forwarder or at a large shipper. Operates at the intersection of legal, operational, and financial risk.

**Volume:** Reviews shipments at booking time and flags exceptions. Also runs periodic compliance audits.

**KPIs they own:** Customs hold rate, compliance violation rate, freight audit recovery (overbilling caught), denied party clearance rate.

**What they want:** Catch problems before shipments move. Audit trails. Evidence that the system checked the rules.

### Questions and Tasks

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

## Master MCP Tool Inventory

All tools derived from the persona requirements above, with priority, description, and underlying components.

### P0 Tools — Must-Have for Prototype

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

### P1 Tools — Built in Phase 5

| Tool | Description | Underlying Components |
|---|---|---|
| `lane_transit_estimate` | Return the current transit time distribution (mean, std dev, P50/P90) for a lane, mode, and carrier. | Transit Time Model, Lane Analytics |
| `mode_compare` | Given a shipment, return a side-by-side comparison of ocean vs. air (vs. combined) on cost, transit time, and on-time probability. | Ocean + Air Optimizers, Rate Engine, Transit Time Models |
| `what_if_scenario` | Given a route and a parameterized scenario (port closure, carrier unavailable, rate change, lane avoidance), return the re-optimized route and cost/time delta. | Ocean Optimizer with constraint injection, Rate Engine |
| `allocation_check` | Return current usage vs. committed minimum/maximum for a carrier on a lane or across all lanes. | Carrier Allocation State, Booking Volume Store |
| `allocation_utilization` | Return carrier allocation utilization summary across all contracts for a period. | Carrier Allocation State, Booking Volume Store |
| `consolidation_evaluate` | Given a set of LCL shipments on the same lane, evaluate whether FCL consolidation is cost-effective and which sailings support it. | Consolidation Model, Rate Engine, Sailing Schedule |
| `schedule_query` | Return available ocean sailings for a lane within a date range, with carrier, vessel, ETD, ETA, and available capacity indicator. | Sailing Schedule Store, Carrier Service Data |
| `trucking_plan` | Given a set of containers arriving at a port within a window, generate an optimal drayage and inland trucking plan. | Trucking Optimizer, Graph (port node data), Road Routing Adapter |
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

### P2 Tools — Deferred

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

## P0 Priority Summary

The 15 P0 tools cover the daily operational core across all four personas. Building only these tools delivers a working system for freight forwarder operations planners and shippers. Everything else is additive.

**Routing core (3 tools):** `route_shipment`, `route_batch`, `carrier_select`

**Tracking and risk core (3 tools):** `track_shipment`, `check_on_time_risk`, `portfolio_risk_scan`

**Exception management core (2 tools):** `reroute_shipment`, `disruption_impact_scan`

**Operations workflow core (2 tools):** `cutoff_alert`, `carrier_select` (routing guide + fallback)

**Analytics core (3 tools):** `freight_spend_analytics`, `carrier_scorecard`, `otd_analytics`

**Compliance core (3 tools):** `trade_compliance_check`, `document_requirements`, `tariff_lookup`
