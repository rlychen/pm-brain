# Critique 13 — Integration & Framework Review (7-agent, pre-2c consolidation pass)

**Session 34, 2026-06-11. Status: COMPLETE.** User-requested deep review before building the 2c
replay loop, motivated by "catch errors early + will everything fit together as the code grows."
Seven independent adversarial agents, each a tight lens:

1. **Code architecture & debuggability** — will it fit together / stay debuggable as 2c lands.
2. **Integration seams** — every module handoff where data can silently mismatch.
3. **Graph-generation logic** — incl. the service-level gatekeeping concentrated there.
4. **Model correctness & planner realism** — is the model right + how real planners interact.
5. **End-to-end simulation realism** — is the sim real enough to publish a number.
6. **Model numerics & magnitudes** — penalty/cost scaling, big-M, MIP-gap conditioning.
7. **Two-fold test-case design** — small e2e + medium use-case matrix (deliverable, not findings).

This round is **broader than 11 (design) and 12 (build)**: it cross-cuts code + model + sim + numerics +
integration at once. The agents converged on a small set of **new** issues the prior rounds could not see,
and independently re-confirmed the critique-12 F-set in code. The test plan is in
`docs/design/e2e_test_plan.md`.

---

## Headline

The **soundness core remains clean** (D-A9 tier-independent headline; D-A13 walk≡scalar — *empirically
verified by two agents this round*, 0 mismatches on real routes; no lookahead; no double-spend; CRN except
F8). The **layering, schema seam, and CRN/frozen-actuals factoring are well-built and were clearly designed
with 2c in mind** — the output tables already exist with correct keys.

The new findings cluster in three places the prior rounds missed:

1. **Numerical conditioning** — the live fixtures still feed `FALLBACK_COST = $1,000,000` to HiGHS, against
   the model's own instruction. The moment any HAWB takes the fallback, the **relative MIP gap stops meaning
   what we think**, and L2 = M₀−M₁ is a *difference* of two such objectives → the conditioning error is the
   same order as the savings signal. **New, sharp, cheap to fix, directly corrupts the proof.**
2. **Determinism contract is documented but unenforced** — `PYTHONHASHSEED=0` is asserted in comments and
   BUILD_STATUS but set *nowhere*; the byte-identity test runs in one process so it can't catch it. Across
   separate M₀/M₁ invocations this can read as **phantom L2**. **New, cheap.**
3. **What the number is allowed to claim** — three agents (model-realism, sim-realism) independently concluded
   the arrival-only L2 is a *bracketed component* (arrival-timing, transit-reliable), **not** "air replan
   savings" unqualified — because the thesis's own primary driver (disruption/readiness recovery) is zeroed,
   the human baseline H₀ is deliberately non-anticipatory (inflates L2), and a single `[CAL]` knob
   (DEFERRED slack) sets both the L2 mechanism and its reporting denominator.

Plus a handful of **latent correctness bugs** (harmless on today's single-gateway / finite-cutoff TPEB
instance, wrong the moment the substrate is generalized) and the already-queued critique-12 fold.

**Verdict:** good substrate, do not churn the core. But **fix the two cheap conditioning/determinism bugs and
the latent correctness set before 2c**, fold critique-12's F1/F2/F4, and **lock the claim-framing** before any
number leaves the building. None of this blocks 2c architecturally; it blocks *trusting 2c's output*.

---

## Convergence map (finding × agents who raised it)

