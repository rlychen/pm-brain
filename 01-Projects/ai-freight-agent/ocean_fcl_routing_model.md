# Ocean FCL Routing Model — LaTeX Mirror

**Source:** `model/ocean_fcl_routing.tex` in repo  
**Status:** Draft v2 — 2026-05-10  
**Last synced:** 2026-05-10

> Vault copy for reference. The authoritative file is the `.tex` in the repo. Do not edit here — edit the `.tex` and re-sync.

---

## Changes in this version (adversarial review round, 2026-05-10)

- **C1** — Unit convention remark added to Section 4.5; rem(s,t) example updated from "200 FEU/month" to "400 TEU slots/month"
- **C2** — Mix algorithm replaced: greedy one-at-a-time substitution → explicit enumeration over all f from f_min to 0; guarantees optimality
- **C3** — New remark: n_k approximation for physical container count; deviation bound |f*+t* − n_k| ≤ 1
- **C4** — Decomposition edge criterion 2: "use" replaced with explicit pre-solve definition (A^k_OC membership)
- **C5** — P.2 and P.3 inner sums: k∈K → k∈K:(j,p)∈A^k_OC; eliminates undefined x^k_jp terms
- **G2** — Ocean sailing pass step 2: min_{h=d(k)} μ^IL → μ_{pE,d(k)}^IL (vacuous min removed)
- **G3** — CYC sync note added: 4-day buffer in sailing generation must match step 2b of min-feasible-time computation
- **G4** — Allocation calibration guidance added to generator step 5
- **G6** — N_k added to Sets; P.1 reindexed from n∈N to n∈N_k in both detailed and complete formulation
- **Container specs** — 40'HC FEU updated 67→76 CBM, 26,000→26,500 kg throughout; container specs table added; n_k formula updated
- **Open Question 6** — BSA contract unit convention: confirm FEU vs TEU quoting with design partner

---

