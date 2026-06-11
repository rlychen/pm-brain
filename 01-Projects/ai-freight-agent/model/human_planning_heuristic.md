# Human Planning Heuristic — the `H₀` Baseline

**Status:** Draft v0.1 (Session 28, 2026-06-05). **Gate: G-Method.** Defines the human-planner
baseline policy (`H₀`) used by the replan-savings backtest (`model/backtest_methodology.md §3`).
Reusable as the human baseline for other modes (ocean, etc.) later.

## Purpose

`H₀` is the "what a competent human actually does" arm in the four-arm decomposition. It must be
a **fair** baseline: not a strawman (too dumb → inflates the model's win) and not unrealistic
(too smart → no human plans like that by hand). The governing constraint: **it must be executable
manually in a spreadsheet** — so no MILP-grade joint optimization, no foresight, no rolling
re-optimization.

Its role in the decomposition:
- **L1 = C(H₀) − C(M₀)** isolates *solver quality* (MILP vs. spreadsheet) by giving `M₀` the
  **identical commitment timing and recourse behavior** as `H₀` — the only difference is the
  planner. So `H₀`'s commitment/recourse rules below are also `M₀`'s.

## The heuristic (spreadsheet-executable)

1. **Route by service buffer (sets the frozen promise).** When a HAWB arrives, pick the route
   whose point-estimate (ETA) arrival clears the deadline by a fixed per-tier buffer (e.g.,
   express ≥ 0 days slack vs. the next valid flight; standard ≥ 1 day). Tie-break to cheapest.
   One lookup per HAWB. This choice sets `H₀`'s **OTP promise**, frozen at booking
   (`backtest_methodology.md §6`).
2. **Stage, don't finalize.** Tag the HAWB to a gateway + target flight but hold it in a staging
   list; do not build the MAWB yet. (Operators hold HAWBs until the flight's cutoff.)
3. **Cutoff sweep — the one consolidation step.** At each flight's cutoff, take all staged HAWBs
   for that gateway/flight and pack them into MAWBs greedily by a **simple density-mix rule**
   (sort by chargeable weight; fill the cheapest contracted/BSA ULD slot first, then the next).
   No foresight — only HAWBs already arrived and targeting this cutoff are considered. **Never
   hold a scarce slot empty for a future urgent shipment** — that foresight is exactly the L2 edge
   being measured, and it would be lookahead.
4. **Allocation = cheapest-first, FCFS.** Consume contracted/BSA capacity before spot, in arrival
   order, until exhausted. Overflow → roll-to-next-flight-on-contract (late but cheap) or spot.
   This FCFS rule is the realistic source of the canonical failure (the early flexible HAWB burns
   the cheap slot the later urgent HAWB needed).
5. **Break-only recourse.** Replan a HAWB *only* when its committed flight is cancelled or its
   realized ETA breaks the connection/deadline. Recovery = cheapest feasible alternative for that
   one shipment. No proactive OTP- or cost-chasing (too much manual work; the operator is not
   actively managing a moving book).

## Realism toggle (bracket the baseline)

To pre-empt "your human is a strawman" from both directions, run `H₀` in two variants and report
the bracket:
- **Inert** — exactly steps 1–5 (commit-and-forget except on breakage).
- **Diligent** — additionally, at a cutoff, re-shift a *still-staged* (not-yet-tendered) HAWB to a
  cheaper flight if a better option is visible *at that moment*. Still **no forward-looking
  slot-holding**.

The honest line `H₀` draws: it optimizes **locally at each cutoff, FCFS on cheap capacity, no
foresight**; `M₁` optimizes **globally with recourse over the not-yet-tendered book**. That gap —
not "the MILP packs ULDs better than a person" (which is L1, measured separately) — is the replan
value the thesis claims.

## What this is NOT

- Not an MILP. No joint optimization across flights/HAWBs beyond the single greedy cutoff sweep.
- Not forward-looking. No anticipatory capacity holding; no probabilistic reasoning over the
  transit distribution (uses point-estimate ETAs + a fixed buffer).
- Not continuously managed. Reacts only to hard breakage between cutoffs.
