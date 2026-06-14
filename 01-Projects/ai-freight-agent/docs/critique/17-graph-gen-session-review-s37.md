# Critique round — graph-gen workstream review (Session 37)

**Run:** 2026-06-14 (S37), 4 parallel agents (general-purpose), on the full S37 graph-gen workstream:
FreightNet → geo candidate selection → build-time fallback redesign → candidate-path retirement → D-F6 v2
pre-committed SLA deadlines → B5 re-measure. **To review by user (RC, tomorrow).** Nothing was fixed in response
yet — this is the recorded feedback.

**Lenses:** (1) correctness/logic of the algorithmic core; (2) methodology soundness (D-F6 v2 + fallback + proof
claims); (3) integration/persistence seam (candidate retirement); (4) adversarial red-team + calibration (B5 claim).

**Suite state at review:** 255 passed, ruff clean. (The findings below are NOT caught by the suite.)

---

## 🔴 BLOCKING

### BLK-1 — B5 is NOT robustly resolved; the "21.2s OPTIMAL" headline was a lucky seed. (red-team)
Re-measured the proof cell **n=15 / days=7 / κ=1 / α=1**, HiGHS `threads=1`, `random_seed=0`, no HiGHS time limit
(so every OPTIMAL is genuine solve-to-optimality, not a truncated incumbent):

| seed | solve time | status |
|---|---|---|
| 0 | 23.3s | OPTIMAL (reproduces the claim) |
| 1 | **131.8s** | OPTIMAL (6× the headline) |
| 2 | **>5 min, did not finish** (killed ~8 min cputime) | — |

- The "21.2s RESOLVED" in BUILD_STATUS is the **best of the seed distribution** presented as the result.
- **Mechanism doesn't generalize (BLK-1b):** the corridor arc-cut is real (measured air-arcs/subgraph 38.3 / 35.9 /
  41.4 for seeds 0/1/2 — the "103→~37" claim holds), BUT seed 1 has the *fewest* arcs and the *slowest* solve.
  Solve time is driven by **consolidation/capacity branching difficulty**, which arc-pruning does not touch. The lever
  the "resolution" rests on is **uncorrelated** with the metric that blocks the sweep.
- **Sweep budget (BLK-1c):** the headline measures ONE solve. The replay proof is M0 + M1 open-book re-solve per
  replay cycle (~3/day × 7 ≈ 21 cycles) × arms × the (κ,α,λ) grid. At a mean ~25–50s and a tail to ∞, one cell is
  minutes-to-hours; the grid is intractable. `arrival_only_replan_methodology.md:248` itself flags a tractability
  re-check is owed before the sweep is trusted. **B5 was never the *sweep* blocker — and even per-solve it isn't cleared.**
- **No safety valve:** `air_milp.py:364` sets no HiGHS time limit, so a hard seed hangs unbounded.
- **Recommendation:** keep B5 **OPEN**. The corridor is a correct, useful *LP-size* reduction; it is not a
  *tractability* guarantee. Real fixes are deferred and now load-bearing for the proof's feasibility (not just speed):
  HiGHS time-limit + incumbent strategy, MIP-gap/warm-start/heuristic tuning, or accept n<15 / tighter φ for the sweep.

### BLK-2 — `air_leg_cost_ub` under-bounds BSA `per_flight` cost → fallback can be under-priced → can strand a routable HAWB. (correctness)
`src/components/air_milp.py:253`: `n_uld_solo = max(1, ceil(cw / 1000))` counts ULDs by **weight only**. The realized
MILP billing it must upper-bound (`_build_c5b_uld` ~683-684, `_build_c13b_pivot` ~709-710) constrains ULD count `η`
by **both** weight and **volume**, and bills `pivot ≥ π·Ση`. For a low-density (volume-bound) HAWB the ULD count is
volume-driven and exceeds `n_uld_solo`, so the "upper bound" falls **below** the realized per-flight cost.

