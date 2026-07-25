> ⛔ **HARD RULE — HOW TO WRITE TO THE USER. Read this every session.** Every reply: plain English,
> short, to the point. Define any term the moment you use it. **No notation dumps, no symbol soup**
> (M1/M3/τ/μ/PIH and the like) — the user does not carry symbols across turns and should never have to
> ask you to rephrase. **No walls of text**; a 500–1000-word answer is a failure, not thoroughness. Use a
> tiny numeric example instead of jargon. Ask ONE question at a time, in prose (never AskUserQuestion
> multiple-choice). This has been repeated across many sessions; making the user restate it is a waste of
> his tokens. **Reach for a simple numeric example whenever it helps** — he grasps intuition faster
> from concrete numbers than from prose or notation (e.g. "a plane holds 3000 kg, a shipment is ~500 kg,
> so ~6 fit" beats "capacity exceeds mean shipment weight by a factor of six"). See memories
> `feedback_plain_language_define_terms`, `feedback_one_question_at_a_time`.

> ⛔ **HARD RULE — TARDINESS PENALTY IS ALWAYS ON. Read this every session.** The quadratic C.10
> tardiness penalty (linearized, with the calibrated per-tier express/standard/deferred weights) must be
> ON for **every** model run — quick probe, debug one-off, calibration cell, or full sweep. **NEVER set
> `tardiness_weight_scale = 0`** or otherwise disable any objective term. The proof calibrates three
> COUPLED outcomes at once — cost savings, OTP/tardiness, and fallback (infeasibility relief) — so a run
> with the penalty off produces meaningless OTP/tardiness and invalid conclusions. This has wasted two
> sessions; do not let it happen a third. Before running ANY `solve`/`run_replay`/probe, verify W>0 (the
> penalty is live). See memory `feedback_tardiness_penalty_always_on`.

> ⛔ **HARD RULE — LOOSE PERMISSIONS, TIGHT CHECKPOINTS. Read this every session.** Permission prompts
> are switched off for reads, web research, edits, and test/solve runs (`.claude/settings.local.json`).
> They therefore **no longer function as review gates** — the user is no longer seeing each step go by.
> The speed that buys is granted on **execution only, never on decisions**. Three obligations replace the
> lost prompts:
> 1. **Never guess. Never fabricate.** If a fact, API, parameter, file's contents, or number is not
>    verified, say **"unverified"** and go verify it. Do not infer a plausible value, invent a mechanism,
>    or state a statistic without a source. (Reinforces memories `feedback_no_fabricated_mechanisms`,
>    `feedback_no_unverified_stats`.)
> 2. **Stop and ask before any directional commitment.** Model or schema changes, new dependencies, new
>    constraints, calibration parameters, anything expensive to unwind — these still require explicit
>    user sign-off, exactly as before. Relaxed permissions do not relax the approval gates below.
> 3. **Report failures plainly.** A green suite that tests the wrong thing is a failure, not a pass.
>    Surface it.
>
> `git push` and `gh repo` still prompt by design — the private-repo check is a mechanical backstop, not
> a matter of memory. Never remove them from the `ask` list.

At the start of every session: read BUILD_STATUS.md, the **last (top) entry only** of SESSION_LOG.md, and any approved LaTeX models under `model/` (in that order). BUILD_STATUS.md is the **single pointer** — the canonical built/remaining tracker plus the locked-decisions section — read it first for current position, what's next, and which decisions are settled. SESSION_LOG.md is the append-only archive (large) — read only the most recent entry (stop at the first `---`) for last-session detail; pull older entries on demand only.

Update SESSION_LOG.md continuously as work progresses — add a new entry at the top each session, update it when direction changes, and record where we left off before the session ends.

Update BUILD_STATUS.md whenever a phase completes, a key decision is made, or the project state materially changes — it is the one place "where are we / what's decided" lives (CONTEXT.md was retired S62; its history is in git and SESSION_LOG).

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

- **Do not auto-compile LaTeX.** When editing `.tex` files under `model/`, write the
  edits and stop. Do not run `pdflatex` or any other compile step. The user compiles
  manually and reviews the rendered PDF themselves.

- **User session notes — capture, do not act.** When a user message starts with
  `note:` (case-insensitive, must be the first non-whitespace token of the message),
  the user is recording an observation for end-of-session review, **not** requesting
  action. Behavior:
    1. Append the note verbatim to `usr_session_notes.md` at the project root
       (create the file if it does not exist), prefixed with an ISO timestamp:
       `- [YYYY-MM-DD HH:MM] {note text}`.
    2. Acknowledge in one short line ("noted" or equivalent). Do **not** search,
       edit, plan, or otherwise act on the note's content.
    3. If the same message contains other content after the note (a question or
       task), handle that part normally — but treat the note itself as inert
       capture.
  `usr_session_notes.md` is gitignored / not committed (it is user-private
  scratch). The end-of-session sign-off protocol reviews and clears it.

- **End-of-session sign-off protocol.** Triggered **only** by an explicit sign-off from
  the user ("wrap up", "we're done", "ok bye", "sign off", "end of session", or
  equivalent) — NOT automatically at the end of substantive work. When triggered, **invoke
  the `/signoff` command** (`.claude/commands/signoff.md`), which holds the authoritative,
  complete, ordered checklist (triage notes → SESSION_LOG → BUILD_STATUS full rewrite →
  memory anchor → vault sync → git commit + private-repo check + push). The steps live in
  that one file so they stay in sync; do not re-transcribe them here. Sign-off is complete
  only when every step in `/signoff` lands; if a step fails, surface it and pause — never
  silently skip. `git push` and `gh repo` still prompt by design.

- **Never add admin collaborators to the GitHub repo without the user's
  explicit confirmation.** Only the owner / admins can change repo visibility,
  so keeping the admin set to one person (the user) makes accidental or
  unauthorized visibility flips structurally impossible from any other party.
  If collaborators are added, grant `Read` or `Write` — never `Admin` — unless
  the user explicitly approves admin scope for a specific person.

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
├── BUILD_STATUS.md              THE pointer: current position, what's next, gates, locked decisions (read first)
├── SESSION_LOG.md               append-only archive; read top entry only
├── .claude/commands/signoff.md  the /signoff end-of-session checklist
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
