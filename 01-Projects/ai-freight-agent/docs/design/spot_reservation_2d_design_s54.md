# Design spec — 2D spot reservation (D-A1 replacement), v1 — REV 3

**Status:** APPROVED for build (user, S54). REV 3 folds the REV-2 re-verification round (3 agents).
Replaces locked D-A1 (single `tender`).

## REV 3 changelog (folded corrections — build follows these)
- **NC-a (monotone double-charge on service reroutes) — DISMISSED by user** as a domain non-issue: a
  cancelled flight voids the commitment (no sunk cost), and air delays are never large enough to strand
  a held reservation while still on the hook. ⇒ **`penalty_frac=1` STANDS** (sunk-cost, no-cancel).
- **`CW^r` weight term carries `(1+ε)`** to match C.4 billing (NC-b/RQ3): `CW^r = max((1+ε)·r^w, 167·r^v)`.
- **Reservation offset is PER-GROUP** for multi-group flat/MFB spot arcs (NC-d): `Σ_g family_cost(CW_g)`
  ≠ `family_cost(Σ_g CW_g)`; define the sunk offset per `(a,g)`, or restrict spot MAWB arcs to one group.
- **`family_cost(0) := 0`** convention (NC-e) — flat arcs' `min_chg` must vanish at zero, or the
  reservation-off control is off by `min_chg`.
- **h0_planner reads the BINDING dimension** `min(C^w, C^v·167)` (not `max`) — the MILP binds on volume;
  `max` would over-pack H0 → inflate H0 fallback / distort L1.
- **Gap-robustness (RQ7):** add a lexicographic **min-reservation tie-break** (secondary objective
  minimizing `Σ_a family_cost(r_a)`) to pin the incumbent's (w,v) split; keep the sweep's significance
  gate `|L2| ≫ gap·C̄`. L2 stays a difference of two 0.5%-gap solves — not a new problem, but now carries
  the committed-cost term (ties to `project_tractability_gap_policy`).
- **Cutoff pin is WHOLE-ROUTE atomic** (§5.6) — at `τ0_a` pin each rider's entire current route (matching
  `mark_tendered`), not the single arc, or a HAWB can be half-committed with no feasible completion.
- **Post-cutoff floor-drop (NC-f)** documented: if `r > Σpins`, the floor drops at `τ0_a` (reserved-but-
  unpinned space effectively released post-cutoff) — intended (no booking post-cutoff).
- **§11 methodology:** chain `guaranteed → empirical` row added (D-A11/D-A23 override); D-A12/D-A10 row
  (re-eval reshuffle/fallback split + negative-control corner at the recalibrated point); recalibration
  names the S49 τ-ladder→regime bands + density floor (C.10 tardiness weights NOT recalibrated); the
  "symmetric across arms" wording corrected — committed spot cost is **arm-varying** (contributes to L2),
  unlike delta-invariant hard-BSA (cancels).
- **Change surface:** `air_milp.py:404` `_build_spot_cap` call-site ripple listed; `scenario_db.py` DDL
  intentionally UNCHANGED (2D collapses to a single CW-equivalent audit line).

**REV 2** folded the 3 user decisions + the first 4-agent validation round. Grounding: `air_cargo_spot_families_grounding_s54.md`,
`air_cargo_reserve_assign_grounding_s52.md`. Memories `reference_air_cargo_spot_family_decay_reserve`,
`project_s52_reserve_assign_synthesis`. Governing methodology: `model/arrival_only_replan_methodology.md`.

⛔ Tardiness penalty ON in every run/test (`tardiness_weight_scale=1`). No exceptions.

---

## 0. What changed in REV 2 (read first)
- **Cost model REWRITTEN (§5.7).** The v1 "friction penalty scored at cutoff" was BROKEN — it was an
  M1-only additive cost that could flip L2 negative (OR agent VQ2; concrete −602·m case). Replaced by the
  **sunk-cost / free-reserved-capacity** model: a booking's cost is sunk into the objective when made, and
  the reserved space is thereafter **available at zero marginal cost**. The optimizer fills free reserved
  space rather than stranding it, so the artifact cannot arise. No separate "friction" term.
