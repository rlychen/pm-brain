# Messaging-Agent Layer-3 Capabilities — Design Exploration

**Date:** 2026-05-27
**Status:** Design exploration. Input to PRD §Layer-3 and Phase-5/6 sequencing.
**Scope:** Layer 3 of the messaging-channel agent (`project_unstructured_channel_wedge.md`) — active participation. Layers 1 (passive listening) and 2 (inbound intake) scoped elsewhere.

Expands the founder's 10-capability sketch into a comprehensive enumeration, maps each capability onto existing components, recommends MVP, enumerates Layer-3 failure modes, and shows where Layer-3 fits in the five-layer stack (`architecture.md §11`).

Conventions: "planner" = consolidation planner / ops handler (collapsed at mid-size per `project_core_user_reality.md`). "LSP" = partner / co-loader / carrier rep in-channel. "KAM" = key account manager. Trust rungs: read-only / propose-only / execute-with-confirm / autonomous-on-lane (`architecture.md §10`).

---

## 1. Capability enumeration (26 capabilities, 7 groups)

For each capability: **Trigger** | **Action** | **HITL** (proposer → confirmer(s) → applier) | **Data touched** | **Trust rung** | **Failure mode**.

### Group A — Correction loops (catalog updates from in-channel data)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| A1 | BSA / allotment correction | LSP message matches BSA-update pattern ("TPE-LAX BSA cut to 8 pallets this week") | Extract (lane, carrier, period, value); propose-card to LSP in-channel + planner console | agent → LSP confirm-in-channel + planner confirm → orchestrator applies | Capacity manager (BSA accumulator), possibly rate catalog | propose-only (MVP) | Silent wrong-allotment write cascades into bad MILP solves; over-booking exceeds real cap; carrier penalty |
| A2 | Cargo-readiness slip capture | Shipper / origin partner: "cargo ready Thursday not Tuesday" | Extract (HAWB-id, new $t_k^{rdy}$); confirm with shipper; orchestrator manual delta-solve | agent → shipper in-channel → planner one-click | HAWB pool (`air_freight_routing.tex §sec:hawb-params`) | execute-with-confirm | Wrong slip applied → planner replans around non-existent slip; missed → fallback-arc rescue |
| A3 | Rate-snapshot validation | In-channel rate quote differs > N% from snapshot; or freshness TTL expired | Flag gap, ask LSP "is this the current rate?", propose update | agent → LSP confirm → planner apply | Rate catalog (snapshot tier per `architecture.md §8`) | propose-only | Bad write → wrong carrier next solve → cost-outlier flag spike, planner trust erosion |
| A4 | Carrier-policy update capture | "We no longer accept lithium on CX880," "EVA refusing DGR to TLV" | Classify as policy vs. one-off; draft rule in rule-author UI pending queue | agent → planner (and compliance) → rule-author UI | Rules engine (carrier-policy cascade, `air_freight_routing.tex §sec:carrier-policy`) | propose-only | False positive narrows eligible carriers wrongly → fallback spike; false negative → tendered to refused lane |
| A5 | Temporary embargo capture | "No DG to TLV next 2 weeks" | Extract (commodity, dest, window); draft temporary embargo with expiry | agent → planner + compliance → rule-author UI | Rules engine, embargo store | propose-only | Missed embargo → tendered to refused lane → emergency re-route, regulatory exposure |
| A6 | Schedule update (flight roll, blank sailing, vessel slip) | "CX123 rolled to 25-May," "blank sailing FE3 W21" | Extract event; match to affected MAWBs/HAWBs; propose schedule update; orchestrator manual delta-solve | agent → planner one-click | Schedule store, plan supersedence chain | execute-with-confirm | Mistakes self-heal on next carrier-feed sync; missed roll → stale plan past commit |
| A7 | Equipment / ULD availability update | "No LD3 pool at TPE this week" | Extract (ULD-type, station, period); propose constraint update | agent → planner / LSP confirm | Supply catalog, ULD specs (`air_freight_routing.tex §sec:uld-specs`) | propose-only | Bad availability → MILP picks unbuildable ULD config → rejection at build-up |

