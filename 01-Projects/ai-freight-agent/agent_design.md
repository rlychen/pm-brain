# Agent Design

*Part of the AI Freight Routing PRD. See [PRD.md](PRD.md) for strategic overview and document map.*

**Sections:** AI-Native Design Philosophy · Agent Capabilities · Agent Architecture

---

## 1. AI-Native Design Philosophy

*This section establishes the operating model that governs all design decisions.*

### 1.1 Autonomy Model

This system is not a planning tool with AI assistance. It is an **autonomous planning system with human governance**. The distinction:

- **A planning tool** asks the human to configure and run a plan. The human is the planner.
- **An autonomous planning system** plans continuously and independently. The human sets policy, handles escalations, and audits. The AI is the planner.

| Actor | Primary responsibility |
|---|---|
| Ocean Planner Agent | Batches unrouted shipments, runs optimizer, selects route per configured policy, commits to dry-run state |
| Compliance Validator Agent | Two deterministic functions: (1) at commit time, re-checks optimized routes against current allocation state to catch staleness from concurrent bookings since solve time; (2) on operator override, runs a structured feasibility check (LP relaxation or heuristic) to verify the override does not violate hard constraints before entering dry-run state. Neither function uses LLM inference. |
| Execution Monitor Agent | Watches active shipments; fires re-plan triggers on delay/disruption; auto-reroutes within policy; escalates when re-route is outside policy |
| Market Intelligence Agent | Refreshes spot rate signals, port congestion alerts, capacity signals; flags when market conditions should trigger a policy review |
| **Human operator** | Sets routing policy; handles exceptions the agents cannot resolve; overrides when they disagree; adjusts guardrails when override patterns reveal policy gaps |

**What never happens without human approval (MVP):**
- Writing a booking to a carrier system (deferred; human-initiated for now)
- Committing beyond the dry-run window after a hard-stop guardrail fires
- Any action after a global pause is set

### 1.2 Decision Confidence Tiers

Every routing decision is classified into one of three tiers determining whether the agent auto-executes, recommends, or escalates. Classification uses two independent axes — **risk** and **confidence** — not a single collapsed score.

**Risk** measures the stakes of being wrong. It is rule-based and deterministic: a checklist of conditions evaluated post-solve. **Confidence** measures how certain the system is in its own output. It is a scalar 0–1 heuristic over the signals below.

#### Risk Classification

A decision is **High risk** if any trigger fires; otherwise **Low risk**.

| Trigger | Default | Configurable |
|---|---|---|
| O-D pair has fewer than N prior confirmed routings | N = 10 | Yes |
| Carrier not previously used on this lane | — | No |
| Shipment value above threshold | $15,000 | Yes |
| OTP risk signal: min route slack below threshold (see below) | 3 days | Yes |
| String utilization would exceed threshold after this booking | 85% | Yes |
| Re-routing an in-flight shipment | — | No |
| Operator override of optimizer recommendation | — | No |

#### OTP Risk Signal

After the optimizer selects a route, compute deadline slack at every node using scheduled arc times (deterministic):

- **EA(n)** — earliest arrival time at node n along the chosen arc sequence. EA(origin) = cargo ready time τ_k(origin); EA(n_next) = departure_time(arc) + scheduled_transit_time(arc).
- **Final delivery buffer** — d(k) − EA(destination): buffer at the destination under scheduled, no-disruption conditions.
- **Connection buffer at intermediate node n_i** — departure_time(n_i → n_next) − EA(n_i): window between arriving from the previous leg and the next sailing's departure.

**OTP risk = min(all connection buffers, final delivery buffer).** This is the tightest point in the chain — the node where a disruption most likely cascades into a deadline miss. If OTP risk < 3 days → High risk trigger fires, regardless of P(on_time).

*Note: OTP risk and P(on_time) are complementary signals. OTP risk is deterministic (scheduled times) — it measures structural fragility under nominal conditions. P(on_time) is probabilistic — it measures expected success given transit time variance. A route can have high P(on_time) and still be high OTP risk if the deadline slack is thin: any variance eliminates the buffer.*

#### Confidence Score

