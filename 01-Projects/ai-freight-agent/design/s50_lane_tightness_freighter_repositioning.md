# S50 — Lane Tightness + Freighter Repositioning (supply model, O-D lane re-key)

**Status: APPROVED v1.0 (S50)** (D-A29/D-A30 approved with §14.1-R). Folds into `arrival_only_replan_methodology.md` §14.1-R.
Amends the S50 §14.1-R draft on one point: **tightness is re-keyed from DEST-region to ORIGIN×DEST
LANE, with the ORIGIN the dominant axis** (§14.1-R currently says "supply/tightness stays keyed on the
dest region" — that sentence is superseded here; the ground-gate / candidacy content of §14.1-R is
unchanged). Everything else in §14 (integer ULDs, decay §14.2, arrival spread §14.3, hard/soft BSA §14.4,
two-cost split §14.5, CRN, tardiness-always-on) carries over verbatim.

Sourcing tags: **SOURCED** (public URL), **INFERRED** (derived from a sourced anchor), **MRN**
(market-research-needed — flagged, not fabricated).

---

## 0. The decision this encodes (given, not relitigated)

Air-cargo tightness is a **lane** (origin-region i × dest-region j) property, **origin-dominant**: same US
destination, different Asian origins → different rates the same week (Baltic **BAI30 Hong Kong** vs **BAI80
Shanghai** move on different MoM/YoY tracks — SOURCED). Lane capacity splits by type with different
mobility:

| pool | mobility | why |
|---|---|---|
| **belly** | lane-frozen | byproduct of a specific passenger sector; capacity = whatever that pax flight leaves (SOURCED) |
| **BSA / allotment** | lane-frozen, per-leg | contracted on a named sector/lane, soft or hard (SOURCED) |
| **freighter / spot** | **repositionable across origins** (medium run) | carriers redeploy freighters on yield toward tight corridors; ANC tech-stops partly decouple the Asia leg from the US endpoint (SOURCED) |

The crux (D-A18): supply must stay **independent of the realized demand draw** (CRN — varying tightness
must leave the demand byte-identical; realized supply-vs-demand mismatch is the value source). Yet
"freighters chase demand." **Reconciliation: repositioning keys on the ANALYTIC/EXPECTED demand signal
(`D_ij`, built from the geometric door-box shares `q_ij` and `E[cw]` — reads zero demand draws), not the
realized book.** Realized mismatch still emerges because realized draws ≠ expectation. This is the heart
of the design (§3).

---

## 1. Lane tightness formulation

### 1.1 Regions and lanes

- **Origin regions** i ∈ {Taiwan (TPE), PRD (HKG,+CAN), East China (PVG,+SHA)} — split by ground group +
  `δ_drive` (§14.1-R D-A28); a door can only truck within its own origin region (Strait / >`δ_drive`
  blocks cross-origin drayage), so **a HAWB's origin region is fixed by its door and is inescapable** —
  this is *why* origin dominates cost.
- **Dest regions** j ∈ {West, Midwest, East} metro clusters (§14.1-R D-A27) — the optimizer *may*
  substitute gateways within a dest region (LA door via SFO); that within-dest substitution is a value
  source.
- **Lane** ℓ = (i, j), 3×3 = **9 lanes**. Each physical flight belongs to exactly one lane via
  `(region_of(off.origin), region_of(off.dest))`.

> **Tightness ≠ candidacy.** These origin/dest regions are a *supply-indexing* partition. The door↔airport
> candidacy graph (which airports a door may truck to; §14.1-R door-distance/SLA gate) is unchanged. A
> HAWB's lane for tightness is a demand attribute fixed by its doors (nearest origin cluster × nearest dest
> cluster), exactly like the existing `q_R` — the optimizer's realized routing may differ.

### 1.2 Analytic geometric lane share `q_ij` (zero demand draws)

