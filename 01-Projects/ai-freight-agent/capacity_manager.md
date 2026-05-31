# Capacity Manager — Design Stub

> ⚠️ **TO BE REVIEWED** — stub pending user review; not approved. Heuristic
> (esp. the `B_c` budget form, `n_rem` definition, overage stance, and `γ_c`
> default) is under review and may change.

**Status:** Stub (Phase 1) — **TO BE REVIEWED**, not approved. Design doc, not
LaTeX (controller + pacing policy, not a monolithic MILP) — same doc class as
`rules_engine.md` and `density_fit_feasibility.md`.

**Architecture role:** Layer-3 intelligence component, "capacity manager
(BSA / NAC budgeting)" in `architecture.md` §11. It is the **only** component
that sees the whole allocation period; the mode optimizers (air first) are
myopic per-batch solvers. The capacity manager maintains contracted-capacity
consumption state across the rolling horizon and emits **per-batch control
inputs** the MILP reads as exogenous parameters.

## Scope

**MVP (this stub):**
- Equalized-settlement BSA contracts: maintain the consumed-weight accumulator;
  emit `A_c` (remaining allowance, air model C.13a) and `cap_a` (per-batch
  pacing throttle, air model C.5c).
- The simple pacing heuristic below.

**Deferred (noted, not built here):**
- `N_{a,u}` free-sale authorization — letting usage exceed the *firm* per-flight
  allotment (C.5) when behind. MVP only throttles at/below the firm allotment
  via `cap_a`.
- Volume-kicker `δ_c` price signal — same controller pattern (see air model
  `item:volume-kicker-accumulator`); fold in once built.
- NAC budgeting; per_flight-settlement BSA needs no period state (bounded
  per-flight by C.5).
- **Forecast-based pacing** — allocate scarce BSA capacity across remaining
  flights against a demand forecast (the real optimization version). The MVP
  heuristic is the placeholder.

## State (per equalized contract `c`, period `t`)

| Symbol | Meaning | Source |
|---|---|---|
| `P_c` | period commitment (kg); take-or-pay base | contract data |
| `consumed_c` | chargeable weight tendered to date this period (kg) | accumulator state, persisted per (tenant, contract, period) |
| `A_c = P_c − consumed_c` | remaining allowance (kg; may be ≤ 0) | derived |
| `n_rem` | remaining solve opportunities in the period (≥ 1) | orchestrator cadence + period calendar |
| `γ_c ≥ 1` | flexibility factor (front-loading tolerance) | tenant / contract config |

## Pacing heuristic (per batch, per equalized contract `c`)

```
A_c⁺  = max(0, A_c)
B_c   = min( A_c⁺ ,  γ_c · A_c⁺ / n_rem )      # per-batch free-capacity budget
```

Then for each BSA arc `a` of contract `c` present in this batch, set the
per-offer cap `cap_a`:
- single contracted arc in the batch: `cap_a = B_c`;
- multiple arcs of the same contract: split `B_c` across them in proportion to
  physical allotment `Σ_u N_{a,u}·W_u` (MVP allocation; refine later).

Emit `A_c` (C.13a free segment) and `cap_a` (C.5c) to the mode solve.

**Fraction of remaining used this batch** = `min(1, γ_c / n_rem)`.
- `γ_c = 1` → strict even pro-rata: `1/n_rem` of remaining per batch.
- `γ_c → n_rem` → may pull all remaining into one batch (no pacing).
- in between → may front-load a heavy batch up to `γ_c ×` the even share, while
  still reserving capacity for later flights.

**Emergent behavior (no extra rules):**
- Behind / under-consuming → `A_c⁺` large vs `n_rem` → bigger budget → catches up.
- Near period end → `n_rem` small → releases full remaining → no stranded sunk
  capacity.
- Hot / over-consumed → `A_c ≤ 0` → `B_c = 0` → BSA throttled, demand spills to
  spot / co-load / fallback.

**Post-solve update:** `consumed_c += realized chargeable(c)`; `n_rem −= 1`.

### Worked example
`P_c = 100,000` kg, `n_rem = 30` solves, `γ_c = 2`:

| Solve | `A_c⁺` | `n_rem` | `B_c` | note |
|---|---|---|---|---|
| 1 | 100,000 | 30 | 6,667 | ~6.7% of remaining released |
| 2 (tendered 5k) | 95,000 | 29 | 6,552 | heavy batch capped, reserves rest |
| … near end | 20,000 | 2 | 20,000 | releases all remaining — no stranding |
| over | 0 | 8 | 0 | BSA throttled, diverts to spot |

## Interface

- **Inputs:** contract master (`P_c`, `N_{a,u}`, `r_c`, period calendar), the
  batch's candidate BSA arcs, orchestrator cadence.
- **Per-solve outputs:** `A_c`; `cap_a` per BSA arc.
- **Persisted state:** `consumed_c` per (tenant, contract, period).

## Gates
- Math/methodology-first: this doc approved before any build (hard gate).
- Isolation: the air MILP is isolation-tested with `A_c` / `cap_a` as **fixed
  exogenous fixtures**, independent of this component — so the air build is not
  blocked on it, and the isolation gate is preserved.

## Open questions
- `γ_c` default (tenant config; `MARKET RESEARCH NEEDED`) — reasonable range
  `[1, 3]`?
- `n_rem` definition: remaining scheduled solves vs. remaining contracted
  flights in the period.
- Controlled overage: should `cap_a` ever be set above `A_c` (allow priced
  overage when beneficial), or is throttling to free capacity the right MVP?
- Multi-arc-per-contract `B_c` allocation rule.
- Fold-in of `N_{a,u}` free-sale authorization and volume-kicker `δ_c`.
