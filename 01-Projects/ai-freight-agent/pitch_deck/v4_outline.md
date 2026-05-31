# v4 outline — reframed pitch structure

Vision lane: A — "The planning brain for global freight."
Scope honesty: air consolidation is the wedge; multimodal is the long arc.
Tier strategy: land in tier 2, expand up to tier 1 (narrow-capability) and down to tier 3 (channel/self-serve).
MILP framing: deterministic, certifiable, autonomy-ready — the moat.

---

## 1. Vision (headline)

**The planning brain for global freight.**

Every shipment in the world is a planning decision — what mode, what carrier, what consolidation, what to do when something breaks. Today that brain is fragmented across five disconnected mode tools, thirty thousand forwarders, and millions of human-hours. We are building one engine, rented by any forwarder, that runs the planning loop end-to-end across air, ocean, and trucking.

Supporting line: "Tier-1 routing intelligence as infrastructure. We don't replace the TMS. We replace the planner's brain."

## 2. Problem

Forwarders compete on intelligence now, not headcount. Margins compressed, volume fragmented, disruptions constant. But the planning layer has not modernized:

- **Fragmented tools**: separate rate engines, consolidation tools, visibility platforms, exception dashboards — one per mode, none aware of the others.
- **Tier-1 advantage is widening**: DSV (Tango), K+N (eTouch), DHL (MyDHLi) each invested 100+ engineers and 5 years into proprietary planning systems. The other 30,000 forwarders cannot build this and cannot survive without it.
- **AI overlays target the wrong layer**: document AI (Raft), inbox copilots (Augment) save minutes per task. They do not make the routing decision.

> Headline impact number — drop the unverified "40% → 15%" claim. Replace with directional language ("hours of planner capacity returned per shipment") until validated by named design-partner conversations. Re-insert with citation only after validation.

## 3. Solution — one engine, agentic, multimodal

A single planning engine that ingests a shipment request and returns an end-to-end route decision across modes. Built as independently testable components — mode optimizers, transit-time predictors, graph generation, rules engine, stitching layer — coordinated by an agentic orchestration layer.

Customer-facing surfaces:
- **Quote desk** (volume entry point, low-stakes wedge)
- **Consolidation planner** (defensible, high-stakes wedge — no incumbent)
- **Exception handler** (replanning loop, same engine reactivated)

## 4. Wedge — air consolidation (MVP)

Why air consolidation first:
- Hardest planning math (build-up windows, density constraints, capacity churn, transshipment timing).
- Highest labor cost per shipment.
- No dominant SaaS incumbent in the consolidation planning layer.
- Same engine extends to ocean LCL with minimal new math (LCL is consolidation, structurally).

## 5. Roadmap — multimodal expansion

| Phase | Component | Why next |
|---|---|---|
| MVP | **Air consolidation planner** | Highest math complexity; biggest labor unlock; no incumbent. |
| Phase 2 | **Ocean LCL consolidation planner** | Same consolidation math as air; immediate engine reuse. |
| Phase 3 | **Ocean FCL routing** | Different math (allocation + vessel cap), already have draft LaTeX model (`ocean_fcl_routing.tex`). |
| Phase 4 | **Trucking routing** | Closes the door-to-door loop. |
| Phase 5 | **Intermodal / stitching** | Cross-mode optimization. The thing tier 1s do that no one else can. |

## 6. Why now

- **Vertical AI ARR ramps are proven**: Harvey $50M → $195M ARR in 12 months at $11B valuation. The services-as-software model works when AI replaces knowledge work, not seats.
- **Agentic infrastructure matured in 2025**: LangGraph, MCP, planner-validator patterns — autonomous planning is now reliable enough to commit without human review *when grounded in deterministic math*.
- **Industry timing**: DSV–Schenker integration (closed April 2025) creates a 2–3 year tier-1 distraction window. CargoWise's dominance (24 of A&A Top-25) means no one is competing to *replace* the TMS — only to add intelligence on top.
- **AI overlays are landing**: Raft (40% of A&A Top-25, document AI), Augment ($85M Series A, inbox copilot). Both validate forwarder appetite. Neither owns the planning decision.

## 7. Why us

Built this architecture twice inside tier-1 platforms. We know the math, the data shape, the failure modes, the operator UX, and the integration surface. We are packaging what only tier-1s could afford to build — and we are doing it before they license theirs out.

