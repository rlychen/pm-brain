# S46 Capacity Redesign — Supply/Demand Architecture (air slice)

**Status:** DESIGN PROPOSAL — behind the formal-model approval gate. No `src/` / `tests/` / `data/`
code is written until the user approves. This is the architecture document only.

**Author role:** Supply–Demand Architect. Consumes the S45 calibration / composition / redesign
docs, the `air_milp_m4_bsa_schema_options.md` BSA model, methodology §13, and the live
`air_generator.py` + `air_milp.py`. Every rate/capacity number traces to
`docs/design/air_pricing_calibration_s45.md` or is labelled **INFERRED** / **MRN**.

**What problem this solves (the S45 killer finding).** The replan headline `L2 = C(M₁') − C(M₁)`
was measured to be **100% consolidation reshuffle, 0% capacity** (`l2_decomposition_s45.md`):
total spot kg is byte-identical between arms, contracted ULD positions are **never used**, fallback
is **never touched**. The cause is structural, not a tuning miss: spot is flat-priced and
effectively unbounded (~150k kg available vs a ~7k kg book = the 48× over-supply), and contracted
is **more expensive than spot** at the drawn levels, so the optimizer rides spot for everything and
the capacity dimension is inert. This redesign makes capacity **finite, tiered, and binding** so
that genuine fallback, genuine roll/tardiness, and genuine contracted↔spot competition exist —
which is the precondition for the three metrics (cost savings / OTP / fallback incidence) to be
non-vacuous.

---

## A. Capacity-type inventory

Common chargeable-weight reference: **LD3 = 1,500 kg chargeable / 4.5 cbm** (SOURCED, AKE max-gross
1,588 kg); chargeable wt = `max(actual, vol·167)` (SOURCED, IATA 1:6). All `$/kg` are per
**chargeable** kg. "MILP support" is the status in `air_milp.py` today.

