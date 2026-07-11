# Air Pricing & Capacity Redesign — Increasing-Block Spot Supply (S45)

**Status:** DESIGN PROPOSAL — behind the formal-model approval gate. No code is written until the user
approves. This is step 2 of 2; it consumes the sourced numbers in
`docs/design/air_pricing_calibration_s45.md`.

**What it fixes:** the measured inertness of the replan headline `L2 = C(M₁') − C(M₁)` to capacity
tightness κ (critique-20 A1, critique-18 F-A). As built, spot capacity is a flat per-kg rate with
effectively unbounded aggregate capacity (~144–152k kg available over the arcs vs a ~7.5k kg book; the
premium `m ~ U(0.85,1.18)` is κ-independent by S38 design). Scarcity never binds, so the κ sweep cannot
move L2 — measured L2 was byte-identical across κ for 5 of 6 seeds, and the loose corner did **not**
collapse to L2≈0.

**Confidentiality:** no prior-employer or product names appear in this document or its proposed code.

---

## 1. Problem recap (what the flat-rate model destroyed)

The thesis mechanism the methodology sells (§13, D-A19) is: under tight contracted capacity, the engine's
value is **juggling scarce cheap capacity** — packing the cheap contracted slots and the cheap spot block
with the cargo that benefits most, and pushing marginal/low-priority cargo to expensive capacity or
deferring it. M₁ can re-pack the cheap capacity when better cargo arrives later; M₁' froze its early
placements and cannot. The *difference* is L2.

That mechanism requires the cheap escape to be **finite**. The current model makes spot:

1. **Flat** — one per-kg rate per arc; the marginal kg costs the same as the first kg.
2. **Effectively unbounded in aggregate** — per-arc caps `U(1,3)×1500 kg`, summed over ~48 spot arcs,
   give ~144k kg against a ~7.5k kg book. No realistic demand ever exhausts it.
3. **κ-independent** — spot is drawn on the `spot_regime` stream so it is byte-stable as κ sweeps (the
   deliberate S38 choice that keeps the κ axis "contracted tightness only").

Net: when κ tightens contracted ULD positions (16→1), cargo just spills onto the always-available
flat-priced spot. The marginal cost of spilling does not rise, so the saved-by-reshuffling premium does
not rise, so L2 does not move. **The escape valve has no back-pressure.**

The fix is to give spot a **finite, upward-sloping (increasing-block) per-lane supply curve** with a hard
ceiling that hands off to an expensive fallback. Then tightening demand-vs-finite-capacity competition
(via κ, see §6) pushes marginal cargo up the block curve, and the reshuffling engine's ability to keep
the cheap blocks for the cargo that benefits most becomes worth real dollars — growing as κ tightens.

---

## 2. The value mechanism (the thesis, restored)

With a finite increasing-block spot curve **per lane**:

- The lane's first ~5,000 kg clears at base spot ($5.5/kg); the next blocks step up ~1.2× each; beyond
  ~10–12k kg the lane is exhausted and cargo rolls to the **fallback** (~2.5× base, ~$13–15/kg).
- **M₁ (open book / reshuffle):** sees the whole revealed book each cycle and can re-pack the cheap B0
  block + cheap contracted ULD slots with whichever cargo is currently most valuable to place cheaply
  (most urgent, or heaviest-discount-eligible, or the consolidation that mixes best). Late-arriving
  high-value cargo can displace earlier-placed marginal cargo *out of* the cheap block and into B1/B2 or
  defer it.
- **M₁' (single-pass / frozen priors):** committed earlier cargo to the cheap block at reveal time and
  **cannot re-pack it**. When better cargo arrives later, the cheap block is already gone; the new cargo
  pays B1/B2/fallback even though, globally, it should have had the cheap slot.

`L2 = C(M₁') − C(M₁)` is exactly the cost of that frozen mis-allocation of finite cheap capacity. **It is
monotone increasing in how hard demand competes for the cheap blocks** — i.e., in tightness. At the loose
corner (abundant cheap capacity relative to demand) nobody is forced up the curve, every arm fills B0,
and L2 → 0 — restoring the falsifiability null the flat model broke.

