# Grounding: spot rate-families — decay & reserve-then-assign (S54)

**Status:** research grounding for the D-A1 redesign (per-tier commit model). 6 parallel `Agent`
web-research calls (4 discovery + 2 adversarial), 2026-07-11. NOT the deep-research workflow harness
(avoided — the no-timeout `WebFetch` bug that hung S52). Companion:
`air_cargo_reserve_assign_grounding_s52.md`, `air_cargo_demand_arrival_grounding.md`.

## The questions

The sim's three "spot families" — `coload_per_kg`, `flat_rate`, `min_flat_breaks` — were tested against:
- **Q1 (decay):** does each family's available capacity deplete toward departure (booking curve)?
- **Q2 (reserve-then-assign):** can you book a *quantity* of space first and firm up *which shipments*
  fill it only near the cutoff?

## Framing correction (confirmed, high confidence)

`flat_rate` and `min_flat_breaks` are **not** two capacity products — they are two **rating structures**
on the *same* physical direct-carrier spot space. TACT weight-break ("next-break-down" round-up) vs flat
per-kg is pricing math applied to the same chargeable weight, determined at the same point (acceptance),
under the same booking-and-cutoff rules. The weight-break tariff imposes **no** earlier weight-declaration
or consolidation-lock. → the two collapse to **one channel** for both questions.
Source: IATA Knowledge Hub, *Air Cargo Tariffs and Rules* — "tariffs are … a rating/pricing mechanism
rather than an operational control system … Neither directly governs booking processes, space reservation
timing, or manifest declarations."
https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/air-cargo-tariffs-and-rules-what-you-need-to-know/

So there are **two real channels**: **direct-carrier spot** (flat + MFB) and **co-load** (coload).

## Q1 — Does capacity decay?

**Direct-carrier spot — YES (high).** Booking curve: <40% of capacity booked ~2 weeks out, >half in the
final week (McKinsey, *Ahead of the curve*). Corroborated by the direct-spot, decay-attribution, and both
adversarial agents.

**Co-load — YES (med-high).** The consolidator holds a *finite* block (a BSA/allotment with the airline)
and resells the remaining space first-come, so the sub-tenant faces a shrinking pool that sells out in
peak. It depletes on the *block-consumption* curve, **not** the airline free-sale s-curve (an
attribution nuance), but the net effect is depletion. Whether the consolidator re-runs its own booking
curve on its block = **NOT FOUND**.

**Load-bearing structural finding — decay is a CHANNEL property, not a physical-flight property (high).**
The flight is one shared pool; only the airline's **free-sale / spot** portion rides the curve down.
Allotment/BSA is carved out and committed up front. (Levin, Nediak & Topaloglu, *Operations Research*
60(2) 2012.) → **validates the code**: `CapDecay` decays only `spot_cap` (which covers coload+flat+MFB)
and passes both BSA tiers through firm. **Q1 requires no code change.**

**Honest caveat — a held spot booking is not perfectly firm.** Carriers overbook the spot pool against
no-shows and offload spot show-ups when capacity binds; there is a **firmness hierarchy**
(freighter/hard-BSA firm → soft-BSA → spot/belly bumped first). This is a tail IROPS event, not the
booking curve. Treating held spot as floored/firm is a defensible first-order simplification; the honest
v2 refinement is a small offload hazard on held **spot** only. (2D cargo overbooking lit; trade sources.)

## Q2 — Reserve space first, name cargo late?

**Direct-carrier spot — YES (high).** IATA Cargo-IMP separates the **FFR** (books space by route + weight
+ volume/dimensions for a nominated AWB number) from the **FWB** (shipper/consignee/final chargeable
weight), which firms at the documentation/cargo cutoff. IAG Cargo e-booking (carrier-primary): "shipment
details may be updated up to local cut-off times." Free to change/cancel before cutoff; penalty after.
https://www.iagcargo.com/en/e-booking-guide/

**Co-load — YES (med-high).** BSA/allotment structure: commit a *quantity* (pivot/committed weight)
early; assign HAWBs at the consolidation cutoff (~24h pre-alert; airline ramp cutoff ~2h before ETD).
Grounded for carrier↔forwarder BSA + the consolidation workflow; the *sub-tenant co-loader* clause
verbatim was **NOT FOUND** (inference).

**Three carve-outs (research scope = GENERAL cargo):**
1. **Special / regulated cargo** (DG, pharma, perishable, live animals) — commodity + identity forced at
   booking; cannot reserve generically or swap across commodity classes. cargo.one handles general cargo
   only. (AA Cargo DG; IATA DGR.)
2. **ACAS (US) / ICS2 (EU) pre-load identity floor (Tier-1 regulators)** — full shipper/consignee/AWB/
   description required **prior to loading**, not the physical cutoff. **Our Asia→US lane triggers ACAS**
   ⇒ identity locks at `min(doc cutoff, ACAS pre-load)`, slightly earlier than the physical cutoff.
   (US CBP ACAS / 19 CFR 122.48b; EU Commission ICS2.)
3. **Reserve requires commodity CLASS** — swap shipments *within* a class, not across. (Cargo-IMP FFR.)

**Friction (confirms the S52 Decisions 2 & 3):** free to change/release before cutoff; fractional or flat
no-show fee only on space still held-and-unfilled at cutoff (AA Cargo $300 >250kg; Lufthansa Cargo
25%<48h / 50%<24h). → reserve is **releasable, not monotone**; penalty is **fractional ~25–50%**.

