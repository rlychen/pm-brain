---
tags:
  - ai-freight-agent
  - index
last_synced: '2026-05-07'
---
# AI Freight Agent — Project Index

Commercial AI-native multimodal freight routing system. Services-as-software (Path A: human approves recommendations). Mid-market freight forwarders and shippers.

**Repo:** `~/Projects/ai-freight-agent/`  
**Current Phase:** 0 — PRD in review

---

## Notes in This Folder

| Note | What It Is |
|---|---|
| [[PRD]] | Full product requirements document — personas, tools, architecture, components |
| [[Rolling Horizon Planning]] | Core architectural concept: G_coarse/G_fine, re-plan triggers, MPC analogy |
| [[Project Guidelines]] | Copy of CLAUDE.md — guardrails, tech stack, build sequence, confidentiality rule |

---

## Key Concepts

- **Rolling Horizon Planning** — maintains full end-to-end plan at all times; future legs on G_coarse, execution legs re-solved on G_fine when ETA confidence threshold exceeded. See [[Rolling Horizon Planning]].
- **Joint multi-echelon optimization** — full door-to-door as a single MILP on G(N,A), not sequential per leg.
- **Probabilistic planning** — transit time distributions (mean + variance), not point estimates; P(on-time ≤ deadline) as optimization objective.
- **LangGraph** — agent orchestration framework; LangSmith for full decision logging; planner → validator loop with independence guarantees.

---

## Phases and Gates

```
Phase 0: PRD ← CURRENT
Phase 1: LaTeX formal models (one per component, each approved individually)
Phase 2: Component builds (one at a time, isolation tests before stitching)
Phase 3: MCP server
Phase 4: Agent layer
Phase 5: Integration testing
Phase 6: Iterate
```

---

## 15 P0 Tools (Must-Have for Prototype)

`route_shipment`, `route_batch`, `carrier_select`, `track_shipment`, `check_on_time_risk`, `portfolio_risk_scan`, `reroute_shipment`, `disruption_impact_scan`, `trade_compliance_check`, `document_requirements`, `tariff_lookup`, `cutoff_alert`, `freight_spend_analytics`, `carrier_scorecard`, `otd_analytics`

---

## Components (14 total)

Graph Generator, Ocean Transit Time Model, Trucking Transit Time Model, Ocean Optimizer, Trucking Optimizer, Multimodal Stitching Layer, Rolling Horizon Controller, Rules Engine, AIS Tracking Adapter, Road Routing Adapter, MCP Server, Planning Agent, Validation Agent, Execution Monitor Agent
