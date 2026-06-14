# Project Context

**Last updated:** 2026-06-14 (Session 37 — **graph-gen workstream BUILT + a 4-agent critique that WALKED BACK the B5 "resolved" claim.** 255 passed, ruff clean. Built, in order: **`FreightNet`** (`src/freightnet.py` — physical freight-node reference DB, all 9 node types, 3,274 real airports from an OurAirports CSV asset; spatial service nodes_within_km/nearest/extend_until_reachable + `load_freightnet` build-if-missing); **`geo_select`** (`src/components/geo_select.py` — per-door candidate selection: k-NN seeds + detour-corridor ellipse φ + exhaustive bidirectional flight-frontier H_max; confirmed NOT a dominance prune); build-time **per-HAWB fallback** (`FallbackPolicy`/`_max_path_cost`/`air_leg_cost_ub` — longest UB-path × 1.5, trivial-when-stranded, arrives at T^abs = max tardiness); **FULL retirement of the hardcoded-region candidate path** (generation is door-only; nominal gateway via FreightNet; candidates + air-arc restriction resolve at build via `build_geo_air_graph`; persistence dropped candidate JSON + persisted `fallback_cost`); and the **D-F6 v2** methodology change (Δ_k = ready + base_transit(lane) + sla_offset(tier) — pre-committed tier×lane SLA, graph-free; `committed_deadline`; sla_offset 12/40/120→12/24/48; `model/precommitted_sla_deadline_proposal.md` APPROVED). **Long deep-dive sequence** with the user on each (fallback economics, $1-vs-dominating, the Δ_k circularity). **B5 re-measured then REOPENED:** the geo corridor cut LP size (103→~37 arcs) but a **4-agent critique (`docs/critique/17`)** found solve time is seed-dependent and unbounded — n=15/d7 = 23s/132s/>5min-unfinished over seeds 0/1/2; "21.2s RESOLVED" was a lucky seed. Critique also found **BLK-2: `air_leg_cost_ub` under-bounds BSA per_flight** (volume-bound cargo) → fallback can strand a routable HAWB. Both BLOCKING logged for user review tomorrow; nothing fixed in response yet. Memory to add S38: D-F6 v2 + B5-reopened. Prior:) 2026-06-14 (Session 36 — **F1 Slice A + B1–B4 BUILT; standing review agents run; B5 found region→region intractable at the proof cell.** 205 passed, ruff clean. Slice A = independent network-supply draw (κ+α integer multinomial on a new `supply` RNG stream, CRN-gated). Ran the 3 standing review agents (`docs/critique/14/15/16`): seam audit no-BLOCKING + validated Slice A; **red-team R1/R2 DEFLATED after user pushback** (loose-corner gate "no-op M₁ passes it" = strawman in a deterministic 2-solver sim; "average hides a lottery" = the real value distribution + correct estimator) → logged as minor scorer-build notes, §13 NOT reopened; drifts fixed (fallback 2.0→1.5; leg-count flagged). Slice B B1–B4 = **region→region routing (D-A24)**: additive `Hawb.origin/dest_candidates` (single-airport default = byte-identical), trucking "diamond" ground chains, **per-tail Δ^post N7 fix** (was a flat subgraph sum → double-counted multi-POD), dispatch-lead per candidate origin, `check_consolidation_coherence` invariant, distance-based (haversine) trucking-matrix generator + **SFO** 3rd dest gateway, `DEMAND_LANES` retired. Fixed latent hash-order determinism bugs region→region exposed (sets/frozensets feeding HiGHS sums + actuals presampler) + persistence round-trip (candidate JSON columns). **B5 = OPEN BLOCKER:** n=15/days=7 >5min; **720→168h backstop (now `config/forwarder_graph_config.json`, forwarder-level) did NOT fix it** — the real fix is geographic graph-gen / the network layer (see RESUME). User rejected cost-based dominance pruning (strands HAWBs under tight supply). Memories: `project_graph_generation_vision`, `feedback_no_standalone_cost_pruning`. Prior:) 2026-06-13 (Session 35 — **F1 REDESIGN → `arrival_only_replan_methodology.md` §13 APPROVED v4 (independent network-supply model + region→region routing).** No code touched; 193 passed (unchanged). User asked for a critique round before building the planned F1 ("continuous κ"); it turned into a ground-up redesign driven by user pushback. **Key reversal:** the original F1 (weight-only kg `cap_a = peak_demand/κ` on `BsaContract.cap`) was found (a) unable to ration volume-limited cargo — all cargo is 120–240 kg/cbm, below the LD3 333 break-even, so C.5b-volume binds first and the kg dial is a near-no-op; and (b) **circular — it constructed supply from the demand just generated.** Redesign (4 critique rounds, §13 v1→v4): **supply is generated INDEPENDENTLY of demand across the network** — per-flight **integer ULD positions** from a new `supply` RNG stream; **κ** = network tightness (`total_N=round(E[Σ SE_k]/κ)`, analytic no-consolidation slot mean — no demand coupling); **α** = Dirichlet concentration (lumpiness); per-lane tightness EMERGES from random supply vs where demand lands; the mismatch IS the problem. **D-A24 LOCKED: region→region routing** (origin/dest airport + lane + flight are optimizer decisions via a per-airport trucking matrix + multi-O/D subgraphs; fixed-lane `DEMAND_LANES` retired — scope expansion, committed). Integer ULDs **dissolve the volume defect** — the existing 2D **C.5/C.5b** is the contracted gate as-is (D-A20/21 WITHDRAWN; cap_a/SE/peak_demand machinery gone). Three supply sources/flight: contracted (C.5b) / spot (NEW per-arc `Σ cw_k·x ≤ cap^spot`, two-sided price) / fallback (1.5× graph-derived worst-spot-route, amends D-A12). **Falsifiability:** dedicated negative-control retired → loose corner of the (κ,α,λ) sweep gated `|L2|<CI`; regret floor demoted to a self-check; M₀ (D-A23) = competent single-pass baseline (deterministic within-cycle consolidation). Memory added: `project_supply_independent_of_demand`. **▶ RESUME = start F1 build against §13 v4.** Prior:) 2026-06-11 (Session 34 — **critique-12 clarity rewrite + 7-agent integration/framework review (`docs/critique/13` + `docs/design/e2e_test_plan.md`) + Wave-0/1/2-partial code fixes.** 193 passed, ruff clean throughout. Rewrote `docs/critique/12` § Numeric walkthroughs (F1/F2/F3) clearer (shared κ×λ picture + ASCII timeline). Ran a **7-agent review** (code-arch / seams / graph-gen / model-realism / sim-realism / numerics / test-design) → new findings beyond F1–F8: **N1** $1M fallback wrecks the relative MIP gap (L2 is a *difference* of two such objectives); **N2** PYTHONHASHSEED documented-but-unset; **N3** no state-owner for sim-clock/ledger/RNG before 2c; **N4/N5/N6** the arrival-only headline excludes the thesis's primary driver (disruption) + one `[CAL]` knob sets both L2 mechanism and `cw_flex` denominator + π_hind floor near-vacuous; **N7–N18** latent correctness (Δ^post subgraph-sum, A_k^min over unfiltered graph, air-arc board-by CO* not STD, dispatch over-applied, carrier deny-layer missing, ULD 8.0-vs-4.5 mismatch). Fixed **Wave 0** (N1 `compute_fallback_cost`≈$40k / N2 cross-process determinism proof-test / F8 CRN / N17 asserts→raises / n1 aux bound / n2 deadline guard / N7 single-tail guard / N18 dedupe) and **Wave 1** (N9 board-by min(CO*,STD) / N10 dispatch→origin-POL / N8 A_k^min over admissible arcs / N12 ULD vol 8→4.5; N11 deferred). **Wave 2 STARTED** (the critique-12 fold; all `[CAL]` locked: L_cut=6h, κ grid {0.5,0.8,1,1.25,1.5,2}, roll $50, F4 ~60/40, **τ=1.5%** from researched forwarder economics): **F2a cutoffs** (`dep−6h`, `_SCHED_OFFSET_H=10`, HKG bank +8) + **F2b binding-leg anchor** (`_contracted_by_dest_day`, `air_graph.latest_ready` clamp = no born-dead) DONE. **▶ RESUME = build F1 (continuous κ = demand/slots).** Prior:) 2026-06-09 (Session 32 — **generator-to-files BUILT** (`write_scenario` persists a full scenario + pre-samples ALL leg/component actuals, deterministic `s=0`; G1–G4 folded; 147 passed, ruff clean). **Arrival-only methodology APPROVED + reconciled** (`model/arrival_only_replan_methodology.md` v0.1 — air transit DETERMINISTIC; per-leg Gaussian jitter AND a discrete disruption-stream BOTH rejected as "fake" manufactured failure; the ONLY stochastic process is demand arrival; predicate-9/z_tier/σ̂ retired for air → deterministic `A ≤ Δ_k`; M₀ incremental-greedy vs M₁ open-book re-opt; disruption recourse = a TESTED capability not a value source; L2 = conservative lower bound) → `air_transit_time.md` v0.3 + `backtest_methodology.md` v0.5; code aligned (deterministic pre-sampling). **Arrival-process shape LOCKED** (§10, D-A5..D-A8: daily departures `D≈7`; `known_at = cutoff(d*) − B`; tier-coupled book-lead [EXPRESS late+tight, DEFERRED early+slack]; fixed `N`; sweep κ×λ). **4-agent simulation design review** (`docs/critique/11`) → hardening folded as §12 / **D-A9..D-A16**: D-A9 independent-arrival HEADLINE (coupled-favorable = upper bracket; removes "built the sim to win"); D-A10 pre-registered null + required negative-control cell; D-A11 M₀=pin-prior-soft + the `M₁'` control arm; D-A12 realized_cost excludes the C.10 penalty + reshuffle-gated headline + retire $1M fallback; D-A13 one time-scalar source of truth; D-A14 batch-cutoff H₀ headline; D-A15 lower-bound scoped to transit-reliability only; D-A16 `cap_a`/`A_c` frozen. **▶ NEXT: λ arrival-stream generator + 2-FLEX** (D-A9 independent-arrival headline; 2c blocked on M₀ soft-pin + `M₁'`). Prior:) 2026-06-08 (Session 31 — **input-layer fork = Option A**; input layer + 5 hardening edits built on `scenario_db.py`; **5-agent proactive review** folded (`docs/critique/10`); **round-trip spike BUILT & GREEN** (`data/synthetic/scenario_io.py`) → schema validated lossless (caught the flight_id-not-unique bug); `data_model.md` reconciled; ERD written; one-offer-one-rate (memory `reference_air_offer_rate_cardinality`); decisions D1 capacity ULD-slot-only / D2 `mct_h` / D3 `{offer_id}:{uld_type}` / D4 delete `contracted_rate_per_kg`. 139 passed, ruff clean. **▶ NEXT: generator-to-files (G1–G4) → 2-FLEX → 2c → arms → Stage 3.** Prior: Session 29 — **Stage 2a synthetic generator + 2b transit-time BUILT & isolation-tested** (110 passed, ruff clean); **OTP-control reframe** locked (OTP = population-over-time binary metric, controlled at graph-gen tier-reliability filter [new air-model predicate 9] + per-shipment penalty, NOT chance constraints; one-draw-per-leg realization, no per-route MC); **specs APPROVED:** `air_transit_time.md` v0.2 (2b), `backtest_methodology.md` v0.4, `flexibility_model.md` v0.3 (2-FLEX), `scenario_io_and_replay.md` v0.2 (SQLite-per-scenario, generate-all-first→deterministic replay, route-versioning plan history); **route-ordering code fix** (`air_graph.order_route`); **6 multi-agent review rounds** folded (OTP reframe ×2, 2-FLEX ×2, proof-wide ×3 [calibration/seam/red-team], scenario-IO ×1). Memories added: `project_otp_control_reframe`, `reference_air_spot_contract_ratio`, `feedback_session_log_last_entry`. **▶ NEXT BUILD: `scenario_db` module (schema-first), then generator-to-files → 2-FLEX → replay loop.** Session 28 follows:) 2026-06-06 (Session 28 — **backtest methodology APPROVED (v0.3)** + reconciled `docs/design/remaining-execution-plan.md` to v0.3 + ran critique agents on both; reconciled `product_thesis.md §2`; **created `BUILD_STATUS.md`** as the canonical built/remaining tracker — now read first at session start + refreshed fully every sign-off, per CLAUDE.md; demand-arrival micro-spike `spikes/stage0_5b_demand_arrival_replan.py` PASSED. Air optimizer M1–M6 complete. **▶ NEXT BUILD: Stage 2a synthetic data generator.** Session 27 context follows:) 2026-06-04 (Session 27 — strategy + planning thread, **no air-build code changed**: created `product_thesis.md` (four-layer value gradient L1 planner→L2 replan→L3 capacity controller→L4 market intel; moat = data flywheel; replan-savings estimation method) + `docs/design/remaining-execution-plan.md` (air-deep-to-thesis-proof bet, revised v0.2 after 3-agent critique). **DECISIONS LOCKED: D1 = Opt 1 (BSA contract entity, builds per_flight + equalized in M4, M4b folded in); D2 = Opt A (base `schedule_legs` + per-mode detail tables; air 2D capacity first-class); route-versioning model (full immutable snapshots, resolution×firmness orthogonal, `supply_ref` ∈ {schedule_leg, ondemand_arc, rate_card_lane, NULL}).** M4 now UNBLOCKED; next is data_model.md spec (option b — sketch then write) → Stage 0.5 fail-fast spike → M4 build. Session 26 context follows:) 2026-06-03 (Session 26 — side thread, **no air-build code changed**: LOC accounting + phased burndown (current ~3,233 real LOC; full commercial product est. ~112K); created `synthetic_data_contract.md` (NEW, repo root) — supply/demand generator output contract vs `data_model.md`. Decisions locked: add `schedule_legs`; pre-generate + sim-clock-view reveal; `external_id` idempotency; orchestrator-driven capacity decrement; defined `demand_generator_configs`. **OPEN:** schedule-schema *structure* (base+per-mode-detail [rec] / separate-per-mode / single-wide) — blocks promoting `schedule_legs` + `demand_generator_configs` into `data_model.md`. Air MILP M4 schema decision remains the PRIMARY next action. See "Parallel thread (Session 26)" below. — Session 25 context:) 2026-06-02 (Session 25 — Air MILP slices **M2** (C.4 chargeable-weight density mixing + flat_rate bucket cost + C.5c cap) and **M3** (min_flat_breaks IATA next-break-down round-up) landed; consolidation now pays; all three non-BSA rate families billed; billing validation reworked to recompute CW from realized routing + assert each family's closed form. Full suite 84 passed, ruff clean. Ended on the **M4 schema decision** — three options (contract entity / flat+tag / per_flight-only) with numeric walkthroughs + a pivot-vs-allowance clarification captured in `docs/design/air_milp_m4_bsa_schema_options.md`; **DECISION OPEN, user deep-dives next session before any M4 code.** Prior context below from Session 23 —)