Origin and dest doors are drawn **independently** (`_draw_doors`: separate uniforms over `_ORIGIN_BBOX`,
`_DEST_BBOX`), so the joint share **factorizes**:

```
  q_ij = q^O_i · q^D_j
```

- `q^D_j` = nearest-dest-cluster share of the dest door box (the existing `_dest_region_shares`, re-keyed
  to clusters per §14.1-R).
- `q^O_i` = **NEW** nearest-origin-cluster share of the origin door box (same haversine-partition
  machinery, origin side).

Both are pure box geometry + airport coords → **read zero demand draws → CRN-safe**, identically to
`E[cw]`. Analytic expected lane demand (chargeable-kg):

```
  D_ij = n_hawbs · q_ij · E[cw]        (E[cw] = _expected_cw_mean, closed form)
```

### 1.3 Origin-dominant lane tightness `τ_ij` — how the global dial maps down

Global dial `τ = ΣS / ΣD`. Map down with a **separable, mean-1-normalized** multiplier per axis:

```
  τ_ij = τ · (u_i / ū) · (v_j / v̄)
       where  ū = Σ_i q^O_i · u_i ,   v̄ = Σ_j q^D_j · v_j
```

- `u_i` = origin tightness multiplier, drawn in **WIDE** short/balanced/slack bands.
- `v_j` = dest tightness multiplier, drawn in **NARROW** bands.
- **Origin dominance = band-width asymmetry** (`Var(u) ≫ Var(v)`), the one design lever that makes origin
  the dominant axis. Illustrative bands (INFERRED; calibrate to BAI30/BAI80 + WorldACD origin dispersion):

  | bucket | origin band `u` (WIDE) | dest band `v` (NARROW) |
  |---|---|---|
  | short    | [0.60, 0.85] | [0.90, 0.97] |
  | balanced | [0.95, 1.05] | [0.98, 1.02] |
  | slack    | [1.20, 1.55] | [1.03, 1.12] |

**Σ recovers τ (proof + validated):**

```
  Σ_ij q_ij τ_ij = τ · (Σ_i q^O_i u_i / ū)·(Σ_j q^D_j v_j / v̄) = τ · 1 · 1 = τ
```

using q_ij = q^O_i q^D_j and the mean-1 normalization. Probe (`scratchpad/probe_lane_tau.py`, τ=1.5):
`Σ q_ij τ_ij = 1.5` exactly; origin multiplier spread 0.224 vs dest 0.052 (origin-dominant, as intended).

Per-lane supply (chargeable-kg): `S_ij = τ_ij · D_ij`, and `Σ_ij S_ij = τ · Σ D_ij = τ·D`. κ is subsumed
(`u_i=v_j=1 ∀i,j` ⇒ uniform-τ, contracted-only — the S38–S45 continuity arm).

---

## 2. Lane cap decomposition by type

For each lane ℓ=(i,j), `S_ij` (chargeable-kg) carves into three pools. Carve **frozen pools first**, the
freighter pool is the remainder:

```
  S_ij  =  S^bsa_ij   +   S^belly_ij   +   F^fr,0_ij
           (lane-frozen)  (lane-frozen)   (repositionable — §3)
```

**(a) BSA / contracted — lane-frozen, per-leg.** Integer ULD positions
`N^bsa_ij = round(contracted_share · S_ij / w_LD3)`, spread over the lane's contracted flights by
`Multinomial(N^bsa_ij, Dirichlet(α))` (`_spread_positions`, unchanged). Split soft (`per_flight`) vs hard
(`equalized`) by `hard_bsa_frac`, hard placed on the **tightest lanes first** (increasing `τ_ij`) so the
non-decaying reservoir sits where late-arrival contention fires (§14.2). `S^bsa_ij = N^bsa_ij · pivot`.
BSA is contracted on a named sector → never repositioned (SOURCED: soft/hard BSA fix a set space on
specific flights). *(Grounding: soft = cancellable with notice, hard = take-or-pay every flight.)*

