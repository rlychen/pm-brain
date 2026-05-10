---
title: Ocean FCL Routing Model (LaTeX Source)
type: model
status: draft-v2
date: 2026-05-10
source_file: model/ocean_fcl_routing.tex
---

# Ocean FCL Multi-Commodity Routing — MVP Formal Model v2

> Full LaTeX source lives at `model/ocean_fcl_routing.tex` in the repo. This note is a vault mirror for reference.

## Model Summary

**Problem type:** Binary Multi-Commodity Network Flow (BMCNF)

**Scope:** FCL only, FEU+TEU containers, string-based carrier allocation, deterministic time windows. Probabilistic objectives and time-phased capacity deferred to P1.

## Graph Structure

- **N_O** — Origin doors (commodity sources)
- **N_POL** — Ports of Loading (SHA, NGB, SZX)
- **N_POD,A** — POD Arrival nodes (vessel arrives)
- **N_POD,E** — POD Exit nodes (customs cleared)
- **N_D** — Destinations (commodity sinks)

Arc types: Pre-carriage (A_PC) | Ocean sailings (A_OC) | Dwell (A_DW) | Inland (A_IL)

Each physical POD split into arrival+exit nodes connected by dwell arc — makes port processing time explicit.

## Key Equations

**Container count:**
```
n_k = max(ceil(v_k/67), ceil(w_k/26000))
```

**Subgraph pre-filter (6-step algorithm):**
1. Pre-carriage pass: τ_k(j) = t_k^rdy + μ_{o(k),j}^PC
2. Ocean pass: include (j,p) if τ_k(j) ≤ CYC_jp AND end-to-end deadline feasible
3. Dwell pass: include if ∃ feasible sailing to POD
4. Inland pass: include if h = d(k) AND deadline feasible
5. Trim pre-carriage: remove if no adjacent feasible ocean arc
6. BFS reachability from both ends; retain arcs on complete paths

**Infeasible commodities:** removed from K before MILP; structured JSON report returned.

**Container mix pre-computation (per commodity k, sailing jp):**
```
cost(k,jp) = f* × c_FEU + t* × c_TEU
slots(k,jp) = 2f* + t*   (TEU slot consumption)
```
Solved analytically as 2-variable integer program.

## Objective

```
min Z = sum_k [ sum_{PC arcs} n_k × c^PC × x + sum_{OC arcs} cost(k,jp) × x + sum_{IL arcs} n_k × c^IL × x ]
```

## Constraints

- **(P.1)** Flow conservation: outflow − inflow = b_k(n); b_k(source)=+1, sink=−1, else 0
- **(P.2)** Vessel capacity: sum_k slots(k,jp) × x_jp^k ≤ cap_jp^TEU, ∀ sailing (j,p)
- **(P.3)** String allocation: sum over all sailings of string s in period t of [sum_k slots(k,jp) × x_jp^k] ≤ rem(s,t)
- **(P.4)** Budget cap (per commodity, optional)
- **(P.5)** Binary domain

## String-Based Carrier Allocation

Carriers sell capacity on service strings (fixed port-call loops). Forwarder's BSA covers ALL (POL,POD) pairs on a string in a monthly period. Two constraints bind simultaneously:
- P.2: vessel-level cap (per sailing)
- P.3: commercial/contractual cap (across all sailings of string in month)

Parameters: alloc(s,t) = contracted block; util(s,t) = already booked (external state); rem(s,t) = alloc − util

## Graph Decomposition

Commodity-supply graph H: edge (k1,k2) if they share a feasible sailing OR same allocation pool.
- Find connected components → solve each independently
- TPEB + FEWB always decompose (different strings, different ports, no shared supply)

## Instance Generator (Demand-First)

1. Generate commodities (origin, dest, volume, weight, cargo-ready date)
2. Compute min_feasible_time per commodity
3. Set deadlines = min_time + Uniform(slack_min, slack_max)
4. Generate sailings calibrated to target load_factor
5. Set rem(s,t) from initial_utilization_fraction
6. Pre-compute container mix
7. Run subgraph construction
8. Run decomposition
9. Serialize to JSON

## P1 Deferred

- Commodity-specific customs inspection model (dwell = f(HS code, C-TPAT, origin, consignee))
- Time-phased capacity release (tranches driven by demand forecast)
- Probabilistic delivery constraint: P(delivery ≤ deadline) ≥ α_k
- Multi-objective Pareto frontier; transshipment arcs

## Open Items

1. String allocation granularity: monthly vs. 13 four-week periods
2. Multi-POL strings: SHA→NGB→SZX sailings treated as independent (may overstate capacity)
3. CYC compliance risk: flag when pre-carriage margin < 0.5 days
4. Infeasibility report schema: finalize before coding pre-filter
5. Generator sailing schedule realism: use public schedule data, not fabricated ETDs