A scalar 0–1 heuristic computed post-solve. **Phase 1:** weighted combination of the five signals below, weights in system config. **Phase 4+:** calibrated ML model trained on routing outcomes. The tier structure and risk classification are stable across phases; only the score computation evolves.

| Signal | What it measures |
|---|---|
| P(on_time) | Probability of meeting deadline; output of transit time model |
| Allocation snapshot age | Time since rem(s,t) was last refreshed; stale data lowers confidence |
| Constraint slack | Slack in binding constraints; tighter = less robust to perturbation |
| Cost deviation from lane median | \|solution_cost − median_cost(lane)\| / median_cost; outlier cost lowers confidence |
| Route familiarity | Count of prior confirmed routings on this O-D × service level × carrier combination |

Confidence threshold for Tier 1 auto-execute: configurable in Policy & Guardrails editor (default: 0.80, range: 0.70–0.95). Floor for Tier 3 escalation: fixed at 0.50, not operator-configurable.

#### Tier Assignment

```
                    High confidence      Low confidence
                    (score ≥ 0.80)       (score < 0.80)
                  ┌──────────────────┬──────────────────┐
    Low risk      │   Tier 1         │   Tier 2         │
                  │   Auto-execute   │   Recommend      │
                  ├──────────────────┼──────────────────┤
    High risk     │   Tier 2         │   Tier 3         │
                  │   Recommend      │   Escalate       │
                  └──────────────────┴──────────────────┘
```

Additionally, regardless of the matrix:
- Any Layer 1 guardrail triggered → Tier 3. Non-negotiable.
- Confidence score < 0.50 → Tier 3. Non-configurable.

#### Tier Behaviors

| Tier | Agent behavior |
|---|---|
| **Tier 1 — Auto-execute** | Commit to dry-run state. Log decision, confidence score, risk classification, OTP risk signal, and tier. Auto-commits after dry-run window (default 60 min) unless operator overrides. |
| **Tier 2 — Recommend** | Present top-3 options (cheapest / fastest / most reliable) with confidence score, risk flags, OTP risk signal, and rationale. Human selects or approves. |
| **Tier 3 — Escalate** | Add to exception queue with full context: binding constraints, OTP risk signal, alternatives with trade-offs. Require explicit human decision before any commit. |

#### Interaction with Deployment Modes

The matrix applies in **Supervised** mode. Deployment mode (§1.4) shifts behavior:

| Mode | Effect |
|---|---|
| **Co-pilot** | All decisions → Tier 2 behavior. Human approves everything. Matrix bypassed. |
| **Supervised** | Matrix applies as defined. |
| **Autonomous** | Tier 2 cells auto-commit after dry-run window with soft notification. Tier 3 always requires human. |

**Every decision is logged** with its confidence score, risk classification (which triggers fired), tier assignment, OTP risk signal, and outcome. This log is the primary input for trust graduation decisions (§1.4) and confidence model calibration (Phase 4+).

---

### 1.3 Guardrails Framework

Four layers. Inner layers are non-configurable; outer layers are operator-tunable. The confidence tier model (§1.2) operates on top of this framework — a decision can be low-risk and still route to Tier 2 if the agent's confidence is below threshold.

#### Layer 1 — Hard Stops (non-configurable; agent cannot override)

| Guardrail | Behavior |
|---|---|
| Allocation cap: `required_slots > rem(s,t)` | Block route; escalate to exception queue |
| Deadline miss: no feasible path within deadline | Block; escalate with earliest achievable date |
| Blacklisted carrier on route | Block; exclude from all options |
| Hazmat / OOG / reefer cargo (MVP) | Reject at pre-filter; escalate with structured report |
| Validator returns FAIL after 3 revision cycles | Auto-escalate; do not commit |
| Agent confidence score < 0.50 | Always escalate; do not commit at any autonomy level |

#### Layer 2 — Threshold Guardrails (configurable by operator)

