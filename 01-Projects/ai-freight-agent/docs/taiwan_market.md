# Taiwan Market Analysis

*Research compiled May 2026. Primary market research not yet done — figures from first principles and public sources. Treat as directional.*

---

## 1. Market Sizing

### Taiwan Freight Forwarding TAM / SAM / SOM

| Metric | Estimate | Derivation |
|---|---|---|
| **TAM** | $15–20M | Taiwan freight forwarding gross revenue ~$6B (2.85% of $214B global market) × 1.75% software spend × 17.5% routing/optimization share |
| **SAM** | $1.5–5M | ~50–100 Tier 2 mid-market forwarders in Taiwan ($50M–$500M revenue) × $30–50K ACV |
| **SOM (3-year)** | $300K–$1M ARR | 10–20 early Tier 2 forwarder wins |

*See [`PRD.md §6.2`](../PRD.md) for the full Global / US / Taiwan side-by-side comparison table.*

**Taiwan is not a standalone TAM story.** The revenue opportunity in Taiwan alone is small. The strategic rationale for Taiwan is:
1. Morrison Express and Dimerco are exactly the Tier 2+ design-partner profile
2. TPEB (Trans-Pacific Eastbound) is our prototype trade lane — Taiwan is the source
3. High-value electronics cargo (TSMC supply chain) = deadline-critical = MILP value is highest here
4. No routing optimization competitor established in Taiwan
5. Win in Taiwan → APAC reference → builds into broader Asia-Pacific expansion

**Taiwan logistics market (broader):** $41.5B total logistics market (2025), growing to $57.9B by 2034 at 3.67% CAGR (IMARC). Freight forwarding is a subset.

---

## 2. Major Freight Forwarders in Taiwan

### Taiwan-Headquartered

| # | Company | Revenue / Scale | Mode focus | TMS |
|---|---|---|---|---|
| 1 | **Morrison Express** | $1.5B (2026) | Air + ocean, TPEB specialist | Likely CargoWise (proprietary claim: "sophisticated information systems"; global scale consistent with CW) |
| 2 | **Dimerco Express** | NT$29.68B (~$900M USD, 2025) | Air + ocean; 232,940 tons air freight | **Proprietary: Dimerco Value Plus System®** — confirmed NOT CargoWise |
| 3 | **Evergreen Logistics Corp** | Not public | Ocean forwarding + customs + project cargo | Likely proprietary (carrier-group affiliated, Evergreen Group IT) |
| 4 | **King Freight Group** | Not public; 60 branches, 1,300 employees | Ocean + air + landside | Unknown; possibly GoFreight or local TMS |
| 5 | **Sun Ocean & Air (SOA Logistics)** | Not public | Air + ocean | Unknown |
| 6 | **Great China Transportation** | Not public | Asia-Pacific focus | Unknown |
| 7 | **Fortune Freight / Fortune Transportation** | ~200+ staff | Ocean + air | Unknown |
| 8 | **Korchina Logistics Holdings** | Not public | Taiwan + mainland China network | Unknown |
| 9 | **Airlife Freight Taiwan** | Not public; est. 1974 | Air specialist | Unknown |
| 10 | **3-Leemark Logistics** | Not public; est. 1993 | Project / heavy cargo multimodal | Unknown |

### International Forwarders with Significant Taiwan Operations

| # | Company | HQ | Global scale | Taiwan operations | TMS |
|---|---|---|---|---|---|
| 11 | **Kintetsu World Express** | Japan | Top-15 global | Deep Taiwan electronics supply chain; air specialist | CargoWise (confirmed globally for Kintetsu) |
| 12 | **Nippon Express Taiwan** | Japan | Top-5 global; 18,104 tons air (Japan base) | Major air + ocean, Taiwan branch | CargoWise (confirmed globally) |
| 13 | **Yusen Logistics** | Japan (NYK Group) | 10,348 tons air (Japan base) | 11 offices, ~450 staff in Taiwan | Likely CargoWise |
| 14 | **Bolloré Logistics Taiwan** | France | 231 global warehouses | Est. 1986, 90+ employees, 4 Taiwan stations | CargoWise (confirmed globally for Bolloré) |
| 15 | **JAS Worldwide** | US | Global | 3 branches: Taipei, Taichung, Kaohsiung | Likely CargoWise |
| 16 | **GEODIS Taiwan** | France | Global | Full forwarding + contract logistics | CargoWise (confirmed globally) |
| 17 | **Hecny Group** | HK | 70+ services, 400K daily pieces | Major Taiwan ops | Likely CargoWise or regional TMS |
| 18 | **AIT Worldwide Logistics** | US | Global | Asia-Pacific including Taiwan | CargoWise or proprietary |
| 19 | **GAC Taiwan** | UAE | Global | Air + ocean + project cargo | Unknown |
| 20 | **Omni Logistics** | US | 50+ global locations | Taiwan presence | Unknown |

