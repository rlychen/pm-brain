# Interface-Seam / Contract Audit — Session 45

**Scope:** cross-seam contract drift only (producer emits X, consumer expects Y — does X==Y?).
Read-only review. New surface since S36: the full 2c replay loop (`src/replay.py`), the
scorer, the `ReplayState` ledger, 2c-7 disruption recourse (slices 1+2a), F1 Slice C
spot-cap persistence, D-A24 region→region, and the MFBlink gap-aware billing validator.

**Verdict headline:** **ONE BLOCKING finding** at the disruption seam: the §6 recourse gate
silently fails to fire for any committed route whose first air leg's *scheduled* departure
precedes the cargo's `ready_at` — which is a large fraction of MILP-feasible routes on the
daily substrate (1–7 of 12 per scenario across seeds 2/3/4/7). For those shipments,
`route_eta_under_disruptions` returns `(inf, broken=True)` with **no disruption present**,
violating the documented "reduces to `route_reliability`'s mean for any feasible route"
contract; this poisons the gate's `base_ok` / `new_break` logic so neither a deadline-blowing
delay **nor a cancellation** of their flights triggers replan. The ledger audit-only invariant,
the pin-on-tender seam, the recourse-W stamp, the billing validator, and both S36 deferred
round-trips are otherwise **PASS, verified empirically**. The two S36 MATERIAL items are now
**partially closed** (round-trip dimension verified clean; the `load`-asymmetry doc gap from M-1
persists, see carry-forward).

---

## NEW PATH — 2c disruption recourse (`route_eta_under_disruptions` vs `route_reliability`)

### The claimed contract (the seam under audit)

`src/components/air_transit_time.py:163` `route_eta_under_disruptions` docstring:

> "With no disruptions this reduces to the mean of `route_reliability` for any feasible
> route (its clock is ≤ each scheduled departure by construction), so an empty disruption
> set leaves the projected ETA — and hence the §6 replan gate — inert."

The §6 gate `_refresh_active_shipments` (`src/replay.py:455`) computes a **baseline**
projection (no disruption) and a **disrupted** projection of each firm route, then flags a
replan only when the disruption *degrades* a previously-OK route:

```python
base_eta, base_broken = route_eta_under_disruptions(arcs, ready_h=ready_at[k])
eta,      broken      = route_eta_under_disruptions(arcs, delays, cancels, ready_h=...)
base_ok   = (not base_broken) and base_eta <= delta_k[k]
now_ok    = (not broken)      and eta      <= delta_k[k]
new_break = broken and not base_broken
if new_break or (base_ok and not now_ok):   # ← the gate
    to_replan.append(k)
```

The contract REQUIRES `base_broken == False` for every committed-feasible route, or the gate
is dead for that shipment.

### EMPIRICAL — the contract is violated, systematically

`route_reliability` rebases each leg with `clock = max(leg.dep_utc_h, clock)` — a cargo ready
after a flight's scheduled departure "waits for the flight" (next-cycle / daily-repeating
semantics; the daily substrate `build_tpeb_daily` tiles each lane as a recurring departure).
`route_eta_under_disruptions` treats every departure as **hard / one-shot**
(`if clock > dep + tol: return inf, True`). On the daily substrate the MILP routinely commits a
HAWB whose `ready_at` exceeds its first air leg's scheduled `dep` (it boards the next operation
of that recurring flight). Those two walks then diverge to `inf` vs finite.

Across four scenario seeds, every committed feasible route counted, **no disruption injected**:

```
seed=2: feasible=12, route_eta 'broken' w/ NO disruption=7, finite-but-mismatch=0
seed=3: feasible=12, route_eta 'broken' w/ NO disruption=1, finite-but-mismatch=0
seed=4: feasible=12, route_eta 'broken' w/ NO disruption=5, finite-but-mismatch=0
seed=7: feasible=12, route_eta 'broken' w/ NO disruption=2, finite-but-mismatch=0
```

Concrete instance (seed 3, `gen-3-hawb-7`): `ready=34.89h`; route boards `CI:TPEHKG#d0`
scheduled `dep=26.0h`. `route_reliability` mean = **106.79h** (rebases to next cycle);
`route_eta_under_disruptions` = **`(inf, True)`**. The "clock ≤ each scheduled departure by
construction" premise is simply false on this substrate.

Root cause walk (seed 2, all 7 broken routes): every one is `ready/clock > first-air dep`:

