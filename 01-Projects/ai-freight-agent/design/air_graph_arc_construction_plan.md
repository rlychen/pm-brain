# Air Graph — Arc Construction Plan (v2, post-critique)

**Status:** Revised after a four-agent adversarial critique (spec-fidelity, test-coverage, data-realism, architecture). Strategy approved; this v2 folds in the BLOCKING findings. Drives code in `src/components/air_graph.py`.
**Date:** 2026-05-31 (Session 23, post air-model approval).
**Upstream spec (approved):** `model/air_freight_routing.tex` (§3 graph construction, §arc-enumeration, §fwd-time-propagation, §prefilter, §fallback-arc) + `model/air_graph_construction.md` (§§2–10).

---

## Locked decisions (this session)

1. **Data doctrine:** real topology + carrier set + representative frequencies → calibrated-synthetic schedule → synthetic commercial. **Schedule is seeded from real carrier-published freighter timetables** (see §0).
2. **Build order:** arc-construction algorithm + A1–C7 unit fixtures first; TPEB realistic instance second.
3. **First instance scope:** 3-origin / 2-gateway / 5-carrier / HKG-CFS-H / 12-HAWB (corrected demand — §6).
4. **Graph data structure:** flat `dict[ArcId, Arc]` + adjacency index. **No networkx.** (Parallel arcs are native; the MILP iterates index sets; matches `air_milp.py`.)
5. **Arc identity:** deterministic structured arc IDs (per-offer for air arcs), so `x_{k,a}` and HiGHS variable ordering are stable.
6. **Predicate attribution order:** strict spec order **1→8**, log first failure.
7. **Doc home:** `docs/design/` working copy, vault mirror at sign-off.

---

## Critique resolution log (what changed from v1)

