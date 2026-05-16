# Freight Industry Concepts — Reference

*Reference document for freight forwarding domain concepts. Not part of the PRD. Intended to give engineering and product team a shared vocabulary for the freight domain.*

*Last updated: 2026-05-16*

---

## 1. The Three-Party Structure of International Freight

Every international shipment involves at least three parties with distinct contracts between them:

```
Shipper (manufacturer / exporter)
        ↕  Commercial contract + HBL
Freight Forwarder (e.g., Dimerco, Morrison Express)
        ↕  MBL
Ocean Carrier / Airline (e.g., Evergreen, China Airlines Cargo)
```

The forwarder is the intermediary — they contract with the shipper to move goods, and separately contract with carriers to actually transport them. The forwarder's margin is the spread between what they charge the shipper and what they pay the carrier.

---

## 2. Bill of Lading — HBL and MBL

### Master Bill of Lading (MBL)

Issued by the **ocean carrier** to the **freight forwarder**. The carrier's contract of carriage.

- Forwarder's name appears as "Shipper"
- Legally binds the carrier to transport the container
- Contains: vessel name, voyage, POL → POD, container number, seal, commodity, weight/measurement
- Issued once per container (for FCL) or once per consolidated container (for LCL)

### House Bill of Lading (HBL)

Issued by the **freight forwarder** to the **shipper** (actual manufacturer/exporter). The forwarder's contract with their client.

- Shipper's (manufacturer's) name appears here
- Mirrors MBL content but reflects the actual shipper–forwarder relationship
- Multiple HBLs can sit under one MBL in LCL consolidations

### HBL/MBL Pairing

| Scenario | MBL count | HBL count |
|---|---|---|
| FCL (one customer, one container) | 1 | 1 |
| LCL (multiple customers, one shared container) | 1 | Many (one per shipper) |
| Multiple containers, one customer | Many | Many (one per MBL) |

In CargoWise/GoFreight, creating an HBL and "pairing" it to the MBL enables:
- Automatic cross-referencing of documents
- P&L tracking (cost from MBL side, revenue from HBL side)
- Arrival notices triggered by vessel arrival on the MBL

### B/L Release Types

| Type | How it works | Speed | Use case |
|---|---|---|---|
| **Original B/L** | 3 original copies issued; consignee must present one to take delivery | Slow — paper must physically arrive at destination | Trade finance, L/C transactions |
| **Telex Release / Express Release** | Carrier waives original; consignee takes delivery with email confirmation | Fast | Trusted parties, intra-group shipments |
| **Sea Waybill** | Non-negotiable; consignee named at booking; no document needed at destination | Fastest | High-trust, repeat lanes |

**Letter of Credit (L/C) impact on routing:** An L/C may specify the latest ship date, named ports (cannot reroute through a different POD), and specific document requirements. These become hard routing constraints — a shipment under L/C cannot be rerouted to an alternative port without risking document discrepancy and payment refusal.

---

## 3. Container Lifecycle — All Stages

The complete lifecycle of a container from origin factory to destination delivery:

| Stage | Event | Data generated |
|---|---|---|
| **1. Booking placed** | Forwarder sends booking to carrier; carrier confirms | Booking ref, container number, vessel/voyage, CY cutoff |
| **2. Empty pickup** | Trucker collects empty container from carrier depot/terminal | Gate-out at depot (CODECO EDI) |
| **3. Stuffing** | Shipper loads cargo at factory, seals container | VGM (Verified Gross Mass) transmitted to carrier (SOLAS mandate) |
| **4. Gate-in at CY** | Full container dropped at port terminal | CODECO gate-in confirmation; free time clock may start |
| **5. CY cutoff** | Deadline for container to arrive at terminal before sailing | Hard constraint in our routing model |
| **6. Loaded on vessel** | Terminal crane loads container | COARRI EDI message; bay plan (BAPLIE) position assigned |
| **7. Vessel departure (ATD)** | Vessel leaves POL | AIS tracking begins; transit time clock starts |
| **8. Transshipment** (if applicable) | Vessel calls hub port (e.g., Singapore, Kaohsiung); container discharged and reloaded | COARRI at each terminal; additional transit time leg |
| **9. Vessel arrival (ATA)** | Vessel arrives at POD | AIS speed → 0 at berth; port authority records ATA |
| **10. Discharge** | Container unloaded from vessel | Terminal places container in yard; demurrage clock starts |
| **11. Customs clearance** | Import entry filed; customs reviews, may hold for exam | CBP/ACE in US; exam adds 1–5 days dwell |
| **12. Terminal release** | After customs release + freight payment: carrier issues Delivery Order (DO) or PIN code | Container available for pickup |
| **13. Gate-out at POD terminal** | Trucker presents DO/PIN, collects container | CODECO gate-out; detention clock starts |
| **14. Delivery to consignee or CFS** | Full container to consignee warehouse (FCL) or CFS for LCL breakdown | Delivery confirmed; milestone update |
| **15. Devanning** | Consignee unloads cargo; for LCL: CFS breaks down container by HBL | Cargo received confirmation |
| **16. Empty return** | Trucker returns empty container to carrier depot/terminal before detention free time expires | Gate-in at depot; detention charges calculated |

