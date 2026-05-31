# Air MILP Formulation Spec — O-D-arc graph

**Status:** Draft v2 (Session 15, 2026-05-23 — post 3-agent critique).
**Not** an approved formulation; transient artifact (deletes after Stage 2
LaTeX rewrite per agreement).
**Purpose:** specify the air routing MILP — variables, constraints, objective —
on the **O-D-arc graph** validated in `model/air_graph_construction.md`. This
spec folds in the locked outcomes of the 19-item review
(`model/air_review_notes.md`): item 2 (currency cross-ref), item 3 (linear soft
tardiness), item 4 (bucket on O-D-arc graph), item 7 (rate families per offer),
item 13-A (P.3 bug fix), item 15 (P.18 removed), item 18 (cleanups), Finding S
Change 1 (TT-quantile hook + planning-bound framing).

This document precedes Stage 2 (LaTeX rewrite). It is meant to be precise enough
for (a) a Session-11-style 3-agent math critique, (b) the LaTeX rewrite, and
(c) the Python implementation.

---

## 0. Cross-references

| Doc | Role |
|---|---|
| `model/air_graph_construction.md` | Graph layer — node types, arc types, MAWB / `g(k)` partition, two-phase construction, case catalogue. **Read first.** This spec assumes it. |
| `model/air_review_notes.md` | 19-item review outcomes. Source of locked decisions cited inline below. |
| `data_model.md` §7 | Currency / FX — single USD-canonical FX table, convert at solve. |
| `data_model.md` §5 | Spot rate snapshots — per-run rate binding. |
| `data_model.md` §4 | Policy rules (carrier blacklist/preference, embargo, lithium, ULD interchange, service products). |
| `data_model.md` §6 | Surcharge catalog (Path-A per-arc / Path-B flight-level). |
| `transit_time_model.md` | Transit Time Service — P85–P90 quantile hook (Finding S Ch 1). |
| `scalability.md` | Decomposition / column-generation alternatives (option D in §sec:consolidation-alternatives). |

---

## 1. Conventions

- **Time:** all timestamps in UTC, units = hours from `t = 0` of the solve horizon.
- **Currency:** all costs / rates in USD. Catalog rates may be quoted in origin-country local currency (per IATA TACT); FX conversion to USD applied once at solve build per `data_model.md §7`. Single FX table per solve; no per-run pinning (deferred Phase 2+).
- **Indexing convention:** function-style `f(·)` denotes deterministic look-ups; subscript denotes the indexed family.
- **"MAWB" in this spec** always means `(arc, g)` — a candidate MAWB. Whether it
  is *instantiated* is a decision variable (§5).

---

## 2. Sets and indices

### 2.1 Nomenclature

| Symbol | Type | Description |
|---|---|---|
| `K` | set | HAWBs (commodities / shipments) to route this solve. |
| `k` | index | A HAWB, `k ∈ K`. |
| `N` | set | Nodes of the routing graph. |
| `N_k ⊆ N` | set | Nodes reachable by HAWB `k` after Phase-1 pre-filter. `O_k ∈ N_k` and `D_k^{node} ∈ N_k` guaranteed by the pre-filter (§4 step 7). |
| `n` | index | A node. |
| `A` | set | Arcs of the routing graph. Each arc is directed: `tail(a) ∈ N` is its origin endpoint, `head(a) ∈ N` is its destination endpoint. |
| `tail(a), head(a)` | function | Endpoints of arc `a`. `a` carries flow from `tail(a)` to `head(a)`. |
| `A_k ⊆ A` | set | Arcs in HAWB `k`'s pre-filtered subgraph (Phase-1, step 11). |
| `A^{MAWB} ⊆ A` | set | MAWB-eligible air arcs (rate_family ∈ `{flat_rate, min_flat_breaks, per_uld_pivot}`). |
| `A^{MFB} ⊆ A^{MAWB}` | set | Arcs with `rate_family_a = min_flat_breaks`. |
| `A^{coload} ⊆ A` | set | Co-load air arcs (per-kg, no MAWB). |
| `A^{ground} ⊆ A` | set | Non-air arcs: pickup, cartage, CFS-dwell, deconsol-dwell, customs-dwell, delivery. |
| `A^{cust} ⊆ A^{ground}` | set | Customs-clearance dwell arcs (between `CFS-D` and final delivery). One per HAWB per route (graph doc §3). |
| `a` | index | An arc, `a ∈ A`. |
| `transit(k, a)` | function | Per-HAWB transit time on arc `a` (hours): for ground arcs, `δ_a + δ^{cust}_k · 𝟙[a ∈ A^{cust}]`; for air arcs, `μ_a` (graph-build-precomputed scalar — see §3.3). Used in C.6. |
| `G` | set | Consolidation groups (range of `g(·)`). |
| `g(k)` | function | The consolidation-group key for HAWB `k`. Formally: `g(k) = (cargo_class(k), HAWB-id(k))` if `cargo_class(k) ∈ {VAL, HUM, AVI}` (non-consolidable; singleton group); else `g(k) = (cargo_class(k), screening(k), temperature(k))` (consolidable). Per graph doc §4. |
| `K_a ⊆ K` | function | HAWBs whose subgraph contains arc `a`: `K_a = {k : a ∈ A_k}`. |
| `G_a ⊆ G` | function | Distinct group keys present on arc `a`: `G_a = {g(k) : k ∈ K_a}`. |
| `M` | set | MAWB candidates: `M = {(a, g) : a ∈ A^{MAWB}, g ∈ G_a}` (Phase-2 step 2). |
| `m = (a, g)` | index | A MAWB candidate, `m ∈ M`. |
| `U` | set | ULD types (LD3, LD7, PMC, AKE, …). |
| `u` | index | A ULD type, `u ∈ U`. |
| `C` | set | Per-ULD-pivot (BSA) contracts. |
| `C^{eq} ⊆ C` | set | Equalized-settlement BSA contracts (the rest are per-flight). |
| `c` | index | A contract, `c ∈ C`. |
| `A_c^{MAWB} ⊆ A^{MAWB}` | function | Arcs tagged to BSA contract `c` (i.e.\ the MAWB-eligible arcs whose offer is contract `c`). |
| `C^{pu} ⊆ A^{MAWB}` | set | Per-ULD-pivot arcs (rate_family `= per_uld_pivot`); equals `⋃_{c ∈ C} A_c^{MAWB}`. |
| `B_a` | set | Weight-break segments for arc `a` with `a ∈ A^{MFB}`. Ordered ascending: `break_{a,1} < break_{a,2} < …`. |
| `b` | index | A break segment, `b ∈ B_a`. |
| `P` | set | Service products (tenant catalog; `data_model.md §4`). |
| `sp(k) ∈ P` | function | The service product bound to HAWB `k`. |
| `p` | index | A service product. |

### 2.2 Notes

- `N`, `A` are constructed by **Phase 1** (`air_graph_construction.md §5`), independent of consolidation logic.
- `M` is constructed by **Phase 2** (overlay): one MAWB candidate per `(MAWB-eligible arc, distinct group present on that arc)`.
- Co-load arcs are skipped by Phase 2 — no MAWB candidates on them.
- **No per-flight capacity coupling in the MILP.** Physical flight legs are a graph-doc concept (`air_graph_construction.md §3`); they enter the MILP only through pre-computed per-arc scalars: `μ_a` (transit time, includes internal MCT for multi-leg arcs) and `CO_a^*` (effective cutoff at `tail(a)` for air arcs leaving a POL). The MILP does **not** index over flights `f` directly.