### Group B — Diagnosis loops (explain why)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| B1 | Fallback-arc diagnosis | Any HAWB in $K^{fb}$ per `air_freight_routing.tex §sec:output-diagnostics` | Walk pre-filter trace; identify excluding predicate(s) per real arc; plain-language explanation in console flag + optional draft message to LSP for spot-capacity ask | agent posts diagnosis; planner triages; planner authorizes outreach | Graph generator pre-filter trace, rules engine, MILP diagnostic output | read-only on diagnosis; propose-only on outbound | Wrong root cause → planner chases wrong fix; trust erosion on bad explanations |
| B2 | Cost-outlier diagnosis | Cost-outlier flag fires (cost > N × lane median, `architecture.md §4`) | Compare cost decomposition to lane median; identify dominant delta | read-only | MILP cost decomp, lane analytics, rate catalog | read-only | Wrong decomp misleads "fix"; low operational impact |
| B3 | SLA-risk diagnosis | SLA-risk flag fires (P(on-time) < tier) | Identify binding leg (transit-time tail, cutoff buffer, MCT) | read-only | Transit-time model, MILP route trace | read-only | Same as B2 |
| B4 | "Why this routing?" Q&A | Shipper / KAM asks in-channel or in console | Retrieve binding-constraint set from MILP output; decompose into 2–3 dominant reasons | read-only → propose-only outbound for shipper-facing replies | MILP output, rate catalog, rules engine, customer routing guide | read-only | Wrong attribution → bad shipper explanation → escalation |
| B5 | Disruption blast-radius scan | Disruption ingested from listener / external feed | Run existing `disruption_impact_scan` (P0 tool, `personas_and_tools.md`); rank by severity | read-only | Shipment state, graph dependency lookup | read-only | False-positive classification (non-event flagged as event) |

### Group C — Procurement loops (active outreach)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| C1 | Spot-capacity outreach for fallback HAWBs | B1 identifies bottleneck; planner authorizes outreach | Message named LSPs / carriers asking spot capacity on bottleneck arc; parse responses; surface offers to planner | planner approves outbound; LSP responds; planner picks; orchestrator re-solves with spot arc injected | Supply catalog (spot), MILP graph (new arc) | propose-only on outbound; execute-with-confirm on capacity injection | Paraphrase error converts LSP "maybe" to "yes" → planner books phantom capacity |
| C2 | Pre-emptive capacity discovery | Fallback-arc usage pattern persists (≥ N HAWBs/week same arc) | Open conversation with allotment manager about next-period capacity | planner authorizes; KAM owns commercial discussion | BSA accumulator, override log, contract store | propose-only on outbound | Low downside (sourcing nudge) |
| C3 | Co-loader negotiation | MILP shows co-load competitive but no named rate in catalog | Ask named co-loaders for in-channel rates on lane / week; summarize | planner confirms outbound; co-loader replies; planner picks | Rate catalog (co-load tier), MILP graph | propose-only | Same as C1 — paraphrase commits phantom rate |
| C4 | Rate-snapshot refresh (proactive) | Freshness TTL approaching on active lane; no in-channel quote in window | Ping carrier / LSP for current rate | planner confirms outbound; planner confirms apply | Rate catalog | propose-only | LSP nuisance if too frequent — rate-limit by lane + carrier |

### Group D — Communication loops (outbound to shipper / KAM / partner)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| D1 | Plan-change (re-route) notification | Re-solve produces materially different plan than previous (carrier change, ETA shift > tier threshold) | Draft customer-facing notification in customer's channel; surface to KAM/CSR; send on approve | CSR/KAM approves before send (MVP); autonomous-on-lane later (per `architecture.md §10`) | Shipment state, plan supersedence chain | propose-only | Premature notification before commit; customer fatigue if over-frequent |
| D2 | ETA-update notification (no plan change) | Schedule update or transit-time refresh shifts ETA materially without routing change | Same as D1 | Same as D1 | Schedule store, plan supersedence | propose-only | Same as D1, lower stakes |
| D3 | Proactive cutoff reminder | `cutoff_alert` fires (P0, `personas_and_tools.md`) — cargo not at CFS / not screened / docs missing | Message shipper: "cargo not yet at CFS; cutoff in 6h" | propose-only initially; templated autonomous later | Shipment state, cutoff store | propose-only → autonomous (low-risk template) | False trigger embarrasses shipper — rate-limit |
| D4 | Handover briefing (shift / region) | Time-of-day shift boundary (per `day-in-the-life/04 §4.3`) | Summarize day's exceptions, open replans, escalations for next desk | outgoing planner reviews | Override log, plan supersedence, in-channel exception log | propose-only | Omission → next desk misses something; mid-size mostly lacks formal handovers anyway |

### Group E — Learning loops (override log → behavior adjustment)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| E1 | Override-pattern flagging | ≥ N overrides with same reason code on same (lane, carrier, customer) in window | Surface pattern in console with proposed rule / rate update | planner reviews; rule-author UI consumes | Override log, rules engine, rate catalog | propose-only | Per `architecture.md §7.2`, founder rejected as LLM task — implement as SQL job, not Layer-3 |
| E2 | Trust-degradation alarm | Override rate > 5–8% threshold (`project_override_rate_kpi.md`) on lane/customer cohort | Demote lane down trust ladder (autonomous → execute-with-confirm); notify admin | automated demotion; admin re-promotes manually | Override log, trust-ladder state | meta-trust (auto-demote, human-promote) | False demotion erodes productivity; false non-demotion keeps broken lane autonomous |

