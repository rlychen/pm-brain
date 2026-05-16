At the start of every session: read CONTEXT.md, SESSION_LOG.md, and any approved LaTeX models under `model/` (in that order).

Update SESSION_LOG.md continuously as work progresses — add a new entry at the top each session, update it when direction changes, and record where we left off before the session ends.

Update CONTEXT.md whenever a phase completes, a key decision is made, or the project state materially changes.

# Project: AI freight agent

Project-level instructions for Claude Code. Overrides global `~/.claude/CLAUDE.md`.

## What this project is

A multimodal freight routing system with an agentic layer. Given a shipment request,
the agent orchestrates optimization and ML components across modes (ocean, air,
trucking) to recommend end-to-end routes. Commercial product, services-as-software
model.

The system is composed of independently testable components — mode optimizers, transit
time predictors, graph generation, rules engine, stitching layer — coordinated by an
MCP-native agent.

## Current status

Phase 0 — PRD v0.2 in review. LaTeX model `ocean_fcl_routing.tex` in draft (not yet approved). No code written. Competitive research in `Research.md` (33 sites, 14 companies, May 2026).

## Build sequence

**Phase 0 — PRD.** Define problem scope, commercial model, data sources, component
inventory, agent capabilities (planning + analytical), and open questions.

**Phase 1 — Formal models (LaTeX).** One model per component. Each approved
individually before code starts for that component.

**Phase 2 — Component builds.** One component at a time. Each component is built,
tested in isolation, and verified correct before the next component starts.

**Phase 3 — Stitching.** Multimodal graph assembly and cross-mode coordination.
Built only after all constituent components pass isolation tests.

**Phase 4 — MCP server.** Expose components as tools. Agent loop on top.

**Phase 5 — Agent capabilities.** Planning and analytical tools defined in PRD,
implemented one at a time.

**Phase 6 — Iterate.** Extend capabilities, add modes, improve models.

## Confidentiality — Hard Rule

**Never reference previous employers or their named products anywhere in this project.**

Banned terms — do not use in code, comments, docstrings, variable names, filenames, commit messages, documentation, diagrams, or any other artifact:
- Company names: Flexport, Amazon, Coupang (and variations/abbreviations)
- Internal product names from those companies (e.g. Atlas as a Flexport product, any internal tool names)

Real-world names — carriers (MSC, COSCO, CMA CGM), ports (USLAX, CNSHA), cities, vessels, shippers, platforms — are encouraged for realism and should be used freely. Public data tied to real nodes (port geometry, lane rates, vessel schedules) is an asset.

If a file you are about to write or edit would contain any banned term, stop and ask before proceeding.

## Guardrails

- **Hard approval gates.** PRD must be approved before any LaTeX is written. Each
  LaTeX model must be approved before code for that component starts. No exceptions.

- **Component isolation before stitching.** Every component — mode optimizers,
  transit time models, graph generator, rules engine, stitching layer — must pass
  correctness tests in isolation before it is integrated with any other component.

- **Math or methodology first.** No agent capability may be added without a
  corresponding approved formal model or documented methodology. No LLM-improvised
  routing logic.

- **Correctness before performance.** No caching, parallelization, or solver tuning
  until correctness is verified on a small example.

- **No scope expansion without explicit confirmation.** New modes, constraint types,
  or capabilities require user approval before implementation.

- **Write supporting notes to the vault, not the repo.** Design reasoning and
  experiment journals belong in `~/Documents/PM-Brain/01-Projects/ai-freight-agent/`.

- **FastMCP stdout constraint.** FastMCP uses stdout for JSON-RPC transport. Any
  `print()` call in code that executes during a tool call will corrupt the protocol
  stream and cause the tool to hang silently. All diagnostic output must use
  `print(..., file=sys.stderr)` or Python's `logging` module. This applies to every
  file in `src/`.

- **Log all agent interactions.** Every user query to the agent and every agent
  response must be logged with a timestamp. Log file path: `logs/agent_interactions.jsonl`.
  Format: one JSON object per line with fields `timestamp`, `role`, `content`, and
  optionally `tool_calls`. This log is the primary input for extending capabilities.

## Data sources

Real network topology + synthetic commercial parameters. Document data provenance
at the point of use. Every source must be identified as real-network or synthetic.

