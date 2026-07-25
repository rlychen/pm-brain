# Simulation design — S56, v3

**DRAFT for approval. Nothing is built until sign-off.**
v3 folds a 6-agent verification round on v2 (5 no-code reviewers + 1 code-as-inventory miner). v2's
constants were mutually inconsistent (§5 note); v3's default cell is re-derived so its own gates pass.
Tags: **[S]** sourced · **[I]** inferred · **[A]** assumed — *user's call* · **[NF]** not found, do not fabricate ·
**[A-carry]** magnitude carried over from the current design unchanged (not part of the disease).

---

## 0. The machine

A forwarder's planning desk, transpac Asia→US, replayed through time.

**Loop** — all arms, same cadence: once daily at hour `x ∈ {6,12,18,24}` (default 24). Per run, per arm:
1. Reveal: book = shipments with `book_at ≤ t`, not retired.
2. Capacity view: spot decays on the booking curve, floored at the arm's own reservation; BSA firm.
3. Solve: consolidation MILP over the movable book. **A shipment may be left UNPLACED (holding
   pattern); fallback is only chargeable at `commit_backstop`, never before** (v3 — else a deferred
   shipment whose flights aren't visible yet is dumped permanently by horizon artifact).
4. Ratchet: the plan's spot usage per flight → committed 2D reservation (element-wise max).
5. Commit: firm by the cutoff of the flight it stands on; an **unplaced** shipment at its
   `commit_backstop` commits to fallback. (The commit rule is now total over all states.)
6. Retire: departed / fallen-back shipments leave the visible book (MILP size flat in horizon).
7. Ad-hoc run: if a shipment's `[book_at, commit_backstop]` contains no scheduled run, one run is
   inserted at its backstop — arm-invariant. **[A]**

**Arms** (identical inputs; differ only in what a run may move): **H0** daily batch greedy
consolidation (supply-first best-fit-decreasing; billed by one all-pinned solve) [A-carry; user
redesign pending] · **M0** newcomers one-at-a-time, priors frozen · **M1p** newcomers jointly, priors
frozen · **M1** everything uncommitted · **PIH** clairvoyant single solve (bound).

**Estimand** `L1 = C(H0) − C(M1p)`, `L2 = C(M1p) − C(M1)` — empirical per-draw; the only theorem is
`C(arm) ≥ C(PIH)`. **The sign of L2 is a RESULT, never a gate** (v3 — gating on `CI > 0` would make
the headline unfalsifiable).

**Decay** (spot only): `avail = r + (C − r)·φ` per dimension; `φ(τ) = A_cut + (1−A_cut)(1−e^{−λτ})`,
`A_cut ~ Beta(1.3,8.7)` freighter / `Beta(1.8,6.2)` belly, `λ ~ U(0.10,0.16)/day` **[S]**.

**Reservation** (S54): assignment buys the flight's `(kg, m³)` envelope; element-wise-max ratchet;
identity mutable until the flight's cutoff. **Cost basis is a user decision — §11 Q2**: the OR review
showed `penalty_frac = 1` on the *horizon-peak* envelope is a directional tax on M1 alone, above the
grounded friction (free change pre-cutoff; 25–50% no-show at cutoff), making a null L2 unidentifiable.
Options: bracketing pair {1, 0.35}, or price the envelope **at its cutoff value** (encodes the grounded
rule with no new machinery).

---

## 1. The failure being fixed

Old generator: `capacity := τ × n_hawbs × E[cw]` split across every flight-day ⇒ 24–197 kg arcs vs
500 kg shipments, a 7-day network smaller than one 777F, and a definitionally inert τ (S45). Six
sessions blamed six mechanisms; the cause was the world. **Fix: supply = exogenous, integral,
per-flight allowance; schedule at full breadth; τ = an output.** v2's first repair re-created the
disease with the opposite sign (τ floor ≈ 2.0, slack locked in); v3's constants are derived to land
the default cell in band — see §5.

---

## 2. Ground truth

| Fact | Value | Tag |
|---|---|---|
| LD3 | 1,500 kg / 4.5 m³; 333 kg/m³ ⇒ always cubes out; **~900 chargeable kg/position** at ρ~200 | [S] |
| Cargo density | U(150, 250), mean ~200; volume binds, weight rows dead at allotment scale | [S] |
| Transpac freighter:belly | ~80:20 capacity ⇒ ~15% of flights belly | [S] |
| Transpac dynamic load factor | **90% = documented PEAK** (near-practical max); market average ~57–63%. **This sim is a declared peak-quarter regime, not "the market."** | [S] |
| Carrier cutoff | departure − 6 h (airline acceptance of built ULD) | [S] |
| Space booked | 5–7 d pre-departure ex-Asia; express latest, deferred earliest | [S] |
| Spot share of forwarder volume | ~half globally (Xeneta); 55% contracted defensible on a dense headhaul | [S] |
| BSA size / lane-week | **no public source** | **[NF]** |

---

## 3. Instance (v3 constants — re-derived so the gates pass; arithmetic in §5)

| | Value | Note |
|---|---|---|
| Book | **240 shipments/week** (~34.3/day) | **§11 Q1 (S57) — v3's own 120/wk assumption REJECTED; the red-team's alternative won.** Tightness comes from a bigger book absorbing a generous allowance, not from shrinking supply: L2 is fed by consolidation density, so the book *is* the estimand's fuel. ~2× solve cost (§9 is the sharp end). **The commercial story is UNVERIFIED** at 240/wk (~6–7 kt/yr, mid-size) — the 120/wk `[S]` tag does **not** carry over; do not assert it until sourced. |
| Weight | lognormal median 250 / mean 500 kg, **cap `w ≤ min(3000, 9ρ)` kg** (≤ 2 positions), rejection-resample, resample rate reported ≤ 5% | v2's 5 t tail exceeded every flight envelope ⇒ born-dead by construction. Truncated mean (~475 kg) is what τ uses. |
| Volume | `w/ρ`, ρ ~ U(150, 250) | [S] |
| Tier mix | 20 / 55 / 25 E/S/D | **[A] [NF] — pinned §11 Q5 (S60); tune-later trigger, see M8.** No source; do not fabricate one. S59 made the deferred share load-bearing (co-load is now cheapest-but-slowest ⇒ deferred is its natural rider), so 25% now sizes the co-load channel, not just demand composition. Retune **only** if M8 shows co-load soaking up the book. |
| Cargo classes | GEN/PER/DGR/VAL/HUM = .60/.20/.10/.05/.05; groups g = (class, temp); VAL/HUM singletons | [A-carry] — defines who may consolidate |
| Arrivals | Poisson, weekday weights **Mon–Fri .18, Sat .06, Sun .04 [A]**, uniform within day | pinned to kill the Monday-batch pathology (§7 M7) |
| Schedule | full breadth: ~26 legs/day, 9 O-D lanes (TPE/HKG/PVG × West/Midwest/East), feeders TPE→HKG + PVG→HKG, ~15% belly, weekly tiling; current bank clock-times [A-carry], phase vs cadence hour reported | |
| **BSA** | **contracts on the 4 dominant lanes ONLY (by q_ij), ~7 departures/lane-week (≈ daily), 1–2 positions (ν = 1.3) ⇒ ~36 positions/week ≈ 26% of book volume** | **§11 Q3 (S60) — held at 36; does NOT scale with the book.** Doubling it to 72 would put ~53% of the book in C.13a's free-at-margin zone (weight 0→`A_c` is sunk, dropped from the argmin) ⇒ no cost gradient ⇒ M1′ ≡ M1 ⇒ **L2 → 0**. Share is a lever on the estimand, not a realism dial. Mirror-image of S45 (contracted *never* used — pivot dearer than spot); both extremes are calibration artifacts ⇒ 26% is **[A]**, on the Q6 re-measure list. Also fixes a pre-existing §3/§4 contradiction: 3–4 lanes × ~4 dep × ν=1.3 = **18**, not 36 (S57's "~5/departure" was an artifact of that). v2 (1 dep/day × 9 lanes, ν∈[1,3]) = 0.9–1.8× the whole book; 55% unreachable; dead freight ≈ book revenue. |
| BSA pivot | **≤ 900 kg CW/position** | 1,200 is unattainable (LD3 cubes out ≤ 1,125 at ρ=250) ⇒ structural dead freight |
| **Spot access** | **φ = 0.25** of departures (~46 accessed flights/wk), `m_f = floor(μ_ℓ + U_f)`, floor ≥ 1 position | **§11 Q1 (S57)** — doubled with the book (φ = 0.125 → 0.25), the straight-doubling that preserved the ladder. v2's φ=0.40 made even the loosest corner τ_v ≈ 2.0. |
| Flight visibility | **H_vis = 30 d** | must exceed max book→dep lead (~29 d) or deferred bookings can't see their flights |
| Warm-up | **30 d**, validated by a **59 d control cell** (default cell, one seed, run once) — not by the stationarity gate alone | **§11 Q4 (S60).** Exact-zero memory needs H_vis + max residence ≈ 59 d (memory composes, it doesn't max — OR F1); 59 d everywhere costs ~+30% on top of Q1's 2×. The v3 stationarity gate (last warm-up week vs window weeks, per arm) is **too weak to rely on** — 1 week vs 3 on a noisy cost series has almost no power, so it would pass almost regardless; "failed to detect a trend" ≠ "no trend". A cheap option guarded by a gate that cannot fail is the S56 failure family. The control cell costs **one cell instead of +30% on every cell** and turns the assumption into a measurement. Gate retained as a **diagnostic, reported never trusted**. |
| Measured window | **21 d, whole weeks**; may shrink only to whole weeks and never below 14 d (max flight-choice span) | |
| Cool-down | `max(deadline − book_at) + T_max` ≈ **33 d** | tardy arcs must exist (penalty always on); they extend past the deadline |
| `T_max` | **96 h** — tardy arcs capped at `deadline + T_max`; a fallback shipment "arrives" at `deadline + T_max` | replaces old T^abs; pins the C.10 PWL span |
| Generated span | ≈ 84 d ⇒ **~2,880 generated, ~720 scored** (12 wk × 240; window 3 wk × 240) | doubled with the book (§11 Q1). tractability re-measure required (§9) — the sharp end |
| Seeds | supply × demand crossed 6×6 | CI via two-way random effects — NOT 36 replicates |

**Timeline** (four events; `d*`, `tender_at`, book-lead machinery all deleted):

| Event | Definition | Exog. |
|---|---|---|
| `book_at` (= `known_at`) | the Poisson arrival; forwarder knows; may reserve | yes |
| `ready_at` | `book_at + gap(tier)`: E U(0,24), S U(24,120), D U(120,456) h | yes [I] |
| `deadline` | `ready_at + base_transit(lane) + W + sla_offset(tier)`; **base_transit = 84/96/112 h** by dest region [A-carry]; **W = 24 h** scheduling allowance [A]; offsets **E 12 / S 24 / D 96 h** | yes |
| `cutoff` | dep − 6 h of the chosen flight | **no** |
| `commit_backstop` | latest first-leg cutoff over eligible **paths** (≤ 3 air legs [A-carry]), capacity ignored; **if the path set is empty, backstop := latest path cutoff ignoring the deadline, shipment flagged** | yes |

`W` exists because accessed-flight spacing ≈ 1/lane-day and cadence = 24 h: without it, express
(12 h offset) is structurally born-tardy and M1 rejects valid worlds (demand review F2). *No booking
revision/cancel event in v1 — deliberate contradiction of grounding F1d/F8; biases toward early
reservation; v2 = lead-time attrition hazard, same cluster as `penalty_frac < 1`.*

---

## 4. Supply

```
BSA:   fixed contract flights (4 dominant lanes × ~7/wk ≈ daily ⇒ ~28 contracted departures/wk);
       n_f = floor(ν + U'_f), ν = 1.3 pinned as a LITERAL at sign-off (⇒ ~36 positions/wk) —
       never a rule that reads the demand config. Held at 36 across the book change (§11 Q3):
       ν is NOT re-sized to any share target, so the 26%-of-book share FALLS OUT rather than
       being aimed at.
       Hard/soft split hard_frac = 0.35 [A-carry]; hard = take-or-pay PER LANE-WEEK (R3);
       soft = pivot ≤ 900 kg with 48 h release cliff [A-carry].
Spot:  m_f = floor(μ_ℓ + U_f), U_f ~ U(0,1) once per flight, frozen across cells (CRN);
       μ_ℓ = μ · s_ℓ, s_ℓ = S50 lane bands, normalized flight-weighted over the accessed set
       PRE-floor; realized mean m̄(μ) reported per cell (the ≥1 floor censors low μ_ℓ lanes —
       report, don't pretend E[m]=μ).
       Feeder legs: no O-D lane ⇒ s_feeder = mean s of the lanes they feed [A]; BSA trunk-only.
capacity(f) = (n_f + m_f) × (1500 kg, 4.5 m³);  R2: capacity keys on the FLIGHT, checkable as
       Σ offer capacities per flight_id ≤ flight capacity.
```
Supply never reads the book (`n_hawbs` appears nowhere in the capacity path). **The last demand
back-door is now closed (§11 Q3):** v3 sized ν to hit a 55%-of-book target — a one-time scale-setting
frozen as a literal (supply review F7), but still supply reading demand. Holding ν = 1.3 across the
book doubling means nothing in the capacity path is aimed at a demand quantity. Supply ⟂ demand is
cleaner than v3 had it.

---

## 5. Tightness lever

**Sweep μ ∈ [1.5, 4.5]** (mean spot positions per accessed flight). **Shifted up from [1.0, 4.0] at
§11 Q3** — holding BSA at 36 while the book doubled moved the whole ladder ~0.26 scarce; the shift
recentres it on the same band. Pathwise monotone (frozen `U_f`;
survives s_ℓ scaling and the floor); arc set invariant; demand byte-identical.

**τ defined** (v2 never defined it): `τ_v = total forwarder position-volume ÷ book volume`, per week,
**supply/demand convention** (τ < 1 = scarce). Report **nominal** and **effective** (decayed at each
shipment's `book_at`, eligibility-restricted); the **effective** one is gated. Report per-lane, both
dims; τ_v binds.

**v4 default-cell arithmetic** (S60; book = **136.2 LD3-volumes/wk** at 240/wk): BSA **36** + spot
**46·μ** positions ⇒ nominal τ_v:

| μ | positions | τ_v | cell |
|---|---|---|---|
| **1.5** | 36 + 69 = 105 | **0.77** | scarce |
| **2.5** (default — the pre-registered headline, §6) | 36 + 115 = 151 | **1.11** | in band |
| **4.5** | 36 + 207 = 243 | **1.78** | slack |

Tracks v3's ladder (0.87 / 1.20 / 1.88) closely enough to keep the band. *At the unshifted range the
same supply gives 0.60 / 0.94 / 1.62 at μ = 1/2/4 — hence the shift.*

**Note the BSA share moves across the sweep**, which is the point: 36/105 = **34%** of supply in the
scarce cell, 36/243 = **15%** in the slack cell. Contracted is a fixed base; spot is the dial.

(v3: BSA 36 + spot 23·μ on a 68.1-volume book ⇒ 0.87 / 1.20 / 1.88 — arithmetically fine, but 53% of
the book rode free-at-margin; see §11 Q3. v2: floor supply 130 positions ⇒ τ_v ≥ 2.0 at μ=1 — band
unreachable everywhere; found independently by three reviewers.)

Rejected dials: flight count (moves scarcity and routing freedom oppositely on L2); book size
(sets arrivals/cycle — at ~1/cycle M0 ≡ M1p and the estimand ceases to exist; to be re-verified on
the new world).

---

## 6. Experiment

- Generate + route the whole span; **score** only the middle. Out-of-window shipments consume
  capacity — that is the difference between warm-up and truncation.
- **Three cost keys, all exogenous and arm-invariant** (v3 — v2's single flight key let an arm be
  *paid to strand*): `C(arm)` = flight-keyed freight + reservation cost over window flights **+**
  `book_at`-keyed tardiness & fallback penalties over window shipments **+** lane-week-keyed BSA
  take-or-pay deficiency over window weeks. Window = whole weeks ⇒ aligns with contract periods.
  Service (OTP 3-state, tardiness stats, fallback 3-cause [A-carry]) keyed on `book_at`.
- **Headline = the default cell, pre-registered.** The sweep is estimation (curve + simultaneous
  band), not 13 hypothesis tests. Report `(ΔL2, ΔOTP)` jointly — an arm may not buy cost with service.
- Gaps: planning 0.005 (the policy); scoring 0 (assert < 1e-6). The scorer **prices the arm's fixed
  assignment** — it does not re-optimize.
- Inference: two-way random-effects CI (Satterthwaite df), PIH control variate, 2–3 seed pilot to
  size σ(L2) + pre-registered minimum effect of interest. Flow-balance diagnostic per arm
  (booked-in/flown-out vs booked-out/flown-in ≈ 0); report cost per window flight AND per window CW.

---

## 7. Metrics & gates

| | Metric | Gate |
|---|---|---|
| M1 | **Eligible itineraries** per shipment (paths ≤ 3 air legs, via HKG dwell where applicable): (i) fits raw (w,v) at nominal AND decayed-at-`book_at` with r=0; (ii) `ready + ground_out + λ_disp ≤ cutoff − prep` and `book_at ≤ cutoff`; (iii) `ETA + ground_in ≤ deadline` | ≥ 1 per shipment — **hard on the default cell only**; sweep cells report born-dead as a censoring outcome. Tier ladder: **hold `ready_at` and all draws fixed, vary `sla_offset` only** ⇒ eligible set weakly grows (v2's version was false — re-drawing gap(tier) shifts the set). |
| M2 | **Path-flow LP deficiency** (v2's bipartite form couldn't see a feeder consuming two legs): x_{s,p} ∈ [0,1] on eligible paths, kg + m³ rows **per leg**, capacity snapshot = each flight at the latest `book_at` among its eligible shipments (conservative). Necessary-condition screen, not a certificate. | = 0 on the default cell; reported on sweep cells |
| M3 | τ_v (effective) vs μ; per-lane | default cell in **[1.0, 1.3]**; monotone in μ. Recalibration procedure pinned: only ν, φ may move, in that order, re-derived in this doc — never at build time. |
| M4 | PIH fallback; `C(arm) ≥ C(PIH)` | fallback = 0 hard on default cell; **re-runs per experiment** (a MILP change can break it with zero instance diff). Seed rejection counted; cell fails if > 10% (silent regeneration = selection bias toward slack). |
| M5 | Consolidation headroom, **per lane**: median accessed-flight capacity ÷ **mean** shipment CW | ≥ 3. v2 used the median shipment and **passed the one-ULD fleet it was built to catch** (5.9× on medians); the mean (≈475 kg) fails it: 900/475 ≈ 1.9 < 3. |
| M6 | Fallback trend vs book-day within window | report-only |
| **M8** | **Co-load channel share** (new, S60 — §11 Q5): co-load's share of routed CW, **overall and per tier**, per arm, on the default cell | **report-only — but it is the tier-mix tuning trigger.** S59 inverted co-load to cheapest-but-slowest ⇒ deferred cargo is its natural rider ⇒ the 25% deferred pin now sizes the channel. If co-load soaks up the book, the scarcity the estimand is measured against drains away and L2 decays — **the same free-capacity failure family as Q3's BSA share, arriving through a different door.** Unquantifiable pre-rebuild (S59) ⇒ measured, not guessed. **No threshold pinned — pinning one would be fabrication.** Report it, look at it, then retune the tier mix if warranted. |
| **M7** | **Information churn** (new): median # planning runs strictly between `book_at` and commit; per-tier | **≥ 2 overall, hard.** Every other gate is a *static* property; the estimand is *dynamic*. The Monday-batch world (all bookings one weekday) passes M1–M6 with τ in band while L2 = 0 by construction. Deterministic from the arm-free timeline. |

**Gate placement**: `validate_instance()` raises in `load()`; round-trip equality in `persist()`;
**scoring reads only persisted-then-loaded instances** (no in-process bypass); per-run raises
`0 ≤ r_f ≤ avail(f,t) ≤ C_f` per dimension (the decayed catalog is computed at runtime — the seam
where `ready = 0` shipped); post-solve raises: per-flight capacity on the **decayed** view, status
OPTIMAL, no departed flight, no impossible connection. **`instance_card.md` schema pinned**: per-lane
τ_w/τ_v (nominal + effective), position histogram, eligibility by tier and by size decile, churn per
tier, born-dead & resample & seed-rejection rates, belly share, flights-per-day profile, phase of the
schedule bank vs cadence hour.

---

## 8. Design requirements

R1 byte-exact persistence (round-trip equality in `persist()`) · R2 one aircraft one capacity
(Σ offers per `flight_id` ≤ flight capacity) · R3 hard-BSA take-or-pay per lane-week, never pooled ·
R4 scorer refuses non-OPTIMAL · R5 a missed connection is expressible (aircraft never waits; min-
connect per hub [A-carry]) · R6 no flight whose cutoff precedes the sim clock · R7 retirement
predicate · **R8 (new)** fallback chargeable only at `commit_backstop` · **R9 (new)** the headline cell
and estimator are pre-registered in this doc.

---

## 9. Tractability

Measured on the old world: R1 fix = 9.2×; daily cadence = 8–15×; retirement predicate ⇒ linear in
horizon; sweep ≈ one overnight on 10 cores. **v3's span (84 d) and live-book size (~140) exceed the
measured shape — re-measure before committing budget; treat prior numbers as planning estimates.**
Never shrink the bookends; the window may shrink to whole weeks ≥ 14 d.

---

## 10. Annex — carried-over magnitudes (the miner's 28; one line each)

**Deleted for v1:** disruptions + recourse machinery O18 (arrival-only headline; retained as tested
capability, out of scope) · `t_dead` biting deadline O11 (deadline = SLA only) · `booking_promise`
table O7 (deadline is exogenous; score against it) · surcharges stay zero O27.

**Resolved above:** cost basis O5 (§6 three keys) · fallback arrival & PWL span O3 (`T_max`, §3) ·
born-dead handling O10 (resample, §3) · prep/dispatch O9 (in M1(ii): prep 2 h, λ_disp 4 h [A-carry]) ·
base_transit O12 (§3) · H0 definition O20 (§0) · short-fuse O21 (ad-hoc run, §0.7) · max path length
O15 (≤ 3 air legs) · flight-bank clock times O24 (§3) · cargo classes O8 (§3).

**[A-carry] — current magnitudes, not part of the disease, kept:** O1 rate system (spot $5.5 ×
U(0.85,1.18), contract $4.2, min_chg U(80,150), MFB 4-break ladders; family per arc as today) ·
O2 MILP scalars (MAWB fix $50, ε = 0.05, PWL grid, 600 s backstop) · O4 fallback pricing (longest-UB
path × 1.5) · O6 scorer states (3-state OTP, 3-cause fallback) · O13 ground scalars (cartage, CFS
6–8 h $75–98, customs 12 h $200, 60 km/h $0.8/km) · O14 hub dwell (HKG CFS-H 6 h $260 re-group
allowed; ANC 2 h tech stop; mct) · O16 geo candidate rules (k-NN 3 / 1500 km, ellipse φ = 1.3,
ground_group gate) · O17 realization deterministic s = 0 (arrival-only) with frozen actuals tables ·
O19 frozen-arm repair kept as safety net + assert never fired by decay · O22 audit ledger ·
O23 telemetry/provenance schema · O25 CRN stream registry (each new draw gets a named stream) ·
O26 cw_flex report · O28 z_tier (inert at s = 0).

---

## 11. Sign-off — ✓ ALL SIX CLOSED (S60)

**Status: ✓ ALL CLOSED. Q1 ✓ (S57) · Q2 ✓ (S58 + S59) · Q3 ✓ (S60) · Q4 ✓ (S60) · Q5 ✓ (S60) · Q6 ✓ (S60). §11 is done — the design gate is clear.**

### Q1 — the repair fork: ✓ **RESOLVED (S57). Grow the book, don't shrink the allowance.**

v3's own assumption **rejected**. Tightness comes from **a bigger book absorbing a more generous
allowance**, not a small forwarder on a big schedule.

| | v3 (rejected) | **Decided** |
|---|---|---|
| Book | 120/wk | **~240/wk** (≈136 LD3-volumes/wk) |
| Spot access φ | 0.125 (~23 flights/wk) | **0.25** (~46 flights/wk) |
| BSA | ~36 positions/wk | ~~**~72 positions/wk**~~ → **SUPERSEDED BY Q3 (S60): held at ~36.** Doubling it would put 53% of the book in C.13a's free-at-margin zone ⇒ L2 → 0. |
| Ladder | τ_v = 0.87 / 1.20 / 1.88 at μ = 1/2/4 | ~~**unchanged**~~ → **SUPERSEDED BY Q3 (S60).** With BSA held at 36 the ladder is **0.77 / 1.11 / 1.78 at μ = 1.5/2.5/4.5** (sweep range shifted up to recentre the same band). |

**Why:** L2 — the estimand — comes from re-consolidating shipments *against each other*. Consolidation
density is the estimand's fuel, and the bigger book buys it directly. Costs ~2× per solve.

**Three consequences:** (a) **Q3 (BSA shape) is now load-bearing, not cosmetic** — ✓ **RESOLVED S60:
BSA does not scale with the book; held at ~36.** Q1 was right that Q3 was load-bearing and wrong about
*why* — the reason is not that ~5 positions/departure is fat for an SME (that figure was itself an
artifact of a §3/§4 arithmetic contradiction), it is that a 53% contracted share kills the estimand.
(b) the commercial story is **UNVERIFIED** at 240/wk (~6–7 kt/yr = mid-size; the `[S]` tag on 120/wk
does **not** carry over — do not assert until sourced) — **still open**; (c) **tractability (§9) is the
sharp end** — ~2× shipments per cycle on a MILP with a history of falling over on hard seeds — **still
open, and now the biggest single risk to the rebuild.**

**§3 and §5 re-derived for the 240/wk book — ✓ DONE S60** (with Q3; §3 book/BSA/spot-access/span rows,
§4 supply block, §5 arithmetic table).

### Q2 — reservation pricing: ✓ **RESOLVED (S58; co-load sub-item closed S59 — Q2 fully closed, zero new columns).**

**Why it matters.** Free release before cutoff ⇒ reserve-everything-then-release **strictly dominates** ⇒
the reservation decision goes degenerate ⇒ the entire S54 reserve-early/assign-late mechanism collapses
into "always reserve the max." **The friction is what makes the reservation a decision** — load-bearing for
the estimand, not decoration.

**Ground truth (the user's own forwarding experience — it OVERRIDES the S52 desk research):**
> "No free lunch. You book what's available, but once you book you can use ±20%. If you decide not to use
> it at all you pay a 35% penalty. It's not ok to consistently over-reserve, under-allocate, and cancel
> right before cutoff — after a while the carrier won't give you space."

Two clauses. **"You book what's available"** — the *creating* cycle is bounded by physical space, not by a
band. **"Once you book you can use ±20%"** — from then on, the floor binds.

#### The formulation (final)

Per **master air waybill** `(a, g)` carrying a live reservation from a prior cycle.

**Parameters**, frozen at solve time from `ReplayState._reserved_spot` (set by a prior cycle's post-solve
ratchet — *this is why nothing is bilinear*): **`R_{a,g}`** = reserved envelope in billing chargeable kg
(**a CONSTANT in this solve**) · **`β = 0.20`** (band half-width, sweepable) · **`ψ = 0.35`** (no-show
fraction).

**New variables: none.** `z_{a,g}` — the **existing** binary "this MAWB is instantiated" — *is* the
used-at-all switch.

**1. The floor is a CONSTRAINT ON USAGE, not a billing rule.**
```
R.1   CW_{a,g}  ≥  (1 − β) · R_{a,g} · z_{a,g}
```
You **must** load at least 80% of what you booked. You always pay **100% of what you actually load** —
there is no phantom shortfall charge. Either use ≥ 80%, or don't use the booking at all.

**2. The no-show is the escape hatch.**
```
objective += ψ · family_cost_a(R_{a,g}) · (1 − z_{a,g})       # family_cost_a(R) is a CONSTANT
```
If only 500 kg wants the flight and you booked 1,000, the 800 kg floor is **unreachable** — the optimizer
takes `z = 0` and eats the 35%. **Without the no-show, R.1 would make the solve infeasible.** The escape
hatch is structural, not decoration. Per booking it is a clean either/or: `z=1` ⇒ load between `0.8·R` and
`avail`, pay for exactly what is loaded; `z=0` ⇒ load nothing (already forced by the existing
`C4ub: CW ≤ CW^ub·z`), pay `ψ · family_cost(R)`.

**3. The ceiling already exists — no new row.** The real ceiling is physical availability, which S54 §5.3
already has: `avail^w_a(t) = R + (C^w_a − R) · φ_a(t)` — your prior booking (protected from decay: it's
yours) **plus** whatever is left of the free pool after the booking curve has eaten into it. First cycle
`R = 0` ⇒ the ceiling is the decayed free space = **"book what's available."** Enforced by the **existing**
`C.5d` capacity row. ⇒ **The `+20%` is not a rule.** It is a *description* of what typically happens, not a
constraint we write. **Only the floor disciplines.**

**4. The ratchet IS booking amendment.** S54's post-solve `R ← max(R, used)` turns out to be a real,
sourced operation — amending a booking upward. Not a modeling convenience.

**Cost of the whole mechanism: one new row (R.1) per live booking + one linear objective term. ZERO new
columns.** Linear everywhere (`R` is a parameter at solve time) ⇒ no bilinearity, no big-M, **no
tractability risk at the 240/wk book.**

**It self-disciplines hoarding with no extra machinery.** Booking five flights means five `0.8·R` usage
floors ⇒ blanket speculative reservation must be *filled* five times over. **No portfolio cap needed.**

#### Keyed per MAWB, not per flight

**`R` must be `R_{a,g}` (per master air waybill), NOT `R_a` (per flight).**

**Sourced [S] (S58 research):** spot booking : MAWB = **1:1**; allotment/BSA : MAWB = **1:many** — and that
is exactly the spot/BSA distinction.
- IATA Cargo-IMP booking message is **"FFR — AWB Space Allocation Request"**: books space *for a nominated
  AWB number*, drawn from the forwarder's own AWB stock. (arc.cdata.com/edi/standards/imp/;
  parse2.com/service-cargoimp.shtml)
- IAG Cargo amend flow: *"Select the AWB that you would like to amend."* You find a booking **by its AWB**.
  (iagcargo.com/en/e-booking-guide/)
- Emirates no-show: *"the AWB will be frozen and cannot be reused. The applicable No-Show fee will be
  charged to the AWB used for the booking."* (skycargo.com, US LSC 01 Jan 2025)
- Allotment side: *"requests from the forwarder arrive one by one; accepted if within the allotment"*
  (Amaruchkul; Levin/Nediak/Topaloglu).

⇒ **S54's per-flight pooled envelope `r^w_a` is ALLOTMENT semantics applied to SPOT.** It is wrong and must
be re-keyed. *Same failure family as the S56 bug: an abstraction that quietly makes something free.*

**Key stability verified** (a reservation from cycle 1 must still mean the same thing in cycle 5):
`group_key(hawb)` = `f"{cargo_type}:{temperature}"` (`air_graph.py:1404-1414`) — a function of **fixed**
HAWB attributes, nothing from the solve.

**Assumption to record, not bury [A]:** the group key gives **one MAWB per cargo class per flight**. The
model therefore cannot represent two separate general-cargo bookings on the same flight made on different
days — it represents **amending** the first instead. Supported by the amendment finding, but not proven
universal. Benign for the estimand (double-booking is strictly *more* expensive, and equally so for both
replan arms).

#### ✓ Co-load — CLOSED (S59): buy-at-cutoff, zero machinery — and remodeled as a lane-SLA channel

**User decision:** co-load carries **no reservation machinery** (no `z`, no floor, no no-show) — you
tender what you have, they take it space-permitting. **Q2 therefore closes at zero new columns.**

**And the channel itself was remodeled** (user decision, S59, on a verified research pass — every claim
below fetched, not snippets):

| Axis | Old model (wrong) | New model (S59) |
|---|---|---|
| Key | per flight (offer), cutoff = STD − 6 h | **per (lane, day) SLA service**; "departure" = the co-loader's daily receiving cutoff at its warehouse |
| Promise | a seat on flight X | **lane SLA**: arrival = receiving cutoff + `SLA_ℓ` = direct transit + **2–3 days** (per-lane constant [CAL]; user: SHA–LAX ~2 d, SHA–CHI ~3 d). Deterministic at the SLA — rolling to the next consolidation is *absorbed* by the promise (arrival-only methodology preserved) |
| Capacity | `spot_wcap` + booking-curve decay | **generous finite per-day acceptance cap** (C.5d = "space permitting"); **NO decay** — decay models visibility of a specific flight filling; day-granular arcs + caps replace the curve (full day ⇒ tomorrow's arc, +1 day) |
| Price | top band (legacy U(3.5,5.5)) / same as spot (lane path) | **`base_spot × U(0.80, 1.00)` per lane [CAL]** — below airline spot **[S]**, at-or-above own-BSA rate **[A]** (research silent; if co-load beat your own BSA and time sufficed, holding the BSA book would be irrational) |
| Billing | per-kg on own `cw_k` | unchanged — and *correct by construction*: the MAWB is the co-loader's |

**Sourcing (fetched, S59 research agent):**
- **Lane service, not a flight [S]:** AMI (air wholesaler) sells destination-level consolidation;
  named-carrier is a separate premium product (Back2Back). BIFA STC 4(B): "full liberty as to the means,
  route and procedure"; STC 25: no responsibility for departure/arrival dates.
  (airmenzies.com/services/air-exports/ + uktradingconditions.pdf)
- **Receiving cutoff [S], thin (one datapoint):** EP America (FRA consolidator): cargo at their airport
  warehouse **≥ 6 h before the airline cutoff**; weekly consols close Friday, fly Saturday.
  (ep-america.com)
- **No F2F booking friction found:** wholesale co-load is account-based quote-book-tender (AMI portals;
  master loaders on WebCargo); **no forwarder-to-forwarder no-show/cancellation fee documented anywhere**
  (absence of evidence — flagged, not [S]). Friction sits on the airline↔MAWB-holder interface = the
  co-loader's problem.
- **Cheaper than spot/direct [S]:** NAC "lower rates than offered by the carrier"; ExFreight consolidation
  "15–30% less than individual direct bookings" at +1–2 d transit; EP America "fixed rate lower than spot
  market". **Refutes the old most-expensive-channel treatment** for the small-forwarder case; silent on
  the position vs a mid-size forwarder's own contract rate ⇒ the [A] above.
- **Bonus [S]: ψ = 0.35 re-sourced** — AF-KLM Cargo local conditions: no-show fee "35% of the total All-in
  Rate… or USD 0.28/kg, whichever is higher"; cancellation <24 h of LAT: 15–25%. (brix.afklcargo.com
  MALE_LOCAL_CONDITIONS_02NOV22.pdf) Replaces the retired LH/AA figures.

**Role inversion, eyes open (user accepted):** co-load flips from expensive-last-resort to
**cheapest-but-slowest**. Deferred-tier cargo becomes its natural rider; spot narrows to fast-and-poolable.
This moves where the L2 consolidation pressure comes from — unquantifiable pre-rebuild, recorded so it is
a decision, not a surprise.

**Generator implications (feeds §4 rebuild spec):** co-load offers are emitted per (lane, day), not per
flight; excluded from `CapDecay`; per-day cap drawn generously; rate anchored per lane. Tex is amended
(S59): §arc-types, §air-arc-params, §supply-option-catalog, C.5d note.

**Side-finding (S59, flagged for user judgment — bug or intent?):** the lane rate catalog **flattens MFB
break rates to a single rate** — `mfb[a] = [Break(b.threshold, rate) for b in _gen_breaks(rng)]`
(`air_generator.py:780`) keeps the ascending thresholds but prices every break at the same `rate`, so the
IATA next-break-down discount is inert on the lane path (the legacy path keeps descending rates). Not
judged, not fixed — listed for the rebuild.

#### Dead / superseded — record the departures, do not silently drop

- **S57's `rate_a · q_a` shortfall charge** — dead. `min_flat_breaks` arcs have **no scalar $/kg**; the
  rate is whichever break `γ` selects, a *decision* not data (`air_milp.py:775-789`). Never writable.
- **S57's `q_a` shortfall column and `y_a` binary** — dead. `z_{a,g}` already is the switch.
- **S57's `CW ≤ (1+β)·R·y` band ceiling** — dead. *Bootstrap bug:* on the creating cycle `R = 0` ⇒ ceiling
  is 0 ⇒ the arc can never be used ⇒ the ratchet never fires. Dead on arrival.
- **S57's "reuse `FLATmin` / `MFBfloor`" idea** — dead. Those are **billing** floors; R.1 is a **physical
  usage** floor. Different object.
- **`penalty_frac = 1` alone** — superseded (null L2 unidentifiable: no value vs *confiscated* value).
- **The portfolio cap** `Σ reservations ≤ (1+β) × live cargo` — superseded by the usage floor.
- **Free release before cutoff** — killed by the user. **Record the departure from the S52 grounding:** LH
  25% under 48 h / 50% under 24 h, free before cutoff. *Caveat:* those LH numbers **and** the AA $300 are
  **UNVERIFIED** — both sites 403'd under S58 re-check. Do not lean on them.
- **Parked as v2:** the reputation/allowance ratchet (carrier shrinks φ/μ/BSA after repeated abuse) — the
  *true* long-run mechanism, but it makes supply a function of forwarder **behavior**, the same family as
  the bug (supply ← the book) that cost this project 16 sessions. Only with eyes open, data-path gates first.

### Q3 — BSA shape: ✓ **RESOLVED (S60). Hold BSA at ~36 positions/week — do NOT scale it with the book.**

**The decision.** BSA stays at **~36 positions/week**, the v3 figure. It does **not** double with the
book. Shape: **4 dominant lanes × ~7 departures/lane-week × 1–2 positions** (ν = 1.3 literal) =
**36.4 positions** — roughly daily contracted service on each of four core lanes. Share of the book
falls **53% → 26%** of LD3-volume. Ladder re-derives; sweep **μ ∈ [1.5, 4.5]** (was [1.0, 4.0]) to
recentre. See §3 (BSA row), §4 (supply block), §5 (arithmetic).

**Why — the user's argument, and it is the binding one.** The reasoning is *not* commercial
plausibility (which is where S57 framed it, and where I opened the walk). It is that **the estimand
stops existing**. C.13a makes chargeable weight from 0 up to the allowance `A_c` **free at the
margin** — sunk via the take-or-pay, dropped from the optimizer's argmin (tex line 1559). At 53% of
the book riding contracted, over half the book carries **no cost gradient**. No gradient ⇒ nothing to
re-optimize ⇒ M1′ ≡ M1 ⇒ **L2 → 0**. BSA share was never a realism question; it is a lever on whether
the thing we are trying to measure exists at all.

**The symmetry worth remembering.** S45 measured the *opposite* failure: contracted was **never** used
at proof scale, because the pivot floor made it dearer than spot. Doubling BSA would have swung us from
never-used to used-for-everything. **Both are calibration artifacts, not findings.** 26% sits between
them — but this is an assumption **[A]**, not a measurement, and it is on the Q6 re-measure list.

**Two options were considered and rejected as cosmetic.** (a) Spread 72 positions over 7–9 lanes;
(b) keep 3–4 lanes but raise departures. Both fix how the *shape* reads (positions per departure) while
leaving the contracted **share** at 53% — same free-at-margin zone, same dead L2. (a) is also already
rejected in §3 as v2's shape (thin lanes, 55% unreachable, take-or-pay dead freight ≈ book revenue),
and a forwarder does not sign BSAs on its thin lanes. A fourth option — sweep the contract/spot mix as
its own axis — directly measures the concern but adds a sweep dimension, and §9 tractability is already
the sharp end. **Parked, not dismissed** (v2 candidate).

**Defect found while re-deriving — §3 and §4 never agreed, and this predates Q1.** §3 claimed
`3–4 lanes × ~4 dep/lane-wk × 1–2 positions ⇒ ~36 positions/wk`. But 3.5 × 4 = **14** contracted
departures, and §4 pins **ν ≈ 1.3** ⇒ 14 × 1.3 = **18 positions, not 36**. Landing 36 on 14 departures
needs ~2.6 each, contradicting "1–2 positions" in the same row. **36 is the load-bearing number** (§5's
ladder is built on it: 36/68.1 = the 53% figure); the *shape* was the broken half. S57's alarming
"~5 per departure" was computed off the 14-departure reading and is an artifact of this inconsistency.
Corrected to 4 lanes × ~7 dep × ν=1.3 = 36.4.

**The `[NF]` on BSA size/lane-week (§2, line 71) is NOT cleared and must not be read as cleared.** No
public source exists for what a mid-size forwarder's BSA looks like, and S60 did not find one — it
resolved an internal arithmetic contradiction and chose the *share* on estimand grounds (L2 survives).
The shape is defensible and self-consistent; it is still **[A]**, still unsourced, and still on Q6.

**Bonus: one demand back-door closes.** §4 previously sized ν to hit a 55%-of-book share — a one-time
scale-setting frozen as a literal, but still supply reading demand. Holding ν = 1.3 from v3 means the
share now **falls out** of the arithmetic instead of being targeted. Strictly better; supply ⟂ demand
is cleaner than it was.

### Q4 — warm-up: ✓ **RESOLVED (S60). 30 d, proven by a 59 d control cell — not by the gate.**

**The decision.** Warm-up = **30 d** for the sweep. Its sufficiency is **measured**, not assumed: run the
**default cell (μ = 2.5) at 59 d warm-up, one seed, once**, as a control. If the headline matches the
30 d run, 30 d stands for the whole sweep. If it does not, warm-up goes to 59 d and the sweep re-runs.

**Why not simply 59 d everywhere.** ~+30% simulation cost on top of Q1's 2×, with §9 tractability
already the sharp end.

**Why not 30 d + the stationarity gate as v3 specified it.** The gate compares the **last warm-up week
against the window weeks, per arm** — 1 week vs 3, on a noisy cost series. That test has almost no
power: it will pass nearly regardless, and "failed to detect a trend" is not "there is no trend." **A
cheap option guarded by a gate that cannot fail is exactly the S56 failure family** (an abstraction that
quietly makes a problem disappear). The control cell costs **one cell rather than +30% on every cell**,
and it converts an untested assumption into a number.

**The gate survives as a diagnostic** (cost-rate & reservation-depth trend) — reported, never trusted,
and never load-bearing on its own.

**Pre-register the comparison** with the rest of the headline (§6): the 30 d-vs-59 d control is a
check on the design, not a result to be reported selectively.

### Q5 — pinned constants: ✓ **RESOLVED (S60). All five pinned as-is; the tier mix gets a measured tuning trigger.**

**Pinned unchanged, all `[A]`, all sweepable later:** weekday weights (Mon–Fri .18, Sat .06, Sun .04 —
kills the Monday-batch pathology M7 exists to catch) · **W = 24 h** (without it express, at a 12 h
offset, is structurally born-tardy and M1 rejects valid worlds) · **T_max = 96 h** (pins the C.10 PWL
span and the fallback arrival) · **K = 3** (the M5 consolidation-headroom threshold).

**Tier mix 20 / 55 / 25 E/S/D — pinned, with a trigger.** `[A] [NF]`: no source, and none was
fabricated. It was harmless when v3 pinned it; **S59 made it load-bearing.** Co-load is now
cheapest-but-slowest, so deferred cargo is its natural rider, and the 25% deferred share now sizes the
co-load channel rather than merely describing demand. Too much deferred ⇒ co-load soaks up the book ⇒
scarcity drains ⇒ L2 decays (**the Q3 failure family through a different door**). Too little ⇒ the
channel S59 spent a session modeling is barely exercised.

**User's call: 25% stands; retune later if too much flows to co-load.** Made concrete as **M8**
(§7) — co-load's share of routed CW, overall and per tier, per arm, report-only on the default cell.
**No threshold is pinned; pinning one would be fabrication.** The point of M8 is that "we'll check
later" has to be something the run actually prints — S56's root lesson is that nobody printed the
instance.

**Check that fell out:** the new default cell (μ = 2.5, τ_v = 1.11) passes **M3**, which requires the
default cell in [1.0, 1.3].

### Q6 — the re-measure list: ✓ **CONFIRMED COMPLETE (S60).**

A consequence, not a decision. Everything the rebuild invalidates and must re-derive **before anyone
quotes it**. Nothing here may be asserted off a pre-rebuild number.

| | Item | Why it's void / its status |
|---|---|---|
| 1 | **Every headline** — L1, L2, OTP, fallback | computed on the physically impossible instance |
| 2 | **τ / κ bands** | τ was definitionally inert (`capacity := τ × n_hawbs × E[cw]`) |
| 3 | **The C.10 tardiness weights** | CLAUDE.md hard rule — anchored to a fallback gap priced off broken capacity |
| 4 | **Decay parameters** (`A_cut` Beta, freighter/belly) | calibrated against the broken supply |
| 5 | **Belly split** (~15%) | `ac_type` hardcoded FREIGHTER on load (D2) — 28 belly flights → 0 |
| 6 | **The S45 L2 decomposition** ("100% consolidation, 0% capacity") | capacity couldn't bind ⇒ forced |
| 7 | **"M0 ≡ M1p at 1 arrival/cycle"** | re-verify on the new world; daily cadence may give L1 content |
| 8 | **BSA share = 26%** `[A]` (Q3) | chosen on estimand grounds, not sourced; `[NF]` on BSA size stands |
| 9 | **30 d warm-up** (Q4) | proven by the 59 d control cell — a measurement, not a pin |
| 10 | **Tier mix 25% deferred** (Q5) | `[NF]`; retune on M8 if co-load soaks up the book |
| 11 | **Co-load rate + SLA** `[CAL]` (S59) | `base_spot × U(0.80,1.00)` and +2–3 d never exercised |
| 12 | **ψ = 0.35 [S], β = 0.20 [CAL]** (S58/S59) | ψ sourced; β untested |
| 13 | **Commercial story at 240/wk** (Q1) | ~6–7 kt/yr **UNVERIFIED** — the 120/wk `[S]` tag does not carry over |
| 14 | **MFB break-flattening** (`air_generator.py:780`) | S59 side-finding — bug or intent, user judges at rebuild |

**Survives the rebuild:** MILP formulation + MFBlink cut, FreightNet topology, cadence work, all
grounding research (ratios, never magnitudes).

---

## §11 — CLOSED (S60). All six questions answered. The design gate is clear; the rebuild may begin.
