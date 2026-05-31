# Open Decisions and Pending Edits

Working catalog of changes proposed across recent sessions. **Nothing here is committed yet** — this is the queue for review and approval before edits land. Triage at sign-off.

Last scan: 2026-05-25, covering Session 18 (forwarder-operations-analysis: 4 parallel research agents on front office / network ops / compliance-customs / exceptions-replanning + synthesis). Session 17 backlog (items A1–G3) is unchanged below.

---

## A. `model/air_freight_routing.tex` (the MILP doc)

| # | Change | Trigger | Priority | Status |
|---|---|---|---|---|
| A1 | Define a per-shipment **slack metric** for SLA buffer; design spec needed before LaTeX edit | Conversation 2026-05-24 17:00 | Med | Open — design needed |
| A2 | Brief paragraph in §5.7 surcharge text noting commit-time **rate surprise** (live API rate may differ from planned snapshot) | Conversation 2026-05-24 17:30 | Low | Open |
| A3 | Brief paragraph in §5.6 clarifying **equalized-BSA semantics**: sunk allowance, marginal cost is zero up to $A_c$, overage at $r_c$/kg | Conversation 2026-05-24 16:30 | Low | Open |
| A4 | **§4.3** — explicitly enumerate all possible consolidation groupings as a table (the 6-row consolidable + VAL/HUM/AVI singletons) | Session note 2026-05-24 15:59 | Low | Open |

**Net:** small additions, no math change.

---

## B. `agent_design.md` (largest restructure)

This doc currently mis-describes its own subject. The 2×2 matrix is decision-routing workflow, not agent architecture. Multiple sections need to be added or rewritten.

| # | Change | Trigger | Priority | Status |
|---|---|---|---|---|
| B1 | **Restructure the doc** into Part I "Agent Architecture" (planner-validator loop, tools, memory, LangGraph state machine) and Part II "Workflow & Decision Routing" (lifecycle, flags, commit timing). One file, two parts. | Conversation 2026-05-24 17:00 | High | Approved approach |
| B2 | **Delete the Risk × Confidence 2×2 matrix.** Replace with the lifecycle model (soft planning → flags → commit). See Appendix A in this doc for the exact lifecycle text. | Conversation 2026-05-24 17:30 | High | Open |
| B3 | **Flag taxonomy as orthogonal dimensions** (SLA risk, cost outlier, rate surprise, capacity risk, disruption), with per-flag resolution paths. | Conversation 2026-05-24 17:30 | High | Open |
| B4 | **Re-plan trigger taxonomy** — 8 event-driven triggers + time-driven (periodic, pre-commit). | Conversation 2026-05-24 17:45 | High | Open |
| B5 | **Commit-window policy** — `commit_offset(k) = cutoff(k) − tier_safety_margin(sp(k))`; defaults Express 6h, Standard 12h, Economy 24h. Tunable per tenant. | Conversation 2026-05-24 17:45 | High | Open |
| B6 | **Override rate as central KPI** — tighten trust-degradation trigger from 15% to **8%** (recommended default; calibrated after MVP data). Rewrite metric section accordingly. | Conversation 2026-05-24 17:00 | High | Open — needs final number |
| B7 | **Agent role taxonomy** — explicit "useful" (exception triage, conversational query, request decomposition, override-log pattern detection, customer comms) vs "not useful" (MILP, ML, rate calc, rules, routine re-solves, carrier APIs). | Conversation 2026-05-24 17:30 | High | Open |
| B8 | **Failure-mode taxonomy** — structured reason codes for overrides: `data-stale`, `constraint-missing`, `wrong-carrier`, `wrong-route`, `transit-time-off`, `operator-preference-not-modeled`, `rate-outdated`, `other`. | Conversation 2026-05-24 17:00 | Med | Open |
| B9 | **Deployment ladder per customer** — Co-pilot ≥4 weeks → Supervised ≥8 weeks → Autonomous per-lane (not blanket). Calendar minimums. From the 95% accuracy trap article. | Conversation 2026-05-24 17:00 | Med | Open |
| B10 | **OOD detection** replaces "familiarity count" as the ML model quality signal. Familiarity-as-count is redundant with model variance; OOD is the principled flag for "we don't know what we don't know." | Conversation 2026-05-24 17:30 | Med | Open |