| # | Type | What it is | Rate structure (sourced anchor) | Capacity unit | Billing rule | MILP support |
|---|---|---|---|---|---|---|
| 1 | **Soft BSA** (`per_flight`) | Negotiated contracted rate + per-ULD pivot minimum; cancellable, nothing sunk, pay-only-if-used | **~$4.2/kg** contract anchor (INFERRED ~0.76× spot); currently `r_a ~ U(2.5,4.0)` | integer **ULD positions** `N_f` per flight (LD3) | `r_a · max(CW, π·Ση)` per MAWB; pivot `π ~ U(1000,1500)` kg (SOURCED mechanics) | **HAVE** — generated today (`settlement="per_flight"` hardcoded), C.5/C.5b/C.13b |
| 2 | **Hard BSA** (`equalized`, take-or-pay) | Pre-paid sunk allowance `A_c`; marginal cost 0 up to `A_c`, overage at `r_c`; pooled across the contract's arcs | `A_c ≈ (positions)·π`; `r_c ≈` contract overage ~$4.2/kg (INFERRED); levels MRN | sunk **allowance** `A_c` (kg), pooled cross-arc | `r_c · max(0, Σ_arc CW − A_c)`; 0 per-MAWB | **BUILT-NOT-GENERATED** — `_build_c13a_equalized`, `over_c`, `allowance_kg`, `r_c` all exist; generator never emits `settlement="equalized"`. **Wire in (B).** |
| 3 | **Spot — flat / co-load / MFB** (current) | Free-sale market space, one per-kg rate per arc, per-arc CW cap | base **$5.5/kg** (SOURCED Xeneta NE-Asia→NA Apr-26 $5.54); co-load `U(3.5,5.5)`, flat `U(3.0,5.0)`, MFB ladder | per-arc **CW cap** (drawn `U(1,3)·1500 kg`) | per-kg on `cw_k` (co-load) or density-mixed `CW_{a,g}` (flat/MFB); C.5d cap | **HAVE** — but the inert structure the redesign replaces |
| 4 | **Spot — increasing-block lane curve** | Finite per-LANE spot pool that clears cheap-first then steps up; convex PWL, no binaries | base $5.5/kg × `[1.00, 1.20, 1.44, 1.73]` over widths `[5000, 3000, 2000, 1250]` kg/lane-wk; ceiling ~11,250 kg (all INFERRED-from-sourced, §4 calibration) | per-**lane(-day)** block ladder + **hard ceiling** | `Σ_i rate_ℓᵢ·b_ℓᵢ`, `b_ℓᵢ ≤ width_ℓᵢ`; lane fill `= Σ spot CW on lane` | **NEW-CONSTRAINT** — the shelved `air_pricing_capacity_redesign_s45.md` design; continuous block vars + one equality/lane (BLK-1c-safe, no binaries). Replaces C.5d. |
| 5 | **Co-load** (per-kg consolidator space) | Buy space inside another forwarder's MAWB at a per-kg rate; no ULD commitment | co-load `U(3.5,5.5)·mult` $/kg | per-arc CW cap (→ folds into lane curve, type 4) | `Σ m^cl·cw_k·x` | **HAVE** — `coload_per_kg`; treat as a spot sub-channel feeding the lane pool |
| 6 | **Fallback — marked-up scheduled** | Last-minute "must-ship-now" scheduled space once contract+spot exhausted | **2.5× base** (~$13.75/kg; 2–4× band SOURCED, 2.5× point INFERRED) | unlimited (relief valve) | route-based `1.5 × worst-spot-route` wrap, per-HAWB | **HAVE** — `ArcType.FALLBACK`, `FallbackPolicy`, `air_leg_cost_ub` |
| 7 | **True charter / ad-hoc** | Whole-aircraft or block charter; priced **per-operation**, not per-kg | "per-operation, significantly more expensive" (SOURCED); per-kg only rational for a large block; level MRN | large **fixed cost** per charter event, with a kg capacity | high fixed charge + capacity, invoked when consolidating a large stranded block | **NEW-CONSTRAINT** — distinct from #6's per-kg markup. A fixed-charge capacitated option (binary `use_charter` + capacity). Minimal: model only if metric 3 needs a *cheaper-than-rolling-everything* large-block escape. **Recommend DEFER to v2** (see H). |
| 8 | **Belly vs freighter split** | Passenger-belly capacity (smaller per-flight, often co-load/spot) vs main-deck freighter (ULD positions, BSA) | belly ~66% of total capacity (SOURCED); belly typically co-load/spot, freighter carries BSA ULD blocks | per-flight kg (belly) vs ULD positions (freighter) | belly → spot/co-load rate families; freighter → ULD-pivot/BSA | **Generator/topology NEW; MILP HAVE** — no new constraint: belly = spot-family arcs with smaller caps, freighter = `per_uld_pivot` arcs. The split is a **schedule/offer** property, not a new optimizer object. |
| 9 | **Deferred-air as a supply product** | A cheaper, slower carrier *service tier* (e.g. consolidated/economy air) offered as a routable arc — not just a demand SLA tier | discount to base spot (INFERRED; level MRN); slower transit leg | per-flight kg, spot-family rate | per-kg at a discount on a slower-transit arc | **Generator/topology NEW; MILP HAVE** — a slower-block, lower-rate spot arc. Couples to OTP: cheap but eats deadline slack. No new constraint; needs slower legs in the schedule substrate. |

**Justification / minimal-design audit** (each type earns its place against the three metrics):

- **#1, #2 (soft + hard BSA)** are the contracted tier whose *binding* is the whole point — without
  a contracted tier that is *cheaper-and-scarce*, there is no contracted↔spot reshuffle margin and
  L2 stays the consolidation artifact (`l2_decomposition_s45.md` VERDICT). Hard BSA (#2) adds the
  take-or-pay sunk-cost dynamic that makes "fill the allowance you already paid for" a real
  incentive — the cheap-and-scarce capacity the thesis needs. **Both required.**
- **#3 → #4** is a *replacement*, not an addition: the flat per-arc cap is retired for the lane
  block curve. Keeps the model minimal (one mechanism, not two).
- **#6 (fallback)** is required for metric 3 (no-feasible-route incidence) and metric 2 (roll →
  tardiness). **Required.**