- **Verified counterexample:** HAWB 500 kg / 20 cbm; ULD V_u=3, W_u=2000; BSA per_flight π=500, r_a=2.0 →
  `air_leg_cost_ub` pivot term = 2·max(3340, 500·4)=**6680** vs realized = 2·max(3340, 500·⌈20/3⌉=7)=**7000**.
- **Bites real instances:** the generator's cargo (density band 120–240, LD3 V=4.5) is **always volume-bound**
  (4.5·density < 1500), so `ceil(cw/1000)` (weight-driven) can be < `ceil(vol/4.5)` (volume-driven). Since the
  fallback = `margin × longest-UB-path` (`air_graph.py:853`), the dominance guarantee breaks whenever a BSA
  per_flight arc is the binding expensive route — the exact `feedback_no_standalone_cost_pruning` failure mode.
- **Other families are valid upper bounds** (flat, coload, min_flat_breaks all checked OK); only BSA per_flight is wrong.
- **Fix:** make `n_uld_solo` volume-aware — `max(ceil(weight/min W_u), ceil(volume/min V_u))` over the contract's
  admissible ULD types. Add the fallback-dominance invariant test (see MAT-2).

---

## 🟠 MATERIAL

### MAT-1 — BUILD_STATUS.md self-contradicts + code uncommitted. (red-team)
`BUILD_STATUS.md:29` "The B5 blocker — RESOLVED ✅ (S37)" with timings, but `:21` lists building FreightNet + geo
graph-gen as "▶ Next (S37)" (not done) and `:79-80,:89-90` mark FreightNet + Geographic graph-gen as ☐ not-started.
Meanwhile `git status`: HEAD is still S36 (`7ac9015`); `src/freightnet.py`, `src/components/geo_select.py` untracked —
the **whole session is uncommitted**. A reader trusting the tables thinks B5 is unaddressed; one trusting the banner
thinks it's done+committed. Neither is true. **Fix:** full BUILD_STATUS rewrite (banner/tables/commit-state agree) +
commit the session. (Being addressed at this sign-off.)

### MAT-2 — Fallback dominance is sound on dollars but has no invariant test; proof-neutrality oversold for W_k>0. (methodology)
- **Dollar + tardiness dominance is actually sound** (fallback is worse on *both* axes than any feasible real route:
  dollars via the UB path, and arrival = T^abs → max tardiness span) — EXCEPT where BLK-2 breaks the dollar UB.
  Recommend an explicit invariant test: for every routable HAWB on a small instance,
  `cost(fallback) > cost(any real route)` AND `τ(fallback) ≥ τ(any real route)`. Currently asserted only in prose.
- **Proof-neutrality (claim 1) is WRONG for W_k>0.** Proposal §6 says Δ_k cancels in L2 "only second-order."
  But `L2_tard(k) = W_k·[max(0,t_k^M0−Δ_k)² − max(0,t_k^M1−Δ_k)²]` — Δ_k enters the tardiness **nonlinearity** and the
  two arms sit at **different t_k**, so the difference does **not** cancel and the shift is **first-order**, not second.
  Headline (W_k=0) is genuinely neutral (pen=0 both arms) — that stands. **Fix:** retract "second-order"; state that
  tardiness-weighted L2 under v2 is a *different metric* (the promise changed), not a perturbation of the v1 number.

### MAT-3 — Born-at-risk p90 guard is necessary-not-sufficient. (methodology)
The proposal claims the p90 `base_transit` calibration makes baseline tardiness congestion-independent. It doesn't:
(a) ~10% still born-late, and a born-late HAWB with *multiple* (all-late) feasible routes can be reshuffled by M1 →
its tardiness differs across arms → enters L2 (congestion-coupled born-at-risk, which the guard claims to remove);
(b) p90 is **lane-level** but slack is **door-level** (drayage via geo_select), so the guard is calibrated on the
wrong granularity. **Fix:** report the born-at-risk fraction (`slack_k<0` count) as a first-class per-run diagnostic;
either exclude their tardiness from L2 or document the congestion-coupled component.

