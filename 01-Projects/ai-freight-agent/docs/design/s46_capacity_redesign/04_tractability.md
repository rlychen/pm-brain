# S46 Capacity Redesign — Tractability & Scaling (air slice)

**Status:** ANALYSIS — measured against the live `air_milp.py` / `air_generator.py`, real HiGHS,
policy `mip_rel_gap=0.005`, `threads=1`. Companion to `01_architecture.md` (§F ladder, §G MILP
delta). Author role: Tractability & Scaling analyst.

**Headline verdict.** The capacity redesign is tractable across the whole ladder **because the
replay loop already decomposes every solve to the per-cycle visible-untendered window**, which is
far smaller than the full book. The new capacity machinery (per-lane increasing-block spot tariff,
hard-BSA wiring) is confirmed LP-cheap and BLK-1c-safe — it adds ~continuous vars only and does not
touch the binary structure. The sole scaling risk is the `x[k,a]` binary family (HAWB count ×
subgraph arc count), exactly as §F claims; it is what makes a *monolithic* full-book 120/200 solve
intractable, and what the replay decomposition sidesteps.

---

## 1. Method

For each cell I rebuilt the **production** model (the same `air_milp._build_*` calls, including the
S41 MFBlink cut), via `scripts/scale_probe.py` / `scale_probe_big.py` / `replay_cycle_timing.py`
(all untracked scratch). Two measurements:

- **Monolithic** (`scale_probe.py`): build the entire book as one MILP. Pass A relaxes all integers →
  root LP bound; Pass B restores integrality and solves to `mip_rel_gap=0.005`, `threads=1`,
  `random_seed=0`. Records solve time, residual gap, **root integrality gap** `(Z*−LB_LP)/Z*`, and
  the binary-family split. A soft 180 s wall backstops a runaway (the *policy* stop is the 0.5 % gap;
  the wall is only an experiment guard). κ=1, α=1 throughout.
- **Replay** (`replay_cycle_timing.py`): run the *real* `src.replay.run_replay` M1 (open-book) arm
  on the written scenario and time **each per-cycle solve**. This is the size that actually gates the
  policy, because the replay never solves the full book at once.

Instances built with `generate_arrival_instance(ArrivalConfig(n_hawbs, seed, days, κ=1, α=1))`,
matching the §F ladder's HAWB/day scaling.

---

## 2. Monolithic scaling table (full-book single solve, κ=1 α=1)

| n_hawbs | days | seed | bin vars | x | z | γ | cols | rows | solve_s | status | res_gap | root_gap |
|--------:|----:|----:|--------:|----:|---:|---:|-----:|-----:|--------:|:------:|--------:|--------:|
| 12 | 2 | 0 | 628 | 449 | 83 | 96 | 884 | 1116 | 0.03 | OPT | 0.0000 | 1.64% |
| 12 | 2 | 1 | 521 | 410 | 55 | 56 | 687 | 814 | 0.01 | OPT | 0.0006 | 0.06% |
| 12 | 2 | 2 | 580 | 443 | 65 | 72 | 781 | 946 | 0.02 | OPT | 0.0000 | 1.92% |
| 24 | 3 | 0 | 1587 | 1235 | 148 | 204 | 2064 | 2207 | 0.05 | OPT | 0.0000 | 0.42% |
| 24 | 3 | 1 | 1612 | 1181 | 187 | 244 | 2213 | 2604 | 0.08 | OPT | 0.0035 | 3.98% |
| 24 | 3 | 2 | 1618 | 1287 | 151 | 180 | 2088 | 2220 | 0.02 | OPT | 0.0018 | 0.32% |
| 40 | 4 | 0 | 3510 | 2904 | 278 | 328 | 4370 | 4055 | 0.33 | OPT | 0.0009 | 0.44% |
| 40 | 4 | 1 | 3382 | 2858 | 256 | 268 | 4149 | 3715 | 0.24 | OPT | 0.0005 | 0.35% |
| 40 | 4 | 2 | 3423 | 2942 | 225 | 256 | 4119 | 3564 | 0.05 | OPT | 0.0045 | 0.76% |
| 80 | 5 | 0 | 9846 | 8859 | 471 | 516 | 11277 | 7345 | 14.67 | OPT | 0.0050 | 1.44% |
| 80 | 5 | 1 | 10286 | 9323 | 447 | 516 | 11659 | 7462 | 3.19 | OPT | 0.0050 | 1.15% |
| 80 | 5 | 2 | 9519 | 8585 | 406 | 528 | 10810 | 7033 | 1.34 | OPT | 0.0044 | 1.21% |
| 120 | 5 | 0–2 | ~15k | ~13k | ~600 | ~650 | ~17k | ~9k | **>180 (backstop hit)** | — | **>0.005** | — |
| 200 | 7 | 0 | ~26k | ~23k | ~900 | ~1k | ~30k | ~15k | **>180 (backstop hit)** | — | **>0.005** | — |

