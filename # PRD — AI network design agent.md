# PRD — AI network design agent

*Status: draft

## Problem statement

Network designers need a way to quickly assess whether the current network of fulfillment centers (FCs), transfer fulfillment centers/vendor flex (TXFCs), and replenishment centers can inbound and store the inventory implied by current buying patterns, given
- fixed storage capacities across: 
	- two storage types (i.e., bins (SLOW) and pallets (FAST)), where all SLOW capacity are combined and all BIN capacity are combined.
	- six storage types (i.e., Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST)fixed storage capacities across:
- current DMS enforcing layer rules (i.e., network specialization constraining what capacity types, velocity (FAST/SLOW), and specialized SKUs can go into each node) and case mappings (vendor constraints)
- six storage types (i.e., Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST)

Today this assessment is manual, slow, and ad-hoc and lack an analytic model to answer these key questions.
## Goal

Enable a user to describe a network capacity assessment problem in natural language, have the agent build and solve a MILP, and return a clear answer about whether inventory fits and where the squeeze is. 

## Background and context

The network has FCs, TXFCs, and RCs. Storage is dimensioned in dimensioned on six capacity types — Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST. The "Days of Cover" (DOC) framing relates inventory to forecasted demand. Buying patterns drive inventory; inventory must fit storage. The MVP focuses on the assessment question: given current capacity and current buying, does it fit?

## Users and use cases

*[2–5 concrete scenarios. Each: who, what they do, what they get. Inferred candidates:*

1. Network planner running simulations based on historical inbound patterns. For example, we want to test whether current network can hold 42 DOC based on historical buying patterns. One way to do this is to take last 6 weeks of buying (6 weeks x 7 days per week = 42 days, which is representative of 42 DOCs). Planner passes historical buying pattern in a spreadsheet (could be csv, excel, or text). Agent builds MILP parameterized by the historical demand input file and graph describing the current network, solves, returns whether the inventory fits and identifies binding capacities.

2. Supply chain analyst auditing current state. Analyst asks "given last quarter's actual inventory, where am I tightest? Do analysis for consecutive 6 weeks (which represents 42 DOC)". Test whether current network can hold 42 DOC based on historical buying patterns. One way to do this is to take last 6 weeks of buying (6 weeks x 7 days per week = 42 days, which is representative of 42 DOCs). Planner passes historical buying pattern in a spreadsheet (could be csv, excel, or text). Agent builds MILP parameterized by the historical demand input file and graph describing the current network, solves, returns whether the inventory fits and identifies binding capacities and breakdown by node and storage type breaches for each six week planning horizon. The duration of the data (e.g., take whole quarter demand of 13 consecutive weeks) and the planning period (e.g., consecutive 6 weeks) are configurable. 

3. Demand planner stress-testing assumptions. Planner asks "what happens to my storage utilization if peak season demand is 20% higher than forecast?" Agent perturbs the input, re-solves, compares to baseline.]

## MVP scope — what's in

- Single problem class: storage capacity assessment (no inbound constraints)
* Static snapshot solve (one rolling window of inventory; no time index in the model)
* Six storage types (i.e., Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST)
* Specialized SKUs can only be stored in specialized locations. These are the following specializations:
	* Dangerous goods (DG)
	* Temperature controlled
	* Heavy and Bulky
	* High-value and high-risk (HVHR)
	* Ship-in-own-container (SIOC)
	* Pets
* Each of the above specialized SKU is a binary attribute of the SKU and an SKU can have no specialization up to all specializations (i.e., a SKU can be both DG and HVHR)
* Specialization storage is also limited and defined by the Graph
* Structure of the input graph must be predefined
* Soft capacity constraints with penalized over-utilization. This will help us understand infeasibility.
* Natural-language input → solver → natural-language output via MCP*
* Reads baseline graph G_0 from a static JSON file
* Reads static demand from past 52 weeks (52 is example only can be any block of demand) and we can do analysis on any of the 52 weeks
* Natural-language input to do analysis on any block of the input and output network performance statics, and report breaches/infeasibility
* Returns: feasibility status, utilization by node × storage type, binding constraints, capacity breaches, specialization breaches

## MVP scope — what's out

*- Flow optimization (no arc decisions in v1)*
*- Inbound capacity constraints (only storage in v1)*
*- Outbound capacity constraints (only storage in v1)*
*- Design mode (capacities are parameters, not decision variables)*
*- Multi-period or time-phased optimization*
*- DOC-based safety stock and lead-time math (treat inventory as given input)*
*- SKU-level granularity (work at storage-type aggregates for MVP)*
*- Alternative problem classes (facility location, fixed-charge network design)*
*- Heuristic or LP-relaxation solvers]*

## Functional requirements

*[What the system must do, behaviorally. Suggested categories:*

*- **Input handling:** accept natural-language problem descriptions; identify problem class; extract relevant parameters; load baseline G_0*
*- **Model construction:** build MILP from G(N,A) graph and parameters*
*- **Solving:** call MILP solver; handle infeasibility gracefully*
*- **Output:** return solution and explanation in natural language; surface binding constraints; surface utilization*
*- **Tool surface:** MCP tools must include solve, validate input, return baseline, explain result]*

## Non-functional requirements

*[Performance, latency, error behavior expectations. Suggested:*

*- **Solve time:** typical instance solves in under 30 seconds on CBC*
*- **Failure modes:** MILP infeasibility, malformed input, missing baseline file — all return clear error messages, not stack traces*
*- **Determinism:** same input produces same output (modulo solver tie-breaking)*
*- **Reproducibility:** every solve logs the input graph and the solution]*

## Success criteria

*[How you'll know the MVP works. Candidates:*

*- A user can describe a hypothetical buying pattern in 1–2 sentences and get a feasible/infeasible answer with utilization breakdown*
*- The agent correctly identifies which capacity is binding when infeasibility occurs*
*- A small hand-crafted example produces the expected solution*
*- The system handles malformed natural-language input without crashing]*

## Open questions

*See [[open-questions]] for the live tracker. Highlights:*

*- G(N,A) data structure — list-of-dicts in JSON is the working choice; revisit during Phase 8b*
*- G_0 baseline format — static JSON, schema TBD*
*- Exact MILP formulation — pending Phase 8a*
*- How the agent maps natural-language problem descriptions to a known problem class — open*

## Dependencies and assumptions

*[Inferred candidates:*

*- Assumes baseline G_0 file is hand-maintained and reasonably current*
*- Assumes user provides inventory data in compatible units*
*- Assumes CBC solver is sufficient for MVP problem sizes*
*- Assumes MCP framework (FastMCP) is stable]*

## Risks

*[Inferred candidates:*

*- LLM may incorrectly classify the problem class from natural language → mitigation: validate inputs before solving*
*- MILP may be infeasible by default if capacities are tight → mitigation: soft constraints with penalties*
*- Static G_0 may go stale if not maintained → mitigation: surface staleness in agent output]*

## Notes

*[Free-form. Reasoning, history, links to related thinking. Move polished items to [[design-decisions]] as they stabilize.]*