## 8. Market

**Sources**: Armstrong & Associates (3PL/forwarding revenue), Transport Intelligence, WiseTech filings, public AI-native logistics raises.

| Layer | 2026 / 2031E | Construction |
|---|---|---|
| **TAM — global forwarding revenue** | **$216B** | A&A 2024. Pool of customer revenue we orchestrate. |
| ‣ Air freight forwarding | ~$110B | Mode split, ±20% (paywalled primary). |
| ‣ Ocean (FCL + LCL) | ~$75B | GMI 2023 estimate, ±20%. |
| ‣ Road / multimodal residual | ~$30B | A&A residual. |
| **Software TAM today** | **$1.5–2B** | WiseTech $680M + Descartes ~$200M forwarder + private mid-market $200–400M. |
| **5-yr AI intelligence layer SAM** | **$4–9B** | 2–4% of forwarder revenue captured as ops-replacement spend (call-center AI economics). |
| **5-yr SOM (base)** | **$20–180M ARR** | 0.5–2% of SAM via tier-2 wedge + narrow tier-1 capability landings. |
| **5-yr SOM (bull)** | **$200–400M ARR** | Harvey trajectory if services-as-software outcome pricing lands. |

**Key reframe**: TAM is the revenue *we orchestrate*, not the routing software slice. The AI intelligence layer expands the wallet by 4–5x over current TMS spend because it replaces ops headcount, not seat licenses.

## 9. Tier strategy — land in 2, expand both ways

| Tier | Revenue band | Forwarder count | Approach | ASP |
|---|---|---|---|---|
| **Tier 1** | >$5B | ~25 globally | Narrow-capability overlay alongside CargoWise. Sell the planning brain, not the TMS. Raft precedent (40% of Top-25). | $250k–$1M+ ACV per capability |
| **Tier 2** | $100M–$5B | ~3,000 globally | **Landing zone.** Full intelligence layer. Direct sales. | $30–100k ACV |
| **Tier 3** | $5M–$100M | Tens of thousands (FIATA ~40k members; IBISWorld 76k US) | Channel partnerships, TMS marketplaces, self-serve. Same engine, packaged. | $1–5k/month |

DSV–Schenker integration distracts the largest tier-1 buyer for the next 2–3 years. Window is open.

## 10. Moat — the math layer

Most AI competitors are LLMs with retrieval. Their agents cannot commit a booking without a human approving the decision — they save minutes, not headcount.

Our engine is **deterministic, certifiable, and autonomy-ready**:
- MILP for high-stakes routing decisions (provable optimality, auditable constraints)
- Fast heuristics for routine cases
- Single orchestration agent that picks the right algorithm per problem
- Every decision carries a certificate the agent uses to commit without human review

This is what tier-1 internal platforms do. No one else is shipping it as a product.

## 11. Competitive

| Player | Surface | Why we win |
|---|---|---|
| CargoWise / WiseTech | TMS system of record | We sit on top, not against. Integration partner, not replacement. |
| Augment, Raft | Inbox / document AI | They save minutes per task. We own the planning decision. |
| Flexport, Forto | Digital forwarder operator | They became forwarders. We sell to forwarders. Forto pivoted to SaaS (FortoLabs, Oct 2025) — vindicates the shift. |
| Tier-1 in-house (Tango, eTouch, MyDHLi) | Proprietary planning brain | Proof the architecture works. We are the only commercial version. |

## 12. Team

(Founder credentials — built tier-1 routing intelligence platforms twice. Specifics to land here.)

## 13. Ask

(Round size, use of funds, milestones — to be set by founder.)

---

## Open items before deck regen

1. **Headline metric replacement** — pick the directional claim that replaces "40% → 15%" until validated. Options:
   - "Returns ops capacity equivalent to 1.5–2 senior planners per $200M forwarder"
   - "Cuts planning time on the worst 3% of shipments by half"
   - "Replaces a tier-1 routing team with software"
   - Drop entirely, lead with the multimodal scope claim.
2. **Validate via design partners** — 3–5 named conversations needed before any specific time-reduction number lands in the deck.
3. **Team slide content** — confirm what's safe to disclose given the no-prior-employer rule.
4. **Round shape** — size, use of funds, 18-month milestones.
