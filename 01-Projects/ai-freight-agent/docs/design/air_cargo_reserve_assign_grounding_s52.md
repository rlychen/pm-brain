# Grounding: "reserve space early, assign cargo late" in air cargo (S52)

**Status:** research grounding for the commit-timing / reserve-vs-assign proposal
(`model/commit_timing_reserve_assign_proposal.tex`). Deep-research harness, S52 (2026-07-05→08):
5 search angles → fetch → **3-vote adversarial verification** → synthesis. **64 claims upheld** (62
high-confidence), **10 refuted**. This doc records what is grounded, what was killed, and the
consequences for the model. Sources are cited inline; academic RM literature + IATA primary standards +
two carrier tariffs are the backbone.

## Question

Is it standard air-cargo practice for a forwarder to **reserve/book flight capacity in advance on
estimated weight/volume WITHOUT yet fixing which specific shipments fill it** — with the final manifest,
piece count, and actual chargeable weight submitted **later at a documentation/cargo cutoff**? I.e. is
"reserve the space quantity early, decide the exact cargo late" real, or a modeling fiction?

## Verdict

**Real and standard** — strongest for (1) the allotment/BSA channel, (2) the booking-vs-documentation
flow, and (3) consolidation. **Weakest for pure spot bookings**, where the booking tends to name the
consignment. Two findings bear directly on the model: **reserved-but-unused space is generally NOT free**
(validates the proposal's friction guard), and the **specific timing numbers (e.g. "48h release") are NOT
universal** (treat as a calibration range).

---

## Sub-claim 1 — Booking vs final documentation are two separate steps: CONFIRMED (IATA primary)

**IATA Recommended Practice 1670** — the e-AWB Model Agreement (Cargo Services Conference Resolutions
Manual), verbatim-verified via PDF text extraction
([rp1670.pdf](https://www.iata.org/contentassets/b38f5c2910e843bc967f4fff2d4fc53a/rp1670.pdf)):

- The forwarder sends the completed air-waybill data (FWB) **"to the Carrier prior to the presentation of
  the consignment at the Carrier point of acceptance."**
- The carrier's "ready-for-carriage" message (FSU/RCS) may **"deviate in weight, volume and/or total
  number of pieces from the … FWB information,"** handled by **pre-agreed exception management
  procedures**.
- **"A Cargo Contract shall be concluded once the Carrier has accepted the cargo"** (Cargo/Warehouse
  Receipt) — so the contract concludes at acceptance, not at data submission.

Declared-early → reconciled-at-acceptance is therefore the *official* standard. The cargo cutoff /
**Latest Acceptance Time (LAT)** is a distinct construct (IATA Cargo Handling Manual,
[ICHM page](https://www.iata.org/en/publications/manuals/iata-cargo-handling-manual/)). *Caveat:* the FWB
is per-consignment and already identifies a specific shipment — so at the individual-booking level this
supports **message-before-goods timing**, not the broader "reserve quantity / assign identity late"
separation. That separation comes from the allotment/consolidation channel (sub-claims 2–3).

## Sub-claim 2 — Allotment/BSA held as a quantity, cargo assigned per-flight later: CONFIRMED (heavy)

Textbook air-cargo revenue management; corroborated across ~40 verified claims spanning 2003–2025.

- **Levin, Nediak & Topaloglu, *Cargo Capacity Management with Allotments and Spot Market Demand*,
  Operations Research (INFORMS)** — [Cornell PDF](https://people.orie.cornell.edu/huseyin/publications/allotments.pdf):
  *"the physical capacity utilized by an allotment contract on the flight is random and becomes known at
  departure time… the actual utilization … realizes shortly before the departure when the shipments
  actually show up, is different than the space initially reserved."*
- **Amaruchkul, Barz & Klein, *Game-theoretic Analysis of Air-cargo Allotment Contract*, ICORES 2018**
  ([SciTePress](https://www.scitepress.org/publishedPapers/2018/65518/pdf/index.html)): air-cargo space is
  *"sold in two stages"* — months before the season the forwarder *"pre-books a certain amount of
  capacity"* on anticipated volume; daily allocation is set later after demand materializes.
- **Amaruchkul (2025), Int. Transactions in OR (Wiley)**
  ([doi:10.1111/itor.13613](https://onlinelibrary.wiley.com/doi/10.1111/itor.13613)): explicit two-stage —
  stage 1 determine allotments before demand known; stage 2 allocate cargo among flights after demand
  materializes.
- **Amaruchkul & Lorchirachoonkul (2011), Transportation Research Part E 47(1):30-40**
  ([ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S1366554510000797)): carrier
  allocates capacity to forwarders **before the booking horizon starts**; contribution based on actual
  usage at horizon end.
- **Carrier tariff (primary):** CMA CGM Air Cargo General Conditions of Sale
  ([PDF](https://www.cma-cgm.com/assets/public/documents/CCAC_General%20Conditions%20of%20Sale_20240331_V2_0.pdf))
  — *"Allotment = Cargo capacity reserved and allowed in advance by the Carrier for the Company, on a given
  flight number or route."*
- **KLM practice (Slager & Kapteijn 2003, J. Revenue & Pricing Management):** bids collected twice a year
  on IATA summer/winter schedules; contract utilization monitored **weekly** against performance targets.

## Sub-claim 3 — Consolidation (MAWB space booked, HAWBs finalized late): CONFIRMED in principle, thinner sourcing

The allotment two-stage structure + RP 1670 reconciliation carry this: a forwarder holds
consolidation/MAWB space as a quantity and finalizes the house-level contents by cutoff. **Honest
caveat:** the strong verified evidence is at the allotment / consolidation-load level; a *primary* source
spelling out "HAWBs swapped in/out of a MAWB until cutoff" at the granular level was not obtained (that
rested on forwarder-glossary/blog sources — lower tier). It is the standard understanding, but the
least primary-sourced of the three.

## Sub-claim 4 — Reserved-but-unused economics: reserving COSTS money (the model-load-bearing finding)

The research **refuted** the strong claim that air cargo "settles only on actual usage, not on reserved
quantity." Reality (upheld):

- **Hard BSA = pay regardless** ("use-it-or-lose-it"; minimum chargeable / pivot weight owed even if space
  isn't filled) vs **soft BSA = penalty-free release** up to a deadline, often with **minimum-utilization**
  clauses. (Amaruchkul allotment papers; **Hellermann 2006, *Capacity Options for Revenue Management*** —
  reservation fee for the *right* to use + execution fee if used; **Gupta 2008**, flexible
  carrier–forwarder contracts.)
- **Spot no-show fee (primary, current):** AA Cargo charges **$300** for failure to show by cutoff
  (shipments booked on/after 2025-08-25, chargeable weight >250 kg); cancel/change **before** cutoff is
  free ([aacargo rates](https://www.aacargo.com/ship/rates.html)).

**Consequence:** charging for reserved-but-unused space is *realistic*, not a modeling convenience — this
grounds the proposal's friction guard (Change 4).

## Sub-claim 5 — E-booking reality: CONFIRMED

IATA e-AWB / ONE Record and platforms (cargo.one, WebCargo by Freightos, IAG Cargo e-booking) lock a
capacity booking ahead of final shipment data; actual weight/dims/pieces reconcile at acceptance per the
RP 1670 FWB → FSU/RCS flow (sub-claim 1). ONE Record data model
([IATA](https://iata-cargo.github.io/ONE-Record/2024-12/Data-Model/waybill/)) preserves the same
booking-then-reconcile sequence.

---

## What got refuted — and why it matters

1. **"Settle only on actual usage / free release"** (3 claims killed): reserved space is commonly paid for
   (lump sums, unused-allotment penalties, minimum-utilization, hard-BSA use-it-or-lose-it, minimum
   chargeable/pivot weight). "Charge only actual usage" is the **soft-allotment regime only**, not
   universal. → reinforces Change 4.
2. **"Release deadline is usually 48 hours"** (3 claims killed): rests on one 2018 paper's uncited hedge;
   real washout/release deadlines are **contract- and route-specific, ~24–96h or even "a few weeks."**
   → our soft-BSA 48h cliff is **one plausible point, not a grounded constant**; treat as a calibration
   range. Resolves the "48h vs 2-week" discrepancy (different events, wide dispersion).
3. **Maersk block-space as evidence** (2 claims killed): that is an **ocean** (FFE / Weekly Volume
   Commitment) contract, not air; also stated weekly not monthly. The use-it-or-pay principle is real in
   air, but must be sourced from **air-specific** BSA terms (unitex-logistics, hkaircargo BSA program),
   not that document.
4. **ICHM Ch.7 / LAT "distinct from booking"** (1 killed): the supporting text was reseller marketing
   changelog copy; IATA's own 2026 changes page does not mention it. LAT-is-a-cutoff is fine; the specific
   ICHM-change citation is weak.

## Consequences for the model (carried into the proposal `.tex`)

- **Friction guard (Change 4) is grounded** — reserving is not free; charge reserved-but-unused space.
- **48h soft-BSA cliff → calibration range** (~24–96h+), not a universal constant.
- **Spot-reserve = consolidation-space framing** — the reserve primitive represents *consolidation /
  allotment-style* space-holding a forwarder will fill with house cargo, **not** a single-consignment spot
  booking (where the FWB names the shipment). State this explicitly in the methodology so the clean
  reserve/assign separation is defended at exactly the point a sharp reviewer would probe.
- **Out of scope but noted:** allotment/spot share one physical pool with show-up uncertainty → carriers
  overbook (Levin/Nediak/Topaloglu). We do not model no-show/overbooking.

## Method note

Verification was adversarial (2/3 refute-votes to kill). Confidence skews high because the core allotment
two-stage structure is decades-stable, multiply-sourced peer-reviewed RM doctrine plus IATA primary
standards. The killed claims were overreaches (universal timing figures, actual-usage-only settlement,
ocean-as-air), not contradictions of the core separation. Full verified claim set:
scratchpad `dr_verify.txt` (session-ephemeral); source URLs list in `dr_urls.txt`.
