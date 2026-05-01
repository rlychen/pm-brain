
*Status: draft*

---

## Problem statement

Network designers need a way to quickly assess whether the current network of fulfillment centers (FCs), transfer fulfillment centers/vendor flex (TXFCs), and replenishment centers (RCs) can store the inventory implied by current buying patterns, given:

- Fixed storage capacities across six storage types: Totable SLOW/FAST, Grander SLOW/FAST, Non-totable SLOW/FAST
- Node-level storage restrictions enforced by DMS layer rules (network specialization constraining what capacity types, velocity FAST/SLOW, and specialized SKUs can go into each node)
- SKU specializations that restrict which nodes can receive which SKUs. Specialized SKUs can only go to nodes certified for that specialization.

Today this assessment is manual, slow, and ad-hoc. There is no analytic model to answer these questions.

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

These thresholds are configurable in `config.yaml`. The thresholds are used for validation only (flag mismatches as warnings, not hard failures) — `capacity_type` in SKU_MASTER is the authoritative classification.

### Velocity (PO line attribute, computed at runtime)

Velocity — FAST or SLOW — is not stored in DEMAND. It is computed at runtime per PO line from quantity and SKU CBM:

```
po_line_volume_cbm = quantity × skuid.cbm
velocity = FAST if po_line_volume_cbm >= FAST_THRESHOLD_CBM else SLOW
storage_type = capacity_type + "_" + velocity   # e.g., grande_fast
```

