# S46 Capacity Redesign — Metrics & Calibration (air slice)

**Status:** DESIGN PROPOSAL — behind the formal-model approval gate. No `src/` / `tests/` code is
written until the user approves. Defines the three metric families the redesign must measure, the
sourced real-world target ranges to calibrate against, the knob→metric map, and the new scorer
outputs required beyond today's `score_run`.

**Author role:** Metrics & Calibration designer. Owns §G items 15–17 of
`01_architecture.md` (their precise definition + calibration). Consumes the methodology
(`arrival_only_replan_methodology.md`), the live scorer (`src/replay.py::score_run`), and the
sourced market context (`air_pricing_calibration_s45.md`). Every external figure is cited with a
URL or labelled **MRN** (market-research-needed). Nothing is fabricated.

**Notation (carried from methodology §3–§4).** Arms `H₀ ≥ M₀ ≥ M₁' ≥ M₁ ≥ π_hind` (PIH). For HAWB
`k`: `A_k` = deterministic realized door arrival (the §3 running-clock walk = `route_reliability`
`Â`); `Δ_k` = the **frozen booking promise** (`booking_promise.promised_deadline_at`, set at
tender, replan-immutable); `T^abs_k` = the hard backstop (`backstop_deadline_at`). `K` = scored
(tendered) HAWBs, `N = |K|`. `C(arm)` = realized **committed** total cost (freight + consolidation +
spot/fallback; **excludes** the C.10 quadratic tardiness penalty — objective-steering, not a cash
outflow, per D-A12). `W_k` = tier priority weight (EXPRESS:STANDARD:DEFERRED = 4:2:1).

---

## 1. Metric family (1) — Cost savings

### 1.1 Per-arm primitive
`C(arm)` for each of `H₀, M₀, M₁', M₁, π_hind` — already produced by `score_run` as `total_cost`
(final all-pinned billing solve). No change to the primitive.

### 1.2 Decomposition (methodology §4, denominators pinned)
```
L1            = C(H₀)  − C(M₁')           (planning value: human → competent single-pass)
L2  (headline)= C(M₁') − C(M₁)            (replan value: cross-cycle open-book reshuffle)
Total         = C(H₀)  − C(M₁) = L1 + L2  (full automation+replan value)
regret        = C(M₁)  − C(π_hind)        (irreducible gap to clairvoyant)
```
Internal `L1` split (ablation, not product-facing): automation `C(H₀)−C(M₀)` + within-cycle
optimization `C(M₀)−C(M₁')`.

### 1.3 Headline savings vs the human baseline — **denominators (the "% of what")**
- **Absolute:** `Δ$ = C(H₀) − C(M₁)` (full) and the headline `L2$ = C(M₁') − C(M₁)` (replan-only).
- **Percent — all on the single denominator `C(H₀)`** (the human spend), so the shares are additive:
  `Total% = (C(H₀)−C(M₁))/C(H₀)`, `L1% = L1/C(H₀)`, `L2% = L2/C(H₀)`, with `Total% = L1% + L2%`.
  **Lead with `L2%`** (M-B3). `cw_flex`-normalized `L2/cw_flex` is reported **peak-cell only** (the
  denominator shifts across the λ grid, so it is not comparable cell-to-cell — M-B3).
- Do **not** use `C(M₁)` as a denominator (flatters the number); the human spend `C(H₀)` is the
  honest "vs status-quo" base.

### 1.4 The S45 re-run: capacity-bearing vs consolidation fraction of L2 (THE redesign test)
S45's killer finding: `L2` was **100% consolidation reshuffle / 0% capacity** (`l2_decomposition_s45.md`).
The redesign exists to make `L2` carry a capacity component. We measure it **two ways**, both
required (they answer different questions):

**(a) Cost-attribution split ($).** Bill each arm into the component vector the MILP objective
already separates: `{contracted$, spot_block$ (per block i), fallback$, consolidation/handling$,
trucking/ground$}`. Group `capacity = {contracted, spot_block, fallback}`,
`structure = {consolidation/handling, trucking/ground}`. Then
```
L2_capacity      = Σ_{c∈capacity}  [C_c(M₁') − C_c(M₁)]
L2_consolidation = Σ_{c∈structure} [C_c(M₁') − C_c(M₁)]
φ_cap            = L2_capacity / L2          (capacity-bearing fraction of the $ saving)
```
Report signed components (one can offset the other — e.g. `M₁` spends *more* on contracted to spend
*less* on spot+fallback; that net is the capacity-bearing saving). `φ_cap` may exceed 1 or go
negative when structure offsets — report the raw signed pair, not just the ratio.