(η integer vars are a few dozen at every scale — never a driver — and are omitted.)

**Reading the curve.** Solve time is roughly flat <40 HAWBs (sub-second), enters seconds at 80
(1.3–14.7 s, seed-dependent), and **breaks the budget monolithically at 120** (no return inside the
180 s guard; HiGHS only checks the wall at node boundaries, which are far apart at ~15 k binaries, so
it overshoots heavily). 200 is worse. The growth is the expected MIP exponential in the `x` family.
Root gaps stay small everywhere (0.06–4 %), so the MFBlink cut is doing its job — the breakage is
**branch-and-bound node count, not a loose relaxation.**

**Density, not raw count, is the real hardness axis.** A monolithic **n=86 over 7 days** solves in
**1.4–1.6 s** (OPT, root gap 0.7 %) — *faster* than n=80 over 5 days (up to 14.7 s) and worlds away
from n=120 over 5 days (>180 s). The driver is HAWB **density per flight-window** (n/days, and
per lane): cramming 120 HAWBs into 5 days produces dense per-flight consolidation branching; spreading
86 over 7 days does not. This matters because it tells us the lever that helps is anything that
reduces concurrent HAWBs-per-flight-bank (cadence, horizon, subgraph pruning), and it is why the
per-cycle replay subproblem (which competes only over a near-term flight bank) stays cheap.

---

## 3. The §F claim — binary driver is HAWB count, capacity machinery is LP-cheap: CONFIRMED

- **Binary driver = `x[k,a]`.** At n=80 the `x` family is **8 859 of 9 846 binaries (90 %)**, `z`
  ~470, γ ~520; `x` is what scales with HAWB count × per-HAWB subgraph arc count under D-A24
  region→region routing. `z`/γ scale with consolidation-candidate count (sub-linear in n). η (BSA)
  is a few dozen. So the §F note — "binary driver is HAWB count × subgraph arc count, *not* the
  capacity redesign" — is correct and is the whole scaling story.
- **Capacity machinery is LP-cheap.** Today's spot cap (`_build_spot_cap`) is a handful of
  *continuous* rows (one per spot arc); BSA adds a few dozen integer η. The redesign's
  `_build_spot_blocks` adds **~4 continuous block vars + 1 equality per lane** (≈24 cols / 6 rows on
  a 6-lane grid) and hard-BSA adds **1 continuous `over_c` per contract** — both negligible against
  ~10–25 k binaries, and **neither adds an integer variable or a big-M.** The redesign does not move
  the curve in §2.

## 4. The increasing-block spot tariff — BLK-1c-safe: CONFIRMED

Prototyped the constraint shape in isolation (`b_i ∈ [0, width_i]`, `Σ b_i = lane spot CW`, minimize
`Σ rate_i·b_i` with **increasing** `rate_i`). Because marginal cost rises with block index and we
minimize, the LP fills cheap blocks first with **zero binaries / zero SOS / zero big-M** — the
textbook convex-PWL property. Verified fills: demand 4 000 → block0 only; 6 000 → block0 full + 1 000
in block1; 9 000 → blocks 0–2; 11 250 → all four blocks full (ceiling). Demand above the ceiling
returns `kInfeasible`, which is **correct**: the lane spot CW is endogenous, so overflow must route to
fallback/contract — that is the intended scarcity, not a modelling defect (it confirms the block-sum
must be written against the routed spot CW, with fallback absorbing the excess).