---

## 3. Parameters

### 3.1 Per-HAWB

| Symbol | Units | Description |
|---|---|---|
| `w_k` | kg | Actual gross weight of HAWB `k`. |
| `v_k` | m³ | Total volume of HAWB `k`. |
| `cw_k = max(w_k, v_k·167)` | kg | Chargeable weight of HAWB `k` (per-HAWB billing reference; **not** used in bucket capacity — see C.4 and item 13-A). |
| `t_k^{rdy,early}, t_k^{rdy,late}` | h | Cargo-ready window. |
| `T_k^{dead}` | h | Hard deadline (delivery). |
| `T_k^{SLA}` | h | Service-product soft deadline `= t_k^{rdy,early} + T^{SLA}_{sp(k)}` (definition of `T^{SLA}_{sp(k)}` in §3.6). |
| `Δ_k = min(T_k^{dead}, T_k^{SLA})` | h | **Effective soft deadline** used in C.10. Renamed from `D_k` to disambiguate from the destination node `D_k^{node}`. |
| `T_k^{abs}` | h | Absolute drop-dead; HAWB infeasible if final-delivery time exceeds this (C.11). `+∞` for GEN; finite for PER, etc. |
| `δ^{cust}_k` | h | Per-HAWB customs-dwell time (on the `A^{cust}` arc between `CFS-D` and final delivery). |
| `O_k ∈ N_k` | node | Origin door node. |
| `D_k^{node} ∈ N_k` | node | Destination door node. |
| `sp(k) ∈ P` | service product | The service product binding (§3.6). |

### 3.2 Per-arc (ground arcs in `A^{ground}`)

| Symbol | Units | Description |
|---|---|---|
| `δ_a` | h | Dwell / transit time for ground arc `a` (pickup, cartage, CFS dwell, deconsol-dwell, delivery). Includes any in-transit hub customs adjustment for deconsol arcs. |
| `c_a^{flat}` | USD/HAWB | Flat per-HAWB handling charge on arc `a` (e.g.\ per-AWB CFS handling fee). May be 0. |
| `c_a^{kg}` | USD/kg | Per-kg handling charge on arc `a` (e.g.\ per-kg cartage). May be 0. |

Total per-HAWB ground-arc cost: `(c_a^{flat} + c_a^{kg} · w_k) · x_{k,a}` (objective term in §7).

### 3.3 Per-arc (air arcs in `A^{MAWB} ∪ A^{coload}`)

| Symbol | Units | Description |
|---|---|---|
| `μ_a` | h | Realized air transit time (graph-build-precomputed scalar). For a multi-leg arc, `μ_a` already includes internal MCT between the legs — same-MAWB through-connection MCT folds in here (no separate constraint). |
| `CO_a^*` | h | Effective cargo cutoff at `tail(a)` for air arcs leaving a POL (graph-build-precomputed: max over DCO, AMS, ICS2, ACI, minus per-HAWB prep time). |
| `rate_family_a` | enum | One of `{flat_rate, min_flat_breaks, per_uld_pivot, coload_per_kg}` (last applies only to `A^{coload}`). |
| `currency_a` | ISO 4217 | Quotation currency (converted to USD at build). |

Per-leg physical-flight metadata (carriers, block times, ETD/ETA) lives in the
graph-construction doc; it does not enter the MILP except through the pre-computed
scalars `μ_a` and `CO_a^*`.

Family-specific:

| Family | Symbols | Description |
|---|---|---|
| `flat_rate` | `m_a` (USD/kg), `min_chg_a` (USD/MAWB), `cap_a` (kg, optional) | Single flat rate; per-MAWB cost `= max(min_chg_a · z_{a,g}, m_a · CW_{a,g})`. `min_chg_a` is applied **per instantiated MAWB on arc `a`** (each distinct group `g` is a separate document, separately subject to the minimum). Optional offer-level cap. |
| `min_flat_breaks` | `{(break_{a,b}, rate_{a,b})}_{b ∈ B_a}`, `cap_a` (kg, optional) | IATA next-break-down. `B_a` is ordered ascending; `break_{a,b}` = minimum billed weight if break `b` is selected; `rate_{a,b}` = per-kg rate. Per-MAWB cost `= min_b rate_{a,b} · max(CW_{a,g}, break_{a,b})`. Optional offer-level cap. |
| `per_uld_pivot` | `r_a` (USD/kg) = `r_c` for the tagging contract `c`; `π_a` (kg) pivot weight; `U_a ⊆ U` admissible ULD types; `N_{a,u}` (count) contracted ULD-position allotment per type (hard cap on `Σ_g η_{a,g,u}`); `W_u` (kg) ULD payload; `V_u` (m³) ULD volume; `settlement_a ∈ {per_flight, equalized}` | BSA take-or-pay; cost `= r_a · max(CW, π_a · η_total)`. |
| `coload_per_kg` | `m_a^{cl}` (USD/kg), `cap_a^{cl}` (kg, optional) | Per-kg, billed on each shipment's own `cw_k`. Optional offer-level cap if the co-loader specified one (often unspecified — treat as uncapped at planning; capacity is request/confirm at booking). |

`cap_a` / `cap_a^{cl}` are bounds on **actual** weight summed across HAWBs on the
arc (across consolidation groups), enforced by C.5c.

### 3.4 Per-flight — graph-build only

Physical flights `f` exist in `air_graph_construction.md` but are **not** indexed
by the MILP. Flight metadata (ETD, ETA, block time, operating/marketing carrier,
DCO / AMS / ICS2 / ACI cutoffs) is consumed by the graph generator to compute
per-arc scalars `μ_a` (§3.3) and `CO_a^*` (§3.3). The MILP sees only the scalars.

**No `W_f`, no `V_f`.** The forwarder cannot know physical-flight capacity in any
planning sense — it depends on other parties' bookings the forwarder doesn't see.
All real capacity bounds are at the **contract / offer** level (§3.3) and the
**ULD** level (§3.3 `per_uld_pivot`): contracted allotment `N_{a,u}`, offer-level
caps `cap_a` / `cap_a^{cl}` (where specified), and per-ULD physical limits `W_u, V_u`.

### 3.5 Per-contract (per_uld_pivot only)

| Symbol | Units | Description |
|---|---|---|
| `r_c` | USD/kg | Per-kg rate for contract `c ∈ C`. Equal to `r_a` for every `a ∈ A_c^{MAWB}` (consistency assumed: all arcs tagged to the same contract carry the same rate). |
| `A_c` | kg | **Remaining sunk allowance** for equalized-settlement contract `c ∈ C^{eq}` this solve (exogenous from the consumed-weight accumulator; §6.13). Undefined for `per_flight` settlement (not used). |

### 3.6 Per-service-product

| Symbol | Units | Description |
|---|---|---|
| `T^{SLA}_p` | h | SLA transit-time bound for service product `p ∈ P`. Used in §3.1 `T_k^{SLA} = t_k^{rdy,early} + T^{SLA}_{sp(k)}`. |
| `w_p` | USD/h-late | Tardiness weight for service product `p`. `CALIBRATION NEEDED`. |

### 3.7 Other

