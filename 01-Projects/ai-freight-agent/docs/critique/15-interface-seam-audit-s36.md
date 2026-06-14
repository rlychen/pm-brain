# Interface-Seam / Contract Audit — Session 36

**Scope:** cross-seam contract drift only (producer emits X, consumer expects Y, does X==Y?).
Read-only review. Focus: the new Slice-A zero-count-allotment supply path + the standing seam list.

**Verdict headline:** No BLOCKING findings. The new zero-count-allotment path is **lossless end-to-end and verified empirically**. Two MATERIAL items are pre-existing documentation/round-trip-completeness gaps, both already known-deferred (replay loop / Postgres parity), not live bugs. Several MINOR notes.

---

## NEW PATH (S36) — zero-count allotment chain

**Producer→consumer chain audited:** `_draw_network_supply` → `_build_rate_catalog` (`BsaContract.allotment={LD3:0}`) → `scenario_io.persist` (`air_uld_allocations.ulds_per_departure=0`) → `load`/`_load_bsa` → `air_milp` `_build_bsa_vars`/C.5/C.5b.

**Empirical verification (PYTHONHASHSEED=0, default GenConfig kappa=1.0):**
- In-mem: `{'AIR:cx_hkg_lax': {'LD3': 0}, 'AIR:cx_hkg_ord': {'LD3': 13}}`
- Persisted rows: `('cx_hkg_lax','LD3',0)`, `('cx_hkg_ord','LD3',13)` — **zero row kept, not filtered.**
- Loaded back: identical dict, zero entry preserved.
- `solve(orig).total_cost == solve(loaded).total_cost` (56398.991) and `routes` identical.

**Contract-match verdict: PASS (clean).** Details:
- `_build_rate_catalog` line 263 writes `{"LD3": allotment_counts.get(arc, 0)}` for *every* per-ULD arc — a zero-count flight stays in `allotment` (and therefore in `BsaContract.arcs`, priced), never dropped. Correct per §13 (dropping would make the arc free-of-charge).
- `_persist_bsa` (scenario_io 227–234) iterates `sorted(c.arcs)` and writes one `air_uld_allocations` row per `(arc, uld_type)` with `ulds_per_departure=count`; **no `if count > 0` filter**. The zero row persists.
- `_load_bsa` (348–363) reconstructs `allotment[arc][uld]=ulds_per_departure` for every row; no positivity assumption.
- `air_milp._build_bsa_vars` 571–572: `eta[(a,g,'LD3')] = h.addIntegral(0, 0)` — a degenerate integer var pinned to 0. **`addIntegral(0,0)` is valid in highspy** (verified). C.5b then forces `w_lhs ≤ 1500·η = 0` and `v_lhs ≤ 0`, so no cargo can ride the zero-capacity contracted MAWB. The arc remains a priced option with zero usable capacity — exactly the intended emergent-scarcity mechanic.
- The existing `test_write_scenario_roundtrips_to_identical_solve` (default kappa) **already exercises a zero-count arc** because the static `build_tpeb_instance` has only 2 CX arcs and one draws 0 at kappa=1.0. So the regression is covered, if implicitly.

**No consumer assumes positive counts, filters zero rows, or breaks the round-trip. RateCatalog round-trips bit-identically through SQLite with the zero-count row present.**

---

## Standing seam list

### PASS (verified clean)

| Seam | Producer | Consumer | Verdict |
|---|---|---|---|
| Zero-count allotment | generator/persist | load/MILP C.5/C.5b | **PASS** (empirical, above) |
| `supply` RNG-stream CRN | `_draw_network_supply` on `"supply"` | demand/leg/component streams | **PASS** — `test_capacity_axis_does_not_shift_demand_or_leg_actuals` proves kappa∈{2.0,0.5} leaves `shipments`/`leg_actuals`/`component_actuals` byte-identical while `air_uld_allocations` moves. Supply rides its own stream; the variable draw-count (`gammavariate`×len(arcs) + `random`×total_N) cannot desync demand. |
| 2b `sample_route`/`route_reliability` realization | `sample_leg_block`/`sample_component_delta` | generator pre-sampler `_presample_legs`/`_presample_components` | **PASS** — both call the *same* primitives with the same `TransitConfig`, so a frozen `leg_actuals.realized_block_h` is byte-identical to what the walk would draw. Block stored (clock-independent), dep/arr walk-derived — matches the schema comment's design. |
| `AirSolution.routes` ordering | `_extract_solution` → `order_route` | 2b `sample_route` (walks arcs on a running clock) / 2c | **PASS** — `routes={k: order_route(rs, arcs)}` emits tail→head traversal order; `sample_route` consumes arcs in that order. Contract satisfied at the producer. (2c not yet built; nothing to drift against yet.) |
| `TierSpec` single source | `flexibility.TIER_SPECS` | 2a `_gen_arrivals`, `derive_deadline`, C.10 weight | **PASS** — one table. `_gen_arrivals` reads `TIER_SPECS[tier].w_sp` (746) and calls `derive_deadline` (744) which reads `TIER_SPECS[tier].sla_offset_h`. C.10 reads the *folded* `tardiness_weight`/`soft_deadline_h` off the Hawb (not the spec directly), so no duplicated tier constants anywhere. `validate_tier_specs()` runs at import. |
| κ/α reproducibility persistence | `GenConfig`/`ArrivalConfig` | `write_config` config.json | **PASS** — both `write_scenario` (556–564) and `write_arrival_scenario` (830–841) persist `kappa` + `alpha`; a scenario is reproducible from its config key. |
| `capacity_scale` purge | — | — | **PASS** — zero stale readers in `src/`, `data/`, `tests/`. Only residue is one doc line (below). `n_uld`-as-κ fully gone. |