**Milestones that feed our Rolling Horizon Controller:** Steps 9 (ATA), 10 (discharge), 12 (terminal release), 13 (gate-out) — these are the downstream events that may trigger destination leg planning or re-optimization.

---

## 4. Ocean Booking Flow

```
1. Shipper sends cargo-ready date and shipment details to forwarder
2. Forwarder queries carrier rates and sailing schedule
3. Forwarder sends BOOKING REQUEST to carrier
   → EDI: IFTMIN message (UN/EDIFACT) or direct carrier API
   → Via INTTRA neutral network (optional intermediary)
4. Carrier sends BOOKING CONFIRMATION
   → Assigns: booking reference, container number, vessel name, voyage number, CY cutoff date, SI cutoff date
   → EDI: IFTMBC message
5. Forwarder sends SHIPPING INSTRUCTIONS to carrier (SI)
   → Final cargo details: exact pieces, marks, consignee, notify party, freight terms
6. Carrier generates B/L DRAFT for forwarder review
   → EDI: IFTMCS message
7. Forwarder amends if needed; shipper approves HBL draft
8. B/L RELEASED after vessel departure + freight payment confirmed
```

**Key cutoffs:**
- **CY cutoff:** Last datetime the container can arrive at terminal and still make the sailing (typically 24–48 hours before vessel departure)
- **SI cutoff:** Last datetime shipping instructions can be submitted (typically 24–48 hours before CY cutoff)
- **VGM cutoff:** Last datetime Verified Gross Mass can be transmitted (often same as SI cutoff, SOLAS mandate)

---

## 5. Vessel Schedule Integration

A vessel schedule describes the full port rotation of a service/string:

| Field | Example |
|---|---|
| Carrier | Evergreen |
| Service/string | AEX (Asia-Europe Express) |
| Vessel name | Ever Apex |
| Voyage number | 0123E |
| Port rotation | CNSHA → CNNGB → SGSIN → DEHAM → NLRTM → GBFXT |
| ETD per port | CNSHA: Mon 06:00, CNNGB: Wed 14:00, SGSIN: Fri 08:00 ... |
| ETA per port | ... DEHAM: Day+28, NLRTM: Day+30, GBFXT: Day+31 |
| CY cutoff | CNSHA: Sat 18:00 (5 days before ETD) |

**How this feeds our Graph Generator:** Each leg of the port rotation becomes an arc in G(N,A). The ETD at POL + expected transit time = arc travel time. CY cutoff = the constraint `τ_k(i) ≤ CYC_{ij}` in P.1.

---

## 6. Trucking Instructions

A trucking instruction is a formal document the forwarder sends to a trucking company authorizing a container move. Required fields:

- **Container number** (e.g., MSKU1234567) and **seal number**
- **Pickup location:** Terminal name, address, gate hours, appointment requirement, PIN/gate reference
- **Delivery address:** Shipper factory (for empty pickup) or consignee warehouse (for delivery), contact name, phone
- **Cargo type:** Commodity, hazmat flag (if DGR), temperature requirement (if reefer)
- **Required arrival window:** Delivery appointment or CY cutoff
- **Job reference:** Forwarder's internal job number, for billing reconciliation
- **Special instructions:** Liftgate required, residential delivery, strapping required, permit required for oversized

In CargoWise/GoFreight, trucking instructions are auto-generated from the shipment record and sent to the trucker electronically (email, EDI, or direct platform access).

---

## 7. Road Consignment Note