| Guardrail | Default | Configurable |
|---|---|---|
| CYC margin minimum | 0.5 days | Yes |
| Single shipment cost ceiling | $10,000 | Yes |
| Dry-run window before auto-commit | 60 min | Yes (per priority tier) |
| Maximum cost deviation from lane median | +30% | Yes |
| Minimum P(on-time) per service level | Economy: 80%, Standard: 90%, Express: 95% | Yes |
| Confidence threshold for Tier 1 auto-execute | 0.80 | Yes (range: 0.70–0.95) |

#### Layer 3 — Soft Guardrails (agent proceeds; notifies)

The agent routes and commits, but logs a notification. Operator sees it in the activity feed.

- Route deviates from routing guide (explain which rule and why)
- Carrier is acceptable (not preferred) on this lane
- Transit time is in top 20% longest observed on this lane
- Cost is in top 20% most expensive observed on this lane
- Allocation utilization for a string will exceed 70% after this batch

#### Layer 4 — Structural Guardrails (architectural)

These are design properties that prevent entire classes of failure.

- **Agent isolation.** Routing Planner and Compliance Validator are separate nodes with no shared context. The validator receives only the optimizer's structured output — not the planner's chain of thought or intermediate reasoning. Enforces a clean commit gate auditable independent of the planner, and prevents any planner reasoning from bypassing the allocation check.
- **Decision logging before commit.** Every routing decision is written to the audit log *before* it enters dry-run state. The log cannot be modified post-hoc.
- **Idempotent optimizer calls.** Running the optimizer twice on the same input must produce the same result.
- **rem(s,t) is read-only during a batch.** Allocation state is snapshotted at the start of each routing run and held fixed throughout. Concurrent runs on the same string are serialized.
- **Kill switch.** A single operator action pauses all autonomous routing globally, per lane, or per agent. Kill switch does not require the AI's consent.
- **Override logging for constraint learning.** Every operator override is written to `logs/overrides.jsonl` with the original agent decision, the override, and the operator's reason.

### 1.4 Progressive Trust Expansion (Deployment Modes)

Customers onboard at the autonomy level appropriate to their trust in the system and progressively expand as the agent establishes a track record. Three modes are available; a customer can run different modes on different trade lanes simultaneously.

| Mode | How it works | Typical stage |
|---|---|---|
| **Co-pilot** | Agent prepares a recommendation for every decision. Human reviews and approves before any action is committed. Dry-run state is not entered until explicit approval. | New customer onboarding; high-stakes or novel lanes |
| **Supervised** | Agent handles Tier 1 (routine) decisions autonomously via the standard dry-run commit flow. Tier 2 and Tier 3 decisions surface to the exception queue for human review. | Standard operating mode once agent performance is established |
| **Autonomous** | Agent routes, validates, and commits all decisions. Humans monitor the exception queue (Tier 3 only) and audit the routing activity log. Tier 2 decisions auto-commit after the dry-run window with soft notification. | Established deployments with documented performance history |

**Trust graduation criteria (moving to a more autonomous mode):**

A lane or customer account qualifies for a more autonomous mode when all three conditions hold over the preceding 30 routing days:
1. Tier 1 decision accuracy ≥ 95% (measured by zero operator override on auto-committed decisions)
2. Agent confidence score mean ≥ 0.82 on that lane
3. No Layer 1 guardrail violation in the window

The system surfaces a trust graduation recommendation in the Policy & Guardrails editor; the operator must explicitly approve the mode change. Mode changes are logged to the audit trail with timestamp and authorizing operator.

**Trust degradation:** If operator override rate on Tier 1 decisions exceeds 15% in any 7-day window, the system automatically downgrades to the previous mode and flags a policy review.

---

### 1.5 Routing Triggers

The routing agent runs automatically on any of these conditions (configurable):

| Trigger | Default | Description |
|---|---|---|
| Scheduled batch | Daily 06:00 local | Route all unrouted shipments with cargo-ready date ≤ T+7 days |
| Accumulation | ≥ 10 unrouted | Run when 10+ unrouted shipments pile up on the same trade lane |
| Urgency | CYC within 48h | Immediately route any unrouted shipment approaching its vessel cutoff window |
| Manual | Operator action | Force a routing run at any time from the dashboard |
| Re-plan | Disruption event | Execution Monitor fires; re-plan affected in-flight shipments |