### MATERIAL

**M-1 — `load()` is not a full inverse of `persist()` for arrival scenarios.**
- Producer: `_persist_hawbs` writes `tier`, `known_at`, `ready_at`, `effective_deadline_at` when `arrivals` is supplied (scenario_io 122–130).
- Consumer: `_load_hawbs` (271–286) reconstructs **only** `soft_deadline_h` (+ the graph scalars). It never reads back `tier`, `known_at`, `ready_at`, or `effective_deadline_at`, and `SimInputs` has no field for them.
- Why it's not BLOCKING: the `load → SimInputs` path feeds the *static* `build_air_graph + solve`, which legitimately doesn't need arrival metadata; the replay loop (2c) reads those columns via the `visible_shipments` SQL view, not via `load`. So today nothing is broken.
- Risk: the contract "`load` reconstructs the instance" is **partial**. When 2c lands, an author may reasonably expect `load` to round-trip the arrival stream and silently get HAWBs with `tier=None`. The asymmetry is undocumented in `scenario_io.load`'s docstring (which says "reconstruct the build_air_graph + solve inputs" — technically accurate but easy to over-read). **Recommend:** either add a `load_arrivals()` companion before 2c, or a one-line docstring note that arrival columns are view-only and not part of `SimInputs`.

**M-2 — `effective_deadline_at` vs `soft_deadline_h`: same value, two columns, divergent read paths.**
- Both columns carry `Δ_k`. The generator stamps `delta_k` onto **both** (`air_generator` 746 → `soft_deadline_h`; 749 → `effective_deadline_at`), so they agree *by construction at generation*.
- But the **MILP penalizes `soft_deadline_h`** (`air_milp` 745 `soft = hawb.soft_deadline_h`) while **`data_model.md` (line 509) and the schema comment (scenario_db 97) declare `effective_deadline_at` as "the OTP/predicate-9 binding deadline"**. Two columns are the authoritative Δ_k depending on which doc you read.
- Why it's not BLOCKING: they're equal today, and the MILP and 2-FLEX both flow from the same `derive_deadline` output. No numeric drift exists.
- Risk: this is a latent foot-gun. The day a `T_dead` tightening or a frozen-promise (`booking_promise.promised_deadline_at`) path updates one column and not the other, the optimizer (reads `soft_deadline_h`) and the reveal/scoring layer (data-model says `effective_deadline_at`) will silently disagree on the deadline. **Recommend:** pick one as canonical Δ_k and make the other a documented denormalized echo (or drop it). At minimum reconcile the two identical "Δ_k = min(T_dead,T_SLA)" comments so it's explicit they must stay equal.

### MINOR

- **`spot_regime` RNG stream declared, not consumed.** `RNG_STREAMS` includes `"spot_regime"` but no producer draws on it yet. Expected — spot pricing is F1 Slice C (not built). Forward-declaration, not drift. Track that the *first* spot draw uses this stream (and not `rates`) to preserve CRN.
- **Schema columns written-but-never-read-back / declared-but-never-written.** `tender_at` and `created_at` (shipments) are in the SQLite schema but never written by `_persist_hawbs` (they're firm-up / replay-loop fields — 2c). `soft_deadline_h` and `tardiness_weight` exist in the SQLite schema + are read by the MILP but are **absent from `data_model.md`** (the Postgres canonical declares `effective_deadline_at`, not `soft_deadline_h`). All known-deferred, but the data-model↔schema parity is incomplete: a Postgres swap would lose `soft_deadline_h`/`tardiness_weight` unless added. Logged, not blocking.
- **`docs/design/scenario_io_and_replay.md:107`** still names the config key as `kappa (capacity_scale)` — stale alias now that `capacity_scale` is purged from code. Doc-only; harmless but should be cleaned in the next doc pass.
- **`mawb_group_key`/`billing_json` (route_legs) + `flex_denominator`/`booking_promise` tables** are output-side schema with no producer yet (2c/scorer). Expected; no current consumer to drift.

---

## Tests run
`pytest tests/test_network_supply.py tests/test_generator_to_files.py tests/test_arrival_stream.py tests/test_scenario_db.py` → **45 passed**. Plus the bespoke zero-count persist→load→solve equality check above.