**(Session 23 context:)** 2026-05-31 (Session 23 — **AIR MODEL APPROVED** + **Phase 2 air component build STARTED**. Air model is the first gated formal model. Then began `src/components/air_graph.py`: produced `docs/design/air_graph_arc_construction_plan.md`, ran a 4-agent critique (spec-fidelity / test-coverage / data-realism / architecture), folded 16 findings into a v2 plan, and landed 3 code slices — (1) air-arc emission with overlapping-emission policy + `flight_arcs` reverse map; (2) origin/dest ground chains (P1–P3, P7–P10) with `:in`/`:out`-split CFS nodes; (3) hub-transit dwell (P5 deconsol / P6 connection) with airport `:in`/`:out` split + estimated `delta_h` data input. Full suite 28 passed, ruff clean. Locked: flat `dict[ArcId,Arc]` store (no networkx), deterministic arc IDs, strict 1→8 predicate order, dwell time = estimated input. `*.key` Keynote files gitignored. Earlier in session — air-model PDF re-review pass. Three substantive changes to `air_freight_routing.tex`: (1) **PWL tardiness grid decision CLOSED** — §16.3 switched from fixed absolute `{0,4,12,36,96}`h to per-HAWB relative even grid `τ̂_{k,j}=α_j(T_k^abs−Δ_k)`, `α={0,.25,.5,.75,1.0}` (J25); (2) orphan `C^pref` bug fixed in §carrier-policy; (3) **Path-B per-ULD surcharge restructured** — `σ^uld` now per-arc-endpoint (build@tail + breakdown@head, once) on `η`, not a per-leg sum, removing through-arc double-count (J26). Full 4,317-line read-through done; per-HAWB cost-attribution aggregate-exactness verified. A separate Claude session's uncommitted work — C.13 BSA-settlement review, new `model/capacity_manager.md` stub, `references/` allotment-contract notes — was committed alongside (user reviewed it independently). **PDF NOT recompiled** — tree PDF is one compile behind the `.tex` and was excluded from the commit.)

## RESUME HERE (next session — Session 38)

**▶ FIRST ACTION: review `docs/critique/17-graph-gen-session-review-s37.md`** (the user said they'd review it
tomorrow) and decide the triage. It records the full 4-agent critique of the S37 graph-gen workstream. Two BLOCKING:

1. **BLK-1 — B5 is NOT robustly resolved (REOPENED).** The geo corridor cut LP size (103→~37 arcs/subgraph, real) but
   solve time is **seed-dependent and unbounded**: n=15/d7 = 23s (seed0) / 132s (seed1) / **>5min unfinished (seed2)**,
   HiGHS threads=1, no time limit. The "21.2s RESOLVED" was a lucky seed; the corridor doesn't touch the
   consolidation/capacity branching hardness. **Pick a tractability strategy:** HiGHS time-limit + incumbent /
   MIP-gap/warm-start/heuristic tuning / accept n<15 / tighter φ. This gates the replay sweep.
2. **BLK-2 — `air_milp.air_leg_cost_ub` under-bounds BSA per_flight** (`n_uld_solo` weight-only; real cargo is
   volume-bound). Fallback can be under-priced → can strand a routable HAWB. Fix: volume-aware `n_uld_solo` + a
   fallback-dominance invariant test.

Then the cheap safety + doc fixes (critique-17 MAT-4/MAT-5/MIN-2/MIN-3/MIN-4 + the proof-neutrality "second-order"
retraction + a born-at-risk-fraction diagnostic), and a **φ sensitivity sweep** (MIN-1) alongside the B5 work.

**▶ THEN (still pending, post-triage): F1 Slice C** — spot per-arc CW-cap `Σ cw_k·x ≤ cap^spot` + two-sided spot
price + route-based fallback, per §13/D-A19. Then Wave-3 N3 `ReplayState` owner → 2c replay loop + arms
(M₀/M₁/M₁'/π_hind) → scorer + (κ,α,λ) sweep = the proof.

**S37 build state (255 passed, ruff clean — committed at S37 sign-off).** Graph-gen workstream BUILT: `FreightNet`,
`geo_select`, build-time `FallbackPolicy`, candidate-path retirement (door-only generation; `build_geo_air_graph`),
D-F6 v2 SLA deadlines (`committed_deadline`). The geo path is the build chain every consumer uses; cross-process
determinism proven. Governing methodology `model/arrival_only_replan_methodology.md` §13 v4 + `flexibility_model.md`
D-F6 v2 + `model/precommitted_sla_deadline_proposal.md` (APPROVED). **Caveat: 255 green does NOT clear the 2 BLOCKING
above** (a solve-time tail + a cost-bound gap the suite doesn't exercise).

**(Prior — Session 33/32 end-state below, historical.)**

**▶ Read `BUILD_STATUS.md` first (canonical tracker), then only the last SESSION_LOG entry (Session 32).** The **governing proof methodology is now `model/arrival_only_replan_methodology.md`** (v0.1, approved S32) — read §10 (arrival process) + §12 (critique-11 hardening, D-A9..D-A16) before building. `docs/critique/11-simulation-design-review.md` holds the full 4-agent review. The two prior specs (`air_transit_time.md` v0.3, `backtest_methodology.md` v0.5) are reconciled to it.

**Session 32 end-state (147 passed, ruff clean).** generator-to-files BUILT (`write_scenario` + `presample_actuals` deterministic `s=0`; G1–G4). Methodology pivoted to **arrival-only / deterministic transit** (the ONLY stochastic process is demand arrival; per-leg jitter and disruption-stream both rejected as fake; predicate-9/z_tier/σ̂ retired for air). Arrival shape + critique-11 hardening locked (D-A5..D-A16).

**▶ NEXT BUILD — λ arrival-stream generator + 2-FLEX** (co-central; arrivals are the only uncertainty): extend the schedule to a daily `D≈7` horizon; `TierSpec` (flex-model v0.3) + `classify` + frozen `cw_flex`; populate `shipments` tier/`Δ_k`/`known_at` (`= cutoff(d*) − B`). **Critical critique-11 constraints:** the HEADLINE draws lateness **tier-INDEPENDENT** (D-A9; coupled-favorable = upper bracket only); κ dialed in binding-ness not ULD integers (D-A10); ≥2 cheap options/lane (M-B5); cutoffs snapped to the sim-step grid (M-B4); tender = cutoff-only. **2c is BLOCKED until the M₀ pin-prior-soft constraint + the `M₁'` control arm are pinned down (D-A11).** Then 2c (M₀ incremental-greedy vs M₁ open-book re-opt; one time-scalar source of truth D-A13; global conservation + 2-arc fixture + per-slip bump cost; 3 disruption-recourse fixtures) → negative-control cell + null (D-A10) → scorer (realized_cost excludes the C.10 penalty; 3-way L2 split; `L2_reshuffle` gated headline) → Stage 3 proof.

**Earlier-session detail retained below (Session 29 OTP reframe + approved specs; Session 27/28 etc.).**

**Built & green:** Stage **2a** synthetic generator (`data/synthetic/air_generator.py` — in-memory `AirInstance`; file-write lands in generator-to-files) + Stage **2b** transit-time (`src/components/air_transit_time.py` — `sample_route` single-draw realization + `route_reliability` deterministic scalar) + the `air_graph.order_route` route-ordering fix (routes time-ordered for the running-clock consumers).

**The OTP-control reframe (load-bearing, memory `project_otp_control_reframe`):** OTP is a **population-over-time binary metric** (`A ≤ Δ_k`), **controlled at graph generation** (new air-model **predicate 9**: tier-reliability admission `Â(r)+z_tier·σ̂(r) ≤ Δ_k`, deterministic, no MC) **+ a per-shipment penalty `W_k` control input** (default tier ratio; prioritization under contention; frozen during the proof) — **NOT chance constraints** (deferred, not in MVP). Transit = **one draw per leg = the actual** (no per-route MC); recourse (roll/replan) is the orchestrator's. Two-sided κ-tied spot:contract gap (memory `reference_air_spot_contract_ratio`).

**Approved specs:** `air_transit_time.md` v0.2, `backtest_methodology.md` v0.4 (single-draw OTP, conservation-identity, schedule+demand tripwires, π_hind = physical-feasible-only + realized actuals, C^fallback=2×worst-feasible-route, L2_reshuffle vs L2_fallback split), `flexibility_model.md` v0.3 (2-FLEX: TierSpec single-source, derived flexibility, frozen cw_flex), `scenario_io_and_replay.md` v0.2 (SQLite-per-scenario, generate-all-first→deterministic replay, route-versioning plan history, §2.1 determinism pins incl. solver `threads=1`+seed). `air_freight_routing.tex` edited (predicate 9 + W_k control-input + deferred-item annotations) — **PDF NOT recompiled** (user renders).

**▶ NEXT BUILD — schema-first (`scenario_io_and_replay.md §9`):** (1) **`scenario_db` module** (SQLite DDL + `sim_state`/`visible_shipments` reveal view + FK pragma + deterministic ids + integer capacity — the SQLite↔Postgres seam); (2) generator-to-files (extend 2a + pre-sample ALL flight-leg actuals via 2b); (3) 2-FLEX populates demand (TierSpec shared object — resolves the seam-audit's no-code-home drift); (4) replay loop (2c). **Pre-build chore:** promote `known_at`/`tender_at`/`tier`/`Δ_k`/`T_dead` cols into `data_model.md §3`. Then Stage 3 proof. Prior (Session 28/27) state retained below.

**STATE: Air model APPROVED. Air graph CONSTRUCTION-COMPLETE + integration-validated. Air MILP — M1 walking skeleton + M2 (density mixing + flat_rate billing) + M3 (min_flat_breaks IATA round-up) landed; consolidation pays, all three non-BSA rate families billed. Full suite 84 passed, ruff clean across src/tests/data. Session 27: D1 (M4 BSA schema) = Opt 1 contract entity; D2 (schedule_legs) = Opt A; route-versioning model locked. data_model.md spec DONE (schedule_legs + per-mode detail tables + route-versioning routes/route_legs/planning_runs + bsa_contracts + demand_generator_configs written into §1.3/§3.3/§3.6). **Stage 0.5 fail-fast spike DONE & PASSED** (`spikes/stage0_5_drift_replan.py`, throwaway): on the 12-HAWB TPEB instance, replan-under-drift (cancel a used flight → reroute) beats no-replan-static (affected HAWBs eat the $1M fallback) on ALL 8 candidate cancellations; dominance invariant (static ≥ replan, by subset-feasibility) + fallback-avoidance held → savings accounting is sound. Caveat: $ magnitude is fallback-avoidance-dominated because M1-M3 air arcs carry no freight cost yet (replan ≈ baseline 10,095); freight-rate-driven savings sharpen at M4. Full S2 (hindsight-regret reconciliation + no-capacity-double-spend) deferred to Stage 3/M4 — nothing to double-spend pre-M4. Mechanic + accounting validated. **M4 DONE & GREEN** (`air_milp.py`): `per_uld_pivot` + BSA contract entity (`BsaContract`/`UldType`, Opt 1); C.5/C.5-act/C.5b-w&v/C.5c-uld + C.13a equalized & C.13b per_flight pivot; integer η, pivot, over_c vars; +4 value-checked tests (per_flight $6250 pivot-floor, equalized $2000 cross-lane overage, C.5 allotment→fallback, C.5b-v volume-drives-η). Full suite 88 passed, ruff clean. **M5 DONE & GREEN** — C.10 destination arrival + quadratic-tardiness PWL: `arr_dest(k,a)` derived in-MILP (terminal air arc ETA + Δ^post dest-chain transit; fallback=deadline; no graph change); `t_k`/`τ_k`/`pen_k` vars; C.10a/C.10b; per-HAWB relative even-5 α-grid tangent cuts (J25); `Hawb.soft_deadline_h` + `tardiness_weight` added (default inert); `AirSolution.hawb_tardiness_h`. +3 value-checked tests (on-time→no pen=150; late τ=4→PWL pen=15→165; W-doubled→pen=30→180). Full suite 91 passed, ruff clean. **M6 DONE & GREEN** — surcharges: Path-A per-HAWB-per-arc (`Surcharge`: per_kg·cw + per_shipment + per_cbm·v, on x) + Path-B per-ULD `σ^uld_{a,u}` on η (J26, per-arc-endpoint, not z); full objective assembled. +2 tests (Path-A 3-bases→+85=735; Path-B σ^uld→+20=6270). Full suite 93 passed, ruff clean. **🎯 AIR OPTIMIZER M1–M6 COMPLETE — Stage 1 DONE (component 2.9 isolation-complete; 25 value-checked air_milp tests; full approved formulation C.1/C.2/C.4/C.5/C.5b/C.5c/C.10/C.13/C.14 + surcharges + MAWB-fix).** **▶ NEXT ACTION: Stage 2** (per remaining-execution-plan.md): write backtest-methodology spec FIRST (G-Method) → 2a synthetic generator (D2=Opt A) → 2b parametric air TT (defer OpenSky) → 2-FLEX flexibility model → 2c rolling-horizon orchestrator → Stage 3 replan-savings proof. Deferred (off critical path): M-cleanup (per-MAWB-break hub-dwell attribution; construction micro-cases); input-seam distribution-ready test. **Full map = `docs/design/remaining-execution-plan.md` (v0.2); spike = `spikes/`.**

**Slice 4 DONE (2026-06-01, Session 24): per-HAWB subgraph pre-filter.** Two-pass: Pass A `_propagate_forward` (earliest feasible arrival, cutoff+dispatch gating, air-arc head reset to ETA) + `_latest_to_dest` (backward DP, predicate-7 dest reachability); Pass B `_first_failing_predicate` strict 1→8 + `RejectRecord`. `fallback_arc`, `PrefilterWarning` (empty → routes via fallback, no exception), `SubgraphResult`, `build_hawb_subgraph`, `build_subgraphs`. Predicates 2–5 via **minimal inline flags** on `Leg`/`Hawb` (decision: real embargo/lithium/carrier engines deferred, these flags are their seam). structlog moved to runtime deps, sinks JSON to stderr. +16 tests.

**Slice 5 DONE (2026-06-01): Phase-2 MAWB overlay + full `AirGraph` contract.** `group_key(hawb)` per approved tex sec:g-of-k ({GEN,DGR,PER}→`(class,temperature)`; {VAL,HUM,AVI}→`(class,hawb_id)` singleton; screening dropped). `temperature` on `Hawb`, `Mawb`, `build_riders`, `overlay_mawbs`, `AirGraph` + `build_air_graph`. +14 tests.

**Slice 6 DONE (2026-06-01): hub-dwell auto-emission (P5/P6).** `airport_code`, `Hub` config, `candidate_hub_codes` (both-ends air arcs; R7 through-arcs carry hub internally → not candidates), `build_hub_dwells` (per-HAWB, P5 if CFS-H else P6), `hubs=` keyword threaded through `build_physical_graph`/`build_air_graph` (emit-then-prune). +7 tests incl. C7 deconsol-divergence.

**Slice 7 DONE (2026-06-01): TPEB realistic integration instance (plan §7).** `data/synthetic/tpeb_air_instance.py` → `build_tpeb_instance()`; real topology/carriers (TPE/PVG/HKG→LAX/ORD, ANC tech stop, CI/BR/CX/CV/MU, HKG=CFS-H), synthetic commerce; 13 offers (all rate families + overlapping emission + multi-stop + interline through), 12 HAWBs (§4.2 8-group set). `tests/components/test_air_graph_tpeb.py` +6 integration tests (all green: 0 fallback-only; 8 HAWBs / 2 origins through HKG; per-flight coupling; VAL singleton; tech-stop no-ANC-dwell).

**Air MILP build (`src/components/air_milp.py`) — sliced. Decisions locked (Session 24): stub REPLACED (old Shipment/Flight gone); rates in a separate `RateCatalog` keyed by arc id, passed to `solve()`; walking-skeleton-first.**

**M1 DONE (2026-06-01):** `solve(air_graph, hawbs, rates, params) -> AirSolution`. Vars `x[k,a]`, `z[a,g]`; **C.1** flow, **C.2** MAWB linkage, **C.14** domain. Routing-only objective (ground/dwell/fallback `ground.cost·x` + co-load `m·cw·x` + MAWB fixed-charge `c_fix·z`). MAWB-eligible air arcs have **no variable freight cost yet**. 7 value-checked tests incl. TPEB solve (OPTIMAL, 0 fallback, 11 MAWBs, ~10,095).

**M2 DONE (2026-06-02, Session 25):** C.4 chargeable-weight density mixing + flat_rate bucket cost + C.5c cap. Continuous `CW[a,g]` per MAWB candidate; C.4a+c `CW ≥ (1+ε)Σ w_k·x`, C.4b+d `CW ≥ Σ v_k·167·x` (Wt/Wv inlined into bounds), upper-link `CW ≤ CW^ub·z` (Eq. cw-ub). flat_rate bucket via aux `c[a,g] ≥ max(min_chg·z, m·CW)` (tex sec:lin-bucket). C.5c `Σ w_k·x ≤ cap_a` for capped flat offers. `RateCatalog.flat: dict[ArcId, FlatRate(m, min_chg, cap)]`, `MilpParams.dunnage_eps=0.05`, `AirSolution.mawb_chargeable_weight`. Post-solve monotonicity assert (`_assert_cw_invariant`). +6 tests incl. density-mixing CW=283.9<400.4 and consolidation-beats-separation (1669.5<2302.0). **Consolidation now pays.** Full suite 79 passed, ruff clean.

**M3 DONE (2026-06-02, Session 25):** min_flat_breaks IATA next-break-down (`_build_min_flat_breaks_cost`). Break selector `γ_{a,g,b}` + bucket weight `BW_{a,g,b}`; `Σ_b γ = z`; 3-inequality BW disaggregation (`BW ≤ M^BW·γ`, `BW ≥ break_b·γ`, `BW ≥ CW − M^BW(1−γ)`, **no** `BW ≤ CW`); `M^BW = max(CW^ub, max_b break_b)` (BUG-1 widening); objective `Σ_b rate_b·BW_b`. `RateCatalog.min_flat_breaks: dict[ArcId, list[Break(threshold, rate)]]`. **Billing validation reworked** — `_assert_cw_invariant` → `_validate_billing`+`_routed_cw`: CW recomputed from realized routing (solver-independent), each family's realized cost asserted == closed form (flat `max(min_chg, m·CW)`, MFB `min_b rate_b·max(CW, break_b)`); `mawb_chargeable_weight` reports routing-derived CW. +5 tests (round-up $800-not-$900, round-down, weight-dominated, no-catalog, density-mixing-into-breaks). Full suite 84 passed, ruff clean.

**GATE before M4 build — RESOLVED Session 27: M4 schema = Opt 1 (contract entity, `BsaContract` + `UldType`; builds per_flight AND equalized; M4b folded in).** (Historical:) the M4 schema was UNDECIDED. M4 introduces a BSA contract entity + ULD-type catalog (not just per-arc rate fields). Three options + full numeric walkthroughs + a pivot-vs-allowance clarification are captured in **`docs/design/air_milp_m4_bsa_schema_options.md`**. **Next session, FIRST: user reads that doc and picks a schema option (Opt 1 contract entity / Opt 2 flat+tag / Opt 3 per_flight-only-then-M4b). Do NOT write M4 code until the schema is chosen.** Drafter's lean: Opt 3 now → Opt 1 for M4b.

**Then slice M4 — `per_uld_pivot` + BSA allotment.** Integer `η_{a,g,u}`, `pivot_{a,g}`, `over_c`. **C.5** `Σ_g η ≤ N_{a,u}`, **C.5-act** `η ≤ N·z`, **C.5b** per-ULD `W_u`/`V_u` (uses `w_k` actual not `cw_k` — item 13-A), **C.5c-uld** per-offer cap. **C.13** BSA settlement: `per_flight` → `cost^MAWB = r_a·pivot`, `pivot ≥ CW`, `pivot ≥ π_a·Σ_u η` (C.13b-1/b-2, tex sec:lin-pivot); `equalized` → `cost^MAWB = 0`, all cost via `r_c·over_c` with `over_c ≥ chargeable(c) − A_c`, `chargeable(c) = Σ CW` over contract arcs (C.13a). Catalog: ULD/BSA tables (`N_{a,u}`, `W_u`, `V_u`, `π_a`, `r_a`, settlement basis, `A_c`, `r_c`, contract→arc map). Pre-filter step 8 (HAWB-too-big-for-any-ULD) already in graph. Then M5 C.10 quadratic-tardiness PWL (pen_k, arr_dest, α-grid), M6 surcharges + full objective.

**Deferred, don't lose:** per-MAWB-break cost attribution for hub dwell (resolve in an objective slice); plan-§6 construction micro-cases (A4a/A4b, B-series, 3-stop-six-candidate); `model/capacity_manager.md` stub review (not gated).

**Deferred, don't lose:**
- Per-MAWB-break **cost attribution** for hub dwell (don't charge once-per-HAWB) — flagged in `build_hub_dwell` docstring; resolve in the objective/overlay slice.
- `model/capacity_manager.md` stub — **TO BE REVIEWED, not approved** (Layer-3 BSA/NAC controller supplying `A_c`, `cap_a`, `δ_c`). Not gated.
- `docs/critique/00-omitted-findings-index.md` — ~40 deferred air-model findings for a future round.

### Parallel thread (Session 26) — synthetic data contract; schedule-schema DECISION OPEN

`synthetic_data_contract.md` (NEW, repo root) specs the supply/demand generator's output contract against `data_model.md`: generator writes to **production tables tagged by provenance** (not a bespoke sim schema), so synthetic→real is a source swap. Locked decisions: (1) add `schedule_legs` physical substrate; (2) reveal via pre-generate + `visible_shipments` sim-clock view; (3) `external_id` idempotency (`gen-{seed}-{seq}`) for upsert path-parity + Postgres NULL-distinct safety; (4) orchestrator-driven `remaining_*` capacity decrement. Defined `demand_generator_configs` (referenced §3.6, never defined). Supply classes: **fixed** = `carrier_allocations`+`air_uld_allocations`; **floating/market** = `spot_rate_quotes`; both slice `schedule_legs.capacity_total`.

**RESOLVED Session 27 — schedule-schema structure = Opt A** (base `schedule_legs` + per-mode detail tables; `trucking_ondemand_arcs` = leadtime arc outside the base; LCL = sailing overlay; air capacity 2D first-class). User correction: trucking has BOTH scheduled linehauls AND on-demand-with-leadtime; LCL rides a sailing. Original framing: The 4 modes are NOT symmetric: FCL+Air = scheduled/capacitated legs; trucking = on-demand rate arc (no schedule); LCL = overlay on an ocean sailing. Options: **(rec) base `schedule_legs` + per-mode detail (`ocean_sailing_legs`, `air_flight_legs`; trucking separate; LCL overlay)** / separate-table-per-mode / single-wide-nullable. User signed off mid-question; structure NOT chosen → tables NOT yet promoted into `data_model.md`.

**Next on resume (this thread, independent of air MILP):** (1) user picks structure; (2) promote `schedule_legs` + `demand_generator_configs` into `data_model.md` (§1.3, §3.6); (3) optionally spec `supply_generator_config`. **Open Q:** `schedule_legs` 2D air capacity (weight + volume/ULD) — single `capacity_total` vs first-class both.

### Air-model pending edits (carried from `usr_session_notes.md`, triaged Session 23)

- **§4.3 enumeration table** — explicitly enumerate all possible consolidation groupings as a table in the air LaTeX (from 2026-05-24). Small edit, not yet done.
- **Per-shipment slack metric** — design a per-shipment SLA-buffer metric (P-quantile arrival vs service-tier deadline) with a portfolio roll-up of most-fragile shipments; replaces "confidence" on the SLA dimension; counterfactual-robustness ideas feed in. Design needed (from 2026-05-24).
- ~~Max tardiness + PWL grid~~ — **CLOSED Session 23 (J25):** per-HAWB relative even grid `α={0,.25,.5,.75,1.0}`.

### Already executed in Session 22 — for context, not action

**Pass A** (+107 lines): BUG-1 `min_flat_breaks` big-M widened to `max(CW^ub, max_b break_b)` (fixes silent ban on IATA round-up case); BUG-2 `cost^MAWB = 0` for equalized BSA; BUG-3 forward-time-window per-outbound admission; TIGHTEN-1 `η ≤ N·z` linkage as C.5-act; C^fallback per-tenant sizing formula replacing $1M default; 3 new walking-skeleton metrics (fractional-x at LP root, per-arc activated-z distribution, fractional fallback usage); P-quantile naive-propagation warning + three-path probabilistic-transit fork in `item:tt-quantile-binding`; ~12 nomenclature additions; `C^pu → A^pu` rename (17 sites); `g` (MIP gap) → `g^mip`.

**Pass B** (+276 lines): per-HAWB cost attribution promoted to MVP with proportional-to-CW rule (multi-line `align` equation `eq:per-hawb-attribution`); per-HAWB tardiness `τ_k^hr` first-class diagnostic; new parameter `λ^disp_k` for truck dispatch backplane with dispatch-feasibility check in §sec:fwd-time-propagation; fallback root-cause attribution (`reject_record(k, a)`, `predicate^dom_k`) in §sec:prefilter + §sec:output-diagnostics; `item:cwlinearizer-interface` deferred with full design sketch + catalog-time validation; NEW §sec:arc-enumeration with overlapping emission policy + TPE→HKG→LAX worked example + Combos A/B + pivot-scope clarification; §sec:through-uld reframed as co-existing constructs not mutually-exclusive alternatives.

**Pass C** (+59 lines): TIGHTEN-2 destination-arrival LB tightened; F2.3 `c^MAWB_fix` scope = "new AWB issued"; F7.2 incumbent-bound spread (later mooted by Pass D); 4 walking-skeleton additions (F4.4, F6.3, F9.3, F12.3); notation cleanup (lifted `chargeable(c)`, removed `K^fb` from sets table, dropped `G`, annotated C.12 placeholder, fixed `eq:flight-uld-surcharge` double-bind, added `T_k^abs` ingestion guard, rewrote pruning-safety prose for per-tenant `C^fallback`); 4 new deferred items (temp poset F3.2, MIQP tardiness F5.3, forecast-aware accumulator F8.3, slot-symmetry breaking F13.2).

**Pass D** (−20 lines): soft-preference layer DROPPED ENTIRELY (user: "this is complete bullshit"). 5-layer cascade → 4-layer. No `C_k^pref`, `ε^pref`, `z*`, `g^mip`, `prefer_ℓ(k)`. No `eq:carrier-pref` or `eq:pass2-obj`. No Pass-1/Pass-2 lexicographic mechanism. No MIP-gap interaction. No "Why lexicographic" justification. §sec:carrier-policy intro rewritten with explicit "why no preference" — volume kickers → rate-card; strategic relationships → BSA negotiations; allocation balancing → equalized accumulator. Single-pass solve paragraph added. Worked example rewritten with hard allow/deny only.

**Pass E** (cosmetic): `\usepackage{float}` added. All 45 tabular envs wrapped in `\begin{table}[H]\centering\caption{…}\label{tab:…}` floats. All bleeding column specs fixed (9 `lll` nomenclature tables → bounded `lp{9cm}p{3.5cm}`; lifecycle states `lp{7.5cm}l` → `lp{6.8cm}p{4.5cm}`; carrier-policy worked example `lp{4cm}l` → `p{3.2cm}p{4.5cm}p{6.5cm}`; all `llp{X}l` parameter tables given bounded Parameter/Symbol/Unit columns). Per-HAWB attribution equation (587pt overfull) converted from one ultra-wide `\underbrace`-chain to multi-line `align` block with per-term `\tag{…}`. PDF compiles cleanly: 69 pages.

### New decisions logged (J20–J24, this session)

- **J20 — Probabilistic transit migration path:** OPEN until P1 promotion. Three structurally distinct paths now named in the deferred item; whoever implements first can no longer silently default to (a).
- **J21 — CWLinearizer:** design sketch + catalog-time validation in deferred item; implementation lives in data layer.
- **J22 — BSA accumulator concurrency:** still J3 open-item; revisit when orchestrator is the active workstream.
- **J23 — Carrier policy simplification:** soft-preference dropped; hard 4-layer cascade only; no Pass 2.
- **J24 — Real-world Top-5 triage:** 4 of 5 picked for Pass B (per-HAWB cost attribution, per-HAWB tardiness, truck dispatch, fallback root-cause); ~30 others deferred to next round.

### Critique deliverables produced this session

- `docs/critique/06-correctness-notation.md` — 22+ findings (3 BUGs, 2 TIGHTEN, ~17 NOTATION)
- `docs/critique/07-real-world-considerations.md` — 34 findings (7 CRITICAL, 9 BLOCKING, 14 MATERIAL, 4 EDGE)
- `docs/critique/08-formulation-goodness.md` — 57 findings (4 REARCHITECT, 7 PROMOTE-EARLIER, ~17 TIGHTEN, ~29 SAFE-TO-DEFER)
- `docs/critique/00-omitted-findings-index.md` — structured index of every finding NOT executed, grouped by agent + severity, for next-round triage.

---

## Session 21 archive — RESUME HERE block from end-of-Session-21

(Superseded by Session 22 RESUME HERE block above; kept for traceability.)

**FIRST THING TO EVALUATE on resume.** Max tardiness allowed at destination node (captured in `usr_session_notes.md`). State today:
- Max tardiness IS defined: $\tau_k \in [0, \max(0, T_k^{\text{abs}} - \Delta_k)]$ in C.14 domain; enforced by destination-reachability pruning in §forward-time-window propagation.
- Late real-route arrivals in $(\Delta_k, T_k^{\text{abs}}]$ are considered, pay quadratic penalty $W_k \cdot \tau_k^2$, compete against on-time options.
- **Open issue surfaced this session:** PWL grid for the quadratic linearization is fixed $\{0, 4, 12, 36, 96\}$ hours. Works for tight backstops (PER, ~6-48h max tardiness); breaks for wide backstops (GEN, ~720h max tardiness with 30-day customer-cancellation horizon) — linearization extrapolates linearly past 96h, underestimating the quadratic by ~4× at the upper end. Optimizer becomes too willing to pick very-late real routes over the fallback.
- **Two candidate fixes pending user decision:** (a) per-HAWB relative grid $\hat\tau_j = \alpha_j \cdot (T_k^{\text{abs}} - \Delta_k)$ with $\alpha \in \{0, 0.05, 0.15, 0.5, 1.0\}$ — spans the actual feasible range; (b) per-service-product tenant-configured grid.

**Status — end of Session 21, 2026-05-27 evening.** Long discussion session (Layer-3 messaging-agent + J19 time-propagation), executed J19 reshape in the air model, executed bloat cleanup pass per user's "doc is too long" directive. User signing off with one new open question to evaluate first thing tomorrow (above). PDF re-review of the post-Session-21 LaTeX is the next gate after the max-tardiness decision.

### Already executed this session (Session 21) — for context, not action

**J19 closed.** Time-propagation reshape committed to the air model. Mechanism = forward-time-window propagation at graph build. C.6, C.7, C.8, C.9, C.11 removed as MILP constraint families. C.10 rewritten as C.10a (destination-arrival definition: $t_k^{D_k^{\text{node}}} = \sum_{a \in \mathcal{A}^{\text{last}}_k} \text{arr\_dest}(k,a) \cdot x_{k,a}$) + C.10b (tardiness vs. soft deadline, quadratic penalty unchanged). Intermediate $t_k^n$ variable family eliminated — only $t_k^{D_k^{\text{node}}}$ survives. Base-scale estimate: continuous variables $\sim$4,500 → $\sim$2,100; constraints $\sim$10,500 → $\sim$8,000. New §forward-time-window propagation added to graph-construction section. Big-M tightening, lin-summary, walking-skeleton telemetry, base-scale estimate, and Open Items P1 #1 (TT-Service quantile binding) all updated for consistency. Recorded in `OPEN_DECISIONS.md` J19 with the full mechanism.

**Layer-3 messaging-agent MVP reshape documented.** New §6 in `docs/design/messaging_agent_capabilities.md` records the discussion outcome: original 6+3 MVP set is misshapen (B1+B4 don't need a chat surface, A6 is half-baked without A1, G1+G2 belongs to Layer-2). Reshaped honest MVP = consent + identity + lifecycle close + cargo-readiness slip + cutoff reminder; B1+B4 ship in planner console; G1+G2 ships in Layer-2 build; A6+A1 paired as the real flagship for later. **Decision: Layer-3 may not be built in MVP at all** — payload math estimates ~5-15% of total afternoon ops time touched by reshaped Layer-3, doesn't earn the buildout cost on its own. Re-evaluate when v2 write capabilities (BSA, rate, policy, embargo, equipment) are ready to bundle.

**Air model bloat cleanup (~180 lines removed).** Deleted: §Consolidation alternatives considered (47 lines), "What is not a variable" paragraph (17 lines), C.7/C.8/C.11 REMOVED stubs, §Mapping from Prior LaTeX (P.x→C.x) (37 lines), §Why consolidation matters economically (20 lines), standalone §Re-ULDing — Operational Mechanics (90 lines — folded by pointing to existing §Through-ULD policy and §1312 ULD interchange subsection), §Excluded from MVP (4 redundant bullets dropped, 5 substantive items folded into Deferred P1 list as new entries: in-transit hub customs, per-HAWB cost attribution, charter/BOR, AWB stock management, time-windowed carrier rules).

### LaTeX state — v3-rev-fallback (Session 20 edits, kept for traceability)

The air model changed structurally this session. Key edits to `model/air_freight_routing.tex`:
- **Abstract** — carrier-policy cascade clarified (explicit 5-layer + deny-wins formalism); quadratic-tardiness + fallback-arc framing replaces "hard backstop the only hard time bound" language
- **§1 Problem Statement** — new bullet on fallback-arc feasibility guarantee; tardiness bullet rewritten quadratic + value-coefficient; "What is not a hard constraint" section adds $T_k^{\text{abs}}$ removal
- **§3 Graph Construction** — NEW subsection `sec:fallback-arc` (full spec); new fallback row in arc-types table
- **§sec:hawb-params** — $T_k^{\text{abs}}$ redefined as mandatory-finite "latest valid arrival time"; new param rows for $\text{value}_k$, $\mu_k$, $W_k$; new paragraph defining tenant-globals $V^{\text{ref}}$ and $C^{\text{fallback}}$
- **§sec:prefilter** — predicate 6 renamed "tightening only"; "Empty subgraph" para rewritten (fallback always present, warning logged, no infeasibility branch)
- **§sec:variables** — new $\text{pen}_k$ variable
- **§sec:constraints** — recap nomenclature updated; C.10 rewritten for quadratic + PWL; C.11 marked REMOVED; C.14 domain adds $\text{pen}_k$ bound
- **§sec:objective** — linear tardiness replaced with $\sum \text{pen}_k$; fallback-cost term added; monotonicity invariant updated
- **§sec:linearization** — NEW subsection `sec:lin-tardiness` with tangent-cut PWL derivation, recommended grid {0, 4, 12, 36, 96}h, cost analysis, summary table row
- **§sec:p-to-c-map** — P.15 and P.20 rows updated
- **NEW §sec:output-diagnostics** — reported quantities table, rescue-signal protocol, post-solve invariants, operator presentation
- **§sec:deferred** — `item:quadratic-tardiness` marked PROMOTED to MVP
- **§sec:scaling-roadmap** — updated counts (continuous +100 for $\text{pen}_k$; constraints +400 for PWL cuts)
- **§sec:service-products** — SLA-soft-constraint paragraph rewritten quadratic; $w_p$ row updated
- **§sec:locked-commitments** — "Pre-MILP feasibility check" → "Pre-MILP reachability check" (early warning, not gate)

Also mirrored to `model/air_graph_construction.md`: new fallback arc row in §3, new step 12 in §5 Phase 1, NEW §10 with full design.

**Carrier-policy cascade abstract clarification (early in session, separate from fallback work):** replaced the jargon line with explicit 5-layer enumeration + "intersect allows, union denies, any deny blocks even when higher layer allows" formalism + cross-reference to `sec:carrier-policy`.

### `usr_session_notes.md` carried items (still pending — user has not triaged)

- §4.3 enumeration table — explicit grouping table in air LaTeX (from 2026-05-24)
- Per-shipment slack metric — P-quantile arrival vs service-tier deadline; replaces "confidence" as SLA quality measure (from 2026-05-24)
- **Max tardiness allowed at destination + PWL grid calibration** (new 2026-05-27, the FIRST THING to evaluate above)

### Pending from prior sessions

- User-initiated air model constraints review — user said "I will continue with constraints review" after Session 21 cleanup pass. Resumed only after the max-tardiness question is resolved.
- Air model PDF re-review post fallback-arc + J19 + quadratic edits (user was mid-§4 when Session 20 ended; Session 21 changed §3 / §sec:constraints structurally — full re-read recommended)
- Pitch deck slide 14 `[N]/[M]` forwarder pipeline placeholder (Session 19)
- All §J open items in `OPEN_DECISIONS.md`: J3 orchestrator concurrency design; J5-J13 critique-driven gaps; J18 pitch v7 update with the three new pitch upgrades (consolidation savings %, 4h→90min reframe, active-participant agent capabilities)

### Critique deliverables produced this session (all in `docs/critique/`)

- `01-commercial-viability.md` — automation envelope, pitch reframe
- `02-consolidation-savings.md` — 7-12% pitch-ready number with sourced backing
- `03-gap-finder.md` — ~30 findings, severity-ranked
- `04-persona-test.md` — per-persona fit + cross-persona gaps + "primary user reality check"
- `05-messaging-agent-prior-art.md` — competitive landscape, empty corner
- `design/messaging_agent_capabilities.md` — Layer-3 deep dive (26 caps, MVP picks, failure modes, architecture)

---

## Session 19 archive — RESUME HERE block from end-of-Session-19

(Superseded by Session 20 RESUME HERE block above; kept for traceability.)

**Status — end of Session 19, 2026-05-26.** Pitch deck rebuilt end-to-end through a six-step pipeline (v4 narrative reframe → 2 critique agents (VC + forwarder COO) → v5 incorporating critiques → designer agent for modern 2025-26 deck style research → v6 with full design system overhaul → PDF export). Primary deliverables: `pitch_deck/v6_final.pptx` + `v6_final.pdf` (14 slides). Vision lane A locked: "The planning brain for global freight" — infrastructure framing, replaces labor-savings/40→15% headline (unverified, dropped). Scope honest: air consolidation = MVP wedge; roadmap Air → Ocean LCL → Ocean FCL → Trucking → Intermodal explicit and phased (SEED / SERIES A+ / SERIES B+). Market sized off real sources (Armstrong $216B TAM, software $1.5-2B TAM today, 5-yr SAM $4-6B from labor-replacement math, SOM $40-120M base / $200-400M bull). MILP reframed from hedged tradeoff to moat ("explainable, auditable, autonomous-when-earned" + buy-vs-build defense). Tier 2 = seed landing, Tier 1 + Tier 3 = post-traction expansions. Design system: warm cream `#F6F5F0` surface, `#EFEEE8` raised tier, indigo `#4F46E5` accent (was cobalt), Geist Sans + Geist Mono (was Inter; installed via brew), letter-spacing via OpenXML `spc` attribute, asymmetric bento on slide 5, redesigned mock UI on slide 7 (no ASCII), footer wordmark + logomark on every content slide.

**Critique agent verdicts (carry into next round of deck work):**
- VC (seed/Series A partner persona): "Second meeting, not term sheet." Three structural problems — (1) `[N]/[M]` placeholder on slide 14 is fatal; need 5 named tier-2 forwarder conversations + ideally one signed LOI before sending; (2) 5-phase roadmap on $2.5M / 14mo not credible (now compressed to Air + Ocean LCL in v5/v6); (3) market math hand-waved (now bottom-up build with labor-cost derivation shown in v6).
- Buyer (tier-2 forwarder COO persona): "Pilot yes, production no." Buyer values **explainability + audit trail, not autonomy** — autonomy framing scares insurance/compliance. Buyer's KPI is **planner hours saved**, not override rate. Math is too clean — must handle allotments/BSAs, cargo-readiness uncertainty, co-loader relationships, DG/lithium, three coupled cutoffs (CFS/flight/dispatch). Buy-vs-build threat from in-house DS teams ($1.5-2M to build 60-70% in-house) is THE REAL COMPETITION, not WiseTech.

**Pitch deck — open items before VC send:**
1. **Replace `[N] / [M]` forwarder pipeline placeholder on slide 14 with real names + counts.** Single biggest disqualifier per VC critique.
2. **Design partner outreach** — line up 3–5 named tier-2 forwarder conversations to validate impact metrics (planner FTEs, OTP lift, cost-to-serve reduction). The "THE IMPACT" section on slide 3 still needs validation.
3. **Keynote font issue (carried)** — user reported Geist/Geist Mono/Helvetica still flagged in Keynote's "Choose which fonts to replace" dialog. Diagnosis path: Font Book → drag `~/Library/Fonts/Geist-*.otf` files in if not auto-registered; OR restart Mac to refresh font caches. Helvetica isn't in pptx XML at all (Keynote inserts as fallback for placeholders) — safe to "(Don't Replace)."

**Air model gate (unchanged from Session 18).** Stage 2 (PDF review of `model/air_freight_routing.tex` v3 → `model/air_freight_routing.pdf`) is still the gating action. Once approved: Stage 2.5 (delete `model/air_milp_spec.md`), then Stage 3a (`src/components/air_graph.py` Phase 1).

**`usr_session_notes.md` carried items (per user, leave in place):**
- §4.3 enumeration table — explicit grouping table in air LaTeX
- Per-shipment slack metric — P-quantile arrival vs service-tier deadline; replaces "confidence" as SLA quality measure

**Status — end of Session 18, 2026-05-25.** User stepped out for 10 hours and asked for an uninterrupted deep dive on workflow-vs-AI-vs-agentic-AI scoping. Executed: 4 parallel research subagents (front office / network ops / compliance-customs / exceptions-replanning) + a 4,600-word synthesis. Net effect: the project's MVP LLM-agent scope grew from 2 to 4 capabilities, the 4-bucket workflow/hybrid/AI-novel/MILP framework is now load-bearing, and the **optimization-first positioning is hardened** by F1 of the synthesis (only genuinely under-served wedge). Session-17 positioning disagreement effectively resolved.

**Competitive deep-dive 2026-05-25 (post-strategy-lock):** primary-source research on cargo.one confirmed they are **Bucket B (rate aggregation + per-shipment quoting + booking with RAG/LLM workflow automation), NOT a consolidation-wedge competitor.** Zero overlap with multi-shipment optimization across 15+ sources. The "AI-native OS" Feb–Mar 2026 launch is workflow rebrand + Cargofive ocean rates, not new optimization. Methodology = learned-preference ranking. MCP claim is positioning, not shipped tools. H11 closed. cargo.one is adjacent platform / potential booking complement via MCP. Separate clarification: CargoWise CTO is drayage planning only, competes with secondary surface (drayage/trucking pickup planning) NOT the primary consolidation wedge. Competitive.md row + Attack 1 + Attack 2 updated. Monitoring routine: quarterly check of `jobs.ashbyhq.com/cargo-one` for OR engineer hires as leading indicator that they're moving into Bucket D.

**Third continuation update — same day, evening.** User walked through the four DITL docs in turn and locked five strategic commitments to memory: (1) **Quote desk + consolidation planner = two-pronged primary MVP surface** — same MILP engine, two distinct UIs ([[project-two-pronged-wedge]]); (2) **Intelligence layer above TMS, TMS-agnostic** — adapter interface targets CargoWise → Magaya → GoFreight → Riege ([[project-intelligence-layer-positioning]]); (3) **Density-fit architecture** — assignment problem + ML feasibility predictor + replan-if-below-threshold; reject 3D bin packing ([[project-density-fit-architecture]]); (4) **Customs = integrate, don't build** — four data intersection points (cost, constraints, transit-time feature, replan trigger) ([[project-customs-integrate-dont-build]]); (5) **Persona 4 ≈ Persona 2 at mid-size** — same humans, replan UX = planning UX (updated [[project-core-user-reality]]). Drayage / trucking pickup *planning* = MVP secondary surface (planning work fits project DNA); drayage *dispatcher* (execution) → P1 / Phase 7. KAM and CFS supervisor deprioritized. **Planning vs execution distinction added as scoping principle** (memory `project_planning_vs_execution_boundary.md`): planning surfaces in scope, execution surfaces out of scope or integrated against. System diagram redrawn as five-layer stack: User surfaces → Agent layer → Intelligence components (we build) → Data adapters → External systems (we integrate). `docs/architecture.drawio` replaced with new five-layer stack.

**The 4-bucket framework** ([[project-workflow-vs-ai-buckets]] memory):
- A. Pure workflow — structured → structured. *Integrate.*
- B. Hybrid (AI parse → workflow execute) — high vol but commoditized vendor space. *Integrate, except WhatsApp/voice gap.*
- C. AI agent for low-volume novel reasoning. *Build.*
- D. Deterministic optimization (MILP/VRP/ML). *Build. Core wedge.*

**MVP LLM-agent capabilities — refined 2 → 4** ([[project-agent-role-taxonomy]] memory updated):
1. Input parsing — wedge is WhatsApp/voice/partner channel, not email
2. Ad-hoc query
3. **Materiality / re-plan-trigger assessment** *(new)*
4. **Re-plan trade-off explanation** *(new — bridges MILP output to operator)*

**Top 3 product-level findings** (full set F1–F10 in synthesis §2):
- F1. MILP optimization at mid-size is the only AI capability not commoditized. (Hardens optimization-first stance.)
- F3. WhatsApp/voice/partner free-text is the only unstructured channel without dense vendor coverage — the project's owned-AI surface. ([[project-unstructured-channel-wedge]] memory.)
- F8. Materiality assessment ("is this 6h delay material?") is a Bucket C task no incumbent owns; load-bearing for the disruption loop.

**Build sequence unchanged from Session 16.** This session was strategic / scoping work, not LaTeX or code. Stage 2 (PDF review of `model/air_freight_routing.tex` v3) is still the gating action; Stage 3 (Phase-1 `src/components/air_graph.py`) is still the bigger next step.

**Major reframes that landed this session** (now in memory + architecture.md):
- **Plan-goodness replaces "confidence"** — two orthogonal dimensions (SLA satisfaction + cost reasonableness); soft-plan-then-commit lifecycle; flags as orthogonal dimensions (SLA risk / cost outlier / rate surprise / capacity risk / disruption); commit at `cutoff − tier_safety_margin`; no auto-execute cell. Memory: `project_plan_goodness_reframe.md`.
- **Agent role scoped down from 5 to 2 for MVP** — input parsing + ad-hoc query; exception triage / customer comms / pattern detection / request decomposition rejected as UI features in disguise or risky LLM apps. Memory: `project_agent_role_taxonomy.md`.
- **Override rate is the central KPI**, not model accuracy. Trust-degradation tighten 15% → 5–8%. Memory: `project_override_rate_kpi.md`.
- **Rate sourcing — 5-tier strategy** with WebCargo as MVP aggregator default. Memory: `reference_rate_api_landscape_2026.md`.

**LaTeX edits this session** (additive to Session 16 v3 baseline; PDF review still pending):
- Screening dropped as consolidation grouping key AND arc-eligibility filter → 24 buckets → 6
- MAWB fixed-charge term added to objective (~$50/MAWB placeholder)
- §4.5 (CW density mixing) restructured with full nomenclature table + per-equation explanations
- §sec:sets migrated into §sec:variables (renamed "Decision Variables, Sets and Indices")
- Recap nomenclature tables added at top of §sec:constraints, §sec:objective, §sec:linearization
- `U_a`, `K_a`, `G_a`, `M` definitions expanded; catalog vs solve-specific paragraph added
- Multi-stop MAWB-arc enumeration policy filed into `model/air_graph_construction.md` §5 Phase 1

**New artifacts created this session:**
- `architecture.md` (root) — system architecture narrative
- `OPEN_DECISIONS.md` (root) — comprehensive pending-changes catalog
- `docs/architecture.drawio` — system diagram
- `docs/agent-critiques-2026-05-24.md` — round 1 critique outputs verbatim
- `docs/industry-precedent-research.md` — **flawed; see SESSION_LOG retractions; kept for record**
- `docs/air-and-lcl-route-planning-research.md` — corrected research
- `docs/agent-critiques-round2-2026-05-24.md` — round 2 critique outputs verbatim
- `docs/academic-literature-references.md` — verified academic refs (top 5)
- `references/` folder — user-added Archetti & Peirano 2020 PDF (the Bergamo case study)

**Infrastructure changes:**
- `CLAUDE.md` — new Guardrail for `note:` capture; sign-off protocol now 5 steps (added Step 1: session-notes triage)
- `.gitignore` — `usr_session_notes.md` excluded
- `usr_session_notes.md` — created; 2 items pending and carried forward: §4.3 enumeration table edit + slack metric design

**End-of-session positioning disagreement — DO NOT re-litigate on resume.** User rejected both "productivity-first reposition" and "doc-AI wedge" framings as narrative-fitting / commoditized. User's position (to verify): **build the optimization product as designed; productivity / interface concerns are implementation detail, not headline.** Tomorrow start from there.

**Status of Session 16 deliverables (unchanged from yesterday):** Stage 2 complete — `model/air_freight_routing.tex` v3 rewritten on the O-D-arc graph with `(arc, g)` MAWB. 3,055 lines (+ today's polish edits), 22 sections. All Session-15 critique fixes baked in (3-inequality `min_flat_breaks`; C.7 removed; `Δ_k` rename; per-MAWB upper-link bounds; per-constraint tight big-M; MCNF supply-form sign convention; etc.). All Session-14 locked decisions baked in (item 3 linear soft tardiness; item 7 IATA next-break-down; item 13-A `cw_k → w_k` fix in C.5b-w; item 15 P.18 removed; item 18 cleanups; Finding S Ch 1 framing + TT-quantile hook). All operational depth preserved verbatim (time-zone, ULD specs, embargo, lithium, screening, locks, service products, carrier policy, surcharges, re-ULDing mechanics).

**Goal:** get the air model working **end-to-end** — graph + MILP — with synthetic shipments and a regression test suite.

### Step-by-step plan

| # | Stage | Deliverable | Tests | Status |
|---|---|---|---|---|
| 1 | **Formulation spec** | `model/air_milp_spec.md` — variables, constraints, objective on the O-D-arc graph; folds in items 3, 15, 18, Finding S Ch 1; refreshed Tractability §13 with instrumentation + base-scale estimate + walking-skeleton subset | spec-level: 3-agent critique | **v2 COMPLETE (Session 15)** |
| 2 | **LaTeX rewrite** | `model/air_freight_routing.tex` v3 rebuilt from spec v2 — wholesale replacement of the per-leg bucket structure; carry-over operational sections preserved; spec deletes after PDF approval | user PDF review | **COMPLETE (Session 16) — awaiting user PDF review** |
| 2.5 | **Spec deletion** | Delete `model/air_milp_spec.md` (transient artifact, by agreement) | --- | gated on PDF approval |
| 3a | Graph generator — Phase 1 | `src/components/air_graph.py` — physical graph (nodes, transport / dwell arcs, per-shipment pre-filter with all 8 §4 steps); precomputes per-arc scalars `μ_a` and `CO_a^*` (folds internal MCT + cutoff stack at build) | `tests/components/test_air_graph.py` Phase-1 cases | next after PDF approval |
| 3b | Graph generator — Phase 2 | MAWB overlay (`g(k)`, MAWB instantiation, co-load skip) | Phase-2 unit tests |
| 4 | **MILP — walking-skeleton ladder** (per LaTeX v3 / spec §13.3) | | |
| 4-v1 | Minimal viable | `src/components/air_milp.py` — C.1 flow, C.2 MAWB activation, C.4 CW aggregation, C.5c per-offer cap, C.6 time propagation, C.9 cutoff, C.10 soft tardiness, C.11 hard backstop, C.14 domain. Rate families: **`flat_rate` + `coload_per_kg` only**. No `γ`/`BW`/`η`. ~3 variable families. Tests `x`-binary scaling, C.6 density, LP-gap, big-M tightness. | trivial + infeasibility + density-mixing tests |
| 4-v2 | Add `min_flat_breaks` | γ + BW + corrected linearization §10.1 (3-inequality form, no `BW_b ≤ CW`) | TACT round-up-to-higher-break case ($800 not $900) |
| 4-v3 | Add `per_uld_pivot` | η + C.5 + C.5b + C.13 (per_flight + equalized); pre-filter step 8 (HAWB-too-big-for-any-ULD) | BSA pivot binding; allowance overage |
| 4-v4 | Tractability instrumentation | All 8 metrics from LaTeX §13 / spec §13.1 wired into solve loop output | shadow-mode (no behavior change) |
| 5 | **Integration** | graph + MILP end-to-end on small instances | `tests/integration/` |
| 6 | **Test shipments** | 5+ synthetic instances covering direct / hub / consolidation / DGR / VAL / co-load | fixtures in `tests/conftest.py` |
| 7 | **End-to-end** | full pipeline (instance → graph → MILP → verified output) | `tests/e2e/` |
| 8 | **Regression** | `uv run pytest` runs the full suite on every change | policy |

**Test plan:** see `TEST_PLAN.md` — philosophy, per-component scenarios, fixtures, regression policy. §10 wires Agent 3's 8-metric walking-skeleton observability suite. CLAUDE.md project rules: no solver/graph mocking; real small instances; solution-value bounds; one happy-path + one infeasibility per module.

### Key docs to read on resume (in order)
1. **`model/air_freight_routing.tex`** v3 — the rewritten formal model. Start with §1 (Problem Statement), §3 (Graph Construction summary), §4 (MAWB and HAWB — including §4.4 alternatives considered A–F), §9 (Constraints C.1–C.14), §10 (Linearization with the corrected 3-inequality `min_flat_breaks`), §13 (P.x → C.x mapping), §16 (Tractability + walking-skeleton instrumentation).
2. **`CONTEXT.md`** (this file).
3. **`model/air_graph_construction.md`** — graph-construction logic the LaTeX sits on. Read §1, §5 (two-phase), §6 (case catalogue), §7 (resolved decisions).
4. **`model/air_milp_spec_critique.md`** — verbatim 3-agent critique findings (61 total) on the spec. Survives spec deletion; the receipts for what was flagged and why.
5. **`model/air_review_notes.md`** — review outcomes and design decisions feeding the spec / LaTeX.
6. **`TEST_PLAN.md`** — testing strategy. §10 has the walking-skeleton observability metrics.
7. **`SESSION_LOG.md`** — Sessions 14 + 15 + 16 detail.
8. **`docs/air_graph_construction.drawio`** — multi-shipment graph with CFS and customs nodes.

**Note on `model/air_milp_spec.md`.** Still on disk; **transient** per Session-14/15 agreement. Deletes after the user PDF-reviews `model/air_freight_routing.tex` v3 and accepts it. The verbatim critique receipts in `model/air_milp_spec_critique.md` survive the deletion.

**Next action on resume — pick one:**
1. **Read `docs/forwarder-operations-analysis/00-synthesis.md`** (~4,600 w) — the executive summary of this session. Then **triage `OPEN_DECISIONS.md §H` (12 new items, H1–H12)** — top 3 needing user input first: H1 (confirm 4-capability agent scope), H4 (WhatsApp in v1 or v2), H11 (cargo.one paid trial before locking agent boundary).
2. **Compile + review `model/air_freight_routing.tex` v3 PDF.** Still pending from Session 16. On approval: (a) delete `model/air_milp_spec.md`; (b) move to Stage 3.
3. **Slack metric design** (session note — now informed by synthesis §5: "materiality assessment" is the Bucket C task the slack metric operationalizes on the SLA dimension).
4. **§4.3 enumeration table** (session note — small LaTeX edit).
5. **Stage 3 — `src/components/air_graph.py` Phase 1** (physical graph: nodes, all 10 arc types per §3.2 of the LaTeX, per-shipment pre-filter with all 8 predicates from §8) with `tests/components/test_air_graph.py` Phase-1 isolation tests per `TEST_PLAN.md §4.1`. Before writing Phase-1 code: verify the carry-over operational dependencies the graph needs (cutoff stack folded into `CO_a^*`; ULD pool / interchange decisions absorbed at graph build into MAWB-arc emission; in-transit hub customs folded into `δ_a` of deconsol-dwell arc).

### Pending user inputs (non-blocking, carried forward)
- Item 3 tardiness penalty weights `w_{sp(k)}` — `CALIBRATION NEEDED` placeholders
- Cost outlier multiplier `N` (Nx-of-lane-median threshold for flagging)
- Commit-window safety-margin defaults (Express 6h / Standard 12h / Economy 24h proposed)
- MVP rate aggregator final pick (WebCargo proposed)
- Final positioning on optimization-first vs anything else (user's frustration at sign-off suggests pure optimization-first; unconfirmed)

### Cross-model TODO (after air-MVP)
- Apply O-D-arc-graph thinking to LCL/ocean and trucking models.
- Cross-mode stitching layer.
- Agent layer / MCP tools.
- Operator UI.

---

## Session 15 — design decisions (now in `model/air_milp_spec.md` v2)

Two waves of decisions in Session 15: the v1-drafting design choices, then the v1→v2 critique-pass refinements. Both are now folded into spec v2 (which is the current source of truth for Stage 2). Full critique receipts in `model/air_milp_spec_critique.md`.

### Wave A — v1 architecture (during spec drafting)

The 5 design Qs opened with the spec draft, all closed in-session:

- **Flight-level physical capacity (`W_f`, `V_f`) — DROPPED entirely.** The forwarder doesn't know flight physical capacity in any planning sense (other parties' bookings invisible). Real caps that remain:
  - **C.5** per-contract allotment `N_{a,u}` (BSA-contracted ULD positions per arc per type).
  - **C.5b** per-ULD physical limits `W_u`, `V_u` (with the item 13-A bug fix `cw_k → w_k`; both weight and volume bounds retained).
  - **C.5c** per-offer cap `cap_a` / `cap_a^{cl}` where the offer specifies one. TACT/NAC typically uncapped at planning; capacity is request/confirm at booking.
  - No flight-level coupling across MAWB-arcs. The `arcs(f)` inverse map removed.
- **Per-ULD volume bound `V_u` — KEPT.** Light/bulky cargo (e.g.\ apparel ~80 kg/m³) binds on volume long before weight.
- **Surcharges — defer to LaTeX rewrite.** Math unchanged from prior LaTeX §6.7. Path-A additive in `c_a^{handle}`; Path-B per-flight indicator term.
- **In-transit hub customs (T&E etc.) — data-only on `δ_a`** of the deconsolidation-dwell arc. No new arc type, no new constraints.
- **Locks — preprocessing, not MILP constraints.** Fully locked HAWB preprocesses out of `K`; partially locked HAWB enters with origin re-pointed + truncated forward subgraph + fixed `t_k^{O_k}`. Lock-break is an **orchestrator decision between MILP runs**; the MILP is lock-agnostic. No `b_k`, no buyout decision variable. `lock-buyout` moved from deferred to excluded.

### Wave B — v1→v2 critique-pass (post 3-agent review, 61 findings)

- **CRITICAL bug fixed — `min_flat_breaks` linearization (spec §7.2).** Previous form had `BW_b ≤ CW` which combined with `BW_b ≥ CW − M(1−γ_b)` forced `BW_{b*} = CW` for the chosen break, then with `BW_b ≥ break_b · γ_b` made any selection with `break_b > CW` infeasible — **banning the IATA round-up-to-higher-break case**. Worked example: 90 kg, breaks `(45, $10/kg)` and `(100, $8/kg)`; IATA = min(10·90, 8·100) = $800; broken spec forced $900. Fix: dropped `BW_b ≤ CW`; `BW_b = max(CW, break_b)` when selected, minimization recovers IATA naturally.
- **C.7 hub MCT family removed entirely.** All hub MCT absorbed at graph build into `μ_a` (same-MAWB through-connection) or deconsol-dwell arc `δ_a` (cross-MAWB transitions). C.6 time propagation alone suffices.
- **Notation pass.** Renamed scalar `D_k` (soft deadline) → `Δ_k` (disambiguates from destination node `D_k^{node}`). Added formal declarations for `tail(a)`, `head(a)`, `transit(k,a)`, `A^{cust}`, `A^{MFB}`, `C`, `C^{eq}`, `A_c^{MAWB}`, `r_c`. Formalized `g(k)` as switch on `cargo_class`. Renamed dunnage `δ → ε`.
- **C.1 flow conservation sign convention** flipped to standard MCNF supply form (outflow − inflow = +1 at source, −1 at sink).
- **Upper-link bounds added to C.14 domain:** `CW ≤ CW^{ub}_{a,g} · z_{a,g}` (empty-bucket ⇒ zero); per-MAWB `η ≤ N_{a,u}` (tighter than aggregate); `pivot`, `over_c`, `τ_k`, `t_k^n` all get tight upper bounds.
- **Per-constraint tight big-M formulas** in new §8.1 — `M^{C.6}_{k,a} = M^{C.9}_{k,a} = T_k^{abs} − t_k^{rdy,early}`, `M^{BW}_{a,g} = CW^{ub}_{a,g}`. Per-shipment, not global.
- **C.3 dropped** as redundant with C.2a (both critique agents confirmed).
- **C.5b accepted-looseness remark.** Per-MAWB aggregate ULD cap doesn't enforce per-HAWB fit. Added pre-filter step 8 in §4 (HAWB-too-big-for-any-ULD on per-ULD-pivot arcs).
- **Walking-skeleton ladder.** §13.3 sequences v1/v2/v3 implementation (reflected in Stage 4 table above).
- **§13 Tractability refreshed** — three concrete forms (`scale-hawb-aggregation` ← `scale-y-aggregation`; `scale-bucket-dominance` ← `scale-option-dominance`; `strat-v2-mawb-rescale` deleted). §13.1 8-metric instrumentation table. §13.2 concrete base-scale estimate (~2,500 binaries at MVP).
- **Misc cleanups.** Split `c_a^{handle}` → `c_a^{flat} + c_a^{kg}`. `min_chg_a` clarified as per-MAWB. `cap_a` actual-weight per-arc. `legs(a)` and `arcs(f)` pushed out of MILP nomenclature (graph-build only). C.8 dropped (duplicate of C.6 initial condition). `pivot_a,g` → `pivot_{a,g}` brace hygiene. Monotonicity invariant stated at top of §7.

Mapping table §12 fully updated to reflect removals (P.14, plus the implicit numbering gaps from C.3/C.7/C.8 deletion).

---

## Session 14 — locked design decisions (carry into the formulation spec)

**Air model 19-item review COMPLETE.** Locked outcomes (full detail in `model/air_review_notes.md`):
- **Item 2 (currency)** — MVP = USD canonical, single FX table, convert at solve; no per-run pinning.
- **Item 3 (soft deadline)** — **LINEAR** tardiness penalty `+ Σ w · τ_k`; quadratic kept as deferred refinement. Weights = `CALIBRATION NEEDED`.
- **Item 4 (bucket)** — **superseded mid-session by the O-D-arc-graph architecture** (see `air_graph_construction.md`). The per-flight-leg bucket was wrong; MAWB is a per-segment object → `MAWB = (arc, g)`.
- **Item 4 / BSA cost** — 3 rate families (`rate_family ∈ {flat_rate, min_flat_breaks, per_uld_pivot}`); `settlement_basis ∈ {per_flight, equalized}`; `per_flight` = P.10 pivot; `equalized` take-or-pay = an exogenous per-solve sunk allowance `A_c`, BSA = 2-segment offer (free up to `A_c`, then `r_c`). Hard period-count P.7 removed.
- **Item 7 (TACT)** — two explicit rate-function families (cumulative vs min-over-flat-breaks); `rate_family` a per-offer catalog attribute.
- **Item 13 (P.3 bug)** — use `w_k` (actual weight) not `cw_k` (chargeable) in the ULD weight cap. Fix outright.
- **Item 15 (P.18 removed)** — per-shipment hard budget cap removed entirely (a hard cap can make a committed must-serve shipment infeasible; budget is a quoting-layer concern). Run-total ceiling removed too. P.19 pre-solve check + P.21 domain updated; renumber P.19/20/21.
- **Item 18** — delete deferred `sla-soft-otp` (superseded by item 3); refresh 3 stale Tractability items; `multi-seg-pu-pwl` softened to "contingency, not assumed" after research (multi-tier per-ULD over-pivot not found as a standard tariff form).
- **Finding S Change 1** — P.20 soft (covered by item 3) + a quantile hook (`t_k(d(k))` → TT-Service P85–P90 quantile once integrated) + "planning bound, not contractual guarantee" framing. Changes 2 (offload priority into the MILP) and 3 (A2A/D2D + `max_hops` attributes) deferred — recorded as Open Items.

**Air graph construction (Session-14 redesign — the new foundation):**
- **Three air arc types:** MAWB-arc (forwarder-consolidated, carries MAWB object), co-load arc (per-kg, no MAWB), deconsolidation-dwell arc (at hub `CFS-H`).
- **Two-phase construction:** Phase 1 = physical graph (nodes + all transport / dwell arcs + per-shipment pre-filter, no MAWB objects); Phase 2 = MAWB overlay (`g(k)` compute, `(arc, g)` MAWB instantiation, co-load arcs skipped).
- **8 node types:** door, CFS-O, POL, hub, CFS-H, POD, CFS-D, door. `CFS-O/D/H` may be off-airport or on-airport (cartage arc time/cost captures the difference). `CFS-H` exists only at hubs where the forwarder operates a warehouse.
- **Cartage arcs** `CFS-O → POL` and `POD → CFS-D` (new).
- **Customs clearance dwell arc** between `CFS-D` and final delivery (per-HAWB `δ_cust_k`; not per-MAWB / per-group).
- **Consolidation group `g(k)`:**
  - Consolidable: `g(k) = (cargo_class, screening_status, temperature_regime)`.
  - Non-consolidable (VAL / HUM / AVI): `g(k) = (cargo_class, HAWB-id)` → singleton → own MAWB.
  - **Partition by construction** (pairwise disjoint and exhaustive) because `g` is a single-valued function of an attribute tuple. The "no subset" rule is too weak (permits partial overlap); pairwise-disjoint is the correct rule.
- **DGR coarse:** all DG in one consolidation group; fine-grained pairwise DG segregation is a ULD-layer concern, deferred.

**Cross-model TODO:** ocean FCL `P.4 budget cap` / LCL / trucking share the P.18 defect — revisit on their rework.

---

## Code build — started Session 13 (2026-05-20)

**First code written.** Project scaffolding + walking-skeleton air MILP.

- `pyproject.toml` — uv-managed; deps: highspy, numpy; dev: pytest, ruff, structlog. Python 3.12+. `src/` layout.
- `src/components/air_milp.py` — walking-skeleton air MILP. Direct flights only, flight-level capacity, flat per-kg rate. Implements simplified P.1 (assignment), P.2/P.3 (capacity), P.15 (deadline pre-filter), P.21 (domain). HiGHS via highspy.
- `tests/conftest.py` + `tests/components/test_air_milp.py` — 5 isolation tests passing (happy path, cheapest-flight, 2× infeasibility, weight-capacity split).
- Build sequence for the air MILP: 8 incremental steps in `graph_generator_build_plan.md` Track A.1 — current state is steps 1–2 (skeleton + capacity). Next: time-window layer (P.11–P.13, P.15 full), then hub MCT (P.14), pivot weight (P.10), cargo type/fit (P.16/P.17), service product + carrier policy (P.20), locked commitments (P.19).

**Build plan rewritten v1 → v2** after 5-agent critique. `graph_generator_build_plan.md` is now air-first, three parallel tracks (A: air MILP, B: test harness, C: calibration data infra), honest 14–18 week Phase 1 estimate. Deferred items (M5 demand stream, M6 variable supply, M7 orchestrator, M8 operator UI, M8.5 agent test harness) moved to `graph_generator_build_plan_phase2.md`.

## Current Phase

**Phase 0 — PRD.** PRD v0.3 in review (reorganized 2026-05-16). Not yet formally approved.

**Phase 1 — LaTeX models drafted in parallel** per user direction (don't wait for PRD approval to flesh out math models). **4 of ~11 models drafted; Air model in v2 scope revision:**

| Model | File | Status | PDF |
|---|---|---|---|
| Ocean FCL | `model/ocean_fcl_routing.tex` | Draft v2 | rendered, 677 KB |
| Air Freight | `model/air_freight_routing.tex` | **APPROVED 2026-05-31 (Session 23)** — first gated formal model; Phase 2 air component build unblocked | ~4,300 lines LaTeX, 69 pages; user-compiled + reviewed |
| Ocean LCL | `model/ocean_lcl_routing.tex` | Draft v1 | 14 pages |
| Trucking (FTL/PTL/LTL) | `model/trucking_routing.tex` | Draft v1 | 16 pages |

**Remaining LaTeX models** (~7): Graph Generator, Instance Generator, Transit Time Models (ocean/trucking/air/path), Destination Leg Planner, Rules Engine.

**Phase 2-7 (code):** Not started. User confirmed laptop-feasibility for entire MVP build.

---

## Air model v2 revision — in progress (Session 10, 2026-05-17)

Two adversarial critique agents (technical + practitioner) reviewed `air_freight_routing.tex` Draft v1. User opted to walk through scope decisions point by point before formalizing v2. Approach: opinionated Claude rec → user final call → inline LaTeX edit → immediate PDF rebuild.

**v2b — Practitioner scope (27 tasks total, 9 closed). P0 Critical cluster fully closed.**

| # | Task | Status |
|---|---|---|
| 1 | MAWB / HAWB restructure | ✓ scope agreed; conceptual content in LaTeX §2; decision-vars deferred to v2 MAWB rewrite |
| 2 | DCO + AMS + ICS2 + ACI cutoff set | ✓ added to LaTeX; P.11 + subgraph step 3 use effective cutoff CO_f* |
| 3 | Embargo modeling | ✓ added; pre-filter pattern; 11-field schema; embargo_ok predicate |
| 4 | Lithium battery PI classification | ✓ added; whitelist matrix; commodity attributes; lithium_ok predicate |
| 5 | Supply layer generalization | ✓ added; supply_type enum; co-loader dual-mode; GSA as markup; spot TTL |
| 6 | Through-ULD ψ policy correction | ✓ closed Session 11. ULD interchange set Π added (Star/SkyTeam/oneworld); 3-case rule; operating vs marketing carrier; cross-ULD case; rationale remark + worked example. Bundled Tech C5+M4 (P.14 + rehandling cost linearization cleanup). |
| 7 | Locked-in commitments (K_locked) | ✓ closed Session 11. K^loc + per-arc locked prefix A_k^loc (and locked-off set); 7-state lifecycle → lock-posture mapping; locks derived from lifecycle + execution events; P.19 Locked Commitments constraint (renamed Domain to P.20); sunk costs retained in objective for traceability; lock-induced infeasibility routed as structured rescue event; P1 lock-buyout deferred. |
| 8 | Service-level commitments | ✓ closed Session 11. Named service-product catalog P (PRM_AIR_EXP, STD_AIR, ECON_AIR, MM_EXPEDITED, MM_STD, MM_ECON, OCN_EXP, OCN_STD); per-shipment sp(k) binding; bundle attributes = mode_allow, carrier_allow, carrier_deny, ac_type_allow, T^SLA, handling_tier, cargo_type_min; subgraph pre-filter predicates mode_ok/carrier_ok/ac_type_ok added to flight reachability pass; P.20 Transit-Time SLA hard constraint (renamed Domain to P.21); P1 soft-OTP penalty deferred (item:sla-soft-otp); SLA breach = rescue event. |
| 9 | Carrier blacklist / preference | ✓ closed Session 11. Layered cascade above service product: tenant blacklist → shipper-lane allow/deny → service product → lane preference → commodity overlay. Deny-wins conflict semantics. Resolved per-shipment sets C_k^allow, C_k^deny, C_k^pref. carrier_ok predicate redefined to use resolved sets (Eq. carrier-ok-resolved supersedes sp-carrier-ok). Soft preference via lexicographic two-pass: Pass 1 cost min → z*; Pass 2 max preferred-carrier count s.t. cost ≤ z* + ε^pref. Rules engine is a separate component (own LaTeX model in Phase 1). Time-windowed rules + ML override-learning deferred to rules engine model + Phase 5 constraint learning. |
| 10–22 | P1 important items + practitioner v2 critique pass | Closed via re-run practitioner agent + Groups 1–5 in Session 11. 17 findings triaged: 12 closed in model (incl. surcharge data model, ULD attribute completeness, screening cert, CGC by cargo type, cargo-ready window, supply-side lock invalidation, time-zone convention, B/L release type, currency/FX, shipment attributes doc, clearance dwell); 2 doc-only (sell-rate scope note, booking-rejection recovery); 2 deferred P1 with sourced rate notes (CFS storage/demurrage, partial-split shipment); 1 SKIP-with-note (per-flight lithium aggregate is carrier-side); 1 skip outright (AWB stock). New supporting files: `shipment_attributes.md` (295 lines); `data_model.md` extended with §4 Policy Rules, §5 Spot Rate Snapshots, §6 Surcharge Catalog, §7 Currency/FX (1,148 lines total). |
| 23–27 | Over-engineering drop decisions | Implicitly handled by the user-driven triage in Session 11 — items skipped or deferred as scope decisions during the 17-finding triage. |

**v2a — Technical math correctness pass — superseded Session 11 by fresh 3-agent critique pass on the post-v2b model:**

Session 11 launched 3 parallel critique agents on the post-v2b air model: (1) **notation & formulation correctness**, (2) **linearization & MILP technique**, (3) **simplification & tractability at scale**. The 3 agents produced ~56 findings (15 + 21 + 20) clustered into 5 themes by fix-shape; all bugs and tightening items closed in Session 11. Original v2a list from Session 10 is superseded by this fresher pass.

| Cluster | Items | Status |
|---|---|---|
| **1. Real bugs (correctness)** | 6 — B1 x_f^k undefined, B2 pickup-window not enforced, B3 τ_k overloaded, B4 per_uld surcharge bilinearity, B5 χ binary misstatement, B6 CO_f^* missing k | ✓ all closed (Cluster 1 sweep). New x_f^k shorthand defined; P.21 extended with cargo-ready-window constraint; categorical `\ctype` macro replaces overloaded τ_k; per_uld surcharge re-attributed to flight-level (Path B); χ now continuous [0,1]; subgraph CO subscript fixed. |
| **2. Tightening (correct but loose)** | 5 — T1 per-constraint tight big-M, T2 P.14b → (1−χ), T3 P.10 disaggregation note, T4 P.19 inequality form + pre-solve check, T5 ε^pref ≥ Pass-1 MIP gap | ✓ all closed (Cluster 2 sweep). §10.3 rewritten with per-constraint M^P11/M^P12/M^P13/M^P14a/M^P14b. P.19 uses two-sided inequalities for clean presolve diagnostics. |
| **3. Notation hygiene** | 9 — cargo-type enum canonical {GEN,DGR,PER,VAL,AVI,HUM}, Hub_k(j)/Hub(k) split, wildcard fix, ζ scope, ξ role note, P.18 attribution, F_c(t) definition, function-style convention note, cargo_type_ok predicate | ✓ all closed (Cluster 3 sweep). |
| **4. Tractability roadmap** | 8 simplification levers + 4 strategy notes | ✓ documented as new §Tractability and Scaling Roadmap with sourced benchmarks. Not active model changes; gated on production solve-time data. |
| **5. Strategy notes** | Folded into Cluster 4 in §Tractability section | ✓ documented |

**Net result:** the air model has had a complete math correctness + linearization + notation pass on the v2b structure. Outstanding tech work is now empirical (instrument pre-filter survival rate; measure LP-gap-source post-deployment).

---

## Immediate next steps (start of next session)

**User is doing a personal PDF review of the air model post-Session 12 PWL-active rewrite**, plus reviewing the three new docs created 2026-05-19 (`scalability.md`, `SYSTEM.md`, `transit_time_model.md`), plus the doc-reorg proposal in `SYSTEM.md` §11. Then continues with LCL model work.

**New docs created in Session 12 (2026-05-18 → 2026-05-19):**
- `scalability.md` — large-scale solver strategies (decomposition, column generation, matheuristics) + SPPRC pricing-subproblem sketch for the air model. Marked as methodology-level, not approved-formal-model.
- `SYSTEM.md` — top-level systems / architectural index. Mermaid diagrams for: system architecture, lifecycle state machine, mode-handoff dataflow, FCL end-to-end shipment journey, transit-time-service three-phase architecture. Doc reorg proposal in §11 (propose only, not executed).
- `transit_time_model.md` — full product spec for the 3-phase Transit Time Service (Quoting / Down-Select / In-Transit ETA). Phase 2 first; Phases 1 and 3 are orchestration layers on top. Per-arc-type LightGBM quantile regression. Composition via Monte Carlo or Gaussian convolution. MCP tool surface defined.

**Doc-reorg executed 2026-05-19 (`SYSTEM.md` §11):**
- ✓ Promoted `docs/freight_concepts.md` → `freight_concepts.md` (top level; central reference)
- ✓ Moved `docs/taiwan_market.md` → `appendices/markets/taiwan.md`
- ✓ Moved `docs/us_market.md` → `appendices/markets/us.md`
- ✓ Cross-references updated in PRD.md and CONTEXT.md
- Held: merging `personas_and_tools.md` + `appendices/capabilities.md` (content merge, deferred for user review)
- Planned for later: add `model/transit_time/` subdirectory when LaTeX models per arc-type land

**Build-sequence implication:** Transit Time Service is on the critical path for component code (Phase 2 of the project build sequence). Every MILP that handles SLAs needs Phase 2 quantile estimates. So `transit_time_model.md` markdown spec needs approval before per-arc-type LaTeX models, which need approval before code.

1. **Air model status:** Draft v2 ready for user review after Session-12 PWL-active rewrite. ~3,400 lines LaTeX. User flagged Section 1 as outdated ("two supply layers" — wrong; reality is many supply types modeled piecewise linear). Session 12 promoted PWL to MVP-active: unified supply-option binary $y_{f,o,k}$ replaces $y_{f,u,k}^c + b_{f,k}$ (latter two kept as derived shorthands); per-shipment options handle TACT/SCR/NAC/GSA-shipment/spot via pre-computed $c_o(cw_k)$; per-ULD-pivot options retain $C_{f,u,c}$ + P.10 for hard BSA / pivot-shape GSA. New §6.7 Supply Option Catalog. §4.7 flipped from "v2 deferred" to "MVP active." Section 1 abstract + Problem Statement + MVP bullets fully rewritten to reflect 5-supply-type catalog, service products + SLA, locked commitments, carrier policy cascade, embargo/lithium/screening gating, cargo-ready window, UTC time-zone. Constraint sweep: P.2, P.3, P.8, P.9, P.10, P.17, P.18, P.19, P.21 + ζ linearization + locked-commitment lifecycle + tractability scale-y-aggregation + scale-option-dominance renamed.
2. **When user resumes:** corrections from their PDF review, or pivot to LCL model (`model/ocean_lcl_routing.tex`) — currently Draft v1 (14 pages, written Session 9, pre-dating the v2b operational additions, the 3-agent math review, AND the Session-12 PWL-active rewrite). The LCL model likely needs the same kind of pass the air model just went through.
3. **LCL work — expected scope:** the operational-realism additions from air v2b (service products, locked commitments, screening, surcharge data model, shipment attributes, time-zone convention, CGC by cargo type, cargo-ready window, customs dwell, B/L release type, currency/FX, supply-side lock invalidation, carrier policy cascade) all apply to ocean LCL with mode-specific adjustments. The math correctness + linearization + notation review pattern (3-agent critique) is the recommended next move once LCL operational scope is locked.
4. **Lesson logged from Session 11:** initial Task-#10 framing as "rolling BSA capacity release" was based on a fabricated tranche schedule. After sourced research (Levin/Nediak/Topaloglu 2012, IATA Net Rates docs, FreightAmigo), retracted and pivoted to spot rate snapshot data model. New memories saved: `feedback_no_fabricated_mechanisms.md`, `feedback_minimal_design_default.md`, `feedback_confirm_before_committing.md`.
5. **PRD review continuation, Graph Generator LaTeX, Transit Time models, Destination Leg Planner, Rules Engine LaTeX** all still pending after LCL.
6. **Obsidian vault** — re-synced at end of Session 11 (after all Cluster 1–4 edits). See `feedback_vault_sync.md`.

## Files modified in Session 11

| File | What changed |
|---|---|
| `model/air_freight_routing.tex` | Massive growth: ~3,162 lines. New §2 Time-zone Convention; expanded §6 with screening cert (§6.12), service products (§6.14), carrier policy (§6.15); expanded §6.1 with cargo-ready window; rewritten §6.4 ULD specs with 16 operational attributes; rewritten §6.7 surcharges with Path-A/B split; new P.20 SLA, expanded P.19 Locked Commitments with supply-side invalidation, P.21 Domain+Initial Conditions; new §Tractability and Scaling Roadmap; updated §10 Linearization with per-constraint tight big-M; new \ctype macro and canonical cargo-type enum |
| `data_model.md` | 1,148 lines. New §4 Policy Rules and Snapshots (generic 3-table framework); §5 Spot Rate Snapshots; §6 Surcharge Catalog (Path-A per-arc + Path-B flight-level); §7 Currency/FX |
| `shipment_attributes.md` | New file, 295 lines. Static + dynamic shipment attribute catalog; lifecycle state mapping; milestone event taxonomy; source-of-truth mapping |
| `CLAUDE.md` | Added "Do not auto-compile LaTeX" rule under guardrails |
| Memory files | `feedback_no_fabricated_mechanisms.md`, `feedback_minimal_design_default.md`, `feedback_confirm_before_committing.md`; updated `feedback_vault_sync.md` date |

## Laptop feasibility — confirmed at end of session 9

- **MVP development end-to-end runs on laptop** (modern Mac M1+/Linux, 16+ GB RAM):
  - HiGHS MILP solves: comfortable for 50-100 shipments per batch across all 4 modes
  - LangGraph agent: trivial (API calls only)
  - Local Postgres + Redis + Celery + FastAPI + Next.js: standard dev stack
  - NOAA AIS historical data: fits on laptop
  - End-to-end agent run (single shipment → graph → optimize → recommend): feasible
- **Production scale requires cloud** (1000+ shipments, multi-tenant, live AIS feed, CargoWise integration, 24/7 uptime)
- **Transition point:** Phase 5 (design partner integrations). Until then, laptop-only.

---

## What exists

### Top-level systems doc (2026-05-19)

- `SYSTEM.md` — top-level architectural index. Mermaid diagrams for system architecture (now including the simulation environment as a 6th layer), lifecycle state machine, mode-handoff dataflow, FCL end-to-end shipment journey, transit-time-service three-phase architecture. Doc map + reorganization status. References every other doc; cross-link companion to PRD.md (commercial index).
- `transit_time_model.md` — Transit Time Service product spec (3 phases: Quoting / Down-Select / In-Transit ETA). Build Phase 2 first; Phases 1 and 3 are orchestration layers. Markdown spec pre-approval; LaTeX models per arc-type deferred.
- `scalability.md` — large-scale solver strategies (decomposition, column generation, matheuristics) + SPPRC pricing-subproblem sketch for the air model. Methodology-level documentation.
- `graph_generator.md` — graph generator + simulation orchestrator. Four data layers (topology, fixed supply, variable supply, demand stream) + three trigger modes (scheduled, event-driven, UI-driven). Test harness for Phase 1 isolation tests, Phase 1.5 TT MVP, Phase 2 end-to-end, Phase 3 operator-UI flows. Build plan: 5–7 weeks of focused work.

### PRD and Specialist Files (v0.3 — reorganized 2026-05-16)

The monolith PRD.md (1,666 lines) has been decomposed into 8 specialist files. PRD.md is now the strategic index.

| File | Contents |
|---|---|
| `PRD.md` (v0.3) | Executive summary, problem statement, modes in scope, document map, differentiation requirements, **market opportunity (§6: TAM/SAM/SOM, target customer, commercial model, competitive window)**, open questions |
| `agent_design.md` | AI-native design philosophy (§1), agent capabilities (§2), agent architecture — LangGraph, hierarchical pattern, HITL, capability registry (§3) |
| `data_model.md` | Supply/demand model, G(N,A) graph, arc schemas, container specs, string allocation, rolling horizon planning, customer/tenant entity model (SQL schemas) |
| `ui_spec.md` | Look & feel, color system, screen inventory, persona views, agent action feed, mobile philosophy, wireframes (6 screens), interaction design |
| `personas_and_tools.md` | 4 personas, MCP tool inventory (P0/P1/P2), P0 priority summary |
| `build_plan.md` | Tech stack (FastAPI + Next.js + Clerk + AWS + Celery), multi-tenancy (Postgres RLS), demand generator, peripheral components (Stripe, notifications, Retool, S3), agent execution architecture, data sources, components inventory, build sequence, unit testing |
| `appendices/capabilities.md` | 60+ agent capabilities across 9 categories |
| `appendices/competitive.md` | 7 differentiation gaps, 14-company competitive landscape (updated with project44 Autopilot, Schematics portfolio companies) |

**Adversarial review history:** v0.1 reviewed 2026-05-08 (27 gaps); all agreed changes implemented 2026-05-10; v0.2 rewrite 2026-05-13; v0.3 reorganization 2026-05-16; adversarial critique of v0.3 not yet done.

**PRD review status:** Substantive review stopped after §3.2 (decision confidence tiers) in old PRD = agent_design.md §1.2 in new structure. §3.3 onward (guardrails, deployment modes, routing triggers) not yet reviewed this pass.

### LaTeX models (`model/`)

**`ocean_fcl_routing.tex`** — Ocean FCL Multi-Commodity Routing — Draft v2, 2026-05-13
- Binary Multi-Commodity Network Flow (BMCNF) formulation, P.1–P.5
- Status: draft, not yet approved. Adversarial review done; all agreed changes implemented.
- Key formulation: P.1 flow conservation (N_k indexed), P.2 vessel cap (α·alloc proxy), P.3 string allocation, P.4 budget, P.5 domain
- Open Items: 7; item 7 = vessel capacity proxy (α=0.20 placeholder)

**`ocean_lcl_routing.tex`** — Ocean LCL Routing with Consolidation — Draft v1, 2026-05-16
- Joint bin-packing × routing MILP on 6-layer graph (O, CFS-O, POL, POD, CFS-D, D)
- 16 constraints (P.1–P.16): flow conservation, shipment-to-container, container vol+wt capacity, type-per-slot, sailing cap, arc-sailing linkage, hazmat pair exclusion, CFS cutoff, time propagation, deadline, DGR compat, piece fit, budget, domain
- Two container types modeled (FEU 76 m³/26.5t and TEU 33 m³/24t)
- CFS dwell explicit at both ends (β° = β^D = 1.5 days configurable)
- Sequential decomposition solution strategy documented for tractability (joint MILP for |K|≤50, decomposition + Benders cuts for larger)
- 10 deferred items
- PDF rendered: 14 pages, `model/ocean_lcl_routing.pdf`

**`trucking_routing.tex`** — Trucking Multi-Mode Routing (FTL/PTL/LTL) — Draft v1, 2026-05-16
- Three-mode MILP (FTL, PTL, LTL) with carrier-tendering semantics
- 17 constraints (P.1–P.17) including 6 hard refusal/feasibility constraints from adversarial critique: LTL linear-foot (12 ft), piece length, piece weight, total weight, contract FTL allocation cap (parallel to ocean string), MABD delivery window
- (Carrier, origin SC, destination SC) tuples instead of hub routing — corrects Powell-Sheffi misapplication for forwarder context
- Tender acceptance probability as first-class parameter: `c_exp = c_base × [1 + (1-p_acc)(ρ_re-1)]`
- LTL pricing: NMFC 2025 Standard Density Scale + FAK class override + weight-break tier deficit rule (pre-computed per shipment per arc)
- Time discretization to days + slot lex-ordering symmetry breaking
- Container-to-chassis: one ocean container per truck (TEU/FEU compatibility)
- 14 deferred items including DOT Hours of Service, backhaul optimization, specialty equipment, dedicated lanes, pool distribution, cross-border, stochastic tender modeling
- Academic anchors: Powell-Sheffi (1983), Powell ADP/Optimal Dynamics, Caplice MIT FreightLab, Sheffi (1985), Erera (2020), 2025 NMFC SDS
- Adversarial critique applied (10 corrections, see §10)
- PDF rendered: 16 pages, `model/trucking_routing.pdf`

**`air_freight_routing.tex`** — Air Freight Multi-Commodity Routing — Draft v1, 2026-05-16
- BMCNF on commodity-specific subgraph G(N_k, A_k) of air network
- 19 constraints (P.1–P.19): flow conservation, ULD volume+weight capacity, per-flight contract cap, aircraft position cap, period cap, hard BSA take-or-pay (M from upstream model), arc-to-ULD linkage, spot/contract exclusivity, pivot weight linearization, cargo cutoff, time propagation, hub MCT, deadline, cargo type compat, ULD physical fit, budget, domain
- Two air capacity layers modeled simultaneously: contracted BSA (LD3/LD7/PMC/AKE per-flight allocation, pivot weight take-or-pay, hard vs. soft) + spot rate-card (IATA weight breaks)
- Each scheduled flight leg is one arc (multi-stop flights decompose into multiple arcs)
- Through-ULD vs. re-ULDing at hub: parameter ψ in MVP (decision var deferred to P1); MCT and rehandling cost differential
- Linearization: pivot weight max() via aux var, bilinear y·y via McCormick, big-M tightening
- Period commitment M_{c,u,t} computed by separate upstream model — routing MILP takes as fixed input per run
- 10 deferred items
- PDF rendered: 17 pages, `model/air_freight_routing.pdf`
- Status: draft, not yet approved

### Competitive research (`Research.md`)
- Created 2026-05-13; 33 URLs, 14 companies
- Full company profiles: capabilities table, autonomy level, operator UI model, guardrails detail
- Synthesis: 8 industry patterns, 5 market gaps
- **Not yet synced to Obsidian vault**

### Adversarial competitive critique (`appendices/competitive.md` — updated May 2026 session 7)
- Added 6 companies (DSV/Tango, Optimal Dynamics, WiseTech detail, DecisionBrain, project44 Intelligent TMS detail, cargo.one updated profile) — now 16 companies total
- C.5: Honest assessment of where MILP differentiator holds vs. doesn't
- C.6: Moat analysis (weak years 1–2; override data + integration depth as path to durable moat by year 3–4)
- C.7: Three specific attack scenarios (cargo.one, WiseTech CargoWise bundling, Tier 1 productization) with timelines and probabilities
- C.8: Source list for adversarial critique

### Obsidian vault mirrors
- `~/Documents/PM-Brain/01-Projects/ai-freight-agent/PRD.md` — last synced 2026-05-10 (v0.1)
- `~/Documents/PM-Brain/01-Projects/ai-freight-agent/ocean_fcl_routing_model.md` — last synced 2026-05-10
- **Both stale — need sync at start of next session**

### New reference docs (`docs/`)

- **`freight_concepts.md`** — Freight domain glossary: HBL/MBL pairing, container lifecycle (16 stages), booking flow, B/L release types, trucking instructions, road consignment note, intermodal rail booking, ULD types and stored fields, chargeable weight formula, surcharge stacks (ocean + air), US customs filings (AMS/ISF/EEI), carrier alliances. (Moved from `docs/` to top level 2026-05-19.)
- **`appendices/markets/taiwan.md`** — Taiwan market analysis: TAM $15–20M / SAM $1.5–5M / SOM $300K–1M ARR; top 20 forwarders with TMS status (Dimerco = proprietary, Morrison = likely CW); Dimerco deep dive (Value Plus System®, API options, IATA ONE Record); competitive software landscape; design partner sequencing (Dimerco #1, Morrison #2, King Freight #3). (Moved from `docs/taiwan_market.md` 2026-05-19.)
- **`appendices/markets/us.md`** — US market analysis: TAM $75–160M / SAM $25–50M / SOM $2–8M; major US Tier 2 forwarder list with TMS; US TMS landscape (CargoWise dominant, GoFreight growing); regulatory complexity (ISF/AMS/PGA); sales motion, conference channels (NCBFAA, TPM Long Beach), ACV targets. (Moved from `docs/us_market.md` 2026-05-19.)

### Diagrams (`docs/`)
- `ocean_fcl_planning_graph.drawio` — FCL planning graph, Shenzhen → Chicago example
- `ocean_fcl_multi_shipment_graph.drawio` — multi-shipment graph (referenced in LaTeX abstract)

---

## Key decisions made

| Decision | Detail |
|---|---|
| Agent framework | LangGraph (not direct Anthropic SDK). LangSmith for observability. |
| MILP solver | HiGHS (`highspy`) |
| Container type (MVP FEU) | 40'HC — 76 CBM usable, 26,500 kg payload |
| Capacity unit | TEU slots throughout. BSA contracts in FEU → ×2 at input. |
| Graph architecture | Commodity-specific subgraphs G(N_k, A_k); 6-step BFS construction |
| Decomposition | Commodity-supply graph H; connected components solved independently |
| Mix algorithm | Explicit enumeration over f from f_min to 0 (not greedy) |
| n_k formula | max(ceil(v_k/76), ceil(w_k/26500)) |
| Instance generation | Demand-first; joint session required before implementation |
| Autonomy model | AI routes autonomously; humans govern exceptions only |
| Deployment modes | Co-pilot / Supervised / Autonomous — customer chooses; progressive trust expansion |
| Confidence tiers | Tier 1 (auto, ≥0.80) / Tier 2 (recommend) / Tier 3 (escalate, <0.50 always escalates) |
| Approval model | Dry-run window (60 min default, 15 min urgency); auto-commits on expiry |
| UI primary surface | Exception Queue + Operations Dashboard |
| Grouping options | By carrier, shipper, or receiver (operator chooses) |
| Options per shipment | Top 3: cheapest, fastest, most reliable |
| Override policy | Requires reason; logged to overrides.jsonl for constraint learning |
| Kill switch | Global + per-lane |
| Agent reasoning | 3 levels: feed line, exception paragraph, full LangSmith trace |
| Closest competitor | cargo.one — multimodal, AI-native, MCP, $20M Bessemer; no MILP optimization layer |

---

## LaTeX model — formulation summary

- **P.1** — Flow conservation, indexed over N_k
- **P.2** — Vessel capacity cap: Σ_{k:(i,j)∈A^k_OC} slots(k,ij)·x_{ij}^k ≤ α·alloc(s_{ij},t_{ij}), α≈0.20
- **P.3** — String allocation cap: sum over sailings of string s in period t ≤ rem(s,t)
- **P.4** — Budget cap (optional, per commodity)
- **P.5** — Domain: x∈{0,1}
- **Open Items** — 7 items; item 7 is vessel capacity proxy (α=0.20 placeholder)
- **P1 deferred** — multi-HS-code commodity schema, booking cutoff, PSS surcharge, per-lane σ split

---

## PRD — open questions (§15)

1. Decision-support vs. autonomous execution trigger conditions
2. Design partner selection
3. Live AIS feed vendor evaluation
4. Carrier booking APIs for autonomous execution
5. Pricing model
6. Emissions data source
7. Multi-agent framework — decided: LangGraph
8. LCL consolidation optimizer scope
9. Time-phased carrier capacity release (P1)

---

## Competitive intelligence — key findings

| Finding | Detail |
|---|---|
| Closest peer | cargo.one — multimodal (air+ocean), AI-native OS, MCP-connected, $20M Bessemer, same mid-market forwarder target. No MILP layer. |
| Best guardrail model | Shipsy AgentFleet — 8 documented guardrails, three confidence tiers, 94.2% autonomous resolution in production |
| Best progressive trust model | cargo.one — Co-pilot / Supervised / Autonomous deployment modes; project44 Autopilot — progressive trust with recommendation-only default |
| Primary market gap | No company has published MILP-based joint optimization for freight forwarders. Everyone does intelligent matching or procurement automation, not constraint-optimal routing. |
| Industry benchmark | 94.2% autonomous task resolution (Shipsy production). This is the target for "working at scale." |

---

## Guardrails reminder

- PRD must be approved before LaTeX models are approved
- LaTeX models must be approved before code starts
- No scope expansion without explicit confirmation
- Instance generator is a joint session — do not implement independently
- Do not reference banned company/product names (see CLAUDE.md)
- Read `Research.md` at session start if doing competitive analysis work