### MAT-4 — `t_dead_offset` floor (48h) below physical min transit (84/96h) → phantom tardiness when the knob is on. (red-team)
`air_generator.py:828` `t_dead = ready + uniform(48,120)` vs `_base_transit_h` 84/96. With `t_dead_prob=1.0`,
33–67% of HAWBs (measured 5–10 of 15 across seeds) get Δ_k < fastest base transit → structurally late on *every* real
route. **Default `t_dead_prob=0` keeps the headline clean**, but the knob is advertised for within-tier slack (D-F6),
and turning it on injects calibration-artifact tardiness that corrupts OTP / tardiness-L2. **Fix:** floor
`t_dead_offset` above max base transit (≥96h).

### MAT-5 — Empty geographic seed set silently reverts to the pre-baked gateway. (correctness)
`resolve_geo_candidates` (`air_graph.py:~1578`): if `select_subgraph` returns no seeds (door has no flight-airport
within `max_radius_km` ∩ allowed), `_drayage_candidates` returns `()`, and `build_physical_graph:741` treats empty
candidates as falsy → reverts to the legacy single `origin_gateway`. `select_subgraph` logs `geo_select.disconnected`
but `resolve_geo_candidates` does not; the revert is invisible to the caller. Inconsistent with the fail-fast posture
elsewhere (missing-FreightNet-airport raises). **Fix:** surface the empty-seed case explicitly (decide intent — raise
or log loudly), don't coalesce into back-compat.

---

## 🟡 MINOR

- **MIN-1 (methodology, red-team):** corridor **φ=1.3 is load-bearing, not a free tractability knob.** It admits the
  PVG→HKG→LAX consolidation at ratio 1.216–1.258 — *under 1.3 but barely*. Dropping to φ=1.2 (advertised as a
  tractability lever in `geo_select.py:34-36`) would prune HKG consolidation for PVG-region doors and strand the exact
  route the fixture tests. Same for `max_air_legs=3` (a 4-leg-only feasible route → fallback). **Recommend a φ
  sensitivity sweep:** if L2 moves with φ, the corridor IS pruning load-bearing routes. At the current bbox no
  shipment is stranded (`n_empty_subgraph=0`, `nfb_routed=0` across seeds 0–2), so it's a thin-margin warning, not a
  live break.
- **MIN-2 (methodology, seam, red-team):** **documentation drift.** `flexibility_model.md` §0/§3/§6 still show the v1
  `Δ_k = ready + min_transit_k + sla_offset` (contradicts the adopted v2 in §1/§2.1). `air_generator.py` docstrings
  (module §, ~629, ~788-793) still describe Δ_k as `A_k^min + sla_offset` and say pass-2 "reads earliest_arrival" (it
  uses `committed_deadline` + a table now; the pass-2 graph rebuild is for the `latest_ready` clamp, not A_k^min).
  Module docstring line 4 still lists `fallback_cost` as a generator output. `tpeb_air_instance.py:55-56` references
  the retired `compute_fallback_cost`. **Fix:** scrub to v2.
- **MIN-3 (methodology):** `committed_deadline` does not assert `T^abs > Δ_k` at construction — the invariant the
  proposal §4 relies on. The only guard is in `air_milp.solve` and fires only when `W_k>0`; a misconfigured
  `base_transit > backstop_buffer` would silently give `span=0` (the "free pass" the proposal says it rejected),
  arriving via misconfig instead of the `max()` hack. **Fix:** assert `Δ_k < T^abs` at construction/generation.
- **MIN-4 (correctness):** `extend_until_reachable` (`freightnet.py:401-406`) doesn't validate `start_km ≤ max_km`;
  a config with `seed_radius_km > max_radius_km` searches *wider* than the stated cap on the first iteration.
  `GeoSelectConfig.__post_init__` validates the others but not this ordering. Add a guard.
- **MIN-5 (correctness):** `geo_select` docstring says "every airport and flight on any ≤H_max-leg corridor path" but
  `air_pairs` is built from offer endpoints `{(o.origin,o.dest)}`, so through-offer intermediate stops are invisible to
  corridor/frontier selection and `max_air_legs` counts **offer-arcs, not physical legs**. Correct-by-design
  (consistent with emission + MILP) but the wording conflates "leg" with "offer-arc." Tighten.