This is the channel critique-18 calls "the price-of-spill channel" and critique-20 A1 calls the binding
mechanism. The flat rate captured only the (always-on) quantity-of-spill channel; the block curve adds
the price-of-spill channel back.

---

## 3. The increasing-block tariff as a MILP cost

### 3.1 Why this is convex and needs NO new binaries

An increasing-block (rising-marginal) tariff is a **convex** piecewise-linear cost in the quantity
shipped. Convex PWL costs in a **minimization** are modelled with **ordered continuous block-fill
variables and NO binaries / NO big-M** — the convexity itself guarantees the solver fills the cheapest
block first (filling an expensive block before a cheaper one is never optimal, so the model needs no
constraint to forbid it). This is the textbook "convex separable PWL = sum of bounded segments" result.

This is the critical tractability property and it is the **opposite** of the `min_flat_breaks` (GCR)
structure, which is *non-convex* (a higher break gives a *lower* rate → decreasing marginal → requires
the `γ` selector binaries and the big-M disaggregation that caused BLK-1c). The new lane block curve adds
**zero** integer variables and zero big-M rows. It cannot reintroduce BLK-1c.

### 3.2 Sets, variables, constraints, objective term

Let `L` be the set of **spot lanes** (defined in §4). For each lane `ℓ ∈ L` the calibration gives an
ordered block schedule `Bℓ = [(width₀, rate₀), (width₁, rate₁), …, (width_{n-1}, rate_{n-1})]` with
`rate₀ < rate₁ < … < rate_{n-1}` (strictly increasing marginal price) and finite widths. Let `Aℓ ⊆ A`
be the spot arcs that draw on lane ℓ's pool (§4).

**Decision variables (all continuous, ≥ 0):**

- `bℓᵢ` — chargeable-kg filled into block `i` of lane ℓ. Bounded `0 ≤ bℓᵢ ≤ widthℓᵢ`.

**Constraints:**

- **Block-fill conservation (replaces the per-arc cap C.5d):** the total chargeable weight routed on
  lane ℓ's spot arcs equals the total filled across its blocks.

  `Σᵢ bℓᵢ = Σ_{a∈Aℓ} (spot chargeable weight on a)`     (BLOCK-SUM[ℓ])

  where the RHS reads chargeable weight from whichever variable each arc's billing uses, exactly as the
  current `_build_spot_cap` does:
  - co-load (per-kg) arc `a`: `Σ_{k∈K_a} cw_k · x_{k,a}`
  - MAWB (flat / MFB) arc `a`: `Σ_{g∈G_a} CW_{a,g}`

