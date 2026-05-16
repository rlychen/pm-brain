# Execution Plan
## AI Multimodal Freight Routing Agent

**Status:** Phase 0 — PRD in review, not yet approved
**Last updated:** 2026-05-16
**Rule:** This document is updated before any work begins on a new phase or component, and after any decision changes scope or sequence.

---

## Gate Rules (Non-Negotiable)

1. PRD must be approved before any LaTeX model is approved
2. Each LaTeX model must be approved before code for that component begins
3. Each component must pass its isolation tests before it is integrated with any other component
4. Each phase must be explicitly approved before the next phase begins
5. No code is written before the Execution Plan is reviewed and approved

---

## Phase 0 — PRD Review and Approval

**Status:** IN PROGRESS — review paused after agent_design.md §1.2 (decision confidence tiers)

**Gate to enter:** None.
**Gate to exit:** User explicitly approves PRD. All open questions either resolved or formally deferred.

### Remaining review checklist

- [ ] `agent_design.md` — §1.3 Guardrails framework through §3.7 (capability registry)
- [ ] `data_model.md` — full review including updated lifecycle states and ingestion layer
- [ ] `ui_spec.md` — full review; add subset selection screen
- [ ] `personas_and_tools.md` — full review; add ingestion and planning trigger tools
- [ ] `build_plan.md` — full review; ingestion API section needed
- [ ] `appendices/competitive.md` — spot-check against new research
- [ ] Adversarial critique — architecture, optimization model, UI, commercial
- [ ] Open questions resolved or formally deferred (PRD.md §6)
- [ ] **Formal approval** ← gate

### Known open issues before approval

| # | Issue | Status |
|---|---|---|
| Vessel capacity proxy | α = 0.20 in P.2 is a placeholder | Deferred; accepted for MVP |
| Time-phased capacity release | MVP commits all rem(s,t) in one run | Deferred to P1 |
| Firm-up trigger config | Feasible set threshold (alert at N sailings remaining, default N=1) + hours-before-CYC-of-current-soft-plan (per-tenant configurable). Service level does NOT govern this. | Designed — see data_model.md §3.5 |
| Ingestion schema | External system push API — what schema do we accept? How do we validate/dedup? | Open — needs design |
| LCL rate data | Use synthetic rates calibrated to market (TPEB LCL ≈ $60–120/CBM) for Phase 2; onboarding CSV upload for Phase 5. No licensed data needed for MVP. | Resolved |
| Air rate structure | Rate card per (lane, airline) at IATA weight breaks. Chargeable weight = max(actual_kg, volume_m³ × 167). No ULD buying for MVP. Capacity constraint: unconstrained (proxy). | Resolved — rate card, no ULD |
| Live AIS feed vendor | NOAA historical sufficient for Phase 2; production needs paid feed | Deferred; evaluate at Phase 5 |
| Pricing model | Per-decision vs. subscription; cost floor unknown until Phase 4 | Deferred |

---

## Phase 1 — Formal Models (LaTeX)

**Status:** NOT STARTED
**Gate to enter:** PRD approved.
**Gate to exit:** All required LaTeX models approved individually.

Models can be written and approved in any order. Code follows the component build sequence in Phase 2 — each model must be approved before its corresponding component begins.

### Model inventory

| Model | File | Status | Blocks |
|---|---|---|---|
| Ocean FCL Optimizer | `model/ocean_fcl_routing.tex` | Draft v2 — not yet approved | Phase 2.6 |
| Graph Generator | `model/graph_generator.tex` | Not started | Phase 2.1 |
| Ocean Transit Time Model | `model/ocean_transit_time.tex` | Not started | Phase 2.2 |
| Trucking Transit Time Model | `model/trucking_transit_time.tex` | Not started | Phase 2.3 |
| Air Transit Time Model | `model/air_transit_time.tex` | Not started | Phase 2.4 |
| Path-Level Transit Time Model | `model/path_transit_time.tex` | Not started | Phase 2.5 |
| Trucking Optimizer (FTL/PTL/LTL) | `model/trucking_routing.tex` | **Draft v1 — not yet approved (2026-05-16)** | Phase 2.7 |
| Ocean LCL Optimizer | `model/ocean_lcl_routing.tex` | **Draft v1 — not yet approved (2026-05-16)** | Phase 2.8 |
| Air Optimizer | `model/air_freight_routing.tex` | **Draft v1 — not yet approved (2026-05-16)** | Phase 2.9 |
| Destination Leg Planner | `model/destination_leg_planner.tex` | Not started | Phase 2.10 |
| Rules Engine | `model/rules_engine.md` | Not started — design doc, not LaTeX | Phase 2.11 |

### Notes

- `ocean_fcl_routing.tex` has 7 open items; item 7 (vessel capacity proxy α) is accepted as a placeholder. All others must be resolved before approval.
- The Rules Engine spec is a structured design doc (`model/rules_engine.md`), not LaTeX. It specifies every rule type, evaluation order, and conflict resolution. Same approval gate applies.
- Path-Level Transit Time Model: specifies how per-leg distributions are aggregated across a multi-modal route. MVP approach: Monte Carlo simulation over independent per-leg distributions. The model spec must address correlation structure between legs (e.g., if ocean is delayed, port dwell often increases too).
- LCL Optimizer model: **Draft v1 complete (2026-05-16)** — `model/ocean_lcl_routing.tex`. Joint bin-packing × routing MILP on 6-layer LCL graph (O, CFS-O, POL, POD, CFS-D, D). 16 constraints (P.1–P.16): flow conservation, shipment-to-container assignment, container volume+weight capacity, one-type-per-slot, sailing container capacity, arc-to-sailing linkage, hazmat pairwise co-loading exclusion, CFS cutoff, time propagation, deadline, cargo type compat, piece dimension fit, budget, domain. **Sequential decomposition solution strategy documented (§7)**: Stage 1 routing (aggregate capacity), Stage 2 bin-packing per sailing, with Benders-style feasibility cuts. Joint MILP for |K|≤50; decomposition for |K|>50. 10 deferred items.
- Air Optimizer model: **Draft v1 complete (2026-05-16)** — `model/air_freight_routing.tex`. Binary Multi-Commodity Flow on commodity-specific subgraph. Models two air capacity layers (BSA contract + spot rate-card), pivot weight take-or-pay, through-ULD vs. re-ULD at hubs, IATA weight breaks. 19 constraints (P.1–P.19), linearization details, 10 deferred items. Pending review and approval.
- Destination Leg Planner: when main leg (ocean/air) is firm_planned and AIS ETA narrows, plan the POD → final destination leg. Decision variables: mode (FTL truck / LTL / rail), carrier, departure timing. LTL case introduces deconsolidation warehouse as an intermediate node.