### Group F — Identity / consent / lifecycle (operational substrate)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| F1 | Channel-join consent flow | Planner / KAM wants to add agent to WhatsApp / LINE group | Post consent message (tenant, scope, data-handling URL); wait for explicit admin ack | external — carrier/LSP/shipper must consent (WhatsApp Jan 2026 policy per `docs/critique/05`) | Channel registry | read-only post-consent | Joining without consent → policy violation → channel ban; hard-required |
| F2 | Identity verification of message sender | Every inbound implicit; explicit for high-impact actions | Maintain (channel_id, handle) → (named_party, tenant_role) registry; gate writes by role | agent blocks unauthorized writes; planner verifies | Tenant identity registry | substrate | Under-verify → spoofed write; over-verify → friction with new legitimate participants |
| F3 | Channel-leave / lifecycle close | Shipment cycle complete, explicit close, or inactive past TTL | Post sign-off; stop reading; archive log per retention | automated on TTL; planner override | Channel registry, conversation log retention | substrate | Failure to close → data collection past consent → compliance exposure |

### Group G — Inbound boundary (intake disambiguation)

| # | Capability | Trigger | Action | HITL | Data touched | Trust rung | Failure mode |
|---|---|---|---|---|---|---|---|
| G1 | Intent classification | Any inbound from shipper-channel | Classify: new-shipment (→ Layer 2), follow-up-on-existing, general-question, social-noise | low-confidence routed to planner triage | HAWB pool, conversation memory | substrate | Misclassified request dropped on floor; noise enters HAWB queue |
| G2 | Missing-data follow-up | Inbound shipment request missing required field (commodity, weight, DGR, ready window) | Ask specific missing question; assemble HAWB once complete | planner inspects only on persistent failure (≥ 3 exchanges) | HAWB pool, intake schema | execute-with-confirm | Over-questioning annoys shipper; under-questioning → malformed HAWB |
| G3 | Quote-back in-thread | HAWB complete from G1+G2 | Orchestrator runs MILP; agent returns price + SLA + 2–3 alternatives | per-tenant policy: autonomous for known shippers; planner-approved for new | Full system | per-tenant, per-customer | Bad quote → trust hit; rate-limit + override-rate gate; this is Layer-2 main job |

**Total: 26 capabilities.** Expansion from the founder's 10 mostly comes from splitting "diagnosis" into specific flag types, adding identity / consent substrate (F group), and adding learning loops (E group).

---

## 2. Component mapping

| Capability | MILP | Rate catalog | BSA / capacity | Rules engine | Schedule | Plan chain | Override log | Orchestrator | Console | Rule-author UI | Transit-time | New components needed |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| A1 | re-solve | — | **W** | — | — | propose | — | trigger | display | — | — | extraction pipeline; propose-card |
| A2 | re-solve | — | — | — | — | propose | — | trigger | display | — | — | extraction |
| A3 | re-solve | **W** | — | — | — | propose | — | trigger | display | — | — | rate-delta classifier |
| A4 | re-solve | — | — | **W** | — | propose | — | trigger | display | **W** | — | policy classifier |
| A5 | re-solve | — | — | **W** | — | propose | — | trigger | display | **W** | — | embargo classifier + expiry |
| A6 | re-solve | — | — | — | **W** | propose | — | trigger | display | — | — | schedule classifier |
| A7 | re-solve | — | **W** | — | — | propose | — | trigger | display | — | — | extraction |
| B1 | **R** (output + pre-filter trace) | R | R | R | R | — | — | — | display | — | — | trace walker + LLM explainer |
| B2 | **R** (cost decomp) | R | — | — | — | — | — | — | display | — | — | cost-decomp formatter |
| B3 | **R** | — | — | — | — | — | — | — | display | — | **R** | explanation formatter |
| B4 | **R** | R | R | R | — | — | — | — | display | — | R | LLM-RAG explainer |
| B5 | — | — | — | — | — | — | — | — | display | — | — | uses existing `disruption_impact_scan` |
| C1 | re-solve | — | **W** (new arc) | — | — | propose | — | trigger | display | — | — | outbound drafter + response parser |
| C2 | — | — | — | — | — | — | **R** | — | display | — | — | pattern detector (SQL preferable) |
| C3 | re-solve | **W** | — | — | — | propose | — | trigger | display | — | — | outbound drafter + parser |
| C4 | — | **W** | — | — | — | — | — | — | display | — | — | TTL scheduler + drafter |
| D1 | — | — | — | — | — | R | — | — | display | — | — | comms drafter (existing `architecture.md §7.2`) |
| D2 | — | — | — | — | R | R | — | — | display | — | R | comms drafter |
| D3 | — | — | — | — | — | — | — | — | display | — | — | uses `cutoff_alert` |
| D4 | — | — | — | — | — | R | R | — | display | — | — | summary generator |
| E1 | — | — | — | propose | — | — | **R** | — | display | propose | — | pattern detector (SQL) |
| E2 | — | — | — | — | — | — | **R** | trigger | display | — | — | trust-ladder state machine |
| F1 | — | — | — | — | — | — | — | — | display | — | — | **NEW: channel registry + consent ledger** |
| F2 | — | — | — | — | — | — | — | gate | — | — | — | **NEW: identity registry + authz gate** |
| F3 | — | — | — | — | — | — | — | — | display | — | — | **NEW: TTL closer + retention policy** |
| G1 | — | — | — | — | — | — | — | — | — | — | — | **NEW: intake classifier** |
| G2 | — | — | — | — | — | — | — | — | — | — | — | **NEW: intake state machine** |
| G3 | **R** | R | R | R | R | propose | — | trigger | display | — | R | Layer-2 scope |