- **Hard ceiling:** implicit and automatic — `Σᵢ widthℓᵢ = ceilingℓ`. Because each `bℓᵢ ≤ widthℓᵢ`, the
  block sum cannot exceed the lane ceiling. Demand that would exceed it has **no spot slot left** and
  must route via the fallback arc (which is unlimited, priced per §7's fallback multiple). The block-sum
  equality is what couples "lane is full" to "this kg pays fallback instead."

**Objective term (added to `_set_objective`):**

  `Σ_{ℓ∈L} Σᵢ rateℓᵢ · bℓᵢ`

This **replaces** the current flat spot objective contribution. Spot arcs no longer carry their own flat
`coload_per_kg`/`flat`/`mfb` rate; the per-kg spot price is delivered entirely through the lane block
ladder. (See §5 on what `RateCatalog` still carries per arc vs per lane, and §5 on the weight-break,
which stays per-shipment.)

### 3.3 Tractability argument (the BLK-1c gate)

1. **No binaries, no big-M.** `bℓᵢ` are continuous; the only structure is bound `bℓᵢ ≤ widthℓᵢ` and the
   linear equality BLOCK-SUM[ℓ]. The LP relaxation is exact for the block fill — convexity means the LP
   already orders cheap-first. This removes, not adds, integer structure relative to a per-arc cap.
2. **Column/row count is small.** One `bℓᵢ` per (lane, block); ~4 blocks per lane × (a handful of lanes
   per O–D × dest gateways) ≈ tens of new continuous columns and one equality row per lane. At proof
   scale this is < ~50 columns / < ~10 rows — negligible vs the ~1000-row MAWB/MFB structure.
3. **It strictly tightens, never loosens.** The block curve makes the LP relaxation *more* informative
   about cost (the cheap escape now has a rising shadow price), which typically *improves* the root gap,
   not worsens it. There is no fractional-cheat analogue of the MFB `γ→0` collapse, because there are no
   selector binaries to relax.
4. **What it replaces:** the per-arc cap `Σ cw_k·x ≤ cap^spot_a` (C.5d, `_build_spot_cap`) is **removed**
   and superseded by the per-lane BLOCK-SUM equality + block bounds. The per-arc cap was a single `≤`
   row per arc; the lane block curve is one `=` row per lane plus bounded columns. Row count is
   comparable; the new structure additionally **prices** the fill (the old cap only bounded it).

**Recommendation:** this is a proof-of-concept-first change. Before wiring the full sweep, build the
block-tariff on one lane and verify (a) the solver fills cheap-first without binaries, (b) the root gap
does not regress, (c) a hand-computed two-block instance bills exactly. (Per global rule 3 / project
"correctness before performance.")

---

## 4. Per-lane scarcity structure (the resource-sharing decision)

**Decision: the block pool is shared per-LANE, not per-arc/flight.** This matches the user's framing
("first 5,000 kg for a lane") and the calibration (§4: "a single origin→gateway lane offers … spot space
per week").

### 4.1 What "lane" means here

Define a **spot lane** `ℓ` as an **(origin-airport-group → dest-gateway) corridor** over a lane-week —
the granularity the sourced ceiling (~10–12k kg/lane-week) is quoted at. Concretely, key the lane by the
**dest gateway + origin region** (e.g. `PVG/TPE/HKG → LAX`), matching `_DEST_BASE_TRANSIT_H`'s
dest-gateway-dominant structure and the daily tiling. The lane pool is shared across:

- the multiple **daily flights** (`#d{day}` tiles) on that corridor, and
- the multiple **spot offers/arcs** (co-load, flat, MFB) that ride those flights.

**Justification for per-lane over per-arc:**

- **It is where scarcity is sourced.** The ~10–12k kg ceiling is a *lane-week* quantity in the
  calibration; splitting it per-arc (as today) is what produced the 48× over-supply.
- **It creates the competition the thesis needs.** When the pool is shared, cargo on different flights of
  the same corridor *compete* for the same cheap blocks — which is exactly the contention M₁ resolves and
  M₁' cannot. A per-arc pool gives each flight its own cheap block, so there is no cross-flight reshuffle
  value (the dominant replan channel).
- **It couples cleanly to the existing graph.** Spot arcs already carry an origin and dest; the lane key
  is a pure function of `(arc.origin_region, arc.dest)`. BLOCK-SUM[ℓ] sums the same per-arc chargeable-
  weight expressions `_build_spot_cap` already builds, just grouped by lane instead of capped per arc.

### 4.2 Coupling to the offer/graph structure — open question

The daily substrate tiles flights by day (`#d{day}`). **Decision needed (OPEN-1):** is the lane pool
**per lane-week** (one pool shared across all 7 days — strongest scarcity, matches the weekly sourcing)
or **per lane-day** (a fresh pool each day — weaker, but matches the daily replan cadence)? The
calibration is weekly; the replay loop is daily. Recommendation: **per lane-day** for the proof (a
per-day ceiling ≈ weekly/7, drawn independently per day), because (a) it matches the daily-cadence replay
clock the arms run on, (b) it keeps each cycle's contention local and interpretable, and (c) a weekly
pool would let day-1 cargo exhaust the whole week's cheap space, which over-states scarcity for a
7-shipment-per-day proof. Flag for user sign-off; it is a one-line keying choice (`(lane, day)` vs
`lane`).

---

## 5. Coexistence with the per-shipment IATA weight break

Both effects are real and simultaneous and must **not double-count**:

- **Per-shipment GCR weight break (`min_flat_breaks`, existing):** a *single heavier booking* gets a
  *lower* base $/kg (quantity discount within one MAWB). This is the existing non-convex break ladder.
- **Per-lane block curve (new):** the *lane's marginal kg* gets *more* expensive as the lane fills
  (scarcity across many bookings).

**Order of application (the no-double-count rule):**

1. The per-shipment weight break determines the **base chargeable weight / base rate** of a booking, via
   the existing MFB or flat billing — i.e. it sets *how much chargeable weight* and *at what base rate*
   the booking contributes. (Unchanged from today.)
2. The lane block curve prices the **lane's fill state**: each chargeable kg the booking adds to the lane
   pool is priced at the marginal block rate `rateℓᵢ` for the block it lands in.

**The clean separation that avoids double-counting:** the block ladder rates are expressed as
**multipliers on a single lane base spot rate** (`baseℓ ≈ $5.5/kg`), and the per-shipment weight break is
a discount on that *same base* — they act on orthogonal axes (one on the booking's own weight, one on the
lane's cumulative fill). Concretely (recommended, OPEN-2): **collapse the spot per-kg price entirely into
the lane block ladder** and let the per-shipment weight break apply as a *discount factor* on the
realized block rate. i.e. a heavy booking pays `(weight-break-factor) × rateℓᵢ` for the block it occupies.

This requires a modelling care point: the block ladder is convex (cheap-first) while the weight break is
non-convex (cheaper-when-heavier). Mixing a per-shipment multiplicative discount into a per-lane fill
variable `bℓᵢ` is not linear if the discount depends on which booking's kg landed in which block. **The
tractable resolution (recommended): keep the two strictly separate ledgers** —

- The **block ladder** prices the *lane scarcity premium* only: `rateℓᵢ` is the *premium multiplier over
  base* (`1.0×, 1.2×, 1.44×, 1.73×`), and the objective term is `Σ baseℓ · (multℓᵢ − 1) · bℓᵢ` — the
  **incremental** scarcity cost above base.
- The **base spot rate × weight break** stays on the existing per-arc/per-shipment billing path
  (`coload_per_kg` / `min_flat_breaks`), pricing the base $5.5/kg and its quantity discount exactly as
  today.

Then total spot cost = (base rate, weight-break-discounted, per booking) + (lane scarcity premium, per
block fill). No kg is priced twice: the base+discount is billed once per booking on its own cw; the
*premium above base* is billed once per kg of lane fill. **OPEN-2 for the user: confirm this
"base+discount on the booking, premium-above-base on the lane" split**, vs the simpler-but-less-faithful
"block ladder is the whole spot price, weight break dropped on spot." I recommend the split because it
preserves both sourced effects; flag that it is the more intricate of the two.

---

## 6. κ / supply-independence mapping (how tightness now reaches L2)

**Hard constraint preserved:** spot capacity must be drawn **independently of realized demand** (memory
`project_supply_independent_of_demand`, methodology §13 D-A18). The ceiling must come from a distribution,
**never** set as a multiple of this instance's demand. The supply/demand *mismatch* is the value source.

### 6.1 Two candidate mechanisms

**Mechanism A — demand-vs-finite-ceiling competition alone (κ stays contracted-only).** Keep the S38
design: κ scales only contracted ULD positions; spot ceiling/blocks are drawn on the κ-independent
`spot_regime` stream and held fixed across κ. The claim: simply making spot **finite** (not 144k kg, but
~10–12k kg/lane) means that as κ tightens contracted positions, more cargo spills onto a *finite rising*
spot curve, climbs into the expensive blocks, and eventually hits the ceiling → fallback. ∂L2/∂κ > 0
emerges purely because the spill now meets back-pressure.

- **Pro:** preserves the clean S38 CRN story — κ moves contracted tightness only; spot stays byte-stable
  across κ (no spot confound). Supply-independence is trivially intact (spot ceiling never reads demand
  or κ).
- **Con (critical, this is critique-18 F-A):** the *premium the spill pays* is then still a κ-independent
  flat band centered at `E[m]≈1.0`. The block ladder adds a *fill-dependent* premium, which does climb
  with spill volume — so ∂L2/∂κ > 0 can still hold via the *quantity* of spill pushing cargo into higher
  blocks. But the methodology §7 / backtest §7 explicitly mandate the spot regime *mean* be κ-tied
  (loose→soft, tight→peak). Mechanism A does **not** implement that.

**Mechanism B — κ also compresses the spot ceiling (the spec's intent).** In addition to scaling
contracted positions, let κ scale the **spot lane ceiling** (and/or shift the block schedule), drawn on a
distribution whose *parameters* depend on κ but whose *realization* never reads demand. E.g. draw the
lane ceiling as `ceilingℓ ~ Distribution(μ(κ), σ)` with `μ(κ)` a **decreasing** function of κ (tight κ →
smaller cheap-block widths / lower ceiling → more cargo forced up the curve and to fallback).

- **Pro:** implements the spec (κ binds both contracted *and* spot tightness); ∂L2/∂κ > 0 is structural
  and strong; resolves critique-18 F-A directly.
- **Con:** re-introduces a κ↔spot coupling *by design* (as F-A notes the spec intends). The CRN gate then
  protects **demand vs supply** only, not **spot byte-stability across κ** — varying κ now legitimately
  moves the spot draw. This is acceptable *if documented* (it is the methodology's own intent), but it
  means amending the S38 `spot_regime`-is-κ-independent decision and the C.5d tex note that says "the κ
  sweep moves contracted tightness only."

**Supply-independence under both:** the lane ceiling is drawn from a distribution on the `supply` (or a
new `spot_supply`) stream; its *parameters* may depend on κ (a knob, not demand) but its *value* is a
random draw that never reads `Σ SE_k` of the realized HAWBs. This is exactly how contracted `total_N`
already works (`round(n_hawbs · E[SE_k] / κ)` uses the **analytic** `E[SE_k]`, a closed-form constant —
zero demand draws). The spot ceiling must follow the same discipline: use an analytic/parametric mean,
not the realized book.

### 6.2 Recommendation

**Recommend Mechanism B**, because it (a) is what the approved methodology actually specifies, (b)
resolves the F-A spec-violation rather than papering over it, and (c) gives the strongest, most defensible
∂L2/∂κ > 0. But it **requires a formal methodology amendment** to D-A19 / backtest §7 acknowledging the
κ↔spot coupling and narrowing the CRN gate to demand-vs-supply. **OPEN-3 (the biggest sign-off): which
mechanism** — A (spot finite but κ-independent; minimal spec change; weaker, quantity-only κ binding) or
B (κ-tied spot ceiling; implements the spec; needs the D-A19/§7 amendment). I recommend B with the
amendment.

A defensible **middle path (OPEN-3b):** ship Mechanism A first (finite block curve, κ-independent
ceiling) to prove ∂L2/∂κ > 0 exists via the quantity channel and restore the loose-corner null — then,
once the structure is validated, add the κ-tied ceiling (B) as a documented second sweep arm. This stages
the change (rule 4: one component at a time) and isolates "did finiteness alone restore the binding" from
"does the κ-tie strengthen it."

---

## 7. Generator integration sketch

Changes are localized to `data/synthetic/air_generator.py` and `RateCatalog` / `_build_spot_cap`. No
change to demand generation, CRN stream separation, or the contracted draw.

### 7.1 Constants (replace the flat-band constants)

Retire / replace:
- `_SPOT_MULT_LO, _SPOT_MULT_HI = 0.85, 1.18` — the flat two-sided band. **Replaced** by the block
  ladder multipliers below (and, under Mechanism A, optionally a small base-level jitter).
- `_SPOT_CAP_ULD_LO, _SPOT_CAP_ULD_HI = 1.0, 3.0`, `_SPOT_CAP_ULD_CW_KG = 1500.0` — the per-arc cap.
  **Replaced** by the per-lane ceiling distribution.

Add (tied to the sourced calibration, §11):
- `_SPOT_BASE_USD_PER_KG = 5.5` — sourced normal-firm transpac base.
- `_SPOT_BLOCK_SCHEDULE` — ordered `[(width_frac, mult)]`: `[(5000, 1.00), (3000, 1.20), (2000, 1.44),
  (1250, 1.73)]` (widths in kg, mult on base). Step ~1.2×, last block ~1.73× (≈ bottom of fallback).
- `_SPOT_CEIL_MEAN_KG`, `_SPOT_CEIL_SPREAD` — lane-ceiling distribution params (~10–12k kg/lane-week; ÷7
  if per-lane-day, OPEN-1). Under Mechanism B, `_SPOT_CEIL_MEAN_KG` becomes a function `μ(κ)`.
- `_FALLBACK_SPOT_MULTIPLE = 2.5` — fallback at 2.5× base (≈ $13.75/kg). Feeds `FallbackPolicy` /
  `air_leg_cost_ub` so the route-based fallback dominates but is anchored to the sourced 2–4× band.

### 7.2 Functions

- **`_draw_spot_regime` → `_draw_spot_lanes`.** Today returns `arc → (mult, cap)`. **New:** returns a
  per-lane block schedule + ceiling: `lane → BlockSchedule(blocks=[(width, rate)], ceiling)`. Drawn in
  canonical (sorted lane) order for enumeration-independence (G2). Under Mechanism A it takes no κ (stays
  on `spot_regime`); under Mechanism B it takes κ and draws the ceiling from `μ(κ)` on the `supply`/
  `spot_supply` stream (and the CRN note changes accordingly — see §8).
- **`_build_rate_catalog`.** Stops writing `spot_cap[arc]`. Instead: (a) assigns each spot arc to its
  lane key `(origin_region, dest[, day])`; (b) populates the new `RateCatalog` lane-block fields; (c)
  under the §5 split, still writes the per-arc *base* `coload_per_kg` / `flat` / `mfb` rate (now
  weight-break only, no `× mult`), and lets the lane ladder carry the premium.
- **`_draw_network_supply`.** **Unchanged** (contracted draw stays exactly as is — this is the κ knob on
  contracted positions). Under Mechanism B, the *spot* ceiling draw is a sibling on the same stream
  discipline, not a change to this function.
- **`RateCatalog` new fields:**
  - `spot_lanes: dict[LaneKey, BlockSchedule]` — the per-lane block ladder + ceiling.
  - `spot_arc_lane: dict[ArcId, LaneKey]` — maps each spot arc to its lane pool (so the MILP can group
    `_build_spot_cap`'s per-arc chargeable-weight expressions by lane).
  - **Remove / deprecate** `spot_cap: dict[ArcId, float]` (superseded). Keep a back-compat shim only if
    isolation tests depend on it.
  - `BlockSchedule` dataclass: `blocks: tuple[(width_kg: float, rate_usd_per_kg: float), …]`,
    `ceiling_kg: float`.

### 7.3 MILP side (`src/components/air_milp.py`)

- **`_build_spot_cap` → `_build_spot_blocks`.** Reuses the existing per-arc chargeable-weight expression
  builders (co-load on `x`, MAWB on `CW`), but instead of one `≤ cap` row per arc, it (a) groups arcs by
  `rates.spot_arc_lane`, (b) adds `bℓᵢ` continuous vars bounded by block widths, (c) adds BLOCK-SUM[ℓ]
  equality, (d) returns the objective terms `Σᵢ rateℓᵢ · bℓᵢ` (or premium-above-base, per §5) to
  `_set_objective`.
- **`_set_objective`.** Adds the block terms; under the §5 split, the per-arc spot base rate term stays
  (weight-break path), and the lane premium term is added.
- **`air_leg_cost_ub` / `FallbackPolicy`.** The fallback per-kg should anchor to `_FALLBACK_SPOT_MULTIPLE
  × _SPOT_BASE_USD_PER_KG` as its top-spot-rate input, so the route-based 1.5× fallback dominates the
  most expensive *real* block (top block ~1.73× base) but sits at the sourced ~2.5× hand-off — keeping
  the "expensive-but-feasible, never pruned" backstop (memory `No-Standalone-Cost-Pruning`).

### 7.4 Determinism / CRN

- Lane block schedule + ceiling drawn in **sorted lane order** (G2).
- **Mechanism A:** keep the draw on the κ-independent `spot_regime` stream → spot stays byte-stable as κ
  sweeps; CRN gate unchanged.
- **Mechanism B:** move the **ceiling** draw to the `supply` (or a new `spot_supply`) stream and let its
  *parameters* depend on κ. The CRN gate then guarantees **demand byte-identical as κ/α sweep** (still
  true — demand never reads supply), but **spot is no longer κ-byte-stable** (intended). Update the
  determinism test + the §13/C.5d docs to say so. Block *multipliers* and *base* stay fixed (sourced), so
  only the ceiling moves with κ — the minimal coupling.

---

## 8. Expected L2(κ) shape & falsifiability

**New expected shape:**

- **Loose corner (small κ, abundant contracted + abundant spot ceiling, even α, early λ):** every arm
  fills the cheap B0 block and cheap contracted slots; nobody is forced up the curve; reshuffling buys
  nothing. **L2 ≈ 0** — the pre-registered `|L2| < CI` null is restored (it currently *fails*, mean $386
  at κ=8).
- **Tightening κ:** contracted positions shrink (both mechanisms) and/or the spot ceiling compresses
  (Mechanism B); demand competes for finite cheap blocks; M₁ keeps them for the cargo that benefits most,
  M₁' cannot. **L2 grows monotonically** — `∂L2/∂κ > 0`.
- **Very tight corner:** cheap blocks + ceiling exhausted, cargo hits fallback; L2 plateaus/peaks at the
  avoided fallback/expensive-block premium, then can flatten when even M₁ cannot avoid fallback (both
  arms saturated). A single-humped or saturating-increasing curve, **not the flat line measured today.**

**Success criterion (gates the headline, per critique-20 A1(c)):** demonstrate **∂L2/∂κ > 0 with a
confidence interval** across the κ ladder, AND the loose corner satisfies `|L2| < CI`. Both must hold on
the same swept surface. This makes the convexity/binding hypothesis falsifiable again — currently
untestable because the surface is flat.

**Interaction with the gap-noise floor (critique-20 A2):** L2 must clear the `mip_rel_gap` noise floor.
Restoring a real ∂L2/∂κ helps (peak-cell L2 should grow well past the ~$180 gap budget), but the headline
cells should still be re-solved at a tight gap (≤1e-4) per A2. This redesign makes A2 *easier* (bigger
signal) but does not by itself resolve it; note both as a paired fix.

**Methodology / spec reconciliation required:**
- **C.5d in `model/air_freight_routing.tex`** (lines 3062–3081): rewrite from "per-arc chargeable-weight
  cap" to "per-lane increasing-block tariff + ceiling." The new subsection should state the convex-PWL /
  no-binary formulation and that the per-arc cap is withdrawn.
- **§13 D-A19** (`arrival_only_replan_methodology.md` lines 327–340): amend the spot row from "drawn cap
  at rate base×m" to the block ladder + ceiling; under Mechanism B, amend the κ-tie and narrow the CRN
  three-stream note (spot no longer byte-stable across κ).
- **Supply-independence memory / §13 D-A18:** explicitly affirm the spot ceiling is drawn from a
  distribution (parametric mean), never from realized demand — same discipline as contracted `total_N`'s
  analytic `E[SE_k]`. Add a test mirroring the contracted one (vary κ/α ⇒ demand byte-identical; under
  Mechanism B, spot ceiling *may* move with κ but never with the demand realization).
- **backtest §7** "spot-vs-contract gap tied to κ": Mechanism B satisfies it directly; Mechanism A does
  not and needs §7 amended to "spot gap fixed/sourced; κ binds via finite ceiling quantity channel only."

---

## 9. Open decisions needing user sign-off

| # | Decision | Recommendation | Why it matters |
|---|---|---|---|
| **OPEN-1** | Lane pool granularity: per **lane-week** vs per **lane-day** | per **lane-day** (ceiling ≈ weekly/7, drawn per day) | matches daily replay cadence; weekly pool over-states scarcity for a 7/day proof |
| **OPEN-2** | Weight-break ↔ block-curve composition: "base+discount on booking, premium-above-base on lane" split vs "block ladder is whole spot price, drop weight break on spot" | the **split** (preserves both sourced effects) | the split is faithful but more intricate; avoids double-counting either way |
| **OPEN-3** | κ-binding mechanism: **A** (finite, κ-independent ceiling) vs **B** (κ-tied ceiling, implements spec) | **B with D-A19/§7 amendment**, or **staged A→B** (3b) | B resolves critique-18 F-A; A needs a §7 amendment instead; the single biggest call |
| **OPEN-3b** | If staged: ship A first to prove ∂L2/∂κ>0 + restore null, then add B as a second arm | yes (rule 4, one change at a time) | isolates "finiteness alone" from "κ-tie strengthens it" |
| **OPEN-4** | Block schedule values: confirm the §11 parameter table (widths, ~1.2× step, ceiling, 2.5× fallback) as the sourced anchors | adopt §11 as the `[CAL]` defaults, all labelled INFERRED-from-sourced | these are the dollar magnitudes; all tie to the calibration ledger |
| **OPEN-5** | RNG stream for the Mechanism-B ceiling draw: reuse `supply` vs new `spot_supply` | new `spot_supply` stream (cleanest CRN separation) | keeps contracted and spot supply draws independently swappable |
| **OPEN-6** | Pair this with the A2 tight-gap fix (re-solve headline at ≤1e-4)? | yes — note as a paired fix | bigger L2 signal helps but doesn't alone clear the gap floor |

---

## 10. Parameter table (tied to the sourced calibration)

All values from `docs/design/air_pricing_calibration_s45.md`; every row labelled by confidence
(SOURCED / INFERRED-from-sourced). Base = **$5.5/chargeable-kg** (Xeneta NE-Asia→NA Apr-26 $5.54).

### 10.1 Spot lane block ladder (per lane-week; ÷7 if per-lane-day, OPEN-1)

| Block | Width (kg) | Mult ×base | Marginal $/kg | Confidence |
|---|---|---|---|---|
| B0 | 5,000 | 1.00× | $5.50 | base SOURCED; width INFERRED |
| B1 | 3,000 | 1.20× | $6.60 | step INFERRED (range 1.15–1.25×) |
| B2 | 2,000 | 1.44× | $7.92 | INFERRED (1.2²) |
| B3 | 1,250 | 1.73× | $9.52 | INFERRED (1.2³) |
| **Ceiling** | **~11,250** (Σ widths) | — | beyond → fallback | INFERRED ~10–12k/lane-wk |

### 10.2 Contracted (unchanged mechanism, levels for reference)

| Param | Value | Confidence | Notes |
|---|---|---|---|
| Contracted base rate `r_a` | ~$4.2/kg | INFERRED (~0.76× spot) | currently `U(2.5,4.0)`; re-anchor toward ~4.2 (OPEN, minor) |
| Positions/flight | ~1–4 ULD (LD3) | INFERRED; per-forwarder MRN | drawn via `total_N = round(n·E[SE_k]/κ)` — unchanged |
| Pivot | ~1,000–1,500 kg | SOURCED mechanics | LD3 ≈1,500 kg chargeable |

### 10.3 Fallback / must-ship-now

| Param | Value | Confidence | Notes |
|---|---|---|---|
| Fallback multiple | **2.5× base** (~$13.75/kg) | INFERRED; 2–4× band SOURCED | feeds `air_leg_cost_ub`; route-based 1.5× wrap dominates top block |

### 10.4 κ-tie (Mechanism B only, OPEN-3)

| Param | Form | Confidence | Notes |
|---|---|---|---|
| Lane ceiling mean `μ(κ)` | decreasing in κ (e.g. `μ₀/κ^β`, β∈(0,1]) | INFERRED — `[CAL]`, needs user pick | tight κ → smaller cheap pool; drawn on `spot_supply`, never reads demand |
| Step multiplier | fixed 1.2× (κ-independent) | SOURCED-ish | only ceiling moves with κ; minimal coupling |

### 10.5 Volumetric

| Param | Value | Confidence |
|---|---|---|
| Volumetric divisor | 167 kg/m³ (IATA 1:6) | SOURCED — confirmed correct, no change |

---

**Bottom line for sign-off:** approve (a) the convex increasing-block per-lane tariff (no new binaries,
BLK-1c-safe), (b) per-lane-day pooling (OPEN-1), (c) the weight-break/block-curve split (OPEN-2), (d) the
κ-binding mechanism (OPEN-3 — recommend B with a D-A19/§7 amendment, or staged A→B), and (e) the §11
parameter anchors. On approval, the .tex C.5d section and §13 D-A18/D-A19 are amended, and implementation
proceeds one lane first (POC), then the sweep.