**Net:** ~50% of `agent_design.md` is being rewritten. Largest single doc change in this round.

---

## C. `data_model.md`

| # | Change | Trigger | Priority | Status |
|---|---|---|---|---|
| C1 | **Rate-sourcing taxonomy** — 5-tier strategy (contract / live API / snapshot / TACT / prediction). Each rate carries source + freshness metadata. Default aggregator: WebCargo. | Conversation 2026-05-24 17:45 | High | Open |
| C2 | **Flag log schema** — flag type, shipment_id, plan_id, threshold breached, timestamp, resolved-by, resolution action | Conversation 2026-05-24 17:30 | Med | Open |
| C3 | **Override log schema** — add structured `reason_code` field (B8 codes) + free-text. Specified in CLAUDE.md as `logs/overrides.jsonl` but format not yet defined. | Conversation 2026-05-24 17:00 | Med | Open |
| C4 | **Plan lifecycle schema** — plan_ids, plan versions per shipment, commit timestamps, supersedence chain. Needed for the soft-plan model. | Conversation 2026-05-24 17:30 | Med | Open |

---

## D. `build_plan.md`

| # | Change | Trigger | Priority | Status |
|---|---|---|---|---|
| D1 | **Per-customer deployment ladder** in Phase 5/6 — 1 lane → multiple lanes → multiple customers → scale. Article-anchored cadence (3 → 30 → 300 → 3000 scoped per customer). | Conversation 2026-05-24 17:00 | Med | Open |
| D2 | **Rate-source integration phase** — sequence aggregator integration (WebCargo MVP) + carrier-direct fallbacks + snapshot catalog + TACT subscription. | Conversation 2026-05-24 17:45 | Med | Open |

---

## E. `PRD.md`

| # | Change | Trigger | Priority | Status |
|---|---|---|---|---|
| E1 | **Agent role clarity** — explicit that AI agent is interface/orchestrator/exception-handler, NOT the routing brain. MILP + ML + rules do the routing. Avoids "AI does everything" expectation drift. | Conversation 2026-05-24 17:30 | High | Open |
| E2 | **KPI section** — override rate as the central operational metric, not model accuracy. | Conversation 2026-05-24 17:00 | High | Open |

---

## F. Memory (auto-memory) — DONE this round

| # | Change | Trigger | Status |
|---|---|---|---|
| F1 | New project memory: **Plan-Goodness Reframe** | Conversation 2026-05-24 17:30 | ✓ Saved |
| F2 | New project memory: **Agent Role Taxonomy** | Conversation 2026-05-24 17:30 | ✓ Saved |
| F3 | New project memory: **Override Rate is the KPI** | Conversation 2026-05-24 17:00 | ✓ Saved |
| F4 | New reference memory: **Rate API Landscape 2026** | Conversation 2026-05-24 17:45 | ✓ Saved |

---

## G. New artifacts created this round

| # | Artifact | Status |
|---|---|---|
| G1 | `architecture.md` — system architecture overview with lifecycle diagram | ✓ Drafted |
| G2 | `docs/architecture.drawio` — simple/clean system diagram | ✓ Drafted |
| G3 | `OPEN_DECISIONS.md` (this file) | ✓ Drafted |

---

## H. From forwarder-operations-analysis (Session 18, 2026-05-25)

The 4-agent + synthesis analysis surfaced 12 new open items. Full derivation in `docs/forwarder-operations-analysis/00-synthesis.md` §8. Tracked here for triage.