### TMS Landscape Summary

| TMS | Taiwan forwarders (confirmed / likely) |
|---|---|
| **CargoWise** | Nippon Express, Kintetsu, Yusen, Bolloré, GEODIS, JAS, Morrison Express (likely) |
| **Proprietary system** | Dimerco (confirmed), Evergreen Logistics (likely) |
| **GoFreight** | Mid-market and smaller Tier 2/3 forwarders; GoFreight explicitly serves Taiwan market |
| **Unknown / local** | King Freight, SOA, Great China, Fortune, smaller operators |

---

## 3. Dimerco Express — Deep Dive

### Overview
- Founded: 1971, Taiwan
- Revenue: NT$29.68B (~$900M USD, 2025) — publicly traded on Taipei Stock Exchange since 2001
- Air freight volume: ~232,940 tons — Taiwan's largest air forwarder by volume
- Operations: air + ocean forwarding, customs brokerage, project cargo, warehousing
- Geography: Strong Asia-Pacific base; global network
- Key carrier relationships: China Airlines Cargo, EVA Air Cargo, Cathay Pacific Cargo, major ocean carriers

### Technology Platform

**Dimerco does NOT use CargoWise.** Confirmed. They run a proprietary in-house system:

| System | Description |
|---|---|
| **Dimerco Value Plus System®** | Proprietary cloud-based logistics management system. Full suite of supply chain management applications. Patented "Data Synchronization Method®" for performance and reliability. ISO 27001:2022 certified. |
| **MyDimerco** | Client-facing portal. 24/7 visibility, customizable dashboards, real-time status refresh every 5 minutes, document access, reporting. |
| **eCall Freight® app** | Mobile app for Dimerco drivers and partner carriers. Barcode/QR scanning at key milestones (pickup, delivery). |
| **PO Management** | Purchase order tracking for shipper clients. |
| **WMS** | Warehouse management with mobile scanning devices. |

### Dimerco API — Integration Options

**1. Client status API (direct)**
- Dimerco offers API integration to clients for shipment status feed
- Allows clients to pull milestone updates into their own systems without leaving their platform
- Not a public developer API — requires commercial relationship with Dimerco as a forwarding client
- Scope: status/milestone updates, document access

**2. AfterShip tracking API (public, third-party)**
- Dimerco tracking data accessible via AfterShip API (no Dimerco account required)
- SDK support: Go, Python, PHP, Java, JavaScript, C#
- Scope: shipment tracking milestones only — not operational data (rates, booking, cargo instructions)
- Use case: for shippers who want to embed Dimerco tracking into their own apps

**3. IATA ONE Record (emerging)**
- Dimerco has implemented IATA ONE Record standard API for air cargo data sharing
- Confirmed via GLS (Global Logistics System HK) integration for CX Ultra Track on Cathay Pacific
- ONE Record is REST + JSON-LD; a modern standardized data sharing model across airlines, forwarders, ground handlers
- Signals: Dimerco is building to modern API standards; not locked into EDI-only integration

**4. No public booking/rate API**
- No evidence of a public API for rate queries, booking requests, or cargo instructions
- Operational integration would require direct commercial relationship and custom API agreement with Dimerco's IT team

### Integration Feasibility for Our Product

| Integration type | Feasibility | Path |
|---|---|---|
| Read shipment demand (origins, destinations, cargo) | Feasible | Dimerco client API or data export; requires commercial relationship |
| Write routing recommendations back to Dimerco | Feasible | Custom API agreement with Dimerco IT; or CSV/portal initially |
| Real-time status milestones | Feasible | Dimerco client API or AfterShip for tracking-only |
| Rate data (contracted rates) | Requires negotiation | Dimerco's contracted rates are in their proprietary system; likely CSV export or API endpoint |
| Carrier schedule data | Independent | We source carrier schedules directly (Descartes or public); Dimerco's schedule view is for their ops, not ours |

**Design partner path:**
Since Dimerco runs a proprietary system, there is no CargoWise eAdaptor complexity — we'd negotiate a direct API with their IT team. This is simpler to start (no partner program, no SOAP/eAdaptor architecture) but less scalable (one custom adapter per proprietary TMS). The Dimerco adapter would be built specifically for the design partner engagement.