## Proposed per-tier commit model (replaces the single-split D-A1)

Time-to-departure axis. Identity-lock (end of swap freedom) = `min(doc cutoff, ACAS pre-load)` on the
Asia→US lane, for all tiers. Scope = general cargo.

| Property | Hard BSA | Soft BSA | Spot | Fallback |
|---|---|---|---|---|
| Reserve (hold quantity) | owned from contract (−6/−12mo); no decision | block held from contract | **planner reserves a quantity EARLY (the fix), while φ still high** | n/a |
| Assign / swap cargo | free until identity-lock | free until identity-lock | free until identity-lock | at fallback |
| Identity-lock | `min(cutoff, ACAS pre-load)` | `min(cutoff, ACAS pre-load)` | `min(cutoff, ACAS pre-load)` | — |
| Release unused | NO (take-or-pay) | YES at release deadline (24–96h; code 48h), penalty-free | **YES — releasable before cutoff, penalty-free** | n/a |
| Friction on unused | pay pivot regardless (sunk `A_c`) | penalty-free if released by deadline; min-util = v2 | **fractional no-show ~25–50% (or flat) on held-and-unfilled at cutoff** | n/a |
| Decays (booking curve) | NO (firm) | NO per-cycle (release cliff only) | YES — free pool rides curve; reserved qty floored `f=max(r,b)` | n/a |

**Symmetry point (the S52 red-team fix):** BOTH replan arms (M1 open-book, M1′ single-pass) **reserve**
spot quantity early; the ONLY difference is **assignment fluidity** (M1 re-assigns the open book each
cycle; M1′ pins assignment early). Reserve floors both, so decay can't infeasibilize M1′ into the
self-inflicted 57%-fallback dump.

## What this changes vs. the current build

- **Q1: no code change** — decay already correct (channel property; coload decays; BSA firm).
- **Q2: build the redesign** — reserve-early / assign-late, grounded for spot + coload (general cargo).
- **Identity-lock** = `min(doc cutoff, ACAS pre-load)`.
- **Friction** = releasable + fractional ~25–50% (Decisions 2 & 3 confirmed).
- **Held-spot firmness** = keep floored now; small offload hazard = v2.

## Deferred to v2 (tracked in BUILD_STATUS "Deferred / parked" — do not lose)

- **Spot reservation CANCELLATION.** The chosen model (Level 2: commit a 2D weight+volume envelope
  `(r^w_a, r^v_a)` per spot arc) ships **v1 = no-cancel / monotone / pay-for-committed** (`penalty_frac=1`,
  `r_a` never decreases). The grounded extension — **release a reserved envelope before cutoff with a
  fractional no-show penalty** (~25–50%, `penalty_frac<1`, `r_a` allowed to drop pre-cutoff) — is v2. A
  `TODO(v2)` marker at the reservation site must point to the BUILD_STATUS deferred entry.
- **Held-spot offload hazard** (tail IROPS: overbooking/bump; spot/belly bumped first) — v2.

## Open gaps (do not overclaim)

- Co-load sub-tenant reserve-then-assign clause — inferred from BSA + consolidation workflow, NOT a
  co-loader tariff verbatim.
- Frequency of involuntary confirmed-cargo offload — NOT FOUND (modeled in RM lit, incidence unpublished).
- Whether the consolidator internally runs a booking curve on its own block — NOT FOUND.
- Soft-BSA release window — "advance notice" confirmed; NO universal number (48h is one point, ~24–96h).
- Coload is a **single arc** in the sim (`co_tpe_lax`); it is a token presence, not a real market.

## Key citations

- IATA Knowledge Hub — Air Cargo Tariffs and Rules. https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/air-cargo-tariffs-and-rules-what-you-need-to-know/
- IAG Cargo — e-Booking guide (carrier-primary reserve-then-assign + cutoff). https://www.iagcargo.com/en/e-booking-guide/
- American Airlines Cargo — Fair Booking Policy ($300 no-show >250kg; free before cutoff). https://www.aacargo.com/about/american-airlines-cargo-implements-fair-booking-policy.html
- Lufthansa Cargo — General Conditions (25%<48h / 50%<24h / 50% no-show). https://www.lufthansa-cargo.com/en/meta/meta/company/general-terms
- Levin, Nediak & Topaloglu (2012). Cargo Capacity Management with Allotments and Spot Market Demand. *Oper. Res.* 60(2):351–365. https://people.orie.cornell.edu/huseyin/publications/allotments.pdf
- McKinsey (2023). Ahead of the curve. https://www.mckinsey.com/industries/logistics/our-insights/ahead-of-the-curve-getting-cargo-revenue-management-right-as-the-cycle-turns
- US CBP — ACAS; 19 CFR 122.48b. https://www.cbp.gov/border-security/ports-entry/cargo-security/acas · https://www.law.cornell.edu/cfr/text/19/122.48b
- EU Commission — ICS2. https://taxation-customs.ec.europa.eu/customs/customs-security/import-control-system-2_en
- IATA DGR; AA Cargo DG. https://www.iata.org/en/publications/dgr/ · https://www.aacargo.com/learn/dg.html
- 2D cargo overbooking. https://www.sciencedirect.com/science/article/abs/pii/S037722170800204X