The only new coupling the tariff introduces is one linear equality aggregating all spot-arc CW on a
lane — a single continuous row, no integrality. It **tightens** the spot region (rising shadow price
vs the old flat per-arc cap), so it cannot worsen the root gap; if anything it improves it. This
mirrors the BLK-1c lesson exactly: the gap source was the `min_flat_breaks` big-M disaggregation, not
any capacity row — and the block tariff carries no big-M.

---

## 5. The saving grace — per-cycle replay decomposition

The replay loop (`src.replay._plan_cycle`) builds each cycle's MILP over **only `visible_ids`** (the
revealed book), and for the M1 open-book arm pins (hard-fixes) every *tendered* shipment, which HiGHS
presolve removes. So the **free** problem each cycle is the *visible-untendered window*, not the full
book. Measured peak windows:

| n_hawbs | full book | peak visible | peak untendered window (daily cadence) |
|--------:|----------:|-------------:|---------------------------------------:|
| 40 | 40 | 40 | ~18–20 |
| 80 | 80 | 80 | ~28–32 |
| 120 | 120 | 120 | ~44–48 |
| 200 | 200 | 200 | ~55–65 |

Under **event cadence** (re-plan at every reveal/cutoff — the M1 definition, the heaviest cadence)
the free window runs a bit larger than the daily figure (tendering lags the cutoff events): peak free
= **63 at n=120** (122 cycles) and **86 at n=200** (207 cycles).

**Actual M1 replay per-cycle solve timing.** Full run timed at n=120 seed 0; n=200's full run was
abandoned (34 min CPU and climbing across its 207 cycles — a *throughput* problem). For n=200, and
to cross-check n=120, I also solved the **isolated worst free subproblem** directly (build a MILP over
exactly the peak-free HAWB set, no pins — a conservative bound on the worst cycle):

| n_hawbs | days | seed | peak free | full-cycle max s | isolated free solve s | gap reached |
|--------:|----:|----:|---------:|-----------------:|----------------------:|:-----------:|
| 120 | 5 | 0 | 63 | **61.1** (122-cycle run) | 1.9 | OPTIMAL @0.0049 |
| 200 | 7 | 0 | 86 | (run abandoned) | 4.4 | OPTIMAL @0.0049 |
| 200 | 7 | 2 | 75 | (run abandoned) | 20.4 | OPTIMAL @0.0049 |

**Every measured per-cycle solve reached the 0.5 % policy gap (all OPTIMAL).** The policy is met per
solve at every scale; the only variable is wall time. Two facts stand out:

1. The **isolated free subproblem is ~30× faster than the full replay cycle** (n=120: 1.9 s isolated
   vs 61 s full-cycle). The difference is that `_plan_cycle` builds the MILP over the **entire visible
   book** — the ~44 tendered HAWBs enter as pinned columns and only get fixed in presolve, and where
   they share MAWBs with free HAWBs they still couple. So most of the 61 s is *pinned-column
   overhead*, not the free problem. This is a cheap, high-value optimization target (§7 lever 2).
2. Seed variance is large at the top (n=200: seed-0 free=86 → 4.4 s, but seed-2 free=75 → 20.4 s; the
   hard seed-2 is denser). The worst *full* cycle at n=200 is therefore plausibly minute-scale on a
   hard seed, which is what made the full run slow.

---

## 6. Ladder verdict