The legal transport document for road freight — the trucking equivalent of a Bill of Lading. Jurisdiction-specific:

| Jurisdiction | Document | Notes |
|---|---|---|
| US domestic | **Bill of Lading (BOL)** | Also serves as receipt of cargo and contract of carriage |
| EU / international road | **CMR Note** (Convention on the Contract for the International Carriage of Goods by Road) | Standardized form; mandatory for EU cross-border road transport |
| Taiwan | Transport waybill (運送契約書) | Per Taiwanese road transport law |

Records: carrier, shipper, consignee, cargo description, weight, pickup address, delivery address, declared value, special conditions. Required for insurance claims if cargo is damaged in transit.

---

## 8. Intermodal Rail Booking

When an inland leg from a US port to a Midwest/Southeast destination moves by rail instead of truck:

**The intermodal move:**
1. Container arrives at marine terminal (USLAX, USLGB)
2. Drayage truck picks up container, delivers to **intermodal ramp** (e.g., BNSF San Bernardino yard)
3. Container loaded onto **doublestack flatcar** (two 40ft containers stacked)
4. Railcar assigned a **railcar number** and tracking ID
5. Train departs to destination ramp (e.g., Chicago Logistics Park, Dallas Wilmer)
6. Container drayed from destination ramp to final delivery address

**Booking:** Placed via BNSF e-Commerce portal / API or UP eService. Returns: rate quote, transit time, container reservation on a specific train, railcar number.

**Transit times (indicative):**
- USLAX → Chicago: 2.5–3.5 days rail (vs. 3–5 days truck)
- USLAX → Dallas: 1.5–2.5 days rail
- USLAX → Atlanta: 4–5 days rail

**Cost advantage:** Rail is typically 20–40% cheaper than FTL truck for hauls over 800 km.

**Data sources:** BNSF/UP public ramp locations from BTS FAF; service days from BNSF/UP published schedules. Live availability not publicly accessible — use published schedules as planning input.

---

## 9. ULD Management (Air Freight)

ULDs (Unit Load Devices) are the containers/pallets used to load cargo onto aircraft. They standardize cargo into aircraft-compatible shapes.

### ULD Types

| Type | Usable volume | Max payload | Fits |
|---|---|---|---|
| **LD3** | 4.5 m³ | 1,587 kg | Belly of wide-body (B777, A330, B787) |
| **LD7** | 11.1 m³ | 4,626 kg | Main deck of freighter |
| **PMC pallet** | 7.5 m³ | 6,804 kg | Main deck; flat pallet with net |
| **AKE** (small LD3) | 4.5 m³ | 1,497 kg | Same positions as LD3 |
| **AAP / AAF** | Various | Various | Narrow-body (B737 — limited belly space) |

### What is stored per ULD record

| Field | Description |
|---|---|
| ULD ID / serial | Physical tag on device (e.g., PMC12345CA) — carrier asset |
| ULD type | LD3, LD7, PMC, AKE |
| Carrier (owner) | Who owns/leases the ULD |
| Tare weight | Empty weight of the device |
| Loaded weight | Actual payload |
| Contents | AWB numbers loaded, piece count, weight, volume per AWB |
| Flight | Carrier, flight number, origin → destination, departure date/time |
| Build location | Where ULD was stuffed (origin CFS or airline warehouse) |
| Status | Planned / Building / Closed / Loaded on aircraft / In-flight / Arrived / Broken down |
| Contour | Profile type — determines which aircraft positions the ULD fits |
| Special handling | DGR (dangerous goods), PER (perishables), VAL (valuables), HEA (heavy) |

### Contracted ULD Allocation Layer

A forwarder pre-commits ULD positions per carrier/schedule per week (BSA-equivalent for air). Our `air_uld_allocations` table stores:
- Carrier + ULD type + origin airport → destination airport + departure days
- ULDs per departure (contracted quantity)
- Contracted all-in rate per kg
- Remaining ULDs (decremented as shipments are assigned)

The optimizer prefers filling contracted ULD positions before going to spot — better rate, guaranteed space. Overflow beyond contracted ULD capacity routes to the spot rate card.

---

## 10. Chargeable Weight (Air)

Air cargo is rated on **chargeable weight**, which is the greater of actual weight and volumetric weight:

```
Volumetric weight (kg) = volume (m³) × 167
Chargeable weight     = max(actual_kg, volumetric_weight_kg)
```

