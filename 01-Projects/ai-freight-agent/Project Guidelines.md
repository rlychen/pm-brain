---
tags:
  - ai-freight-agent
  - guidelines
  - setup
last_synced: '2026-05-07'
---
# Project Guidelines (CLAUDE.md)

*Source of truth is `~/Projects/ai-freight-agent/CLAUDE.md` — this is a read copy.*

---

## What This Project Is

A multimodal freight routing system with an agentic layer. Given a shipment request, the agent orchestrates optimization and ML components across modes (ocean, air, trucking) to recommend end-to-end routes. Commercial product, services-as-software model.

---

## Current Status

Phase 0 — PRD in progress. No LaTeX, no code yet.

---

## Build Sequence (Gates)

Each phase requires explicit approval before the next begins.

1. **Phase 0 — PRD** ← CURRENT
2. **Phase 1 — Formal Models (LaTeX)** — one model per component, each approved individually
3. **Phase 2 — Component Builds** — Graph Generator → Transit Time Models → Mode Optimizers → Rules Engine → Adapters → Stitching Layer → Rolling Horizon Controller
4. **Phase 3 — MCP Server** — expose all verified components as tools
5. **Phase 4 — Agent Layer** — Planning Agent → Validation Agent → Execution Monitor
6. **Phase 5 — Integration and End-to-End Testing**
7. **Phase 6 — Iterate** — air mode, improved models, extended agent capabilities

---

## Hard Guardrails

- **PRD must be approved before any LaTeX is written.** Each LaTeX model must be approved before code for that component starts.
- **Every component must pass correctness tests in isolation before it is integrated.**
- **No agent capability without an approved formal model or documented methodology.** No LLM-improvised routing logic.
- **Correctness before performance.** No caching, parallelization, or solver tuning until correctness is verified on a small example.
- **No scope expansion without explicit confirmation.**
- **Design reasoning goes in this vault.** Not in the repo.

---

## Confidentiality — Hard Rule

Never reference previous employers or their named products in any artifact.

Banned: Flexport, Amazon (non-AWS), Coupang (and variations), internal product names from those companies.

Encouraged: real carrier names (MSC, COSCO, CMA CGM), real ports (USLAX, CNSHA), real vessels, real shippers, real platforms.

---

## Tech Stack

| Tool | Role |
|---|---|
| Python 3.12+ / uv | Language + package management |
| HiGHS (highspy) | MILP solver |
| FastMCP | MCP server framework |
| LangGraph | Agent orchestration (model-agnostic, native planner-validator, LangSmith observability) |
| Claude via LangGraph | Primary LLM |
| pytest | Testing |

**Key constraint:** FastMCP uses stdout for JSON-RPC. All diagnostic output in `src/` must use `print(..., file=sys.stderr)` or `logging`. Never `print()` in a tool call path.

---

## Data Sources

- UN/LOCODE, IATA — free, real topology
- NOAA AIS (historical, US waters) — free, real signal
- Google Maps Routes API / OSRM — real signal
- OpenSky Network — air schedules, historical, free
- Trucking rates — synthetic distributions (BTS FAF for lane structure)
- **DAT: NOT licensed. Do not use without explicit license.**

---

## File Layout

```
ai-freight-agent/
├── CLAUDE.md
├── CONTEXT.md
├── SESSION_LOG.md
├── pyproject.toml
├── model/          LaTeX formulations (one per component)
├── src/
│   ├── components/ one module per component
│   ├── server.py   FastMCP server
│   └── agent.py    agent loop
├── data/
│   ├── reference/  port codes, network topology
│   └── synthetic/  generated test instances
├── logs/
│   └── agent_interactions.jsonl
└── tests/
    └── components/ one test file per component
```