- **2D cap = honest tighter volume** (Decision 2). `C^v` derived from realized `C^w` (÷333.33), NOT a
  fresh draw — preserves the live lane-supply path (B1) and CRN. The operating point is **recalibrated**
  via the tightness knob; the cap change is a real capacity change, not a neutral refactor.
- **S51 M1′-dump reversal BLESSED** (Decision 3) — the forced fallback dump was an artifact of M1′ holding
  no space, not a real cost of rigidity. Eliminated; §14.2 item-3 reconciled.
- **Physical caps use raw `w_k` (no `(1+ε)`)** to match C.5b (VQ1); billing keeps `(1+ε)` via C.4/CW.
- Change-surface expanded to ledger (B2) + `h0_planner` (B3) + CapDecay signature (M1); test contract
  expanded (I5, symmetry, infeasibility, CRN byte-stability, chain, cost-bounds, file assignment).

## 1. Purpose
Fix the S51 commit-timing bug: cargo only touches spot at the cutoff (φ≈15% free), never at −5d (φ≈56%),
and M1′ commits a route early but holds no space → decayed spot infeasibilizes it → `_repair_frozen_
infeasible` dumps ~57% to fallback (self-inflicted). Fix: a run's spot **usage becomes a committed 2D
reservation** that persists across daily runs, does not decay, and is paid for (sunk); both replan arms
reserve symmetrically; the only M1-vs-M1′ difference is assignment fluidity.

## 2. Locked decisions
- Reserve **early**; v1 = **no-cancel, monotone, sunk-cost** (`penalty_frac=1`); releasable/fractional =
  v2 (`penalty_frac<1`).
- **Level 2**: reservation = **2D physical envelope** (weight + volume).
- Identity-lock = `min(doc cutoff, ACAS pre-load)` (v1 `acas_lead_h=0` ⇒ = the physical cutoff);
  scope = **general cargo**.
- **Both arms reserve**; **M1 swaps** within the envelope until cutoff; **M1′ pins** assignment early.
  `L2 = C(M1′) − C(M1)` = value of swapping. M1′ **no longer dumps** (Decision 3).
- 5 sub-decisions: (1) caps from ULD-equivalents; (2) one φ per flight, both dims; (3) independent
  monotone ratchet per dim; (4) split schema `spot_wcap`/`spot_vcap`; (5) `penalty_frac=1` (knob).
- 3 REV-2 decisions: sunk-cost model; honest tighter cap + recalibrate; S51 reversal blessed.

## 3. Current code baseline (validated by the code-fidelity agent — accurate except claim 7)
- Spot capacity is 1D chargeable weight: `RateCatalog.spot_cap: dict[ArcId,float]` (`air_milp.py:170`).
  `_build_spot_cap`/C.5d caps `Σ_g CW_{a,g}≤cap` (MAWB) / `Σ_k cw_k·x≤cap` (co-load) (`air_milp.py:661-669`).
  No volume cap on spot. 2D caps exist only for contracted ULDs — C.5b uses **raw `w_k`** (no ε)
  (`air_milp.py:890-892`).
- `_chargeable_weight = max(w_k, v_k·167)`, `VOLUMETRIC_DIVISOR=167` (`air_milp.py:248-249,75`). C.4:
  `CW≥(1+ε)Σw x` (weight, ε on) and `CW≥Σv·167 x` (volume, no ε) (`:587-590`); `ε=dunnage_eps`.
- `_ULD_TYPES={LD3: w=1500,v=4.5}`; `_DENSITY_LOW/HIGH=120/240`; `4.5·240=1080<1500` ⇒ cargo volume-bound
  in a ULD (`air_generator.py:215,223,331`).