**Why Dimerco is a strong design partner candidate:**
- Air freight volume (232,940 tons) means air optimizer is immediately relevant — not just a future-phase feature
- TPEB focus aligns with our prototype trade lane
- Proprietary TMS = cleaner integration negotiation than CargoWise
- Publicly traded = credible reference customer; investor-grade case study
- Taiwan HQ = easy face-to-face relationship development

---

## 4. Morrison Express — Notes

- Revenue: $1.5B (2026), privately held, Taipei HQ
- 2,000+ employees, 90+ owned offices globally
- Specializes in TPEB and tech/electronics supply chain
- Markets "sophisticated information systems" and client TMS integration capabilities
- TMS: likely CargoWise (scale and global operations consistent; unconfirmed)
- Considered a $1B acquisition target in 2022 (Bloomberg) — still privately held as of 2026
- Ideal CargoWise integration design partner if confirmed on CW

---

## 5. Competitive Software Landscape in Taiwan

**No routing optimization competitor is established in Taiwan.** This is a whitespace.

| Software category | Incumbents in Taiwan | Gap |
|---|---|---|
| TMS / operations platform | CargoWise (large forwarders), GoFreight (mid-market) | Neither does routing optimization |
| Route optimization | None identified | Our product |
| Visibility / tracking | project44, FourKites (limited Taiwan presence) | Visibility only; no optimization |
| Document automation | FreightMate.ai (US-focused, limited Asia presence) | Document processing only |
| Rate management | WebCargo / Freightos (air rates), Xeneta (benchmarking) | Rate data only; no optimization |

**GoFreight's Taiwan position:** Explicitly markets to Taiwan, Hong Kong, and greater China. Most accessible TMS integration for early design partners not on CargoWise.

---

## 6. Electronics Supply Chain — Why Taiwan Is a Strong Beachhead

Taiwan's freight profile differs from most markets in ways that favor our product:

**High-value, deadline-critical cargo:** TSMC, ASE, MediaTek, Foxconn supply chains move semiconductors, IC components, PCBs, and assembled electronics on tight delivery windows. A missed sailing on a TPEB route for a semiconductor customer is not a 3% cost variance — it can delay a production line. This is exactly the case where MILP value (certifiable optimality, probabilistic on-time modeling) is most defensible.

**Peak season capacity crunch:** Taiwan's tech export peaks align with US consumer electronics cycles (Q3/Q4). During these windows, TPEB BSA allocations fill up weeks in advance, spot rates spike 3–5×, and vessel slot availability becomes a genuine constraint. This is when our allocation constraint modeling (P.3) and portfolio-aware routing are directly tested.

**Direct carrier relationships:** Evergreen Marine and Yang Ming (both Taiwan-based) are among the world's largest ocean carriers. Taiwan forwarders have preferential BSA access to these carriers. Modeling Evergreen and Yang Ming allocations accurately is a baseline capability for any Taiwan forwarder deployment.

**China Airlines and EVA Air:** Taiwan's two major carriers operate among the world's largest air cargo networks. Both are top-10 global air cargo carriers by volume. Taiwan forwarders have preferential ULD access. Accurate ULD allocation modeling is immediately relevant for Dimerco's air freight operations.

---

## 7. Recommended Design Partner Sequencing

| Priority | Company | Rationale | Integration path |
|---|---|---|---|
| 1 | **Dimerco Express** | Largest air freight volume in Taiwan; proprietary TMS = simpler integration negotiation; IATA ONE Record adoption signals API readiness | Custom API with Dimerco IT team; Dimerco client status API |
| 2 | **Morrison Express** | $1.5B revenue, TPEB specialist, matches prototype trade lane exactly; likely CargoWise | CargoWise eAdaptor integration (if confirmed on CW) |
| 3 | **King Freight Group** | Midsized Taiwan-HQ, 1,300 employees; likely GoFreight or accessible TMS | GoFreight REST API (accessible; no partner program required) |

---

## Sources

- Morrison Express revenue: ZoomInfo, 2026
- Dimerco revenue: Public financial disclosures, Taipei Stock Exchange, 2025
- Dimerco air freight volume: Air Cargo News Top 25 Forwarders, 2025
- Dimerco technology: [dimerco.com/about/digital-freight-forwarding-and-logistics-technology](https://dimerco.com/about/digital-freight-forwarding-and-logistics-technology/)
- Dimerco ONE Record: [glshk.com Dimerco ONE Record](https://www.glshk.com/dimerco-uses-gls-one-record-api-to-monitor-live-shipment-status-on-cx-ultra-track/)
- Dimerco AfterShip API: [aftership.com/carriers/dimerco](https://www.aftership.com/carriers/dimerco/api)
- Taiwan logistics market: IMARC Group, 2025
- GoFreight Taiwan market: [gofreight.com](https://gofreight.com/)