Triggers can be paused per lane. Urgency triggers cannot be paused (safety feature).

---

## 2. Agent Capabilities

### 2.1 Core Routing and Planning

- Route a single shipment: return all viable options given constraints
- Lowest-cost routing with feasibility check against delivery window
- Fastest routing (minimize expected transit time)
- Reliability-optimized routing (maximize on-time probability using transit time distributions)
- Multi-objective: return Pareto frontier of cost vs. time vs. reliability
- Mode selection: ocean vs. trucking vs. combined multimodal
- Carrier selection within mode against routing guide and preference rules
- FCL vs. LCL decision (consolidation economics)
- Direct vs. transshipment routing
- Cargo-ready-to-vessel-cutoff feasibility check
- Rolling horizon re-plan trigger evaluation and execution

### 2.2 Constraint and Rule Handling

- Hard time windows: pickup window, latest-arrival delivery window
- Soft time windows: prefer within range, penalize violations in objective
- Service level tiers: Economy, Standard, Express
- Carrier preference / blacklist / allocation cap enforcement
- Port or lane avoidance (e.g., Red Sea, Panama congestion)
- Weight and volume constraints per leg and per vessel service
- Commodity restrictions: hazmat class, temperature-controlled, OOG
- Trade lane regulatory constraints
- Budget cap per shipment or per lane
- Dangerous goods and temperature segregation (cannot co-load)

### 2.3 Batch and Fleet Operations (Autonomous)

- Route all unbooked shipments in a portfolio simultaneously
- Demand-supply graph decomposition: partition the batch into independent subproblems (shipments sharing no feasible supply arcs) before solving; each partition dispatched to the optimizer separately and results merged
- Priority segmentation: "route these N urgently, rest economical"
- Volume consolidation: identify which shipments can merge into one container
- Carrier allocation compliance: flag shipments violating contracted allocation caps
- Exception queue: surface shipments requiring human decision, ranked by urgency and impact
- Bulk re-routing: identify all active shipments affected by a specific disruption
- Portfolio status: how many shipments are at-risk vs. on-track right now?

### 2.4 Scenario Analysis and What-If

- What if I shift origin port?
- What if I accept N days more transit time — how much do I save?
- What if I upgrade from LCL to FCL?
- What if carrier X is unavailable on this lane?
- What if I split this shipment across two vessels?
- Red Sea avoidance: route via Cape of Good Hope, show full cost/time delta
- Air vs. ocean: full cost and time comparison for same shipment
- Service level upgrade cost: what does 5 days faster cost?
- Tariff/duty change impact: how does a new duty rate affect landed cost by route?
- Capacity constraint scenario: model what happens if a major port goes down

### 2.5 Disruption and Exception Management

- Detect predicted delay: weather, port congestion, vessel rollover, anchorage wait
- Alert with ranked recommended actions (rebook, reroute, notify customer)
- Autonomous rerouting recommendation on carrier failure or missed cutoff
- Port strike / closure contingency routing
- Vessel schedule change: recalculate all impacted shipments and options
- Customs hold detection and recommended next steps
- Missed pickup window recovery options ranked by cost and time impact
- Proactive risk scoring: which booked shipments are most at risk this week?
- Rolling horizon re-plan on disruption: re-solve fine graph with updated constraints

### 2.6 Tracking and Visibility

- Where is this shipment right now? (AIS position on ocean legs)
- Current ETA prediction (ML-based, not just carrier schedule)
- Full milestone trace: cargo ready → picked up → departed → transshipment → arrived → customs cleared → delivered
- Is this shipment on track or at risk? (vs. committed delivery window)
- How many days remain to delivery?
- Remaining legs with mode transitions
- All shipments at risk: portfolio-level exception view
- Filter active shipments by mode, carrier, lane, risk status

### 2.7 Analytics and Performance

- Cost breakdown by lane, carrier, mode, time period
- Transit time performance vs. committed SLA by carrier and lane
- On-time delivery rate by carrier, lane, mode
- Carrier scorecard: cost, reliability, rollover rate, on-time delivery
- Lane performance trends: is this lane getting more expensive or slower?
- Route explanation / audit trail: why was this specific route chosen?
- Savings attribution: how much did optimization save vs. default routing?
- Counterfactual / regret analysis: what would this shipment have cost if we had chosen differently?
- Carrier volume commitment utilization: am I meeting minimum contracted volumes?
- Emissions estimate: CO₂ per route option