- **Claim 7 CORRECTED:** the LIVE proof path is `_build_lane_rate_catalog` (used when `tau` set,
  `air_generator.py:993`), where `spot_cap` is **lane-supply-derived** (`(1−contracted)·S_ℓ`, belly ×0.4,
  freighter-repositioned; `:721-764`), NOT `n·1500`. The `U(1,3)·1500` form is only the legacy-κ path
  (`_build_rate_catalog`) and a feeder fallback. → §5.2 must derive `C^v` from the realized `C^w`.
- Decay (`cap_decay.CapDecay.__call__:120-138`): only `spot_cap` decays; both BSA firm.
  `avail=booked+max(0,C0−booked)·φ`, `booked=Σ cw of PINNED hawbs`; convex φ; `A_cut` Beta (0.13/0.225),
  λ~U(.10,.16); `cap_decay` RNG stream, reads no demand.
- ReplayState `_tendered`/`_committed`/`_pins` (`replay.py:86-88`). Model-Y floor tendered-only for
  M1/M1p/PIH, tendered∪placed M0, all-placed H0 (`:914-919`); M1p hard-pins placed priors `_route_pins`
  (`:554-584,601-602`). `_SOFT_BSA_RELEASE_LEAD_H=48` (`:501`, unchanged). `_ARMS={M0,M1p,M1,H0,PIH}`
  (`:258`); arc cutoff = `offer.cutoff_utc_h`.

## 4. Notation
| Symbol | Meaning |
|---|---|
| `a` | spot arc (co-load/flat/MFB); BSA excluded |
| `t`, `τ0_a` | sim clock; arc a identity-lock = `min(offer.cutoff_utc_h, ACAS pre-load)`, v1 = `cutoff_utc_h` |
| `C^w_a, C^v_a` | nominal 2D spot caps (kg, m³); `C^w_a` = the live path's current `spot_cap`; `C^v_a = C^w_a/333.33` |
| `r^w_a, r^v_a` | reserved committed envelope (state; monotone non-decreasing, v1) |
| `φ_a(t)` | booking curve ∈ [A_cut,1], one per arc (unchanged) |
| `avail^w_a, avail^v_a` | decayed, reservation-floored available caps |
| `x_{k,a}` | HAWB k on arc a this cycle; `w_k,v_k` actual weight/volume; `ε` dunnage |
| `CW^r_a` | reserved envelope's billing chargeable-weight = `max((1+ε)·r^w_a, r^v_a·167)` (ε on weight, matches C.4) |

## 5. The model

### 5.1 Reservation object & state
`ReplayState._reserved_spot: dict[ArcId, tuple[float,float]] = (r^w_a, r^v_a)`, default `(0,0)`, each
component monotone non-decreasing (v1). **Supersedes** the Model-Y tendered-cw decay floor for spot: the
floor is now the reservation envelope, accumulated by each arm from its OWN solve usage (this is what
makes M1/M1′ symmetric on capacity, and is legitimately arm-varying replay state — **NOT** part of the
D-A16 frozen capacity vector, which keeps `(C^w,C^v,φ)` bit-identical across arms).

### 5.2 2D capacity (generator) — B1 fix
Keep `C^w_a` = whatever the live path already produces for `spot_cap[a]` (lane-supply-derived in the
region path, `round(n_a·1500)` in the legacy path) — **untouched**, so the S50 lane-supply/repositioning
mechanism and CRN byte-stability are preserved. Derive the volume cap:
```
C^v_a = C^w_a / 333.33        # 333.33 = 1500/4.5 = LD3 physical density
```
This is the physically-honest volume envelope (Decision 2); it is ~2× tighter than the old 1D CW cap
implied for volume-bound cargo. **The operating point is recalibrated via the tightness knob (τ/supply),
not by inflating the cap.** No new RNG draw ⇒ every existing decay/demand golden stays byte-identical
(only the attribute name changes).