- **#7 (true charter)** is the only genuinely *optional* addition. It changes the cost of the tight
  tail but the three metrics work without it (fallback #6 already provides the feasible-but-expensive
  backstop). **Recommend defer** unless the user wants the charter-vs-roll tradeoff measured.
- **#8 (belly/freighter)** is required to make capacity *heterogeneous per flight* — it is what
  lets a lane be short on freighter ULD blocks while belly co-load is slack, which is realistic and
  creates the within-lane competition. Cheap to add (topology only). **Recommended.**
- **#9 (deferred-air supply)** is what makes metric 2 (OTP/tardiness) have a *cost lever*: a cheaper
  slow arc that the optimizer takes when slack allows and avoids when the deadline is tight. Without
  it, OTP is purely a roll-driven phenomenon. **Recommended, low cost.**

---

## B. Supply generator design

All supply draws stay on the **`supply` / `spot_supply` RNG sub-streams, never reading the demand
draw** (methodology §13 D-A18; memory `project_supply_independent_of_demand`). The discipline is:
a supply quantity's *distribution parameters* may depend on the tightness dial (a knob), but its
*realized value* is a random draw that never reads `Σ SE_k` of the realized HAWBs — exactly how
`total_N = round(n·E[SE_k]/κ)` already uses the **analytic** `E[SE_k]`.

### B.1 Per-type quantity draws (per flight / lane / lane-week)