---

## Phase 2 — Component Builds

**Status:** NOT STARTED
**Gate to enter:** Phase 1 approved (all required LaTeX/design models approved).
**Gate to exit:** All components pass isolation tests.

Build sequence is fixed by dependency. No component begins until its predecessor passes isolation tests.

### Build sequence

```
2.0   Instance Generator          ← must be built first; all test fixtures depend on it
      │
2.1   Graph Generator             ← depends on: 2.0
      │
2.2   Ocean Transit Time Model    ← depends on: 2.0
2.3   Trucking Transit Time Model ← depends on: 2.0
2.4   Air Transit Time Model      ← depends on: 2.0
      │
2.5   Path-Level Transit Time Model  ← depends on: 2.2, 2.3, 2.4
      │
2.6   Ocean FCL Optimizer (MILP)  ← depends on: 2.1, 2.2
2.7   Trucking Optimizer (MILP)   ← depends on: 2.1, 2.3
2.8   Ocean LCL Optimizer (MILP)  ← depends on: 2.1, 2.2
2.9   Air Optimizer (MILP)        ← depends on: 2.1, 2.4
      │
2.10  Destination Leg Planner     ← depends on: 2.7 (trucking), rail data, 2.5 (TTM)
2.11  Rules Engine                ← depends on: 2.1 (graph to filter)
      │
2.12  AIS Tracking Adapter        ← depends on: nothing (standalone data pipeline)
2.13  Road Routing Adapter        ← depends on: nothing (standalone API client)
2.14  Air Schedule Adapter        ← depends on: nothing (OpenSky / airline data)
      │
2.15  Multimodal Stitching Layer  ← depends on: 2.6, 2.7, 2.8, 2.9, 2.10, 2.11
2.16  Rolling Horizon Controller  ← depends on: 2.15, 2.12 (AIS), 2.5 (TTM)
```

**Note on 2.8 and 2.9 (LCL and Air):** These require external data that must be sourced before implementation: NVOCC consolidation schedules + LCL rates (for 2.8), airline schedule feeds (for 2.9). Identify data sources and confirm access before the corresponding LaTeX models are written.

### Per-component definition of done

- Implementation exists at `src/components/{component}.py`
- Test file exists at `tests/components/test_{component}.py`
- All isolation tests pass (real HiGHS for MILP components; real API for adapters)
- Infeasibility test passes (structured error, not exception)
- Correctness verified on at least one manually computed instance

---

### 2.0 Instance Generator

**Why first:** Every component test needs consistent, deterministic synthetic instances matching the model parameter schemas.

**Design constraint:** **Joint session required.** Design the output schema and generation parameters together before implementation. Do not implement independently.

**Scope:** Produces structured JSON instances for all modes — ocean FCL, ocean LCL, air, trucking. Port geometry from UN/LOCODE. Sailing schedules from public data or synthetic distributions. Haversine-based transit time validation.

**Isolation tests:**
- Generates valid instance (passes schema validation) for each mode type
- Reproducible given a fixed seed
- Ocean FCL transit time estimates within ±20% of published carrier schedules on known lanes

---

### 2.1 Graph Generator

**Spec:** `model/graph_generator.tex`

**Scope:** Constructs commodity-specific subgraph G(N_k, A_k) for each shipment. Applies reachability sweep, CYC compliance filter, and decomposition into independent components. Handles all arc types: ocean FCL, ocean LCL (NVOCC sailings), air (airline legs), pre-carriage truck, drayage, inland truck, rail, transshipment, dwell. New node types added for LCL: **deconsolidation warehouses** (intermediate nodes between POD and destination for LCL shipments requiring transloading).

**Isolation tests:**
- Correct subgraph from known synthetic schedule (arcs in, arcs out)
- Deadline-infeasible commodity → empty A_k with structured report
- No dangling arcs after reachability sweep
- CYC compliance: arc excluded when τ_k(i) > CYC_ij
- Decomposition: TPEB + FEWB batch → two disconnected components
- LCL: NVOCC consolidation cutoff enforced on LCL arcs
- Air: airline flight cutoff enforced; volumetric weight computed per arc

---

### 2.2 Ocean Transit Time Model

**Spec:** `model/ocean_transit_time.tex`

**Inputs:** (POL, POD, carrier, sailing_date)
**Output:** (mean_days, sigma_days) — parametric distribution (gamma fit) over transit time

Data source: NOAA historical AIS + carrier schedule data. Phase 2: parametric model. Phase 4+: ML regression on realized shipment data.

**Isolation tests:**
- Known lane (SHA → USLAX, MSC Tiger) → mean within ±10% of published schedule
- Sigma ≈ CV × mean for configured CV
- Unknown lane → default envelope returned, not exception

---

### 2.3 Trucking Transit Time Model

**Spec:** `model/trucking_transit_time.tex`

**Inputs:** (origin_latlon, destination_latlon, truck_type)
**Output:** (mean_days, sigma_days)

Phase 2: Haversine × road factor / average truck speed. Phase 4+: Google Maps historical data.

**Isolation tests:**
- Known city pair → within ±15% of Google Maps estimate
- Reproducible given same inputs

---