| # | Decision | Recommendation | Priority | Status |
|---|---|---|---|---|
| H1 | **MVP LLM-agent capability set** — refine from 2 to 4 capabilities | input parsing + ad-hoc query + materiality assessment + re-plan trade-off explanation (per synthesis §5) | High | Open — supersedes `project_agent_role_taxonomy.md` MVP scope if approved |
| H2 | **TMS integration partner architecture** — single primary or pluggable? | Pluggable, with CargoWise as v1 reference (mid-market dominance) | High | **Closed by I2 (2026-05-25)** — pluggable adapter; priority CargoWise → Magaya → GoFreight → Riege |
| H3 | **Email-parse vendor — Sedna vs Wisor vs Shipamax vs cargo.one** | Integrate one as input adapter; focus build energy on WhatsApp/voice gap. Sedna stronger on shipping email; Wisor on quote-desk. | Med | Open — needs vendor demos |
| H4 | **WhatsApp Business API integration as the project's owned-AI-surface** | Yes — F3 of synthesis identifies as least-crowded AI wedge. Need legal / partner-consent design. | High | Open — v1 vs v2 question raised by Agent 4 |
| H5 | **Visibility platform partner** — project44 / FourKites / Beacon / GoComet | Build signal layer with adapters; subscribe per design partner | Med | Open |
| H6 | **Customs data into the optimizer — which subset is MVP?** | v1: HS-restricted lanes + FTA-effective duty per lane; v2: PGA-supporting ports + bond capacity (synthesis §10) | Med | **Closed by I4 (2026-05-25)** — integrate-don't-build; four intersection points (cost / constraints / transit-time feature / replan trigger) |
| H7 | **Schedule-reliability priors per carrier/alliance as MILP input** | New input — Sea-Intelligence variance Gemini 90%+ vs Premier ~57%. Schema + source decision needed. | Med | Open |
| H8 | **Materiality threshold calibration source** — per-customer / per-tier / per-lane | v1: per-service-product tier (Express/Standard/Economy) with per-customer override hook for v2 | Med | Open |
| H9 | **Tender-acceptance-probability ML for drayage dispatch (I2)** | New ML model. Decision: build now or defer past MVP? | Low | Open |
| H10 | **D&D-aware drayage acceleration as MILP sub-model** | New. Decision: in trucking model v1 or v2? | Low | Open |
| H11 | **cargo.one competitive scoping** — paid trial / sales conversation needed | Their March 2026 MCP-compatible AI-native OS is the closest competitive overlay. Scope before locking agent boundary. | High | **Closed 2026-05-25** — deep-dive primary-source research confirmed: cargo.one = Bucket B (rate aggregation + per-shipment quoting + booking with RAG/LLM workflow automation). Zero overlap with consolidation planner / multi-shipment optimization. The Feb–Mar 2026 "AI-native OS" launch is workflow-automation rebrand + Cargofive ocean rates, not new optimization. Methodology = learned-preference ranking, not MILP. MCP claim is positioning, not shipped tools. **Adjacent platform / potential booking complement, not consolidation competitor. Wedge holds.** Paid trial not needed; sales conversation optional (forward-looking only). Monitor Ashby for OR engineer hires as leading indicator. |
| H12 | **Quote-response-time as a headline pitch — yes or no?** | NO. Commoditized (Wisor / cargo.one / Expedock all cover). Optimizer's pitch is mode-choice + consolidation. Let vendors own quote-turnaround narrative. | High | **Closed by I9 (2026-05-25)** — confirmed NO. Quote desk is internal MVP surface; pitch is cost-to-serve grounded in optimization + speed combined |

**Cross-cutting note from synthesis:** The four-bucket framework (workflow / hybrid / AI-agent-novel / MILP) should also live in `agent_design.md` Part I when B1 restructure happens. Mark to cross-reference there.

---

## I. From Session 18 third continuation (2026-05-25)

User walked through the four day-in-the-life docs (`docs/forwarder-operations-analysis/day-in-the-life/`) and locked five strategic commitments to memory + one secondary-surface call still pending. All saved as memory; doc updates landed in CONTEXT.md, SESSION_LOG.md, architecture.md §11, `docs/architecture.drawio` (new five-layer stack), PRD.md §3.6, EXECUTION_PLAN.md.