| # | Finding | Sev | Agents | Status vs 11/12 |
|---|---|---|---|---|
| N1 | `FALLBACK_COST = $1M` wrecks relative MIP gap; corrupts L2 difference | BLOCKING | numerics (+ code-arch, sim touch) | **NEW** |
| N2 | `PYTHONHASHSEED=0` documented but never set; no cross-process determinism test | BLOCKING | code-arch | **NEW** |
| N3 | No state-owner object for sim-clock / capacity-ledger / RNG; 2c will bolt onto free functions | BLOCKING | code-arch | **NEW** |
| N4 | Arrival-only zeroes the thesis's *primary* driver (disruption/readiness); headline = calm-water component | BLOCKING | sim-realism (+ model-realism) | sharpens D-A15/F6 |
| N5 | DEFERRED 120h slack sets **both** L2 mechanism and `cw_flex` denominator (grades own homework) | BLOCKING | sim-realism | extends F6 |
| N6 | `π_hind` near-vacuous under arrival-only; promote `π_hind_locked` to peak-cell DoD | BLOCKING | sim-realism | extends D-A14 |
| N7 | `Δ^post` summed over whole subgraph dest-chain, not the chosen terminal tail (latent wrong-linkage) | MATERIAL | model-realism + graph-gen | **NEW** |
| N8 | `earliest_arrival` (A_k^min) computed over **unfiltered** graph (pred 6 only); can anchor Δ_k to an MILP-illegal route | MATERIAL | seams + graph-gen | **NEW** |
| N9 | Air-arc backward board-by uses `CO_a*` (→∞ when cutoff absent), not `STD_a`; can false-admit unreachable schedules | MATERIAL | graph-gen | **NEW** |
| N10 | Dispatch-feasibility check applied to **every** air arc, not just origin-POL; inert today, masks a hub-leg control | MATERIAL | graph-gen | **NEW** |
| N11 | Carrier **deny/blacklist** cascade enforced **nowhere** (allow-set only) | MATERIAL | graph-gen (+ model-realism) | **NEW** |
| N12 | ULD volume mismatch: CX offer `uld_max_volume_cbm=8.0` vs contract `LD3 v=4.5`; pred-8 vs C.5b disagree | MATERIAL (BUG) | test-design | **NEW** |
| N13 | H₀ rule-banned from anticipatory slot-holding = exactly the L2 edge; inflates L2 vs a real expert | MATERIAL | model-realism | **NEW** |
| N14 | Phase-locked cross-lane arrival waves (daily 24h tiling + tier-independent B) manufacture contention | MATERIAL | sim-realism | extends M-B9 |
| N15 | Demand too thin (~0.7 HAWB/day/lane) + 7-day toy horizon truncates DEFERRED window; scale-up should *gate* not follow | MATERIAL | sim-realism | extends F4/M-B7 |
| N16 | Replan friction under-modeled (M₁ reshuffles for ε); add an operator-tolerance hurdle | MATERIAL | sim-realism | extends C5 |
| N17 | Correctness asserts (`_validate_billing`/`_validate_bsa`) stripped by `python -O` | MATERIAL | code-arch | **NEW** |
| N18 | Duplicated, already-drifting HAWB-draw logic across `_gen_hawbs` / `_gen_arrivals` | MATERIAL | code-arch | **NEW** |
| F1 | κ is integer-quantized ULD count (retired); no reachable binding/abundant cell | BLOCKING | code-arch (M4) | **confirmed** (=F1) |
| F2 | Cutoff anchor broken: degenerate cutoffs + 27% of HAWBs clamp to `known_at=0` + wrong cutoff for through lanes | BLOCKING | seams (quantified) | **confirmed** (=F2) |
| F5 | Persistence drops `target_offer_id`/`t_dead_at`; t=0 `cw_flex` unreconstructable from db | MATERIAL | seams + code-arch | **confirmed** (=F5) |
| F8 | `t_dead_prob` conditional draw desyncs CRN | MINOR | seams + code-arch | **confirmed** (=F8) |
| n1 | flat-bucket aux `c` unbounded above (~1e30 in matrix); bound it | MINOR | numerics | **NEW** |
| n2 | `deadline_abs_h`/`soft_deadline_h` default `1e9` sentinel; assert finite at MILP build | MINOR | numerics | **NEW** |
| n3 | Tier mix 20/40/40 duplicated literal (`DEFAULT_TIER_MIX` vs `ArrivalConfig.tier_mix`) | MINOR | seams | **NEW** |
| n4 | `mct_h = dwell_h` placeholder; 2c connection-check will read dwell as MCT | MINOR | code-arch + seams + test | **NEW** |
| n5 | Leg capability cols (`ac_type`/`lithium_ok`/`embargoed_cargo`) not persisted → U7–U9 lost on round-trip | MINOR (latent) | test-design | **NEW** |
| n6 | `scenario_db` documents a UTC→sim epoch subtraction that `persist` doesn't do | MINOR (latent) | seams | **NEW** |

---

## §A — New BLOCKING

