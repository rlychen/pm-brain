# End-to-End Test Plan — Air Routing Pipeline (Phase 2, pre-replay)

**Source:** critique-13 test-design agent (Session 34). Buildable design, not yet implemented. Two tiers per
the owner's ask: (1) a small/moderate end-to-end correctness test touching most basic pieces; (2) a medium
use-case-coverage matrix.

## Why this exists

Existing coverage (191 tests) is strong but **almost entirely per-component / per-seam isolation**. The gaps
are deliberate **end-to-end** identities that no current test asserts:

1. No single test threads generator → graph → MILP → persist → reload → arrival-stream → flexibility as one
   instance **with hand-computed value oracles** (`test_generator_to_files` checks `load==in-memory` on a
   random instance with no oracle — only self-consistency).
2. No assertion that `air_graph.earliest_arrival` (A_k^min) ≡ the MILP's `arr_dest` (the D-A13 "walk ≡ scalar"
   identity — two code paths over the same scalars; nothing pins them equal).
3. No assertion that the MILP route re-walked by `air_transit_time.sample_route(s=0)` reproduces `arr_dest`
   (this is exactly the scorer's walk; untested as a closed loop).
4. No assertion that `flexibility.classify` on options **enumerated from a real `AirGraph`** yields a `flex_k`
   consistent with the graph/MILP (flexibility is tested only on synthetic `RouteOption`s).
5. No use-case matrix solved through the **whole stack** (consolidation, BSA bind/slack, DG/temp gatekeeping,
   fallback, through-hub) — these are MILP- or prefilter-isolated, never asserted as pipeline outcomes.
6. `cw_flex` / `flex_denominator` persistence is unbuilt (critique-12 F5) → end-to-end flexibility must be
   computed in-process for now.

---

## TIER 1 — Small/moderate end-to-end correctness test

**Goal:** one hand-verifiable instance traced through the whole pipeline, every step asserted against an
**oracle or bound**. New file `tests/test_e2e_pipeline.py`; fixtures in `tests/conftest.py`.

### The instance (`e2e_small`) — hand-built, ≤5 HAWBs, deterministic

Do **not** use random `generate_air_instance` draws (no value oracle). Reuse real `_gateways()`/`_hubs()`
topology from `tpeb_air_instance`, supply a fixed offer/HAWB set:

- **Offers (4):** `o1` BR `flat_rate` TPE→LAX; `o2` `coload_per_kg` TPE→LAX; `o3` CX `per_uld_pivot` HKG→LAX
  under a 1-ULD `per_flight` BSA (`cap_a` set); `o4` CI `min_flat_breaks` TPE→HKG + HKG→LAX segment (gives a
  through-HKG consolidation path).
- **HAWBs (4):** `k1`,`k2` GEN/ambient TPE→LAX (same group, dense+light → density mixing pays on a shared
  MAWB); `k3` HKG→LAX GEN onto the BSA; `k4` PER/chilled TPE→LAX (separate group, cannot consolidate with
  k1/k2). Deadlines loose → all real-route; `s=0`.

Touches simultaneously: flat bucket, co-load, BSA per_flight pivot+cap, min_flat_breaks, MAWB overlay (2
groups), hub dwell (HKG CFS-H), density mixing, prefilter cargo/temperature split.

### Step-by-step assertions with oracles

| # | Step | Function(s) | Invariant | Oracle / bound |
|---|------|------------|-----------|----------------|
| 1 | Generate | fixture | every HAWB ≥1 non-fallback arc | hand-enumerate `len(real_arcs(A_k))>=1` |
| 2 | Build graph | `build_air_graph` | k4 (PER) in a different MAWB group than k1/k2 (GEN) on same arc | `group_key(k1)!=group_key(k4)`; exactly 2 MAWB groups on TPE→LAX flat arc |
| 3 | Prefilter | `build_hawb_subgraph` | no dangling arcs; k3 BSA arc survives all 8 predicates | `rejections[k3,o3]` empty; every kept arc head ∈ backward-reachable set |
| 4 | A_k^min | `earliest_arrival` | finite; equals hand-walked earliest path | hand-compute ready+pickup+cfs+cartage → board earliest flight → ETA+dest cartage+cfs+customs+delivery |
| 5 | Solve | `air_milp.solve` | `OPTIMAL`; total == hand-summed; k1+k2 share one MAWB; k4 solo | cost oracle: flat `max(min_chg,m·CW)` w/ `CW=max((1+ε)Σw,Σv·167)` + BSA `r_a·max(CW_k3,pivot)` + ground + co-load; `==` to 1e-6 |
| 6 | **A_k^min ≡ arr_dest** | `earliest_arrival` vs C.10a | for the fastest-path HAWB, `A_k^min == arr_dest(k)` | equality 1e-6 — **D-A13 identity isolation can't catch** |
| 7 | **Re-walk solved route** | `sample_route(route, s=0)` | `end_to_end_arrival_h == arr_dest`; `on_time == (arr ≤ Δ_k)` | equality 1e-6; matches MILP `τ_k==0` |
| 8 | Persist | `scenario_io.persist` (+actuals) | tables non-empty; FK integrity (`PRAGMA foreign_keys=ON`) | row counts == in-memory object counts |
| 9 | **Reload → re-solve** | `load` → `solve` | `total_cost`/`routes`/`active_mawbs`/`fallback` bit-identical to step 5 | exact equality **and** == oracle |
| 10 | **Persisted actuals == walk** | `leg_actuals`/`component_actuals` | frozen `realized_block_h` re-walked reproduces step 7 | == `arr_dest` 1e-6 |
| 11 | Arrival stream | `generate_arrival_instance` (n≤5, days=2) | `0≤known_at≤ready_at≤effective_deadline_at<backstop`; `effective_deadline == A_k^min + sla_offset(tier)` | `derive_deadline` oracle; `known_at == cutoff(d*) − B` |
| 12 | Reveal view | `set_sim_clock`/`visible_shipments` | visible count at `t` == `#{known_at ≤ t}`; monotone in `t` | hand count |
| 13 | **Flexibility on real options** | enumerate admissible routes → `route_reliability` → `classify` | HAWB w/ ≥2 θ-separated non-dominated on-time real routes → `flex_k=True`; single-route → `False` | hand-derive from enumerated arrival_h set; `cw_flex` sums exactly those `cw_k` |

Steps **6, 7, 10, 13** are the point of Tier 1 — the identities 2c + scorer silently depend on, that no
isolation test exercises.

---

## TIER 2 — Medium use-case-coverage matrix

One minimal instance per use case (≤5 HAWBs each), solved through the full stack, asserted against a
bound/oracle. New file `tests/test_e2e_usecases.py`.

| # | Use case | Minimal instance | Expected | Oracle | e2e today? |
|---|----------|------------------|----------|--------|-----------|
| U1 | Pure direct routing | 1 HAWB, 1 direct flat | air arc not fallback | cost==flat bucket; `arr_dest`==hand-walk | No |
| U2 | Consolidation pays | 2 GEN (dense+light), 1 flat | shared MAWB | shared < sum of solos | Partial (MILP only) |
| U3 | Consolidation does NOT pay | 2 HAWBs, high MAWB-fix/both dense | 2 MAWBs (or per-kg) | joint ≥ separate | No |
| U4 | BSA binding | 2 HKG→LAX, BSA 1×LD3, both >1 ULD | one rides, one spills | C.5 binds; spill cost = spot/fallback | No |
| U5 | BSA slack | same, allotment 4×LD3 | both ride, no spill | `fallback==[]` | No |
| U6 | Each rate family billed | 4 single-HAWB, one/family | each bills closed form | flat/mfb/coload/bsa closed forms | Partial |
| U7 | DG/lithium gatekeeping | lithium HAWB, one leg `lithium_ok=False` | arc pruned (pred 4), reroute/fallback | pred-4 rejection present | No |
| U8 | Temp/cargo-class gatekeeping | PER/frozen, offer leg lacking cap | arc pruned (pred 2) | pred-2 rejection | No |
| U9 | Embargo gatekeeping | embargoed cargo on a leg | pred-3 prune | pred-3 rejection | No |
| U10 | Tight-deadline → fallback | deadline < earliest feasible arrival | routes fallback | `PrefilterWarning`; in `fallback_hawbs` | No |
| U11 | Through-routed via hub | PVG→ORD feasible only via HKG CFS-H | route incl. HKG dwell + 2 air arcs | time-ordered O→HKG→D; hub dwell present | No |
| U12 | Multi-day arrival contention | `build_tpeb_daily(D=2)`, BSA 1-ULD/day, 2 HAWBs diff days | each takes own day's slot | per-day allotment respected | No |
| U13 | Tier-mix flexibility | arrival: EXPRESS (1 tight) + DEFERRED (≥2 sep) | EXPRESS `flex_k=False`, DEFERRED `True` | hand-derive; `cw_flex`==DEFERRED mass | No |
| U14 | Empty book | n=0 | OPTIMAL $0; schedule still persists | `shipments==[]` | Yes — keep |
| U15 | Reload identity (hard) | full TPEB 12-HAWB | `load`→solve == in-memory | exact route/cost equality | Partial |

### Must-haves before the replay loop (priority 5–8)

1. **A_k^min ≡ arr_dest** (Tier-1 #6) — the one-time-scalar SoT; drift → every OTP and L2 wrong.
2. **Re-walk + persisted-actuals reproduce arr_dest** (Tier-1 #7,#10) — the scorer's walk == planner arrival.
3. **U4 BSA binding e2e** — the κ sweep rides on capacity binding; conservation identity never exercised.
4. **U10 tight-deadline fallback** — fallback accounting splits `L2_reshuffle` from `L2_fallback`.
5. **U2/U3 consolidation pays / doesn't** — L2 reshuffle value *is* re-consolidation; both directions.
6. **U13 + Tier-1 #13 flexibility on real options** — `cw_flex` is the headline denominator, only validated on
   synthetic options today.
7. **U15 / Tier-1 #9 reload identity with oracle** — the file is the replay source of truth.
8. **U6 rate-family billing from a generated+persisted instance** — billing self-checks live inside `solve`,
   never run on a reloaded-from-DB rate catalog; round-trip can silently corrupt a rate while the in-solver
   assert still passes on consistent-but-wrong data.

---

## Use cases the substrate/generator CANNOT yet express (need generator work first)

1. **BSA binding from the random generator** (U4/U5/U12) — blocked by critique-12 F1 (κ-as-integer-ULD) + F4
   (only 2/6 lanes capacitated, demand-starved). Hand-build fixtures until F1/F4 land.
2. **`cw_flex`/`flex_denominator` persisted e2e** (U13 via DB) — F5: nothing writes `flex_denominator`. Compute
   in-process until F5 wires it.
3. **Disruption/connection-slip recourse** — `s=0` for air; no cancellation column, no recourse arm. Deferred
   to 2c (3 fixtures). Out of scope for this plan.
4. **Tardiness e2e with a biting penalty** — `tardiness_weight_scale` defaults 0.0; needs a config override +
   hand-tightened deadline.
5. **Leg capability variation persisted** (`ac_type`/`lithium_ok`/`embargoed_cargo`) — `scenario_io` does NOT
   persist these, so U7–U9 survive in-memory build+solve but are **lost on persist→reload** (latent bug: a
   reloaded lithium-restricted instance changes the solved route vs in-memory, undetected). Assert U7–U9 on the
   in-memory `AirGraph`; to test through the DB round-trip, the schema needs those leg columns.

## Correctness issues noticed while reading (folded into critique-13)

- **[BUG] N12** `tpeb_air_instance.py:166-169` CX offer `uld_max_volume_cbm=8.0` vs contract `LD3 v=4.5` —
  pred-8 vs C.5b disagree; a 4.5<v≤8.0 HAWB passes prefilter then spills with no warning.
- **[smell] `air_generator.py:543`** cutoff fallback `else day*24` — dead for the real substrate (CX BSA offers
  carry cutoffs); would matter if a cutoff-less lane is added.
- **n4** `scenario_io._persist_hubs` writes `mct_h = dwell_h` — a 2c connection-check reading `mct_h` from the
  DB gets the dwell value, not a real MCT.