FAST_THRESHOLD_CBM` defaults to 0.8 CBM and is configurable in `config.yaml`. The same SKU can be FAST in one PO line and SLOW in another depending on the quantity ordered. Velocity is computed in the preprocessing step before the MILP is built.

---

## Configuration

All tunable thresholds and output settings live in `config.yaml` at the repo root. This file is version controlled. No thresholds are hardcoded in the solver or agent.

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

# Penalty weights
capacity_penalty_weight: 1000       # Per CBM of capacity over-utilization
infeas_penalty_weight: 100000       # Per unit unassigned fraction when Vp = empty
                                    # Must be >> capacity_penalty_weight

# Output settings
output:
  tables: true                      # Render structured tables to terminal per solve
  log_data: true                    # Write input/output data files to solve log folder
  charts_auto: false                # true = generate Plotly HTML charts automatically per solve
                                    # false = on demand via render_chart tool only
  chart_format: html                # html only (Plotly interactive)

# Rolling window analysis
rolling_window_stats_granularity: storage_type  # network | storage_type | node
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

**NODE_REST** — node-level storage restrictions. One row per node. Binary flags indicate whether that node can store each type. CBM columns define specialization capacity limits.

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
| dg_cbm | float | DG capacity limit (CBM); -1 = no limit enforced |
| temp_cbm | float | Temp-controlled capacity limit (CBM); -1 = no limit enforced |
| hnb_cbm | float | Heavy & Bulky capacity limit (CBM); -1 = no limit enforced |
| hvhr_cbm | float | HVHR capacity limit (CBM); -1 = no limit enforced |
| sioc_cbm | float | SIOC capacity limit (CBM); -1 = no limit enforced |
| pets_cbm | float | Pets capacity limit (CBM); -1 = no limit enforced |

A node with binary certification = 0 cannot receive that specialization regardless of the CBM column. A node certified (binary = 1) with CBM = -1 is certified but uncapacitated for that specialization. Owned by IPC team that manages DMS layer rules and case mappings. Git-tracked alongside NODES. Maintained separately from NODES because it has a different owner and change cadence.

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
- Specialization capacity limits per node (CBM) where defined; -1 disables the limit
- Storage-type and specialization restrictions are hard exclusions — valid node sets computed per PO line before solving
- Soft capacity constraints with penalized over-utilization (enables diagnosis of infeasibility)
- Infeasibility penalty for PO lines with no valid node in the network
- Natural-language input → MILP solver → natural-language output via MCP
- Reads NODES, NODE_REST, SKU_MASTER from static repo files at server startup
- Accepts DEMAND and SOH as runtime CSV inputs per session
- Velocity (FAST/SLOW) computed at runtime per PO line — not stored in DEMAND
- All thresholds and solver settings configurable via `config.yaml` — nothing hardcoded
- Agent can operate on any contiguous time window within the provided DEMAND data
- Rolling window analysis: solve across consecutive windows, return per-window results and cross-window summary statistics
- Demand perturbation: agent can modify DEMAND inputs (e.g., increase demand by 20%) and re-solve against baseline
- Comprehensive structured output across seven layers (see Output and analysis section)
- Charts rendered on demand via Plotly HTML, or auto-generated per solve (config-controlled)
- Every solve writes full input and output artifacts to timestamped `/logs` folder for reproducibility
- Test instance generator for producing synthetic input files of configurable size and tightness

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

Every solve returns output across seven layers. Layers 1–5 apply to a single solve. Layers 6–7 apply to rolling window and scenario analyses respectively.

### Layer 1 — Network headline

Single verdict row:

`feasibility_verdict | total_cap_cbm | total_soh_cbm | total_new_demand_cbm | total_used_cbm | util_pct | nodes_breached`

Feasibility verdict: Feasible if objective = 0. If Term 1 > 0: capacity squeezed but all PO lines have valid nodes. If Term 2 > 0: at least one PO line has no valid node.

### Layer 2 — Storage type profile (network-wide)

Six rows, one per storage type. Sorted by `util_pct` descending.

`storage_type | total_cap_cbm | soh_cbm | new_demand_cbm | total_used_cbm | util_pct | headroom_cbm | nodes_breached`

### Layer 3 — Node profile

One row per node. Six storage types shown as column groups side by side. Each storage type has three columns: cap, used, util_pct.

`nodeid | nodename | nodetype | tot_fast_cap | tot_fast_used | tot_fast_util_pct | tot_slow_cap | tot_slow_used | tot_slow_util_pct | grd_fast_cap | grd_fast_used | grd_fast_util_pct | grd_slow_cap | grd_slow_used | grd_slow_util_pct | nt_fast_cap | nt_fast_used | nt_fast_util_pct | nt_slow_cap | nt_slow_used | nt_slow_util_pct | any_breach`

`any_breach` is a flag if any storage type at this node exceeds capacity.

### Layer 4 — Specialization profile

One row per node × specialization combination. For capped specializations (CBM ≠ -1): show cap, used, util_pct, breach flag. For uncapped (CBM = -1): show actual usage only; cap and util_pct as null.

`nodeid | nodename | specialization | cap_cbm | soh_cbm | new_demand_cbm | total_used_cbm | util_pct | headroom_cbm | breached`

### Layer 5 — Infeasibility report

Only rendered when any PO line has no valid node (u_p > 0). Grouped by gap reason.

`poid | skuid | storage_type | specializations_required | vol_cbm | gap_reason`

Gap reasons: no certified node for storage type; no certified node for specialization combination; both.

### Layer 6 — Rolling window summary

**Per-window results table** — one row per window:

`window_id | start_date | end_date | feasibility_verdict | total_used_cbm | total_cap_cbm | util_pct | nodes_breached | infeasible_po_lines`

**Cross-window summary statistics** — granularity controlled by `rolling_window_stats_granularity` in `config.yaml` (network | storage_type | node):

`dimension | mean | min | max | p25 | p75 | p90 | breach_frequency`

### Layer 7 — Scenario comparison

Full Layers 1–4 rendered side by side for baseline and perturbed scenario. Plus delta summary below:

`storage_type | baseline_util_pct | perturbed_util_pct | delta_util_pct | new_breach`

### Natural-language summary

Every solve returns a plain-English narrative alongside the structured output stating: whether inventory fits, the tightest constraint, the most at-risk nodes, and (for rolling window) the most stressed period.

### Visualizations

Charts are Plotly interactive HTML. Behavior controlled by `charts_auto` in `config.yaml`:
- `charts_auto: true` — charts generated automatically per solve and written to the solve log folder
- `charts_auto: false` — charts generated on demand via `render_chart` MCP tool

Available chart types (`chart_type` enum for `render_chart`):
- `network_gauge` — network utilization KPI (Layer 1)
- `storage_type_bar` — horizontal bar chart, utilization by storage type (Layer 2)
- `node_heatmap` — node × storage type heatmap, colored by utilization %; cells with no capacity render grey/null (Layer 3)
- `specialization_bar` — utilization by specialization grouped by node (Layer 4)
- `rolling_window_series` — line chart of utilization across windows (Layer 6)
- `scenario_comparison` — grouped bar chart, baseline vs. perturbed (Layer 7)

---

## Test instance generator

A standalone Python script (`scripts/generate_instance.py`) that produces synthetic input files matching the exact schemas of all five data files. Used for testing the solver at varying sizes and constraint tightness levels. Controlled by a single config file (`instance_config.yaml`).

### Generator parameters

```yaml
# instance_config.yaml

# Network structure
num_fc: 3                          # number of FC nodes
num_txfc: 1                        # number of TXFC nodes
num_rc: 2                          # number of RC nodes

# Node capacity
capacity_mean_cbm: 500             # mean total CBM capacity per node
capacity_stddev_cbm: 100           # stddev across nodes (controls unevenness)
capacity_split:                    # fraction of total CBM per storage type
  totable_fast: 0.20
  totable_slow: 0.20
  grande_fast: 0.15
  grande_slow: 0.15
  nt_fast: 0.15
  nt_slow: 0.15