**(b) belly — lane-frozen, passenger-schedule-tied.** A spot arc is belly iff any leg is `PAX_BELLY`
(`_is_belly`, unchanged). Belly arcs get the **0.4× freighter spot-cap thinning** (D2, GROUNDED — memory
`reference_belly_freighter_capacity`). Belly capacity is a byproduct of the pax sector and **cannot be
repositioned for cargo** (SOURCED: belly capacity follows the passenger network — added/removed with pax
frequency and aircraft swaps). **LEAN keeps belly as the existing per-arc thinning** (no separate pax
proxy). `S^belly_ij` = the thinned spot budget landing on the lane's belly arcs.

**(c) freighter / spot — the repositionable remainder.**

```
  F^fr,0_ij = (1 − contracted_share) · S_ij  restricted to the lane's FREIGHTER spot arcs
            = the pre-repositioning freighter-spot budget (chargeable-kg)
```

This is the only mobile pool; §3 reallocates it across origins. Belly and BSA are subtracted first and
stay put, so the three pools always sum to `S_ij` by construction.

---

## 3. The freighter repositioning mechanism (the core)

**One-line:** within a dest region j, redistribute the total freighter-spot budget across ORIGINS toward
**expected residual demand** (freighters chase yield), on an **analytic** signal, at generation time,
frozen across arms.

### 3.1 The signal — expected residual demand (analytic, zero demand draws)

Per lane, the demand the freighter+fallback pool is expected to absorb after the frozen pools:

```
  R_ij = max(0,  D_ij − S^bsa_ij − S^belly_ij )
```

Every term is analytic (`D_ij` from `q_ij·E[cw]`; `S^bsa_ij`, `S^belly_ij` from `τ_ij` + composition
constants). **`R_ij` reads zero demand draws.** No circularity: the freighter pool itself is excluded from
`R_ij` (it's what we're sizing). `R_ij` is the honest "what's left for freighters to chase" — high on
tight origins, low on slack ones. *(Grounding: carriers redeploy freighters toward high-yield corridors,
trimming weaker lanes — SOURCED, The Loadstar.)*

### 3.2 The reallocation rule (deterministic, conserves capacity)

Within each dest region j, conserve the total freighter-spot budget
`G_j = Σ_i F^fr,0_ij` and redistribute by residual-demand weight, with a repositioning intensity
`ρ ∈ [0,1]`:

```
  ω_ij  = R_ij / Σ_i' R_i'j                       (expected-yield share across origins; Σ_i ω_ij = 1)
  F^fr_ij = (1 − ρ)·F^fr,0_ij  +  ρ·G_j·ω_ij       (blend: frozen ↔ fully-chased)
```

- **`ρ = 0`** → no repositioning (S38–S45 continuity; the **null/negative-control** — freighters stay
  where sized).
- **`ρ = 1`** → freighters fully chase expected residual demand across origins.
- **Conservation:** `Σ_i F^fr_ij = G_j` for every j (freighters swap Asian departure origins for the same
  US market — medium-run redeploy; ANC tech-stop decouples the Asia leg — SOURCED), so `Σ_ij S_ij = τ·D`
  is preserved and the global dial is untouched. Reposition is **across origins, within a fixed dest
  region** — the origin-dominant axis. (Cross-dest ANC repositioning = FULL/v2, §6.)
- **Edge handling:** if `Σ_i R_i'j = 0` (all slack), fall back to `ω_ij = F^fr,0_ij/G_j` (no move);
  `F^fr_ij ≥ 0` by construction.

### 3.3 CRN proof — reads zero realized demand draws (D-A18 preserved)