### N1 — `FALLBACK_COST = $1,000,000` is a live conditioning defect (numerics)
`tpeb_air_instance.py:55` hardcodes `FALLBACK_COST = 1_000_000.0`, threaded through the generator. The model
already forbade this: `air_freight_routing.tex:1043-1068` specifies `C^fallback = max_k W_k(T^abs−Δ_k)² +
10·max real_cost(k,a)` (≈ 2× worst feasible route ≈ **$50k–$150k** for these fixtures) and explicitly says
"the prior $1M placeholder was unnecessarily large." **The refinement was never adopted in code.**
Measured consequences: objective coefficient spread ~**1e5–1e6** (legit band is [3, ~6000]); the instant one
HAWB takes the fallback, the incumbent jumps ~$1M, so HiGHS's default `mip_rel_gap=1e-4` permits ~**$100
absolute slack** — which swamps real decisions worth $50–$300 (co-load vs break, $50 MAWB fix, $160 cartage).
**"OPTIMAL" can hide a commercially-material error, and L2 is a difference of two such objectives.**
**Fix (#1 action):** compute `C^fallback` per the tex formula at instance build; collapses spread to ~1e2–1e3.

### N2 — Determinism contract documented but unenforced (code-arch)
`air_milp.py:256-259` + BUILD_STATUS state column order is deterministic "because the harness sets
`PYTHONHASHSEED=0`." There is no harness: `pyproject.toml` has no `env`, `conftest.py` is empty, the only
occurrence is inside two manual command strings in `.claude/settings.local.json`. The byte-identity test
(`test_generator_to_files.py:87`) builds both DBs **in one process** → structurally cannot catch hashseed
nondeterminism. Across separate M₀/M₁ invocations (or CI with a random seed), two equally-optimal incumbents
can diverge → **phantom L2**.
**Fix:** set `PYTHONHASHSEED=0` in one enforced place (`[tool.pytest.ini_options] env` via `pytest-env`, or a
`conftest.py` re-exec guard); have the 2c entrypoint assert it at startup; add a **subprocess** byte-identity
test.

### N3 — No state-owner for the sim-clock / capacity-ledger / RNG triad (code-arch)
The clock is a DB row mutated by a free function (`set_sim_clock`); `visible_shipments` has **two divergent
code paths** (global view when `t is None`, parameterized query otherwise — a determinism footgun across
sequential arms); the `capacity_ledger` table exists as DDL with **zero writers**. Nothing owns
"(sim_clock, capacity ledger, RNG sub-streams, arm identity)" as a coherently-advancing unit. Building 2c on
these scattered mutators forces a god-function or hidden global state — *exactly* the "harder to debug as it
grows" failure.
**Fix (before writing 2c):** introduce one `ReplayState`/orchestrator object that owns the clock, the per-arm
ledger, and the RNG streams and is the only thing allowed to advance them; collapse `visible_shipments` to the
parameterized form.

### N4 — The headline excludes the thesis's primary driver (sim-realism, model-realism concurs)
`product_thesis.md:48-52` makes the transit-time sensor the thing that *triggers* replan ("that synergy is the
wedge's value"); the methodology + critique-11 both concede disruption-recovery is "the larger real driver."
Yet the headline L2 runs on **zero transit variance + zero disruptions** — the calm-water component of a
storm-reaction thesis. D-A15's "conservative lower bound" is honest but a bound that *excludes the primary
driver* may be a tiny, non-additive fraction of the real number — nobody has bounded the ratio.
**Fix:** run the §6 disruption-recovery sensitivity arm at the peak cell **once** before publishing; report
arrival-timing L2 as a *named component* beside it; reframe the headline as "**arrival-timing replan value
(transit-reliable lower bound)**," never "air replan savings" unqualified. Cheapest of the realism blockers —
the recourse code is already a required 2c capability.

### N5 — One `[CAL]` knob sets both the L2 mechanism and its denominator (sim-realism)
DEFERRED `sla_offset_h=120` simultaneously (a) creates the reshuffle headroom that *is* L2 (M₁ bumps
DEFERRED→d*+1), (b) drives `cw_flex` (the per-flexible-kg denominator), (c) is 40% of the mix. Halving it moves
numerator and denominator together → the band-method "robustness" can be an artifact of correlated motion.
**Fix:** in the sandbag sweep, decompose the DEFERRED-slack sensitivity into numerator (reshuffle headroom) vs
denominator (`cw_flex` mass) effects; report **`L2%` (denominator-independent) as the primary robustness
curve**; verify `L2%` survives slack-halving independently. If it collapses at realistic slack, F6 becomes
headline-gating, not deferred.

### N6 — `π_hind` regret floor is near-vacuous under arrival-only (sim-realism)
With transit deterministic, `π_hind` knows the *only* stochastic primitive, so `C(M₁) − C_hind` collapses to
the value of sequential-commitment structure alone — and nothing in the DoD checks the floor is *non-trivial*.
A broken M₁ that's near-optimal-by-slackness passes. The arm that would split recoverable suboptimality from
the irreducible commitment gap (`π_hind_locked`, M-B6) is deferred.
**Fix:** promote `π_hind_locked` into the peak-cell DoD; report `C(M₁)−C(π_hind_locked)` (recoverable) vs
`C(π_hind_locked)−C(π_hind)` (irreducible) separately; assert the floor is informative at the peak cell.

---

## §B — New MATERIAL (latent correctness; harmless today, wrong on generalization)

- **N7 `Δ^post` subgraph-wide sum** (`air_milp.py:701-703`): sums dwell over *all* `_DEST_CHAIN` arcs in the
  subgraph and adds it to *every* terminal air arc's `arr_dest`. Correct only for single-POD HAWBs (today's
  case). Two candidate PODs / alternate CFS → `arr_dest` overstated on every terminal arc. **Fix:** compute
  `Δ^post` per terminal arc along its actual dest tail, or assert single-tail at build.
- **N8 `earliest_arrival` over the unfiltered graph** (`air_graph.py:1095`): `_propagate_forward` gates only on
  cutoff+dispatch (pred 6), not pred 2–5/8 — so `Δ_k = A_k^min + sla_offset` can anchor to a route the MILP
  can't legally select. Inert now (weights < ULD cap; all flags permissive); bites with heavier weights or a
  tighter ULD cap. **Fix:** compute `A_k^min` over the admitted subgraph (or document the gap loudly).
- **N9 air-arc backward board-by = `CO_a*` not `STD_a`** (`air_graph.py:959`): when an offer has no cutoff,
  `_co_eff→inf`, so `latest[tail]=+inf` ("be present arbitrarily late and still catch the flight") → can
  false-admit unreachable schedules. Masked because every TPEB offer carries a finite cutoff, but
  `cutoff_utc_h` is `Optional` by design. **Fix:** `latest[tail] = min(CO_a*, STD_a)`.
- **N10 dispatch check over-applied** (`air_graph.py:914,1011`): the origin-dispatch-lead predicate fires on
  *every* air arc, using origin `ready_early`/`λ_disp` against that arc's cutoff. On a hub-outbound leg it's a
  near-tautology (can't reject) — so it gate-keeps the wrong control downstream and the real hub-retender lead
  is enforced nowhere. **Fix:** gate on `arc.tail == airport_out(origin_gateway)` (origin POL only).
- **N11 carrier deny/blacklist enforced nowhere** (`air_graph.py:799-813`): the model's four-layer cascade
  (intersect-allows / union-denies, any deny wins) is collapsed to a single allow-set; there is no deny-layer
  or blacklist input. Acknowledged slice-4 seam, but "explicit deny overrides allow" service semantics are
  currently gate-kept nowhere — the highest-leverage missing service-level control.
- **N12 ULD volume mismatch (BUG)** (`tpeb_air_instance.py:166-169`): CX `per_uld_pivot` offers declare
  `uld_max_volume_cbm=8.0`; the BSA's `UldType("LD3")` is `v_cbm=4.5`. Pred-8 screens against 8.0, C.5b binds
  against 4.5 → a 4.5<v≤8.0 HAWB passes prefilter then spills in the MILP with no warning. Verify intent before
  any BSA-binding e2e test.
- **N17 asserts stripped by `-O`**: convert the load-bearing `_validate_billing`/`_validate_bsa` reconciliation
  asserts to explicit `raise` (keep cheap internal asserts).
- **N18 duplicated HAWB-draw logic** (`air_generator.py:209-218` vs `557-561`): copy-paste that has already
  diverged (density divide, deadline derivation) and shares the `"demand"` RNG stream → a future edit to one
  silently shifts the draw sequence. **Fix:** extract one `_draw_cargo_profile(rng)`; delete Part A if dead.

---

## §C — Realism / validity (what the number is allowed to claim)

These do **not** break L2's internal validity; they constrain the claim and should land in the methodology +
calibration note (not all are code).

- **N13 H₀ is non-anticipatory by rule** — real senior planners hold scarce space on gut feel; banning it
  attributes 100% of slot-holding value to the optimizer → inflates L2. **Fix:** add a "Diligent+" H₀ variant
  with one crude tier-reservation heuristic, OR pre-register L2 as measured against a deliberately
  non-anticipatory human (an upper bracket on the holding-value component).
- **Cargo-readiness variance zeroed** (model-realism, = N4's twin) — the planner's dominant replan driver is
  readiness slippage; keep the readiness caveat load-bearing in every framing.
- **N14 phase-locked arrival waves** — add inter-lane phase jitter; report whether L2 survives
  de-synchronization; list the wave structure as L2-*inflating* alongside fixed-N as L2-*deflating*.
- **N15 thin demand / toy horizon** — bring the forwarder-scale instance forward to *gate* the headline;
  report L2 with the count of distinct reshuffle events behind it (if < ~5, it's anecdote); exclude
  warm-up/cool-down days so DEFERRED windows don't run off the 7-day edge.
- **N16 reshuffle friction** — add an operator-tolerance hurdle (min net-saving below which M₁ won't bump);
  report L2 at hurdle=0 vs realistic; the gap is the "operational friction discount." Ties to the
  override-rate-is-the-KPI principle.
- **m1 tier mix 20/40/40** unsourced; EXPRESS-light is the conservative direction — sweep it or state as the
  conservative anchor.

---

## §D — Confirmed SOUND (do not churn)

- **Layering** graph←milp←io←generator is clean, one-directional; 2c sits on top without inverting it.
- **Schema seam** is a real seam (4-function Postgres surface); output tables (`runs`/`routes`/
  `capacity_ledger`/`booking_promise`/`flex_denominator`) already keyed for 2c; conservation `CHECK` + partial
  unique index push invariants into the DB.
- **CRN / frozen-actuals factoring** — `sample_leg_block`/`sample_component_delta` are the single shared
  primitive; named RNG sub-streams with a whitelist; κ(rates) isolated from demand.
- **D-A13 walk≡scalar** — *empirically verified* this round (0 mismatches), because `delta_post` sums unique
  per-HAWB dest-chain arcs and the terminal ETA resets the clock identically in graph and MILP.
- **Graph gatekeeping core** — strict 1→8 predicate cascade, inclusive `≤` cutoff, air-arc head ETA-reset,
  CFS in/out splits (no self-loop), hub P5/P6 branching + through-arc R7 exclusion, fallback-always-present.
  **Predicate 9 genuinely retired in code** (no `z_tier`/`σ̂`); only the *tex* is stale on it.
- **MILP economics** — C.4 density mixing (consolidation pays, monotonicity-asserted), C.13 pivot/equalized
  settlement, C.10 relative-α-grid PWL (no extrapolation past fallback; penalty excluded from realized cost so
  PWL slack can't contaminate L2). Objective attribution charges freight once per MAWB, no double-count.
- **Numerics discipline** — `M^BW` correctly tight (BUG-1 widening did *not* over-widen); every cost-bearing
  aux bounded to a real physical quantity; PWL inert by default; relative-tolerance billing checks.

---

## §E — Proposed consolidated sequencing (for user decision)

Grouped by risk/cost. Ordering is a recommendation, not a commitment.

**Wave 0 — cheap correctness/conditioning (low risk, high value; do first):**
- N1 compute `C^fallback` (~2× worst route) — **the single highest-value fix**.
- N2 enforce `PYTHONHASHSEED=0` + subprocess determinism test.
- F8 always-consume the `t_dead` uniform (CRN).
- N17 billing asserts → raises; n1 bound aux `c`; n2 finite-deadline guard.
- N7 `Δ^post` per-terminal-arc (or assert single-tail); N18 dedupe `_draw_cargo_profile`.

**Wave 1 — graph-gen service-level correctness (latent, real):**
- N9 board-by `min(CO*, STD_a)`; N10 dispatch gated to origin POL; N8 `A_k^min` over admitted subgraph.
- N12 resolve the ULD-volume mismatch; n5 persist leg-capability columns (or assert constancy).
- N11 carrier deny-layer — bigger; can stay tracked-deferred but is the top service-level hole.

**Wave 2 — the critique-12 fold (already queued, directional):** F1 continuous-κ → F4 capacitate an
origin-diverse lane + `lane_mix` → F2 cutoff derivation + binding-leg anchor → F3 → D-A17. **(Confirm before
implementing — changes how the load-bearing instance is built.)**

**Wave 3 — architecture before 2c:** N3 `ReplayState` owner + collapse `visible_shipments`; n4 source `mct_h`.

**Wave 4 — claim-framing / methodology folds (mostly text + one sensitivity run):** N4 disruption sensitivity
+ reframe headline as a named component; N5 `L2%`-primary + decomposed slack sensitivity; N6 promote
`π_hind_locked`; N13 Diligent+ H₀; N14 phase-jitter; N15 scale-gates-headline; N16 reshuffle hurdle.

**Wave 5 — test build-out:** Tier-1 e2e pipeline test (the cross-component identities) + Tier-2 use-case
matrix. See `docs/design/e2e_test_plan.md`. Several use cases need Wave-1/2 first (BSA binding needs F1/F4;
`cw_flex` persistence needs F5).

---

## Pointers

- Per-agent full reports are in this session's transcript (7 agents). Agent IDs retained for follow-up.
- Test plan: `docs/design/e2e_test_plan.md`.
- Prior rounds: `docs/critique/11` (design, D-A9..D-A16), `docs/critique/12` (build, F1..F8 + numeric
  walkthroughs).