### 2.8 Advisory and Decision Support

- Is this quote from my forwarder reasonable vs. market?
- What are the most reliable carriers for this lane?
- Am I using my carrier allocations efficiently?
- What is my exposure if [port / lane / carrier] goes down?
- Should I pre-book capacity given current demand signals?
- What is the market benchmark rate for this lane?

---

## 3. Agent Architecture

### 3.1 Framework: LangGraph

**Decision: LangGraph, not direct Anthropic SDK.**

| Criterion | LangGraph | Direct Anthropic SDK |
|---|---|---|
| Behavioral control | High — explicit graph state, conditional edges | High — but you build everything |
| Decision logging | LangSmith — best-in-class, zero-build | Build it yourself |
| MCP integration | `langchain-mcp-adapters` — production-tested | Native Anthropic SDK |
| Model swappability | Yes — model-agnostic | No — Claude-only |
| Planner-validator pattern | Native supervisor with conditional edges | Custom build |
| Human-in-the-loop | First-class `interrupt()` + PostgreSQL checkpointer | Build it yourself |
| Debuggability | Time-travel debugging in LangSmith | Build it yourself |
| Production maturity | Tier 1 — thousands of production deployments | Tier 3 — newer |

**Why this matters for this system:**
- Full logging of every agent decision (CLAUDE.md requirement) — LangSmith provides this with zero custom code
- 6+ agent personas with branching control flow — managing this in a hand-rolled loop is fragile
- Model-agnosticism — different agents can run on different models without architectural changes
- The planner-validator supervisor pattern is a native LangGraph construct

### 3.2 Architecture Pattern: Hierarchical with Hub-and-Spoke Leaves

```
                    ┌─────────────────────┐
                    │  Top-Level Router   │
                    │   (Orchestrator)    │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
  ┌───────────────────────┐       ┌─────────────────────────┐
  │  Planning Supervisor  │       │  Operations Supervisor  │
  └───────────┬───────────┘       └────────────┬────────────┘
              │                                │
       ┌──────┴──────┐                ┌────────┴────────┐
       ▼             ▼                ▼                 ▼
  ┌─────────┐  ┌──────────┐   ┌───────────┐   ┌────────────────┐
  │Routing  │  │Compliance│   │Execution  │   │Market          │
  │Planner  │→ │Validator │   │Monitor    │   │Intelligence    │
  └─────────┘  └──────────┘   │(event-    │   │(scheduled +    │
                               │driven)    │   │on-demand)      │
                               └───────────┘   └────────────────┘
```

The Planning Supervisor runs the planner → validator loop for every routing request. The Operations Supervisor manages two background agents running continuously. The Top-Level Router receives incoming requests and dispatches to the appropriate supervisor.

### 3.3 Agent Personas

#### Routing Planner Agent
**Responsibility:** Given a shipment request or batch, call the optimization and ML tools to generate end-to-end route recommendations with supporting rationale.

**Tools available:** Graph generator, ocean optimizer, trucking optimizer, transit time models, rolling horizon controller, rules engine, AIS adapter, road routing adapter, rate lookup

**Output:** Structured route recommendation: `{route: [legs], total_cost, expected_transit_days, p_on_time, constraints_checked: [list], rationale: str}`

**Does not have access to:** Carrier booking APIs (writes), external notification systems

#### Compliance/Validation Agent

**Two deterministic functions. No LLM inference in either path.**

**Function 1 — Pre-commit allocation check (every routing batch)**

After the optimizer produces a solution and before it enters dry-run state, re-read current `rem(s,t)` for every string used in the solution. If any string's remaining allocation has been consumed by concurrent activity (manual bookings, parallel batches) since solve time, block the affected routes and return them to the planner for re-solve on the updated snapshot.

This is optimistic concurrency control: solve optimistically against a snapshot, validate against current state at commit. The Layer 4 guardrail (§1.3) serializes concurrent batch solves on the same string, but does not cover changes from manual bookings or operator activity that occur between solve and commit.

