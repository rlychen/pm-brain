# S51 — Air Cargo Capacity-Type Review

**Purpose:** enumerate the full real-world air-cargo capacity/procurement taxonomy, state exactly which
pieces our code models, and run two checks: **(1)** realism + comprehensiveness (independent research),
**(2)** what `src/` actually implements. Both complete.

---

## Read this first: three independent axes (the thing that got conflated)

The taxonomy, the billing mechanism, and the small set we actually build are **three separate things**.
They are not a cross-product. Every real-world type maps to exactly one disposition; you do not multiply
21 types by N cost bases by 6 buckets.

- **(A) Commercial capacity types** — the exhaustive real-world list (hard/soft BSA, allotment, GRA,
  tariff, spot, dynamic spot, co-load, GSA, charter, ACMI, deferred, express, NFO, interline, RFS, …).
  This is a **realism checklist**, not a build list. It answers "did we forget a way capacity is sold?"
- **(B) Cost basis** — *how a unit is billed*: per-kg flat, per-kg IATA weight-breaks, per-ULD pivot
  (flat-to-pivot + over-pivot), charter block/block-hour. This is an **attribute (a column)** of a
  type, correlated with it — not a free axis. A spot arc can bill per-kg *or* weight-breaks; a BSA
  bills per-ULD-pivot. Same billing math is reused across types.
- **(C) Modeled buckets** — the ~6 things the MILP + generator actually implement (Table 3). This is
  the single source of truth for "what's in the model."

Every row in Table 1 has **one honest disposition**: it *is* a modeled bucket, *is* a cost-basis
(attribute of a bucket), *collapses into the spot pool* (a channel/product on the same capacity),
is modeled *demand-side* (an SLA tier, not supply), is a *graph extension* (interline/RFS), or is
*out-of-scope*. Read the Disposition column that way.

One framing correction from check (1) that the taxonomy must respect: **price and space are orthogonal.**
A *price* contract (tariff, GRA, promo) fixes `$/kg` and reserves **no** space; a *space* contract
(allotment / soft BSA / hard BSA) reserves a **block** (nearly always with a rate too). Spot pre-commits
neither. So BSA ≠ GRA ≠ tariff, and "allotment" (the reserved block) vs "BSA" (the contract governing it)
are the capacity vs the paper. Our code folds rate onto offers (`reference_air_offer_rate_cardinality`);
"held space" is a separate attribute from "price basis."

---

## Table 1 — Commercial capacity-type taxonomy (axis A) with disposition

Exhaustive realism checklist. The **Disposition** column is the only thing that matters for the build:
it maps each type to bucket (C) / cost-basis (B) / spot-collapse / demand-side / graph-ext / out-of-scope.
Attribute columns are research-verified (check 1); dispositions are code-verified (check 2).