**New components Layer-3 introduces** (not in existing architecture):

1. **Channel registry + consent ledger** (F1, F3) — per-tenant (channel_id, platform, members, consent timestamp, doc URL, lifecycle state).
2. **Identity registry + authorization gate** (F2) — in-channel identity → role; gates writes.
3. **In-channel extraction pipeline** (A1–A7, C-response parsing) — domain LLM extractor with Pydantic-validated output. Distinct from email parsing: shorter, more telegraphic, more multi-lingual / emoji-heavy text.
4. **Propose-card service** (all A, C) — posts structured propose-cards to (a) in-channel confirmer and (b) planner console; collects both before commit. **Audit-trail choke point.**
5. **Intake classifier + state machine** (G1, G2).
6. **Outbound rate limiter** (C, D, §4 J) — per-channel, per-recipient, per-day caps.
7. **Conversation memory store** — per-channel rolling window, per-tenant-isolated.

MILP, rate catalog, BSA accumulator, rules engine, schedule store, plan supersedence chain, override log, orchestrator, console, rule-author UI, and customs adapter all exist or are already MVP-scoped. Layer-3 is mostly **glue + propose-card service + identity/consent substrate**, not net-new optimization or data components.

---

## 3. MVP capability subset

Selection criteria: high operator value at real Persona-2/4 pain point, low compliance risk, high override-rate signal, synergy with the four MVP surfaces (`project_mvp_surface_scope.md`).

### Recommended MVP: 6 active + 3 substrate

**Substrate (non-negotiable; ship before any active capability):**

- **F1** Channel-join consent — required by WhatsApp Jan 2026 policy. Cannot ship without.
- **F2** Identity verification — substrate for any write-capable capability.
- **F3** Channel lifecycle close — retention / privacy substrate.

**Active capabilities:**