| # | Decision | Outcome | Priority | Status |
|---|---|---|---|---|
| I1 | **MVP user-surface scope** | Two prongs: quote desk (customer-facing, Persona 1) + consolidation planner (internal, Persona 2). Same MILP engine, two UIs. | High | Closed — memory `project_two_pronged_wedge.md` |
| I2 | **Above-TMS positioning + TMS-agnostic adapter** | Intelligence layer above TMS; not replacing. Adapter interface priority: CargoWise → Magaya → GoFreight → Riege. Regional secondaries Logitude / Softlink / AKANEA. Four TMS gaps the project fills (path-based cost-to-serve, stateful shipment graph + model-ETA, capacity-aware quoting, cross-mode stitching). | High | Closed — memory `project_intelligence_layer_positioning.md`; supersedes H2 |
| I3 | **Density-fit packing architecture** | Assignment problem (volume + weight + segregation) + ML feasibility predictor + replan-if-below-threshold. REJECT 3D bin packing (over-precise, operators won't follow, catastrophic failure mode). Threshold = business control (start 95%, tunable per customer / lane / time). Failed packs → labeled training data. Cold-start with heuristic predictor. | High | Closed — memory `project_density_fit_architecture.md` |
| I4 | **Customs persona = integrate-don't-build** | Partner-integration boundary, not user surface. AI-commoditized across HS / doc parse / DGD / screening / FTA / e-B/L / filing. LCB attestation non-delegable. Four data intersection points: (1) cost component for quoting, (2) hard constraints in graph (DG/screening segregation), (3) stochastic feature in transit time estimator (P(customs hold)), (4) event trigger for replan. | High | Closed — memory `project_customs_integrate_dont_build.md`; supersedes H6 |
| I5 | **Persona 4 ≈ Persona 2 at mid-size** | Same humans, two operational modes (planning vs reactive). Replan UX = planning UX (same MILP, different trigger + boundary conditions). No separate exception-handler UI in Phase 5. | Med | Closed — updated memory `project_core_user_reality.md` |
| I6 | **Drayage dispatcher = execution, not planning** | User decision 2026-05-25: drayage dispatcher (real-time tender cascade, driver SMS, gate-time monitoring) is *execution* and moves to P1 / Phase 7. Closer-to-product MVP-secondary surface is drayage / trucking pickup *planning* — see I13. | Med | **Closed by user** — P1 / Phase 7 |
| I7 | **KAM AI surfaces = post-MVP** | Auto-QBR, at-risk-account flagging, cross-sell signal, account briefing, forecast support, contract renewal prep. All amplification not displacement; relationship trust + executive judgment are non-displaceable. | Low | Closed — after MVP |
| I8 | **CFS supervisor as user surface** | NO. GHA-owned labor; physical judgment calls. Density-fit handled inside the consolidation MILP (per I3), not as a separate UI. | Low | Closed — not addressed |
| I9 | **Quote-response-time as headline pitch — final** | Confirmed NO (was H12). Quote desk is internal MVP surface but pitch is cost-to-serve grounded in optimization + speed combined, not pure response-time pitch. | High | Closed — supersedes H12 |
| I10 | **System diagram structure** | Five-layer stack: User surfaces → Agent layer → Intelligence components (we build) → Data adapters → External systems (we integrate). | High | Closed — `docs/architecture.drawio` replaced; narrative in `architecture.md` §11 |
| I11 | **Density-fit ML predictor — Phase 1 doc placement** | User decision 2026-05-25: **standalone Phase 1 design doc**, not subsumed into per-mode LaTeX. Proposed filename `model/density_fit_feasibility.md` (markdown, similar to `model/rules_engine.md` — ML spec, not LaTeX MILP). To be drafted when Phase 1 begins for consolidation MILP. | Med | **Closed by user** — standalone Phase 1 doc |
| I12 | **Per-mode MILP scope update (LCL §2.8, Air §2.9)** | Constraints are volume + weight + segregation only. Physical pack feasibility is a separate ML call. | Med | Closed — to apply when those components begin |
| I13 | **Drayage / trucking pickup planning as MVP secondary surface** | Route sequencing (VRPTW) + appointment booking + carrier allocation across drayage panel + acceptance-probability priors fed to MILP. Planning work — fits project DNA. Exact scope per-mode (drayage vs LTL pickup vs FTL pickup) to define when surface begins. | Med | Closed by user 2026-05-25 — MVP secondary surface |
| I14 | **Planning vs execution boundary as scoping principle** | Project's DNA is planning. Execution surfaces (real-time dispatch, physical supervision, regulatory filing, gate monitoring) are out of scope or integrated against, not built. Memory `project_planning_vs_execution_boundary.md` enumerates planning vs execution surfaces. Use this distinction when scoping any new surface. | High | Closed — durable scoping principle |

**Cross-cutting note:** I1–I14 are durable strategic commitments. All MVP-scoping decisions closed by user 2026-05-25. Next phase begin (Phase 1 density-fit doc; Phase 5 product layer with three surfaces; Phase 7 dispatcher / drayage execution) inherits these constraints.

---

## Open product decisions (need your call before edits land)

**From Session 17 backlog:**

1. **Override-rate degradation threshold** — 5% / 8% / 10%? (8% recommended)
2. **`agent_design.md` restructure** — one file two parts? (recommended) or split into two files?
3. **MVP rate aggregator** — WebCargo? (recommended; largest coverage) or cargo.one or CargoAi?
4. **Architecture diagram location** — top-level `docs/`? (recommended) or `model/docs/`?
5. **Commit-window safety margins** — Express 6h / Standard 12h / Economy 24h defaults? Confirm.
6. **Cost outlier multiplier N** — what's the Nx-of-lane-median threshold for flagging? 2x? 3x? 5x?

**From Session 18 (this round) — see H1–H12 above.** Top three needing user input first:

7. **H1 — confirm 4-capability MVP agent scope** (replaces the 2-capability list in memory)
8. **H4 — WhatsApp Business API in v1 or v2** (depends on legal/consent design effort) — *partially answered by J6 below; H4 is now superseded for the inbound-intake scope*
9. **H11 — cargo.one paid trial / sales convo before locking agent boundary** (largest competitive unknown)

---

## J. From Session 20 critique pass + design decisions (2026-05-27)

Triggered by 4-agent critique run on the v3 air model (commercial viability / consolidation savings / gap finder / persona test) plus user-driven design decisions on orchestrator and messaging-agent layers. Critique outputs in `docs/critique/01-04` files. Messaging-agent prior-art research in `docs/critique/05-messaging-agent-prior-art.md` (pending — Session 20 background agent).

### J. Locked design decisions (user 2026-05-27)

| # | Decision | Rationale | Priority | Status |
|---|---|---|---|---|
| J1 | **Four UI/UX surfaces are MVP scope, not v2.** Console (planner view) + rule-author (carrier-policy / service-product / allotment editor) + override-log (first-class capture with structured reason codes) + orchestrator-config UI. | Per agent-4 (persona test): planner is a senior coordinator wearing the hat 2-4 hr/day, interrupt-bursty. Without these surfaces the MILP is unusable in workflow — week-1 rejection risk even though the math is correct. Override log is also load-bearing for the override-rate KPI (memory `project_override_rate_kpi.md`). | High | **Closed by user 2026-05-27** |
| J2 | **Orchestrator = scheduled job with manual-trigger escape hatch.** Configurable cadence at deployment (default 3 runs/day: 1 global + 2 update runs). Planner does NOT need to trigger automatically. Manual trigger always available from the console. | User decision 2026-05-27. Operationally honest: planners are interrupt-bursty, can't be expected to remember to push a button. Scheduled cadence is the workhorse; manual is the escape valve. | High | **Closed by user 2026-05-27** |
| J3 | **Concurrency model for orchestrator (design needed).** Considerations to address: (a) snapshot semantics — every solve binds to a snapshot of rate catalog + BSA accumulator + supply data at run start; (b) two simultaneous runs on the same snapshot must produce identical output (idempotency); (c) deadlock risk on shared resources (BSA accumulator updates, plan supersedence chain, rate-catalog reads); (d) what wins when manual trigger fires during a scheduled run — cancel + restart, queue, or run concurrently with different scope; (e) plan supersedence chain (see C4 from Session 17 backlog); (f) result-merge when a scheduled run completes after a more-recent manual run. | Surfaced by user 2026-05-27 alongside J2. Needs design before MVP code. | High | **Open — design needed** |
| J4 | **Messaging-channel agent — three-layer stack (intake + parsing + active participation).** Expand the prior unstructured-channel wedge (memory `project_unstructured_channel_wedge.md`) into a three-layer design. **Layer 1 (passive listening):** extract events from in-channel comms (flight roll, allotment cut, cargo-readiness slip). **Layer 2 (inbound intake):** shipper texts → candidate HAWB → MILP test-route → price + SLA back in-thread. **Layer 3 (active participant) — NEW 2026-05-27:** agent asks questions, requests missing data, proposes corrections to supply catalog with HITL confirmation, diagnoses infeasibility (graph analysis on which arc / constraint is binding), proactively procures spot capacity on identified bottlenecks. Channels: WhatsApp (US/EU/APAC), LINE (Taiwan/Japan/Thailand), email, SMS; WeChat (China) secondary. Pattern: proposer → confirmer(s) → applier with full audit trail. | User decision 2026-05-27. The three-layer stack closes the disruption-handling loop end-to-end: messages in (Layer 1), requests in (Layer 2), operational action out (Layer 3). Connects to override-log KPI, trust ladder, orchestrator (Layer 3 triggers re-solves), fallback-arc design (Layer 3 acts on fallback HAWBs). Competitive: per `docs/critique/05-messaging-agent-prior-art.md`, **no incumbent does Layer 3** — Shipsy Clara has channels + outbound only, Sedna has listen + write-back but no optimizer wiring, HappyRobot has channels but FTL-broker scope. Combined three-layer + multi-shipment MILP is the empty corner. | High | **Closed by user 2026-05-27** (3-layer architecture); deep-dive agent running on capability enumeration |
| J17 | **Layer-3 capability deep-dive (research pending).** Full design space of active-participant agent capabilities; recommended MVP subset; HITL pattern per capability; trust ladder per surface; failure modes and abuse vectors. Output to `docs/design/messaging_agent_capabilities.md`. | Triggered by user 2026-05-27 active-participant expansion. | High | **Research pending — Session 20 deep-dive agent** |
| J18 | **Pitch deck v7 update — active-participant agent.** Layer 3 capabilities (supply correction loop, infeasibility diagnosis + spot procurement, proactive capacity discovery, rate validation, cargo-readiness slip detection, carrier-policy capture, status outbound, dispute resolution, temporary embargo capture, co-loader negotiation) are pitch-grade differentiators because no incumbent does them. Frame as "the agent that operates with you, not next to you." Pair with J14 (consolidation savings %) and J15 (4h → 90min) for the three pitch upgrades from Session 20. | Triggered by user 2026-05-27. | High | Open — incorporate into pitch v7 |
| J19 | **Time-propagation modeling: do we need C.6 per-arc, or can we precompute end-to-end path traversal at graph layer?** User intuition (sign-off 2026-05-27): if transit times are deterministic, sum across arcs is precomputable. If uncertain, end-to-end transit-time *distribution* is computable, and quantile-based arrival (P85/P90/P95) + a tardiness-risk penalty in the objective should ensure high-prob arrival for premium shipments. Architectural simplification candidate: eliminating C.6 + per-shipment $t_k^n$ variables removes ~1,500 constraints + ~2,500 continuous variables from base-scale estimate. **Open questions for the next-session discussion:** (a) what binds at intermediate nodes that requires per-node arrival times (cutoff at intermediate POL via C.9, MCT at hubs absorbed into $\mu_a$ vs. C.6, locked-commitment timing)? (b) under consolidation, multiple HAWBs share a MAWB-arc — does per-MAWB arrival suffice if per-HAWB is the same on the shared sub-path? (c) how does quantile-based arrival interact with the fallback arc (which currently sets arrival = $T_k^{\text{abs}}$ exactly via C.6)? (d) what's the relationship to the existing Finding S Ch 1 hook (TT-Service P85-P90 quantile as later-phase replacement for $\mu_a$)? | Triggered by user 2026-05-27 sign-off. | High | **Closed 2026-05-27 — forward-time-window pruning at graph build.** Mechanism: per-HAWB forward-propagate arrival window $[t^{\text{lo}}_n, t^{\text{hi}}_n]$ from origin ready window; admit an arc into a flight-tail node iff propagated lower bound ≤ cutoff $\text{CO}_a^*$; admit other arcs by propagating the window forward; flight head nodes anchor to scheduled ETA. C.6 + C.9 eliminated; intermediate $t_k^n$ variables removed; only $t_k^{D_k^{\text{node}}}$ survives, defined by C.10a as $\sum_{a \in \mathcal{A}^{\text{last}}_k} \text{arr\_dest}(k, a) \cdot x_{k,a}$ over terminal arcs (POD-landing air arcs + fallback). Resolves all four sub-questions: (a) intermediate-node bindings pre-resolved by pruning; (b) per-HAWB DAGs naturally synchronize on shared MAWB sub-paths via flow conservation; (c) fallback is one more terminal arc with arr\_dest = $T_k^{\text{abs}}$; (d) Finding S quantile substitutes both into propagation (window) and into arr\_dest (end-to-end quantile per terminal arc). Air model edits landed in `model/air_freight_routing.tex` 2026-05-27 (new §forward-time-window-propagation; C.6/C.7/C.8/C.9/C.11 removed; C.10 rewritten as C.10a + C.10b; variable / domain / big-M / scaling sections updated). Base-scale variable count drops $\sim 4{,}500 \to \sim 2{,}100$ continuous; constraint count $\sim 10{,}500 \to \sim 8{,}000$. |

### J. Critique-driven gaps to triage (from agents 1-4)

| # | Gap / Finding | Source | Priority | Status |
|---|---|---|---|---|
| J5 | **Operator-facing translation layer for plans.** Plain-language per-shipment "why this routing" + plan-vs-plan delta + binding-constraint readout. `architecture.md §7.2` names it for re-plans only; planning case unscoped. | Agent 4 (persona test) top gap | High | Open |
| J6 | **Sensitivity / shadow-price / binding-constraint readout.** Add to §Output and Diagnostics: dual values, slack, "what would change if we relaxed X." Small additive on top of existing solve. | Agent 4 | High | Open |
| J7 | **Plan-to-execution serialization** (build sheets, tender instructions, customer comms). Persona 2 coordinators / CFS supervisors and Persona 4 handlers all consume MILP output through this missing layer. | Agent 4 | High | Open (gated on Phase 5) |
| J8 | **Data-feed delivery as the real critical path.** Rate catalog (5-tier), BSA accumulator, schedule + cutoff + MCT, carrier eligibility tables — model assumes these are clean but they're data-engineering deliverables. Agent 1 names this as the #1 blocker to year-one automation. | Agent 1 (commercial viability) | High | Open — adjust EXECUTION_PLAN |
| J9 | **Per-flight carrier daily allowance variance.** BSA `N_{a,u}` is contract-level; carriers cut daily allotment when pax-belly capacity gets pulled. Model treats BSA as date-invariant. | Agent 3 (gap finder) top gap | Serious | Open |
| J10 | **Time-windowed / seasonal carrier rules.** Hajj / monsoon / CNY rules are the predominant rule shape in peak season. Currently `Out of MVP scope` — agent argues this is wrong-sized for the override-rate KPI. | Agent 3 | Serious | Open — reconsider MVP scope |
| J11 | **Cargo-readiness slip as a structured replan trigger.** Most common pre-departure exception per DITL, but the supply-side lock-invalidation list enumerates only carrier events. | Agent 3 | Serious | Open |
| J12 | **Allocation pull-forward / cross-period BSA dynamics.** Equalized-allowance correct within a batch but loses the use-it-or-lose-it monthly feedback loop. | Agent 3 | Serious | Open — design needed |
| J13 | **In-bond / T&E US hub movement.** `δ_cust_k` destination-side only; US T&E routings (LAX→ORD→EWR pattern) have materially different dwell — model overstates. | Agent 3 | Serious | Open |
| J14 | **Consolidation savings pitch frame.** Pitch-ready: 7-12% reduction on air procurement spend (conservative-to-base), ~20% upside high-mix lanes, $10-28M/year at $200M air revenue. Sources catalogued in `docs/critique/02-consolidation-savings.md`. | Agent 2 (consolidation savings) | High | **Ready to incorporate into pitch deck v7** |
| J15 | **Automation framing for pitch.** Avoid "90%+ automation" — per agent 1, the defensible frame is "compress consolidation planner's afternoon from 4h → 90min, and air-cancellation replan from a workday → a coffee break." Matches 3% shipments / 40% of day Pareto. | Agent 1 | High | **Ready to incorporate into pitch deck v7** |
| J16 | **Three structural clusters across the gap-finder findings:** (a) time-and-state — per-batch loses cross-period dynamics; (b) rule-engine richness — no time-windows / alliance / hard-contract mandates; (c) non-monotone cost terms — monotonicity invariant rules out volume-discount rebates. Use as design lens for v4 of the air model. | Agent 3 synthesis | Med | Open |

---

## Appendix A — The canonical lifecycle (for reference when restructuring `agent_design.md`)

This is the model B2 will install into the agent design doc. Pasted verbatim for use during the rewrite.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Shipment arrives                                                   │
│      ↓                                                              │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │ SOFT PLANNING PHASE                                       │       │
│  │ ─────────────────                                         │       │
│  │ • MILP solve produces candidate plan                      │       │
│  │ • Plan published to operators (visible, reviewable)       │       │
│  │ • Flags generated and surfaced to operator inbox          │       │
│  │ • Re-plan triggers fire continuously (events listed below)│       │
│  │ • Plan and flags update with each re-solve                │       │
│  │ • Operators can: review, override, request alternatives,  │       │
│  │   accept risk, force-commit early                         │       │
│  └──────────────────────────────────────────────────────────┘       │
│      ↓ (at scheduled commit moment OR operator force-commit)        │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │ COMMIT PHASE                                              │       │
│  │ ─────────────                                             │       │
│  │ • Query live rates via carrier APIs (where available)     │       │
│  │ • Rate-surprise check: live vs. planned within tolerance? │       │
│  │ • If clean and no unresolved flags → BOOK                 │       │
│  │ • If rate surprise → enqueue for batch re-solve, no book  │       │
│  │ • If unresolved flag → escalate                           │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

Flags are orthogonal dimensions, not 2×2 cells. A shipment can have 0, 1, or several. Each has its own resolution path:

| Flag | Generated by | Resolution path |
|---|---|---|
| SLA risk | $P(\text{on-time}) <$ tier threshold | Re-plan (try premium route); or operator accepts the risk |
| Cost outlier | $\text{cost}(k) > N \times$ lane median | Human picks up phone; finds alt rate; rate catalog gets updated |
| Rate surprise | Live API rate > X% above planned rate | Batch re-solve; or operator approval to book at surprise rate |
| Capacity risk | Allocation utilization tight; risk of being squeezed out | Re-plan if alternative exists; book early to lock |
| Disruption | Event-driven (port closure, weather, strike) | Re-plan affected shipments |

The commit phase doesn't have an auto-execute cell — every commit goes through the same gate: are flags resolved (or risk-accepted)?

---

## Appendix B — The 8 re-plan triggers (for reference when adding B4)

**Event-driven:**
1. New shipment added to the batch
2. Rate catalog refresh (or live API delta noticed at sample-time)
3. Schedule update (flight slip, cancel, new flight added)
4. Capacity update (allocation consumed elsewhere, or new allocation arrived)
5. Disruption alert (operations feed)
6. Operator override (forces a re-solve)
7. Carrier-policy change (allow/deny list update from rules engine)
8. Rate-surprise batch reaches threshold (X shipments queued for re-solve)

**Time-driven (belt-and-suspenders):**
- Periodic refresh — every N hours, re-solve to incorporate accumulated drift
- Pre-commit refresh — at `cutoff − 2 × safety_margin`, force re-solve before human-review window
- Commit window — at `cutoff − safety_margin`, commit machinery kicks in