| # | Capacity type | Price/space | Deck | Offload priority | Disposition |
|---|---|---|---|---|---|
| 1 | **Hard BSA** (take-or-pay block) | Space | Freighter | Guaranteed (2nd) | **BUCKET** → Hard BSA |
| 2 | **Soft BSA** (cancellable block) | Space | Freighter | Guaranteed until release (3rd) | **BUCKET** → Soft BSA |
| 3 | **Allotment (belly)** | Space | Belly | Guaranteed | **Out-of-scope as a block**; belly is modeled spot-only → the *capacity* lands in the Spot-belly bucket, but there is **no reserved belly block** |
| 4 | **BSA secondary trading** (Airblox) | Space, resold | Either | Guaranteed | **Out-of-scope** (niche instrument) |
| 5 | **Published tariff / TACT / GCR** | Price (list) | Either | Space-available | **Cost-basis** → IATA weight-breaks (B), priced on the spot pool |
| 6 | **GRA / rate agreement** (price, no space) | Price | Either | Space-available | **Out-of-scope** — no committed-*price*-without-space contract (our cards are spot-priced) |
| 7 | **Promotional rate** | Price (tactical) | Belly | Space-available | **Spot-collapse** — approximated by the two-sided spot regime multiplier |
| 8 | **Ad-hoc / spot / free-sale** | Spot | Either | Space-available | **BUCKET** → Spot-freighter / Spot-belly |
| 9 | **Dynamic digital spot** (WebCargo / cargo.one / CargoAi) | Spot | Either | Space-available | **Spot-collapse** — a booking *channel* on the same pool as #8 |
| 10 | **Co-load / consolidator wholesale / GSSA** | Spot (resold block) | Either | Inherits | **BUCKET** → Co-load |
| 11 | **GSA / GSSA-sourced** | Either | Either | Inherits | **Spot-collapse** — a sourcing channel, not a distinct pool |
| 12 | **Per-ULD / pivot** | (mechanism) | Freighter | — | **Cost-basis** → per-ULD pivot (B); the BSA billing mechanism, not a standalone type |
| 13 | **Deferred / economy** | Spot | Either | Lowest (first offloaded) | **Spot-collapse + demand-side** — same spot pool; emerges from spot × slack deadline. Dropped S49 |
| 14 | **Standard / general** | Spot/contract | Either | Mid | **Demand-side** — the default SLA tier |
| 15 | **Express / priority / guaranteed** | Spot (premium) | Either | High (offload-protected) | **Demand-side** — an SLA tier, not a supply type |
| 16 | **NFO / OBC / time-critical** | Spot | Belly | Absolute top | **Out-of-scope** |
| 17 | **Full charter** | Spot/series | Freighter | Fully guaranteed | **Out-of-scope** — real peak-season lever (block/block-hour cost, not per-kg) |
| 18 | **Part / block charter** | Spot | Freighter | Guaranteed within charter | **Out-of-scope** |
| 19 | **ACMI / wet lease** | Contracted | Freighter | Guaranteed | **Out-of-scope** |
| 20 | **Interline / codeshare** | Either | Either | Inherits | **Graph-ext** — a multi-leg/stitched offer (`il_tpe_ord_thru`), not a pool |
| 21 | **RFS road-feeder** | Either | Feeds either | Follows air product | **Graph-ext** — ground arcs feed air nodes |
| — | **Fallback / backstop** | — | — | Last resort | **BUCKET** → Fallback (infeasibility relief valve, not a market product) |

**Physical source** (belly vs freighter main-deck) is a **column**, not a row: belly rides the pax
schedule with a thinner spot pool and softer cutoff; the freighter main-deck is larger, steadier, and
carries the BSA. It cross-cuts the buckets (see Spot-freighter vs Spot-belly in Table 3).

---

## Table 2 — Cost basis (axis B): how a unit is billed

Billing mechanisms, reused across types. Not a free axis — each type carries one (or, for spot, a small
menu). Code pointers are `RateCatalog` / `RateFamily` (`src/components/air_milp.py`).

| Cost basis | Billing math | Code | Used by |
|---|---|---|---|
| **Per-kg flat** | `max(min_chg·z, m·CW)` | `FlatRate` (`RateFamily.FLAT_RATE`); `coload_per_kg` | Co-load, flat spot |
| **Per-kg IATA weight-breaks** | `min_b rate_b·max(CW, break_b)` (round-up-for-cheaper-rate) | `min_flat_breaks` (`Break`) | Tariff/TACT spot |
| **Per-ULD pivot** | flat-to-pivot then over-pivot: `r_a·max(CW, π·Ση)` (soft) / sunk `A_c` + `r_c` overage (hard) | `BsaContract` (`RateFamily.PER_ULD_PIVOT`, `per_flight`/`equalized`) | Hard + Soft BSA |
| **Charter block / block-hour** | per-flight lump or block-hour | — | **Not modeled** (charter/ACMI out-of-scope) |

---

## Table 3 — Modeled buckets (axis C): the single source of truth

Exactly what the code builds. Verified against `src/`. Everything else in Table 1 is a disposition into
one of these (or out-of-scope).

| Bucket | Deck | Cost basis | Decays? | Code |
|---|---|---|---|---|
| **Hard BSA** | Freighter | per-ULD pivot, `equalized` (sunk `A_c` + `r_c` overage) | **No** (fully reserved anchor) | `BsaContract` `id="bsa-hard"`, `settlement="equalized"`; `_split_contracted` (`air_generator.py`); passed through untouched by `CapDecay` (`cap_decay.py`) |
| **Soft BSA** | Freighter | per-ULD pivot, `per_flight` (`r_a·max(CW, π·Ση)`) | **No — firm until the `dep−48h` release cliff** (F1/item-1), then capped to used-at-cliff | `BsaContract` `settlement="per_flight"`; C.13b (`_build_c13b_pivot`); firm in `CapDecay`; cliff in `run_replay` (`_soft_bsa_release_map`/`_apply_soft_bsa_locks`) |
| **Spot-freighter** | Freighter | per-kg flat / weight-breaks / co-load | **Yes** — convex booking curve | `spot_cap` on `flat`/`min_flat_breaks`/`coload_per_kg`; C.5d (`_build_spot_cap`); `CapDecay.phi`, `A_cut≈0.13` (freighter Beta(1.3,8.7)) |
| **Spot-belly** | Belly | per-kg (same menu), `0.4×` thinned | **Yes** — softer (higher residual) | same as above with `belly_spot_cap_frac=0.4` (`air_generator._is_belly`); `A_cut≈0.225` (belly Beta(1.8,6.2)); belly carries **no BSA** |
| **Co-load** | Belly (5J) | per-kg | Yes (inherits spot) | `coload_per_kg` (`RateFamily.COLOAD_PER_KG`); 5J co-loader in `tpeb_air_instance._offers` |
| **Fallback** | — | per-HAWB fixed | No | `FallbackPolicy` (`air_graph`); infeasibility relief, arrives `T^abs` |

