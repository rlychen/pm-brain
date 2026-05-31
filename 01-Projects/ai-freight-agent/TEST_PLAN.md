# Test Plan — AI Freight Agent

**Status:** draft (Session 14, 2026-05-23). **Scope:** air model first, end-to-end; extends to LCL / ocean / trucking as those components come online.

## 1. Philosophy

- **Test-as-you-build.** Every component has unit tests before the next component starts. No "I'll add tests later."
- **Regression on every change.** The full suite re-runs on every change.
- **Test pyramid** — many fast unit tests, fewer integration tests, a small set of end-to-end tests.
- **Real, not mocked.** No solver mocking, no graph mocking — real HiGHS, real graphs, small synthetic instances.
- **Verify correctness, not just status.** MILP solution value within a manually-derived bound; not just `solver.status == OPTIMAL`.
- **Fail fast.** First test per component is the happy path; second is an infeasibility / invalid-input case.

## 2. Pyramid

| Layer | Location | Scope | Speed |
|---|---|---|---|
| **Unit** | `tests/components/test_<component>.py` | one component (graph generator, MILP solver, rules engine…) | fast (<1s typical) |
| **Integration** | `tests/integration/test_<flow>.py` | 2+ components together | medium |
| **End-to-end** | `tests/e2e/test_<scenario>.py` | full pipeline (instance → graph → MILP → output verification) | slower |

**Regression** = the whole pyramid, run on every change.

## 3. Rules (from `CLAUDE.md`)

- One test file per component, mirroring `src/components/<x>.py` ↔ `tests/components/test_<x>.py`.
- Shared fixtures in `tests/conftest.py` — small, deterministic, seeded (`random.seed(42)`).
- Test names: `test_<unit>_<scenario>` — diagnose-without-reading-the-body level of detail.
- Every module: **at minimum one happy-path + one infeasibility/invalid-input** test.
- MILP components: verify optimal value against a **manually-derived bound**, not just solver status.
- **Never mock** the HiGHS solver, the graph, or database state — real small instances, in-memory or tmp fixtures.
- **No timing assertions** in unit tests; performance is a separate, deferred suite.

## 4. Per-component tests — air model

### 4.1 `air_graph` — graph generator (Phase 1 + Phase 2)

File: `src/components/air_graph.py` · Tests: `tests/components/test_air_graph.py`

**Phase 1 (physical graph):**
- `test_air_graph_phase1_direct_journey` — `O → CFS-O → POL → POD → CFS-D → Customs → D`; correct nodes and arcs.
- `test_air_graph_phase1_hub_with_cfsh` — hub variant; CFS-H present only at forwarder-operated hubs.
- `test_air_graph_phase1_hub_without_cfsh` — non-forwarder hub allows carrier-side connection only.
- `test_air_graph_phase1_cfs_on_vs_off_airport` — cartage time/cost differs correctly.
- `test_air_graph_phase1_customs_dwell_per_hawb` — per-HAWB `δ_cust` applied between `CFS-D` and final delivery.
- `test_air_graph_phase1_per_shipment_subgraph_prefilter` — `cargo_type_ok`, embargo, lithium, screening, deadline reachability prune correctly.
- `test_air_graph_phase1_empty_subgraph_infeasibility` — `A_k = ∅` → structured rescue event (not exception).

**Phase 2 (MAWB overlay):**
- `test_air_graph_phase2_group_g_attribute_tuple` — `g(k) = (cargo_class, screening, temp)` for consolidable; `(class, HAWB-id)` for VAL/HUM/AVI.
- `test_air_graph_phase2_partition_property` — groups are pairwise disjoint; every HAWB in exactly one group.
- `test_air_graph_phase2_val_hum_avi_singleton` — non-consolidable cargo → singleton groups.
- `test_air_graph_phase2_mawb_instantiation` — for each MAWB-arc, one MAWB per distinct `g` present.
- `test_air_graph_phase2_coload_no_mawb` — co-load arcs skip MAWB instantiation.