```
gen-2-hawb-0 : ready=84.98 > dep=54.00 (CI:TPELAX#d1)   gen-2-hawb-7 : ready=55.13 > dep=54.00 (CI:TPELAX#d1)
gen-2-hawb-1 : ready=58.28 > dep=50.00 (CI:TPEHKG2#d1)  gen-2-hawb-9 : ready=89.08 > dep=54.00 (CI:TPELAX#d1)
gen-2-hawb-3 : ready=62.82 > dep=50.00 (CI:TPEHKG2#d1)  gen-2-hawb-11: ready=102.28> dep=54.00 (CI:TPELAX#d1)
gen-2-hawb-4 : ready=52.10 > dep=50.00 (CI:TPEHKG2#d1)
```

### EMPIRICAL — the gate is dead for these shipments (the BLOCKING consequence)

Because `base_broken=True`, `base_ok` is permanently `False`, so the `(base_ok and not now_ok)`
branch can never fire. And `new_break = broken and not base_broken = (True and not True) = False`,
so the cancellation branch is dead too. Injecting a **+500h delay** (far past `Δ_k=161.8`) on the
last flight of a base_broken shipment:

```
target shipment gen-2-hawb-7: base_broken=True, last flight CI:TPELAX#d1, delta_k=161.8
§6 gate flags for replan under +500h delay: ['gen-2-hawb-6', 'gen-2-hawb-8']
  -> is gen-2-hawb-7 in the replan set? False
```

And **cancelling** that shipment's flight outright:

```
CANCEL flight CI:TPELAX#d1 of base_broken shipment gen-2-hawb-0: flagged for replan? False
```

A firm shipment whose flight is literally cancelled receives **no recourse**. The capability
that §6 / 6.1 specifies as "a TESTED CAPABILITY that must work and not corrupt state" silently
no-ops for a large minority of the book. The existing recourse tests
(`test_disruption_cancel_reroutes_and_freezes_promise`,
`test_flown_prefix_is_departed_by_decision_clock`) pass **only because** they select via
`_first_single_air_arc_shipment`, which happens to land on a non-broken-baseline shipment; they
never assert that a base_broken shipment recovers.

### Classification: **BLOCKING**