### 2.4 Air Transit Time Model

**Spec:** `model/air_transit_time.tex`

**Inputs:** (origin_airport, destination_airport, carrier, flight_date)
**Output:** (mean_hours, sigma_hours) — air freight uses hours, not days

Includes: block time (gate to gate), transit time at connecting airports, customs/handling dwell at destination airport. Data source: OpenSky Network historical flight data.

**Isolation tests:**
- Known route (PVG → LAX, direct) → block time within ±10% of published flight time
- Multi-stop routing (PVG → NRT → LAX) → sum of legs + connection dwell time

---

### 2.5 Path-Level Transit Time Model

**Spec:** `model/path_transit_time.tex`

**Why this exists:** A multi-modal route is a sequence of legs with transit time distributions. The end-to-end distribution is not simply the sum of per-leg distributions — legs are correlated (ocean delay → port congestion → longer dwell) and the path may have conditional branches (if ocean is delayed past a threshold, re-route to different inland carrier).

**Approach (MVP):** Monte Carlo simulation. For each route, simulate N realizations of the leg sequence (sampling from per-leg distributions with a correlation structure), compute the end-to-end transit time distribution, and return (mean, sigma, P(≤ deadline)). N = 1,000 per route is sufficient for MVP-scale instances.

**Inputs:** Ordered leg sequence with (mean, sigma) per leg + correlation matrix between adjacent legs
**Output:** (path_mean_days, path_sigma_days, p_on_time_given_deadline)

**Isolation tests:**
- Single-leg path → output matches input distribution
- Two-leg independent path → output mean = sum of means; sigma = sqrt(sum of variances)
- Two-leg correlated path → sigma > independent case
- P(on_time) decreases monotonically as deadline tightens

---

### 2.6 Ocean FCL Optimizer

**Spec:** `model/ocean_fcl_routing.tex` (Draft v2 — pending approval)

MILP: Binary Multi-Commodity Network Flow. Joint optimization across all shipments in batch. Enforces P.2 (vessel cap), P.3 (string allocation), P.4 (budget). Returns structured solution or structured infeasibility report per commodity.

**Isolation tests:** Per CLAUDE.md — trivial feasibility, P.2 binding, P.3 binding, P.4 binding, container mix correctness, optimal value bound, decomposition independence, infeasibility propagation.

---

### 2.7 Trucking Optimizer (FTL / PTL / LTL)

**Spec:** `model/trucking_routing.tex` (Draft v1 — pending approval, 2026-05-16)

Binary Multi-Commodity Flow MILP covering three trucking service modes on the same graph: **FTL** (full truckload), **PTL** (partial truckload / Volume LTL), and **LTL** (less-than-truckload). Joint optimization across all shipments in a batch with per-mode pricing logic and hard tender-refusal constraints. Substantially more complex than ocean FCL or air models due to mode diversity and US-domestic trucking real-world rules.

**Three modes — distinct pricing regimes:**

| Mode | Range | Pricing | Network |
|---|---|---|---|
| **FTL** | ≥ 10K kg / ≥ 26 m³ | Per-mile × distance + fuel + accessorials | Direct on-demand (spot) or scheduled (contract) |
| **PTL** (Volume LTL) | 4.5–25K kg / 6–18 pallets | Per-shipment + per-mile, no terminal handling | Direct on-demand |
| **LTL** | < 4.5K kg / < 6 pallets | Density-based class × weight-break tier $/CWT | Scheduled, carrier-internal hub-and-spoke |

**Critical design correction from adversarial critique:** LTL network modeled as `(carrier, origin SC, destination SC)` tuples with carrier-published transit times — NOT as a forwarder-side hub-routing problem. As a forwarder we tender to a carrier; the carrier routes through their internal network. This is the principal departure from textbook Powell-Sheffi load planning (which is the carrier's internal problem, not the forwarder's).

**Tender acceptance probability — first-class parameter:**
- `p_{ij}^acc` = probability tender accepted at quoted rate (per carrier per lane per day)
- `ρ_{ij}^re` = re-tender premium multiplier (typ. 1.15–1.25 FTL spot, 1.05–1.10 LTL contract)
- **Expected cost: `c_exp = c_base × [1 + (1 − p_acc)(ρ_re − 1)]`**
- MVP assumption (stated explicitly): deterministic expected-cost adjustment. P1 extension: explicit stochastic tender process with re-routing via rolling horizon
- Without this adjustment, deterministic-only cost would under-quote actuals by 15–25% (per critique)

**17 constraints (full list in LaTeX):**

| # | Constraint | Notes |
|---|---|---|
| P.1 | Flow conservation | Standard |
| P.2 | FTL/PTL truck volume capacity | η = 0.85 (road fill rate, lower than ocean/air) |
| P.3 | FTL/PTL truck weight capacity | |
| P.4 | Single equipment type per slot | 53′/48′ dry van |
| P.5 | Arc-to-slot linkage | |
| P.6 | **LTL linear-foot rule (HARD REFUSAL)** | ℓ_k ≤ 12 ft for LTL; capacity load above |
| P.7 | **LTL piece dimensions & weight refusal (HARD)** | Piece >8–16 ft length; piece >2,500 lb; shipment >20–25K lb |
| P.8 | LTL pickup cutoff | |
| P.9 | Time propagation (day-bucket discretized) | 10–100× solve speedup vs. continuous |
| P.10 | **Delivery window / MABD** | [T_open, T_close] hard window; wide window if no appointment |
| P.11 | **Carrier-lane service availability** | Liftgate, residential, limited access, inside delivery as feasibility flags |
| P.12 | **Contract FTL allocation cap** | Parallels ocean string allocation |
| P.13 | Cargo type compatibility | DGR / temp / oversize |
| P.14 | **Equipment compatibility (container ↔ chassis)** | TEU→20′ chassis, FEU→40′ chassis, one container per truck |
| P.15 | **Symmetry breaking on slot index** | z_{c} ≤ z_{c−1}; 5–20× solver speedup on symmetric instances |
| P.16 | Budget cap | |
| P.17 | Domain | |

