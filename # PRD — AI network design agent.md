# PRD — AI network design agent

*Status: draft scaffold. Sections below contain placeholder descriptions of what to fill in. Edit each section with the actual content.*

## Problem statement

Large-scale e-commerce supply chain network designers need a way to quickly assess whether the current network of fulfillment centers, transfer fulfillment centers, and replenishment centers can inbound store the inventory implied by current buying patterns, given fixed storage capacities across two storage types (i.e., bins and pallets). Today this assessment is manual, slow, and ad-hoc. Each IB 

## Goal

*[One sentence on what the MVP must do. Candidate: enable a user to describe a network capacity assessment problem in natural language, have the agent build and solve a MILP, and return a clear answer about whether inventory fits and where the squeeze is.]*

## Background and context

*[Domain context for someone unfamiliar with the network. Suggested content: the network has FCs, TXFCs, and RCs. Storage is dimensioned in four types — pallet storage (fungible across FAST velocity SKUs), and three bin storage types (Totable, Non-totable, Grande) for SLOW velocity SKUs. The "Days of Cover" (DOC) framing relates inventory to forecasted demand. Buying patterns drive inventory; inventory must fit storage. The MVP focuses on the assessment question: given current capacity and current buying, does it fit?]*

## Users and use cases

*[2–5 concrete scenarios. Each: who, what they do, what they get. Inferred candidates:*

*1. **Network planner running a what-if.** Planner describes a hypothetical buying pattern in natural language. Agent builds MILP, solves, returns whether the inventory fits and identifies binding capacities.*

*2. **Supply chain analyst auditing current state.** Analyst asks "given last quarter's actual inventory, where am I tightest?" Agent loads baseline data, runs assessment, returns utilization breakdown by node and storage type.*

*3. **Demand planner stress-testing assumptions.** Planner asks "what happens to my storage utilization if peak season demand is 20% higher than forecast?" Agent perturbs the input, re-solves, compares to baseline.]*

## MVP scope — what's in

*[The minimum that must work for v1. Inferred candidates to confirm or override:*

*- Single problem class: storage capacity assessment (not flow optimization, not facility location, not design mode)*
*- Static snapshot solve (one rolling window of inventory; no time index in the model)*
*- Four storage types: pallet, NT bin, Grande bin, Totable bin*
*- All three node types treated as functionally identical for storage feasibility*
*- Soft capacity constraints with penalized over-utilization*
*- Natural-language input → solver → natural-language output via MCP*
*- Reads baseline graph G_0 from a static JSON file*
*- Returns: feasibility status, utilization by node × storage type, binding constraints]*

## MVP scope — what's out

*[Explicit out-of-scope list. Inferred candidates:*

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
