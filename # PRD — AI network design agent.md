# PRD — AI network design agent

*Status: draft*

---

## Problem statement

Network designers need a way to quickly assess whether the current network of fulfillment centers (FCs), transfer fulfillment centers/vendor flex (TXFCs), and replenishment centers (RCs) can store the inventory implied by current buying patterns, given:

- Fixed storage capacities across six storage types: Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST
- Node-level storage restrictions enforced by DMS layer rules (network specialization constraining what capacity types, velocity FAST/SLOW, and specialized SKUs can go into each node)
- SKU specializations that restrict which nodes can receive which SKUs. Specialized SKUs can only go to nodes certified for that specialization.

Today this assessment is manual, slow, and ad-hoc. There is no analytic model to answer these questions consistently.

---

## Goal

Enable a user to describe a network storage capacity assessment problem in natural language, have the agent build and solve a MILP, and return a clear answer about whether inventory fits and where the squeeze is.

---

## Background and context

The network has three node types: FCs, TXFCs, and RCs. Storage capacity is dimensioned across six capacity types — Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST. The "Days of Cover" (DOC) framing relates inventory volume to forecasted demand. Buying patterns drive what inventory needs to be stored; that inventory must fit within storage capacity.

The MVP focuses on a single assessment question: given current network capacity and a specified buying pattern, does the inventory fit? Inbound processing capacity is explicitly out of scope for MVP.

---

## Capacity type and velocity definitions

### Capacity type (SKU attribute)

Capacity type is a physical classification of a SKU based on its dimensions and weight. It is hand-assigned per SKU in SKU_MASTER and optionally validated at load time against the thresholds in `config.yaml`.

| Type | Max weight (kg) | Max CBM | Notes |
|---|---|---|---|
| totable | 3.0 | 0.03 | Small items stored in bins on shelves |
| grande | 15.0 | 0.12 | Larger than totable; differentiated from totable primarily by weight |
| non-totable | > 15.0 or > 0.12 | — | Large or heavy items stored in large bins or on pallets directly |

These thresholds are configurable in `config.yaml`. The thresholds are used for validation (flag mismatches as warnings, not hard failures) — `capacity_type` in SKU_MASTER is the authoritative classification.

### Velocity (PO line attribute, computed at runtime)

Velocity — FAST or SLOW — is not stored in DEMAND. It is computed at runtime per PO line from quantity and SKU CBM:

```
po_line_volume_cbm = quantity × skuid.cbm
velocity = FAST if po_line_volume_cbm >= FAST_THRESHOLD_CBM else SLOW
storage_type = capacity_type + "_" + velocity   # e.g., grande_fast
```

`FAST_THRESHOLD_CBM` defaults to 0.8 CBM and is configurable in `config.yaml`. The same SKU can be FAST in one PO line and SLOW in another depending on the quantity ordered. Velocity is computed in the preprocessing step before the MILP is built.

---

## Configuration

All tunable thresholds live in `config.yaml` at the repo root. This file is version controlled. No thresholds are hardcoded in the solver or agent.

```yaml
# config.yaml

# Velocity classification
fast_threshold_cbm: 0.8        # PO line volume >= this threshold is FAST
pallet_cbm: 1.4                # Reference pallet size in CBM (informational)

# Capacity type thresholds
# capacity_type is hand-assigned in SKU_MASTER; these are used for validation only
totable:
  max_weight_kg: 3.0
  max_cbm: 0.03

grande:
  max_weight_kg: 15.0
  max_cbm: 0.12

non_totable:                   # Anything exceeding grande thresholds
  min_weight_kg: 15.0
  min_cbm: 0.12

# Solver settings
solver: CBC
max_solve_time_seconds: 30

# Soft constraint penalty weights
capacity_penalty_weight: 1000
specialization_penalty_weight: 1000
```

---

## Data model

Five files define the inputs to the system. Three are static baseline files maintained in the repo. Two are runtime inputs loaded per session.

### Static baseline files (repo-resident, version controlled)

**NODES** — physical node definitions and storage capacity limits. One row per node.