**LTL pricing — pre-computed (NMFC 2025 SDS + FAK):**
```
c_LTL_base(k, i, j) = max(AMC, min_charge,
                          min over tier b: max(w_k_lb, b) × R(class*, b) / 100)
                      × (1 + fuel_surcharge_pct)

class* = FAK class override if applicable, else NMFC SDS density-based class
```
**FAK class override:** shipper contracts collapse NMFC classes 60-200 into single class (e.g., 92.5). Most mid-market shippers buy LTL this way. Without FAK support, model mis-quotes the majority of customers.

**Symmetry breaking implementation (`§9 in LaTeX`):**
- Slot c on FTL trucks is symbolic — slot 1 vs slot 2 interchangeable
- Lex-order: `z_{c,ij,e} ≤ z_{c-1,ij,e}` forces filling lower-index slots first
- Implementation: add constraints in model-build right after z variable creation
- Profile speedup on 50-shipment representative instance

**Cost components in objective:**
- Pickup + delivery ground
- FTL truck cost (per-truck, expected — Eq. P.tender)
- PTL per-shipment cost (expected)
- LTL pre-computed cost (expected)
- Accessorials (liftgate, residential, inside, limited access, reconsignment, detention, layover, stop-off, TONU, chassis day rate, per diem, demurrage, pier pass)

**Isolation tests:**
- LTL hard refusal: linear-foot >12 ft rejected (P.6)
- LTL piece length >12 ft rejected (P.7)
- FTL/LTL boundary: cost-minimizer picks correct mode at appropriate volume
- PTL band: 10-pallet 12K-kg shipment selected for PTL not FTL or LTL
- Multi-shipment FTL consolidation: 2 shipments share one 53′ truck when total ≤ capacity
- Contract FTL allocation cap (P.12) binds
- Delivery window infeasibility (P.10): tight window → empty A_k
- FAK class override produces different LTL cost than NMFC class
- Tender acceptance: expected cost > base cost for low p_acc
- Container-chassis: TEU shipment cannot use 53′ trailer
- Symmetry breaking: solver time reduction with lex-order vs without (profiling test)

**Adversarial critique applied — 10 critical corrections** (see LaTeX §10):
1. PTL added as third mode
2. (Carrier, origin SC, destination SC) tuples replace hub routing
3. LTL hard refusal constraints (P.6, P.7)
4. Contract FTL allocation cap (P.12)
5. MABD delivery window (P.10)
6. Carrier-lane service availability matrix (P.11)
7. FAK class override
8. Tender acceptance probability (§4.8)
9. Chassis day rate, per diem, demurrage, pier pass costs
10. Time discretization to days + slot lex-ordering (P.15)

**Deferred to P1 (LaTeX §11 — 14 items):** Stochastic tender modeling, DOT Hours of Service, backhaul/round-trip optimization, driver/truck dispatch, specialty equipment (reefer, flatbed, hazmat, oversize), dedicated lanes, pool distribution, cross-border (MX/CA), stop-off/TONU/reclass variance, spot rate stochastic availability, twin-20 combo chassis, drop-and-hook, dock appointment API integration, LTL re-classification variance.

**Academic anchors:** Powell-Sheffi (1983) load planning, Powell (Optimal Dynamics, ADP for truckload), Caplice (MIT FreightLab, combinatorial procurement), Sheffi (1985) urban transportation networks, Erera et al. (2020) dynamic LTL routing.

---

### 2.8 Ocean LCL Optimizer

**Spec:** `model/ocean_lcl_routing.tex` (Draft v1 — pending approval, 2026-05-16)

Joint bin-packing × routing MILP on a 6-layer LCL graph. Simultaneously decide: (a) which LCL shipments to consolidate, (b) which container type (FEU/TEU) to use, (c) which NVOCC sailing carries each consolidated container.

**Graph structure (6 layers):**
- Origin shipper → Origin CFS → POL → POD (arrival) → POD (exit, post-customs) → Destination CFS → Consignee
- CFS nodes are explicit at both ends — LCL requires consolidation (build-up) at origin and deconsolidation (breakdown) at destination, with fixed dwell times (~1.5 days each, configurable)

**Operating model — NVOCC perspective:**
Forwarder acts as NVOCC: buys FCL containers from ocean carrier (per-container or BSA), resells space inside containers to LCL shippers at W/M rates. Optimization is on the carrier-facing side — minimize containers purchased to carry all shipments. Customer-facing pricing is out of scope.

**Container types:** FEU (76 CBM, 26,500 kg) and TEU (33 CBM, 24,000 kg). Optimizer selects mix per sailing. Fill rate cap η = 0.97.

**16 constraints (full list in LaTeX):**