| Symbol | Units | Description |
|---|---|---|
| `ε` | dimensionless | Dunnage factor for density mixing (≈ 0.05 = 5%). Renamed from `δ` to avoid overload with `δ_a` (arc dwell) and `δ^{cust}_k` (per-HAWB customs dwell). |

---

## 4. Pre-filter (Phase 1, step 11 — replicated here for completeness)

For each HAWB `k`, the per-shipment subgraph `A_k ⊆ A` is constructed by pruning arcs that fail any of:

1. **Lane / reachability** — arc must lie on a path from `O_k` to `D_k^{node}`.
2. **Cargo type compatibility** — `cargo_type_ok(k, a)` (e.g.\ AVI requires AVI-capable equipment; DGR requires DGR-accepting carrier on each underlying flight).
3. **Embargo** — no active embargo on `(carrier^{op}_f, k.commodity)` for any `f ∈ legs(a)`.
4. **Lithium** — `lithium_accept_f` whitelist on all `f ∈ legs(a)` if `k.lithium_spec ≠ ⊥`.
5. **Screening** — `screening_status(k)` compatible with each flight's required chain of custody.
6. **Service product** — `mode_ok ∧ carrier_ok ∧ ac_type_ok` per `sp(k)`'s bundle (resolved carrier sets per the 5-layer cascade, §6.15 of the prior LaTeX / `data_model.md §4`).
7. **Deadline reachability** — earliest arrival via this arc still ≤ `T_k^{abs}`.
8. **ULD physical fit** — for any `a ∈ C^{pu}` (per-ULD-pivot arc), prune if
   `w_k > max_{u ∈ U_a} W_u` or `v_k > max_{u ∈ U_a} V_u` (HAWB physically does
   not fit in any single ULD of the arc's admissible types). Catches the cargo
   that the per-MAWB-aggregate C.5b cannot reject (see C.5b remark below). HAWB
   may still route via `flat_rate` / `min_flat_breaks` / `coload_per_kg` arcs
   where ULD claiming does not apply.

If `A_k = ∅` after pruning → HAWB `k` is reported as a **structured rescue event**; it does **not** enter the MILP.

---

## 5. Decision variables

### 5.1 Nomenclature

| Symbol | Type | Index | Description |
|---|---|---|---|
| `x_{k,a}` | binary | `k ∈ K`, `a ∈ A_k` | HAWB `k` uses arc `a`. |
| `z_{a,g}` | binary | `(a, g) ∈ M` | MAWB `(a, g)` is **instantiated** (active). |
| `CW_{a,g}` | continuous ≥ 0 | `(a, g) ∈ M` | Chargeable weight booked on MAWB `(a, g)`. |
| `Wt_{a,g}` | continuous ≥ 0 | `(a, g) ∈ M` | Aggregate actual weight on MAWB `(a, g)` (incl. dunnage). |
| `Wv_{a,g}` | continuous ≥ 0 | `(a, g) ∈ M` | Aggregate volumetric weight on MAWB `(a, g)`. |
| `γ_{a,g,b}` | binary | `(a, g) ∈ M, a ∈ min_flat_breaks, b ∈ B_a` | Break selector for MAWB `(a, g)`. |
| `BW_{a,g,b}` | continuous ≥ 0 | as above | Disaggregated billed weight on break `b`. |
| `η_{a,g,u}` | integer ≥ 0 | `(a, g) ∈ M, a ∈ C^{pu}, u ∈ U_a` | Number of ULDs of type `u` claimed for MAWB `(a, g)`. |
| `pivot_{a,g}` | continuous ≥ 0 | `(a, g) ∈ M, a ∈ C^{pu}` | Pivot-active billed weight: `max(CW_{a,g}, π_a · Σ_u η_{a,g,u})`. |
| `over_c` | continuous ≥ 0 | per equalized contract `c` | Allowance overage: `max(0, Σ chargeable on `c` − A_c)`. |
| `t_k^n` | continuous ≥ 0 | `k ∈ K, n ∈ N_k` | Arrival time of HAWB `k` at node `n`. |
| `τ_k` | continuous ≥ 0 | `k ∈ K` | Tardiness of HAWB `k` (item 3). |

### 5.2 Notes

- **No `y_{f,o,k}` and no `(f, o)` bucket variables** — superseded by the O-D-arc-graph + `(arc, g)` MAWB.
- **No per-shipment cost-precomputation `c_o(cw_k)`** — bucket cost is on MAWB aggregate `CW_{a,g}`.
- **No `h_{k,m}` membership variable** — membership of HAWB `k` in MAWB `(a, g)` is implied by `x_{k,a} ∧ (g(k) = g)`.
- `η` is the ULD claim *vector*; `pivot_a,g` is the pivot-floor variable (linearization of `max`).

---

## 6. Constraints

Numbering is **fresh** — old `P.x` numbers from the prior LaTeX are listed for traceability in §6.* headers but are not authoritative. Mapping: prior P.1 → C.1, prior P.2/P.3 → C.4/C.5, prior P.10 → C.5b, prior P.11–P.14 → C.6–C.9, prior P.15 → soft (C.10/C.11), prior P.17 → C.4d, **prior P.18 removed (item 15)**, prior P.19 → C.12, prior P.20 → soft (C.10), prior P.21 → C.14.

### C.1 Flow conservation

For each HAWB `k ∈ K` and each node `n ∈ N_k`:

```
   Σ_{a ∈ A_k : tail(a) = n} x_{k,a}   −   Σ_{a ∈ A_k : head(a) = n} x_{k,a}
   =
   { +1   if n = O_k
   { −1   if n = D_k^{node}
   {  0   otherwise
```

(Standard MCNF balance, supply form: `outflow − inflow = b(n)` with `b(O_k) =
+1`, `b(D_k^{node}) = −1`, `b(n) = 0` elsewhere. HAWB enters at origin door,
leaves at destination door, conserves at every intermediate node.)

### C.2 MAWB instantiation linkage

For each MAWB `(a, g) ∈ M` and each HAWB `k ∈ K_a` with `g(k) = g`:

```
   x_{k,a}  ≤  z_{a,g}                                                      (C.2a)
```

For each MAWB `(a, g) ∈ M`:

```
   z_{a,g}  ≤  Σ_{k ∈ K_a : g(k) = g} x_{k,a}                               (C.2b)
```

C.2a forces MAWB activation whenever any HAWB rides the arc in that group. C.2b prevents phantom activation. Together: `z_{a,g} = 1 ⇔ ∃ k ∈ K_a, g(k)=g, x_{k,a}=1`.

### C.3 — REMOVED (Session-15 critique)

C.3 was `x_{k,a} ≤ z_{a, g(k)}` ∀ `(a ∈ A^{MAWB}, k ∈ K_a)`. Both critique
agents confirmed it is **literally redundant** with C.2a — C.2a already covers
the same `(a, g, k)` triples once `g` is fixed to `g(k)`. Dropped.

### C.4 Chargeable-weight aggregation (density mixing)

For each MAWB `(a, g) ∈ M`:

```
   Wt_{a,g}  =  (1 + ε) · Σ_{k ∈ K_a : g(k) = g} w_k · x_{k,a}              (C.4a)
   Wv_{a,g}  =  Σ_{k ∈ K_a : g(k) = g} v_k · 167 · x_{k,a}                  (C.4b)
   CW_{a,g}  ≥  Wt_{a,g}                                                    (C.4c)
   CW_{a,g}  ≥  Wv_{a,g}                                                    (C.4d)
```

C.4c–d give `CW_{a,g} = max(Wt, Wv)` at optimum because cost is non-decreasing in `CW` (correctness condition stated in §6 of `air_review_notes.md` item 4 scrutiny point 1).

For co-load arcs `a ∈ A^coload`, no MAWB → no `CW_{a,g}` — costing is per-HAWB on `cw_k · x_{k,a}` (§7).

### C.5 Per-contract allotment cap (per_uld_pivot)

For each per-ULD-pivot arc `a ∈ C^{pu}` and each ULD type `u ∈ U_a`:

```
   Σ_{g ∈ G_a} η_{a,g,u}   ≤   N_{a,u}                                      (C.5)
```

`N_{a,u}` is the **contracted allotment** of type-`u` ULD positions the forwarder
holds on arc `a` (BSA contract data). Sums across consolidation groups — multiple
MAWBs sharing the same BSA contract share the allotment.

**No flight-level physical-capacity constraint.** The forwarder does not know
`W_f` and cannot plan against it. The airline's overbooking and offload decisions
are out of the forwarder's planning scope. The constraints that matter — and
that the forwarder *does* know — are:

1. Per-contract allotment `N_{a,u}` (this C.5),
2. Per-ULD physical capacity `W_u, V_u` (C.5b),
3. Per-offer caps `cap_a` / `cap_a^{cl}` where the offer specifies one (C.5c).

If a TACT/NAC offer specifies no cap (subject-to-availability at booking), the
MILP treats it as uncapped — capacity check happens at downstream booking, not
in planning.

### C.5b Per-ULD physical capacity (per_uld_pivot offers — **item 13-A bug fix**)

For each MAWB `(a, g)` with `a ∈ C^{pu}` (per-ULD-pivot offer):

```
   Σ_{k ∈ K_a : g(k) = g} w_k · x_{k,a}   ≤   Σ_{u ∈ U_a} W_u · η_{a,g,u}   (C.5b-w)
   Σ_{k ∈ K_a : g(k) = g} v_k · x_{k,a}   ≤   Σ_{u ∈ U_a} V_u · η_{a,g,u}   (C.5b-v)
```

**Critical:** uses **`w_k` (actual mass), not `cw_k` (chargeable)** — `W_u` is a
physical payload limit. The prior model used `cw_k` here, double-counting
light-bulky cargo against both volume and weight. Worked LD3 case:
`W_u ≈ 1588 kg`, `V_u ≈ 4.5 m³`; dense 1400 kg / 1 m³ + light 30 kg / 3 m³ →
fits physically (actual 1430 kg, volume 4 m³), but old-form `cw` sum
`1400 + 501 = 1901 > 1588` wrongly rejects. **Volume bound C.5b-v is
independent** — light/bulky cargo (e.g.\ apparel ~80 kg/m³) binds on volume
long before weight; a 400 kg apparel HAWB at 5 m³ needs 2 LD3 by volume even
though weight says 1.

**Accepted looseness (no per-HAWB-to-ULD-instance assignment in MVP).** C.5b is
a per-MAWB aggregate bound on the total weight / volume against the *sum* of
ULD-type capacities. It does **not** enforce that each individual HAWB fits
inside one ULD — e.g.\ a single 3000 kg HAWB on `η = 2` LD3 satisfies
`3000 ≤ 2·1588 = 3176` even though no LD3 individually holds it. True bin-
packing (HAWB-to-ULD-instance assignment via `y_{k,u,i}` binaries) is deferred.
The §4 pre-filter step 8 catches the pathological per-HAWB case (any single
HAWB exceeding `max_u W_u` or `max_u V_u` on a per-ULD-pivot arc), which is
the failure mode worth blocking at MVP.

### C.5c Per-offer cap (when specified)

For each arc `a` with a specified offer-level cap `cap_a` (or `cap_a^{cl}` for
co-load):

```
   Σ_{k ∈ K_a} w_k · x_{k,a}   ≤   cap_a                                    (C.5c)
```

Applies wherever the catalog offer carries a cap (BSA secondary cap, GSA tier
limit, co-loader quote). Offers without a specified cap (typical TACT/NAC,
typical informal co-load) skip C.5c — capacity is request/confirm at booking,
not a planning-time constraint.

### C.6 Time propagation

For each HAWB `k ∈ K` and each arc `a ∈ A_k`:

```
   t_k^{head(a)}  ≥  t_k^{tail(a)} + transit(k, a) − M^{C.6}_{k,a} · (1 − x_{k,a})    (C.6)
```

where `transit(k, a)` is defined in §2.1 (ground arcs: `δ_a + δ^{cust}_k ·
𝟙[a ∈ A^{cust}]`; air arcs: `μ_a`).

Initial condition (cargo-ready window): `t_k^{O_k} ≥ t_k^{rdy,early}`. Optional
upper `t_k^{O_k} ≤ t_k^{rdy,late}` (release-window constraint) deferred — MVP
uses only the lower bound.

**MVP — deterministic times.** `transit(k, a)` is a point estimate. Finding S
Change 1 hook: in a later phase, `transit(k, a)` for air arcs is replaced by
the **P85–P90 quantile** from the Transit Time Service. MVP stays deterministic;
spec calls out the hook as a future point of integration, not a current
constraint change.

`M^{C.6}_{k,a}` is per-shipment tight; see §8.

### C.7 — REMOVED (Session-15 critique, Cluster J)

C.7 was the hub-MCT constraint family. The O-D-arc graph absorbs **all** hub
MCT into per-arc scalars at graph-build time:
- **Same-MAWB through-connection** (same carrier, multi-leg, or interline under
  one AWB) — internal MCT folds into `μ_a` of the single MAWB-arc.
- **Cross-MAWB transition** at a forwarder-operated hub (`CFS-H`) — handled by
  the deconsolidation-dwell arc's `δ_a`.
- **Cross-carrier re-tendering at a hub without `CFS-H`** — graph generator
  emits a synthetic carrier-side connection dwell arc carrying the MCT in
  `δ_a`.

In all three cases, C.6 alone enforces the timing via ground-arc / air-arc
time propagation. No separate hub-MCT constraint family is needed in the MILP.

### C.8 — REMOVED (duplicate of C.6 initial condition)

C.8 was `t_k^{O_k} ≥ t_k^{rdy,early}`. C.6 already states this as an initial
condition. Dropped to avoid duplication.

### C.9 Cargo cutoff at POL

For each HAWB `k`, each air arc `a ∈ A_k ∩ (A^{MAWB} ∪ A^{coload})` whose
`tail(a)` is a POL node:

```
   t_k^{tail(a)}  ≤  CO_a^*  +  M^{C.9}_{k,a} · (1 − x_{k,a})               (C.9)
```

`CO_a^*` is the precomputed effective cutoff for the first physical leg of arc
`a` (§3.3 — max over DCO, AMS, ICS2, ACI, minus per-HAWB prep time, folded into
a per-arc scalar at graph build). `M^{C.9}_{k,a}` is per-shipment tight; see §8.

### C.10 Soft deadline + tardiness (item 3)

For each HAWB `k`:

```
   τ_k  ≥  t_k^{D_k^{node}} − Δ_k                                           (C.10)
   τ_k  ≥  0  (domain, C.14)
```

with `Δ_k = min(T_k^{dead}, T_k^{SLA})` (§3.1). Tardiness enters the objective
linearly (§7). **No quadratic penalty in MVP.** (Quadratic = deferred; tangent-cut
spec preserved in `air_review_notes.md` item 3 for future-self.)

**Finding S Ch 1 framing.** When `T_k^{SLA}` derives from a service product,
this is a *planning bound*, not a contractual guarantee. The clock binds against
deterministic `t_k^{D_k^{node}}` in MVP; later phase will swap for the TT-Service
P85–P90 quantile (hook noted, not implemented).

### C.11 Hard backstop `T_k^{abs}`

```
   t_k^{D_k^{node}}  ≤  T_k^{abs}                                           (C.11)
```

This is the only hard time bound. PER cargo gets a finite `T_k^{abs}`; GEN gets
`+∞`. Beyond it → genuine infeasibility (rescue event).

### C.12 Locked commitments — handled by preprocessing, not by MILP constraints

Locks are **not** modeled as MILP constraints. They are resolved at the
preprocessing layer before the MILP is built:

- **Fully locked HAWB** (entire route committed; cargo may or may not have left
  origin): preprocess **out** of `K` entirely. The HAWB does not appear in the
  MILP. Its committed cost is recorded for reporting / accounting but the MILP
  has no decision to make about it.
- **Partially locked HAWB** (some upstream legs executed, downstream open):
  HAWB enters `K` with three rewrites:
  1. **Origin re-pointed** to the current observed position (the node the
     cargo currently sits at — e.g.\ `CFS-H` at a hub mid-journey).
  2. **Subgraph truncated** to forward arcs only (`A_k` excludes any arc
     wholly upstream of the current position).
  3. **Initial time** `t_k^{O_k}` is fixed to the observed arrival time at the
     current node (replaces the cargo-ready window C.8 for partial-locked
     HAWBs).
  From there, standard MILP. No special constraints, no `b_k`, no buyout
  decision variable.
- **Lock break** is an **orchestrator decision between MILP runs**, not a
  decision the MILP makes. If the orchestrator decides to break a lock
  (weighing buyout cost against predicted improvement), the HAWB is added back
  to the *next* routing instance with no lock — same mechanism as
  preprocessing-out, but with the lock cleared first.

**Pre-MILP feasibility check.** If the current locked state has no feasible
forward path (e.g.\ committed connection misses the new deadline), the HAWB is
surfaced as a **structured rescue event** to the orchestrator. The orchestrator
then decides: accept tardiness (do nothing — MILP runs with the HAWB and C.10
absorbs the slip), break the lock (re-add HAWB unlocked), or escalate.

The MILP itself is **lock-agnostic**.

### C.13 BSA settlement

#### C.13a Equalized-settlement allowance

For each equalized-settlement contract `c ∈ C^{eq}` with allowance `A_c`:

```
   over_c  ≥  chargeable(c) − A_c                                           (C.13a)
   over_c  ≥  0  (domain, C.14)

   where chargeable(c) := Σ_{(a, g) : a ∈ A_c^{MAWB}, g ∈ G_a} CW_{a,g}
```

Cost contribution = `r_c · over_c` (free up to `A_c`, then `r_c`/kg). The sunk
portion `r_c · A_c` is constant (booked outside the MILP, doesn't affect
argmin) — dropped from the objective for tractability; surfaced in cost
reporting separately.

#### C.13b Per-flight settlement pivot

For each MAWB `(a, g) ∈ M` with `a ∈ C^{pu}` and `settlement_a = per_flight`:

```
   pivot_{a,g}  ≥  CW_{a,g}                                                 (C.13b-1)
   pivot_{a,g}  ≥  π_a · Σ_{u ∈ U_a} η_{a,g,u}                              (C.13b-2)
```

Cost contribution = `r_a · pivot_{a,g}`.

### C.14 Domain (with explicit upper bounds where they tighten the LP)

```
   x_{k,a}      ∈  {0, 1}                              ∀ k ∈ K, a ∈ A_k
   z_{a,g}      ∈  {0, 1}                              ∀ (a, g) ∈ M
   γ_{a,g,b}    ∈  {0, 1}                              ∀ (a, g) ∈ M, a ∈ A^{MFB}, b ∈ B_a
   η_{a,g,u}    ∈  ℤ ∩ [0, N_{a,u}]                    ∀ (a, g) ∈ M, a ∈ C^{pu}, u ∈ U_a
                                                       (per-MAWB cap — tighter than aggregate C.5)
   CW_{a,g}     ∈  [0, CW^{ub}_{a,g} · z_{a,g}]        ∀ (a, g) ∈ M
                                                       (upper-link: empty bucket ⇒ CW = 0)
   Wt_{a,g}, Wv_{a,g}, BW_{a,g,b}  ∈  [0, CW^{ub}_{a,g}]
   pivot_{a,g}  ∈  [0, max(CW^{ub}_{a,g}, π_a · Σ_u N_{a,u})]               ∀ (a, g) ∈ M, a ∈ C^{pu}
   over_c       ∈  [0, Σ_{(a,g) : a ∈ A_c^{MAWB}, g ∈ G_a} CW^{ub}_{a,g}]   ∀ c ∈ C^{eq}
   τ_k          ∈  [0, max(0, T_k^{abs} − Δ_k)]                             ∀ k ∈ K
   t_k^n        ∈  [t_k^{rdy,early}, T_k^{abs}]                             ∀ k ∈ K, n ∈ N_k
```

where the per-MAWB **chargeable-weight upper bound** is precomputed at MILP build:

```
   CW^{ub}_{a,g}  =  (1 + ε) · Σ_{k ∈ K_a : g(k) = g} max(w_k, v_k · 167)
```

This is the tightest valid upper bound on `CW_{a,g}` from C.4a–d (sum of the
larger of actual and volumetric for each eligible HAWB, plus dunnage). All
big-M values in C.6, C.9, and §7.2 use a scalar derived from `CW^{ub}_{a,g}`
or the per-shipment horizon (see §8).

---

## 7. Objective

Minimize **total cost + tardiness penalty**:

```
   min   Σ_{(a,g) ∈ M} cost^{MAWB}_{a,g}                                    [bucket cost on MAWBs]
       + Σ_{a ∈ A^{coload}} Σ_{k ∈ K_a} m_a^{cl} · cw_k · x_{k,a}           [co-load per-kg cost]
       + Σ_{a ∈ A^{ground}, k ∈ K_a} (c_a^{flat} + c_a^{kg} · w_k) · x_{k,a}   [ground handling / cartage]
       + Σ_{c ∈ C^{eq}} r_c · over_c                                        [BSA overage on equalized contracts]
       + Σ_{k ∈ K} w_{sp(k)} · τ_k                                          [linear tardiness, item 3]
```

**Monotonicity invariant.** All five cost contributors have non-negative
coefficients on `CW_{a,g}` (and on `Wt`, `Wv`, `pivot`, `over`, `τ`). This is
the correctness condition that lets C.4c/C.4d be inequalities (`CW ≥ Wt, CW ≥
Wv`) rather than the explicit max — minimization drives `CW` down to
`max(Wt, Wv)` exactly. If any future rate family introduces a negative-
coefficient term (rebates, marketing credits), C.4c/C.4d **must** be tightened
to equality via a PWL formulation; the current spec does not permit such terms.

The bucket cost `cost^{MAWB}_{a,g}` is dispatched on `rate_family_a`:

### 7.1 `flat_rate`

```
   cost^{MAWB}_{a,g}  =  max( min_chg_a · z_{a,g} ,  m_a · CW_{a,g} )
```

Linearized: introduce auxiliary `c_{a,g} ≥ 0` with
`c_{a,g} ≥ min_chg_a · z_{a,g}`, `c_{a,g} ≥ m_a · CW_{a,g}`; minimize `c_{a,g}`.

### 7.2 `min_flat_breaks` (TACT — item 7)

IATA next-break-down: `cost = min_b rate_{a,b} · max(CW_{a,g}, break_{a,b})`.
The "round up to a higher break for a lower rate" case **must** be representable
(it is the whole point of break-down rating). Linearized as:

```
   Σ_b γ_{a,g,b}  =  z_{a,g}                                        (exactly-one IF active)

   BW_{a,g,b}  ≤  M^{BW}_{a,g} · γ_{a,g,b}                          ∀ b   (force BW=0 when γ=0)
   BW_{a,g,b}  ≥  break_{a,b} · γ_{a,g,b}                           ∀ b   (when γ=1, BW ≥ break)
   BW_{a,g,b}  ≥  CW_{a,g} − M^{BW}_{a,g} · (1 − γ_{a,g,b})         ∀ b   (when γ=1, BW ≥ CW)

   cost^{MAWB}_{a,g}  =  Σ_b rate_{a,b} · BW_{a,g,b}
```

Three inequalities on `BW_b` (not four). **No `BW_b ≤ CW`** — that constraint
would force `BW_{b*} = CW` for the chosen break, combined with `BW_{b*} ≥
break_{b*}` would then make any selection with `break_{b*} > CW` infeasible,
banning the round-up-to-higher-break case. Without it: `BW_b = max(CW, break_b)`
when `γ_b = 1`, `BW_b = 0` when `γ_b = 0`, and minimization over the break
binaries naturally recovers `min_b rate_b · max(CW, break_b)` = IATA cost.

Worked check. 90 kg shipment, breaks `(b=45, rate=$10/kg)` and `(b=100, rate=$8/kg)`:
- `γ_{45}=1`: `BW_{45} = max(90, 45) = 90`. Cost = `10·90 = $900`.
- `γ_{100}=1`: `BW_{100} = max(90, 100) = 100`. Cost = `8·100 = $800`.
- Min over breaks: `$800` — matches IATA (round up to b=100 for the cheaper rate).

The exactly-one-IF-active form (`Σγ = z`, not `Σγ = 1`) handles empty buckets:
`z = 0` → all `γ_b = 0` → all `BW_b = 0` → cost contrib = 0.

`M^{BW}_{a,g}` is the per-MAWB-tight big-M; see §8.

### 7.3 `per_uld_pivot` (BSA)

Per-flight settlement (C.13b):

```
   cost^{MAWB}_{a,g}  =  r_a · pivot_{a,g}
```

Equalized settlement: in-MAWB cost contribution is *just* `r_a · CW_{a,g}` for
the chargeable accumulator (rolled into `over_c` via C.13a). The pivot floor is
handled at the contract level by `A_c`, not per-MAWB. (Per-flight ULD physical
capacity C.5b still applies.)

### 7.4 Per-shipment cost term notes

- **No min-charge by co-load HAWB.** Co-load arcs are per-kg, no minimum.
  Co-loader's minimum (if any) is folded into `m_a^{cl}` at catalog time.
- **Surcharges (Path-A / Path-B per `data_model.md §6`).** Path-A per-arc
  additive surcharges enter via `c_a^{handle}`. Path-B flight-level surcharges
  enter as an additional objective term per arc indicator (deferred from this
  spec — keep terms in the LaTeX rewrite per the existing model §6.7).
- **No `P.18` budget cap.** Removed entirely (item 15). All cost minimization
  via the objective; budget is a quoting-layer concern, not a routing constraint.

---

## 8. Linearization summary

| Mechanism | Trigger | Technique |
|---|---|---|
| `CW = max(Wt, Wv)` | All MAWBs | Two ≥ inequalities (C.4c, C.4d). Correctness via monotonicity invariant (§7). |
| `cost = max(min_chg_a · z, m_a · CW)` | `flat_rate` | Aux `c ≥` both. Empty-bucket: `z=0 ⇒ CW=0` (via C.14 upper-link) ⇒ `c=0`. |
| `cost = min_b rate_b · max(CW, break_b)` | `min_flat_breaks` | Binary `γ_b` exactly-one IF active (`Σγ = z`) + **3-inequality** disaggregation on `BW_b` (no `BW ≤ CW` — that would forbid round-up-to-higher-break). Verified §7.2 worked example. |
| `pivot = max(CW, π · Ση)` | `per_uld_pivot`, per-flight | Two ≥ inequalities (C.13b). |
| `over = max(0, chargeable − A)` | `per_uld_pivot`, equalized | One ≥ + non-negativity (C.13a). |
| `τ = max(0, arrival − Δ)` | All HAWBs (soft deadline) | One ≥ + non-negativity (C.10). |
| Time propagation with arc-choice | C.6, C.9 | Standard big-M `M · (1 − x)`. Per-constraint tight M — see §8.1. |
| MAWB activation | C.2 | `Σx`-bounded; no auxiliary linearization. |
| Bucket aggregate upper-link | C.14 | `CW ≤ CW^{ub} · z` ties bucket cost vars to MAWB activation; tightens LP. |

### 8.1 Big-M values (per-constraint tight)

Use these in place of a global `M`. All values precomputed at MILP build.

| Constraint | Formula | Rationale |
|---|---|---|
| `M^{C.6}_{k,a}` (time propagation) | `T_k^{abs} − t_k^{rdy,early}` | Tightest horizon-bounded value per shipment. When `x_{k,a} = 0`, the RHS slack absorbs the longest possible time-of-day difference between any two nodes the shipment could touch. |
| `M^{C.9}_{k,a}` (cutoff at POL) | `T_k^{abs} − t_k^{rdy,early}` | Same envelope as C.6 (the only time-of-day on either side of the inequality is bounded by the shipment's horizon). |
| `M^{BW}_{a,g}` (TACT break disaggregation) | `CW^{ub}_{a,g}` (from C.14) | Per-MAWB tight: `BW_b ≤ M · γ_b` and `BW_b ≥ CW − M(1−γ_b)` both use the same scalar bound on `BW`. |

Per-shipment scaling beats global scaling by 1–2 orders of magnitude on
realistic horizons; LP-relaxation quality of C.6 is the typical bottleneck and
this tightening directly attacks it. Further tightening (per-arc shortest-path
bounds on the time horizon) is a P1 optimization, not MVP.

**Anticipated LP-relaxation slack on C.6.** Even with per-shipment tight M,
fractional `x_{k,a}` can let `τ_k = 0` for a shipment whose integer-feasible
path is genuinely tardy. This is an inherent weakness of big-M time propagation
and is accepted for MVP; the heavy fix (time-indexed formulation, per-arc dual
time vars) is a P1 if branching becomes slow.

---

## 9. Open design questions — RESOLVED

All five are closed. Decisions recorded below for traceability.

**Q1. Flight-level physical capacity — DROPPED entirely.** The forwarder does
not know `W_f` and has no operational basis to plan against it. The realized
flight-level capacity is the airline's overbooking / offload concern, not the
forwarder's. The constraints that matter at planning time are per-contract
allotment `N_{a,u}` (C.5), per-ULD physical limits `W_u, V_u` (C.5b), and
per-offer caps where the offer specifies one (C.5c). Co-load arcs follow the
same rule — if `cap_a^{cl}` is specified by the co-loader, it gates; if not, the
MILP treats the co-load offer as uncapped at planning time and the booking layer
handles request/confirm.

**Q2. Per-ULD volume bound — KEPT.** Light/bulky cargo binds on volume long
before weight. Worked apparel case (80 kg/m³, 400 kg → 5 m³): a single LD3
holds it by weight (`W_u ≈ 1588`) but not by volume (`V_u ≈ 4.5`). Both bounds
needed. C.5b-w and C.5b-v both retained.

**Q3. Surcharges — deferred to LaTeX rewrite.** Path-A enters via `c_a^{handle}`
(per-arc additive), Path-B as a per-flight indicator term. Math unchanged from
the prior LaTeX §6.7; no re-derivation here. Schema in `data_model.md §6`.

**Q4. In-transit hub customs — out of modeling scope, in data scope.** The time
impact of in-transit declarations (T&E, EU summary declaration, etc.) is an
additive component of the existing deconsolidation-dwell arc's `δ_a`. The graph
generator computes it from hub type + cargo type + in-transit flag. No new arc
type, no new constraints in the MILP.

**Q5. Lock-buyout — handled at orchestrator layer, not MILP.** See C.12. Locked
HAWBs preprocess out (full lock) or enter with truncated subgraph + initial
conditions (partial lock). Lock-break is the orchestrator's call between MILP
runs. No `b_k`, no buyout decision variable in the MILP.

---

## 10. Deferred (P1)

Recorded as Open Items for the LaTeX rewrite — **not** in MVP:

- **`tt-quantile-binding`** — bind `t_k^{D_k^{node}}` against TT-Service P85–P90 quantile instead of deterministic times (Finding S Ch 1 hook).
- **`offload-priority`** — Finding S Change 2: `supply_class ∈ {confirmed, best_effort}` per-offer attribute + `confirmed_only` product attribute + new subgraph pre-filter predicate (express ⟹ confirmed-capacity options only).
- **`sla-endpoint-and-hops`** — Finding S Change 3: A2A/D2D SLA-endpoint attribute on service products + `max_hops` / `direct_only` product attributes.
- **`quadratic-tardiness`** — promote `+ Σ w · τ_k²` (PWL-linearized via convex tangent cuts) if linear behavior is unsatisfactory after calibration.
- **`split-shipment`** — one HAWB across two flights/MAWBs (capacity / partial-roll). Option 1 (virtual sub-HAWBs at intake) preferred (graph doc §8).
- **`pairwise-dg-segregation`** — fine-grained pairwise DG segregation at the ULD layer (graph doc §8).
- **`temp-band-refinement`** — temperature-band refinement within `PER` (e.g.\ pharma sub-bands).
- **`per-flight-fx-pinning`** — per-run FX pinning for audit reproducibility (item 2).
- **`multi-seg-pu-pwl`** — multi-tier per-ULD over-pivot rate structure (currently "contingency only, not assumed standard"; item 7 amendment).

---

## 11. Excluded from MVP (not deferred — out of scope or handled outside MILP)

- **Flight-level physical capacity (`W_f`, `V_f`).** Not knowable by the forwarder; airline-side overbooking/offload concern. See Q1.
- **Hub-MCT as a MILP constraint family.** Removed in Session-15 critique. All hub MCT is absorbed into pre-computed per-arc scalars at graph build (`μ_a` for same-MAWB through-connection; `δ_a` for cross-MAWB transitions). C.6 time propagation alone suffices. See C.7 removal note.
- **Lock-buyout decision in MILP.** Orchestrator decides between MILP runs; locked HAWBs preprocess in/out. See C.12 + Q5.
- **In-transit hub customs as a graph element.** Data-only; folded into `δ_a` on the deconsolidation-dwell arc. See Q4.
- Per-HAWB cost attribution under consolidation (cost is per-MAWB by construction; allocate downstream if needed for billing).
- Multiple MAWBs on one `(arc, g)` (concave rate → optimizer never wants this; document-splitting is post-optimization output).
- Charter, broker-of-record, alliance slot sharing.
- AWB stock management.
- Per-flight lithium aggregate (carrier-side responsibility per Session-11 triage).
- CFS storage / demurrage cost (deferred research note, P1).

---

## 12. Mapping from prior LaTeX

For traceability during the rewrite. Items removed are noted.

| Prior `P.x` | This spec | Note |
|---|---|---|
| P.1 flow conservation | C.1 | Same; reindexed on `A_k`. Sign convention = standard MCNF supply form. |
| P.2 ULD volume cap | C.5b-v | Per-ULD only; flight-level dropped (Q1). |
| P.3 ULD weight cap | C.5b-w | **Item 13-A bug fix: `cw_k` → `w_k`.** |
| P.4 per-flight contract cap | C.5 | Reframed: per-arc contracted allotment `Σ_g η_{a,g,u} ≤ N_{a,u}` on ULD positions. No flight-level coupling (Q1). |
| P.5 aircraft position cap | C.5 + C.14 | Aggregate cap in C.5; per-MAWB bound `η_{a,g,u} ≤ N_{a,u}` in C.14 domain. |
| P.6 period cap | C.13a | Equalized BSA via `A_c` (per-solve sunk allowance). |
| P.7 hard BSA take-or-pay min | C.13a (equalized) / C.13b (per-flight retained) | **Hard period-minimum removed for equalized contracts** (replaced by `A_c` allowance mechanism). **Per-flight take-or-pay retained** via C.13b pivot floor `pivot ≥ π·Ση`. |
| P.8 arc-to-supply-option linkage | C.2 | Reframed for `(arc, g)`. (C.3 was redundant duplicate of C.2a — removed.) |
| P.9 supply-option exclusivity | implicit | Each arc has one offer (`rate_family_a`); exclusivity is at arc-choice level via C.1. |
| P.10 pivot weight linearization | C.13b + §8.1 | Per-flight settlement only. |
| P.11 cargo cutoff | C.9 | Same; `CO_a^*` precomputed per-arc scalar (was per-flight). |
| P.12 time propagation | C.6 | Same; per-shipment tight big-M (§8.1). |
| P.13 deadline (hard) | C.11 | Now `T_k^{abs}` only; soft via C.10. |
| P.14 hub MCT | — | **Removed.** All hub MCT absorbed into `μ_a` (same-MAWB through-connection) or deconsol-dwell arc `δ_a` (cross-MAWB) at graph build. No MILP constraint family (Cluster J). |
| P.15 deadline (now soft) | C.10 | Was hard; item 3. Tardiness `τ_k` against `Δ_k`. |
| P.16 cargo type compat | pre-filter §4 step 2 | Out of MILP body. |
| P.17 ULD physical fit | pre-filter §4 steps 2 + 8, plus C.5b | Pre-filter for dim fit and per-HAWB ULD overflow; C.5b for per-MAWB aggregate enforcement. |
| **P.18 budget cap** | — | **Removed entirely (item 15).** |
| P.19 locked commitments | C.12 (preprocessing) | **Not a MILP constraint** — preprocessing layer (Q5). |
| P.20 SLA (now soft) | C.10 | Was hard; item 3 + Finding S Ch 1. |
| P.21 domain + initial conditions | C.14 + C.6 init | Extended with `τ_k`, `CW`, `BW`, `γ`, `η`, `over`, and upper-link bounds. |

---

## 13. Tractability notes (refreshed Session 15 — per Agent 3 critique)

Three prior items are stale under this spec and are replaced with refreshed
forms:

- **`scale-hawb-aggregation`** (replaces `scale-y-aggregation`). At Phase-2
  load (`|K| = 500–1500`), many HAWBs share identical
  `(g, O_k, D_k^{node}, deadline tier, sp(k))` — fully interchangeable from
  the MILP's standpoint. Treat each equivalence class as a super-shipment with
  combined `w_k`, `v_k`. The §10 deferred `consolidation_mode = preprocess`
  toggle is the implementation home; **promote to walking-skeleton-instrumented**
  via the `aggregation_potential` shadow metric (§13.1) before deciding
  default-on.
- **`scale-bucket-dominance`** (replaces `scale-option-dominance`). Pre-MILP,
  drop arc `a' ∈ A^{MAWB}` if `∃ a ∈ A^{MAWB}` with same `G_a`, same O-D,
  `μ_a ≤ μ_{a'}`, `cap_a ≥ cap_{a'}`, and `rate_a(w) ≤ rate_{a'}(w)` for all
  feasible `w`. Hard pre-filter step; estimate 20–40% offer pruning at MVP.
- **`strat-v2-mawb-rescale`** — **deleted.** Assumed a future `h_{k,m}` MAWB
  restructure that the O-D-arc-graph + bucket already *is*. Obsolete.

### 13.1 Walking-skeleton instrumentation

Measure these on every solve; output to structured log files alongside
solutions. They answer the scale questions empirically before they bite.

| Metric | Output | Purpose |
|---|---|---|
| `|A_k|` histogram + per-predicate drop counts | `pre_filter_stats.jsonl` per HAWB | Determines `x_{k,a}` binary count. The single most important number. |
| `|G_a|` distribution + activated bucket count `Σ z_{a,g}` | `bucket_stats.jsonl` per solve | Drives `|M|` and `γ` count estimates. |
| LP-vs-MIP gap by constraint family | `lp_gap_breakdown.json` | Identifies which family loosens the LP relaxation most. |
| Realized big-M slack on C.6, C.9 | `bigm_slack_warning` if median > 0.5·M | Catches loose M values empirically. |
| Connected components of H (HAWB-sharing-arc graph) — **shadow** | `component_decomposition_shadow.jsonl` | Indicator for when decomposition becomes mandatory. |
| Super-shipment equivalence classes — **shadow** | `aggregation_potential = 1 − |classes|/|K|` | Decides default for `consolidation_mode`. |
| BSA-contract cross-coupling fraction | `bsa_coupling_fraction` | Predicts Lagrangian-relaxation value for Phase-2 decomposition. |
| Post-solve invariant assertions | `CW = max(Wt, Wv)`; `τ_k = max(0, lateness)`; `Σ_g η ≤ N`; no `z=1` with empty bucket | Catches silent bugs (especially monotonicity violations on future rate-family extensions). |
| HiGHS phase breakdown (presolve / LP root / B&B / cuts) | callback-derived `solve_phase_times.json` | Tells us where time actually goes. |

### 13.2 Base-scale estimate (concrete — Agent 3)

At MVP-production: `|K| = 100`, `|A| = 500`, `|A_k|_avg ≈ 15` (10% pre-filter
survival assumed; **to be measured**), `|A^{MAWB}| ≈ 250`, `|A^{MFB}| ≈ 80`,
`|C^{pu}| ≈ 40`, `|U_a|_avg ≈ 2.5`, `|G_a|_avg ≈ 1.5`, `|M| ≈ 200`.

- **Binaries:** `x_{k,a} ≈ 1,500`; `z_{a,g} ≈ 300`; `γ_{a,g,b} ≈ 720`. **Total ~2,500.**
- **Integers (general):** `η_{a,g,u} ≈ 150`, bounded `[0, N_{a,u}]` (typically `≤ 10`).
- **Continuous:** `t_k^n ≈ 2,500`; `CW + Wt + Wv ≈ 900`; `BW ≈ 720`; `pivot + over + τ ≈ 250`. **Total ~4,400.**
- **Constraints:** ~10,000. C.6 time propagation is the densest (~1,500 rows, each big-M-coupled to one binary) — first-binding family for LP-relaxation quality.
- **Expected HiGHS solve time:** seconds to a few minutes single-component, well-tuned.
- **Phase-2 (`|K| = 500`):** ~12,500 binaries, ~50k constraints — minutes to ~1 hour monolithic. Decomposition mandatory before then.
- **Phase-2 (`|K| = 1500`):** monolithic infeasible without column generation (see `scalability.md`).

### 13.3 Walking-skeleton minimum-viable subset (Agent 3 #16)

The first runnable MILP does **not** need all 14 constraint families. Implement
the minimal subset that exercises core tractability questions:

- **In:** C.1 flow, C.2 MAWB activation, C.4 CW aggregation, C.5c per-offer cap,
  C.6 time propagation, C.9 cutoff, C.10 soft tardiness, C.11 hard backstop,
  C.14 domain. Rate families: `flat_rate` + `coload_per_kg` only.
- **Deferred to subsequent v2/v3:** `min_flat_breaks` (defers γ/BW family),
  `per_uld_pivot` (defers η/C.5/C.5b/C.13), C.12 preprocessing layer.

This is ~6 constraint families and ~3 variable families (`x`, `z`, `CW`/`Wt`/`Wv`,
`τ`, `t_k^n`). Tests `x`-binary scaling, C.6 density, LP-gap, and big-M tightness
without the per_uld_pivot tail. CONTEXT.md Stage 4 refines to:
- **v1:** `flat_rate` + `coload_per_kg`.
- **v2:** add `min_flat_breaks` (γ + BW + corrected linearization §7.2).
- **v3:** add `per_uld_pivot` (η + C.5 + C.5b + C.13).

---

## 14. What this spec does NOT yet contain

(To be added before the LaTeX rewrite if the critique demands.)

- Detailed surcharge math (Path-A / Path-B) — referenced, not specified.
- Carrier-policy cascade resolution mechanics — referenced (`data_model.md §4`),
  not re-spec'd here.
- Locked-commitment lifecycle state derivation — referenced.
- Embargo / lithium / screening predicate definitions — referenced.
- Service-product catalog schema — referenced.

These are **stable** in the prior LaTeX / `data_model.md`; they carry over
unchanged in the rewrite.

---

## 15. Open items to fold in from prior LaTeX

The prior LaTeX (`air_freight_routing.tex`) carries supporting material that
must migrate into the rewrite but is unchanged by this spec:

- §1 abstract / Problem Statement (rewrite to reference O-D-arc graph + MAWB-as-`(arc, g)`).
- §2 time-zone convention (UTC; no change).
- §6.1 cargo-ready window + per-HAWB prep time.
- §6.4 ULD specs and operational attributes.
- §6.6 ULD interchange set `Π` (interline through-ULD).
- §6.7 supply option catalog (now: arc attributes; same content).
- §6.12 screening certification.
- §6.13 locked commitments and execution state.
- §6.14 service products.
- §6.15 carrier policy and rules resolution.
- §Tractability and Scaling Roadmap (refresh per §13 above).

---

**End of spec v1.** Next step per CONTEXT.md: 3-agent critique pass on this
spec, then Stage 2 (LaTeX rewrite).