**Output:** `{status: PASS|FAIL, stale_strings: [{string_id, required_slots, current_rem}]}`

- `PASS` → commit to dry-run state
- `FAIL` → return affected routes to planner; re-snapshot allocation; re-solve

**Function 2 — Post-override feasibility check (every operator override)**

When an operator overrides a committed route, the optimizer does not re-run. A deterministic feasibility check verifies the operator's proposed route against hard constraints:
- Allocation cap: `required_slots ≤ rem(s,t)` for all strings on the proposed route
- Deadline: earliest achievable arrival on the proposed route ≤ delivery deadline
- Vessel/container capacity: cargo fits on each leg

This is an LP relaxation or structured heuristic — the same logic as the graph pre-filter, applied to the operator's specific proposed route. If the override fails, the operator is shown the specific constraint violated and must correct or escalate.

**Output:** `{status: PASS|FAIL, violations: [{constraint: str, value: float, bound: float}]}`

**What is not this agent's job** (handled at graph generation, before the optimizer sees the problem):
- Carrier restriction enforcement → arcs removed at subgraph construction
- Routing guide hard rules → arcs removed at subgraph construction per customer config
- Sanctions and embargoes → nodes/arcs pre-filtered before solve
- Commodity restrictions → commodity-specific subgraphs enforce at construction

**Isolation note:** The agent node receives only the optimizer's structured output — not the planner's chain of thought or intermediate reasoning. This is preserved even though the functions are deterministic: it enforces clean interface boundaries and makes the commit gate auditable independent of the planner.

#### Execution Monitor Agent
**Responsibility:** Continuously watches active shipments against their planned itineraries. Detects exceptions (delay, rollover, customs hold, missed pickup), fires rolling horizon re-plan triggers, and generates proactive alerts with recommended actions.

**Operates:** Event-driven (subscribes to AIS updates, EDI 214 events, carrier APIs) + polling for shipments approaching re-plan trigger thresholds. Runs asynchronously — not in the request/response path.

**Tools available:** AIS adapter, shipment state store, rolling horizon controller, routing planner (can request a re-plan), alert dispatch

**Output:** Exception alerts with ranked recommended actions; re-plan triggers with updated state context.

#### Market Intelligence Agent
**Responsibility:** Maintains a live view of market conditions — spot rate indices, carrier capacity signals, port congestion alerts, weather disruptions. Provides context to the Routing Planner on demand and pushes alerts when conditions change materially.

**Operates:** Scheduled refresh (configurable interval) + on-demand query from Routing Planner.

**Tools available:** Rate index feeds, port congestion APIs, weather/disruption feeds, news monitoring

**Output:** `{market_context: {lane: {spot_rate_range, capacity_signal, disruption_alerts}}}` — queryable by the Routing Planner as a tool call.

#### Future Personas (Deferred)
- **Customer Communication Agent** — translates routing decisions and exceptions into shipper-facing language, manages notification drafts. No write access to booking systems.
- **Scenario / What-If Agent** — stress-tests approved routes against parameterized scenarios. Used in planning mode, not real-time.

### 3.4 Human-in-the-Loop Checkpoints

LangGraph `interrupt()` fires at the following points, pausing execution for human review:

1. **Validator returns ESCALATE** — recommendation has an unresolvable conflict
2. **Planner-validator loop reaches 3 revision cycles without PASS** — auto-escalate
3. **Any booking action** (future, when autonomous execution is enabled) — all writes to carrier systems require explicit human approval until the system is validated at scale
4. **Exception requires rerouting a high-value or time-critical shipment** — configurable threshold (e.g., shipments with <24h delivery window remaining)

State is persisted via PostgreSQL checkpointer — human can resume the workflow after reviewing without losing context.

### 3.5 Logging

All agent interactions are logged via LangSmith. Every tool call, every state transition, every agent input and output is captured with timestamp.

Additionally, the agent interaction log (`logs/agent_interactions.jsonl`) captures the user-facing query/response pairs for capability extension analysis — separate from the internal LangSmith trace.