### 5.3 2D decay floor (CapDecay) — one φ, both dims; handoff branch
```
avail^w_a(t) = r^w_a + max(0, C^w_a − r^w_a) · φ_a(t)
avail^v_a(t) = r^v_a + max(0, C^v_a − r^v_a) · φ_a(t)
```
Pre-cutoff (t < τ0_a): floor = the quantity envelope `(r^w_a, r^v_a)`.
Post-cutoff (t ≥ τ0_a): floor = the frozen identity pins —
`avail^w_a ≥ Σ_{k∈pins(a)} w_k`, `avail^v_a ≥ Σ_{k∈pins(a)} v_k` (raw, no ε). **Branch, never sum** —
`r` and pin-sums never add (no double-count). `CapDecay.__call__` signature gains the reservation
envelope + pin identities (threaded from `run_replay`); it now returns decayed `spot_wcap`/`spot_vcap`.

### 5.4 2D MILP caps (replaces C.5d) — raw w/v, no ε (VQ1)
```
Σ_{k on a} w_k x_{k,a}  ≤  avail^w_a(t)          (C.5d-w)   # raw, matches physical C.5b
Σ_{k on a} v_k x_{k,a}  ≤  avail^v_a(t)          (C.5d-v)
```
Capacity is 2D-physical on raw (w,v); **billing is unchanged** — CW = max((1+ε)Σw, Σv·167) via existing
family formulas. For MAWB arcs the sums are over group members; for co-load, per-rider on `x`. (Semantic
change from capping the density-mixed CW to capping raw physical dims — intended.)

### 5.5 Usage ratchet (post-solve, each cycle)
```
used^w_a = Σ_k w_k x*_{k,a}      used^v_a = Σ_k v_k x*_{k,a}      (raw, match C.5d)
r^w_a ← max(r^w_a, used^w_a)     r^v_a ← max(r^v_a, used^v_a)
```
Reads only the realized solve (no demand draws) ⇒ CRN-safe. `r ≤ C` always. Cross-dim cancellation
(OR agent): `CW^r_a = max(r^w_a, r^v_a·167) = max_t CW_{a,t}` = peak single-cycle billing CW — the box
introduces no extra charge beyond the peak.

### 5.6 Reserve→identity handoff at cutoff
At `t ≥ τ0_a`, freeze `{(k,a): x*_{k,a}=1}` into identity pins; arc closes to swaps. Post-cutoff decay
floor uses pins (§5.3). Pre-cutoff: NO `(hawb,arc)` pins on spot. **`τ0_a` vs `tender_at`:** `τ0_a` is
the per-ARC identity-lock (`offer.cutoff_utc_h`); the existing per-SHIPMENT `tender_at` still governs
when a HAWB becomes irreversibly tendered. Rule: a HAWB firm-tendered (`tender_at ≤ t`) is pinned to its
current arc even before `τ0_a`; an arc reaching `τ0_a` pins ALL its current riders. Reconcile both in S5.