| # | Finding (agent) | Resolution |
|---|---|---|
| R1 | Fused propagation + predicate filter is a correctness hazard (D) | **Two passes.** Pass A settles time windows; Pass B runs predicates over settled values. |
| R2 | Propagation collapses to per-node; ETA pinning lost (A) | Air arc **pins head window to `[ETA_a, ETA_a]`**; labels are **per-(node, incoming-arc)**; union only for destination-reachability, never cutoff admission. |
| R3 | Dispatch-feasibility `λ^disp_k` dropped (A) | Predicate 6 also checks `t_k^{rdy,early} ≤ CO_a* − λ^disp_k`. |
| R4 | Predicate 1/7 fusion mis-attributes rejections (A, B) | **Split:** predicate 1 = pure BFS both ends (no time bound); predicate 7 = `≤ T_k^abs` destination reachability inside the propagation step. |
| R5 | Per-leg quantification `∀f∈legs(a)` lost for predicates 2/3/4 (A) | Predicates 2/3/4 stated as per-leg conjunctions over `legs(a)`. |
| R6 | `carries_mawb = kind != COLOAD` proxy (A) | `carries_mawb := rate_family ∈ {flat_rate, min_flat_breaks, per_uld_pivot}`. |
| R7 | P5/P6 "stays within one MAWB" guard is dead code (A) | Through/same-carrier arcs carry the hub **internally** (no graph node, MCT in `μ_a`); they never enter the (inbound,outbound) join loop. The join loop runs only over arcs terminating at a hub graph node; every such meet is a MAWB boundary → P5 if CFS-H else P6. |
| R8 | Missing scalars `STD_a`, `arr_dest`, fallback scalars (A) | `STD_a` carried + build-time assert `CO_a* ≤ STD_a ≤ ETA_a − μ_a`; `arr_dest(k,a)` on terminal arcs; fallback `transit = T_k^abs − t_rdy,early`, `arr_dest = T_k^abs`. |
| R9 | Graph structure undefined; networkx collapses parallel arcs (D) | Flat `dict[ArcId, Arc]` + adjacency index (decision #4). |
| R10 | Output contract + `flight_arcs` reverse map missing (D) | `AirGraph` result dataclass defined (§4), including `flight_arcs: {flight_id → {arc_id}}` for per-flight capacity coupling. Built by inversion inside `air_graph.py`. |
| R11 | structlog is dev-only but runtime logging specified (D) | Move structlog → runtime deps **in the slice that adds reject_record logging** (not the first slice). Sink to stderr/file, never stdout (FastMCP rule). reject_record is **returned in the contract** and logged. |
| R12 | Test matrix is single-form, count-only, has coverage holes (B) | Expanded matrix (§5): add predicate-7, multi-failure ordering, C7-split, two-groups-on-through, fallback-excluded-from-shared-MAWB, CFS-H-same-carrier-no-deconsol, full ground chain, per-HAWB customs; upgrade asserts to identity + rider-sets + `μ_a` composition; `prefilter_empty` asserts a structured warning object. |
| R13 | §0 falsely claims no free schedule exists (C) | Corrected §0: carrier-published freighter timetables ARE free; reframe synthesis justification. |
| R14 | §4.2 demand set incomplete (no O-D, no dims; dests JFK/SIN ≠ instance) (C) | §6 adds an explicit 12-HAWB O-D + weight/volume table over TPE/PVG/HKG→LAX/ORD, ≥3 routed through HKG CFS-H. |
| R15 | Supply mix misses A2/A3/through-vs-segment; ANC miscast (C) | §6 adds a CV multi-stop via ANC (A2 + realistic tech stop) and a CI same-carrier connection with a through rate alongside segment rates (A3 + overlap). |
| R16 | Synthetic rate levels unanchored (C) | §6 anchors levels to Baltic/TAC HKG→NA ($5–7/kg, 2026); BSA ~15–25% below spot. Matters at optimization stage, not construction. |

---

## 0. Data sourcing (corrected)

The **flight layer** (carrier, flight number, O-D, frequency, scheduled times) **is freely available** from carrier-published freighter timetables — e.g. Cathay Cargo (`cathaycargo.com/en-us/flight-schedule.html`, refreshed monthly) and China Airlines Cargo (`cargo.china-airlines.com/.../ScheduleDisplay.aspx`); EVA/Cargolux publish equivalents. What is **not** free: (a) a single aggregated machine-readable feed across carriers, (b) cargo-terminal cutoffs/MCTs, (c) the entire commercial layer (BSA allotments, pivots, rate cards). OpenSky is historical ADS-B position tracks only — useful for transit-time calibration, not forward schedules.

**Doctrine:** seed the synthesized schedule from the real published CI/CX/BR/CV freighter timetables for TPE/PVG/HKG→LAX/ORD; synthesize cutoffs/MCTs and the entire commercial layer.

---

## 1. Two components, not one

| Component | Input | Output | Tested by |
|---|---|---|---|
| **Arc construction** (`air_graph.py`) | a G(N,A) instance + HAWBs | `AirGraph` (subgraphs, MAWBs, scalars, reverse maps) | tiny deterministic A1–C7 fixtures |
| **Instance generator** (`data/synthetic/` + `tests/.../conftest.py`) | real topology + synthetic assumptions | a realistic TPEB G(N,A) | realism checks |

Algorithm + catalogue fixtures gate the component **first**; TPEB instance is integration **second**.

---

## 2. Physical (connecting) arc forms

| # | Arc form | MAWB? | Emitted when |
|---|---|---|---|
| P1 | Pickup `O→CFS-O` | no | per shipment |
| P2 | Origin CFS dwell | no | per origin gateway |
| P3 | Origin cartage `CFS-O→POL` | no | per origin gateway (≈0 if on-airport) |
| P4 | **Air arc** `POL→POD / POL→H / H→POD` | depends | per priced offer sub-segment |
| P5 | **Deconsolidation-dwell** at `CFS-H` | no | cross-MAWB hub transition, **only where forwarder has CFS-H** |
| P6 | **Carrier-side connection-dwell** at a hub *without* CFS-H | no | cross-carrier re-tender, MCT only |
| P7 | Destination cartage `POD→CFS-D` | no | per dest gateway |
| P8 | Destination CFS dwell | no | per dest gateway |
| P9 | Customs clearance dwell | no | per **HAWB** (`δ_cust_k`) |
| P10 | Final delivery `CFS-D→D` | no | per shipment |
| P11 | **Fallback** `O→D` | no | **always, per HAWB**; exempt from all predicates |

Through / same-carrier-connection arcs carry the hub **internally** (no graph node, MCT folded into `μ_a`) — they do not emit P5/P6. The (inbound,outbound) join loop runs only over arcs terminating at a hub *graph node*; every such meet is a MAWB boundary → P5 if CFS-H present, else P6 (R7).

---

## 3. MAWB logical-arc forms

**Axis A — physical realization of one MAWB-arc:** M1 direct (A1); M2 multi-stop, one flight # (A2); M3 same-carrier connection, 2+ flights/one AWB (A3); M4a interline through, alliance/Cargo-SPA (A4a); M4b interline through, bilateral SPA (A4b). M2–M4 are single through arcs; legs/hub are metadata.

**Axis B — emission on a multi-leg chain (overlapping emission, §arc-enumeration):** one arc **per priced offer sub-segment**; single-leg `a→b`,`b→c` and through `a→c` coexist iff the catalog publishes each. Never synthesize an unpublished through rate. 3-stop → up to 6 candidates.

**Axis C — `(arc, g)` MAWB instantiation (Phase 2):** C1 consolidation (N same-g → 1 MAWB); C3 two groups on same arc → 2 MAWBs; C4 VAL/HUM/AVI singleton; C5 co-load → no MAWB; C6/C7 membership changes across arcs (subset relationships normal). Group B = sequence-of-MAWBs (B1 separate contracts, B2 BSA-stitched, B3 consol→hub→deconsol→reforward, B4 offer change same carrier).

---

## 4. Architecture (locked)

**Arc store:** `dict[ArcId, Arc]` (flat, frozen dataclasses). **Adjacency index** `dict[NodeId, list[ArcId]]` (out-arcs) built once for the propagation sweep. No networkx.

**Identity:** `NodeId = str` structured (`AIRPORT:TPE`, `CFS:HKG:H`, `DOOR:k1:O`). Airport nodes are **role-agnostic shared nodes** (an airport may be a hub for one HAWB and a gateway for another). `ArcId = str` deterministic; air arcs keyed per-offer (`AIR:{offer_id}`) so parallel arcs are distinct. MAWBs key on `(arc_id, group_key)`.

**Arc dataclass:** one frozen `Arc` with an `ArcType` (StrEnum) discriminator; air-specific scalars grouped in an optional `air: AirScalars | None` (present iff `AIR`). Avoids 11 classes and avoids one fat all-optional blob.

**Output contract** (built inside `air_graph.py`, the explicit MILP interface):
```
@dataclass(frozen=True)
class AirGraph:
    arcs: dict[ArcId, Arc]
    subgraphs: dict[HawbId, frozenset[ArcId]]          # A_k
    riders: dict[ArcId, frozenset[HawbId]]             # K_a (inverted once)
    mawbs: dict[tuple[ArcId, GroupKey], Mawb]          # M
    flight_arcs: dict[FlightId, frozenset[ArcId]]      # per-flight capacity coupling (R10)
    rejections: list[RejectRecord]                     # diagnostics, returned AND logged
```

**Module layout:** module-level functions over immutable inputs (no stateful builder class), matching `air_milp.py`. Start flat `air_graph.py`; split to an `air_graph/` package (`physical.py`, `prefilter.py`, `overlay.py`, `types.py`) once it crosses ~300 lines. `group_key(hawb) → GroupKey` is a pure function defined once. Instance generator lives outside `src/components/`; A1–C7 fixtures in `tests/components/conftest.py`.

---

## 5. The algorithm

### Phase 1 — physical graph (consolidation-agnostic)

```
build_physical_graph(network, supply_catalog, hawbs, cfs_h_hubs) -> arcs, adjacency:
    add AIRPORT nodes; CFS:*:H only for cfs_h_hubs
    per HAWB: add DOOR:k:O, DOOR:k:D; per gateway: CFS:*:O, CFS:*:D (dedup)
    per gateway: emit P1,P2,P3,P7,P8,P9,P10 ground/dwell arcs
    # air arcs (P4) — one per priced offer sub-segment (overlapping emission)
    for offer o in supply_catalog:
        a = Arc(AIR, tail=o.origin, head=o.dest,
                carries_mawb = o.rate_family in MAWB_ELIGIBLE,          # R6
                flights = o.leg_flight_ids,
                air = AirScalars(mu = eta-std, std, eta, cutoff_raw))   # R8: ETA=STD+mu by construction
    # hub dwell (P5/P6) — join loop over arcs terminating at a hub GRAPH node only (R7)
    for hub node h, for (inbound air arc, outbound air arc) meeting at h:
        if h in cfs_h_hubs: emit deconsol-dwell(δ=break+rebuild)       # P5
        else:               emit connection-dwell(δ=MCT)               # P6
    flight_arcs = invert {flight_id: {arc_id}}                          # R10
```

### Phase 1 (cont.) — per-HAWB subgraph: TWO PASSES (R1)

```
build_hawb_subgraph(arcs, adjacency, k) -> A_k, rejections:
    # PASS A — settle time windows (order-dependent)
    per-(node, incoming-arc) labels [t_lo, t_hi] from O_k             # R2
    on each admitted AIR arc a: pin head window = [ETA_a, ETA_a]      # R2
    union windows only for destination-reachability checks           # R2
    # PASS B — predicate cascade over settled windows, strict 1→8, log first fail (dec.6, R4)
    for arc a reachable from O_k:
        1 lane reachability (pure BFS both ends, NO time bound)       # R4
        2 cargo_type_ok : ∀f∈legs(a)                                 # R5
        3 embargo_ok    : ∀f∈legs(a)                                 # R5
        4 lithium_ok    : ∀f∈legs(a)                                 # R5
        5 mode_ok ∧ carrier_ok ∧ ac_type_ok
        6 cutoff: t_lo[tail(a)] ≤ CO_a*  AND  t_rdy,early ≤ CO_a* − λ_disp   # R3
        7 destination reachability: ∃ path head(a)→D_k with arrival ≤ T_k^abs # R4
        8 per-HAWB ULD fit (a ∈ A^pu only)
        first failing → reject_record(k,a,idx); else A_k.add(a)
    A_k.add(fallback_arc(k))   # P11, exempt; transit=T_abs−t_rdy,early, arr_dest=T_abs (R8)
    if A_k \ {fallback} == ∅: emit structured warning object (not a log line)   # R12
```

### Phase 2 — MAWB overlay

```
overlay_mawbs(arcs, subgraphs) -> mawbs:
    g = { k: group_key(k) }
    for air arc a where a.carries_mawb:                               # R6
        riders = { k : a ∈ A_k }
        for val in distinct({ g[k] for k in riders }):
            mawbs[(a.arc_id, val)] = Mawb(arc=a.arc_id, group=val)
    # co-load + fallback arcs skipped — no MAWB
```

---

## 6. Test matrix (expanded — R12)

Fixtures = tiny deterministic A1–C7 cases in `conftest.py`. No solver, no realistic data. Assertions check **arc identity (tail/head/type)**, **MAWB rider sets**, and **`μ_a` composition**, not just counts.

**Construction (physical + overlay):** `A1_direct`, `A2_multistop` (1 arc, legs metadata, `μ_a` incl. MCT), `A3_same_carrier_conn`, `A4a_interline`, `A4b_bilateral_no_alliance` (split), `B1_two_carriers`, `B2_bsa_stitched`, `B3_hub_deconsol` (assert **P5** type + δ=break+rebuild), `B4_offer_change`, `C1_consolidation`, `C3_two_groups`, `C4_singleton`, `C5_coload` (0 MAWBs), **`C7_deconsol_divergence`** (4 arcs/3 MAWBs, membership sets), `overlap_emission` (3 candidates incl. through), `no_through_offer`, `through_only_no_segments`, `3stop_six_candidate`, `hub_no_cfsh` (**P6** not P5), **`cfsh_same_carrier_through_no_deconsol`**, `mixed_hubs_p5_and_p6_coexist`, **`two_groups_on_through_arc`**, **`fallback_hawb_excluded_from_shared_mawb`**, `coload_skipped_in_overlay`, `phase1_full_ground_chain` (P1–P10 by type set), `customs_per_hawb_not_shared`, `P3_P7_onairport_zero_offairport_nonzero`, `phase2_partition_property`.

**Pre-filter / subgraph:** `prefilter_predicate1_no_lane`, **`prefilter_predicate7_dest_unreachable`** (asserts index 7, not 1), `prefilter_each_predicate` (1–8), `prefilter_first_failure_wins_cargo_before_service`, `prefilter_first_failure_wins_reachability_before_cutoff`, `prefilter_cutoff_boundary_inclusive`, `dispatch_feasibility_lambda_disp`, `prefilter_empty` (structured warning object + fallback selected, no exception).

---

## 7. TPEB realistic instance (integration, after unit gate)

- **Nodes:** origins TPE, PVG, HKG; gateways LAX, ORD; **HKG = the one forwarder CFS-H hub.**
- **Carriers (real TPEB freighter ops):** TPE-origin offers on CI/BR; HKG-origin on CX/CV; ANC tech-stop multi-stop on CV; KZ reserved for a connection/interline role (or dropped to keep tight).
- **Schedule (~12–15 flights):** seeded from real published CI/CX/BR/CV freighter timetables; TPE/HKG→LAX direct ≈ 11–12h; **one CV multi-stop TPE/HKG-ANC-ORD (single flight #, ANC tech stop — A2)**; **one CI same-carrier connection TPE-HKG-LAX with a published through rate alongside segment rates (A3 + overlap)**.
- **Supply mix (synthetic):** BSA on CX HKG→LAX (allotment+pivot); TACT min-flat-breaks on CI; NAC/spot on BR; co-loader per-kg; one interline through-rate. Levels anchored to Baltic/TAC HKG→NA ($5–7/kg, 2026); BSA ~15–25% below spot.
- **Demand (12 HAWBs — explicit O-D + dims, R14):** the §4.2 commodity/`g`-group set, assigned origins across TPE/PVG/HKG and destinations across LAX/ORD, with **≥3 origin-diverse HAWBs routed through HKG CFS-H**, and realistic air densities (apparel ~100–150 kg/m³, electronics ~150–250, machine parts/gold high) so predicate 8 + chargeable-weight aggregation actually bind.

---

## First code slice (now)

Phase 1 **air-arc emission** with overlapping-emission policy + `flight_arcs` reverse map, and tests A1/A2/A3/overlap/no-through/coload/flight-arcs/malformed-input. Ground/dwell arcs, the two-pass subgraph, and Phase 2 are subsequent slices.