### 4.2 `air_milp` — MILP build and solve

File: `src/components/air_milp.py` · Tests: `tests/components/test_air_milp.py`

**Core:**
- `test_air_milp_trivial_feasibility` — 1 shipment, 1 direct arc, feasible.
- `test_air_milp_cheapest_arc_selected` — 2 options, optimizer picks the cheaper.
- `test_air_milp_infeasibility_no_path` — no-path shipment → structured infeasibility output.
- `test_air_milp_infeasibility_deadline_too_tight` — deadline before any feasible arrival → infeasibility.
- `test_air_milp_capacity_binding` — physical capacity blocks an otherwise cheaper route.

**Rate families:**
- `test_air_milp_rate_family_flat_rate` — NAC / spot single flat $/kg correct.
- `test_air_milp_rate_family_min_flat_breaks_tact` — TACT next-break-down with `γ` binary; `$960` at 200 kg (vs `$1,052.50` from a wrong cumulative formula).
- `test_air_milp_rate_family_per_uld_pivot_bsa` — BSA pivot floor binds; cost = `r_c · max(CW, π·z)`.

**Bucket / consolidation:**
- `test_air_milp_density_mixing` — dense + light HAWBs on one MAWB; `CW = max(Σw, Σ volumetric)`.
- `test_air_milp_consolidation_group_split` — DGR + GEN on same arc → 2 separate MAWBs.

**BSA settlement basis:**
- `test_air_milp_settlement_per_flight` — per-flight pivot binds each flight.
- `test_air_milp_settlement_equalized_allowance` — sunk allowance `A_c` → 2-segment cost on bucket aggregate.

**Item 3 (soft deadline):**
- `test_air_milp_soft_deadline_tardiness` — late shipment → `τ_k > 0`; objective gains `w·τ_k`.
- `test_air_milp_hard_backstop_T_abs` — past absolute drop-dead → infeasible.

**Solution-value bound:**
- `test_air_milp_solution_value_upper_bound` — optimal cost ≤ a manually computed upper bound on a tiny instance.

### 4.3 Future components (add tests as built)
- `transit_time_service` — per-arc-type quantile estimates.
- `rules_engine` — carrier policy cascade, embargo, lithium, screening.
- `mcp_server` — schema rejection on malformed input; stdout cleanliness on tool-call path; output-schema verification.
- `stitching` — multimodal handoff (LCL → trucking, etc.).

## 5. Test fixtures (`tests/conftest.py`)

Shared deterministic fixtures:
- `tiny_air_network` — 4 airports (TPE, HKG, JFK, LAX), 3 flights, 1 hub (HKG).
- `synthetic_offers` — 1 hard BSA, 1 soft BSA, 1 TACT card, 1 NAC, 1 spot.
- `synthetic_shipments` — 6 HAWBs covering the 6 cargo classes (GEN / PER / DGR / VAL / HUM / AVI) with varied screening and temperature.
- `tiny_solver_instance` — `(graph, shipments, offers)` ready for MILP build.

All fixtures use `random.seed(42)`.

## 6. End-to-end scenarios (`tests/e2e/`)

Each runs the full pipeline (`graph build → MILP solve → verify outputs`):
1. `test_e2e_3_shipments_direct` — 3 GEN shipments, direct flights, no hub.
2. `test_e2e_hub_routing_consolidation` — 1 shipment routed via hub; verify multi-arc path + capacity coupling.
3. `test_e2e_dgr_group_separation` — 1 GEN + 1 DGR on same flight → 2 MAWBs.
4. `test_e2e_coload_arc_per_kg` — 1 shipment on a co-load arc; per-kg cost; no MAWB instantiated.
5. `test_e2e_consolidation_savings` — 3 GEN shipments same O-D; consolidated cost < sum of individual through-MAWB costs.

## 7. Running tests