Deck split (`AcType.FREIGHTER` / `PAX_BELLY`) is set in `tpeb_air_instance._leg` via `_BELLY_CARRIERS =
{5J, MU}`. The MILP itself is deck-agnostic; the deck only changes the generated `spot_cap` (belly
thinning) and the `A_cut` draw (`cap_decay`).

**Calibrated constants (do not re-invent):** `contracted_share ≈ 0.55` of `S_ℓ` is contracted, rest spot;
`hard_bsa_frac = 0.35` of contracted positions are hard, rest soft; `belly_spot_cap_frac = 0.4`; cutoff
bookable fraction `A_cut` freighter mean `0.13` / belly mean `0.225`; spot base `$5.5/kg × Uniform(0.85,
1.18)` regime; contract rate re-anchored to `≈$4.2 < $5.5` spot (so contracted is worth filling).

---

## Check (2) — code verdict

The code models **contracted-block (hard + soft BSA) + spot (3 cost bases) + co-load + belly/freighter
deck split + fallback** — the core of transpacific-headhaul forwarder procurement. Rate families =
cost bases = per-ULD pivot+overpivot (BSA), per-kg (co-load / flat), IATA weight-breaks. **Not modeled:**
charter / part-charter / ACMI (no whole-aircraft procurement); GRA / pure price-agreement; belly
allotment blocks; deferred/economy as a product (dropped — same spot pool); priority/guaranteed as a
supply product (it is demand-side SLA); dynamic-marketplace channel (folded into generic spot). The
material realism gaps are **charter** (a real peak lever) and the **GRA-vs-BSA** distinction.

---

## Check (1) — realism + comprehensiveness (VERIFIED)

One independent research agent survived (two died on infra "connection closed" errors mid-run); its read
is comprehensive, well-sourced, and consistent with existing memos, so it stands as the verification.

**Set is comprehensive** after adding tariff/TACT-GCR, promotional rate, GSA/GSSA channel, per-ULD/pivot
as an explicit cost mechanism, standard product, NFO/OBC, RFS, and BSA secondary trading (all now in
Table 1 with dispositions). The **price-vs-space orthogonality** is the main structural correction.
Nothing else material is missing for a transpacific-headhaul consolidation model.

**Dominant transpac-headhaul procurement mix** (SOURCED for direction; exact % is ASSUMPTION — no public
split): contracted **allotment/BSA is the base-load backbone** (carriers gate access — "No BSA, no way" —
and BSA rates run below spot); **spot / free-sale (increasingly digital) is the overflow tier**;
**co-load** matters for smaller forwarders; **charter/ACMI is a peak/disruption spike**, not steady-state.
This **validates the two-pool (BSA + spot) structure** and the mid-market `contracted_share ≈ 0.55`
(a BSA-heavy large forwarder would sit higher). Consistent with `reference_air_spot_contract_ratio` +
`project_supply_independent_of_demand`.

### Modeling findings