# Node restrictions
storage_type_restriction_rate: 0.8  # probability a node allows a given storage type
specialization_cert_rate: 0.4       # probability a node is certified for a given specialization
specialization_cap_rate: 0.5        # probability a certified node has a CBM cap (vs -1)
specialization_cap_fraction: 0.2    # cap = this fraction of node total capacity

# SKU master
num_skus: 50                        # number of SKUs in SKU_MASTER
capacity_type_mix:                  # fraction of SKUs per capacity type
  totable: 0.50
  grande: 0.30
  non_totable: 0.20
specialization_rate: 0.15           # probability a SKU has any specialization
multi_spec_rate: 0.10               # probability a specialized SKU has > 1 specialization
                                    # set to 0 for single-specialization SKUs only

# Demand
num_po_lines: 500                   # total PO lines in DEMAND
date_range_days: 90                 # span of order/delivery dates
fast_fraction: 0.40                 # target fraction of PO lines classified as FAST

# Tightness control
network_load_factor: 0.85           # total demand volume as fraction of total network capacity
                                    # < 1.0 = fits with headroom
                                    # ~1.0 = tight
                                    # > 1.0 = guaranteed capacity breach

# SOH
include_soh: false                  # whether to generate a non-empty SOH file
soh_load_factor: 0.20               # SOH pre-fills this fraction of capacity before new demand

# Infeasibility injection
inject_infeasible_pos: 0            # number of PO lines to make deliberately infeasible
                                    # (specialization combo with no certified node anywhere)