| Column | Type | Description |
|---|---|---|
| nodeid | string | Unique node identifier |
| nodename | string | Human-readable node name |
| nodetype | string | FC, TXFC, or RC |
| latitude | float | Node latitude (for distance calculations and visualization) |
| longitude | float | Node longitude |
| max_totable_fast | float | Max Totable FAST capacity (CBM) |
| max_totable_slow | float | Max Totable SLOW capacity (CBM) |
| max_grande_fast | float | Max Grander FAST capacity (CBM) |
| max_grande_slow | float | Max Grander SLOW capacity (CBM) |
| max_nt_fast | float | Max Non-totable FAST capacity (CBM) |
| max_nt_slow | float | Max Non-totable SLOW capacity (CBM) |

Owned by IPC and network design teams. Changes when physical infrastructure changes. Git-tracked with versioning strategy TBD (dated filenames vs. `/versions` folder).

---

**NODE_REST** — node-level storage restrictions. One row per node. Binary flags indicate whether that node can store each type.

| Column | Type | Description |
|---|---|---|
| nodeid | string | Foreign key to NODES |
| totable_fast | bool | Can store Totable FAST |
| totable_slow | bool | Can store Totable SLOW |
| grande_fast | bool | Can store Grander FAST |
| grande_slow | bool | Can store Grander SLOW |
| nt_fast | bool | Can store Non-totable FAST |
| nt_slow | bool | Can store Non-totable SLOW |
| dg | bool | Certified for Dangerous Goods |
| temp | bool | Certified for Temperature Controlled |
| hnb | bool | Certified for Heavy and Bulky |
| hvhr | bool | Certified for High-Value High-Risk |
| sioc | bool | Certified for Ship-in-Own-Container |
| pets | bool | Certified for Pets |

Owned by IPC team that manages DMS layer rules and case mappings. Changes when certifications or business rules change. Git-tracked alongside NODES. Maintained separately from NODES because it has a different owner and change cadence.

---

**SKU_MASTER** — SKU attributes and specialization flags. One row per SKU.

| Column | Type | Description |
|---|---|---|
| skuid | string | Unique SKU identifier |
| skuname | string | Human-readable SKU name |
| weight | float | Weight (kg) |
| length | float | Length (cm) |
| width | float | Width (cm) |
| height | float | Height (cm) |
| cbm | float | Volume (cubic meters) |
| capacity_type | string | totable, grande, or non-totable (hand-assigned) |
| dg | bool | Dangerous Goods |
| temp | bool | Temperature Controlled |
| hnb | bool | Heavy and Bulky |
| hvhr | bool | High-Value High-Risk |
| sioc | bool | Ship-in-Own-Container |
| pets | bool | Pets |

Owned by supply chain management tech team. Git-tracked. Each SKU in DEMAND must have a corresponding row in SKU_MASTER. At load time, `capacity_type` is optionally validated against weight and CBM thresholds in `config.yaml`; mismatches surface as warnings.

---

### Runtime input files (loaded per session, not repo-resident)

**DEMAND** — PO-level buying data representing the inventory that needs to be stored. One row per PO line.

| Column | Type | Description |
|---|---|---|
| poid | string | Purchase order ID |
| skuid | string | Foreign key to SKU_MASTER |
| quantity | int | Units ordered |
| order_date | date | Date order was placed |
| estimated_delivery_date | date | Expected arrival date |

User provides per session as CSV. Can be historical actuals or a forecast scenario. Velocity (FAST/SLOW) is not stored here — it is computed at runtime per PO line from `quantity × cbm`. The agent can operate on any contiguous time window within the provided data.

---

**SOH** — current inventory state at each node before new demand lands. One row per inventory parcel. If starting greenfield, SOH is empty.

