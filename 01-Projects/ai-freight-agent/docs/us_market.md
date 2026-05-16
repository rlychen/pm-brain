# US Market Analysis

*Research compiled May 2026. Estimates from first principles and public sources. Primary market research not yet done. Treat as directional.*

---

## 1. Market Sizing

### US TAM / SAM / SOM

| Metric | Estimate | Derivation |
|---|---|---|
| **TAM** | $75M–$160M | US international freight forwarding ~$35–40B revenue × 1.75% software spend × 17.5% routing/optimization share |
| **SAM** | $25M–$50M | ~300–500 Tier 2 US forwarders ($50M–$500M revenue) × $30–50K ACV |
| **SOM (5-year)** | $2M–$8M ARR | 50–150 Tier 2 US wins at $30–50K ACV |

**US market size basis:**
- Total US freight forwarding and brokerage market: **$127.7B** (IBISWorld, 2026) — includes domestic trucking brokerage
- International freight forwarding portion (our relevant market): **~$35–40B** — sea and air forwarding only
- US is the single largest national market; ~20–25% of global freight forwarding software spend given higher tech adoption rate

**Sources:**
- US freight forwarding market: [IBISWorld 2026](https://www.ibisworld.com/united-states/industry/freight-forwarding-brokerages-agencies/1209/)
- US digital freight forwarding: ~$8.25B in 2025, growing at 19% CAGR (Mordor Intelligence)

---

## 2. Why the US Is a Priority Market

**TPEB destination:** The Trans-Pacific Eastbound lane (China/Taiwan → US West Coast) is our prototype trade lane. US-based forwarders operating TPEB imports are natural early customers — they're on the same lane we're building first.

**CargoWise concentration:** US large and mid-size forwarders disproportionately run CargoWise (among the highest adoption rates globally). The CargoWise integration we're building as critical path directly unlocks the majority of the US SAM.

**Regulatory complexity:** US customs (CBP/ACE, ISF, AMS, PGA holds) creates planning complexity that manual routing handles poorly at scale. HS code risk tiers and exam rates affect dwell time — modeling this correctly is a real advantage.

**Electronics supply chain:** Taiwan → US is the world's most important electronics trade lane (TSMC, Foxconn, ASE). High-value, deadline-critical cargo is exactly where our MILP value (P(on-time delivery), cost-optimal allocation) is most defensible.

---

## 3. Major US-Based International Freight Forwarders

US market has three tiers of players for our purposes:

### Tier 1 — Global Giants (NOT our target)
These companies build or buy their own tools. CargoWise handles their operations but they're too large for our sales motion and have internal optimization teams.

| Company | Estimated revenue | Notes |
|---|---|---|
| C.H. Robinson | ~$16B | Largest US-based, mostly domestic + intermodal |
| Expeditors International | ~$10B | Seattle HQ; privately held; known for proprietary systems |
| UPS Supply Chain Solutions | Part of UPS (~$91B) | Integrator network; heavily proprietary |
| FedEx Logistics | Part of FedEx (~$90B) | Same |
| Kuehne+Nagel US | Part of K+N (~$28B global) | Swiss HQ; strong US operations |
| DB Schenker US | Part of DB (~$21B global) | Now independent post-DSV Schenker deal |

### Tier 2 — Mid-Market (PRIMARY TARGET: $50M–$500M revenue)

These are the design partner candidates. They have dedicated ops teams, feel the routing labor pain, and can't afford to build their own optimizer.

| Company | Est. revenue | Mode focus | Likely TMS | Notes |
|---|---|---|---|---|
| **OIA Global** | ~$500M | Ocean + air, supply chain | CargoWise | Portland HQ; strong electronics/tech |
| **Radiant Logistics** | ~$400M | Multi-modal | CargoWise | Vancouver HQ, extensive US network |
| **AFN (American Freight Network)** | ~$300M | LTL + truckload focus | Varies | More domestic-focused |
| **Crane Worldwide Logistics** | ~$200M | Air + ocean, project cargo | CargoWise | Houston HQ; energy sector focus |
| **Agility Logistics US** | ~$200M | Air + ocean | CargoWise | Part of Agility Global |
| **AIT Worldwide Logistics** | ~$700M | Air + ocean, time-critical | CargoWise | Chicago HQ; strong air specialist |
| **Flexport** (post-restructuring) | ~$1B | Digital-native multimodal | Proprietary | SF HQ; raised $935M; direct competitor risk |
| **Echo Global Logistics** | ~$600M | Truckload focus | Varies | More domestic |
| **Worldwide Logistics Group** | Mid-market | Ocean FCL/LCL | GoFreight likely | NY/NJ area; TPEB specialist |
| **Shapiro** | Mid-market | Customs + ocean forwarding | CargoWise | Baltimore; NCBFAA member |

*Revenue estimates from public sources, ZoomInfo, and SEC filings where available. Private companies are approximations.*

### Tier 3 — SMB Forwarders (<$50M revenue): NOT our target
~5,000–8,000 small US forwarders. Use GoFreight, Magaya, or local systems. Price-sensitive; MILP complexity not valued at this volume.

---

## 4. US TMS Landscape

| TMS | US market position | Notes for our integration |
|---|---|---|
| **CargoWise** | Dominant for Tier 1–2 enterprise forwarders | Partner program required (4–12 weeks); per-customer integration 6–9 months; eAdaptor SOAP architecture (see `build_plan.md §8.1`) |
| **GoFreight** | Growing mid-market Tier 2–3 | Explicit US presence; REST API + webhooks; accessible without partner program |
| **Magaya** | SMB, US/LatAm focus | Below our target segment |
| **Descartes** | Enterprise, customs-heavy | Large forwarders and customs brokers; deep US customs integration |
| **Flexport (proprietary)** | Flexport only | Not relevant for our integrations |
| **Expeditors (proprietary)** | Expeditors only | Tier 1; not a target |

**Integration priority for US market:** CargoWise first (unlocks majority of Tier 2), GoFreight second (accessible earlier in product lifecycle).

---

## 5. US Competitive Landscape

### Who is in the US market for routing/optimization

| Player | What they do | Threat level |
|---|---|---|
| **project44 Intelligent TMS** | Visibility + TMS with "AI-driven optimization"; 1,000+ carrier integrations; US-headquartered | High — well-funded, broad distribution, heuristic optimization but improving |
| **Flexport** | Digital-native forwarding with internal optimization; not a SaaS product | Medium — they're a forwarder, not a software vendor; but could pivot to SaaS |
| **C.H. Robinson Navisphere** | Proprietary TMS sold externally; strong US domestic focus | Low for international routing |
| **Oracle TM** | Shipper-side TMS; enterprise; not forwarder-focused | Low — wrong buyer |
| **No MILP routing competitor** | No company offers MILP-based multimodal routing to US forwarders | Whitespace — same gap as globally |

**GoFreight in the US:** GoFreight has active US market penetration. Mid-market US forwarders on GoFreight have no routing optimizer today. Same whitespace as Taiwan, larger market.

### What US forwarders use today for routing
Manual planning. A planner opens the rate management screen in CargoWise or GoFreight, looks at 2–4 carrier options, picks based on experience, relationship, and gut feel. No optimization. No constraint modeling. No probabilistic transit time. No portfolio awareness.

This is the status quo we're replacing.

---

## 6. US Regulatory Complexity — Why It Matters for Our Model

US imports have the most complex pre-arrival filing requirements of any major market. This complexity creates planning constraints that our system must model:

| Requirement | Timing | Impact on routing |
|---|---|---|
| **AMS** (ocean manifest) | 24h before loading at origin | Hard: AMS filing must match the booked sailing; route changes after AMS filing require amendment and carrier coordination |
| **ISF** (10+2 security filing) | 24h before vessel loading | Hard: late ISF = $5,000 CBP penalty; routing must ensure ISF can be filed before departure |
| **PGA flags** | Determined at entry filing | Stochastic: FDA hold adds 1–5 days dwell; HS code risk tier feeds our customs inspection model |
| **C-TPAT status** | Per importer | Affects dwell time modeling: C-TPAT importers have ~60% lower exam rate |
| **USDC / OFAC sanctions** | Per consignee | Hard: routing must exclude carriers subject to sanctions orders |

These constraints are inputs to our rules engine (§2.11 in EXECUTION_PLAN.md) and the customs inspection dwell model (Phase 1 deferred item).

---

## 7. US Go-To-Market Considerations

**Sales motion:** Direct outbound to VP Operations / COO at Tier 2 forwarders. Target companies where TPEB is a significant volume lane. Initial messaging: routing labor reduction + allocation constraint optimization during peak season.

**Conference channels:**
- NCBFAA Annual Conference (National Customs Brokers and Forwarders Association)
- CSCMP EDGE (Council of Supply Chain Management Professionals)
- TPM (Trans-Pacific Maritime Conference) — held annually in Long Beach; the premier TPEB industry event; ideal for meeting Tier 2 forwarder ops leadership

**Partnership channel:** CargoWise partner ecosystem — once enrolled in the partner program, WiseTech's partner directory and events provide access to CargoWise-using forwarders who are actively looking for integrations.

**ACV target:** $30–50K/year for Tier 2. Higher ACV ($50–100K) possible for forwarders with high TPEB volume (>200 FCL shipments/month) where the routing labor savings are measurable and significant.

**Sales cycle estimate:** 3–6 months for mid-market forwarder. Decision maker is VP Ops or COO. Technical evaluation involves IT team for TMS integration. Budget authority varies — some have software budget lines, others need to build ROI case to CFO.