**(b) kg-movement split (reshuffle composition).** The S45 verdict was stated in kg: total spot kg
byte-identical, contracted never used, fallback never touched ⇒ capacity fraction 0. Mirror it:
`reshuffle_capacity_kg = Σ |kg shifted between capacity tiers {contracted, spot-block-i, fallback}|`
between `M₁'` and `M₁`; `reshuffle_consolidation_kg = Σ |kg re-grouped onto a different MAWB at the
**same** tier|`. `ψ_cap = reshuffle_capacity_kg / total_reshuffled_kg`.

**Pre-registered gate (carry from D-A12 reshuffle-share floor):** the redesign passes only if at the
tight cells **`L2_capacity` CI is separated from 0** and **`ψ_cap ≥ 0.30`** (proposed floor —
confirm with user; S45 had `ψ_cap = 0`). Reported alongside the consolidation fraction so the fix is
observable, not asserted.

---

## 2. Metric family (2) — OTP statistics

### 2.1 Delivery-state classification (precise on-time definition)
Each scored `k` lands in exactly one of **three** states — this is the "late but delivered" vs "fell
to backstop" distinction the brief requires:

| state | predicate | arrival used |
|---|---|---|
| **on_time_real** | `¬on_fallback ∧ A_k ≤ Δ_k` | walk `A_k` |
| **late_real** | `¬on_fallback ∧ A_k > Δ_k` | walk `A_k` (delivered on a real route, missed promise) |
| **fallback** | `on_fallback` | `A_k = T^abs_k` (backstop; **always late** — assert `T^abs_k > Δ_k`) |

A fallback delivery is **never** counted on-time even if its `T^abs` happened to precede `Δ_k`
(by construction it does not; the assert guards a generator bug). On-time = delivered on a **real**
(non-backstop) route within the **frozen** promise `Δ_k` — not the live replan-mutable deadline, and
not `T^abs`.

### 2.2 Counts, rates, tardiness
```
n_on_time = |on_time_real|,  OTP   = n_on_time / N            (= today's score_run otp)
n_delayed = |late_real| + |fallback|,  delayed% = 1 − OTP
tard_k    = max(0, A_k − Δ_k)        (hours; = T^abs_k − Δ_k for fallback)
```
Over the **delayed** set: `mean / median / p95 tard_k`; `total_tardiness = Σ_k tard_k`;
**tardiness-weighted penalty** `Σ_k W_k · tard_k` (linear, cash-equivalent; the C.10 *quadratic*
form is objective-steering only and stays out of the cash metric).