Producer (`build_tpeb_daily` daily-repeating departures + `route_reliability`'s `max(dep,clock)`
rebasing, the established feasibility semantics the MILP optimizes against) → consumer
(`route_eta_under_disruptions`'s hard one-shot departures) → the §6 gate. The two walks
disagree on what "feasible route" means. The headline (no-disruption) L2 numbers are unaffected
(the refresh returns `[]` when `affected` is empty, short-circuiting before the broken
projection — `src/replay.py:470`), so this is not a value-number corruption. But the recourse
**capability** is broken for the affected shipments, and the gated spec (§6) claims it works.

**Suggested fix direction (not implemented — read-only):** give `route_eta_under_disruptions`
the same daily-rebasing `max(dep, clock)` semantics as `route_reliability` on the *undisrupted*
legs, and apply the hard-departure miss test only to legs that a delay/cancel actually perturbs
relative to that rebased clock — i.e. the disruption walk should reduce **exactly** to
`route_reliability`'s mean when `delays={}`/`cancels=∅`, which is the contract the docstring
already promises. Then re-point the gate's baseline at that reduced form. Add a regression that
asserts `route_eta_under_disruptions(committed_route, no-disruption) == route_reliability(...)[0]`
for every MILP-committed route, and a recourse test that *targets* a `ready_at > first-dep`
shipment.

---

## Standing seam table

| Seam | Producer | Consumer | Verdict |
|---|---|---|---|
| **Disruption baseline ↔ route_reliability** | `route_eta_under_disruptions` (no-disrupt) | §6 gate `_refresh_active_shipments` | **BLOCKING** — `(inf,True)` on MILP-feasible `ready_at>dep` routes; gate dead (empirical above) |
| Ledger audit-only `tendered==0` | `reconcile(tendered=0, committed_untendered=claim)` | `capacity_ledger` / scorer / generator | **PASS** — distinct tendered = `[0]` (uld) / `[0.0]` (spot) across all 5 arms; `cap = committed + free` every row, 0 violations; no consumer reads a tendered split |
| Pin-on-tender | `state.mark_tendered` → `tendered_set()` | `solve(pinned=...)` | **PASS** — `test_tendered_route_is_frozen_after_cutoff` + `test_m1p_freezes_route_from_first_placement` (31/31 green); air-arc set identical every cycle post-tender |
| `_flown_prefix` ↔ M1 re-solve pins | `_flown_prefix` (departed-by-clock) | `state.release` + `placed_routes[k]=prefix` | **PASS** — `test_flown_prefix_is_departed_by_decision_clock`; delay pushes departure ⇒ air arc leaves the prefix; cargo re-routes from anchor head |
| Recourse-W stamp ↔ C.10 | `replace(hawb, tardiness_weight=_RECOURSE_W[tier])` | `air_milp` C.10 PWL (`hawb.tardiness_weight`) | **PASS** — W reaches the PWL; M5 explosion-guard (air_milp:336) protects the ∞-deadline×W>0 case; cancel recovery test green |
| Gap-aware billing validator | `_check_billing(got, expected, mip_gap)` | `solve()` (MFBlink cut, gap-stopped OPTIMAL) | **PASS** — under-bill raises at gap∈{0,0.5}; over-bill raises only at gap≤1e-9, tolerated at gap=0.005 (empirical) |
| F1 Slice C spot-cap round-trip | `_serialize_rate` `rate_json.spot_cap` | `_load_rates` `spot_cap[arc]` | **PASS** — 48 entries round-trip bit-identical through SQLite; `solve` cost+routes identical |
| D-A24 region→region round-trip | `_persist_hawbs` door coords + offers | `_load_hawbs` / `build_geo_air_graph` | **PASS** — door coords identical; `solve(orig)=solve(roundtrip)=34146.0756`, routes identical |
| Disrupted-sim rebuild | `_disrupted_sim` | `build_geo_air_graph` re-solve | **PASS** — `test_disrupted_sim_shifts_delayed_and_drops_cancelled`; delayed legs slip dep+arr, cancelled offers dropped, empty ⇒ same object |
| PIH terminal-cycle collapse | `times=[base[-1]]` | replay loop / scorer | **PASS** — `test_pih_is_a_single_full_book_cycle`; one planning cycle, full book, no pins |

---

## Carry-forward — the two S36 MATERIAL items

**S36 M-1 — `load()` not a full inverse of `persist()` for arrival scenarios → PARTIALLY CLOSED.**
The *replay loop* now reads `tier`/`known_at`/`ready_at`/`tender_at`/`effective_deadline_at`
directly from the `shipments` table via `sdb.select`/`visible_shipments` (`src/replay.py:667–671`),
NOT via `load()`/`SimInputs` — exactly as M-1 predicted ("2c reads those columns via the SQL
view, not via `load`"). So the asymmetry caused no bug: the design routed around it. **Still
open (doc only):** `scenario_io.load`'s docstring still says "reconstruct the `build_air_graph`
+ solve inputs" and `SimInputs` still carries no arrival fields, so the partial-inverse
foot-gun M-1 flagged is unmitigated in documentation. Downgrade to MINOR.

**S36 M-2 — `effective_deadline_at` vs `soft_deadline_h`, two columns / divergent readers →
STILL OPEN, latent.** Confirmed both columns persist and the split readers persist: the MILP
penalizes `soft_deadline_h` (C.10), while the replay loop/scorer bind the promise on
`effective_deadline_at` (`run_replay` reads `delta_k = r["effective_deadline_at"]`,
`src/replay.py:669`; `booking_promise.promised_deadline_at ← delta_k`; the scorer's OTP is
`arrival <= promised_deadline_at`). They agree by construction at generation (`derive_deadline`
stamps both), so no numeric drift today — but the optimizer-objective deadline and the
OTP-scoring deadline are now genuinely **two different columns read by two different layers**.
M-2's foot-gun is realized in structure (not yet in value). Recommend picking one canonical
Δ_k. **Not closed.**

**S36 Postgres-parity MINOR** — unchanged: `soft_deadline_h`/`tardiness_weight` live in SQLite
+ are read by the MILP but remain absent from `data_model.md`'s Postgres canonical; a Postgres
swap would still lose them. Deferred.

---

## Tests run

`PYTHONHASHSEED=0 uv run pytest tests/test_replay_loop.py -q` → **31 passed in 54.42s**.
Plus the bespoke seam snippets shown inline above (disruption-baseline divergence sweep, gate
dead-fire under +500h delay and under cancel, ledger tendered-invariant across 5 arms, billing
validator truth table, F1 spot-cap + D-A24 round-trip equality).

**Note on the green suite:** all 31 pass, yet the BLOCKING defect is live — the recourse tests
do not cover the `ready_at > first-dep` shipment class. A passing suite is not coverage of this
seam; the missing regression is itself part of the finding.
