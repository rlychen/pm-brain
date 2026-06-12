# Critique 12 — Simulation BUILD Review (4-agent, post-arrival-stream / pre-2c)

**Session 33, 2026-06-10. Status: COMPLETE.** Four adversarial agents (soundness/confounds,
falsifiability, OR-mechanism, realism) reviewed the **built** λ arrival stream (slices 1–4)
against the critique-11 decisions D-A9..D-A16 and the new build choices. The launch brief is
below (§ Brief); the consolidated findings are in § Findings. Critique 11 (S32) reviewed the
arrival-only **design**; this round reviews the **code**.

**Headline.** The *soundness* core is clean and confirmed in code: the headline draws lateness
tier-independently (D-A9), `A_k^min`/`arr_dest`/scorer-walk are arithmetically identical for a
committed route (D-A13), there is no lookahead leak, no double-spend identity, and CRN holds
across the κ/λ grid. The failures are **structural to the substrate + the κ dial**, all four
agents converging on the same three: (1) the κ axis is the integer-quantized ULD count D-A10
retired → no reachable binding/abundant cell; (2) the substrate cutoffs are degenerate and `d*`
anchors `known_at` to the wrong (origin, not binding-hub) cutoff; (3) the test has no expressible,
symmetric, pre-registered null. Plus two material substrate facts: only 2 of 6 lanes carry
capacity, and they're demand-starved. **Fix the code (κ + cutoffs) and capacitate/load before 2c.**

---

## § Brief (launch package)

## Why a second round (what's different from critique 11)

Critique 11 reviewed prose/methodology. The risks then were design-level. Now there is
running code with concrete parameter values and structural choices that critique 11 could
not see. Two questions this round must answer that critique 11 could not:

1. **Build fidelity** — does the code actually honor D-A9..D-A16, or did the build quietly
   re-introduce something a decision retired? (One suspected gap pre-listed below.)