### 2.3 Per-arm reporting (so OTP is comparable across replan strategies)
Emit the full state vector `(n_on_time_real, n_late_real, n_fallback, OTP, mean/median/p95 tard,
total_tardiness, Σ W_k·tard_k)` for **each** arm `H₀/M₀/M₁'/M₁/π_hind`. Cross-arm `OTP(M₁) − OTP(M₁')`
= the SLA effect of open-book replanning; `OTP(H₀)` = the human's realized service level. Deferred-air
arcs (architecture #9) are the **cost↔OTP lever**: cheap-but-slow legs the optimizer takes when slack
allows, trading `tard_k` for `C`.

---

## 3. Metric family (3) — Fallback / no-feasible-route incidence

### 3.1 Count and rate (denominator = N, all scored HAWBs)
`fallback_count(arm) = |{k : on_fallback}|`, `fallback% = fallback_count / N`. Already in `score_run`
as `fallback_count`; surface `%` and the per-arm series as first-class.

### 3.2 Three causes (new instrumentation — the brief's a/b/c)
| cause | definition | detection signal |
|---|---|---|
| **(a) structural-infeasible** | no real path exists in `k`'s subgraph at planning (fallback is the *only* exit) | build-time: `A_k` has no non-fallback path / prefilter empties it |
| **(b) capacity-exhaustion roll** | a real path **exists** in the subgraph but contracted+spot is exhausted, so the solve routes `k` to fallback | a real path exists **and** a lane/arc on the cheapest would-be path is at-capacity in the ledger at solve time |
| **(c) disruption-induced** | a firm route the §6 recourse could not recover lands on the node-anchored fallback | `k ∈ to_replan` history **and** recovered to a `FALLBACK` arc |

In the clean (no-disruption) headline, (c) ≡ 0, so fallback is **(a) + (b)**; (a) is a graph/deadline
property invariant across arms (a floor every arm shares), (b) is the **capacity-driven, arm-varying**
component — the one the redesign moves.

### 3.3 Per-arm comparison
`fallback_count` per arm; `fallback_count(M₁') − fallback_count(M₁)` = **fallback avoidance by
replanning** (= the existing `metrics.l2_fallback_avoidance` column, now populated). `H₀`'s count is
the human roll rate (its `_plan_cycle_h0` capacity-roll path). Report the (a)/(b)/(c) split per arm so
"structural floor" is separated from "replan-addressable rolls."

---

## 4. Calibration targets (sourced real-world ranges)

Confidence: **SOURCED** (public figure + URL), **INFERRED** (derived from a sourced anchor), **MRN**
(no free public figure — flagged, not fabricated).

| Target | Realistic value / range | Regime | Confidence | Source |
|---|---|---|---|---|
| **Carrier leg-level OTP** ("Delivery-As-Promised": shipment Notified-For-Delivery within **6 h** of planned arrival) | **62.7%** (2025, all routes); ~67% implied 2024 (9/12 months below 2024) | normal-to-stressed year | **SOURCED** | CargoAi 2025 Air Cargo Quality Report, via Air Cargo News (1) |
| Best vs worst **route** DAP | best **93–96%** (intra-Asia short-haul: TPE–ICN 96%, ICN–NRT 94%); worst **25–26%** (Chengdu/Shanghai→Frankfurt) | annual | **SOURCED** | CargoAi report, via ebidfreight (2) |
| Peak / disruption OTP degradation | **−4 to −5 pp** YoY in May–Jul 2025 (network reshuffle + de-minimis disruption) | peak/disruption | **SOURCED** | CargoAi report (1) |
| **Transpac-specific** OTP | not broken out publicly | — | **MRN** | (TAC/Xeneta lane series are paid) |
| **Door-to-door forwarder OTP vs a committed SLA `Δ_k`** (what our sim actually scores) | **~80–92% normal / ~70–80% peak** | INFERRED band | **INFERRED / MRN** | derived — see §6 calibration risk |
| **Roll / offload incidence** (% shipments bumped to a later flight) | no direct public % | — | **MRN** | — |
| — proxy: not-delivered-as-promised | **~37%** (1 − 62.7%) is an **upper bound** on degraded shipments (incl. minor delays, not only rolls) | 2025 | **SOURCED** (derived) | (1) |
| — proxy: dynamic load factor (vol+wt) | **~62%** (Dec 2025, all lanes); transpac headhaul peak **80–90%+** ⇒ roll pressure concentrates at peak/headhaul | Dec / peak | **SOURCED** | Xeneta dynamic load factor (3) |
| — INFERRED realistic fallback% for our sim | **~2–8% normal, ~15–30% tight/peak** | by τ regime | **INFERRED** | bounded by expedite-spend benchmark below |
| **Expedite spend** as % of total logistics cost | **top performers 3% / bottom 10%** (7 pp gap); **49%** of expedite events caused by forecast error | benchmark | **SOURCED** | APQC via Logility (4) |
| **Expedite / fallback price multiple** | expedited air **4–10× standard**; general expedite premium **+30–100%**; capacity-crunch **+150–200%** | normal → crunch | **SOURCED** | search synthesis (5) / WebCargo (6) |
| — our fallback multiple (2.5× base spot) | sits at the **low-realistic** end of the band ⇒ conservative | — | **INFERRED** (consistent) | `air_pricing_calibration_s45.md` §5 |
| **Replan / re-optimization cost-savings %** benchmark | no direct public figure | — | **MRN** | — |
| — proxy ceiling | the **3%→10%** expedite-spend gap (7 pp of logistics cost), **49%** forecast-/planning-addressable, ⇒ planning+replan plausibly recovers **low-to-high single-digit %** of freight spend | — | **INFERRED** | (4) |
| — disruption cost context | supply disruptions ≤30 days cost **3–5% of EBITDA** | context only | **SOURCED** | McKinsey via Logility (4) |

**Sources:**
1. https://www.aircargonews.net/airlines/air-cargo-on-time-performance-drops-in-2025/1081153.article
2. https://www.ebidfreight.com/post/cargoai-ranks-best-and-worst-air-cargo-routes-as-reliability-declines
3. https://www.xeneta.com/news/will-air-cargos-tailwinds-hold-out-into-peak-season-as-july-marks-a-sixth-consecutive-month-of-rates-increase (dynamic load factor); CLIVE/Xeneta methodology: https://worldaviationfestival.com/blog/airlines/air-cargo-load-factor-metrics-can-be-misleading/
4. https://www.logility.com/blog/the-real-cost-of-expedited-freight-in-your-supply-chain/ (APQC 3%/10%, 49% forecast-error, McKinsey 3–5% EBITDA)
5. Expedited-vs-standard multiples (4–10× air; +150–200% crunch): search synthesis, cross-checked against https://www.cogisticstransportation.com/expedited-freight-vs-standard-shipping-understanding-the-cost-vs-speed-tradeoff/
6. https://www.webcargo.co/blog/air-cargo-price-per-kg/ (express +20–40%, density/thin-lane spread)

**Reading.** The one solid public OTP number (**62.7% DAP**) is a **carrier-leg, 6-h-window** metric —
*tighter* and at a *different grain* than our door-to-door SLA `A_k ≤ Δ_k`. Calibrating our sim OTP to
62.7% would mis-set the target (we'd inject far too much lateness). Forwarders pad the door SLA, so
realistic **door-level** OTP runs **higher** (INFERRED 80–92% normal) — but that exact figure is
**MRN**. This is the single biggest calibration risk (§6).

---

## 5. Calibration loop — the knob→metric sensitivity map

The knobs are the `01_architecture.md` generator dials. Arrows give the dominant direction; "(primary)"
marks the knob that *owns* a metric.

| Knob (architecture ref) | Cost / L2 | OTP | Fallback% | Notes |
|---|---|---|---|---|
| **`τ` global tightness** (D.1) ↓ | savings ↑, L2 ↑ | OTP ↓ | fallback ↑ **(primary)** | the master dial; `τ<1` = demand>supply |
| **`lane_mix` n_short** (D.3) ↑ | L2_capacity ↑ | tardiness ↑ | fallback ↑ **(primary, localizes)** | short lanes are where rolls/tardiness fire |
| **short-band floor** `[0.6,0.85]` ↓ | — | tardiness ↑ | fallback ↑ | raise floor → "roll-but-rarely-fallback"; lower → force backstop |
| **`r_contract` vs `r_spot`** (B.3) — contracted **below** $5.5 spot ($4.2) | **L2_capacity / φ_cap ↑ (primary)** | small | fallback small | **the S45 root-cause fix**: if contracted ≥ spot, capacity stays inert (`ψ_cap=0`) |
| **composition** contracted share ↑ (B.2) | φ_cap ↑ | OTP ↑ | fallback ↓ | too cheap+abundant kills the binding; tune against the 0.70/0.22 default |
| **block ceiling** (A#4) ↓ | cost ↑ | — | fallback ↑ | spot exhausts sooner → up the curve → backstop |
| **block step multiplier** (1.15–1.25×) ↑ | L2 ↑ (reshuffle to cheaper blocks pays) | — | — | steeper curve = more reshuffle margin |
| **fallback multiple** (2.5×, A#6) ↑ | L2_fallback_avoidance ↑ | — | **count unchanged** | price lever, not feasibility — **decouples cost magnitude from fallback count** |
| **deferred-air share + transit slack** (A#9) ↑ | cost ↓ | **tardiness ↑ (OTP cost-lever, primary)** | — | cheap-slow arc eats deadline slack |
| **`α` within-lane concentration** (E) ↓ (lumpy) | — | — | fallback ↑ at fixed `τ` | local scarcity noise; sharpens metric 3 without moving `τ` |
| **`λ` arrival lateness / book-lead compression** (§10) ↑ | **L2 ↑ (primary)** | OTP ↓ | small | more book unknown at cutoff → premature commit |
| **deadline tightness** (tier `Δ_k` offsets) tighter | — | **OTP ↓ (primary level-setter)** | — | the dial that lands OTP in the target band |

### 5.1 Recommended loop order (to land all three in-range simultaneously)
1. **Make capacity bind:** set `r_contract < r_spot` (≈$4.2 < $5.5) and the 0.70 contracted share so
   `ψ_cap > 0` — *without this, metric 1's capacity fraction is dead-on-arrival (S45)*. Verify on C0.
2. **Set the regime:** dial `τ` + `lane_mix` until **fallback%** lands in the INFERRED band (single
   digits normal at `τ≈1.1`; ~15–30% at `τ≈0.7`). `α` and the short-band floor fine-tune.
3. **Tune OTP:** adjust tier `Δ_k` offsets + deferred-air slack until **door-level OTP** lands ~80–92%
   (normal) / ~70–80% (tight). *Do not target 62.7%* — that is the leg-level DAP, a different metric.
4. **Confirm metric-1 quality:** check `L2_capacity` CI separated from 0 and `ψ_cap ≥ 0.30` at the
   tight cells; check `Total%`/`L2%` are plausible single-digit-% of `C(H₀)` (the expedite-gap proxy).
5. Iterate 2–4 (they interact: tighter `τ` lowers OTP and raises fallback together).

Because `τ`/`λ` move all three at once, the loop converges by fixing the *structural* knobs (step 1)
first, then using the *level* knobs (`Δ_k`, deferred slack, short-floor) to separate OTP from fallback.

---

## 6. Biggest calibration risk (flag)

**OTP.** The only robust public anchor — CargoAi **DAP 62.7%** — measures *carrier-leg* delivery
within a *6-hour* window, a tighter and different-grained metric than our *door-to-door* SLA
`A_k ≤ Δ_k` (our `Δ_k = ready + base_transit + sla_offset` is a padded door promise). Calibrating our
OTP to 62.7% would over-inject lateness. The figure we actually need — **door-level forwarder OTP
against a committed SLA** — is **MRN** (lane-resolved service data is paid/internal). We proceed on an
**INFERRED 80–92% normal / 70–80% peak** band and flag it: if the user can source a door-level
forwarder OTP series, it supersedes the inference. Secondary MRN gaps: transpac-specific OTP, any
direct **roll-incidence %**, and any direct **replan-savings %** benchmark — all bounded by proxies
(DAP complement, the 3%/10% expedite-spend gap, the 4–10× expedite multiple) but none directly
sourced.

---

## 7. New scorer outputs required (delta vs `score_run` today)

`score_run` currently emits per arm: `total_cost`, `otp` (on-time fraction), `fallback_count`, and
per-shipment `ShipmentScore(realized_arrival_h, promised_deadline_h, on_time, on_fallback)`. The
`metrics` row leaves `l1 / l2_reshuffle / l2_fallback_avoidance` **NULL** (sweep-level). Needed:

1. **Cost-component vector per arm** — `{contracted$, spot_block$ by block, fallback$,
   consolidation/handling$, trucking/ground$}` from the final all-pinned solve. Enables §1.4(a).
   *(New: the billing solve already separates these in the objective; expose them.)*
2. **kg-tier-movement instrumentation** — per-HAWB capacity tier (`contracted / spot-block-i /
   fallback`) and MAWB group id, so §1.4(b) `ψ_cap` is computable between `M₁'` and `M₁`.
3. **Sweep-level decomposition function** — populate `L1, L2, Total, regret`, plus **new**
   `L2_capacity / L2_consolidation` ($ and kg), `φ_cap`, `ψ_cap`, and `savings_abs / savings_pct`
   (vs `C(H₀)`), `L1% / L2%`. (Currently no code computes these; `metrics` columns are placeholders.)
4. **Per-shipment tardiness + state** — `tardiness_h = max(0, A_k − Δ_k)`, a `delivery_state ∈
   {on_time_real, late_real, fallback}` enum, and tier `W_k`. (Today only the boolean `on_time`.)
5. **Per-arm tardiness aggregates** — `n_delayed, n_late_real, n_fallback, mean/median/p95 tardiness,
   total_tardiness, Σ W_k·tard_k`.
6. **Fallback cause tag** per fallback shipment — `{structural, capacity, disruption}` (§3.2), needing
   a build-time "real path exists?" signal + a ledger "would-be lane at capacity?" signal + recourse
   history. Report the per-arm (a)/(b)/(c) split.
7. **Calibration report table** — per-arm OTP / fallback% / `φ_cap` / savings% against the §4 target
   bands, so the user sees at a glance whether all three metrics are simultaneously in range.

**Schema note.** `metrics` today has `(run_id, total_cost, otp, l1, l2_reshuffle,
l2_fallback_avoidance, fallback_count)`. New fields above (tardiness aggregates, cost components,
fallback-cause split, `L2_capacity/L2_consolidation`, savings_abs/pct, n_delayed) need either added
columns or a sibling `metrics_ext` table — a build decision for the scorer slice, flagged here, not
resolved.