| Column | Type | Description |
|---|---|---|
| nodeid | string | Foreign key to NODES |
| totable_fast_cbm | float | Totable FAST inventory (CBM); null if none |
| totable_slow_cbm | float | Totable SLOW inventory (CBM); null if none |
| grande_fast_cbm | float | Grander FAST inventory (CBM); null if none |
| grande_slow_cbm | float | Grander SLOW inventory (CBM); null if none |
| nt_fast_cbm | float | Non-totable FAST inventory (CBM); null if none |
| nt_slow_cbm | float | Non-totable SLOW inventory (CBM); null if none |
| dg | bool | Contains Dangerous Goods |
| temp | bool | Contains Temperature Controlled |
| hnb | bool | Contains Heavy and Bulky |
| hvhr | bool | Contains High-Value High-Risk |
| sioc | bool | Contains Ship-in-Own-Container |
| pets | bool | Contains Pets |

Each row represents one inventory parcel. Exactly one CBM column is populated per row; the rest are null. Specialization flags indicate what restrictions apply to that parcel. The solver aggregates rows by nodeid to compute current utilization before adding new demand.

---

## Users and use cases

**Use case 1 — Network planner: capacity simulation**
Planner wants to test whether the current network can hold 42 DOC based on historical buying patterns. Takes last 6 weeks of buying data (6 weeks × 7 days = 42 days). Passes DEMAND as CSV, specifies the 6-week window. Agent builds and solves the MILP, returns whether inventory fits, identifies binding capacity constraints, and reports utilization by node and storage type.

**Use case 2 — Supply chain analyst: audit across a planning horizon**
Analyst asks "given last quarter's actual buying, where am I tightest across the full quarter?" Passes 13 weeks of DEMAND data. Specifies a rolling 6-week planning window. Agent solves for each consecutive window and returns both period-by-period results and summary statistics across all windows (see Output and analysis section). The data duration (e.g., 13 weeks) and planning window (e.g., 6 weeks) are configurable.

**Use case 3 — Demand planner: stress testing**
Planner asks "what happens to my storage utilization if peak season demand is 20% higher than forecast?" Agent perturbs the input demand, re-solves, and returns a side-by-side comparison of utilization and breaches against the baseline solve.

---

## MVP scope — what's in

- Single problem class: storage capacity assessment
- Static snapshot solve — one planning window of inventory; no time index in the model
- Six storage types: Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST
- Six SKU specializations: Dangerous Goods (DG), Temperature Controlled, Heavy and Bulky (HnB), High-Value High-Risk (HVHR), Ship-in-Own-Container (SIOC), Pets
- Each specialization is a binary attribute; a SKU can have zero to all specializations simultaneously
- Specialized SKUs can only be stored at nodes certified for that specialization
- Soft capacity constraints with penalized over-utilization (enables diagnosis of infeasibility)
- Natural-language input → MILP solver → natural-language output via MCP
- Reads NODES, NODE_REST, SKU_MASTER from static repo files at server startup
- Accepts DEMAND and SOH as runtime CSV inputs per session
- Velocity (FAST/SLOW) computed at runtime per PO line — not stored in DEMAND
- All thresholds and solver settings configurable via `config.yaml` — nothing hardcoded
- Agent can operate on any contiguous time window within the provided DEMAND data
- Rolling window analysis: solve across consecutive windows, return per-window results and cross-window summary statistics
- Demand perturbation: agent can modify DEMAND inputs (e.g., increase demand by 20%) and re-solve against baseline
- Comprehensive output and analysis (see Output and analysis section)
- Every solve writes full input and output artifacts to `/logs` for reproducibility

---

## MVP scope — what's out

- Inbound processing capacity constraints (storage only in MVP)
- Outbound capacity constraints
- Facility layout constraints
- Design mode (capacities as decision variables)
- Multi-period or time-phased optimization
- DOC-based safety stock and lead-time math (inventory/demand volume treated as given input)
- SKU-level inbound processing capacity modeling
- Arc definitions and inter-node flow modeling (nodes only in MVP)

---

## Output and analysis

Every solve returns output at three levels: headline, detailed, and visual.

### Headline output
- **Feasibility verdict:** Feasible / Infeasible (with soft-constraint penalty magnitude surfaced alongside — high penalty signals tight or violated constraints even in a feasible solution)
- **Overall network utilization:** Total used CBM vs. total available CBM across the network, by storage type
- **Top binding constraints:** Ranked list of the most constrained node × storage type combinations

