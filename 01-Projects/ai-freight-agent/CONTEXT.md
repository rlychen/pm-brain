# Project Context

**Last updated:** 2026-05-17 (Session 11 — Task #6 closed)

---

## Current Phase

**Phase 0 — PRD.** PRD v0.3 in review (reorganized 2026-05-16). Not yet formally approved.

**Phase 1 — LaTeX models drafted in parallel** per user direction (don't wait for PRD approval to flesh out math models). **4 of ~11 models drafted; Air model in v2 scope revision:**

| Model | File | Status | PDF |
|---|---|---|---|
| Ocean FCL | `model/ocean_fcl_routing.tex` | Draft v2 | rendered, 677 KB |
| Air Freight | `model/air_freight_routing.tex` | **Draft v1 → v2 scope revision in progress (session 10)** | ~25+ pages, rebuilt cleanly |
| Ocean LCL | `model/ocean_lcl_routing.tex` | Draft v1 | 14 pages |
| Trucking (FTL/PTL/LTL) | `model/trucking_routing.tex` | Draft v1 | 16 pages |

**Remaining LaTeX models** (~7): Graph Generator, Instance Generator, Transit Time Models (ocean/trucking/air/path), Destination Leg Planner, Rules Engine.

**Phase 2-7 (code):** Not started. User confirmed laptop-feasibility for entire MVP build.

---

## Air model v2 revision — in progress (Session 10, 2026-05-17)

Two adversarial critique agents (technical + practitioner) reviewed `air_freight_routing.tex` Draft v1. User opted to walk through scope decisions point by point before formalizing v2. Approach: opinionated Claude rec → user final call → inline LaTeX edit → immediate PDF rebuild.

**v2b — Practitioner scope (27 tasks total, 6 closed):**

| # | Task | Status |
|---|---|---|
| 1 | MAWB / HAWB restructure | ✓ scope agreed; conceptual content in LaTeX §2; decision-vars deferred to v2 MAWB rewrite |
| 2 | DCO + AMS + ICS2 + ACI cutoff set | ✓ added to LaTeX; P.11 + subgraph step 3 use effective cutoff CO_f* |
| 3 | Embargo modeling | ✓ added; pre-filter pattern; 11-field schema; embargo_ok predicate |
| 4 | Lithium battery PI classification | ✓ added; whitelist matrix; commodity attributes; lithium_ok predicate |
| 5 | Supply layer generalization | ✓ added; supply_type enum; co-loader dual-mode; GSA as markup; spot TTL |
| 6 | Through-ULD ψ policy correction | ✓ closed Session 11. ULD interchange set Π added (Star/SkyTeam/oneworld); 3-case rule; operating vs marketing carrier; cross-ULD case; rationale remark + worked example. Bundled Tech C5+M4 (P.14 + rehandling cost linearization cleanup). |
| 7 | Locked-in commitments (K_locked) | **next up — pending P0** |
| 8 | Service-level commitments | pending P0 |
| 9 | Carrier blacklist/preference | pending P0 |
| 10–22 | P1 important items | pending |
| 23–27 | Over-engineering drop decisions | pending |

**v2a — Technical math correctness pass (27 tasks total, 2 closed by v2b):**

| Bucket | Count | Status |
|---|---|---|
| Critical (C1–C7) | 7 | C3 resolved (MAWB restructure), C6 resolved (cutoff set), **C5 resolved Session 11** (P.14 explicit per-tuple MCT). C4 ↔ Task #24. C1, C2, C7 pending pure-tech fix. |
| High (H1–H6) | 6 | H1 ↔ Task #20, H3 partial via Task #4. H2, H4, H5, H6 pending pure-tech. |
| Medium (M1–M6) | 6 | **M4 resolved Session 11** (Σ_c y activation on P.14). M1+M6, M2, M3, M5 pending pure-tech. |
| Low (L1–L4) | 1 cluster | pending pure-tech notation hygiene batch |
| Reformulation (RC1, RH1–RH3, RM1–RM4) | 8 | RC1 ↔ Task #24. RH1–RH3, RM1–RM4 pending implementation phase. |
| Scalability mitigation doc | 1 | pending |

**v2 plan:** finish v2b walkthrough (22 points remaining), then execute v2a as a single math correctness pass (~17 pure-tech items batched).

---

## Immediate next steps (start of next session — 2026-05-18)

1. **Resume air model v2b walkthrough at Task #7** (Locked-in commitments, K_locked). P0 Critical.
2. **Continue v2b sequentially through Tasks #8–27.** 21 points remaining. Approach validated.
3. **Execute v2a math correctness pass** (~21 remaining tech findings) once v2b complete.
4. **Then formalize as Air Model Draft v2** and submit for LaTeX approval.
5. **After Air Model approved**, return to deferred work: Graph Generator LaTeX model, remaining LaTeX models (Transit Time, Destination Leg Planner, Rules Engine), PRD review continuation.
6. **Obsidian vault sync** — synced 2026-05-17 (start of Session 11): CONTEXT, SESSION_LOG, air model tex+pdf. Re-sync needed after Session 11 edits (air model rebuilt to 31 pages with Task #6 changes).

## Laptop feasibility — confirmed at end of session 9

- **MVP development end-to-end runs on laptop** (modern Mac M1+/Linux, 16+ GB RAM):
  - HiGHS MILP solves: comfortable for 50-100 shipments per batch across all 4 modes
  - LangGraph agent: trivial (API calls only)
  - Local Postgres + Redis + Celery + FastAPI + Next.js: standard dev stack
  - NOAA AIS historical data: fits on laptop
  - End-to-end agent run (single shipment → graph → optimize → recommend): feasible
- **Production scale requires cloud** (1000+ shipments, multi-tenant, live AIS feed, CargoWise integration, 24/7 uptime)
- **Transition point:** Phase 5 (design partner integrations). Until then, laptop-only.

---

## What exists

### PRD and Specialist Files (v0.3 — reorganized 2026-05-16)

The monolith PRD.md (1,666 lines) has been decomposed into 8 specialist files. PRD.md is now the strategic index.

| File | Contents |
|---|---|
| `PRD.md` (v0.3) | Executive summary, problem statement, modes in scope, document map, differentiation requirements, **market opportunity (§6: TAM/SAM/SOM, target customer, commercial model, competitive window)**, open questions |
| `agent_design.md` | AI-native design philosophy (§1), agent capabilities (§2), agent architecture — LangGraph, hierarchical pattern, HITL, capability registry (§3) |
| `data_model.md` | Supply/demand model, G(N,A) graph, arc schemas, container specs, string allocation, rolling horizon planning, customer/tenant entity model (SQL schemas) |
| `ui_spec.md` | Look & feel, color system, screen inventory, persona views, agent action feed, mobile philosophy, wireframes (6 screens), interaction design |
| `personas_and_tools.md` | 4 personas, MCP tool inventory (P0/P1/P2), P0 priority summary |
| `build_plan.md` | Tech stack (FastAPI + Next.js + Clerk + AWS + Celery), multi-tenancy (Postgres RLS), demand generator, peripheral components (Stripe, notifications, Retool, S3), agent execution architecture, data sources, components inventory, build sequence, unit testing |
| `appendices/capabilities.md` | 60+ agent capabilities across 9 categories |
| `appendices/competitive.md` | 7 differentiation gaps, 14-company competitive landscape (updated with project44 Autopilot, Schematics portfolio companies) |

**Adversarial review history:** v0.1 reviewed 2026-05-08 (27 gaps); all agreed changes implemented 2026-05-10; v0.2 rewrite 2026-05-13; v0.3 reorganization 2026-05-16; adversarial critique of v0.3 not yet done.

**PRD review status:** Substantive review stopped after §3.2 (decision confidence tiers) in old PRD = agent_design.md §1.2 in new structure. §3.3 onward (guardrails, deployment modes, routing triggers) not yet reviewed this pass.

### LaTeX models (`model/`)

**`ocean_fcl_routing.tex`** — Ocean FCL Multi-Commodity Routing — Draft v2, 2026-05-13
- Binary Multi-Commodity Network Flow (BMCNF) formulation, P.1–P.5
- Status: draft, not yet approved. Adversarial review done; all agreed changes implemented.
- Key formulation: P.1 flow conservation (N_k indexed), P.2 vessel cap (α·alloc proxy), P.3 string allocation, P.4 budget, P.5 domain
- Open Items: 7; item 7 = vessel capacity proxy (α=0.20 placeholder)

**`ocean_lcl_routing.tex`** — Ocean LCL Routing with Consolidation — Draft v1, 2026-05-16
- Joint bin-packing × routing MILP on 6-layer graph (O, CFS-O, POL, POD, CFS-D, D)
- 16 constraints (P.1–P.16): flow conservation, shipment-to-container, container vol+wt capacity, type-per-slot, sailing cap, arc-sailing linkage, hazmat pair exclusion, CFS cutoff, time propagation, deadline, DGR compat, piece fit, budget, domain
- Two container types modeled (FEU 76 m³/26.5t and TEU 33 m³/24t)
- CFS dwell explicit at both ends (β° = β^D = 1.5 days configurable)
- Sequential decomposition solution strategy documented for tractability (joint MILP for |K|≤50, decomposition + Benders cuts for larger)
- 10 deferred items
- PDF rendered: 14 pages, `model/ocean_lcl_routing.pdf`

**`trucking_routing.tex`** — Trucking Multi-Mode Routing (FTL/PTL/LTL) — Draft v1, 2026-05-16
- Three-mode MILP (FTL, PTL, LTL) with carrier-tendering semantics
- 17 constraints (P.1–P.17) including 6 hard refusal/feasibility constraints from adversarial critique: LTL linear-foot (12 ft), piece length, piece weight, total weight, contract FTL allocation cap (parallel to ocean string), MABD delivery window
- (Carrier, origin SC, destination SC) tuples instead of hub routing — corrects Powell-Sheffi misapplication for forwarder context
- Tender acceptance probability as first-class parameter: `c_exp = c_base × [1 + (1-p_acc)(ρ_re-1)]`
- LTL pricing: NMFC 2025 Standard Density Scale + FAK class override + weight-break tier deficit rule (pre-computed per shipment per arc)
- Time discretization to days + slot lex-ordering symmetry breaking
- Container-to-chassis: one ocean container per truck (TEU/FEU compatibility)
- 14 deferred items including DOT Hours of Service, backhaul optimization, specialty equipment, dedicated lanes, pool distribution, cross-border, stochastic tender modeling
- Academic anchors: Powell-Sheffi (1983), Powell ADP/Optimal Dynamics, Caplice MIT FreightLab, Sheffi (1985), Erera (2020), 2025 NMFC SDS
- Adversarial critique applied (10 corrections, see §10)
- PDF rendered: 16 pages, `model/trucking_routing.pdf`

**`air_freight_routing.tex`** — Air Freight Multi-Commodity Routing — Draft v1, 2026-05-16
- BMCNF on commodity-specific subgraph G(N_k, A_k) of air network
- 19 constraints (P.1–P.19): flow conservation, ULD volume+weight capacity, per-flight contract cap, aircraft position cap, period cap, hard BSA take-or-pay (M from upstream model), arc-to-ULD linkage, spot/contract exclusivity, pivot weight linearization, cargo cutoff, time propagation, hub MCT, deadline, cargo type compat, ULD physical fit, budget, domain
- Two air capacity layers modeled simultaneously: contracted BSA (LD3/LD7/PMC/AKE per-flight allocation, pivot weight take-or-pay, hard vs. soft) + spot rate-card (IATA weight breaks)
- Each scheduled flight leg is one arc (multi-stop flights decompose into multiple arcs)
- Through-ULD vs. re-ULDing at hub: parameter ψ in MVP (decision var deferred to P1); MCT and rehandling cost differential
- Linearization: pivot weight max() via aux var, bilinear y·y via McCormick, big-M tightening
- Period commitment M_{c,u,t} computed by separate upstream model — routing MILP takes as fixed input per run
- 10 deferred items
- PDF rendered: 17 pages, `model/air_freight_routing.pdf`
- Status: draft, not yet approved

### Competitive research (`Research.md`)
- Created 2026-05-13; 33 URLs, 14 companies
- Full company profiles: capabilities table, autonomy level, operator UI model, guardrails detail
- Synthesis: 8 industry patterns, 5 market gaps
- **Not yet synced to Obsidian vault**

### Adversarial competitive critique (`appendices/competitive.md` — updated May 2026 session 7)
- Added 6 companies (DSV/Tango, Optimal Dynamics, WiseTech detail, DecisionBrain, project44 Intelligent TMS detail, cargo.one updated profile) — now 16 companies total
- C.5: Honest assessment of where MILP differentiator holds vs. doesn't
- C.6: Moat analysis (weak years 1–2; override data + integration depth as path to durable moat by year 3–4)
- C.7: Three specific attack scenarios (cargo.one, WiseTech CargoWise bundling, Tier 1 productization) with timelines and probabilities
- C.8: Source list for adversarial critique

### Obsidian vault mirrors
- `~/Documents/PM-Brain/01-Projects/ai-freight-agent/PRD.md` — last synced 2026-05-10 (v0.1)
- `~/Documents/PM-Brain/01-Projects/ai-freight-agent/ocean_fcl_routing_model.md` — last synced 2026-05-10
- **Both stale — need sync at start of next session**

### New reference docs (`docs/`)

- **`docs/freight_concepts.md`** — Freight domain glossary: HBL/MBL pairing, container lifecycle (16 stages), booking flow, B/L release types, trucking instructions, road consignment note, intermodal rail booking, ULD types and stored fields, chargeable weight formula, surcharge stacks (ocean + air), US customs filings (AMS/ISF/EEI), carrier alliances
- **`docs/taiwan_market.md`** — Taiwan market analysis: TAM $15–20M / SAM $1.5–5M / SOM $300K–1M ARR; top 20 forwarders with TMS status (Dimerco = proprietary, Morrison = likely CW); Dimerco deep dive (Value Plus System®, API options, IATA ONE Record); competitive software landscape; design partner sequencing (Dimerco #1, Morrison #2, King Freight #3)
- **`docs/us_market.md`** — US market analysis: TAM $75–160M / SAM $25–50M / SOM $2–8M; major US Tier 2 forwarder list with TMS; US TMS landscape (CargoWise dominant, GoFreight growing); regulatory complexity (ISF/AMS/PGA); sales motion, conference channels (NCBFAA, TPM Long Beach), ACV targets

### Diagrams (`docs/`)
- `ocean_fcl_planning_graph.drawio` — FCL planning graph, Shenzhen → Chicago example
- `ocean_fcl_multi_shipment_graph.drawio` — multi-shipment graph (referenced in LaTeX abstract)

---

## Key decisions made

| Decision | Detail |
|---|---|
| Agent framework | LangGraph (not direct Anthropic SDK). LangSmith for observability. |
| MILP solver | HiGHS (`highspy`) |
| Container type (MVP FEU) | 40'HC — 76 CBM usable, 26,500 kg payload |
| Capacity unit | TEU slots throughout. BSA contracts in FEU → ×2 at input. |
| Graph architecture | Commodity-specific subgraphs G(N_k, A_k); 6-step BFS construction |
| Decomposition | Commodity-supply graph H; connected components solved independently |
| Mix algorithm | Explicit enumeration over f from f_min to 0 (not greedy) |
| n_k formula | max(ceil(v_k/76), ceil(w_k/26500)) |
| Instance generation | Demand-first; joint session required before implementation |
| Autonomy model | AI routes autonomously; humans govern exceptions only |
| Deployment modes | Co-pilot / Supervised / Autonomous — customer chooses; progressive trust expansion |
| Confidence tiers | Tier 1 (auto, ≥0.80) / Tier 2 (recommend) / Tier 3 (escalate, <0.50 always escalates) |
| Approval model | Dry-run window (60 min default, 15 min urgency); auto-commits on expiry |
| UI primary surface | Exception Queue + Operations Dashboard |
| Grouping options | By carrier, shipper, or receiver (operator chooses) |
| Options per shipment | Top 3: cheapest, fastest, most reliable |
| Override policy | Requires reason; logged to overrides.jsonl for constraint learning |
| Kill switch | Global + per-lane |
| Agent reasoning | 3 levels: feed line, exception paragraph, full LangSmith trace |
| Closest competitor | cargo.one — multimodal, AI-native, MCP, $20M Bessemer; no MILP optimization layer |

---

## LaTeX model — formulation summary

- **P.1** — Flow conservation, indexed over N_k
- **P.2** — Vessel capacity cap: Σ_{k:(i,j)∈A^k_OC} slots(k,ij)·x_{ij}^k ≤ α·alloc(s_{ij},t_{ij}), α≈0.20
- **P.3** — String allocation cap: sum over sailings of string s in period t ≤ rem(s,t)
- **P.4** — Budget cap (optional, per commodity)
- **P.5** — Domain: x∈{0,1}
- **Open Items** — 7 items; item 7 is vessel capacity proxy (α=0.20 placeholder)
- **P1 deferred** — multi-HS-code commodity schema, booking cutoff, PSS surcharge, per-lane σ split

---

## PRD — open questions (§15)

1. Decision-support vs. autonomous execution trigger conditions
2. Design partner selection
3. Live AIS feed vendor evaluation
4. Carrier booking APIs for autonomous execution
5. Pricing model
6. Emissions data source
7. Multi-agent framework — decided: LangGraph
8. LCL consolidation optimizer scope
9. Time-phased carrier capacity release (P1)

---

## Competitive intelligence — key findings

| Finding | Detail |
|---|---|
| Closest peer | cargo.one — multimodal (air+ocean), AI-native OS, MCP-connected, $20M Bessemer, same mid-market forwarder target. No MILP layer. |
| Best guardrail model | Shipsy AgentFleet — 8 documented guardrails, three confidence tiers, 94.2% autonomous resolution in production |
| Best progressive trust model | cargo.one — Co-pilot / Supervised / Autonomous deployment modes; project44 Autopilot — progressive trust with recommendation-only default |
| Primary market gap | No company has published MILP-based joint optimization for freight forwarders. Everyone does intelligent matching or procurement automation, not constraint-optimal routing. |
| Industry benchmark | 94.2% autonomous task resolution (Shipsy production). This is the target for "working at scale." |

---

## Guardrails reminder

- PRD must be approved before LaTeX models are approved
- LaTeX models must be approved before code starts
- No scope expansion without explicit confirmation
- Instance generator is a joint session — do not implement independently
- Do not reference banned company/product names (see CLAUDE.md)
- Read `Research.md` at session start if doing competitive analysis work