- **F1 — soft-BSA decay was modeled from the wrong side. FIXED + BUILT + independently reviewed (S51).**
  A soft BSA/allotment is the *forwarder's reserved space* — **firm until a release deadline**, at which
  *unused* space reverts to the carrier (a discrete washout, a forwarder *choice*), not a
  fill-against-the-holder curve. **Built (2 independent code reviews, PASS-WITH-NITS, 386 tests green):**
  `cap_decay` now decays **only `spot_cap`**; both BSA tiers pass through firm. Soft BSA is firm until a
  **48h-before-departure release cliff**, then capped to used-at-cliff (item-1, `run_replay`). Released
  positions are **dropped** (a BSA flight has no spot arc — "revert to spot" was corrected to "released to
  the carrier"). The spot decay floor is **tendered-only for M1/M1′/π_hind** (Model Y, item-2), with
  M1′'s decay-invalidated frozen priors dumped to fallback (item-3) + a fail-fast guard. See
  methodology §14.2 (updated) and `src/cap_decay.py` / `src/replay.py`.

  **Confirmed lock/release mechanism (implementation, user-approved S51).** The keep-or-release decision is
  **deferred to the last minute** — executed on the **last planning run before `dep − 48h`** for each
  soft-BSA flight (add `dep − 48h` as a per-flight planning cadence point, the way flight cutoffs already
  are, so the decision fires at exactly the last moment with max information). On that run, **position by
  position**: if the current plan **uses** the soft-BSA position → **lock it in** (it stays firm at the BSA
  rate through tender); if **unused** → **do not lock** → **release it to the carrier** (those positions
  leave the soft-BSA pool and revert to the spot pool at spot price, under the spot booking curve). It is a
  one-time discrete event per flight — no continuous decay. Min-utilization floor (~60–70%), tier/peak
  dependence, and partial-release granularity are **v2** (grounded: Amaruchkul 2018; the 48h anchor is
  SOURCED — see check-1 sources).

- **F2 — no explicit offload-priority ordering.** Real bumping under overbooking follows a hard rank:
  NFO/OBC → express-guaranteed → hard BSA → soft BSA/allotment → standard → deferred (first offloaded).
  Our model has no priority-based bump; short capacity is resolved by MILP cost-min + fallback. Defensible
  for a *cost* proof; the missing piece only if we ever score *which* shipments strand (service realism).
  Ties to `feedback_no_standalone_cost_pruning`.

### Confirmed-correct modeling choices (research backs them)

- **Deferred = same spot pool** (drop was right; `reference_deferred_air_capacity`).
- **Hard BSA = sunk take-or-pay → cheap marginal fill** up to the block (`A_c` + `r_c` overage).
- **Pivot/ULD kept as a step function** (not linearized) — the core consolidation lever (C.5b integer ULD).
- **RFS/interline as graph extensions**, not new capacity pools (`project_trucking_approximation_boundary`).
- **Belly = spot-only** (no belly BSA), `0.4×` thinned, higher cutoff residual (softer) than freighter.

### Named gaps (deliberate)

- **Charter/ACMI** — absent; the genuine peak-season capacity-creation lever. Add only if a peak scenario
  or large-forwarder persona needs it (per-flight/block-hour cost, not per-kg).
- **GRA vs BSA distinction** — we model space-with-price (BSA) and spot-priced cards, but not a
  committed-*price*-without-space GRA. Low priority for the replan proof.
- **Belly allotment blocks / dynamic-marketplace channel / promo rate** — folded into spot; acceptable at
  proof scale.

### Sources (agent-retrieved)

BSA hard/soft + allotment: cargo.flowers/en/blog/post/why-do-you-need-bsa · media.shipco.com (block-space
uptick) · Loadstar "No BSA, no way" (corroborated). Price-vs-space (GRA/TACT/GCR/promo): iata.org/en/
services/compliance/tact/ · CMA-CGM air general conditions (General/Promotional/Contract rates) ·
aviation-professional.net GCR. Weight-breaks / pivot / chargeable weight: maersk.com logistics-explained ·
learnvern.com ULD & pivot rates · cargo.koreanair.com tariff. Digital spot: webcargo.co · cargo.one ·
cargoai.co · stattimes.com e-booking. GSA/GSSA + co-load: help.cargoai.co what-is-a-gsa · kales.com/
gssa_cargo. Charter/ACMI: aircharterservice.com guide-to-acmi · atlasair.com/cargo-services/acmi. Service
tiers/offload: exfreight.com service-levels · freightamigo.com express-standard-deferred. Guaranteed
products: skycargo.com emirates-airfreight-priority · lufthansa-cargo.com (td.* names INFERRED from
snippets — verify before hardcoding). RFS/interline: lufthansa-cargo.com road-feeder-service. Academic
(spot overflow above contract): Amaruchkul 2025 (onlinelibrary.wiley.com/doi/10.1111/itor.13613);
flexible carrier-forwarder contracts (ResearchGate 32028882). BSA trading: stattimes.com Airblox.
Marketplace share ~41% 2026: futuremarketinsights.com (vendor figure — soft).