### Detailed output — per node
For each node, report:
- Utilization by storage type: used CBM, available CBM, utilization % for each of the six storage types
- Headroom: remaining CBM per storage type
- Breach flag and breach magnitude (CBM over capacity) per storage type
- Specialization utilization: for each of the six specializations, whether the node is certified, how much certified capacity is used, and whether a breach occurred
- Penalty contribution: how much of the total objective penalty comes from this node

### Detailed output — by storage type (network-wide)
For each of the six storage types across the full network:
- Total capacity, total used, total headroom, utilization %
- Number of nodes breached and breach magnitude
- Distribution of utilization across nodes (min, max, mean, median, p25, p75)

### Detailed output — by specialization (network-wide)
For each of the six specializations:
- Total certified capacity across certified nodes
- Total specialization demand (CBM)
- Utilization % and breach flag
- Nodes where specialization capacity is binding

### Rolling window analysis (use case 2)
When the agent solves across multiple consecutive windows:

**Per-window results:**
- Feasibility verdict per window
- Utilization by storage type per window
- Binding constraints per window
- Breach count and magnitude per window

**Cross-window summary statistics (across all windows):**
- Feasibility rate: % of windows that are feasible
- Utilization per storage type: mean, min, max, range, p25, p75, p90
- Breach frequency: how often each node × storage type combination breaches across windows
- Most persistently constrained nodes: ranked by breach frequency
- Distribution of total penalty across windows

### Scenario comparison (use case 3)
When a demand perturbation is applied:
- Side-by-side baseline vs. perturbed: utilization by node × storage type, breach count, total penalty
- Delta table: which nodes/storage types got tighter, which got looser
- New breaches introduced by the perturbation
- Headroom consumed by the perturbation

### Natural-language summary
Every solve returns a plain-English narrative summary alongside the structured output. The summary states: whether inventory fits, the tightest constraint, the most at-risk nodes, and (for rolling window) the most stressed period.

### Visualizations
- **Network map:** nodes plotted by lat/long, colored by overall utilization (green/yellow/red)
- **Utilization heatmap:** node × storage type grid, colored by utilization %
- **Bar charts:** utilization by storage type (network-wide and per node)
- **Rolling window time series:** utilization and breach count across windows
- **Scenario delta chart:** baseline vs. perturbed utilization side by side

---

## Functional requirements

- **Input handling:** Accept natural-language problem descriptions; identify problem class; extract relevant parameters; load NODES, NODE_REST, SKU_MASTER from repo at startup; accept DEMAND and SOH as session-time CSV inputs
- **Preprocessing:** Compute velocity per PO line (`quantity × cbm` vs. `fast_threshold_cbm`); validate capacity_type against config thresholds (warn on mismatch); aggregate SOH by nodeid
- **Data validation:** Validate all inputs before solving — referential integrity (every skuid in DEMAND exists in SKU_MASTER, every nodeid in SOH exists in NODES), schema conformance, missing file detection. Return clear error messages, not stack traces.
- **Configuration:** Load all thresholds and solver settings from `config.yaml` at startup. No hardcoded thresholds anywhere in the codebase.
- **Model construction:** Build MILP from NODES, NODE_REST, SKU_MASTER, DEMAND, SOH, and config
- **Solving:** Call MILP solver; handle infeasibility gracefully via soft constraints with penalties; surface penalty magnitude in output
- **Output:** Return full structured output and natural-language narrative as defined in Output and analysis section; generate visualizations
- **Rolling window:** Solve across consecutive windows of configurable width; compute cross-window summary statistics
- **Scenario construction:** Agent can construct new demand scenarios from natural language (e.g., "increase demand by 20% for totable fast") and re-solve against baseline; return scenario comparison output
- **Solve artifact logging:** Every solve writes a timestamped folder to `/logs/YYYYMMDD_HHMMSS/` containing:
  - `NODES.csv` — full snapshot of NODES at solve time
  - `NODE_REST.csv` — full snapshot of NODE_REST at solve time
  - `SKU_MASTER.csv` — full snapshot of SKU_MASTER at solve time
  - `DEMAND.csv` — the specific demand window used in this solve
  - `SOH.csv` — SOH input (empty file if greenfield)
  - `config.yaml` — config snapshot at solve time
  - `output.json` — full structured output including feasibility, utilization, breaches, penalties, and summary statistics
  - Purpose: any prior solve must be exactly reproducible from its log folder alone