- **MIN-6 (red-team):** the "before" baseline (region path, "103 arcs, >5min") is **no longer reproducible from HEAD**
  — the old `_draw_region_routing`/`_ORIGIN_REGION` is removed. Attestable only from commit `8e388e2`. The before/after
  delta is half-auditable.
- **MIN-7 (seam):** `test_roundtrip_reconstructs_objects:84` spot-checks only 2 of 4 door coords (the solve-equality
  tests cover the rest, so not a real gap).

---

## ✅ Confirmed SOUND (checked, no action)

- **geo_select is genuinely NOT a disguised dominance prune** (methodology claim 4 — the most important). It is
  exhaustive-within-corridor: keeps **every** leg on any ≤H_max corridor path (`f[u]+1+b[v] ≤ max_air_legs`), not just
  one connecting path; `connected` is a feasibility flag only, never a termination. Directly honors
  `feedback_no_standalone_cost_pruning`. Pruning axes are geographic (φ) + hop-budget (H_max), not cost.
- **Bidirectional frontier retention is PROVEN sound** (correctness): any ≤H_max path through leg (A,B) satisfies
  `f(A)+1+b(B) ≤ L ≤ H_max` because f/b are global min-hop depths → no valid leg dropped. Over-inclusion possible but
  safe (option-richness).
- **Corridor ellipse gate correct** — admits HKG-farther-than-LAX consolidation at φ=1.3 (1.14 < 1.3); seeds always
  force-added; zero-distance degenerate-safe.
- **`_max_path_cost`** — correct DAG longest-cost path, memoized DFS, cycle guard sound (back-edge via on_stack),
  deterministic (sorted adjacency).
- **Determinism** — no hash-order leak found anywhere in the new code; every ordering point is sorted; cross-process
  (`test_determinism.py`) green.
- **Integration/persistence seam — clean, no BLOCKING/MATERIAL** (seam agent): doors round-trip **float-exact**;
  loaded scenario rebuilds+solves bit-identically; `_default_fn()` cache safe (read-only FreightNet, invocation-order
  independent, builds-if-missing from the committed CSV); CRN preserved (door draw = 4 fixed uniforms on the demand
  stream; κ/α/λ axes don't desync); back-compat scalar `fallback_cost` API cleanly preserved (XOR guard at
  `air_graph.py:1234`, both/neither tested); no dead code / orphaned columns.
- **`T^abs > Δ_k` invariant holds** across 48 sampled cells (κ∈{1,2}, α∈{1,0.1}, t_dead∈{0,1}, λ∈{0,0.5}, seeds 0–2)
  at default `t_dead_prob=0`; the 12/24/48 `sla_offset_h` ordering passes `validate_tier_specs()`. (Only failure path
  is MAT-4, the t_dead floor.)
- **`_non_dominated`** keeps a same-price-later option (strict-both-axes domination only) — correctly preserves
  "bump to a later same-price flight to free a scarce slot" as a reshuffle target.

---

## Suggested triage order (for tomorrow)

1. **BLK-1 (B5 not resolved):** decide the tractability strategy (time-limit + incumbent / solver tuning / smaller n /
   tighter φ). Reopen B5. **Biggest item.**
2. **BLK-2 (BSA UB bug):** volume-aware `n_uld_solo` + the dominance invariant test (MAT-2). Contained correctness fix.
3. **MAT-4 / MIN-3 (cheap calibration safety):** t_dead floor; `Δ_k < T^abs` construction assert; MIN-4 guard;
   MAT-5 loud empty-seed.
4. **MAT-2/MAT-3 doc + MIN-2 drift:** retract "second-order"; report born-at-risk fraction; scrub v1 formula drift.
5. **MIN-1 (φ sensitivity sweep):** confirm the corridor isn't pruning load-bearing routes (defensible, do alongside
   the B5 tractability work since both touch φ).