# Reproducibility
random_seed: 42                     # fix for reproducible instance generation
```

### Generator outputs

Running `python scripts/generate_instance.py --config instance_config.yaml --output data/test/` produces:
- `NODES.csv`
- `NODE_REST.csv`
- `SKU_MASTER.csv`
- `DEMAND.csv`
- `SOH.csv` (empty if `include_soh: false`)
- `instance_config.yaml` snapshot (copy of config used)

All files match the exact column schemas defined in the Data model section.

### Key design notes

`network_load_factor` is the primary tightness control. It scales demand volumes so that total demand CBM = `network_load_factor × total_network_capacity_cbm`. Use it independently from `inject_infeasible_pos` to separately control capacity tightness vs. specialization infeasibility:
- Capacity-tight, all PO lines placeable: `network_load_factor: 1.1`, `inject_infeasible_pos: 0`
- Comfortable capacity, some unplaceable PO lines: `network_load_factor: 0.7`, `inject_infeasible_pos: 5`
- Both tight: `network_load_factor: 1.1`, `inject_infeasible_pos: 5`

`multi_spec_rate` controls how many specialized SKUs carry more than one specialization flag, making them harder to place. Setting it to 0 means every specialized SKU has exactly one specialization.

---

## Functional requirements

- **Input handling:** Accept natural-language problem descriptions; identify problem class; extract relevant parameters; load NODES, NODE_REST, SKU_MASTER from repo at startup; accept DEMAND and SOH as session-time CSV inputs
- **Preprocessing:** Compute velocity per PO line (`quantity × cbm` vs. `fast_threshold_cbm`); validate capacity_type against config thresholds (warn on mismatch); aggregate SOH by nodeid; compute valid node sets per PO line
- **Data validation:** Validate all inputs before solving — referential integrity (every skuid in DEMAND exists in SKU_MASTER, every nodeid in SOH exists in NODES), schema conformance, missing file detection. Return clear error messages, not stack traces.
- **Configuration:** Load all thresholds and solver settings from `config.yaml` at startup. No hardcoded thresholds anywhere in the codebase.
- **Model construction:** Build MILP from NODES, NODE_REST, SKU_MASTER, DEMAND, SOH, and config; apply valid node sets as hard assignment constraints
- **Solving:** Call MILP solver; handle infeasibility gracefully via soft capacity constraints and infeasibility penalty; surface penalty breakdown in output
- **Output:** Return all seven output layers as structured data; render tables to terminal if `output.tables: true`; generate natural-language narrative summary
- **Rolling window:** Solve across consecutive windows of configurable width; compute cross-window summary statistics at granularity set by `rolling_window_stats_granularity`
- **Scenario construction:** Agent can construct new demand scenarios from natural language (e.g., "increase demand by 20% for totable fast") and re-solve against baseline; return Layer 7 scenario comparison
- **Visualizations:** When `charts_auto: true`, generate all Plotly HTML charts per solve and write to log folder. When `charts_auto: false`, generate on demand via `render_chart` tool.
- **Solve artifact logging:** Every solve writes a timestamped folder to `/logs/YYYYMMDD_HHMMSS/` containing:
  - `NODES.csv` — full snapshot of NODES at solve time
  - `NODE_REST.csv` — full snapshot of NODE_REST at solve time
  - `SKU_MASTER.csv` — full snapshot of SKU_MASTER at solve time
  - `DEMAND.csv` — the specific demand window used in this solve
  - `SOH.csv` — SOH input (empty file if greenfield)
  - `config.yaml` — config snapshot at solve time
  - `output.json` — full structured output (all layers)
  - Chart HTML files (if `charts_auto: true`)
  - Purpose: any prior solve must be exactly reproducible from its log folder alone
- **Tool surface:** MCP tools must include: `solve`, `validate_input`, `get_baseline`, `explain_result`, `compare_scenarios`, `get_solve_log`, `render_chart`

---

## Non-functional requirements

- **Solve time:** Typical instance solves in under 30 seconds on CBC
- **Failure modes:** Malformed input, missing baseline file, schema violations, solver errors — all return clear error messages, not stack traces
- **Determinism:** Same input produces same output (modulo solver tie-breaking)
- **Reproducibility:** Every solve is fully reproducible from its log folder alone — log contains complete input snapshots, config, and complete structured output
- **Config-driven:** No thresholds, penalty weights, or solver parameters are hardcoded; all live in `config.yaml`

---

## Success criteria

- A user can describe a hypothetical buying pattern in 1–2 sentences and get a feasible/infeasible answer with full utilization breakdown across all seven output layers
- The agent correctly identifies which capacity is binding when infeasibility occurs
- A small hand-crafted example produces the expected solution
- The system handles malformed natural-language input without crashing
- Velocity classification (FAST/SLOW) is correctly computed per PO line at runtime
- Valid node sets correctly exclude nodes that fail storage-type or specialization restrictions
- PO lines with no valid node are correctly identified and reported in Layer 5
- Rolling window analysis returns both per-window results and correct cross-window summary statistics at configured granularity
- Demand perturbation produces a correctly re-solved Layer 7 scenario comparison against baseline
- Every solve produces a complete log folder and a prior solve can be exactly reproduced from it
- All thresholds can be changed in `config.yaml` without touching any other code
- Test instance generator produces valid input files for any combination of generator parameters
- `network_load_factor` correctly controls overall demand tightness relative to network capacity

---

## Open questions

- NODES versioning mechanism — dated filenames vs. `/versions` folder vs. Git history alone
- How the agent maps natural-language descriptions to problem class — open
- DEMAND time window selection — how does the user specify which block to analyze (natural language, explicit date range, or week index)?
- Penalty weight calibration — default weights in `config.yaml` are placeholders; calibrate during Phase 8c against test instances
- Integer assignment — should PO lines be assigned integrally (one node per line)? Converts LP to MIP. Deferred to Phase 8c after benchmarking LP solve times.
- Separate penalty weights for storage capacity vs. specialization capacity — currently both use `capacity_penalty_weight`; add separate weights in `config.yaml` if operational priority differs

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
- Soft constraint penalty weights may be miscalibrated, obscuring true infeasibility → mitigation: surface penalty breakdown in output; make weights configurable
- NODES/NODE_REST may go stale if not maintained → mitigation: surface file timestamps in agent output
- Rolling window solves may be slow for large DEMAND files and many windows → mitigation: benchmark during Phase 8c; add window-count limit to config if needed
- Test instance generator may produce degenerate instances at extreme parameter settings → mitigation: generator logs warnings when valid node sets are sparse

---

## Notes

*Free-form. Reasoning, history, links to related thinking. Move polished items to [[design-decisions]] as they stabilize.*

- Dropped G(N,A) arc framing from MVP — current data model is node-only; arcs are deferred to a future version when inter-node flows are in scope
- NODE_REST is a separate file from NODES because it has a different owner (IPC/DMS team vs. facilities) and a different change cadence
- SOH rows are per inventory parcel, not per node — solver aggregates by nodeid at solve time
- Velocity (FAST/SLOW) is a PO-line attribute, not a SKU attribute — the same SKU can be FAST in one PO line and SLOW in another depending on quantity ordered
- Storage-type and specialization restrictions are hard exclusions enforced via valid node sets — not soft penalties; M_type and M_spec removed from model
- `config.yaml` is the single source of truth for all thresholds; nothing is hardcoded
- Charts use Plotly interactive HTML only; chart generation is config-controlled (auto per solve or on demand)
- Node heatmap: cells where a node has no capacity for a storage type render grey/null; zero utilization renders at bottom of color scale
- UI interaction model: user interacts via Claude Code terminal in natural language; charts open in browser as Plotly HTML; future path to standalone web app (React + Plotly + FastAPI) after MVP solver is proven
- Phase 8a (LaTeX MILP formulation) complete — see `model/network_design.tex`
- Phase 8c (minimal Python solver) is next