The 167 factor is IATA standard (1 m³ = 167 kg volumetric equivalent). This means bulky, light cargo (e.g., furniture, empty packaging) is rated on volume; dense cargo (e.g., machinery, metal parts) is rated on actual weight.

**IATA weight breaks:** Rate per kg decreases at volume thresholds:
- Minimum charge (N)
- +45 kg break
- +100 kg break
- +300 kg break
- +500 kg break
- +1,000 kg break

A shipment just under a weight break costs more per kg than one just over it — the optimizer must account for this non-linearity.

---

## 11. Surcharge Stack (Ocean and Air)

### Ocean surcharges

| Surcharge | Full name | What it covers |
|---|---|---|
| BAF / BSC | Bunker Adjustment Factor / Bunker Surcharge | Fuel cost fluctuation |
| PSS | Peak Season Surcharge | Carrier capacity management in high demand periods |
| EBS | Emergency Bunker Surcharge | Rapid fuel cost increases |
| CAF | Currency Adjustment Factor | FX fluctuations on some lanes |
| LSS | Low Sulphur Surcharge | IMO 2020 low-sulphur fuel compliance |
| THC | Terminal Handling Charge | Origin and destination terminal fees |
| AMS | Automated Manifest System | US import manifest filing fee |
| ISF | Importer Security Filing | US 10+2 filing fee |
| ISPS | International Ship and Port Facility Security | Security surcharge |
| GRI | General Rate Increase | Carrier periodic rate adjustment |

### Air surcharges

| Surcharge | Full name | What it covers |
|---|---|---|
| FSC | Fuel Surcharge | Jet fuel cost — refreshed monthly |
| SSC | Security Surcharge | Screening and security measures — relatively stable |
| AMS | Automated Manifest System | US air import manifest filing fee |
| THC | Terminal Handling Charge | Origin and destination airport handling |
| SCC | Screen Charge | Cargo screening requirement |
| AWC | Airway Bill Charge | Document fee |

In the air optimizer, the total rate = (base rate × chargeable weight) + FSC + SSC + origin THC + destination THC + AMS (US imports). All surcharges must be stored and applied correctly in the MILP cost function — not just the base rate.

---

## 12. Customs Filing — US Import

| Filing | System | Timing | Content |
|---|---|---|---|
| **AMS** (Automated Manifest System) | CBP / ACE | 24 hours before loading at origin port | Ocean manifest: all containers and commodities on vessel |
| **ISF** (Importer Security Filing / 10+2) | CBP / ACE | 24 hours before vessel loading | 10 importer data elements + 2 carrier data elements: supplier, manufacturer, country of origin, HS code, buyer, consignee, ship-to party, etc. |
| **Entry filing** | CBP / ACE | Before or within 15 days of arrival | Import declaration: HS classification, entered value, duty calculation, admissibility |
| **EEI** (Electronic Export Information) | AES / ACE | Before export | Export declaration for shipments >$2,500 or export-controlled items |

**Partner Government Agency (PGA) flags:** Certain HS codes trigger review by agencies other than CBP — FDA (food, drugs, cosmetics), EPA (chemicals), FWS (wildlife), USDA (agricultural products). These flags are determined at entry filing based on the HS code and add unpredictable hold time.

**C-TPAT:** Customs-Trade Partnership Against Terrorism. US importers with C-TPAT certification receive expedited customs processing — lower exam rates, priority release. Relevant for dwell time modeling: a C-TPAT importer has significantly lower inspection dwell than a non-certified importer.

---

## 13. Carrier Alliances (Ocean)

Ocean carriers operate in alliances that pool vessel capacity and share slot allocations:

| Alliance | Members | Market share |
|---|---|---|
| **Gemini** (as of 2025) | Maersk + Hapag-Lloyd | ~22% |
| **Ocean Alliance** (extended to 2032) | CMA CGM + COSCO + OOCL + Evergreen | ~29% |
| **Premier Alliance** | ONE + HMM + Yang Ming | ~17% |
| **Independent** | MSC | ~20% (largest single carrier) |

Alliance membership affects slot availability: a forwarder with a BSA on CMA CGM may be able to access COSCO slots on the same string via the Ocean Alliance agreement. This is a nuance in our allocation modeling — BSA contracts are carrier-specific but slot sharing within alliances creates cross-carrier access.