| Type | Drawn at | Quantity | Scales with dial |
|---|---|---|---|
| Soft/Hard BSA (#1/#2) | per **flight** (freighter arcs) | integer ULD positions `N_f` via `Multinomial(N_ℓ, Dirichlet(α))` over the lane's freighter flights | lane allowance `N_ℓ = round(τ_ℓ^{-1} · D_ℓ^{kg} · s_BSA / 1500)`; tight lane → more positions? **No — see note.** |
| Spot block curve (#4) | per **lane(-day)** | block ladder `[(width, rate)]` + ceiling | ceiling `= τ_ℓ^{-1}? ` — **the dial moves the cheap-block width / ceiling**; see §D |
| Belly co-load (#8) | per **flight** (belly arcs) | per-flight CW cap (smaller than freighter) | folds into the lane spot pool ceiling |
| Deferred-air (#9) | per **lane** | one slow discounted arc/flight | fixed share of lane capacity |
| Fallback (#6) | per **HAWB** at build | unlimited, 2.5× base wrap | none (relief valve) |
| Charter (#7, if built) | per **lane-day** | one fixed-charge capacitated option | none |

**Note on direction (important).** "Tight lane" means **capacity < demand**, i.e. the dial should
make the lane's *total* capacity *small relative to its expected demand*. Concretely, for a target
lane tightness `τ_ℓ = capacity/demand`, the **total** lane capacity (all types) is
`S_ℓ = τ_ℓ · D_ℓ` and is then **split across types by a fixed composition vector** (B.2). A short
lane (`τ_ℓ < 1`) therefore gets *fewer* ULD positions AND a *lower* spot ceiling — both tiers shrink
together, which is what forces cargo up the curve and into fallback.

### B.2 Type composition vector (how `S_ℓ` splits across tiers)

The dense transpac headhaul is **~80%+ contract/allotment-locked** (HKG→US <20% spot, SOURCED
`air_spot_composition_research_s45.md`). So the default composition for a dense lane:

| Tier | Share of `S_ℓ` (kg-equiv) | Confidence |
|---|---|---|
| Contracted (BSA #1/#2, freighter ULD) | **0.70** | INFERRED from <20% spot share (SOURCED) |
| Spot block curve (#4, incl. belly co-load #8) | **0.22** | INFERRED |
| Deferred-air (#9) | **0.05** | INFERRED / MRN |
| Reserve headroom (unsold) | **0.03** | INFERRED |

(Thin-lane variant — e.g. an ex-secondary origin — would invert toward ~80% spot; expose as a
per-lane composition override, not the transpac default. Memory `reference_air_spot_contract_ratio`.)

Within contracted, split #1/#2 by a **`hard_bsa_frac`** knob (default 0.5): half the contracted
positions are pooled into a `per_flight` (soft) contract, half into an `equalized` (hard) contract
with `A_c = (its positions)·π`. This is the wiring the proof has never exercised.

### B.3 Hard-BSA (`equalized`) wiring — the BUILT-NOT-GENERATED fix

`_build_rate_catalog` currently emits a single `BsaContract(settlement="per_flight", …)`. Change to
emit **two** contracts over disjoint subsets of the contracted arcs:

```
soft = BsaContract(id="bsa-soft", settlement="per_flight",
                   arcs=soft_arcs, allotment={a: {"LD3": N_a} …},
                   pivot=π, r_a≈4.2)
hard = BsaContract(id="bsa-hard", settlement="equalized",
                   arcs=hard_arcs, allotment={a: {"LD3": N_a} …},
                   allowance_kg=A_c, r_c≈4.2)     # A_c = Σ_{a∈hard} N_a · π
```

`A_c` is drawn on the `supply` stream from the *expected* contracted block (analytic), never the
realized book. The MILP already prices both (`_build_c13b_pivot`, `_build_c13a_equalized`,
objective lines 1057–1061) — zero MILP change for hard BSA itself. Re-anchor `r_a`/`r_c` toward
**~$4.2/kg** (the sourced contract anchor) so contracted sits **below** base spot $5.5/kg — without
this, contracted is never worth filling (the S45 root cause).

### B.4 Belly/freighter + deferred-air (topology, in `tpeb_air_instance`)

These are **schedule-substrate** changes, not generator-math changes:

- Tag each flight `deck ∈ {belly, freighter}`. Freighter flights carry `per_uld_pivot` (BSA) arcs;
  belly flights carry co-load/spot arcs with **smaller** per-flight caps (belly ≈ 66% of capacity
  but spread thin and mostly non-ULD). The optimizer sees them as the existing rate families on
  distinct arcs — **no new constraint**.
- Add one **deferred-air** arc per lane: a slower transit leg (longer `arr−dep`) at a discounted
  spot rate. It competes on cost vs deadline slack — the OTP cost lever.

---

## C. Demand generator design

Demand generation is **unchanged in shape** from `_gen_arrivals` (tier-coupled arrival stream,
`Δ_k = ready + base_transit(lane) + sla_offset(tier)`, CRN on the `demand` stream). The redesign
touches **only the scale and the lane-assignment**, so demand stays byte-identical as the tightness
dial sweeps (the hard CRN gate).

**Sizing demand to exceed supply.** The dial `τ` is defined (D) as `Σ capacity / Σ demand`, so
demand-exceeds-supply is `τ < 1` *by construction*. The generator does **not** shrink supply to
meet demand or vice-versa; it draws demand at the configured `n_hawbs`, computes **analytic expected
demand** `D = n_hawbs · E[cw_k]`, and sizes total supply `S = τ · D` (then splits per B). With
`E[cw_k] = E[w]·E[max(1, 167/density)]` (`w ~ triangular(50,1200,300)` → `E[w]=516.7`;
`density ~ U(120,240)`), this is a closed-form constant — zero demand draws, so CRN holds.

**Per-lane short/balanced/slack — the realized lane book.** Each HAWB's lane is an **optimizer
decision** under region→region routing (D-A24): the HAWB lands on whichever (origin airport, dest
gateway) the cost/capacity favours. So the generator cannot *assign* a HAWB to a lane. Instead it
controls **expected lane demand share** `q_ℓ` via the door-coordinate bounding boxes and the lane's
base-transit attractiveness, giving analytic `D_ℓ = n_hawbs · q_ℓ · E[cw_k]`. Per-lane tightness is
then set on the **supply** side: `S_ℓ = τ_ℓ · D_ℓ` with `τ_ℓ` drawn per the mix mechanism (D). This
is the clean separation — **demand realization is never touched by the dial; only the supply mean
per lane is.**

**Why this still produces genuine short/slack (not a deterministic structure the solver exploits).**
`τ_ℓ` sets the *expected* tightness, but the *realized* tightness fluctuates because (i) which HAWBs
actually land on lane ℓ is a demand+routing draw, (ii) their actual weights/densities/arrival times
are random, and (iii) the supply realization (`Multinomial` ULD spread, block-ceiling draw) is random
around its `τ_ℓ`-set mean. A lane with `τ_ℓ = 1.0` (balanced in expectation) still realizes short on
some seeds and slack on others. The dial makes the *distribution* of short/balanced/slack
reproducible; the *realization* preserves the mismatch that is the value source.

---

## D. The global tightness parameter

### D.1 Formal definition

Let all capacity and demand be expressed in a **common unit: chargeable-kg** (ULD position →
1,500 kg; spot blocks already kg; belly/deferred in kg). Define the **global tightness dial**

```
        Σ_ℓ Σ_types  capacity(type, ℓ)        S
  τ  =  ───────────────────────────────   =   ─
            Σ_ℓ  D_ℓ                            D
```

where `D = n_hawbs · E[cw_k]` is **analytic expected demand** (closed-form, zero demand draws).
`τ < 1` ⇒ demand exceeds supply overall (the user's firm requirement). Recommended sweep ladder
`τ ∈ {0.7, 0.9, 1.1}` (tight / balanced / slack network-wide).

### D.2 Mapping `τ` → per-type quantities

1. Total supply `S = τ · D`.
2. Per-lane expected demand `D_ℓ = q_ℓ · D` (`q_ℓ` from door-box geometry, analytic).
3. Per-lane target tightness `τ_ℓ` drawn by the mix mechanism (D.3), **normalized so the
   demand-weighted mean returns `τ`**: `Σ_ℓ q_ℓ · τ_ℓ = τ` (a one-line rescale of the band centers).
4. Per-lane total capacity `S_ℓ = τ_ℓ · D_ℓ`.
5. Split `S_ℓ` across types by the composition vector (B.2): contracted ULD positions
   `N_ℓ = round(S_ℓ · 0.70 / 1500)`, spot ceiling `= S_ℓ · 0.22`, deferred `= S_ℓ · 0.05`.
6. Spread `N_ℓ` over the lane's freighter flights via `Multinomial(N_ℓ, Dirichlet(α))`; split the
   spot ceiling into the block ladder widths `[5000,3000,2000,1250]` scaled to the lane ceiling.

This makes `τ` the single dial the user asked for: one number tunes total-capacity-across-all-types
vs total-demand, and it deterministically maps to every per-type quantity.

### D.3 Per-lane short/balanced/slack mix mechanism

**Recommended: explicit bucket assignment with within-bucket jitter** (reproducible + controllable,
the user's stated want), over a purely random per-lane multiplier (less controllable).

- User specifies a **composition** `(n_short, n_balanced, n_slack)` over the lane set (e.g. 2/2/2
  on the 6-lane grid).
- Each bucket has a tightness band: **short** `τ_ℓ ∈ [0.6, 0.85]`, **balanced** `[0.95, 1.05]`,
  **slack** `[1.2, 1.6]` (INFERRED bands; the short band must dip far enough below 1 to exhaust
  contracted+cheap-spot and force fallback — see metric requirement).
- Draw `τ_ℓ` uniform within its band on the `spot_supply` stream, then apply the D.2-step-3
  normalization so the network mean is exactly `τ`.

The bucket *assignment* to specific lanes is fixed by seed (reproducible); the *within-band* draw
plus the demand/supply realization give the randomness. Short lanes reliably produce fallback and
tardiness; slack lanes reliably stay on the cheap block — so metrics 2 and 3 are non-vacuous **by
construction of the band placement**, while no lane is deterministic.

### D.4 Reconciling the central tension (supply-independence vs controlled tightness)

These pull against each other only if "supply independent of demand" is read as "supply
distribution must not depend on a tightness target." It does not. The **operative** invariant is:
*supply's realized value must not be a function of the realized HAWB book.* The current code already
embodies the resolution — `total_N` uses analytic `E[SE_k]` and analytic expected demand, never the
realized draw. We generalize exactly that: `τ_ℓ` sets a **mean** against **analytic** expected lane
demand; the **realization** (which HAWBs land where, their weights, the Multinomial/ceiling draws)
is untouched by the dial. So:

- **Controllability** ✓ — `τ` and the bucket composition reproducibly place short/balanced/slack.
- **Supply-independence** ✓ — no supply quantity reads the realized demand; CRN gate (vary `τ`/α ⇒
  demand byte-identical) holds unchanged.
- **Mismatch-is-value** ✓ — realized per-lane tightness still fluctuates around `τ_ℓ`, so the
  open-book reshuffle still has genuine mismatch to exploit.

---

## E. Relationship to current κ / α

**Recommendation: GENERALIZE κ → per-lane `τ_ℓ` vector; KEEP α unchanged.**

- **κ (current):** a single global scalar, `total_N = round(n·E[SE_k]/κ)`, network-wide contracted
  tightness only. It is exactly the **special case `τ_ℓ = κ⁻¹·(const)` for all ℓ** — uniform
  tightness, contracted-tier-only. The new `τ` + per-lane `τ_ℓ` vector is the strict generalization:
  it (a) spans *all* capacity types (not just contracted), (b) allows *per-lane* targets (the
  short/slack mix κ cannot express), and (c) reduces to κ when the bucket composition is "all
  balanced" and only the contracted tier scales. So κ is **subsumed**, not discarded — keep the
  `κ`-style sweep as the *uniform-τ* arm for continuity with the S38–S45 results.
- **α (Dirichlet concentration):** unchanged in role. It now governs the spread of `N_ℓ` ULD
  positions *across the flights within a lane* (was: across all network flights). Low α = lumpy
  (some flights in a short lane get 0 positions → severe local scarcity even when the lane mean is
  balanced); high α = even. α is the **within-lane** realization-noise knob; `τ_ℓ` is the
  **between-lane** mean knob. They are orthogonal and both wanted. **Keep α.**

Reasoning for generalize-not-replace: the methodology §13 machinery (analytic `E[SE_k]`, the
`supply`-stream discipline, the frozen-capacity-vector-across-arms invariant) is all reusable
verbatim; `τ_ℓ` slots in as the parameter that *sets the mean* those draws are taken around. A
clean rewrite would throw away the CRN/independence scaffolding that already works.

---

## F. Proposed instance set (ladder)

The scale tension is real: **filling lanes needs forwarder scale** (100s of HAWBs to actually
exhaust a ~11k kg/lane-week ceiling), but binary count (`x` per HAWB-arc, `z` per MAWB-group) scales
with HAWB count under region→region multi-O/D subgraphs. A separate **Tractability agent** will
stress-test the top of this ladder; the new capacity machinery itself is cheap (block vars add ~4
continuous + 1 equality per lane; hard BSA adds 1 `over_c` per contract — both negligible vs the
MAWB binary structure). So the ladder climbs HAWB count, not capacity complexity.

| Cell | Purpose | Lanes | Mix (short/bal/slack) | n_hawbs | days | seeds | τ ladder | Rough MILP size |
|---|---|---|---|---|---|---|---|---|
| **C0 — mechanism-proof** | Verify each capacity type bills correctly; block curve fills cheap-first; hard BSA `A_c` sunk; one short lane forces 1 fallback | 2 | 1 / 0 / 1 | 12 | 2 | 1 | {0.8} | ~10²–10³ bin vars (current proof scale) |
| **C1 — mix-proof** | Verify short→fallback+tardiness, slack→cheap-block, balanced→contract-binds; L2 capacity-bearing | 4 | 1 / 2 / 1 | 40 | 4 | 3 | {0.9} | ~10³–10⁴ bin vars |
| **C2 — fill-scale** | Lanes actually consumed/short at forwarder volume; metrics non-vacuous at scale; ∂L2/∂τ measurable | 6 | 2 / 2 / 2 | 120 | 5 | 5 | {0.7, 0.9, 1.1} | ~10⁴–10⁵ bin vars — **Tractability gate** |
| **C3 — stress (optional)** | Upper bound for the Tractability agent; peak regime | 6 | 3 / 1 / 2 | 200 | 7 | 5 | {0.7} | **flag — may need decomposition** |

Rationale:

- **C0** is the POC the project's "correctness before performance" rule mandates: build the block
  tariff on one lane, the hard-BSA wiring, and hand-verify a two-block bill before any sweep
  (`air_pricing_capacity_redesign_s45.md` §3.3, global rule 3).
- **C1** proves the *mix* mechanism: the three buckets behave as designed on a small-but-non-trivial
  grid, and L2 now has a capacity component (the S45 finding's fix is visible).
- **C2** is the headline scale where ~120 HAWBs over 5 days × 6 lanes can plausibly exhaust an
  ~11k kg/lane-week ceiling and produce real roll/fallback — the regime where the three metrics are
  meaningful. Spans the τ regimes (tight/balanced/slack) to show ∂(metrics)/∂τ.
- **C3** is named only as the Tractability agent's stress ceiling; do not run for results until C2
  solves within the gap-policy budget (memory `project_tractability_gap_policy`: solve to
  `mip_rel_gap=0.005`, threads=1/solve).

**Note for the Tractability agent:** the binary driver is HAWB count × subgraph arc count under
D-A24 region→region routing, *not* the capacity redesign. The redesign's net effect on the LP is
*tightening* (the block curve gives the cheap escape a rising shadow price), which typically
*improves* the root gap. The open risk is solely the HAWB-count scaling of `x`/`z`.

---

## G. Build delta

### Generator-only changes (`data/synthetic/air_generator.py`, `tpeb_air_instance.py`)

1. **Replace κ scalar with `τ` + per-lane `τ_ℓ` vector** in `GenConfig`/`ArrivalConfig`
   (`tau`, `lane_mix=(n_short,n_bal,n_slack)`, keep `alpha`, keep `kappa` as the uniform-τ shim).
2. **New `_size_total_supply`**: `S = τ · n·E[cw_k]` (analytic `E[cw_k]` helper, mirrors
   `_expected_slot_mean`).
3. **New `_draw_lane_tightness`**: bucket-assign lanes, draw `τ_ℓ` within band, normalize to `τ`
   (on `spot_supply` stream).
4. **Generalize `_draw_network_supply`**: per-lane `N_ℓ = round(S_ℓ·0.70/1500)`, then
   `Multinomial(N_ℓ, Dirichlet(α))` over that lane's freighter flights.
5. **Emit hard BSA**: split contracted arcs into soft (`per_flight`) + hard (`equalized`, `A_c`,
   `r_c`) contracts via `hard_bsa_frac`; re-anchor `r_a`/`r_c` ≈ $4.2/kg (below spot $5.5).
6. **Replace `_draw_spot_regime` → `_draw_spot_lanes`**: per-lane block ladder + ceiling (the
   shelved §7.2 design); retire `_SPOT_MULT_*`, `_SPOT_CAP_ULD_*`.
7. **`RateCatalog` new fields**: `spot_lanes: dict[LaneKey, BlockSchedule]`,
   `spot_arc_lane: dict[ArcId, LaneKey]`; deprecate `spot_cap`.
8. **Topology (`tpeb_air_instance`)**: tag flights `deck ∈ {belly, freighter}`; add one
   deferred-air slow-discount arc per lane; (charter option deferred).
9. **New RNG stream** `spot_supply` for the lane-ceiling/τ_ℓ draws (cleanest CRN separation).

### Optimizer changes (`src/components/air_milp.py`)

10. **`_build_spot_cap` → `_build_spot_blocks`** (NEW): group spot arcs by `spot_arc_lane`, add
    continuous `b_ℓᵢ ≤ width_ℓᵢ`, the BLOCK-SUM equality `Σ b_ℓᵢ = Σ spot CW on lane`, and return
    objective terms `Σ rate_ℓᵢ·b_ℓᵢ`. **No binaries, no big-M** (convex PWL). Replaces C.5d.
11. **Hard BSA: NO change** — `_build_c13a_equalized` / objective already price `equalized`. Only
    confirm the generator now feeds two contracts (add an isolation test).
12. **Belly/freighter, deferred-air: NO new constraint** — they arrive as existing rate families on
    new arcs.
13. **(If charter built, #7): NEW** — `use_charter` binary + capacity + fixed charge. **Deferred.**
14. **`air_leg_cost_ub` / `FallbackPolicy`**: anchor the per-kg fallback to
    `2.5 × $5.5/kg` so it dominates the top real block (1.73× base) but sits at the sourced 2.5×
    hand-off (no-standalone-cost-pruning preserved).

### Scorer changes (replay loop / `mfb_lab`-style scorer)

15. **Metric 1 (cost savings):** already the `total_cost` per arm — no change, but report the
    **capacity-bearing fraction** of L2 (contracted↔spot reshuffle kg) alongside the consolidation
    fraction, so the S45 decomposition is re-run and the redesign's effect is observable.
16. **Metric 2 (OTP):** classify each HAWB delivered ≤ `Δ_k` (on-time) vs `> Δ_k` (delayed); report
    count, %, and tardiness hours (`max(0, arrival − Δ_k)`). Requires short lanes to roll cargo →
    the design produces these. Deferred-air arcs are the cost lever that trades cost for OTP.
17. **Metric 3 (fallback incidence):** count/% of HAWBs on `ArcType.FALLBACK` (and charter if built)
    per arm — already extractable from `sol.fallback_hawbs`; surface as a first-class metric.

### Methodology / spec reconciliation (no code; gates the build)

18. **`model/air_freight_routing.tex` C.5d**: rewrite per-arc cap → per-lane increasing-block tariff
    (convex PWL, no-binary).
19. **`arrival_only_replan_methodology.md` §13 D-A18/D-A19**: amend κ → per-lane `τ_ℓ`; amend the
    spot row to the block ladder + ceiling; affirm `τ_ℓ` sets a mean over **analytic** expected
    demand (independence preserved); note hard BSA is now generated.

---

## H. Open questions / decisions for the user

1. **κ-tie of the spot ceiling (the S45 OPEN-3).** Should `τ_ℓ` move *only* contracted positions
   (Mechanism A — spot ceiling fixed/sourced, κ-independent), or *both* contracted and the spot
   ceiling (Mechanism B — the spec's intent, resolves critique-18 F-A)? This design assumes **B**
   (the dial scales total `S_ℓ`, so both tiers shrink together on a short lane) because that is what
   makes a short lane actually exhaust cheap capacity and fall to fallback — A alone leaves a fixed
   spot ceiling that may not bind. **B requires the D-A19/§7 methodology amendment.** Confirm B, or
   stage A→B.

2. **True charter (#7) — build or defer?** The three metrics work without it (fallback #6 is the
   feasible-but-expensive backstop). Building it adds a binary + a charter-vs-roll-everything
   tradeoff and one new constraint family. **Recommend defer to v2**; confirm.

3. **Composition vector + hard-BSA fraction.** Defaults proposed: 0.70 contracted / 0.22 spot /
   0.05 deferred / 0.03 reserve (INFERRED from the SOURCED <20%-spot dense-lane fact), and
   `hard_bsa_frac = 0.5`. These are the load-bearing realism knobs. Confirm, or adjust the
   contracted share / soft-hard split. (Everything else traces to the sourced calibration.)

**Secondary assumptions made (flag if wrong):** short-bucket band `[0.6, 0.85]` is set low enough to
force fallback — if the user wants short lanes that roll-but-rarely-fallback, raise the floor;
deferred-air at 0.05 share and the belly/freighter split are topology defaults in
`tpeb_air_instance` that need a schedule-realism pass; `r_a`/`r_c` re-anchored to ~$4.2/kg (below
spot) is the single change that makes contracted worth filling and is INFERRED, not SOURCED at lane
granularity (MRN).