- Port reference: UN/LOCODE, IATA — free, real topology
- AIS ocean GPS: NOAA historical (US waters, free) — real signal, rate-limited
- Road transit: Google Maps Routes API or OSRM — real signal
- Air schedules: OpenSky Network (historical, free) — real signal
- Trucking rates: synthetic distributions (BTS FAF for lane structure) — synthetic
- DAT: NOT licensed. Do not ingest DAT data without an explicit license.

## Tech stack

- Python 3.12+ managed by `uv`
- MILP solver: HiGHS (`highspy`)
- MCP framework: FastMCP
- Agent framework: LangGraph (model-agnostic, native planner-validator pattern, LangSmith observability)
- LLM: Claude via LangGraph (model-agnostic; Anthropic SDK used for direct Claude calls where needed)
- Testing: pytest

## Unit testing

**Gate:** A component does not move to integration until its isolation tests pass. This is a hard gate, same as the LaTeX approval gate.

**What a passing isolation test means:**
- Correct output on the happy path (small, deterministic synthetic instance)
- Graceful handling of infeasibility or invalid input (structured error, not exception)
- For MILP components: solution value within expected bounds, not just "solver returned OPTIMAL"

**Test structure:**
- One test file per component: `tests/components/test_{component}.py` mirrors `src/components/{component}.py`
- Shared fixtures in `tests/conftest.py` — small synthetic instances, fixed seed, ≤5 commodities, ≤10 sailings
- Test names: `test_{component}_{scenario}` — e.g., `test_ocean_optimizer_vessel_cap_binding`

**What to test per component type:**

*Graph Generator:* subgraph construction (known schedule → correct arc set), infeasible commodity (deadline too tight → empty A^k with report), reachability sweep (no dangling arcs), CYC compliance (arc excluded when τ_k(i) > CYC_ij), decomposition (TPEB+FEWB batch → two disconnected components in H)

*Ocean Optimizer:* trivial feasibility (1 commodity, 1 arc → that arc), infeasibility propagation (no-path commodity rejected before MILP build), vessel cap binding (P.2), string allocation binding (P.3), budget cap (P.4), container mix correctness (specific v/w/ρ → manually verified f*/t*/cost/slots), optimal value bound (solution cost ≤ expected upper bound on small instance), decomposition independence (joint vs. separate solve → same result)

*Rules Engine:* blacklisted carrier never selected, preferred carrier selected when feasible, allocation-exhausted carrier excluded

*MCP tools:* schema rejection on malformed input, stdout cleanliness (no print() on tool call path), correct output schema on happy path

**Hard rules:**
- Never mock the MILP solver — HiGHS must run on real small instances
- Never mock the graph or database state — use in-memory or tmp fixtures
- Performance tests are deferred until correctness is confirmed at scale; do not add timing assertions in isolation tests

## File layout

```
ai-freight-agent/
├── CLAUDE.md                    this file
├── CONTEXT.md                   compressed session context
├── SESSION_LOG.md               running session log
├── PRD.md                       strategic index: exec summary, modes in scope, doc map, differentiation, open questions (v0.3)
├── EXECUTION_PLAN.md            living build plan: phases, gates, component sequence, open decisions — updated before any phase begins
├── agent_design.md              AI-native design philosophy, autonomy model, guardrails, agent architecture
├── data_model.md                supply/demand model, graph G(N,A), arc schemas, entity model, SQL schemas
├── ui_spec.md                   look & feel, screen inventory, wireframes, agent feed design
├── personas_and_tools.md        four personas, MCP tool inventory (P0/P1/P2)
├── build_plan.md                tech stack, multi-tenancy, demand generator, peripheral components, build sequence
├── Research.md                  competitive intelligence (33 sites, 14 companies, May 2026)
├── appendices/
│   ├── capabilities.md          full agent capability inventory (60+ capabilities)
│   └── competitive.md           differentiation gaps, competitive landscape, market gaps
├── pyproject.toml               uv-managed
├── model/                       LaTeX formulations (one per component)
│   └── ocean_fcl_routing.tex    Ocean FCL optimizer — Draft v2, not yet approved
├── docs/                        diagrams
├── src/
│   ├── components/              one module per component
│   ├── server.py                FastMCP server
│   └── agent.py                 agent loop (LangGraph)
├── data/
│   ├── reference/               port codes, network topology
│   └── synthetic/               generated test instances
├── logs/
│   ├── agent_interactions.jsonl agent query/response log
│   └── overrides.jsonl          operator override log (constraint learning feed)
└── tests/
    └── components/              one test file per component
```
