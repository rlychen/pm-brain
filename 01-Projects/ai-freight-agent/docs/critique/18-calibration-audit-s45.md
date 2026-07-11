# Calibration / Assumptions Audit — Session 45 (post Slice-C + 2c replay + 2c-7 recourse)

**Auditor role:** Assumptions / Calibration Auditor (1 of 3 standing proof-hardening agents). This
is the S45 run; last ran S36 (`docs/critique/14-calibration-audit-s36.md`).
**Scope:** read-only (this report is the only written artifact). Headline result under audit:
**L2 = C(M₁') − C(M₁)** (intra-engine, Reading B — superseding the S36 `C(M₀)−C(M₁)` framing),
swept over the **(κ, α, λ)** grid; L1 = C(H₀) − C(M₁').
**Calibrated to:** S44 — F1 Slice C BUILT (spot cap + two-sided rate + κ-independent `spot_regime`
stream); full 2c replay machinery (`src/replay.py`, all 5 arms + scorer); 2c-7 disruption recourse
slices 1+2a with the recourse tardiness weights W=(1.3, 0.65, 0.32); H₀ daily cadence;
`mip_rel_gap=0.005`.

**Verdict in one line:** the dominant $ knob (Slice C spot pricing) is now real code — but as built it
**contradicts its own governing spec**: the methodology mandates the spot:contract gap be a
*regime-mixture tied to κ* (loose→soft, tight→peak), and the code draws it on a deliberately
**κ-independent** stream as a flat `Uniform(0.85, 1.18)`. That is the single most dangerous finding
this session: it both decouples the dominant $ lever from the swept axis (so the κ-sweep no longer
moves the spot premium that L2 monetizes) *and* leaves the gap's center adversary-pickable. The S36
F1 (α unanchored) survives unfixed; F4 (fallback drift) is genuinely **resolved** by the
`FallbackPolicy` rewrite; the new recourse weights (1.3/0.65/0.32) are an undocumented, untested,
unfalsifiable anchor that is currently inert on the headline (W=0) but load-bearing the moment the
§6 sensitivity arm is reported.

---

## Method note

Enumerated every `[CAL]`, `CALIBRATION NEEDED`, hard-coded band, and uncited magnitude across:
`arrival_only_replan_methodology.md` (§6/§6.1, §10, §12, §13 v4, Reading-B §4), `backtest_methodology.md`
(§7), `flexibility_model.md`, `precommitted_sla_deadline_proposal.md`, `air_transit_time.py`,
`air_generator.py`, `air_milp.py` (gap stop, spot cap, C.10 PWL, fallback), and `src/replay.py` (the
new surface: arms, H₀ cadence, `_RECOURSE_W`). Each rated by **headline-sensitivity** = how much L2
(the *difference* C(M₁')−C(M₁)) moves when the knob is swept within a defensible range; flagged
**unfalsifiable** if its value can produce the desired answer with no external anchor.

The discipline question (§13 D-A10): the loose-corner `|L2| < CI` gate is an honest null but it only
constrains the abundant-κ × even-α × early-λ corner. Nothing constrains where the *peak* sits. As at
S36, several knobs inflate the peak without ever violating the loose-corner null — and Slice C, now
built, adds a new way to do it (a κ-decoupled premium that the sweep cannot null out).

---

## Ranked findings (highest headline-risk first)

### F-A — Slice C spot gap is built κ-INDEPENDENT, contradicting the spec that ties it to κ. **[HIGH, SPEC-VIOLATION / UNFALSIFIABLE]**
*(This is F3 from S36, now built — and built wrong relative to its own approved methodology.)*

`backtest_methodology.md §7` (lines 394–404) is explicit and underlined: *"Spot-vs-contract gap = a
regime-mixture tied to κ, not a free scalar (the dominant $ knob)... regime **tied to the swept κ**
(loose κ → soft → <1; tight κ → peak → >1)."* §13 D-A19 repeats it: spot rate = base × `m`, `m` from
the two-sided band.

The **built** code (`air_generator._draw_spot_regime`, lines 307–322) draws
`mult = Uniform(0.85, 1.18)` on a dedicated **`spot_regime` RNG stream that takes no κ argument**, and
the docstring states the intent as the *opposite* of the spec: *"on a stream separate from supply...
so the two-sided price + capacity stay byte-stable as κ/α/λ sweep — the κ axis then moves contracted
tightness ONLY (no spot confound)."* So:

1. **The dominant $ knob is now decoupled from the swept axis.** The methodology's whole L2-dollar
   story is "tight κ ⇒ peak spot ⇒ the avoided premium is large." As built, tightening κ raises
   contracted *scarcity* (fewer ULD positions ⇒ more spill to spot) but does **not** raise the spot
   *price* — the premium that spill pays is a flat draw centered at `E[m] = 1.015`, the same at κ=1 as
   at κ=4. The L2 the sweep reports therefore captures *only* the quantity-of-spill channel, not the
   price-of-spill channel the spec says dominates. This is not a smaller effect; it removes the lever
   the methodology calls "the most controllable lever on the L2 dollar figure."
2. **The gap center is adversary-pickable.** `E[m] ≈ 1.015` makes spot ≈ contract on average, which
   *deflates* the dollar L2 (conservative — good). But nothing external pins 1.015 vs. the spec's
   tight-cell ~1.18. Whoever picks the band picks the headline dollar magnitude with no anchor, and
   the loose-corner null never sees it (it nulls the *quantity* channel at abundant κ, not the price).

**Reconcile direction — two honest options, pick one and document:** (a) implement the spec — make
`m`'s regime mean a function of κ (loose κ draws from a soft-centered band <1, tight κ from a
peak-centered band >1), accepting that this re-introduces a κ↔spot coupling *by design* (the
methodology's intent), with the CRN gate then protecting only demand vs. supply, not spot; or (b)
formally **amend the methodology** to drop the κ-tie and report L2 with spot held at a fixed,
sourced, conservative gap — and state plainly that the dollar headline therefore *excludes* the
peak-season price channel and is a lower bound on the avoided premium. Today the code does (b) while
the spec says (a), and no decision record reconciles them. **This is the single most important thing
to fix before the sweep is trusted** (see closing section). Provenance for either path is already
sourced: WorldACD/avitrader Nov-2024 (~1.18 peak) + Xeneta/Supply-Chain-Dive (soft ~0.85), URLs in
`backtest §7`.

### F-B — α (Dirichlet lumpiness) is still an unanchored L2 amplifier; nothing changed since S36. **[HIGH, UNFALSIFIABLE]**
*(S36 F1, still open.)* `GenConfig.alpha` / `_draw_network_supply` (lines 277–304) is unchanged: low
α piles ULD positions onto a few flights ⇒ severe idle-here/spill-there mismatch ⇒ large L2; high α ⇒
even spread ⇒ L2 → 0. It remains the most direct manufacture-or-destroy knob, with **zero external
anchor** (the docstring still just says "low = lumpy"). The loose-corner gate nulls only the *even-α*
corner; it says nothing about the lumpy corner where the headline is reported. The S36 mitigation
(anchor α's *range* to an observable capacity-dispersion HHI/Gini from BTS FAF lane flow + OpenSky
freighter-frequency dispersion, and **lead with the L2(α) curve, not a peak cell**) is still the
required fix and is still unbuilt. `BUILD_STATUS` Open-items list "α grid + external anchor" as a
`[CAL]` to source — so it is tracked but not closed. Under D-A24 region→region, where the mismatch
*bites* is now an emergent (α × per-airport trucking-matrix) property, which makes the α anchor *more*
load-bearing, not less.

### F-C — The recourse tardiness weights (1.3 / 0.65 / 0.32) are an undocumented, untested, unfalsifiable anchor. **[HIGH on the §6 arm, SILENT — currently inert on the headline]**
New this session (`replay.py` `_RECOURSE_W = {1: 1.3, 2: 0.65, 3: 0.32}`, lines 414–417). The prompt
asks me to audit this specifically. Findings:

- **The anchor exists only as three magic constants + a docstring claim.** The claim is "EXPRESS
  full-span penalty ≈ the fallback gap (~$16.4k)." I can confirm the *mechanism* (the weight is
  stamped FLAT onto `tardiness_weight`, and C.10 bills `pen ≥ W·τ²` with `τ ≤ span = T^abs − Δ_k`, so
  EXPRESS full-span = `1.3 × span²`). For that to equal ~$16.4k requires `span ≈ 112h`, plausible
  given the 168h backstop buffer. **But nothing pins the $16.4k anchor itself**, and BUILD_STATUS
  separately flags the live fallback gap as **≈$16–18k and the $40k figure as stale** — so the anchor
  was calibrated to a *range that is itself uncalibrated and drifting*. There is no committed test
  asserting "EXPRESS full-span ≈ fallback gap" (grep of `tests/` finds none); the calculation was an
  S44 OR-subagent ephemeral. **An anchor with no test and no documented derivation is unfalsifiable by
  construction** — the next person cannot check it or reproduce the $16.4k.
- **The 4:2:1 tier ratio is defensible as an *ordering*, not a magnitude.** It reuses
  `TIER_SPECS.w_sp`, whose *ordering* (EXPRESS > STANDARD > DEFERRED) is an asserted invariant
  (`flexibility_model.md` line 60). The specific 4:2:1 *spacing* is `[CAL]` with no source — same
  status as the 90/80/70 OTP anchors (user-stated working anchors). Defensible to ship as a
  bracket; not defensible as a point claim.
- **Flat-per-tier (not ·weight_kg) does make tier the priority lever, as claimed — verified.** The
  headline arms stamp `W_k = scale · w_sp · weight_kg` (generator line 944, weight-scaled); the
  *recourse* path overwrites that with a flat per-tier `W` (replay line 743, no weight). So under
  recourse, a 50kg EXPRESS and a 1200kg EXPRESS carry the same penalty ⇒ the solver prioritizes by
  *tier*, not by mass. The claim holds. Whether tier-priority (vs. mass-priority, vs.
  cw-priority) is the *right* recourse objective is a modeling choice with no external anchor — flag
  it as a design decision, not a calibrated fact.
- **Net headline risk = zero today, high the moment the §6 arm is reported.** The headline keeps W=0
  (D-A12 excludes the C.10 penalty from realized cost), so `_RECOURSE_W` is inert in L2. But §6/§12
  M-B9 already says "disruption-recovery is the larger real driver — run the §6 recourse sensitivity
  before claiming the air thesis *fully* proven." The instant that sensitivity arm runs, these three
  unanchored, untested constants become the load-bearing calibration of its entire dollar figure.
  **Fix before that arm reports:** (i) commit the derivation as a doc + a test asserting EXPRESS
  full-span ≈ the *live* fallback gap (and update it when the $16–18k figure firms); (ii) report the
  §6 value as a curve over the 4:2:1 spacing, not a point.

### F-D — `mip_rel_gap = 0.005` sits INSIDE L2 (a difference of two gap-stopped objectives); 0.5% may not be tight enough relative to measured L2. **[MED→HIGH, SILENT — magnitude-dependent]**
New surface this session (the prompt flags it). `MilpParams.mip_rel_gap = 0.005` (air_milp lines
195–199): every per-cycle solve stops at 0.5% relative gap, deterministically. L2 = C(M₁') − C(M₁) is
a **difference of two such gap-stopped incumbents**. Each objective carries up to +0.5% of its own
value as unproven slack; the difference can carry up to ~0.5% of the *larger* objective as pure
solver noise. If the freight cost per cell is, say, $200k and the headline L2 is a few % of that
(single-digit thousands of dollars), then **0.5% of $200k = $1k of gap noise can be a non-trivial
fraction of the L2 signal** — and the two arms need not gap-stop symmetrically (M₁ optimizes a
superset, so its incumbent and bound move differently). The methodology's own `run_replay` docstring
concedes this: *"all per-cycle solves stop at mip_rel_gap so cross-arm comparisons carry ≤ that
relative gap (the BLK-1 caveat)."* That caveat is acknowledged but **not yet quantified against
measured L2**. **Required check before Stage 3 locks:** at the peak cell, re-solve M₁'/M₁ at
`mip_rel_gap = 0` (or 1e-4) and confirm L2 moves by ≪ the reported CI half-width. If 0.5% gap noise is
comparable to L2, the headline cells must be solved to a tighter gap (the determinism argument for
0.005 holds for *any* tolerance — 1e-4 is equally deterministic, just slower). MFBlink already brought
the root gap to 0.55% on the hard cells, so a tighter final stop is likely affordable.

### F-E — H₀ daily cadence is a NEW L1 calibration surface; the 24h grid + short-fuse escape can bias L1's sign. **[MED, SILENT — affects L1, not headline L2]**
New this session (`_daily_times`, replay lines 310–329; `_plan_cycle_h0`). The grid is anchored at
the *first arrival* and steps by exactly `_DAY_H = 24.0`; a shipment whose `[known_at, tender_at]`
window contains no grid point is booked **ad-hoc at its own cutoff** (on the spot, not dropped). Three
calibration commitments hide here, all bearing on **L1 = C(H₀) − C(M₁')** (not the headline L2, so
lower priority — but L1 is a reported number):

- **The 24h period is a hard-coded magnitude, not swept.** A human planner's batch cadence (daily vs.
  twice-daily vs. every-other-day) directly sets how much cargo H₀ co-batches, hence its
  consolidation, hence L1. 24h is plausible but uncited.
- **Grid phase is arrival-dependent.** Anchoring at `min(known_at)` means the grid's *phase* relative
  to cutoffs is a function of the demand draw — so two cells with the same κ/α/λ but different first
  arrivals get different H₀ batching. This is a subtle, undocumented coupling of L1 to a nuisance
  parameter.
- **Short-fuse → book-at-cutoff is L1-favorable to M₁'.** D-A14/D-A15 already flag that H₀'s
  batch-at-cutoff timing (commit later, more info) can *beat* M₁''s commit-at-reveal, so
  `L1 = C(H₀) − C(M₁')` may go ≤ 0 at tight cells. The short-fuse escape *adds* book-at-cutoff events
  to H₀, pushing in the same favorable direction. The `run_replay` docstring honestly calls this "a
  finding, not a bug," which is the right posture — but the 24h period and the grid phase are
  un-swept companions of an already-delicate L1. **Sensitivity-check the H₀ period** (12/24/48h) the
  way λ-spread should be checked; report L1 as a small band, not a point.

### F-F — Density band (120–240 kg/cbm) still silently caps the proof at low-density cargo; the 333 tripwire is live. **[MED, SILENT]**
*(S36 F2/F8, still open.)* `_DENSITY_LOW/HIGH = 120/240` (generator line 188). The closed-form
`E[SE_k]` assertion `4.5·240 < 1500` (line 268) is a real tripwire: real air cargo (machinery, auto
parts) routinely exceeds 333 kg/cbm, and recalibrating the band up past 333 breaks the closed form and
silently changes what κ means. Separately, the slot-density (volume binds ≤333) vs. billing-density
(`VOLUMETRIC_DIVISOR = 167`, air_milp line 75) divergence is unchanged: for the upper half of the band
(167–240), capacity is rationed by volume while cost is billed by weight, so the *tightness* (κ) and
the *value of freeing a slot* (what L2 monetizes) are coupled through two different density
conventions. Still a hidden calibration commitment that the docstring documents as serving only the
slot-count purpose. **Fix:** pin weight+density to BTS FAF commodity mix (free, real), accept that the
band will cross 333, and re-derive `E[SE_k]` as `E[max(w/W, v/V)]` over the crossing band (an integral,
not a product). This simultaneously fixes F-B's κ-label meaning.

### F-G — Book-lead spread (24h) + uniform shape are still silently fixed companions of the swept λ. **[MED]**
*(S36 F5, still open, unchanged.)* `ArrivalConfig.book_lead_spread_h = 24.0`, shape uniform, both
`[CAL]`; λ compresses the *mean* but the *spread* is fixed. A wider spread at fixed λ-mean manufactures
more late arrivals ⇒ more premature commitment ⇒ more L2, **without moving the swept λ axis**. λ is
honestly swept; its companion spread is a back-door L2 lever. No free source gives air booking
lead-time distributions — flag inferred, sensitivity-check the spread. (Tier-coupled lead 12/48/96h,
S36 F6, remains correctly quarantined behind `tier_coupled_arrival=False` ⇒ honestly handled, no
action.)

### F-H — Weight triangular(50,1200,300) jointly sets E[SE_k] and the κ-label; unchanged. **[MED, coupling]**
*(S36 F8, still open.)* `_WEIGHT_LOW/HIGH/MODE = 50/1200/300` feed `E[SE_k]` (and thus `total_N`)
directly. Every one of these moves the absolute supply level at a given κ, so the κ label is only as
anchored as this distribution. §13 D-A18 already requires reporting realized post-consolidation
occupancy per κ-cell — that is the honest check that converts the κ-label into an observable; it must
actually be reported in Stage 3, not just specified. Sourceable from BTS FAF (free).

### F-I — `n_hawbs` fixed-N is an acknowledged L2-deflating simplification. **[LOW]**
*(S36 F9, unchanged, honestly handled.)* `n_hawbs = 20/30`. M-B9 already says report the `known_at`
distribution and state fixed-N as L2-deflating. Conservative; no manufacture risk.

### F-J — `V_ref`/`w_p` .tex constants inert while headline W=0. **[LOW for headline L2]**
*(S36 F10, unchanged.)* The C.10 `V_ref`/`w_p` (CALIBRATION NEEDED) steer routing but D-A12 excludes
the penalty from realized cost; inert on the headline. **Becomes MED on the §6 recourse arm** (where W
is set to `_RECOURSE_W`, see F-C) and if the cost–OTP frontier is ever folded into the headline.

### Resolved this session

### F-K (was S36 F4) — fallback dominance factor drift is RESOLVED. **[RESOLVED]**
S36 flagged `_FALLBACK_DOMINANCE_FACTOR = 2.0` + a hard-coded `2.0·per_leg` (two air legs) drifting
from §13's `1.5 × graph-derived max-air-legs`. **That code is gone.** The live mechanism is
`air_graph.FallbackPolicy` (lines 792–811): `margin = 1.5` × the **longest-cost upper-bound path over
the HAWB's real pre-filtered subgraph** (`_max_path_cost`, a DAG longest-path — so legs are
graph-derived, not literal-2), with each air arc priced at `air_leg_cost_ub` (worst single-leg cost
incl. MAWB fix) and a `trivial = 1.0` for HAWBs with no feasible real path. This matches §13 D-A19's
`1.5 × worst-spot-route, max-air-legs graph-derived` exactly. The S36 33%-swing concern is closed.
*Residual:* the live fallback gap is **≈$16–18k and the $40k figure in older docs is stale** (a doc
reconcile item, not a code defect) — and F-C's recourse anchor was tuned to this still-soft figure.

---

## Honest accounting: swept vs. silently-fixed

| Knob | Status | Risk if the silent ones bite |
|---|---|---|
| κ (network tightness) | **swept** (coarse integer ladder) | its *meaning* rides on F-F/F-H distributions |
| α (Dirichlet lumpiness) | **swept** | F-B: range still unanchored ⇒ peak uncitable |
| λ (book-lead compression) | **swept** | F-G: companion *spread* fixed |
| spot gap `m` (Slice C, **BUILT**) | **κ-INDEPENDENT flat band** | **F-A: contradicts the spec's κ-tie; dominant $ lever decoupled from the sweep** |
| spot CW cap (Slice C, BUILT) | **fixed band** `U(1,3)×1500` | low-MED (sets spill volume); reasonable |
| `mip_rel_gap = 0.005` | **fixed** | F-D: 0.5% noise sits inside the L2 difference |
| H₀ daily period (24h) + grid phase | **silently fixed / arrival-coupled** | F-E: biases L1 sign |
| recourse W (1.3/0.65/0.32) | **silently fixed, untested, inert on headline** | F-C: load-bearing the moment §6 arm reports |
| `E[SE_k]` weight/density bands | **silently fixed** | F-F/F-H: recalibrate κ-label; 333 tripwire live |
| fallback factor | **1.5 × graph-derived path** | **resolved (F-K)** |
| M₀/M₁'/M₁ pin structure | **fixed-deterministic** | none — required for the nested-feasible-set chain |
| tier mix 20/40/40 | **swept** | low |

**The loose-corner `|L2| < CI` gate remains an honest null** — it forces L2≈0 in the
abundant-κ × even-α × early-λ corner. But it constrains only the *quantity-of-spill* channel; with
Slice C built κ-independent (F-A), the *price-of-spill* channel never enters the sweep at all, so the
gate cannot even see the dominant $ lever it was meant to discipline. F-A/F-B/F-D/F-G all inflate (or
silently set) the peak without touching the loose corner.

---

## Calibration-note must-pin-down checklist (free sources only)

| # | Knob (location) | Current state | Defensible real range | Source (free) | L2-sensitivity |
|---|---|---|---|---|---|
| C1 | **Spot gap `m` κ-tie** (`_draw_spot_regime`) | κ-INDEPENDENT `U(.85,1.18)` — **violates spec** | regime tied to κ (soft<1 / peak~1.18) OR fixed+amended | WorldACD/avitrader, Xeneta (URLs in backtest §7) | **HIGH** |
| C2 | **α Dirichlet** (`GenConfig.alpha`) | 1.0, swept, unanchored | map to capacity-dispersion HHI/Gini | BTS FAF lane flow + OpenSky freighter freq | **HIGH** |
| C3 | **recourse W** (`_RECOURSE_W`) | 1.3/0.65/0.32, untested anchor | 4:2:1 ordering OK; magnitude = live fallback gap | internal (commit derivation + test); fallback gap ≈$16–18k | **HIGH (on §6 arm)** |
| C4 | **mip_rel_gap** (`MilpParams`) | 0.005 | tight enough that 0.5%·obj ≪ CI(L2) | internal (re-solve peak at 1e-4, compare) | **MED→HIGH** |
| C5 | **Density band** (`120/240`) | uniform; crosses 333 if widened | ~80–600+ kg/cbm | BTS FAF commodity→density per lane | **MED→HIGH** |
| C6 | **Weight tri** (`50/1200/300`) | triangular | per-lane HAWB weight dist | BTS FAF tonnage/shipment | **MED→HIGH** (sets E[SE_k]) |
| C7 | **H₀ period** (`_DAY_H=24`) + grid phase | fixed, arrival-coupled | 12/24/48h human batch cadence | inferred; sensitivity-check | **MED** (L1) |
| C8 | **Book-lead spread** (`24h`) | fixed, uniform | air booking-curve shape | no free source; flag inferred + sweep | **MED** |
| C9 | **SLA offsets / θ_flex / 90-80-70** | `[CAL]`, sandbag = pessimistic edge | standard express/std/deferred structure | product structure (ordering); levels need partner | **MED** |
| C10 | **Tier mix** (`20/40/40`) | swept | per-lane service mix | BTS FAF + partner | LOW |

DAT remains **not licensed** — no item depends on it. UN/LOCODE + IATA fix topology only (already
real). The two highest-value free sources for *this* proof are unchanged from S36: **BTS FAF**
(weight/density/flow concentration → C2, C5, C6 + the κ-label) and the **already-sourced
WorldACD/Xeneta anchors** (C1) — plus two internal-only fixes new this session (C3 recourse anchor, C4
gap tightness) that need no external data, just a committed derivation + test.

---

## The single most important thing to pin before the (κ,α,λ) sweep is trusted

**Reconcile Slice C's spot gap with its own spec (F-A) — decide whether `m` is κ-tied or κ-fixed, and
make the code and the methodology agree before the sweep runs.** Spot pricing is, by the
methodology's own words, "the dominant $ knob" and "the most controllable lever on the L2 dollar
figure." Right now the code draws it on a deliberately κ-independent stream as a flat
`Uniform(0.85, 1.18)`, while `backtest §7` and §13 D-A19 mandate a κ-tied regime mixture (loose→soft,
tight→peak). This is not a missing anchor — it is a *built contradiction* between approved spec and
shipped code on the one lever that sets the dollar headline. Until it is resolved one way (implement
the κ-tie, accepting the κ↔spot coupling the spec intends) or the other (amend the spec to a fixed
conservative gap and state the dollar L2 excludes the peak-season price channel), the L2 dollar figure
is governed by an undocumented, adversary-pickable, κ-decoupled band that the loose-corner null cannot
discipline. Pin this first; α (F-B) is second and unchanged from S36 (anchor its range to BTS FAF +
OpenSky dispersion and lead with the L2(α) curve, not a peak cell).

---

## S36 F1–F10 status carry-forward

| S36 finding | This session | Status |
|---|---|---|
| F1 — α unfalsifiable amplifier | now F-B; nothing changed; D-A24 makes it more load-bearing | **OPEN** |
| F2 — slot(333) vs billing(167) density divergence | now F-F; unchanged; 333 tripwire live | **OPEN** |
| F3 — κ→regime spot map unbuilt | **now BUILT — and built κ-INDEPENDENT, contradicting the spec** → F-A | **CHANGED (built wrong)** |
| F4 — fallback factor 2.0-vs-1.5 drift | `FallbackPolicy` rewrite: 1.5 × graph-derived path → F-K | **RESOLVED** |
| F5 — book-lead spread fixed | now F-G; unchanged | **OPEN** |
| F6 — tier-coupled lead quarantined | unchanged; still behind `tier_coupled_arrival=False` | **RESOLVED-by-design (no change needed)** |
| F7 — SLA offsets / θ_flex fixed, sandbag=pessimistic edge | now C9; unchanged | **OPEN** |
| F8 — weight/density set E[SE_k] & κ-label | now F-H; unchanged | **OPEN** |
| F9 — fixed-N L2-deflating | now F-I; unchanged, honestly handled | **OPEN (acknowledged, conservative)** |
| F10 — V_ref/w_p inert while W=0 | now F-J; inert on headline, **MED on the new §6 recourse arm** | **OPEN (newly relevant via F-C)** |

**New findings this session (the S44 surface):** F-A (Slice C κ-decoupling — promoted from F3),
F-C (recourse weights), F-D (mip_rel_gap inside L2), F-E (H₀ daily cadence). F-K resolves S36 F4.