### 5.7 Committed spot cost — SUNK-COST / FREE-RESERVED-CAPACITY (REV 2, the core correction)
The reservation cost is **sunk into the objective when booked**; reserved space is thereafter **free at
the margin**. Concretely, per cycle t, the spot cost the objective charges on arc a is only the part
**beyond** the standing reservation:
```
spot_cost_a(cycle t) = family_cost(CW_{a,t}) − family_cost(CW^r_a,prev)     # clamped ≥ 0
```
where `CW^r_a,prev = max((1+ε)·r^w_a, r^v_a·167)` from prior cycles (a constant in this solve; `(1+ε)` on
weight matches C.4 billing; offset is PER-GROUP for multi-group flat/MFB arcs; `family_cost(0):=0`). Using reserved
space (up to `CW^r_a`) costs **0** at the margin ⇒ the optimizer FILLS free reserved space rather than
stranding it ⇒ the M1-only "friction" artifact cannot arise (VQ2 resolved). After the solve the ratchet
grows `r` and the newly-bought increment is sunk. **Total committed spot cost for arc a over the horizon
= `family_cost(CW^r_a,final)` = `family_cost(peak reserved CW)`, paid once.**
- `family_cost` is the arc's own family formula (co-load `m·CW`; flat `max(min_chg, m·CW)`; MFB
  next-break-down). Using a *difference of the same monotone `family_cost`* (not a marginal break rate)
  keeps it monotone in reserved size and coherent across families (VQ5 resolved; the old "MFB rate at the
  break covering CW^r" is deleted).
- **D5:** this committed cost is legitimate `real_cost` (real money to the carrier), in-objective,
  symmetric across arms ⇒ it does NOT contaminate `L2` (no separate friction line needed). Tardiness
  (C.10) is still NOT summed in; `fallback_penalty` stays separate.
- **`penalty_frac` knob:** v1 `=1` ⇒ full sunk, no recovery. v2 `<1` ⇒ release unused reserved space
  before cutoff, paying only `penalty_frac·family_cost(released)` (this is where cancellation lives).
- **Determinism caveat (monitor):** `r` reads the realized (w,v) split of a `mip_rel_gap=0.005`
  incumbent. Byte-reproducible for a fixed build (I10); flagged as gap-sensitive — since the committed
  cost is now in-objective the solver optimizes it (less ill-conditioned than the scored version), but
  a chain/CRN test must confirm stability.

### 5.8 Arms
- **M1** (open book): reserves (§5.5); re-solves assignment each cycle → swaps within `(r,avail)` until
  cutoff. Free reserved space ⇒ it will not strand paid capacity.
- **M1′** (single-pass): reserves (§5.5); pins assignment at first placement (`_route_pins`) → no swap.
  Its reserved space is held+free ⇒ decayed spot no longer infeasibilizes it ⇒ **no forced fallback dump
  (Decision 3, S51 reversal).** Difference vs M1 = assignment fluidity only.
- **M0** (greedy), **H0** (human daily): also reserve (§5.5). **π_hind** (clairvoyant): single terminal
  solve ⇒ no cross-cycle accumulation; reservation trivial.
- **Chain:** `C(M1′) ≥ C(M1)` is now **empirical under reservation coupling** (per-arm divergent `r`),
  not a by-construction inequality — a per-draw chain-monotonicity test replaces the guarantee.

## 6. Loop control flow (each cycle t)
1. Decay catalog to t, floored per §5.3 (envelope pre-cutoff / pins post-cutoff).
2. Solve against `avail^w,avail^v`; objective charges spot per §5.7 (reserved free).
3. Ratchet `r^w,r^v` from `x*` (§5.5).
4. Freeze pins for arcs with `τ0_a ∈ [t, next_t)` (§5.6).
5. Advance to next event (cadence unchanged + soft-BSA cliffs).
Terminal: realized cost = family costs incl. committed spot (§5.7) + OTP + fallback split.

## 7. Change surface (complete — from code-fidelity agent)
- `air_milp.py`: `RateCatalog` split `spot_cap`→`spot_wcap`,`spot_vcap` (:170); `_build_spot_cap`→C.5d-w/v
  raw dims over members (:640-669).
- `cap_decay.py`: `CapDecay.__call__`/`__init__` 2 dicts + reservation+pins args + cutoff branch (:120-138).
- `replay.py`: `ReplayState._reserved_spot` + ratchet; loop (:906-920) pass reservation, §6 order; ledger
  `_ledger_caps`(:351)/`_solve_claims`(:367-377)/registration(:823-827) → **2D collapses to a CW-equivalent
  audit line** (`max(C^w, C^v·167)`; ledger is audit-only, tendered=0) (B2); handoff pins (§5.6);
  committed-spot cost (§5.7).
- `h0_planner.py`: `:98-102` single-float `spot_cap` read → consume the **CW-comparable** dimension
  `max(C^w, C^v·167)`; membership test → "arc is spot" predicate (B3).
- `air_generator.py`: `_draw_spot_regime`/both catalog builders — add `C^v=C^w/333.33` (B1); `C^w` and all
  RNG draws unchanged.
- `scenario_io.py`: `_serialize_rate`(:215-226)/`_deserialize`(:340-359) two-key round-trip.
- `tpeb_air_instance.py:70`: doc comment.
- `# TODO(v2): spot reservation cancellation (penalty_frac<1, releasable r)` at the ratchet site →
  BUILD_STATUS deferred entry.
- **Green tests that must migrate** (schema/floor): `test_cap_decay.py:62-106`,
  `test_air_milp.py:193-233`, `test_air_generator.py:131-140`, `test_network_supply.py:194-334`,
  `test_replay*.py` ledger reads. Regression gate: full 386 must pass after migration.

## 8. Invariants (each has a test)
- **I1** `r^w,r^v` monotone non-decreasing. **I2** `avail^{w,v} ≥ r^{w,v}` all t. **I3** free pool decays:
  `avail = r+(C−r)φ` per dim. **I4** `r ≤ C`, `avail ≤ C`. **I5** every solve respects C.5d-w AND C.5d-v;
  no negative free pool. **I6** CW-neutral dense→bulky swap breaching `avail^v` is infeasible. **I7**
  weight-breach symmetric. **I8** swap pre-`τ0_a` / lock post. **I9** committed cost = `family_cost(peak
  reserved CW)`; reserved space free at margin (using it adds 0); reservation-off reduces to current
  billing. **I10** CRN: same seed ⇒ identical `(r^w,r^v)` trajectory; demand/τ/α leave decay byte-identical;
  `spot_wcap[a]==` old `spot_cap[a]`. **I11** BSA firm; only spot floor changes. **I12** M1 & M1′ share the
  reserve floor (same `r` trajectory shape) yet differ in assignment; M1′ no forced fallback. **I13**
  per-draw chain `C(M1′) ≥ C(M1)` on real_cost (empirical check). **I14** determinism/CRN byte-stability
  of the schema split across a full golden run.

## 9. Test contract (isolation, real HiGHS, no mocks, tardiness ON; value-bounds not status-only)
| # | Test | File | Invariant | Assert |
|---|---|---|---|---|
| 1 | reserve_ratchet_monotone (incl. **less-then-more**) | test_replay_loop | I1,I4 | `r` trajectory, never drops on the low cycle |
| 2 | reserved_no_decay_free_pool_decays | test_cap_decay | I2,I3 | `avail=r+(C−r)φ` per dim, exact |
| 3 | **volume_breach_swap_infeasible** (headline) | test_air_milp | I6 | dense w=240/v=1.0 (CW240) vs bulky w=180/v=1.437 (CW240); `avail^v=1.2`,`avail^w=250`; **cost bound** (dense-on-spot + bulky-spilled-to-alt) |
| 4 | weight_breach_swap_infeasible | test_air_milp | I7 | symmetric; **cost bound** |
| 5 | viable_swap_taken_when_optimal | test_replay_loop | I8 | swapped cost == cheaper hand-bound |
| 6 | identity_lock_after_cutoff | test_replay_loop | I8 | no post-`τ0_a` reassignment |
| 7 | committed_cost_and_reservation_off | test_air_milp/scorer | I9 | reserved free at margin; reservation-off == current billing (exact) |
| 8 | cap_geometry_volume_bound | test_air_generator | §5.2 | `C^w/C^v==333.33`; ≤240 cargo volume-bound |
| 9 | crn_reservation_determinism + **spot_wcap==old spot_cap** | test_cap_decay/determinism | I10,I14 | byte-identical golden run |
| 10 | bsa_firm_under_reservation | test_cap_decay | I11 | BSA untouched |
| 11 | m1prime_no_forced_fallback | test_replay_loop | I12 | fallback_count drop + **C(M1′) vs C(M1) cost values** |
| 12 | cross_arm_reserve_symmetry + **chain** | test_replay_loop | I12,I13 | same `r`-shape, different assignment, `C(M1′)≥C(M1)` |
| 13 | graceful_infeasibility (2D volume-tight) | test_air_milp | I5 | structured status/fallback, NOT raise |
| 14 | full 386-suite regression gate | (CI) | — | all pass post-migration |

## 10. Build slices (one at a time, isolation-tested)
- **S1** generator `C^v=C^w/333.33` + schema split + persistence round-trip (tests 8,9,14-partial).
- **S2** `CapDecay` 2D floor + reservation/pins args + cutoff branch (tests 2,9,10).
- **S3** `ReplayState._reserved_spot` + ratchet (tests 1).
- **S4** C.5d-w/v 2D MILP caps (tests 3,4,5,13).
- **S5** reserve→identity handoff + `τ0_a`/`tender_at` reconcile (tests 6).
- **S6** committed-spot sunk cost in objective + scorer + ledger 2D + h0_planner (tests 7,11).
- **S7** wire arms + M1′ coherence (S51 reversal) + recalibrate operating point + NC1–5 (tests 11,12).

## 11. Methodology reconciliations (decided — fold into `arrival_only_replan_methodology.md`)
| Item | Action |
|---|---|
| D-A1 (single tender) | REPLACED by reserve-early/assign-late; rewrite §4/§8. |
| §4 tender binary | → three-state (reserved-space / assigned-pinned / open-book). |
| §14.2 Model-Y floor (tendered-only) | superseded by the reservation floor; re-derive per-arm floor table; verify M0/H0/π_hind + green tests (VQ3). |
| §14.2 item-3 (M1′ dump) | **REVERSED (Decision 3)** — dump was artifact of unheld space; eliminate. |
| D-A19/§14.7 spot cap (1D CW) | → 2D raw w/v (C.5d-w/v). Reconcile D-A20/D-A21 narrative. |
| D5 (two-number cost) | committed spot cost ∈ real_cost, in-objective, symmetric (NOT a separate friction line). |
| D-A16 frozen vector | one line: `_reserved_spot` is arm-varying replay state, not the frozen `(C^w,C^v,φ)`. |
| S52 decision-2 (releasable/fractional) | v1 `penalty_frac=1` sunk = deliberate simplification; v2 = releasable. |
| Operating point | RECALIBRATE (Decision 2) — 2D cap is a capacity change; L2 not comparable to old stale runs. |
| supply⟂demand (D-A18) | no leak — reservation reads `x*` like the approved §14.2 floor; draws no RNG. |

## 12. Validation questions for REV-2 review agents (challenge these)
- **RQ1** Does the sunk-cost/free-reserved model (§5.7) actually remove the VQ2 L2 artifact, and is the
  per-cycle "charge only beyond reservation" objective well-posed for all three families (esp. MFB
  next-break-down with a constant `family_cost(CW^r)` offset)?
- **RQ2** With committed cost in-objective + reserved-free, does `C(M1′) ≥ C(M1)` hold per-draw, or can M1
  over-reserve (higher peak) and invert the chain? Construct a case.
- **RQ3** `CW^r_a = max(r^w, r^v·167)` mixes the raw-physical reservation with the 167 billing divisor —
  is that the right billing CW-equivalent of a physical envelope, or should ε/167 enter differently?
- **RQ4** Does deriving `C^v=C^w/333.33` interact correctly with the lane-supply/belly-thinning/
  repositioning path (does belly ×0.4 on `C^w` propagate sensibly to `C^v`)?
- **RQ5** Ledger CW-collapse (B2) — is `max(C^w, C^v·167)` the right audit-only conservation quantity, and
  does it keep the ledger's ε-conservation intact?
- **RQ6** `τ0_a` (arc) vs `tender_at` (shipment) rule (§5.6) — any ordering where a tendered HAWB and an
  arc-lock conflict and leave a route infeasible at handoff?
- **RQ7** Determinism: is the reserved `(r^w,r^v)` gap-robust at `mip_rel_gap=0.005`, or can two cost-equal
  incumbents give different `r` and thus different committed cost feeding L2?
