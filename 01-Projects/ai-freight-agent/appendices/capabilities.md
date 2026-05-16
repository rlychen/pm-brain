# Appendix A: Full Agent Capability Inventory

*Part of the AI Freight Routing PRD. See [PRD.md](../PRD.md) for strategic overview and document map.*

*Complete list of 60+ capabilities across 8 categories. In-scope items are implemented in the prototype. Deferred items are documented for future phases.*

## A.1 Core Routing and Planning (In Scope)
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

## A.2 Constraint Handling (In Scope)
- Hard and soft time windows (pickup and delivery)
- Service level tiers
- Carrier preference / blacklist / allocation caps
- Port / lane avoidance
- Weight and volume constraints
- Commodity restrictions (hazmat, temperature, OOG)
- Trade lane regulatory constraints
- Budget caps
- Dangerous goods / temperature segregation

## A.3 Batch Fleet Operations (In Scope)
- Route full portfolio simultaneously
- Priority segmentation
- Volume consolidation identification
- Carrier allocation compliance monitoring
- Exception queue with urgency ranking
- Bulk re-routing on disruption
- Portfolio risk status

## A.4 Scenario Analysis (In Scope)
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

## A.5 Disruption and Exception Management (In Scope)
- Predicted delay detection
- Ranked recommended actions
- Rerouting on carrier failure
- Port strike contingency
- Vessel schedule change impact
- Customs hold handling
- Missed pickup recovery
- Proactive risk scoring

## A.6 Tracking and Visibility (In Scope — Simplified)
- Real-time position (AIS)
- ML-based ETA prediction
- Full milestone trace
- On-track vs. at-risk status
- Remaining legs and mode transitions
- Portfolio exception view

## A.7 Analytics (In Scope)
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

## A.8 Advisory (In Scope)
- Forwarder quote reasonableness check
- Carrier reliability by lane
- Allocation efficiency
- Disruption exposure assessment
- Capacity pre-booking signal
- Market rate benchmark

## A.9 Deferred Capabilities (Future Phases)
- 3D load building (weight/cube/pallet bin-packing for trucking)
- Dangerous goods and temperature segregation routing (constraint modeling in scope; physical co-load planning deferred)
- Backhaul and continuous move optimization (driver trip chaining)
- Carbon / emissions as optimization objective (emissions estimation in scope; optimization deferred)
- Freight audit (actual invoice vs. planned cost matching)
- Vendor routing guide compliance (inbound supplier shipment rules)
- Air mode (Phase 6+)
- Rail mode (future)
- Autonomous booking execution (requires carrier API integrations)