- **Tool surface:** MCP tools must include: `solve`, `validate_input`, `get_baseline`, `explain_result`, `compare_scenarios`, `get_solve_log`

---

## Non-functional requirements

- **Solve time:** Typical instance solves in under 30 seconds on CBC
- **Failure modes:** Malformed input, missing baseline file, schema violations, solver errors — all return clear error messages, not stack traces
- **Determinism:** Same input produces same output (modulo solver tie-breaking)
- **Reproducibility:** Every solve is fully reproducible from its log folder alone — log contains complete input snapshots, config, and complete structured output
- **Config-driven:** No thresholds or solver parameters are hardcoded; all live in `config.yaml`

---

## Success criteria

- A user can describe a hypothetical buying pattern in 1–2 sentences and get a feasible/infeasible answer with full utilization breakdown
- The agent correctly identifies which capacity is binding when infeasibility occurs
- A small hand-crafted example produces the expected solution
- The system handles malformed natural-language input without crashing
- Velocity classification (FAST/SLOW) is correctly computed per PO line at runtime
- Rolling window analysis returns both per-window results and correct cross-window summary statistics
- Demand perturbation produces a correctly re-solved scenario comparison against baseline
- Every solve produces a complete log folder and a prior solve can be exactly reproduced from it
- All thresholds can be changed in `config.yaml` without touching any other code

---

## Open questions

- NODES versioning mechanism — dated filenames vs. `/versions` folder vs. Git history alone
- Exact MILP formulation — pending Phase 8a
- How the agent maps natural-language descriptions to problem class — open
- DEMAND time window selection — how does the user specify which block to analyze (natural language, explicit date range, or week index)?
- Penalty weight calibration — default weights in `config.yaml` are placeholders; need calibration against real instance sizes
- Visualization library — TBD (candidates: matplotlib for static, plotly for interactive)

---

## Dependencies and assumptions

- NODES, NODE_REST, SKU_MASTER are hand-maintained and reasonably current
- Every skuid in DEMAND has a corresponding row in SKU_MASTER
- `capacity_type` in SKU_MASTER is hand-assigned and treated as authoritative; threshold validation is advisory only
- CBC solver is sufficient for MVP problem sizes
- FastMCP framework is stable
- SOH is empty (greenfield) if not provided

---

## Risks

- LLM may incorrectly classify the problem class from natural language → mitigation: validate inputs before solving
- Soft constraint penalty weights may be miscalibrated, obscuring true infeasibility → mitigation: surface penalty magnitude in output; make weights configurable
- NODES/NODE_REST may go stale if not maintained → mitigation: surface file timestamps in agent output
- Rolling window solves may be slow for large DEMAND files and many windows → mitigation: benchmark during Phase 8c; add window-count limit to config if needed

---

## Notes

*Free-form. Reasoning, history, links to related thinking. Move polished items to [[design-decisions]] as they stabilize.*

- Dropped G(N,A) arc framing from MVP — current data model is node-only; arcs are deferred to a future version when inter-node flows are in scope
- NODE_REST is a separate file from NODES because it has a different owner (IPC/DMS team vs. facilities) and a different change cadence
- SOH rows are per inventory parcel, not per node — solver aggregates by nodeid at solve time
- Velocity (FAST/SLOW) is a PO-line attribute, not a SKU attribute — the same SKU can be FAST in one PO line and SLOW in another depending on quantity ordered
- `config.yaml` is the single source of truth for all thresholds; nothing is hardcoded