```latex
%% ocean_fcl_routing.tex
%% Ocean FCL Multi-Commodity Routing — MVP Formal Model
%% Phase 1 — must be approved before any code starts
%% Status: Draft v2 — 2026-05-10

\documentclass[11pt,a4paper]{article}

\usepackage{amsmath,amssymb,amsthm}
\usepackage{booktabs}
\usepackage{geometry}
\usepackage{hyperref}
\usepackage{array}
\usepackage{xcolor}
\usepackage{enumitem}
\usepackage{longtable}

\geometry{margin=2.5cm}
\setlength{\parskip}{0.4em}
\hypersetup{colorlinks=true, linkcolor=blue}

\newcommand{\ceil}[1]{\left\lceil #1 \right\rceil}
\newcommand{\floor}[1]{\left\lfloor #1 \right\rfloor}
\newcommand{\defeq}{\triangleq}

\theoremstyle{definition}
\newtheorem{defn}{Definition}
\newtheorem{remark}{Remark}

\title{\textbf{Ocean FCL Multi-Commodity Routing}\\[0.4em]
\large MVP Formal Model --- Phase 1, v2}
\author{AI Freight Agent}
\date{Draft --- 2026-05-10}

\begin{document}
\maketitle

\begin{abstract}
Formal specification of the optimization model for the ocean FCL multi-commodity routing
problem. The model is a Binary Multi-Commodity Flow problem on a commodity-specific
subgraph derived from the physical freight network.
Scope: FCL only, FEU+TEU containers, string-based carrier allocation capacity, deterministic
time windows. Probabilistic objectives, hazmat/OOG cargo differentiation, and time-phased
capacity management are explicitly deferred to P1.
\end{abstract}

%%--------------------------------------------------------------------
\section{Problem Statement}
%%--------------------------------------------------------------------

Given a set of shipment requests (commodities), each defined by an origin location,
destination location, cargo volume, cargo weight, and time constraints, find a
minimum-cost assignment of commodities to paths through the multimodal freight network.

\paragraph{MVP scope.}
- FCL only; LCL deferred
- Container types: FEU (40'HC, ~76 CBM, ~26,500 kg) and TEU (20', ~33 CBM, ~24,000 kg). Optimizer selects mix.
- Port dwell: fixed port-specific mean
- Carrier allocation: string-level monthly contracted block
- Objective: minimize total freight cost (deterministic time windows)
- No transshipment in MVP

%%--------------------------------------------------------------------
\section{Sets and Indices}
%%--------------------------------------------------------------------

| Symbol | Description |
|--------|-------------|
| K | Commodities |
| N, A | All nodes and arcs |
| A_PC, A_OC, A_DW, A_IL | Arc subsets by type |
| A^k | Feasible arc set for commodity k |
| A^k_S = A^k ∩ A_S | Feasible arcs of type S for commodity k |
| S | Carrier service strings |
| T | Allocation time periods (monthly) |
| P(s) | (POL, POD) pairs covered by string s |
| N_k | Nodes incident to A^k: N_k = {n : ∃(n,j)∈A^k or ∃(i,n)∈A^k}. Used to eliminate vacuous flow constraints in P.1. |

%%--------------------------------------------------------------------
\section{Parameters}
%%--------------------------------------------------------------------

### 4.1 Commodity Parameters

n_k = max(ceil(v_k/76), ceil(w_k/26500))   [eq:nk]

76 CBM per 40'HC FEU; 26,500 kg payload limit.

**Container specifications:**

| Container type | Code | TEU slots | Usable volume | Payload |
|---|---|---|---|---|
| 20' standard | TEU | 1 | 33 CBM | 24,000 kg |
| 40' standard | FEU (std) | 2 | 67 CBM | 26,500 kg |
| 40' High Cube | FEU (HC) | 2 | 76 CBM | 26,500 kg |

MVP uses 40'HC as FEU type. 40' standard deferred to P1.

**Remark [n_k and physical container count]:** n_k is the min number of FEU-sized moves for land
transport costs. On ocean arcs, mix pre-computation gives physical count f*+t* which may differ
from n_k when a FEU is substituted by a TEU. In practice (ρ∈[0.63,0.83]), f*+t* = n_k in
virtually all instances. Deviation bound: |f*+t* − n_k| ≤ 1.

### 4.5 Carrier Allocation Parameters

alloc(s,t), util(s,t), rem(s,t) all in **TEU slots**.
BSA contracts may be quoted in FEU — convert at input: alloc_FEU × 2 = alloc in TEU slots.
(Open Question 6: confirm with design partner.)

### 4.6 Pre-Computed Container Mix

For each (commodity k, sailing (j,p)):

  min  f·c_FEU + t·c_TEU
  s.t. 76f + 33t ≥ v_k       (volume)
       26500f + 24000t ≥ w_k  (weight)
       f, t ∈ Z≥0

**Enumeration algorithm (replaces greedy):**
1. f_v = ceil(v_k/76), f_w = ceil(w_k/26500). f_min = max(f_v, f_w).
2. For f in {f_min, f_min-1, ..., 0}:
   a. t_v = max(0, ceil((v_k - 76f)/33))
   b. t_w = max(0, ceil((w_k - 26500f)/24000))
   c. t = max(t_v, t_w)
   d. C(f,t) = f·c_FEU + t·c_TEU
3. Return (f*, t*) = argmin C(f,t)

Runs over ≤ f_min+1 candidates. FEU always cheaper per CBM (ρ∈[0.63,0.83] → 1/ρ∈[1.20,1.59]
TEU-equivalents per FEU-volume). Enumeration guarantees optimality.

Derived: cost(k,jp) = f*·c_FEU + t*·c_TEU; slots(k,jp) = 2f* + t*

%%--------------------------------------------------------------------
\section{Subgraph Construction}
%%--------------------------------------------------------------------

Algorithm builds G(N_k, A_k) for each commodity k:
1. Pre-carriage pass — τ_k(j) = t_rdy + μ_PC; include if can make ≥1 sailing
2. Ocean sailing pass — include (j,p) iff τ_k(j) ≤ CYC_jp AND
   ETD_jp + μ_OC_jp + μ_DW_p + μ_IL_{pE,d(k)} ≤ T_dead_k
   (Note: min_{h=d(k)} replaced with μ_{pE,d(k)} — there is only one destination)
3. Dwell pass — include dwell arc iff some sailing arrives at p
4. Inland pass — include (pE,h) iff h=d(k) and deadline feasible
5. Trim pre-carriage — remove POLs with no feasible sailings
6. Reachability sweep — BFS forward from o(k) and backward from d(k); retain intersection

%%--------------------------------------------------------------------
\section{Formulation (P)}
%%--------------------------------------------------------------------

min Z = Σ_k [ Σ_{(i,j)∈A^k_PC} n_k·c_PC·x + Σ_{(j,p)∈A^k_OC} cost(k,jp)·x + Σ_{(pE,h)∈A^k_IL} n_k·c_IL·x ]

P.1: Σ_{(n,j)∈A^k} x_nj - Σ_{(i,n)∈A^k} x_in = b_k(n)   ∀ k∈K, n∈N_k
     (indexed over N_k not N — eliminates vacuous 0=0 constraints)

P.2: Σ_{k∈K:(j,p)∈A^k_OC} slots(k,jp)·x_jp ≤ cap_jp_TEU   ∀ (j,p)∈A_OC
     (inner sum restricted to k where (j,p) is feasible — x^k_jp undefined otherwise)

P.3: Σ_{(j,p): s_jp=s, t_jp=t} Σ_{k:(j,p)∈A^k_OC} slots(k,jp)·x_jp ≤ rem(s,t)   ∀ s∈S, t∈T
     (same restriction on inner k sum)

P.4: Budget constraints for commodities with finite B_k

P.5: x_ij^k ∈ {0,1}   ∀ k∈K, (i,j)∈A^k

%%--------------------------------------------------------------------
\section{Graph Decomposition}
%%--------------------------------------------------------------------

Commodity-supply graph H = (K, E_H). Edge (k1,k2) iff:
1. ∃(j,p) ∈ A^k1_OC ∩ A^k2_OC (shared feasible sailing), OR
2. ∃s∈S, t∈T: ∃(j,p)∈A^k1_OC with s_jp=s,t_jp=t AND ∃(j',p')∈A^k2_OC with s_j'p'=s,t_j'p'=t
   (shared allocation pool — "use" means "has feasible sailing in A^k_OC", determined pre-solve)

Connected components of H are solved independently.

%%--------------------------------------------------------------------
\section{Transit Time}
%%--------------------------------------------------------------------

Trucking: road_dist = haversine × 1.25 (China) or 1.20 (US)
          transit = road_dist / 600 (China) or 800 (US) km/day

Ocean: sail_dist = haversine × 1.15 (TPEB lane factor)
       transit = sail_dist / 600 km/day (~14 knots)
       Validation: SHA→USLAX ~9,200 km geodesic → ~10,580 km sail → ~17.6 days ✓

%%--------------------------------------------------------------------
\section{Instance Generator}
%%--------------------------------------------------------------------

Demand-first:
1. Generate commodities:
   - Sample v_k, w_k; compute n_k = max(ceil(v_k/76), ceil(w_k/26500))
   - Sample cargo-ready date
2. Compute min feasible time per commodity (pre-carriage + 4-day CYC buffer + ocean + dwell + inland)
3. Set deadlines (slack above min feasible time)
4. Generate sailings:
   - Weekly per string per POL/POD pair
   - Set CYC_jp = ETD_jp - 4 days
     [SYNC NOTE: this buffer must match step 2 CYC buffer — update both together if changed]
   - Calibrate cap_jp_TEU to hit load_factor
5. Set allocation state:
   rem(s,t) = alloc_per_month × (1 - util_fraction)
   [CALIBRATION: set alloc_per_month ≈ Σ cap_jp_TEU × target_alloc_util_fraction
    so that P.3 binds at intended load level]
6. Pre-compute container mix (enumeration algorithm above)
7. Run subgraph construction
8. Run decomposition — verify TPEB/FEWB decompose independently
9. Serialize to JSON

%%--------------------------------------------------------------------
\section{Open Items}
%%--------------------------------------------------------------------

1. String allocation granularity: monthly vs 13 four-week periods — confirm with carrier
2. Multi-POL strings: SHA→NGB→SZX→USLAX treated as independent sailings; may overstate capacity
3. CYC compliance risk: flag when margin < 0.5 days
4. Infeasibility report schema: define JSON before coding pre-filter
5. Generator sailing schedule realism: use public schedule data, not fabricated
6. BSA contract unit convention: confirm with design partner whether TPEB contracts are quoted
   in FEU or TEU; model uses TEU slots internally with ×2 conversion for FEU-quoted contracts

**P1 additions (Other P1 Items):**
- Multi-HS-code commodity schema: real FCL bookings can consolidate multiple products with
  different HS codes. MVP uses single hs_k field (harmless while customs model deferred).
  P1: replace hs_k with list; inspection risk = max tier across codes.
```