### 3.6 Production Failure Mode Mitigations

| Failure Mode | Mitigation |
|---|---|
| Infinite planner-validator loop | Max 3 revision cycles, then auto-ESCALATE |
| Sycophantic validator | Independent tool access, explicit skeptic framing, PASS rate monitoring |
| Context loss on agent handoff | Typed Pydantic schemas at every boundary; pass only what downstream needs |
| Concurrent booking conflicts | Optimistic locking on carrier capacity state; serialize writes through a single stateful node |
| Latency cascade | Execution Monitor and Market Intelligence run async — never in the request path |
| Model version drift | Version-pin model + system prompt pairs; test planner-validator pair as a unit before deployment |
| Validation theater | Monitor PASS rate in production; >90% PASS rate without findings triggers review |
| Intent-to-capability mismatch | Structured refusal — classifier returns intent category and list of registered capabilities; agent never approximates via freeform reasoning or generates code to fill the gap |

---

### 3.7 Capability Registry and Bounded Dispatch

The agent's action space is explicitly bounded. The Top-Level Router does not do open-ended reasoning to decide what to do with an incoming request — it classifies intent against a finite capability registry and either dispatches to a defined tool sequence or returns a structured refusal. This is the primary defense against the agent inferring capabilities it does not have.

#### Why this matters

Without an explicit registry, an LLM-based router can:
- Hallucinate tool parameters not present in the MCP schema
- Answer from parametric knowledge instead of tool output — confident, plausible, wrong
- Chain tools in unintended sequences to approximate an unsupported operation
- Generate code or ad-hoc reasoning to fill a capability gap — the most dangerous failure mode

The registry makes the capability boundary explicit and machine-enforceable, not a property of prompt quality.

#### Dispatch flow

```
NL input
   │
   ▼
Intent classifier (LLM)
   │
   ├─ Match found ──► Parameter extractor ──► Schema validation ──► Tool call(s) ──► Structured output
   │
   └─ No match ─────► Structured refusal: {intent_understood, reason_not_mapped, available_capabilities: [list]}
```

The classifier's job is narrow: map NL to one of N registered intent categories. It does not reason about what to do — it only classifies. Parameter extraction is a separate step with explicit field mapping. If parameter extraction fails (missing required field, type mismatch), the agent asks for the missing input rather than inferring it.

#### Registered capability structure

Each entry in the registry specifies:
- **Intent pattern** — description of what this capability handles (used by classifier)
- **Required tool(s)** — ordered list of MCP tool calls
- **Parameter mapping** — how NL fields map to tool input schema fields
- **Output schema** — what the agent returns; validated before returning to caller

#### Initial capability registry (Phase 4 baseline)

| Intent category | Tool sequence |
|---|---|
| Route a shipment batch | `graph_generator` → `ocean_optimizer` → `trucking_optimizer` |
| Re-plan an active shipment | `rolling_horizon_controller` → `ocean_optimizer` |
| Check string allocation | `allocation_snapshot_service` |
| Get market rate context for a lane | `market_intelligence_agent` → `rate_lookup` |
| Explain a routing decision | `routing_run_log` → `langsmith_trace_retrieval` |
| Report on batch routing results | `routing_run_log` |

New capabilities are added by: (1) defining the intent pattern, (2) specifying the tool sequence, (3) writing the parameter mapping, (4) adding to the registry. The agent cannot acquire new capabilities at runtime.

#### Explicit capability boundary

The following are out of scope for the agent at any confidence level. A request touching these always returns a structured refusal:
- Shipper-facing pricing or quoting (pricing engine not yet built)
- Direct carrier booking (autonomous execution deferred to a future phase)
- Any answer not grounded in tool output — including routing advice based on the LLM's parametric knowledge of carriers, rates, or schedules
- Code generation or ad-hoc computation to answer a question outside the registry

#### Preventing freeform inference

- The agent has no access to a code execution tool or shell
- Every agent response is validated against a defined output schema; freeform text is not accepted where a structured tool result is expected
- The classifier is evaluated periodically against a held-out set of in-scope and out-of-scope queries; classification accuracy and false-positive refusal rate are tracked