| # | Constraint | Notes |
|---|---|---|
| P.1 | Flow conservation per shipment per node | Standard |
| P.2 | Each shipment in exactly one container slot | Σ y = 1 |
| P.3 | Container volume capacity | Σ v_k · y_{k,c,s} ≤ η · V_t · z_{c,s,t} |
| P.4 | Container weight capacity | Σ w_k · y_{k,c,s} ≤ W_t · z_{c,s,t} |
| P.5 | Single type per container slot | Σ_t z ≤ 1 |
| P.6 | Sailing container capacity per type | Σ_c z ≤ N_max^t(s) |
| P.7 | Arc-to-sailing linkage | x_{ij}^k = Σ_c y_{k,c,s} |
| P.8 | Hazmat pairwise co-loading exclusion | y_{k,c,s} + y_{k',c,s} ≤ 1 if incompatible |
| P.9 | CFS cutoff | t_k(POL) ≤ CFC_s |
| P.10, P.11 | Time propagation (ground, ocean) | |
| P.12 | Deadline | |
| P.13 | DGR cargo on DGR-certified sailing only | |
| P.14 | Piece dimensions fit container | Pre-processing exclusion |
| P.15, P.16 | Budget cap, domain | |

**Sequential decomposition solution strategy (per technical critique — joint MILP intractable at scale):**

LaTeX §7 documents the decomposition. Joint problem has O(|K|·|S|·|C_s|) binary variables — at production scale (|K|=50–200 shipments, |S|=10–30 sailings, |C_s|=10–30 containers): 10⁵–10⁶ binaries. Decomposition:

1. **Stage 1 — Routing without bin-packing:** Aggregate sailing capacity (CBM, kg). Shipment-to-sailing LP. Solves in seconds.
2. **Stage 2 — Bin-packing per sailing:** Variable-size bin-packing MILP per sailing (≤30 shipments per sailing). Container type + slot assignment. Solves in seconds.
3. **Coupling — Benders-style feasibility cuts:** If Stage 2 infeasible on a sailing, add cut to Stage 1, re-solve. Typical convergence: 2–5 iterations.

Strategy selection by batch size:
- **|K| ≤ 50:** solve joint MILP directly (correctness reference)
- **|K| > 50:** sequential decomposition; joint solver for validation only

Optimality gap of decomposition: empirically 2–8% on realistic LCL instances.

**Linearization and tightening (LaTeX §8):** Big-M tightening on time constraints, symmetry breaking on interchangeable container slots, valid inequality on minimum container count per sailing.

**Decision variables:** `x_{ij}^k` (arc binary), `y_{k,c,s}` (shipment-to-container binary), `z_{c,s,t}` (container slot type binary), `t_k(i)` (arrival time continuous).

**Objective:** Minimize ground costs (pickup + CFS + delivery) + container costs (Σ κ_s^t · z_{c,s,t}) + surcharges.

**Isolation tests:**
- Single LCL shipment fills partial FEU (verifies container cost charged regardless of fill)
- Two shipments consolidated into one container (bin-packing constraint binds)
- Three shipments cannot fit one FEU → optimizer picks 1 FEU + 1 TEU (mix correctness)
- Hazmat pair excluded from same container (P.8 binding)
- CFS cutoff infeasibility (cargo ready too late for sailing)
- NVOCC sailing container cap binding (P.6)
- Decomposition convergence: joint vs. sequential on same small instance — verify gap ≤ 10%
- Feasibility cut: Stage 2 infeasibility triggers correct Stage 1 cut

**Deferred to P1 (LaTeX §10):** Probabilistic transit time objective, full 3D bin-packing, multi-leg NVOCC transshipment, full IMDG hazmat segregation, CFS dock door capacity, third-party NVOCC vs. self-consolidation, joint LCL+FCL optimization, rolling-horizon consolidation, per-shipment cost linearization, equipment availability at CFS.

---

### 2.9 Air Optimizer

**Spec:** `model/air_freight_routing.tex` (Draft v1 — pending approval, 2026-05-16)

Binary Multi-Commodity Network Flow on commodity-specific subgraph G(N_k, A_k). Joint optimization across all shipments in a batch, with two air capacity layers (contracted BSA + spot rate-card) modeled simultaneously.

**Graph structure:**
- 7 node types: origin shipper → origin CFS → origin airport → hub airport(s) → destination airport → destination CFS → consignee
- 5 arc types: pickup (A_PU), CFS→airport (A_OC), air leg (A_AIR), airport→CFS (A_DC), final delivery (A_FD)
- **Each scheduled flight leg is one arc** (not one per route). Multi-stop flights (e.g., 5Y9123 TPE→ANC→JFK) decompose into multiple arcs sharing the same flight number metadata — preserves transit time visibility and per-leg ULD accounting.

**Two air capacity layers (both modeled simultaneously):**

1. **Contracted BSA (Block Space Agreement)** — per-flight per-ULD-type allocation. ULD types: LD3, LD7, PMC, AKE. Each contract `c` has: per-flight allocation `alloc(c,f,u)`, period total `alloc(c,u,t)`, pivot weight `π_c,u` (take-or-pay minimum), contract rate `r_c`, hard vs. soft flag. Aircraft physical position cap `P_{f,u}` is a separate hard limit across all contracts.

2. **Spot rate-card** — IATA weight breaks (N → +45 → +100 → +300 → +500 → +1000 kg). Per-shipment cost is pre-computed via `cost^spot(k,f) = min over tiers b: r_{f,b} · max(cw_k, b)`. No capacity constraint at planning level.

Optimizer per-shipment per-flight chooses: ULD path under contract c, or spot rate-card, or not on this flight.

**Through-ULD vs. re-ULDing at hubs:**
- For each consecutive arc pair (f, g) at hub h, pre-computed parameter `ψ_{f,g,u} ∈ {0,1}` indicates whether ULDs of type u can transit hub without breakdown
- `ψ = 1` when same flight number through-stop OR same airline with through-cargo agreement
- `ψ = 0` when interline (different carriers) — re-ULDing required, ULD broken down at hub and rebuilt for onward flight
- MCT (minimum connect time) differs: through-ULD ~1.5–4h, re-ULDing ~6–12h
- Rehandling cost `ρ^reULD ≈ $150–500/ULD` added when re-ULDing
- MVP: ψ is parameter, not decision variable. P1: promote to decision (allow forwarder-elected re-ULD for cargo re-sorting)

**19 constraints (full list in LaTeX):**

| # | Constraint | Notes |
|---|---|---|
| P.1 | Flow conservation per shipment per node | Standard |
| P.2, P.3 | ULD volume + weight capacity (per flight, ULD type, contract) | η = 0.97 fill rate cap |
| P.4 | Per-flight contract allocation cap | z ≤ alloc(c,f,u) |
| P.5 | Aircraft physical position cap | Σ_c z ≤ P_{f,u} across all contracts |
| P.6 | Period contract allocation cap | Σ_f z ≤ rem(c,u,t) |
| P.7 | Period min utilization (hard BSA take-or-pay) | M_{c,u,t} from upstream model |
| P.8, P.9 | Arc-to-ULD linkage; spot/contract exclusivity | |
| P.10 | Pivot weight linearization | Two ≥ constraints replace max() |
| P.11–P.14 | Cargo cutoff, time propagation, hub MCT | |
| P.15 | Deadline | |
| P.16, P.17 | Cargo type compatibility, ULD physical fit | DGR/PER/VAL/HEA |
| P.18, P.19 | Budget cap, domain | |

**Period commitment (P.7) is upstream:** A separate model computes `M_{c,u,t}` — how much of each hard BSA's period take-or-pay should be allocated to each routing run. The routing MILP takes M as a fixed per-run target. The MILP does NOT solve period-level commitment jointly with routing.

**Linearization details (see LaTeX §9):**
- Pivot weight max(): aux variable + two ≥ constraints
- Bilinear rehandling cost (y·y): McCormick linearization with binary aux `ζ`
- Big-M tightening: use `M = T_k^dead - t_k^rdy + ε` not loose constants
- Spot rate cost: pre-computed per (k, f) since cw_k is parameter

**Decision variables:** `x_{ij}^k` (arc binary), `y_{f,u,k}^c` (ULD assignment binary), `z_{f,u}^c` (ULD count integer), `b_{f,k}` (spot rate binary), `C_{f,u,c}` (effective chargeable weight continuous), `t_k(i)` (arrival time continuous).

**Objective:** Minimize total cost = ground costs + contracted ULD cost (with pivot floor) + spot rate cost + surcharges (FSC + SSC + THC + AMS) + rehandling cost at re-ULD hubs.

**Isolation tests:**
- Chargeable weight calculation at volumetric/actual boundary
- Spot rate at correct IATA weight break (verify min across tiers)
- Pivot weight floor binding (load less than pivot → charge at pivot)
- BSA per-flight cap binding (P.4)
- BSA period cap binding (P.6)
- Aircraft position cap binding (P.5) when sum of contracts exceeds P_{f,u}
- Through-ULD MCT vs. re-ULD MCT applied correctly at hub
- Re-ULD rehandling cost binding when interline route selected
- Direct vs. multi-hop tradeoff (cheaper multi-hop selected over expensive direct)
- Cargo type exclusion (DGR cargo never on non-DGR flight)
- Infeasibility propagation (deadline-tight shipment → empty A_k)

**Deferred to P1 (see LaTeX §11):** Period commitment joint optimization, through-ULD as decision variable, probabilistic transit time objective, full 3D bin-packing within ULD, ULD availability at CFS, CFS dock capacity, cargo type segregation within ULD, multiple ULD types per shipment, carrier alliance slot sharing, spot capacity uncertainty.

---

### 2.10 Destination Leg Planner

**Spec:** `model/destination_leg_planner.tex`

**When it runs:** Main leg (ocean/air) is `firm_planned` AND AIS/flight ETA confidence exceeds threshold (default: 72h from POD arrival). This is the late-binding precision planning step.

**Problem:** Given confirmed POD arrival (time + volume), plan the optimal POD → final destination move.

**Decision variables:** Mode selection (FTL truck / LTL truck / intermodal rail), carrier selection, departure timing from POD.

**Mode logic:**
- **FTL truck**: volume ≥ [configurable threshold, e.g., 15 CBM or fills 80% of trailer]. Direct, fastest.
- **LTL truck**: volume below FTL threshold. May route through a **deconsolidation warehouse** (new graph node type) if FCL/LCL freight must be broken down before final delivery. Rates per pallet or cwt.
- **Intermodal rail**: distance ≥ [configurable threshold, e.g., 800 km]. BNSF/UP ramp-to-ramp plus drayage at each end. Generally cheapest for long haul but adds 1–3 days transit.

**Rail nodes:** BNSF and UP intermodal ramps (from BTS FAF + public ramp location data). Arc attributes: ramp location, service days, cutoff times, ramp-to-ramp transit time by lane.

**LTL complication:** If multiple LCL shipments arrive at the same POD for the same inland destination cluster, the Destination Leg Planner should consider consolidating them into an FTL move. This requires seeing the full set of arriving LCL shipments, not optimizing one at a time.

**Isolation tests:** FTL vs. LTL mode selection at boundary volume, rail vs. truck tradeoff at boundary distance, deconsolidation warehouse routing for LTL, deadline infeasibility when no departure meets window.

---

### 2.11 Rules Engine

**Spec:** `model/rules_engine.md` (design doc)

Applies business rules at graph construction time. Filters ineligible carriers (blacklist, no contract), enforces routing guide carrier preferences, removes allocation-exhausted carriers. For LCL: checks cargo compatibility rules between co-loaded shipments.

**Isolation tests:** Blacklisted carrier never appears in any arc, preferred carrier selected when feasible, carrier at 100% utilization excluded.

---

### 2.12 AIS Tracking Adapter

**Scope:** Ingests NOAA historical AIS data (MVP), maps vessel positions to shipment milestones, produces ETA updates. Returns ETA confidence level — used by Rolling Horizon Controller to trigger destination leg planning.

**Isolation tests:** Known vessel track → correct milestone sequence; ETA confidence level increases as vessel approaches POD.

---

### 2.13 Road Routing Adapter

**Scope:** Wraps Google Maps Routes API (primary) and OSRM (free fallback). Returns road distance and transit time. Caches in Redis. Used by Trucking TTM and Destination Leg Planner.

---

### 2.14 Air Schedule Adapter

**Scope:** Ingests OpenSky Network historical flight data (MVP). Returns scheduled flights for a (origin_airport, destination_airport, date_range) query. Phase 5: live airline schedule feed.

---

### 2.15 Multimodal Stitching Layer

**Scope:** Takes mode-specific solutions (2.6, 2.7, 2.8, 2.9, 2.10) and assembles them into a coherent end-to-end route. Validates mode transitions (ocean arrival → dwell → inland departure timing). Computes full door-to-door cost, transit time distribution (calls 2.5), and P(on-time).

**Isolation tests:** Correct leg ordering, total cost, timing feasibility at each mode transition, P(on_time) from path TTM.

---

### 2.16 Rolling Horizon Controller

**Scope:** Monitors soft-planned shipments. Fires firm-up alert when CYC cutoff - firm_threshold_days is reached. Fires destination leg planning trigger when AIS ETA confidence > threshold. Coordinates with Planning Agent in Phase 4.

**Key behavioral rules:**
- Soft plan = one specific tentative sailing. The system does NOT automatically migrate a soft plan when a CYC passes. All plan changes are deliberate optimizer decisions.
- Rolling Horizon Controller monitors the CYC of the current soft-planned sailing per shipment. When within the alert threshold, it fires an alert — it does not migrate the plan.
- Active replanning (scheduled batch, accumulation trigger, or manual) may produce a new soft plan on a different sailing. This is an optimizer output, not automatic expiry handling.
- If a soft-planned sailing's CYC expires without firm-up or active replanning: infeasibility escalation. This should never occur if alerting is working — it is a failure mode, not normal operation.
- Firm-planned legs are frozen. The agent will not replan them.
- Destination leg planning trigger fires when AIS ETA confidence crosses threshold — this is the one automatic transition (in_transit → destination_planning).

**Isolation tests:** CYC alert fires at correct threshold (not before, not after), firm leg is not replanned, infeasibility escalation fires when soft-planned sailing's CYC expires without action, destination leg trigger fires at correct AIS confidence threshold.

---

## Phase 3 — MCP Server

**Status:** NOT STARTED
**Gate to enter:** All Phase 2 components pass isolation tests.
**Gate to exit:** All P0 tools pass tool-level tests.

`src/server.py` — FastMCP. One tool per logical operation. P0 tool list per `personas_and_tools.md`.

**Critical FastMCP constraint:** No `print()` on any tool call path — stdout is the JSON-RPC transport. All diagnostic output must use `logging` module or `stderr`.

### Tool-level tests (every tool)
- Schema rejection on malformed input (structured error, not exception)
- No stdout output on tool call path
- Correct output schema on happy path

---

## Phase 4 — Agent Layer

**Status:** NOT STARTED
**Gate to enter:** Phase 3 MCP server passes all tool-level tests.
**Gate to exit:** End-to-end routing run produces correct output on synthetic instances; LangSmith traces populated.

### Components

| Component | Description |
|---|---|
| Planning Agent | LangGraph node: intent classifier → capability registry → tool dispatch → structured output |
| Compliance Validator | Two deterministic LangGraph nodes: allocation re-check at commit; override feasibility check |
| Execution Monitor | Polls shipment states; fires re-plan on disruption; triggers destination leg planning on AIS signal |
| Market Intelligence | Refreshes spot rates, congestion signals; flags policy review triggers |
| Agent Interaction Logger | Logs every query + response to `logs/agent_interactions.jsonl` |

### Soft plan / firm plan in agent behavior

- Planning Agent produces `soft_planned` routes. These can be replanned.
- Compliance Validator is invoked when soft → firm transition is triggered (by Rolling Horizon Controller's CYC proximity alert). It checks allocation state one final time before firming.
- Once a route is `firm_planned`, Planning Agent will not replan that leg unless the operator explicitly overrides or a hard disruption event fires.
- Execution Monitor is the component that watches for the destination leg planning trigger (AIS ETA confidence).

---

## Phase 5 — Product Layer

**Status:** NOT STARTED
**Gate to enter:** Phase 4 agent layer passes end-to-end tests.
**Gate to exit:** Full stack running locally with synthetic data; operator completes core daily workflow end to end.

### Shipment Ingestion Layer (new — Phase 5.1–5.2)

Two production ingestion modes:

**Mode 1 — Push API (webhook / REST)**
External shipper TMS, ERP, or WMS systems post shipments to the platform. The API validates the shipment schema, deduplicates against `external_id`, assigns `tenant_id` from API key, and inserts with `status = unrouted` and `ingestion_source = push_api`. Idempotent — repeated pushes of the same external_id are safe.

Key design questions:
- Schema: accept our canonical schema, or translate from common external schemas (EDI 204, FMS, shipper-specific JSON)?
- Auth: API key (per-tenant) for external system → shipment ingestion endpoint
- Error handling: validation failures return structured errors; the external system must handle and retry

**Mode 2 — Manual UI entry**
Forwarder ops manually enters a shipment via a form in the frontend. Fields match the canonical shipment schema. For high-volume forwarders, also support CSV bulk upload (multiple shipments in one file).

**Demand Generator** — dev/testing only; not exposed in production UI.

### Batch Planning and Subset Selection (new — Phase 5.3)

When a forwarder ops planner wants to trigger a routing run manually, they select a subset of unrouted/soft-planned shipments. Subset dimensions:

| Filter | Use case |
|---|---|
| All unrouted | Run everything — the default scheduled batch |
| By shipper/client | "Plan all Acme Corp shipments now" |
| By trade lane | "Plan all TPEB shipments" (ship together — better consolidation) |
| By destination port | "Plan everything going to USLAX this week" |
| By cargo-ready date window | "Plan everything ready in the next 7 days" |
| By urgency (CYC proximity) | "Plan anything with CYC cutoff within 5 days" |
| Custom selection | Checkbox individual shipments from the shipment list |

**UI:** A "Plan Shipments" screen (or modal from the Operations Dashboard). Filters narrow the visible shipment list; operator confirms selection; system shows estimated batch size and expected joint optimization benefit (e.g., "14 shipments on TPEB — joint optimization may improve carrier utilization by up to X%"). Triggers a Celery routing job.

**Why batch > individual:** The MILP jointly optimizes all selected shipments — respecting shared allocation caps, consolidating LCL freight into containers, filling vessels more efficiently. Sequential individual routing is myopic and greedy; it can exhaust a preferred sailing for one shipment that would have been better shared across five.

### Soft Plan / Firm Plan in the UI (Phase 5.4)

Shipment status column shows plan state visually:
- `unrouted` — gray, no plan
- `soft_planned` — blue, tentative plan assigned; shows carrier + ETD + "soft"
- `firm_deadline` — amber, CYC cutoff within [threshold]; operator alerted; "Firm required in Xd"
- `firm_planned` — green, booking committed to carrier
- `in_transit` — green, physically moving
- `destination_planning` — purple, destination leg being optimized
- `delivered` — gray, complete

**Firm-up workflow:**
- Rolling Horizon Controller flags shipments approaching CYC threshold → status = `firm_deadline`
- Operator sees amber alert on Operations Dashboard and Exception Queue
- In Co-pilot/Supervised mode: operator confirms firming (or adjusts carrier) → status = `firm_planned`
- In Autonomous mode: auto-firms after a short confirmation window (configurable, default 4 hours)
- Once firm_planned: route is locked; replanning is blocked for that leg

### Full Phase 5 Build Order

```
5.1   Shipment Ingestion — Push API endpoint + auth (API key)
5.2   Shipment Ingestion — Manual UI entry form + CSV bulk upload
5.3   Database schema + RLS setup (includes updated lifecycle states)
5.4   FastAPI skeleton + Clerk middleware + tenant session variable
5.5   Core shipment/route/booking APIs (CRUD + state machine)
5.6   Celery + Redis setup + routing job enqueue/dequeue
5.7   Next.js skeleton + Clerk auth flow + sidebar nav
5.8   Operations Dashboard screen (includes firm_deadline alerts)
5.9   Exception Queue screen
5.10  Batch Planning / Subset Selection screen
5.11  Routing Activity screen
5.12  Shipment Detail screen (with LangSmith trace retrieval)
5.13  Policy & Guardrails editor (includes firm-up threshold config)
5.14  Allocation Monitor screen
5.15  Demand Generator (joint design session first — dev/test only)
5.16  Onboarding wizard
5.17  Stripe billing integration
5.18  Notifications (SSE → email → Slack)
5.19  Audit log + override log wiring
```

---

## Phase 6 — Integration and End-to-End Testing

**Status:** NOT STARTED
**Gate to enter:** Phase 5 complete; full stack running locally.

**Scope:**
- End-to-end: push API ingestion → scheduled batch → soft_planned → firm_deadline alert → firm_planned → in_transit → destination leg planning → delivered
- Tenant isolation: two synthetic tenants cannot see each other's data
- Billing event fires on firm_planned commit, not on soft_planned
- LangSmith trace retrievable from Shipment Detail for every committed decision
- Kill switch stops all agent activity within one routing cycle
- Firm leg is never replanned by agent
- Destination leg planning triggered at correct AIS ETA confidence threshold

---

## Phase 7 — Extensions

Ordered by likely priority:

1. **Autonomous booking execution** — carrier API integrations (MSC, CMA CGM as first ocean targets; trucking TMS APIs). Requires legal/liability framework. Booking is currently human-initiated.
2. **Time-phased capacity release** — rolling demand forecast driving allocation in tranches. See PRD §6 Open Question 9.
3. **Learned constraint inference** — systematic analysis of override log; automatic constraint suggestion.
4. **Counterfactual / regret analysis** — post-hoc optimal route computation after delivery.
5. **Pricing engine** — markup layer above routing cost; prerequisite for portfolio/margin optimization.
6. **Mobile app** — React Native + Expo; push notifications + quick actions only.
7. **Rail mode (standalone)** — full intermodal rail optimization, currently handled as an arc type within Destination Leg Planner only.

---

## Open Decisions Blocking Phase 1

These must be resolved before the corresponding LaTeX model can be written:

| Decision | Options | Current status |
|---|---|---|
| Rules Engine spec format | LaTeX vs. structured design doc | Design doc — rules are procedural, not optimization |
| Instance Generator output schema | Needs joint design session | Joint session required |
| Container type scope for MVP | FEU only, or include TEU? | FEU only (MVP) |
| Per-leg transit time distribution family | Gamma vs. lognormal vs. empirical KDE | Parametric (gamma) — requires less data, interpretable |
| Firm-up trigger design | Feasible set threshold N (alert when N sailings remain) + hours-before-current-soft-plan-CYC; both per-tenant configurable. Service level does not govern this. | Resolved — see data_model.md §3.5 |
| LCL NVOCC rate data | Synthetic rates for Phase 2 (TPEB LCL ≈ $60–120/CBM); CSV upload from forwarder in onboarding for Phase 5. No licensed feed needed for MVP. | Resolved |
| Air schedule data source | OpenSky (free, historical) for Phase 2; commercial feed (IATA/Cirium/OAG) for Phase 5 | Resolved |
| Air rate structure | Rate card per (lane, airline) at IATA weight breaks. Chargeable weight = max(actual_kg, vol_m³ × 167). Surcharges: FSC + SSC + THC + AMS. No ULD buying for MVP. Capacity: unconstrained proxy. | Resolved |
| Destination Leg Planner scope | FTL (flat rate per lane zone) + LTL (3-break rate card, no NMFC class for MVP) + rail (arc type). All three in Phase 2. | Resolved |
| LTL deconsolidation warehouse data | Which CFS warehouses? What is the data source for locations, dwell times? | Open |

---

## Current Status Summary

| Phase | Status | Blocker |
|---|---|---|
| 0 — PRD | In review | Resume review from agent_design.md §1.3 |
| 1 — LaTeX models | Not started | PRD approval |
| 2 — Components (×16) | Not started | LaTeX approvals |
| 3 — MCP server | Not started | All Phase 2 isolation tests |
| 4 — Agent layer | Not started | Phase 3 tool-level tests |
| 5 — Product layer | Not started | Phase 4 agent tests |
| 6 — Integration | Not started | Phase 5 complete |
| 7 — Extensions | Deferred | Design partner feedback |