2. **New surface** — the build made decisions critique 11 never saw (`d*` = origin-departing
   offer, the cutoff anchor, the `[CAL]` magnitudes, the substrate's capacity topology). Are
   any of them quietly load-bearing for L2?

## What is built (artifacts to review)

| Artifact | What it is |
|---|---|
| `src/components/flexibility.py` | 2-FLEX: `Tier`, single-source `TierSpec` table, `derive_deadline`, `classify` (`flex_k`), `cw_flex`. Pure. |
| `data/synthetic/tpeb_air_instance.py::build_tpeb_daily` | daily `D=7` substrate (tiles the single-cycle TPEB offers at 24h). |
| `data/synthetic/air_generator.py` Part B | `ArrivalConfig`, `generate_arrival_instance`, `HawbArrival`, `write_arrival_scenario`. The λ stream. |
| `src/components/air_graph.py::earliest_arrival` | the generator→graph `A_k^min` edge. |
| `data/synthetic/scenario_io.py` | arrival columns persisted to `shipments`; reveal view. |
| `SESSION_LOG.md` (S33 entry) | the build narrative + every `[CAL]` value committed. |

Governing docs (unchanged): `arrival_only_replan_methodology.md` (§10 arrival, §12 D-A9..D-A16),
`flexibility_model.md` v0.3, `backtest_methodology.md` v0.5.

## Build decisions made this session (the probe targets)

- **`d*` = a specific target departure (offer), anchored to its own cutoff** — resolves "which
  cutoff." Keyed by **origin gateway** (through-routed lanes have no full-O-D offer), preferring
  the contracted (BSA per-ULD) offer, else earliest-cutoff. `known_at = cutoff(d*) − B`.
- **Headline `B` is tier-INDEPENDENT** (`ArrivalConfig.tier_coupled_arrival=False` default) — a
  single `uniform(mean ± spread)`; `tier_coupled_arrival=True` is the D-A7 upper bracket.
- **`Δ_k` tier-derived** via `TierSpec.sla_offset_h` from `A_k^min` (`derive_deadline`).
- **`λ` (`lambda_compress`)** scales the *mean* of `B` toward the cutoff.
- **`[CAL]` magnitudes committed:** sla_offset 12/40/120h; z_tier 2/1/0.5; w_sp 4/2/1; θ_flex 24h;
  book-lead mean 48h ±24h; coupled means 12/48/96h; backstop +720h; D=7; tier mix 20/40/40.

## Pre-listed gaps the build is already suspected to have (confirm / refute / add)

- **G-A (build-fidelity, likely BLOCKING): the κ axis is still the integer-quantized one D-A10
  retired.** `_build_rate_catalog` keeps `n_uld = max(1, round(2.0 * capacity_scale))`
  (`air_generator.py`). D-A10 said retire this as the κ axis and dial κ in *binding-ness*
  (peak-concurrent-demand / slots). Until fixed, the required abundant-capacity negative-control
  cell (D-A10) can't be reached and "L2≈0 when slack" stays asserted, not demonstrated.
- **G-B: only HKG→LAX/ORD carries capacitated (BSA) supply.** The other 4 demand lanes are
  free-sale spot (uncapped), so κ-driven contention → L2 structurally only appears on 2 of 6 lanes.
  Is that a fatal narrowing of the test, or acceptable if reported honestly?
- **G-C: tractability.** ~47s solve on the 91-offer daily graph (30 HAWBs × 7 days). The 2c replay
  loop solves repeatedly (per cycle × arms × κ×λ grid × replications). Does the proof need a
  smaller substrate, warm-starts, or a column-count cut before 2c is feasible?
- **G-D: `flex_k`/`cw_flex` deferred.** The reporting denominator is not yet computed; the
  non-dominance filter needs per-route cost. Does deferring it risk an arm-dependent denominator
  later (D-F7 demands `t=0`, arm-invariant)?

## The four agents (reuse the critique-11 lenses — they converged well)

Each agent reads the artifacts + governing docs and returns findings ordered by severity, tagged
`[BLOCKING] / [MATERIAL] / [MINOR]`, each with a concrete fix. Each must explicitly rule on G-A..G-D.

1. **Simulation soundness / confounds.** Does the *built* stream smuggle in a confound (lookahead,
   double-spend-as-identity, denominator inflation)? Is the headline default genuinely the
   tier-independent draw (D-A9)? Does `earliest_arrival`-as-`A_k^min` match what the scorer walk
   will compute (one-time-scalar, D-A13)?
2. **Falsifiability / clear test.** Can this sim demonstrably FAIL? Is there a reachable
   abundant-capacity cell that must show `|L2| < CI` (D-A10)? Rule on G-A. Is the pre-registered
   null actually expressible against the built knobs?
3. **OR mechanism rigor.** Is `d*` / cutoff-anchor / `Δ_k`-derivation coherent? Does the
   origin-cutoff anchor distort `B` for through-routed lanes? Is the κ axis (once G-A fixed) a
   clean tightness dial? Any degeneracy in `classify`'s non-dominance / θ-separation logic?
4. **Operational realism.** Are the `[CAL]` magnitudes defensible bands (no fabricated mechanism)?
   Is "EXPRESS late / DEFERRED early" still only an upper bracket, never the headline? Is the
   HKG-only-capacity topology (G-B) representative enough to claim an air result?

## Cadence after the run

Findings → fold the BLOCKING/MATERIAL set into the methodology (new D-A## as needed) + fix the
code (G-A first) → THEN build 2c. Same design→critique→fold→build loop as critique 11.

---

## § Findings (4-agent, consolidated; ordered by convergence then severity)

Convergence = signal (count of independent agents who hit it). Each finding: severity, the
agents that raised it, the concrete fix with file/function cites.

### F1 — [BLOCKING] κ is the integer-quantized ULD count D-A10 retired (ALL 4 agents).

`air_generator.py::_build_rate_catalog` line ~169: `n_uld = max(1, round(2.0 * capacity_scale))`,
binding in the MILP via C.5 `Σ_g η ≤ N_{a,u}` per arc. Three compounding defects: (a) **non-monotone
dead zones** — banker's `round(2·s)` maps `capacity_scale ∈ (0.25,0.75)→1`, jumps at 0.75→2, so the
L2-vs-κ curve is a step artifact, not a tightness response; (b) **floored at 1 ULD** — cannot ration
into a genuinely binding regime *at resolution*; (c) **confounded with packing granularity** — κ moves
in 1500 kg LD3 quanta against a continuous demand stream, so κ ≡ ULD-integer-fit, not "binding-ness."
Net: the D-A10 **required negative-control cell** (abundant κ ⇒ `|L2|<CI`) is not cleanly constructible
and the **binding peak cell** is unreachable at fine resolution, so the headline `L2(κ)` scarcity claim
cannot be traced or falsified.
**Fix:** make κ a continuous binding-ness dial = peak-concurrent-committed-demand ÷ contracted slots.
Size the BSA `cap_a` continuously from the realized per-departure demand mass (`BsaContract` already has
a `cap` field; `FlatRate.cap` exists and is plumbed through persist/load but never set). Keep `n_uld`
only as a physical billing integer; decouple it from the κ axis. Edit site: `_build_rate_catalog` +
the allotment construction; κ becomes an `ArrivalConfig`/sweep field sized against generated demand.

### F2 — [BLOCKING] The cutoff anchor is broken — two facets (realism + OR-mechanism).

`d*` anchors the whole stream: `known_at = cutoff(d*) − B`, so the cutoff-vs-departure lead is
load-bearing. (a) **Degenerate substrate cutoffs** (realism B1): `mu_pvg_hkg` has cutoff 14h *after*
dep 12h (impossible); the CX BSA offers have **cutoff = departure (zero lead)** — and those are exactly
what `_target_offer` picks as `d*`, so the contended cell anchors to a zero/negative-lead cutoff. (b)
**Wrong cutoff for through lanes** (OR-mechanism): `_target_offer`/`_origin_offers_by_day` key `d*` on the
*origin gateway*, but for through-routed lanes (PVG/TPE→ORD via HKG) the binding scarce-capacity decision
fires at the *HKG hub* CX cutoff ~14h later, so `B` is systematically inflated (~14h) for those lanes and
λ compresses toward the origin, not the binding, cutoff.
**Fix:** derive every cutoff as `dep − L_cut` with `L_cut > 0` (methodology §10 specifies this form; the
substrate `tpeb_air_instance.py` lines ~146–183 don't honor it — pick a forwarder-consol `L_cut`, sourced).
Anchor `d*` to the HAWB's *binding* contracted leg (extend the pass-2 `earliest_arrival` call to return the
binding air arc on the min-arrival path and anchor to its cutoff), not the earliest origin offer.

### F3 — [BLOCKING] No expressible, symmetric, pre-registered null (falsifiability; soundness concurs).

D-A10's null ("peak-cell L2 CI straddles 0") is not evaluable — "peak cell" presupposes a κ axis F1
removed — and it is one-sided (mandates the abundant cell show `|L2|<CI` but sets no effect-size floor on
the positive claim, so a commercially-meaningless `[0.1%,0.4%]` CI "passes"). The favorable knobs
(`tier_coupled_arrival=True`, `lambda_compress` stacking) are reachable from the same function with no
guard preventing a coupled/compressed cell from being *published as the headline* — D-A9 is honored by
default value only, not structurally.
**Fix (fold as D-A17):** pre-register both tails — negative control `|L2_reshuffle| < CI` at the slack
cell (else *reject the build*); positive claim `L2_reshuffle` CI lower bound `> τ` (pre-registered min
effect size, e.g. 2% of C(M₀)) AND reshuffle-share ≥50% at the peak cell (else *thesis unsupported*);
`L2_reshuffle(κ)` monotone non-increasing in κ. Add a `cell_role ∈ {negative_control, peak, headline,
upper_bracket}` field to `ArrivalConfig`, persist it to `config.json`, and assert at scenario-write time
that `tier_coupled_arrival=True ⇒ role=upper_bracket` so the favorable number cannot be the headline.

### F4 — [MATERIAL] G-B: only 2 of 6 lanes carry capacity, and they're demand-starved (ALL 4).

Only `cx_hkg_lax`/`cx_hkg_ord` (PER_ULD_PIVOT) are capacitated; the other 4 lanes carry no `cap` into the
MILP. With ~30 HAWBs / 6 lanes / 7 days ≈ 0.7 HAWB/day on each HKG lane (mean weight ~520 kg) vs the
floored 1500 kg ULD, the capacitated lanes are *typically slack even at minimum κ* → the negative control
is near-automatic and the positive cell is contention-starved (few binding-lane flexible HAWBs → CIs won't
separate, M-B7). Compounds F1.
**Fix:** capacitate ≥1 origin-diverse lane (set `FlatRate.cap` on a BR/MU offer — one-line, the field is
already plumbed) and/or model the shared HKG→US CX leg as the resource through-routed TPE/PVG cargo also
competes for; add a `lane_mix` knob to `ArrivalConfig` so the peak cell can concentrate demand on
capacitated lanes; build the M-B5 roll-to-next-contract-flight option so the no-replan failure is
"late-but-cheap," not always spot-3×. Report L2 by lane; scope the headline to "contracted-capacity lanes"
until origin-diversified.

### F5 — [MATERIAL] G-D: `cw_flex` deferral is clean today, but the schema can't reconstruct the t=0 frozen denominator (ALL 4).

`classify`/`cw_flex` have zero callers outside `flexibility.py` (deferral is clean, no arm-dependent leak
*today*; the component is arm-invariant and un-inflatable by construction). But `_persist_hawbs` drops
`target_offer_id` and `t_dead_at` from `HawbArrival`, so the t=0 `cw_flex` can only be regenerated from the
in-memory `ArrivalScenario`, never from `scenario.db` alone — and D-F7 requires it frozen at t=0 and
arm-invariant.
**Fix (with 2c):** compute `cw_flex` once at generation time over the pre-predicate-9 option set *with
cost* (the pass-2 `arcs/adjacency` are already in hand), persist it as a scenario-level scalar in
`config.json`; 2c reads the frozen value for every arm. Add the D-F7 bit-identical-across-arms pytest as a
2c gate. Populate `RouteOption.cost` from the same enumeration the MILP sees.

### F6 — [MATERIAL] B5: DEFERRED's 120h `sla_offset` manufactures flexibility (realism).

`TIER_SPECS[DEFERRED].sla_offset_h = 120` (5 days slack over fastest transit). With θ_flex=24h daily, this
makes DEFERRED reach `d*`..`d*+5` — *structurally* flexible by construction, and DEFERRED is 40% of the mix
and the bump engine's slack source. It drives the `cw_flex` denominator, so the number partly decides the
answer.
**Fix:** source DEFERRED's realistic slack-over-fastest-transit in the calibration note before it carries
the headline; run the flex-model §7 sandbag sensitivity precisely on this knob — if L2 collapses when the
offset is halved, that's a finding. Ordering invariant (E<S<D) is fine; the *level* is the issue.

### F7 — [MINOR, gating] G-C: 47s × replications × grid can't yield CIs (falsifiability/OR/soundness).

Tractability is a *falsifiability* prerequisite (need `R` replications × ≤5 arms × κ×λ grid for CI
half-width < L2/4, M-B7). **Fix:** window-prune each HAWB's subgraph to a ±2-day departure window around
`d*` (the `flex_k` θ-logic already implies it); warm-start the per-cycle re-solve. **Soundness caveat:**
per-arm warm-starts are a determinism hazard (tie-break leakage) — gate them behind the D-A11 `M₁'`
invariant `C(M₁')==C(M₀)` and keep HiGHS `threads=1`+`random_seed`.

### F8 — [MINOR] `t_dead_prob` desyncs CRN (soundness).

The conditional `rng.uniform(48,120)` for `t_dead_at` is short-circuited when the Bernoulli fails, so
sweeping `t_dead_prob` off its default 0 desyncs the demand stream downstream. Headline is `t_dead_prob=0`
so unaffected today. **Fix:** always-consume the uniform (draw then discard) or document `t_dead_prob` as a
non-CRN axis.

### Confirmed SOUND (stated plainly, so the fold doesn't churn them)

- **D-A9 headline tier-independence** — `tier_coupled_arrival=False` default; only `Δ_k` is tier-coupled on
  the headline path. (soundness, realism)
- **D-A13 walk ≡ scalar** — `earliest_arrival`'s `eta + Σ dest-chain delta_h` is arithmetically identical to
  the MILP `arr_dest`; hub dwell correctly precedes the terminal arc and is excluded from `delta_post`. (soundness)
- **No lookahead leak** — `known_at`/`ready` anchored to past info; `d*`/`t_dead` not persisted so the replay
  view can't peek; frozen actuals pre-sampled `s=0` in canonical order, route-independent. (soundness)
- **No double-spend identity** — daily `#d{day}` tiling makes each departure a distinct, separately-capacitated
  arc. (soundness)
- **CRN sound** (except F8) — named sub-streams isolate κ (rates) from demand; λ changes only the args to one
  `uniform` call. (soundness)
- **Deadline chain non-circular** — `A_k^min` via `_propagate_forward` reads nothing tier-derived; two-pass
  split honors flex-model §2.1; `Δ_k` telescoping exact. (OR-mechanism)
- **`classify` correct** — strict-on-both non-dominance, θ-separation, σ=0 z_tier collapse, empty-options
  fallback corner all match the spec; no `flex_k` mis-label or `cw_flex` inflation in the component. (OR-mechanism)

---

## Fold / fix sequencing (proposed)

1. **Code, before 2c (BLOCKING):** F1 continuous-κ dial + F2 cutoff derivation & binding-leg anchor. These
   two unblock the negative-control cell and de-bias `B`. F4's one-line `FlatRate.cap` + `lane_mix` rides with F1.
2. **Methodology fold:** F3 → new **D-A17** (symmetric null + τ + `cell_role` guard); annotate F6 (DEFERRED slack
   is `[CAL]`, sandbag-gated) and F2's `L_cut` as calibration-note items.
3. **Cheap code now:** F8 (always-consume the uniform).
4. **With 2c:** F5 (persist frozen `cw_flex` + D-F7 pytest), F7 (window-prune + gated warm-start).
5. **Calibration note (pre-Stage-3):** source `L_cut` (F2), DEFERRED slack (F6), contracted-vs-spot share &
   which lanes (F4).

---

## § Numeric walkthroughs (F1 / F2 / F3)

### Shared picture — what the proof measures, and where each finding breaks it

The replan-savings proof sweeps a 2-D grid of scenarios and, in every cell, reports **one number**:

```
   L2  =   C(M₀)    −    C(M₁)
         no-replan      open-book
         (greedy pin)   re-optimize
```

The grid's two axes:

- **κ (capacity tightness)** — how scarce the contracted slots are relative to demand. Slack → tight.
- **λ (arrival lateness)** — how late, relative to the cutoff, demand is revealed to the planner.

The claim under test: **L2 grows as capacity tightens (↑κ) and as demand reveals later (↑λ), and L2 ≈ 0
when capacity is abundant.** Each of the three BLOCKING findings breaks one piece of that machinery:

| Finding | What it breaks | One-line symptom |
|---|---|---|
| **F1** | the **κ axis** | capacity moves in integer-ULD jumps → no smooth dial, no reachable binding *or* abundant cell |
| **F2** | the **λ / timing axis** | `known_at` anchored to the wrong cutoff → reveal-to-tender lead `B` mis-measured (~14h on through lanes) |
| **F3** | the **decision rule** on L2 | even on a clean grid, a trivially-small L2 "passes" and a rigged-favorable L2 can be the headline |

All numbers below use the S33 substrate: **30 HAWBs, 6 lanes, 7 days**; HAWB weight ~ triangular(50, 1200, 300)
→ mean ≈ **517 kg**; one LD3 ULD = **1500 kg**; book-lead `B ∈ [24, 72]`h (mean 48 ± 24).

---

### F1 — the κ axis is quantized: capacity can't be tightened or loosened smoothly

**Claim.** Slot capacity is set by an integer ULD count, `n_uld = max(1, round(2.0·capacity_scale))`
(`air_generator.py::_build_rate_catalog`). That integer is the *only* knob on the κ axis, and it has three
defects that compound.

**Step 1 — see the quantization.** Python's `round` is round-half-to-even, so the `capacity_scale` knob lands
on capacity in lumpy 1500 kg steps:

| `capacity_scale` | `2·scale` | `round` | `n_uld` | cap/day |
|---|---|---|---|---|
| 0.25 | 0.5 | 0 | **1** (floor) | 1500 kg |
| 0.5 | 1.0 | 1 | 1 | 1500 kg |
| 0.7 | 1.4 | 1 | 1 | 1500 kg |
| 0.75 | 1.5 | 2 | 2 | 3000 kg |
| 1.0 | 2.0 | 2 | 2 | 3000 kg |
| 1.5 | 3.0 | 3 | 3 | 4500 kg |

**Step 2 — defect (a), dead zones.** Sweeping `capacity_scale` 0.3 → 0.7 changes *nothing* (still 1 ULD, 1500
kg); then it jumps a full 1500 kg at 0.75. An L2-vs-κ curve built on this knob is a step function — flat, flat,
cliff. That's an artifact of integer rounding, not a tightness response.

**Step 3 — defect (b), floored at 1.** `max(1, …)` means you can never go below one ULD = 1500 kg/day. You
cannot ration capacity into a genuinely binding regime at the resolution the sweep needs.

**Step 4 — defect (c), demand never reaches even the floor.** Each HKG lane sees ≈ 30 ÷ 6 ÷ 7 ≈ **0.7 HAWB/day**
at ~517 kg. To fill one ULD (1500 kg) needs ~3 HAWBs on the *same* departure; at 0.7/day that essentially never
happens. So at default `scale=1.0` (3000 kg/day) capacity is wildly slack — and even at the 1500 kg floor it's
still slack. **There is no reachable binding cell, and no way to separate "abundant" (the negative control) from
"tight" — the entire κ axis collapses to one regime.**

**The fix — define κ as a continuous ratio, then size capacity to hit it.** Make κ a real dial,
`κ = peak concurrent demand ÷ contracted slots`, and size the slots from the demand the generator actually
produced:

> **Worked example.** Suppose 3 HAWBs target the `cx_hkg_lax#d2` departure, totaling **1,600 kg**. Set
> `cap_a = peak_demand / κ`:
> - κ = 1.0 → cap = 1,600 kg → **exactly binding** (demand == capacity)
> - κ = 0.8 → cap = 1,280 kg → **oversubscribed**: 320 kg must spill to spot/roll (a tight cell)
> - κ = 2.0 → cap = 3,200 kg → **slack** → the negative-control cell that must show L2 ≈ 0

κ is now continuous, monotone, and placeable anywhere on the axis. `n_uld` survives only as a physical billing
integer, decoupled from κ. (`BsaContract.cap` / `FlatRate.cap` are already plumbed through persist/load — they're
just never set.)

---

### F2 — the cutoff anchor is broken: `B` is measured against the wrong clock

**Claim.** The whole arrival stream hangs off one equation:

```
   known_at  =  cutoff(d*)  −  B
```

`known_at` is when the planner first sees the HAWB; `B` is the reveal-to-cutoff lead — the **information-timing
lever the proof sweeps** (it's what λ compresses). So *which cutoff* `d*` anchors to decides what `B` actually
measures. Two things are wrong.

**Facet (a) — the substrate cutoffs are degenerate.** A cutoff must precede its departure
(`L_cut = dep − cutoff > 0`). The S33 substrate violates this:

| offer | dep | cutoff | `L_cut = dep − cutoff` | problem |
|---|---|---|---|---|
| `mu_pvg_hkg` | 12 | 14 | **−2h** | cutoff *after* departure — impossible |
| `cx_hkg_lax` | 28 | 28 | **0h** | zero lead — and `_target_offer` picks this BSA as `d*` |

For an HKG→LAX HAWB, `d* = cx_hkg_lax` (cutoff 28), so `known_at = max(0, 28 − B)` with B∈[24,72] →
`known_at ∈ [0, 4]`h. The HAWB is "revealed" essentially at t=0, against a zero-lead cutoff — there's no
meaningful lead window for λ to compress.

**Facet (b) — through lanes anchor to the wrong cutoff** (the load-bearing one). Walk a **PVG→ORD** HAWB
through both clocks:

```
          d* anchors HERE                         binding decision HERE
          (origin cutoff)                         (hub BSA cutoff)
                │                                         │
   t=0          │ 14                            28        │ 30
    ●───────────┼──────────────────────────────┼─────────┼────────▶  time (h)
    │           │                              │         │
 known_at    mu_pvg_hkg                  cx_hkg_ord    cx_hkg_ord
 = 14 − B    cutoff = 14                 cutoff = 28   dep = 30
             (PVG origin offer)          (HKG→ORD BSA = the scarce slot)

  route:  PVG ──mu_pvg_hkg──▶ HKG ──dwell──▶ HKG ──cx_hkg_ord──▶ ORD
                (spot leg)                        (CONTRACTED, capacitated)
```

Step by step:

1. `_origin_offers_by_day` keys `d*` on the **origin gateway, PVG** → candidates `{mu_pvg_lax, mu_pvg_hkg}`,
   both cutoff **14**, neither contracted. So `d*` cutoff = 14 → `known_at = 14 − B`.
2. But the HAWB routes PVG→HKG→ORD, and the **scarce contracted capacity it competes for is the CX HKG→ORD BSA,
   cutoff 28.** That's where the irreversible tender / slot decision actually fires.
3. So the real reveal-to-binding-decision lead is `28 − known_at`, yet `B` was drawn against **14** → **`B` is
   understated by 14h** for through lanes, and λ compresses toward the *origin* cutoff, not the binding one.

**The fix.**
- Derive every cutoff as `cutoff = dep − L_cut` with a sourced forwarder-consol `L_cut > 0` (e.g. 4h →
  `cx_hkg_lax` cutoff 24, `mu_pvg_hkg` cutoff 8). Methodology §10 already specifies this form; the substrate
  (`tpeb_air_instance.py` ~L146–183) doesn't honor it.
- Anchor `d*` to the HAWB's **binding contracted leg** (the CX HKG→ORD cutoff), not the earliest origin offer.
  Extend the pass-2 `earliest_arrival` call to also return the binding air arc on the min-arrival path, and
  anchor to *its* cutoff.

---

### F3 — the null isn't symmetric or pre-registered: the test can't demonstrably fail

**Claim.** D-A10's null ("the peak-cell L2 CI straddles 0") can't do its job, in two distinct ways — both let
junk through.

**Failure 1 — a trivially small L2 "passes."** The current rule is one-sided: it only asks "does the CI exclude
0?", with no floor on *how big* L2 must be.

> **Worked example.** Peak cell measures `L2_reshuffle = $1,200`, 95% CI `[$300, $2,100]`, against
> `C(M₀) = $90,000`.
> - **Today:** CI excludes 0 → "passes." But $1,200 is **1.3% of cost** (CI lower bound 0.3%) — commercially
>   meaningless.
> - **With a pre-registered floor** `τ = 2% of C(M₀) = $1,800`: the test asks "is the CI *lower bound* > τ?" →
>   `$300 < $1,800` → **fails → thesis unsupported.** That's the test doing its job.

**Failure 2 — a rigged-favorable L2 can be published as the headline.** The favorable knobs
(`tier_coupled_arrival=True`, `lambda_compress` stacking) are reachable from the same generator call, with
nothing structurally stopping a coupled/compressed cell from being reported *as* the headline.

> **Worked example.** The independent draw (the D-A9 headline) gives `L2 = $800`. Turning on tier-coupling
> (EXPRESS-late / DEFERRED-early — a deliberately favorable arrival shape) gives `L2 = $5,000`. Nothing today
> bars reporting the $5,000 as "the air replan savings." D-A9 is honored only by a *default value*, not by
> structure.

**The fix → fold as D-A17 (a symmetric, pre-registered null).** Pre-register *both* tails before running:

- **Negative control** (abundant-κ cell): must show `|L2_reshuffle| < CI` — capacity is slack, replanning can't
  help. If it shows savings → **reject the build** (the sim is manufacturing value).
- **Positive claim** (peak cell): CI **lower bound > τ** (pre-registered min effect size, e.g. 2% of C(M₀))
  **AND** reshuffle-share ≥ 50%. Else → **thesis unsupported.**
- **Monotonicity:** `L2_reshuffle(κ)` non-increasing in κ (tighter ⇒ more savings).
- **`cell_role` guard:** add `cell_role ∈ {negative_control, peak, headline, upper_bracket}` to `ArrivalConfig`,
  persist it to `config.json`, and assert at scenario-write time that `tier_coupled_arrival=True ⇒
  role=upper_bracket` — so the favorable number is *structurally barred* from being the headline.

**Why the fold order is F1 → F4 → F2 → F3.** F3's "peak cell" and "negative control" presuppose a real κ axis
(F1) with capacitated lanes carrying enough demand to bind (F4); only then is the cutoff/timing de-biasing (F2)
meaningful; and D-A17's decision rule is written against that fixed grid.
