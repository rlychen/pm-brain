# AI network design agent — hub

The orienting note for this project. Read me first when entering the folder.

## What this is

An MCP-based agent that helps users design supply chain networks. The user describes a problem in natural language; the agent builds a MILP, solves it, and explains the result. The MVP is a simplified problem class (e.g., facility location with capacity and demand constraints), not full enterprise scale.

This is a learning vehicle. The optimization is well-trodden ground. The interesting work is in how the LLM and the solver coexist as a system — what data formats survive the LLM's editing, how baseline state is shared, what the tool surface should look like.

## Status

Greenfield. No code written. Designing data formats and tool surface. First deliverable is a formal LaTeX MILP model for review.

## Code path

`~/Projects/ai-network-design-agent/`

The code repo is separate from this vault. Code, tests, data files, and the LaTeX model live in the repo. Design decisions, PRDs, experiments, and architectural reasoning live in this folder.

## Build sequence

The project follows a math-first sequence. Each phase is a fail-fast gate.

- **8a.** LaTeX MILP model. Approved before any code is written.
- **8b.** Lock in G(N,A) data structure and G_0 baseline format.
- **8c.** Minimal MILP solver in Python. Hand-crafted test case.
- **8d.** Wrap solver as MCP tools using FastMCP.
- **8e.** Agent loop with system prompt for natural-language → graph → solution.
- **8f.** Iterate: more constraint types, more problem classes.

Currently in 8a planning. No phase is complete.

## Architecture commitments

- MCP server pattern (solver exposed as tools, LLM drives them)
- Generic G(N,A) input format (nodes + arcs with attributes)
- Baseline G_0 + scenario G_i (reference graph, LLM constructs variants from NL)
- MILP only, exact solutions, no heuristics for MVP

See [[design-decisions]] for the reasoning behind each commitment.

## Open questions

- What does the standard G(N,A) data format look like in concrete schema terms? Initial pick: list-of-dicts in JSON. Revisit during Phase 8b if math surfaces issues.
- How should G_0 (baseline graph) reach the LLM? Initial pick: static JSON file in `data/G_0.json` in the repo, loaded at MCP server startup. Updated by hand periodically.
- What is the precise MILP formulation for the MVP problem class? Pending Phase 8a.

See [[open-questions]] for the live tracker.

## Tech stack

- Python 3.12+ via `uv`
- `mip` (Python-MIP) for modeling, CBC bundled solver
- FastMCP for the MCP server
- pytest for solver tests

## Sub-notes

- [[# PRD — AI network design agent]] — product requirements, MVP scope, user flows
- [[design-decisions]] — architectural choices and reasoning
- [[experiments]] — what was tried, what worked, what failed
- [[open-questions]] — running list of open architectural and modeling questions

## Recent activity

*(Append entries here as work progresses. Format: `YYYY-MM-DD — short note`.)*

- 2026-04-26 — project hub created. Initial architecture committed. Awaiting Phase 8a LaTeX model.
