# Project Context

**Last updated:** 2026-05-17 (Session 11 — air model math review complete; ready for user PDF review then LCL)

---

## Current Phase

**Phase 0 — PRD.** PRD v0.3 in review (reorganized 2026-05-16). Not yet formally approved.

**Phase 1 — LaTeX models drafted in parallel** per user direction (don't wait for PRD approval to flesh out math models). **4 of ~11 models drafted; Air model in v2 scope revision:**

| Model | File | Status | PDF |
|---|---|---|---|
| Ocean FCL | `model/ocean_fcl_routing.tex` | Draft v2 | rendered, 677 KB |
| Air Freight | `model/air_freight_routing.tex` | **Draft v2 — P0 cluster + math review complete (Session 11); awaiting user PDF review** | ~3,162 lines LaTeX (estimated ~55–65 pages once rebuilt); user compiles PDF manually per CLAUDE.md rule |
| Ocean LCL | `model/ocean_lcl_routing.tex` | Draft v1 | 14 pages |
| Trucking (FTL/PTL/LTL) | `model/trucking_routing.tex` | Draft v1 | 16 pages |

**Remaining LaTeX models** (~7): Graph Generator, Instance Generator, Transit Time Models (ocean/trucking/air/path), Destination Leg Planner, Rules Engine.

**Phase 2-7 (code):** Not started. User confirmed laptop-feasibility for entire MVP build.

---

## Air model v2 revision — in progress (Session 10, 2026-05-17)

Two adversarial critique agents (technical + practitioner) reviewed `air_freight_routing.tex` Draft v1. User opted to walk through scope decisions point by point before formalizing v2. Approach: opinionated Claude rec → user final call → inline LaTeX edit → immediate PDF rebuild.

**v2b — Practitioner scope (27 tasks total, 9 closed). P0 Critical cluster fully closed.**

| # | Task | Status |
|---|---|---|
| 1 | MAWB / HAWB restructure | ✓ scope agreed; conceptual content in LaTeX §2; decision-vars deferred to v2 MAWB rewrite |
| 2 | DCO + AMS + ICS2 + ACI cutoff set | ✓ added to LaTeX; P.11 + subgraph step 3 use effective cutoff CO_f* |
| 3 | Embargo modeling | ✓ added; pre-filter pattern; 11-field schema; embargo_ok predicate |
| 4 | Lithium battery PI classification | ✓ added; whitelist matrix; commodity attributes; lithium_ok predicate |
| 5 | Supply layer generalization | ✓ added; supply_type enum; co-loader dual-mode; GSA as markup; spot TTL |
| 6 | Through-ULD ψ policy correction | ✓ closed Session 11. ULD interchange set Π added (Star/SkyTeam/oneworld); 3-case rule; operating vs marketing carrier; cross-ULD case; rationale remark + worked example. Bundled Tech C5+M4 (P.14 + rehandling cost linearization cleanup). |
| 7 | Locked-in commitments (K_locked) | ✓ closed Session 11. K^loc + per-arc locked prefix A_k^loc (and locked-off set); 7-state lifecycle → lock-posture mapping; locks derived from lifecycle + execution events; P.19 Locked Commitments constraint (renamed Domain to P.20); sunk costs retained in objective for traceability; lock-induced infeasibility routed as structured rescue event; P1 lock-buyout deferred. |
| 8 | Service-level commitments | ✓ closed Session 11. Named service-product catalog P (PRM_AIR_EXP, STD_AIR, ECON_AIR, MM_EXPEDITED, MM_STD, MM_ECON, OCN_EXP, OCN_STD); per-shipment sp(k) binding; bundle attributes = mode_allow, carrier_allow, carrier_deny, ac_type_allow, T^SLA, handling_tier, cargo_type_min; subgraph pre-filter predicates mode_ok/carrier_ok/ac_type_ok added to flight reachability pass; P.20 Transit-Time SLA hard constraint (renamed Domain to P.21); P1 soft-OTP penalty deferred (item:sla-soft-otp); SLA breach = rescue event. |
| 9 | Carrier blacklist / preference | ✓ closed Session 11. Layered cascade above service product: tenant blacklist → shipper-lane allow/deny → service product → lane preference → commodity overlay. Deny-wins conflict semantics. Resolved per-shipment sets C_k^allow, C_k^deny, C_k^pref. carrier_ok predicate redefined to use resolved sets (Eq. carrier-ok-resolved supersedes sp-carrier-ok). Soft preference via lexicographic two-pass: Pass 1 cost min → z*; Pass 2 max preferred-carrier count s.t. cost ≤ z* + ε^pref. Rules engine is a separate component (own LaTeX model in Phase 1). Time-windowed rules + ML override-learning deferred to rules engine model + Phase 5 constraint learning. |
| 10–22 | P1 important items + practitioner v2 critique pass | Closed via re-run practitioner agent + Groups 1–5 in Session 11. 17 findings triaged: 12 closed in model (incl. surcharge data model, ULD attribute completeness, screening cert, CGC by cargo type, cargo-ready window, supply-side lock invalidation, time-zone convention, B/L release type, currency/FX, shipment attributes doc, clearance dwell); 2 doc-only (sell-rate scope note, booking-rejection recovery); 2 deferred P1 with sourced rate notes (CFS storage/demurrage, partial-split shipment); 1 SKIP-with-note (per-flight lithium aggregate is carrier-side); 1 skip outright (AWB stock). New supporting files: `shipment_attributes.md` (295 lines); `data_model.md` extended with §4 Policy Rules, §5 Spot Rate Snapshots, §6 Surcharge Catalog, §7 Currency/FX (1,148 lines total). |
| 23–27 | Over-engineering drop decisions | Implicitly handled by the user-driven triage in Session 11 — items skipped or deferred as scope decisions during the 17-finding triage. |

**v2a — Technical math correctness pass — superseded Session 11 by fresh 3-agent critique pass on the post-v2b model:**

Session 11 launched 3 parallel critique agents on the post-v2b air model: (1) **notation & formulation correctness**, (2) **linearization & MILP technique**, (3) **simplification & tractability at scale**. The 3 agents produced ~56 findings (15 + 21 + 20) clustered into 5 themes by fix-shape; all bugs and tightening items closed in Session 11. Original v2a list from Session 10 is superseded by this fresher pass.

| Cluster | Items | Status |
|---|---|---|
| **1. Real bugs (correctness)** | 6 — B1 x_f^k undefined, B2 pickup-window not enforced, B3 τ_k overloaded, B4 per_uld surcharge bilinearity, B5 χ binary misstatement, B6 CO_f^* missing k | ✓ all closed (Cluster 1 sweep). New x_f^k shorthand defined; P.21 extended with cargo-ready-window constraint; categorical `\ctype` macro replaces overloaded τ_k; per_uld surcharge re-attributed to flight-level (Path B); χ now continuous [0,1]; subgraph CO subscript fixed. |
| **2. Tightening (correct but loose)** | 5 — T1 per-constraint tight big-M, T2 P.14b → (1−χ), T3 P.10 disaggregation note, T4 P.19 inequality form + pre-solve check, T5 ε^pref ≥ Pass-1 MIP gap | ✓ all closed (Cluster 2 sweep). §10.3 rewritten with per-constraint M^P11/M^P12/M^P13/M^P14a/M^P14b. P.19 uses two-sided inequalities for clean presolve diagnostics. |
| **3. Notation hygiene** | 9 — cargo-type enum canonical {GEN,DGR,PER,VAL,AVI,HUM}, Hub_k(j)/Hub(k) split, wildcard fix, ζ scope, ξ role note, P.18 attribution, F_c(t) definition, function-style convention note, cargo_type_ok predicate | ✓ all closed (Cluster 3 sweep). |
| **4. Tractability roadmap** | 8 simplification levers + 4 strategy notes | ✓ documented as new §Tractability and Scaling Roadmap with sourced benchmarks. Not active model changes; gated on production solve-time data. |
| **5. Strategy notes** | Folded into Cluster 4 in §Tractability section | ✓ documented |

**Net result:** the air model has had a complete math correctness + linearization + notation pass on the v2b structure. Outstanding tech work is now empirical (instrument pre-filter survival rate; measure LP-gap-source post-deployment).

---

## Immediate next steps (start of next session)

**User is doing a personal PDF review of the air model**, then will continue with LCL model work.

1. **Air model status:** Draft v2 ready for user review. ~3,162 lines LaTeX; rebuild for personal PDF review will produce ~55–65 pages.
2. **When user resumes:** they'll pick up either with corrections from their PDF review, or with the LCL model (`model/ocean_lcl_routing.tex`) — which is currently Draft v1 (14 pages, written Session 9, pre-dating the v2b operational additions and the 3-agent math review). The LCL model likely needs the same kind of pass the air model just went through.
3. **LCL work — expected scope:** the operational-realism additions from air v2b (service products, locked commitments, screening, surcharge data model, shipment attributes, time-zone convention, CGC by cargo type, cargo-ready window, customs dwell, B/L release type, currency/FX, supply-side lock invalidation, carrier policy cascade) all apply to ocean LCL with mode-specific adjustments. The math correctness + linearization + notation review pattern (3-agent critique) is the recommended next move once LCL operational scope is locked.
4. **Lesson logged from Session 11:** initial Task-#10 framing as "rolling BSA capacity release" was based on a fabricated tranche schedule. After sourced research (Levin/Nediak/Topaloglu 2012, IATA Net Rates docs, FreightAmigo), retracted and pivoted to spot rate snapshot data model. New memories saved: `feedback_no_fabricated_mechanisms.md`, `feedback_minimal_design_default.md`, `feedback_confirm_before_committing.md`.
5. **PRD review continuation, Graph Generator LaTeX, Transit Time models, Destination Leg Planner, Rules Engine LaTeX** all still pending after LCL.
6. **Obsidian vault** — re-synced at end of Session 11 (after all Cluster 1–4 edits). See `feedback_vault_sync.md`.

## Files modified in Session 11

| File | What changed |
|---|---|
| `model/air_freight_routing.tex` | Massive growth: ~3,162 lines. New §2 Time-zone Convention; expanded §6 with screening cert (§6.12), service products (§6.14), carrier policy (§6.15); expanded §6.1 with cargo-ready window; rewritten §6.4 ULD specs with 16 operational attributes; rewritten §6.7 surcharges with Path-A/B split; new P.20 SLA, expanded P.19 Locked Commitments with supply-side invalidation, P.21 Domain+Initial Conditions; new §Tractability and Scaling Roadmap; updated §10 Linearization with per-constraint tight big-M; new \ctype macro and canonical cargo-type enum |
| `data_model.md` | 1,148 lines. New §4 Policy Rules and Snapshots (generic 3-table framework); §5 Spot Rate Snapshots; §6 Surcharge Catalog (Path-A per-arc + Path-B flight-level); §7 Currency/FX |
| `shipment_attributes.md` | New file, 295 lines. Static + dynamic shipment attribute catalog; lifecycle state mapping; milestone event taxonomy; source-of-truth mapping |
| `CLAUDE.md` | Added "Do not auto-compile LaTeX" rule under guardrails |
| Memory files | `feedback_no_fabricated_mechanisms.md`, `feedback_minimal_design_default.md`, `feedback_confirm_before_committing.md`; updated `feedback_vault_sync.md` date |

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