```bash
uv run pytest                        # full suite (regression)
uv run pytest tests/components/      # unit only
uv run pytest tests/integration/     # integration only
uv run pytest tests/e2e/             # end-to-end only
uv run pytest -k "test_air_milp"     # subset by name
uv run pytest -x                     # stop on first failure
```

## 8. Regression policy

- **Active development:** unit tests run on save (e.g.\ `pytest-watch` or IDE integration).
- **Before commit:** full `uv run pytest` must pass.
- **After any formulation / spec change:** full re-run.
- **CI/CD:** deferred to Phase 5 (production-readiness); manual `pytest` discipline until then.

## 9. Anti-patterns we avoid

- Mocking the HiGHS solver. Mocking the graph. Mocking database state.
- Skipping tests to land code faster.
- Timing assertions inside unit tests (performance is a separate, deferred suite).
- "I'll add tests later."

## 10. Walking-skeleton observability (from spec §13.1 instrumentation)

The spec's tractability section identifies eight metrics whose empirical values
drive every later scaling decision. **Wire them into the solve loop output from
v1 of the walking skeleton onward** — they cost nothing extra at small scale
and answer the questions that bite at production scale before they bite.

Output to structured log files alongside each `AirSolution`. All metrics are
**shadow-mode** — no behavioral change to the solve; pure observation.

| Metric | Output file | Decision it informs |
|---|---|---|
| Per-HAWB `|A_k|` histogram + per-pre-filter-predicate drop counts (the 8 §4 steps) | `pre_filter_stats.jsonl` (one record per HAWB per solve) | Determines `x_{k,a}` binary count. The single most important number for scaling. Refuse to scale up `|K|` until this is empirical. |
| `|G_a|` distribution per arc + activated bucket count `Σ z_{a,g}` post-solve | `bucket_stats.jsonl` (one record per solve) | Drives `|M|` and `γ` count estimates. |
| LP-vs-MIP gap broken down by constraint family | `lp_gap_breakdown.json` | Identifies which constraint family loosens the LP relaxation most (C.6 expected to dominate). |
| Realized big-M slack on C.6 and C.9 — warn if median > 0.5·M | `bigm_slack_warning.json` | Empirical confirmation that the per-shipment tight-M values in spec §8.1 actually bind. |
| Connected components of H (HAWB-sharing-arc graph) — **shadow** (compute, don't decompose) | `component_decomposition_shadow.jsonl` | Indicator for when monolithic-vs-decomposed mandate kicks in at Phase-2. |
| Super-shipment equivalence classes — **shadow** | `aggregation_potential = 1 − |classes|/|K|` in `aggregation_shadow.jsonl` | Decides default value for `consolidation_mode` toggle in v3+. |
| BSA-contract cross-coupling fraction (fraction of components with shared BSA contracts) | `bsa_coupling_fraction` in solve summary | Predicts Lagrangian-relaxation value for Phase-2 decomposition. |
| Post-solve invariant assertions — `CW = max(Wt, Wv)`, `τ_k = max(0, lateness)`, `Σ_g η ≤ N_{a,u}`, no `z=1` with empty bucket | `invariant_violations.json` (empty on success) | Catches silent bugs — especially monotonicity violations if a future rate-family extension adds a negative-coefficient term. |

Plus HiGHS phase breakdown (presolve / LP root / B&B / cuts) via solver
callbacks → `solve_phase_times.json`. Tells you where the time actually goes;
guides which constraint family to attack next.

**Implementation note.** Don't gate v1 on having all metrics live — instrument
incrementally. `pre_filter_stats.jsonl` and the invariant assertions are
mandatory from v1. The others can land in v4 (the dedicated instrumentation
stage of Stage 4 in CONTEXT.md).

## 11. Future test layers (deferred)

- Performance / load tests on production-scale instances.
- Property-based tests (Hypothesis) — partition disjointness, density-mixing identity, monotonicity of `R_o`, etc.
- Stochastic tests (when probabilistic transit-time / exam-hold modelling lands).
- Multi-mode integration (LCL + trucking + air, stitched).