`F^fr_ij` is a deterministic function of `{q_ij, E[cw], τ_ij, composition constants, ρ}`. `q_ij` and
`E[cw]` are geometry/closed-form; `τ_ij` is drawn on the `region_tightness` stream; ρ is a config scalar.
**None reads the `demand` stream, and repositioning itself draws no RNG at all** (pure arithmetic on
analytic quantities). Therefore varying ρ (or τ, or α) leaves the `demand` draw **byte-identical** — the
hard CRN gate holds. The repositioned budget sets the frozen `C0_a` amplitudes, **bit-identical across
H₀/M₀/M₁'/M₁/π_hind** (D-A16).

### 3.4 Why realized mismatch still emerges (the value source survives)

Repositioning aligns freighter supply with **expected** residual demand `R_ij`. But the realized instance
draws (i) *which* doors actually land in each lane (a finite-n realization scattered around `q_ij`), (ii)
*when* each HAWB arrives (the §14.3 bucket mixture) interacting with the §14.2 booking-curve decay. So the
realized lane load ≠ `D_ij`, and realized ≠ expected on both count and timing. Freighters positioned on
expectation are therefore **still idle on some lanes and short on others** at realization — the
supply/demand mismatch the optimizer (and the open-book replan) exists to resolve. Repositioning makes the
carrier *smart* (realistic — it doesn't strand capacity on obviously-slack origins) **without erasing the
stochastic mismatch**, so L2 is measured against a credible baseline rather than an artificially dumb one.
*(Directional check to run: L2 must survive ρ>0; if high ρ collapses L2, the mismatch was carrying it and
we report that honestly — see §6 risk.)*

### 3.5 Generation-time draw, NOT a decision-clock adjustment

Repositioning is a **generation-time** allocation (like `τ_R`, the α-spread) — it sets `C0_a`, frozen for
the whole proof (D-A16). It is **not** a decision-clock decay like the booking curve (§14.2). Rationale:
repositioning is a **medium-run carrier decision on forecast, made before the booking window opens**;
making it clock- or arm-dependent would confound L2 (the arms must see one identical capacity substrate).
The booking-curve decay §14.2 then acts *on top* of the repositioned `C0_a` per decision clock, unchanged.

---

## 4. Generator change map (`data/synthetic/air_generator.py`)

**Legacy κ path (`tau is None`) untouched. Region path (`tau` set) is re-keyed dest-region → lane.**

### CHANGES

| S47 function | change |
|---|---|
| `_dest_region_shares` → **keep**, add **`_origin_region_shares`** | new q^O_i, nearest-origin-cluster partition of `_ORIGIN_BBOX` (mirror of dest side); `q_ij = q^O_i·q^D_j` assembled by a small helper `_lane_shares`. |
| `_draw_region_tightness` → **`_draw_lane_tightness`** | draw `u_i` (WIDE bands) + `v_j` (NARROW bands) in canonical order on the **existing `region_tightness` stream**; return `τ_ij = τ·(u_i/ū)(v_j/v̄)`. Draw-count change is CRN-safe (own stream, not `demand`). |
| `_size_region_supply` → **`_size_lane_supply`** | `S_ij = τ_ij · n · q_ij · E[cw]`; sums to `τ·D`. |
| `_draw_region_network_supply` → **`_draw_lane_network_supply`** | group flights by `lane(off) = (region_of(off.origin), region_of(off.dest))` (was `off.dest`); `N^bsa_ij = round(S_ij·contracted_share/w_LD3)` spread per lane. |
| `_build_region_rate_catalog` | re-key spot grouping by lane; apply **repositioned** freighter-spot budget `F^fr_ij` to freighter arcs, belly budget to belly arcs (belly thinning `×belly_frac` unchanged). |
| `_split_contracted` | hard-BSA placement sort key `τ_R(dest)` → `τ_ij(lane)` (tightest lane first). |
| `_gen_hawbs` / `region_of` | add airport→region map for BOTH axes (origin clusters, dest clusters). |

### NEW

| item | role |
|---|---|
| **`_reposition_freighter_spot(F0, R, rho)`** | the §3 mechanism: per dest region j, `G_j=Σ_i F0_ij`, `ω_ij=R_ij/Σ R`, `F_ij=(1-ρ)F0_ij+ρG_jω_ij`. **Deterministic, no RNG.** |
| **`_expected_residual(D_ij, S_bsa_ij, S_belly_ij)`** | analytic `R_ij` (§3.1). |
| `GenConfig` / `ArrivalConfig`: **`reposition_rho: float = 0.0`** | swept dial + null (default 0 = continuity/negative control). Headline sweep `{0.0, 0.5, 1.0}`. |
| `GenConfig` / `ArrivalConfig`: **`origin_mix: tuple[tuple[str,str],...]`** | origin-region → bucket (parallel to `region_mix`, now = dest mix). |
| band tables `_ORIGIN_TAU_BANDS` (WIDE) / `_DEST_TAU_BANDS` (NARROW) | replace the single `_TAU_BANDS`. |

### STAYS (no change)

`_expected_cw_mean`, `_size_total_supply`, `_spread_positions`, `_draw_spot_regime`, `_is_belly`, the
belly `×0.4` thinning, CapDecay §14.2, hard/soft BSA MILP wiring, all four MILP capacity gates,
tardiness-always-on, generate-all-first, the `demand`/`rates`/`supply`/`spot_regime`/`region_tightness`/
`cap_decay` streams (no new stream — repositioning is deterministic).

### CRN gates preserved

`demand` stream untouched (byte-identical under ρ/τ/α sweeps — the hard gate). Supply drawn on `supply` in
canonical **lane** order (sorted) → enumeration-independent. `region_tightness` draws `u` then `v` in
canonical order. Repositioning deterministic. Frozen capacity vector bit-identical across arms (D-A16, now
includes `F^fr_ij`).

---

## 5. Grounding

| claim | tag | source |
|---|---|---|
| Origin is the dominant tightness axis: per-origin indices move on different tracks (BAI30 HK MoM/YoY vs BAI80 Shanghai) | SOURCED | Baltic Exchange BAI Nov-2025 & Aug-2025 columns — balticexchange.com/en/news-and-events/news/guest-column/2025/ |
| HK→US vs other-Asia→US rate spread same week (origin dispersion) | SOURCED (per prior pass) | WorldACD weekly (wk50 2025); TAC/Baltic per-origin indices dashboard.tacindex.com |
| Freighters redeploy across corridors on yield; carriers move aircraft where demand is strongest rather than back to the same lane | SOURCED | The Loadstar, "Forwarders and airlines reposition as air cargo market steadies" — theloadstar.com/forwarders-and-airlines-reposition-as-air-cargo-market-steadies/ |
| China→US freighter demand pull / capacity redeployment to other lanes | SOURCED | same Loadstar column |
| Belly capacity follows the passenger network (added/removed with pax frequency, lost on aircraft swaps) | SOURCED | Transport&Logistics ME "Airlines Maximize Belly-Hold Cargo 2025"; Etihad Cargo summer/winter schedule releases; Cooperative Logistics Network belly-vs-freighter (2025) |
| BSA reserves fixed space on **specific flights / a named lane**; soft = cancellable w/ notice, hard = pay every flight | SOURCED | cargo.flowers/en/blog/post/why-do-you-need-bsa; airsupplycn.com/blocked-space-agreement; Dimerco Xiamen BSA; hkaircargo.com 2026 BSA Program |
| belly spot-cap = 0.4× freighter spot cap | SOURCED (derived) | memory `reference_belly_freighter_capacity` (777 ACAP, Atlas 747-400F, 747-8 wiki, A350/787 belly study, IATA transpac-freighter-share) — §14.4 |
| spot:contract two-sided band (~0.85 soft … ~1.18 peak) | SOURCED | memory `reference_air_spot_contract_ratio` (Xeneta/WorldACD 2023–26) |
| contract ~$4.2/kg < spot base $5.5/kg (NE-Asia→NA Apr-26) | SOURCED | Xeneta, §14.4 |
| **`ρ` (repositioning intensity) realistic value** | **MRN** | no public figure for the freighter-share reallocated per unit yield gap — swept, not asserted |
| **origin vs dest band widths (u vs v spread)** | **INFERRED / MRN** | shape from BAI30/BAI80 dispersion; exact widths need WorldACD origin-variance calibration |

---

## 6. Open calibration, risks, LEAN vs FULL

### LEAN (build first, for the proof)

1. `_origin_region_shares` + `q_ij = q^O_i·q^D_j`.
2. Separable origin-dominant `τ_ij` (WIDE origin / NARROW dest bands), Σ recovers τ (validated).
3. Lane-keyed supply sizing + three-pool carve (BSA, belly-thinned spot, freighter-spot remainder).
4. `_reposition_freighter_spot` with `reposition_rho` swept **{0.0, 0.5, 1.0}**, per-dest-region
   conservation, expected-residual weights, generation-time frozen.
5. Reuse existing belly `×0.4` thinning, soft/hard BSA, CapDecay, cost split, CRN gates.

### FULL / v2 (defer)

- **ANC cross-dest repositioning** (a second, weaker axis moving freighters across US endpoints).
- **Explicit passenger-schedule belly proxy** (per-lane pax-frequency signal) instead of the flat `×0.4`.
- **Yield-weighted** repositioning (an explicit rate/margin term, not residual-demand alone).
- **Correlated origin×dest doors** (drops the `q_ij` factorization) — only if geography demands it.
- Calibrate origin/dest band widths to measured WorldACD/BAI dispersion.

### Risks

- **R1 — over-repositioning collapses L2.** If ρ→1 perfectly tracks expectation, expected mismatch
  shrinks; L2 must survive on *realized* (count+timing) mismatch (§3.4). **Gate:** L2 CI > 0 at the
  headline ρ; if not, report ρ-sensitivity honestly (the null at ρ where mismatch vanishes is a *finding*,
  not a failure).
- **R2 — D-A18 leak.** Any path where `F^fr_ij` reads a demand draw breaks CRN. **Test:** vary ρ/τ/α ⇒
  `demand` stream byte-identical (extend the existing CRN gate to cover `reposition_rho`).
- **R3 — factorization assumption.** `q_ij = q^O_i·q^D_j` holds only while origin/dest doors are drawn
  independently (they are, today). Assert it; revisit if doors correlate.
- **R4 — conservation vs global τ.** Repositioning moves only the freighter slice within a dest region;
  BSA+belly frozen. Assert `Σ_ij S_ij` unchanged pre/post reposition (global τ invariant).
- **R5 — coarse at proof scale.** Few flights per lane ⇒ the Dirichlet spread and the reposition split are
  lumpy; smooths at forwarder scale (§11 stress). Report per-lane binding-rate, not a network average.

### Open calibration items

- `reposition_rho` headline value (MRN) — sweep meanwhile.
- Origin vs dest band widths (INFERRED) — calibrate to BAI30/BAI80 + WorldACD origin dispersion.
- `q^O_i` origin door-box partition (Taiwan / PRD / East China) — geometry, needs the §14.1-R origin boxes.
- Per-lane composition (contracted / belly / freighter split) — reuse §14.4 defaults; confirm at lane grain.

### Decisions to fold into §14.1-R

- **D-A29 (proposed):** tightness re-keyed to **O-D lane**, origin-dominant, via separable
  `τ_ij = τ·(u_i/ū)(v_j/v̄)` with `Var(u) ≫ Var(v)`; supersedes "supply/tightness stays keyed on the dest
  region."
- **D-A30 (proposed):** freighter-spot is the sole **repositionable** pool; belly + BSA lane-frozen.
  Repositioning = generation-time, deterministic, expected-residual-weighted, per-dest-region conserved,
  dialed by `reposition_rho` (default 0 = null).