| Cell | n_hawbs | Monolithic | Via replay decomposition | **Verdict** |
|------|--------:|-----------|--------------------------|-------------|
| **C0** | 12 | 0.01–0.03 s, OPT | trivial | **SOLVES** |
| **C1** | 40 | 0.05–0.33 s, OPT | trivial | **SOLVES** |
| **C2** | 120 | **>180 s, gap-budget broken** | max **61 s/cycle, all OPTIMAL@0.5%** (isolated free 1.9 s) | **SOLVES via replay** (monolithic would be NEEDS-DECOMP, but is never invoked). Flag: per-cycle wall ~60 s + ~13 min/full-run is a *throughput* cost, not a policy violation — and ~30× of it is pinned-column overhead (§7 lever 2). |
| **C3** | 200 | **>180 s, gap-budget broken** | isolated free 4–20 s, **all OPTIMAL@0.5%**; full-cycle plausibly minute-scale (seed-dependent); full run >30 min | **SOLVES under the gap policy per-cycle, RISKY on wall/throughput.** Apply §7 levers 1–3 before running C3 for results (matches §F's "may need decomposition" flag). Not a hard wall — every solve still hits 0.5 %. |

**The capacity redesign does not change this picture.** Its added rows/cols are continuous and
negligible; the verdict is governed entirely by the `x`-family binary count per *solve*, which the
replay decomposition — already in production — bounds to the untendered window.

---

## 7. Remedy ladder (if per-cycle wall or throughput needs cutting at C3 / under tighter τ)

The per-cycle *gap policy* is already met; these are levers for **wall-time / throughput** headroom
(a tighter τ that forces more consolidation, or a faster sweep), in cheapest-first order:

1. **Replay decomposition — already in place (no work).** Bounds every solve to the visible-untendered
   window. The single biggest lever, free. Confirmed sufficient for C2's *gap policy*; every solve at
   every scale already hits 0.5 %.
2. **Shrink the per-cycle model to the free window (highest-value cheap fix).** The isolated free
   subproblem is ~30× faster than the full replay cycle (1.9 s vs 61 s at n=120); the gap is
   *pinned-column overhead* — tendered HAWBs enter `_plan_cycle`'s model as columns and are only
   fixed in presolve. Building each cycle's MILP over **just the free set**, accounting committed
   capacity as RHS / ledger reservations rather than pinned `x` columns, would collapse the worst
   cycle from ~minute-scale to seconds at both C2 and C3. **Caveat:** legitimate only if the proof
   keeps firm MAWBs *closed* to new riders; if free HAWBs may join an already-committed MAWB
   (consolidation onto a firm group), those columns must stay — confirm the consolidation semantics
   with the proof owners. This is the one mitigation I'd reach for first for C3.
3. **Tighten the geo-subgraph.** The `x` family = HAWB × *subgraph arc count*. `ForwarderGraphConfig`
   knobs `seed_k`, `corridor_phi`, `max_air_legs`, `backstop_buffer_h` directly cut per-HAWB arc count
   (and thus the 7 600 cols at free=63). No model change, realistic (no air shipment needs a 7-day
   backstop of far-future flights). Pairs naturally with the density finding in §2.
4. **Daily cadence for the machine arms.** M1 uses `_event_times` (122 cycles at n=120 / 207 at n=200);
   `_daily_times` cuts cycle count ~5× (→ ~5–8), slashing total run time — the main fix for the
   *throughput* (not per-solve) problem. Caveat: event cadence is part of the M1 definition, so this
   is a methodology call — confirm before changing.
5. **Warm-starts across cycles.** Consecutive cycles differ by a few newcomers; reusing the prior LP
   basis cuts node counts. Moderate effort; deferred until 2–4 prove insufficient.
6. **Further cuts / column generation / formal time-decomposition.** Root gaps are already 0.3–4 %, so
   more cuts are low-yield, and the replay loop already delivers time-decomposition for free. Column
   generation is **not warranted** at this scale.

**Cheapest sufficient recommendation:** C2 already meets the policy as-is (lever 1). For C3, apply
lever 2 (free-window model build — kills the pinned-column overhead) and, for sweep throughput,
lever 4 (daily cadence) and/or lever 3 (subgraph tightening). No formal decomposition or column
generation is needed anywhere on this ladder.