1. **B1 Fallback-arc diagnosis** — read-only over MILP output. Directly closes planner's "why is this HAWB in rescue?" pain (`air_freight_routing.tex §sec:output-diagnostics`). Highest demo value, zero write risk, generates indirect override-log signal (planner accepts or overrides agent root-cause).
2. **A6 Schedule update capture** — flight roll / blank sailing is highest-volume in-channel signal (user's CX123 example, `day-in-the-life/04 §4.2`). Execute-with-confirm because schedule resyncs from carrier feeds — mistakes self-heal.
3. **A2 Cargo-readiness slip capture** — own-HAWB-only blast radius; shipper is the natural in-channel confirmer; directly prevents downstream fallback-arc rescue events.
4. **G1 + G2 Intent classification + missing-data follow-up** — precondition for any Layer-2 quote-back. Counted as one MVP item.
5. **D3 Proactive cutoff reminder** — templated outbound, low ambiguity, high value (cutoff misses are the most expensive in-channel preventable event).
6. **B4 Why-this-routing Q&A** — read-only over MILP. High pitch value (differentiates from TMS); low risk (agent explains a decision the planner already endorsed).

### Excluded — rationale

- **A1, A3, A4, A5, A7** — all write to high-blast-radius catalogs (BSA, rate, rules, embargo, equipment). Defer until F2 / propose-card service has 4+ weeks of override-rate signal from lower-risk capabilities. The pitch lure is high; the operational risk is higher.
- **B2, B3** — useful but lower urgency than B1 (planners already understand cost/SLA flags from TMS workflows; fallback arc is novel to this product).
- **B5** — uses existing `disruption_impact_scan` tool. Ship via console first; add channel surface v2.
- **C1–C4** — all outbound LLM-drafted messages to external carriers/co-loaders. Highest paraphrase-error risk on capacity / rate commits. Defer until inbound extraction is proven.
- **D1, D2** — outbound customer comms have highest per-message reputational risk. Per `architecture.md §7.2` the founder rejected "customer-facing autonomous comms drafting" from MVP.
- **D4** — marginal value vs. risk; mid-size mostly lacks formal handovers (`day-in-the-life/04 §3`).
- **E1** — explicitly rejected as LLM task (`architecture.md §7.2`); implement as SQL job.
- **E2** — substrate spanning Layer-3, Layer-2, MILP-flow layer. Implement as orchestrator-level state machine; out of Layer-3 MVP scope, in trust-ladder MVP scope.

### Counterargument to my own MVP set

Three places to push back:

1. **B1 + B4 alone don't justify a Layer-3 buildout.** Both are read-only explanations that could ship in the planner console without any messaging-channel surface. A skeptical reviewer would say: ship B1+B4 in console; skip messaging until A6/A2 are also ready. The Layer-3 surface only earns its keep if A6 + A2 ship together with the read-only explanations.
2. **A6 schedule capture is half-baked without A1 BSA capture.** Flight rolls often arrive in the same WhatsApp burst as "and BSA was cut to compensate." If MVP captures the schedule but not the BSA, MILP re-solves on stale capacity and probably picks the same rolled flight. Either ship A6 + A1 together, or accept that A6's value is bounded by whatever the current capacity model says.
3. **G1+G2 is a Layer-2 commitment, not Layer-3.** Including it in Layer-3 MVP risks scope confusion. If MVP target is "ship Layer-3 first, before Layer-2 inbound intake," G1+G2 belong to Layer-2 MVP and Layer-3 MVP shrinks to 5.

Honest minimum if cutting further: drop G1+G2 (Layer-2 scope), drop D3 (existing `cutoff_alert` with a chat wrapper — could be deferred), leaving a 4-capability Layer-3 MVP: F1+F2+F3 substrate + B1, A6, A2, B4.

---

## 4. Failure modes and abuse vectors

Layer-3-specific risks. Real, plausible failure modes — not generic LLM risk.

**A. Identity verification — who is authorized to propose what.** The channel knows there is a participant at `+886-xxxx`; it does not know whether that participant is authorized to update CX's BSA. Per-tenant identity-to-role mapping (F2) gates writes by data store: BSA writes require LSP allotment manager; rate writes require rate desk; embargo writes require compliance approval. Without the gate, agent has no basis to write. **Mitigation:** F2 ships as substrate; every write capability runs the authorization check; unauthorized actor's proposed write is reflected back in-channel as "need confirmation from authorized party" and surfaced to console.

**B. Idempotency under message churn.** LSP sends "BSA cut to 8" at 10:01, "actually 6" at 10:03, "scratch that, 10" at 10:07. Naive flow posts three propose-cards and triggers three re-solves in 7 minutes. **Mitigation:** debounce window per (channel, data-field, key) — same propose-card key merges to latest within 10-min window. Only final value goes to console / re-solve. Schedule writes additionally cross-check carrier feed for partial mitigation.

**C. Conflict resolution — LSP and planner disagree.** LSP says "BSA cut to 6"; planner says "the cut was reversed yesterday in email." Both confirmers required, they disagree. **Mitigation:** propose-card stays `disputed`, surfaces to senior planner or KAM for resolution. Agent never picks a side. Dispute itself becomes a structured override-log event.

**D. Spoofing / impersonation.** Someone outside the consented channel forwards a screenshot to the agent; or an unverified handle joins the WhatsApp group and starts pushing updates. **Mitigation:** F1 — read only from consented channels; F2 — only writes from recognized identities; new-identity-in-channel triggers verify-via-known-party before that identity's messages can drive writes.

**E. Multi-tenant data leakage — the most dangerous failure mode.** Agent in WhatsApp Group A for Tenant X (CX rate desk + Tenant X) and Group B for Tenant Y (same CX rate desk + Tenant Y). Same human in both. Agent must never let Group A knowledge bleed into Group B responses or writes. **Mitigation:** all Layer-3 state — channel registry, conversation memory, propose-cards, extracted facts, rate snapshots — is namespaced by `tenant_id` as a first-class key. LLM context window per turn bounded to (this channel, this tenant). No cross-channel retrieval at inference time. Tested as a security property in MVP gate, not as emergent behavior.

**F. Compliance liability when agent-proposed + planner-confirmed update is wrong.** Agent extracts "BSA cut to 8," planner confirms, MILP books per new allotment, exceeds real allotment (was 12), carrier penalty. Who's liable? Contractual question, not technical. **Mitigation:** technical substrate produces audit-trail reconstructable by all parties — what was proposed, by whom, confirmed by whom, with source-message link. Audit-trail completeness is a hard MVP requirement (G).

**G. Audit trail completeness.** Every Layer-3 action logs: timestamp, channel-id, source-message-id (in-channel link), extracted-fact, propose-card-id, confirmer-identities, confirm-timestamps, applied-write target/delta, re-solve trigger, resulting plan-id. **Mitigation:** propose-card service is the choke point — nothing writes without going through it. Logging is structured JSONL (per `CLAUDE.md` pattern for `logs/agent_interactions.jsonl`).

**H. Conversation memory bounds.** Per-channel memory ends at: shipment-lifecycle close, channel-leave, retention TTL (GDPR for EU channels), or explicit operator command. **Mitigation:** F3 close-out + per-tenant retention config. Memory is bounded, not forever.

**I. Hallucination on Layer-3 reasoning.** Highest-stakes hallucination: extraction error converts "BSA 18 pallets" to "BSA 8 pallets" and propose-card commits. Numbers, units, lane codes, carrier codes all hallucination-prone. **Mitigation:** every extracted fact has Pydantic schema with validation; source-text shown in propose-card alongside extracted value so confirmer sees both. Below-threshold logprob on numeric field marks `low-confidence` and requires explicit re-entry by confirmer (not just a click). **Pre-launch eval gate:** extraction-eval suite over 200+ real in-channel messages with ground-truth annotations. Target: <1% misextraction on numeric fields, <2% on enum fields.

**J. LLM cost — per-message economics.** Every active-participation message has LLM tokens. 1 LLM call/inbound × 200 msgs/day/channel × N channels × N tenants scales linearly. **Mitigation:** tiered processing — small classifier (regex / distilled model) decides whether a message warrants LLM treatment. Only ~10–20% of messages are routing-relevant; rest is social / out-of-scope / acks. Outbound rate-limited per recipient per day. Track LLM cost per shipment as MVP operational metric. **Honest assessment:** the open question I'm least confident on — cost economics depend on what fraction of in-channel messages are routing-relevant, unknown until production. Substrate must support rate-limiting from day 1, right limits are empirical.

---

## 5. Architecture sketch — Layer-3 in the five-layer stack

Layer-3 lives primarily in **Layer 2 (Agent Layer)** with reach into Layer 4 (messaging-channel data adapters) and Layer 3 (intelligence components, via orchestrator).

```
┌─────────────────────────────────────────────────────────────────────┐
│ L1 USER SURFACES                                                    │
│   Planner console (flag inbox, propose-card review, plan diff)     │
│   Rule-author UI (consumes A4/A5 propose-cards)                     │
│   Override-log (records confirmer decisions on propose-cards)       │
│   Orchestrator-config (manual trigger for forced re-solve)          │
└─────────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────────┐
│ L2 AGENT LAYER                                                      │
│   ┌──────────────────────── LAYER-3 SUBSYSTEM (NEW) ────────────┐  │
│   │  Channel registry + consent ledger  (F1, F3)                 │  │
│   │  Identity registry + authorization gate  (F2)                │  │
│   │  Conversation memory (per-tenant, per-channel, TTL-bounded)  │  │
│   │  Intake classifier + state machine  (G1, G2)                 │  │
│   │  Extraction pipeline (Pydantic-validated)  (A, C-response)   │  │
│   │  Propose-card service  ←── audit-trail choke point           │  │
│   │  Outbound drafter + rate limiter  (C, D)                     │  │
│   │  Diagnosis explainer (LLM-RAG over MILP output)  (B group)   │  │
│   └──────────────────────────────────────────────────────────────┘  │
│   Existing agent capabilities (architecture.md §7.2):                │
│     Materiality assessment, query answering, replan-explanation     │
└─────────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────────┐
│ L3 INTELLIGENCE COMPONENTS                                          │
│   MILP optimizer + fallback-arc output  ──► feeds B1 diagnosis     │
│   Rules engine          ──► receives A4/A5 writes (after confirm)  │
│   Rate catalog          ──► receives A3/C3/C4 writes (after confirm)│
│   Capacity manager      ──► receives A1/A7/C1 writes (after confirm)│
│   Schedule store        ──► receives A6 writes (after confirm)     │
│   Graph generator       ──► consumes above on next re-solve         │
│   Orchestrator         ←── Layer-3 triggers re-solve via manual     │
│   Override log         ←── propose-card outcomes feed reason codes  │
└─────────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────────┐
│ L4 DATA ADAPTERS                                                    │
│   WhatsApp Business API adapter   (per-tenant push/pull)            │
│   LINE Business API adapter        (per-tenant push/pull)            │
│   Email adapter (Sedna / Wisor integration; not built)              │
│   SMS adapter (Twilio)                                              │
│   TMS adapter (write-through after confirm)                         │
└─────────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────────┐
│ L5 EXTERNAL SYSTEMS                                                 │
│   WhatsApp / LINE / SMS networks                                    │
│   Carrier portals, TMS, customs platforms                           │
└─────────────────────────────────────────────────────────────────────┘
```

**Proposer → confirmer(s) → applier flow:**

1. Inbound message via L4 channel adapter.
2. L2 Layer-3: identity check (F2) → intent classifier (G1) → extraction pipeline.
3. Extracted fact becomes a propose-card holding: source-message link, extracted value, target data-store, target key, current value, proposed delta, required confirmer-roles.
4. Propose-card service posts to (a) in-channel confirmer (LSP / shipper); (b) planner console (L1).
5. Both confirmations → propose-card `applied`. One missing → `pending`. Disagreement → `disputed`.
6. On `applied`: write to target data store (L3), log to override log with structured reason code, trigger orchestrator manual re-solve if plan-affecting.
7. Re-solve produces new plan in supersedence chain; console (L1) refreshes; affected customers receive D1/D2 notifications (separately HITL-gated).

**Orchestrator interaction.** Layer-3 never calls the MILP solver directly — re-solve triggers go through `orchestrator.manual_trigger(scope=affected_HAWBs_or_lane)`. Per `project_orchestrator_design.md` §4, manual triggers can cancel scheduled runs in progress; Layer-3 triggers are manual triggers. Scope bounded: A2 triggers re-solve on affected HAWB pool only; A6 on affected route; A1 on affected lane.

**Override-log feedback loop.** Every propose-card outcome (confirmed / rejected / disputed) generates a log entry extending the existing reason codes (`architecture.md §9`). Layer-3-specific codes: `agent-extraction-error`, `agent-source-misread`, `confirmer-disagreement`, `unauthorized-proposer`, `low-confidence-extraction`. E2 trust-degradation alarm reads this log; when Layer-3 extraction errors exceed threshold for an A-group capability, that capability auto-demotes on the trust ladder.

**Per-tenant isolation.** Channel registry, conversation memory, propose-card store, extracted-fact cache: all scoped by `tenant_id` as first-class key. No cross-tenant query path exists. Identity registry per-tenant — same human at CX rate desk may be authorized in Tenant X and not in Tenant Y. LLM prompt construction includes only current tenant's channel context; no retrieval from other tenants. Multi-tenant isolation tested as a security property in MVP gate.

---

## 6. Reshape recommendation (2026-05-27 discussion) — Layer-3 may not be in MVP

After review, the §3 MVP set is misshapen and the messaging-channel buildout (Layer 3) **may be deferred entirely from MVP**. Documenting the reshape so the decision is preserved.

### Why the §3 MVP is misshapen

1. **Two of the six "active" capabilities don't need a messaging surface.** Fallback-arc diagnosis (B1) and why-this-routing Q&A (B4) are plain-English summaries of MILP output. The planner is already looking at MILP output in the console; an "explain this" button there delivers the same value with zero messaging-channel infrastructure. Counting them as Layer-3 MVP inflates the apparent payload.
2. **Schedule-update capture (A6) is half-baked without capacity-cut capture (A1).** Flight rolls and BSA cuts arrive in the same chat burst. Capturing the schedule change but not the allotment cut → MILP re-solves on stale capacity and probably picks the same rolled flight. A6 alone delivers a partial picture; the demonstrable value requires the pair, and A1 is explicitly deferred for blast-radius reasons.
3. **Inbound intake (G1+G2) is Layer-2 scope.** The doc's own layering (`project_unstructured_channel_wedge.md`) places new-shipment intake in Layer 2. Including it in Layer-3 MVP conflates layers and inflates scope.

### Reshape (if Layer-3 is built in MVP)

| Item | Recommendation | Why |
|---|---|---|
| Channel consent / identity verification / lifecycle close (F1, F2, F3) | **Keep — non-negotiable substrate** | WhatsApp Jan 2026 policy + write-authorization + retention compliance. Required before any active capability. |
| Cargo-readiness slip capture (A2) | **Keep** | Own-HAWB blast radius, shipper is the natural in-channel confirmer, prevents downstream rescue events. |
| Cutoff reminder (D3) | **Keep** | Templated outbound, low ambiguity, prevents the single most expensive in-channel-preventable event (missed cutoff). |
| Fallback-arc diagnosis (B1) | **Move to planner console — no messaging surface** | Read-only LLM-RAG over MILP output. Ship as "explain this" in console. |
| Why-this-routing Q&A (B4) | **Move to planner console — no messaging surface** | Same as B1. |
| Schedule-update capture (A6) | **Hold — pair with A1 (capacity-cut capture) as the real flagship, or defer both** | A6 alone misleads the optimizer on the next solve. The chat-burst pattern requires both to land together. |
| Inbound intake (G1, G2) | **Move to Layer-2 inbound-intake build** | Not Layer-3 scope per the wedge doc. |
| All higher-blast-radius writes (A1, A3, A4, A5, A7), procurement outreach (C1–C4), customer comms (D1, D2, D4), learning loops (E1, E2) | **Defer (already deferred in §3)** | Stand. |

**Honest Layer-3 MVP after reshape:** consent + identity + lifecycle close + cargo-readiness slip + cutoff reminder. Two read-only console features (fallback diagnosis, why-this-routing) ship without messaging. Inbound intake ships in Layer 2.

### Why Layer-3 may be deferred from MVP entirely

**The "60–80% of afternoon is reactive ops" payload calculation.** Rough back-of-envelope:

- 60–80% of the forwarder ops afternoon is reactive (`docs/forwarder-operations-analysis/`).
- ~40–60% of reactive ops is message-driven (rest is phone, internal chat, portal-watching, TMS clicks). Asia-Pacific skews higher on messaging; North America lower.
- The reshaped Layer-3 MVP (cargo-readiness slip + cutoff reminder + two read-only console features) addresses maybe ~20–35% of message-driven volume — the high-volume write capabilities (BSA, rate, policy, embargo, equipment) are all deferred.
- Compound: reshaped MVP plausibly touches **~5–15% of total afternoon ops time**.

The big unlock is v2 (capacity-cut capture, rate updates, carrier-policy updates, embargo updates, procurement outreach). The reshaped MVP doesn't earn the buildout cost of channel registry + identity registry + propose-card service + extraction pipeline + rate limiter + conversation memory store on its own. Holding Layer-3 until the v2 capability bundle is ready is a defensible call.

**Multi-tenant isolation infrastructure is real net-new work.** Per-tenant scoping of channel registry, conversation memory, propose-card store, extracted-fact cache, identity registry, and LLM prompt construction — this is infrastructure that does not yet exist elsewhere in the system. It must be built before any Layer-3 capability that writes. This is a Phase-5 cross-cutting concern, not something solved inside the Layer-3 component.

### Decision

**Status: Layer-3 may not be built in MVP.** Two read-only diagnosis features (fallback diagnosis, why-this-routing) ship in the planner console. Inbound intake ships in the Layer-2 build. The flagship pair (schedule + capacity capture in the same chat burst) is the real Layer-3 unlock and should ship as a paired bundle when the supporting infrastructure (channel registry, identity registry, propose-card service, multi-tenant isolation, extraction eval gate) is built — likely post-MVP. Cargo-readiness slip and cutoff reminder are deferrable on the same timeline.

This decision is not final — re-evaluate when (a) v2 write capabilities are ready to bundle, (b) multi-tenant infrastructure is built for other reasons and Layer-3 becomes incremental, or (c) a design-partner contract specifically requires it.

---

## Executive summary

Layer 3 of the messaging-channel agent (active participation — propose corrections, diagnose infeasibility, run procurement loops) decomposes into 26 capabilities across seven groups: correction loops, diagnosis loops, procurement loops, communication loops, learning loops, identity/lifecycle substrate, and inbound-boundary handling. Most are integration / propose-card glue over existing components (MILP, rate catalog, BSA accumulator, rules engine, orchestrator, override log, console). Net-new components are channel + identity registries, propose-card service (the audit-trail choke point), intake classifier, extraction pipeline, outbound rate limiter, and conversation memory store.

Recommended Layer-3 MVP: six active capabilities plus three substrate items — fallback-arc diagnosis (B1), schedule-update capture (A6), cargo-readiness slip capture (A2), why-this-routing Q&A (B4), proactive cutoff reminder (D3), and inbound intent + missing-data follow-up (G1+G2), on top of channel-consent (F1), identity verification (F2), and lifecycle close (F3). All higher-blast-radius capabilities — BSA / rate / policy / embargo writes, procurement outreach, plan-change customer notifications — defer to v2 pending override-rate signal from the lower-risk MVP set.

The single most dangerous failure mode is **multi-tenant data leakage**: an agent participating in channels for multiple tenants must never let knowledge from one tenant's channel reach another's reasoning or writes. This is a security property, not an emergent behavior, and must be tested as an MVP gate. Hallucination on numeric extraction (BSA "8" vs "18") is the second-most-dangerous risk, mitigated by Pydantic-validated extraction with source-text shown alongside the extracted value in confirmer cards and a pre-launch extraction-eval gate (<1% misextraction on numeric fields).
