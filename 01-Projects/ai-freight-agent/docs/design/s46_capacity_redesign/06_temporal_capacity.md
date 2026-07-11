# S46 Capacity Redesign — Temporal Capacity & Arrival-Timing (air slice)

**Status:** DESIGN PROPOSAL — behind the formal-model approval gate. **DESIGN ONLY**; no `src/` /
`tests/` / `data/` code is written until the user approves. Static arithmetic probes only (cited
inline).

**Author role:** Temporal Capacity & Arrival-Timing Architect. Consumes `01_architecture.md` (the
*spatial* τ_ℓ design), `05_redteam.md` (the FATAL F1/F2 findings), `l2_decomposition_s45.md` (the
killer measurement), `arrival_only_replan_methodology.md` §13, and the live `air_generator.py` /
`replay.py` / `air_milp.py`. Every quantitative claim is tagged **SOURCED** / **INFERRED** / **MRN**
(market-research-needed).

---

## LOCKED DECISIONS (user, S46) — supersede the proposal where they conflict

- **D-T1 — Booking-curve shape = LINEAR.** Spot (and decaying contracted, D-T2a) available capacity
  falls at a steady rate toward departure: `φ(Δt) = φ_min + (1−φ_min)·clip(Δt/H, 0, 1)` (β=1). No
  last-day cliff.
- **D-T2a — Decay is tied to CONTRACT TYPE, not a blanket "contracted protected":**
  - **Hard / sunk-cost BSA** (`equalized`, pre-paid allowance `A_c`) — **does NOT decay.** It is
    reserved take-or-pay space; it is yours regardless of clock. This is the stable reshuffle anchor.
  - **Soft / pay-to-use BSA** (`per_flight`, no sunk block) — **DOES decay**, exactly like spot,
    because it is shared market capacity other forwarders can also tap; availability falls toward
    departure.
  - **Spot** — decays (unchanged from proposal).
- **D-T2b — Per-flight jitter on the decay curve** (each flight draws its own `φ_min`/rate; one
  network-wide curve rejected).
- **D-T3 — NO planted/hard-wired scenario.** Do not force a captive late-express into the kill-shot.
  Instead **scale `n` up** so the EXPRESS count is large enough that P(≥1 very-late express) is high
  by natural draw (`E≈12–16` express ⇒ P≈0.95+ at a ~0.25 very-late tail). Realistic arrivals only.

Implication of D-T2a for the value mechanism: the bump TARGET that M₁ frees for the late express must
be a **hard-BSA** position (it doesn't decay, so it's still there when the express arrives late);
soft-BSA and spot have partly evaporated by then. So the reshuffle is: M₁ moves an early DEFERRED off
a hard-BSA position to a later flight and seats the late EXPRESS there; M₁′ (froze the deferred's
placement) cannot, and spills the express to fallback. Hard BSA = the reservoir; soft BSA + spot =
the depleting market.

---

---

## 0. What this document fixes (one paragraph)

The spatial design (doc 01) makes capacity **finite** but **static in time**: a flight is given the
same capacity regardless of how close to departure it is viewed. Red-team F1/F2 then proved that
finiteness alone does *not* break S45: both replan arms are cost-minimizers facing the *same* finite
cheap capacity, so both fill cheap-first and spill the **same total kg** to a flat unbounded fallback
⇒ `L2_capacity ≈ 0`. The arm difference (M₁ can reshuffle an untendered prior; M₁′ cannot) is a real
*channel* but it only **binds** when cheap capacity is genuinely contested at the moment a captive
high-value shipment needs it — and at proof scale, with a couple of our own shipments against a
static ~1.5–4.5k kg spot cap, that contention is a low-probability accident (F2: "rare at proof
scale"). This document adds the two missing real-world mechanisms — **(A) capacity that decays toward
departure** and **(B) an arrival spread with a late-express tail** — that turn that accident into a
*designed-in*, deterministic contention: the late express finds the spot escape **closed by the
booking curve** exactly when it arrives, so the only cheap seat is the contracted ULD a deferred
shipment is squatting on. M₁ bumps the (still-untendered) deferred to a later flight; M₁′ has it
pinned and spills the express to fallback. The arms then route **different total kg per tier** —
the precise quantity F1/F2 require and S45 never produced.

---

## A. Time-decaying capacity model (the booking curve)

### A.1 Functional form

For an arc `a` belonging to a flight departing at absolute time `T_a`, viewed at decision clock `t`,
the **available-to-book capacity** is

```
  K_a(t)  =  max(  C0_a · φ_a(T_a − t) ,  firm_floor_a(t)  )
```

- `C0_a` — the **time-zero amplitude**: the spatial initial capacity from doc 01 (`τ_ℓ · D_ℓ` split
  across tiers). Drawn in generation, frozen across arms (D-A16).
- `φ_a(Δt) ∈ [φ_min, 1]` — the **booking curve**: the *fraction of `C0_a` still available* when the
  flight is `Δt = T_a − t` hours from departure. Monotone non-decreasing in `Δt` (more time → more
  space). The **exogenous** source.
- `firm_floor_a(t)` — the chargeable-kg our own **tendered (firm)** cargo already holds on `a` at
  time `t`. The **endogenous** source / live-ledger floor (A.3).

Proposed shape (a standard revenue-management booking curve — convex, slow-early/fast-late
depletion):

```
  φ_a(Δt)  =  φ_min  +  (1 − φ_min) · clip(Δt / H_a, 0, 1)^β
```

- `H_a` — the **booking horizon** over which spot fills (e.g. 120–168 h ≈ 5–7 d). **INFERRED**;
  align with `backstop_buffer_h = 168 h`.
- `β ≥ 1` — curvature. `β = 1` linear; `β > 1` back-loads the depletion (most space vanishes in the
  last day, the realistic regime). Shape **INFERRED**; magnitude **MRN** — I have **not** located a
  citable air-cargo spot booking curve, so the convexity is a modelling assumption, not sourced.
- `φ_min` — residual free-sale fraction at departure (never exactly 0; carriers hold a sliver).
  Default `0.1`. **INFERRED**.

Static probe (linear `β=1`, `H=120`, `φ_min=0.1`, `C0=1500 kg`): `Δt=96h → K=1230`, `48h → 690`,
`12h → 285`, `2h → 172`. A late express (`Δt≈12h`) sees ≈19 % of the spot pool — the closed escape.

### A.2 The two sources and their balance

| source | what it represents | applies to | how drawn | per-arm? |
|---|---|---|---|---|
| **exogenous** `φ_a(·)` | the background market booking the free-sale pool over time | **spot only** | on the `supply` / new `cap_decay` stream, in generation, **never reading the demand book** (D-A18 discipline) | **no** — identical across arms (frozen with `C0`) |
| **endogenous** `firm_floor_a` | our own tendered cargo permanently holding capacity | spot + contracted | not drawn — it is the live consequence of our tenders | **yes** — differs across arms |

**Decision — contracted (BSA) does NOT decay; spot does.** A BSA allotment is *reserved* space:
take-or-pay positions the forwarder owns up to the cutoff whether or not used. Modelling contracted
as time-invariant (the spatial `N_f`, doc 01) is both realistic and **resolves the retroactive-
infeasibility trap**: if a decaying cap could shrink below cargo we already firmed, the pinned
tendered set would become infeasible mid-replay. Decaying only the *unreserved spot* pool, and
flooring `K_a(t)` at our firm holdings, makes a firm booking permanent by construction. **SOURCED**
mechanics (BSA = reserved, protected); the decay-spot-only split is **INFERRED**.

**Balance.** Exogenous is the **primary** driver of the capacity component of L2 — it is what closes
the cheap escape at the late-arrival moment *independent of how few of our own shipments are on the
flight*, which is exactly the proof-scale binding F2 says is otherwise rare. Endogenous is secondary:
it makes the ledger honest (firm cargo really is gone) and protects firm bookings, but it cannot by
itself carry an arm difference (both arms firm the same cutoff cargo — W4).

### A.3 Interaction with the spatial τ_ℓ (doc 01)

**Complement, not replace.** τ_ℓ sets the *amplitude* `C0_a` of each tier at time zero (how scarce the
lane is overall); φ_a modulates the *spot tier's availability over the decision clock* (how much of
that is left when you look late). They are orthogonal:

- **τ_ℓ** stays **primary** for the *standing scarcity level* — fallback incidence, the network-tight
  vs slack regime, the metric-3 story.
- **φ_a** is **primary** for the *capacity component of L2* — it is the lever that makes the
  reversibility channel (M₁ vs M₁′) actually *bind* at proof scale.

A short lane (`τ_ℓ < 1`) with a back-loaded curve (`β > 1`) is the cell where both bite: scarce
contracted (spatial) creates the single-ULD contention; the decayed spot (temporal) closes the
escape that would otherwise absorb the late express.

---

## B. Arrival-timing spread model

### B.1 What's wrong with the current two modes

`_gen_arrivals` offers exactly two book-lead regimes (`air_generator.py` L866–873):

- `tier_coupled_arrival=False` (headline): `B ~ U(48±24)`, **tier-independent** — timing decoupled
  from tier entirely; no late-express signal.
- `tier_coupled_arrival=True`: per-tier means `(12, 48, 96)` ± a single spread — pins EXPRESS
  *uniformly* late (mean 12 h) with **no within-tier spread**: there is no early-express and no
  very-late-express tail; every express looks the same.

Neither produces what the value thesis needs: a genuine spread **within** a tier, with a heavy
late tail on EXPRESS so that *some* express shipments hit a depleted book.

### B.2 Proposed design — per-tier lead-time bucket mixture

Replace the binary flag with a **lead-time bucket mixture**. Define four buckets over book-lead `B`
(hours before the d* cutoff):

| bucket | `B` range (h) | meaning |
|---|---|---|
| early | [96, 144] | planned well ahead |
| medium | [48, 96] | normal |
| late | [12, 48] | short-fuse |
| very-late | [`min_b`, 12] | just-in-time (`min_b = prep+dispatch = 6 h`, the existing floor) |

Each tier carries a **bucket-weight row** `π_tier` (a 3×4 matrix, the one load-bearing knob). The
draw: `bucket ~ Categorical(π_tier)`, then `B ~ U(bucket_range)`. Illustrative weights (**INFERRED**;
the heavy-tail magnitudes are **MRN**):

| tier | early | medium | late | very-late |
|---|---:|---:|---:|---:|
| EXPRESS | 0.15 | 0.25 | 0.35 | **0.25** |
| STANDARD | 0.25 | 0.40 | 0.25 | 0.10 |
| DEFERRED | **0.50** | 0.30 | 0.15 | 0.05 |

This gives (i) a genuine **early/medium/late/very-late spread**, (ii) **within-tier** spread —
EXPRESS has 15 % early *and* 25 % very-late (the tail the thesis needs), (iii) DEFERRED skewed
early — the natural **bump candidate** that arrives long before its cutoff and sits untendered.

The relationship between tier and arrival-bucket is a **per-tier categorical** (option chosen over a
fully-independent 2-D mix because it lets EXPRESS carry a *designed* heavy late tail while keeping a
small early mass — a clean single knob). The within-tier slack heterogeneity (`t_dead_prob`) is
unchanged and composes with this.

### B.3 CRN / supply-independence preservation

- The bucket draw + the within-bucket uniform are drawn on the **`demand`** stream, in a **fixed draw
  order and fixed draw count** (always draw the categorical and the uniform; the bucket only selects
  the range) — mirroring the existing `t_dead` Bernoulli-then-uniform discipline (L876–887). So
  varying any *capacity* knob (τ_ℓ, φ, β, `H`) leaves the demand realization **byte-identical** (the
  hard CRN gate).
- The booking curve φ is drawn on a **supply-side** stream and never reads the realized book — D-A18
  preserved.

---

## C. The value mechanism, made explicit (worked mini-scenario)

**Setup** — one short lane TPE→LAX at `τ_ℓ ≈ 0.7`, booking curve linear (`H=120, φ_min=0.1`),
contracted re-anchored to **$4.2/kg** (< spot base **$5.5/kg**), fallback **$13.75/kg** (2.5×).
Two flights:

- **F1** — dep `T=120 h` (day 5): 1 LD3 contracted ULD (1500 kg @ $4.2, pivot π≈1200), spot
  `C0=1500 kg`.
- **F2** — dep `T=144 h` (day 6): 1 LD3 contracted ULD, spot `C0=1500 kg`.

Two HAWBs (both 1400 kg chargeable):

- **D** — DEFERRED, known `t=24 h` (early bucket), tender/cutoff `t=110 h`, `Δ_D=160 h`
  (slack: F1 *or* F2 both deliver in time).
- **E** — EXPRESS, known `t=108 h` (very-late tail), tender/cutoff `t=118 h`, `Δ_E=128 h`
  (tight: only F1 arrives in time; F2 arrives ≈150 h, too late). **Captive to F1.**

**Replay trace:**

- `t=24` (only D visible): cheapest seat for D = F1 contracted ULD ($4.2). D placed on F1-contracted.
  - **M₁′** pins D on F1-contracted (frozen at first placement — `_plan_cycle` pins all priors).
  - **M₁** commits D on F1-contracted but **untendered** (D's cutoff is 110 > 24 → reshuffleable).
- `t=108` (E arrives): F1 contracted ULD has 100 kg free (D holds 1400). F1 spot `K=C0·φ(12)=285 kg`
  — **decayed shut**. F2 is too late for E. E (1400 kg) fits nowhere cheap on F1.
  - **M₁′**: D is pinned ⇒ E spills to **FALLBACK** — 1400 kg, misses `Δ_E`.
  - **M₁**: D still untendered (cutoff 110 > 108, the committed-but-untendered window) ⇒ reshuffle D
    → F2-contracted (D's `Δ_D=160` allows it), freeing F1's ULD. E → **F1-contracted**, on-time.
- Cutoffs fire; both arms tender their final routes.

**Per-tier kg totals (the F1 kill-shot quantity):**

| tier | M₁′ | M₁ | Δ (M₁′−M₁) |
|---|---:|---:|---:|
| contracted kg | 1400 | 2800 | **−1400** |
| spot kg | 0 | 0 | 0 |
| fallback kg | **1400** | **0** | **+1400** |

**Costs** (static probe): `C(M₁′) = 1400·4.2 + 1400·13.75 = $25,130`; `C(M₁) = 2·1400·4.2 =
$11,760`; **`L2_capacity = $13,370`, 100 % capacity-driven, 0 % consolidation** — and OTP differs
(E late under M₁′, on-time under M₁). This is the airtight rebuttal to F1/F2: `total_fallback_kg(M₁′)
= 1400 ≠ 0 = total_fallback_kg(M₁)`, and the contracted totals differ — the arms route *different
total kg per tier*.

**Where it can still fail (stated honestly):**

1. **Cutoff ordering.** If D tenders *before* E arrives, D is firm in *both* arms ⇒ M₁ can't bump ⇒
   reverts to F1. The express late-tail must land **inside** the deferred's open-book (untendered)
   window. This is a *coupling* between B.2's express late-tail and the deferred's cutoff — a
   calibration the kill-shot must verify, not assume.
2. **Captivity.** If E can take a later flight (loose `Δ_E`) or a slack lane (endogenous routing,
   red-team S2), it routes around and no contention arises. The express tail must be paired with a
   *tight deadline on a short lane* — short-lane + very-late + tight-`Δ` is the captive cell.
3. **Decay strength.** If φ is too shallow (β≈1, large `H`), spot at `Δt=12h` is still big enough to
   absorb E ⇒ no reshuffle needed ⇒ F1 reproduced. The curve must close the escape at the tail.
4. **Scale dilution.** At proof scale we must *guarantee* ≥1 such (D, E) contest per short lane by
   construction (the C0/kill-shot cell), not hope the random stream produces one.

---

## D. Integration with the replay loop

The decomposition (per-cycle solve, pins, scorer) is preserved verbatim — tractability depends on it.
Changes are localized:

1. **Capacity-as-function-of-decision-clock.** Today `run_replay` loads one static `rates` and feeds
   it to every cycle's `solve` (L664, L803). Add a per-cycle **decayed rate view**: before each
   cycle's solve at clock `t`, derive `rates_t` with `spot_cap[a] = max(C0_a · φ_a(T_a − t),
   firm_floor_a)`. **Contracted `N_f` is untouched** (no decay). `_build_spot_cap` reads
   `rates.spot_cap[a]` directly (verified, `air_milp.py` L653) ⇒ **zero MILP constraint change** —
   only the cap *value* per solve. `T_a` (flight departure) is already on the offer legs.

2. **Live ledger (was audit-only).** `ReplayState.reconcile` currently records `tendered=0` and the
   whole claim as `committed_untendered` (L827). Make it **live**: split the claim into
   `tendered` (firm, from `tendered_set`) vs `committed_untendered` (reshuffleable). `firm_floor_a` =
   the tendered chargeable-kg on `a`; it (a) protects firm bookings from decay (A.1 floor), (b) is
   the endogenous source. The existing conservation invariant (`free = cap − tendered − committed`,
   raise on over-commit) becomes a real runtime check rather than a recorded number.

3. **"Firm vs reshuffleable" under decay.** Unchanged in *meaning*, sharper in *effect*:
   - **firm (tendered)** — pinned for *all* arms; holds capacity permanently; protected by
     `firm_floor`.
   - **committed-but-untendered** — pinned for **M₁′** (frozen at first placement, `_plan_cycle`),
     **free for M₁** (only tendered pinned). Under a decaying spot cap this is where the value lives:
     M₁′'s premature commitment of D occupies a *now-scarce* (decayed) seat it cannot release; M₁
     releases it back into the (smaller) free pool for the captive express. The decay is what makes
     "the seat M₁′ can't release" actually scarce at the late-arrival clock.

4. **M₁ vs M₁′ pinning × decay — no new pin logic.** The pin sets are exactly as today; the only new
   coupling is that the *cap they're solved against* shrinks with `t`. Feasibility of a pinned set is
   guaranteed by the `firm_floor` (firm cargo never exceeds the floored cap) and by the
   contracted-no-decay rule (contracted pins never face a shrinking cap).

5. **Determinism.** φ is a deterministic function of `(T_a − t)` and the generation-time draw; the
   cadence/solve are unchanged ⇒ byte-identical reruns hold.

---

## E. How this changes the red-team kill-shot

The kill-shot (`05_redteam.md`) measures **per-tier kg per arm on one short lane at τ≈0.7** and
gates on `Δkg ≠ 0`. The temporal mechanism changes its *construction* and *PASS condition*:

- **New required construction** (3 minimal patches on top of the spatial kill-shot):
  1. a **decaying spot cap** `K_a(t) = C0·φ(T_a − t)` fed per cycle (the new mechanism);
  2. **one captive express** on the short lane — very-late book-lead + tight `Δ_E` so only the
     depleting flight is feasible;
  3. **one early deferred** whose **cutoff is after the express's arrival** (the open-book window).
- **PASS** (capacity binds across arms) **iff** at the tight cell: `total_fallback_kg(M₁′) ≠
  total_fallback_kg(M₁)` **OR** the contracted/spot per-tier totals differ — *and* the difference is
  attributable to the reshuffle of the early-deferred (not consolidation re-mix). The worked scenario
  (C) is the pre-registered expected signature: `Δfallback = +1400 kg`, `Δcontracted = −1400 kg`.
- **FAIL** (S45 reproduced) iff every per-tier total is equal between arms even with the curve on —
  which now isolates *which* sub-mechanism is dead: if the express never hits a depleted book, the
  **curve** is too shallow (raise β / shrink `H`); if it does but D is already firm, the **arrival
  spread / cutoff coupling** is wrong (failure mode C.1).
- **New diagnostic to report** alongside Δkg: **realized spot `φ` at each tender** and the
  **committed-but-untendered window width** per (deferred, express) pair — these tell you *why* a
  cell passed or failed, which the spatial kill-shot could not.

The cheaper static pre-check (S1) survives unchanged: confirm soft-contracted effective per-kg
`4.2·max(CW,π)/CW < $5.5` at expected fill before trusting the contracted re-anchor.

---

## F. Build delta & open questions

### F.1 Build delta (gated; no code until approved)

**Generator (`air_generator.py`)**

1. **Booking curve.** New `BookingCurve` params on `ArrivalConfig`/`GenConfig`: `H_h`, `beta`,
   `phi_min`. New `_draw_booking_curve(rng_cap, arcs)` on a new **`cap_decay`** RNG sub-stream (per-
   arc φ params if jittered; or one network curve). Emit `C0_a` (time-zero amplitude) separately from
   the live cap.
2. **Arrival spread.** Replace `tier_coupled_arrival: bool` + `book_lead_coupled_h` with a per-tier
   **bucket-weight matrix** `lead_buckets: dict[Tier, tuple[float,float,float,float]]` + bucket
   ranges. Rewrite the `B` draw in `_gen_arrivals` (L866–873) as categorical-then-uniform, fixed draw
   count (CRN). Keep `tier_coupled_arrival=True` reproducible as a special case (all mass in one
   bucket) for continuity with S44/S45.

**Replay (`replay.py`)**

3. Per-cycle `rates_t` with decayed `spot_cap[a] = max(C0_a·φ_a(T_a−t), firm_floor_a)`; thread `t`
   and flight `T_a` into the cap derivation. Contracted untouched.
4. Make `_reconcile_cycle` **live**: split tendered vs committed-untendered; expose `firm_floor_a`.

**MILP (`air_milp.py`)** — **no constraint change** (cap is a value, `_build_spot_cap` reads
`rates.spot_cap` directly). Add an isolation test that the decayed cap binds.

**Methodology (`arrival_only_replan_methodology.md` §13)** — amend D-A19: spot cap is now a function
of decision clock (`K_a(t) = C0_a·φ_a(T_a−t)`); affirm φ drawn on `cap_decay` stream, contracted does
not decay, firm-floor protects tenders, CRN/independence preserved. Note the arrival stream is the
booking-curve's demand-side counterpart (§1 premise).

### F.2 Open calibration questions (flagged for the user)

1. **Booking-curve shape — `β` and `H`.** No citable air-cargo spot booking curve found (shape
   **INFERRED**, magnitude **MRN**). How back-loaded should depletion be — linear (`β=1`) or
   last-day-cliff (`β=2–3`)? And the horizon `H` (120 h vs 168 h)? This is the load-bearing knob for
   whether the escape closes at the express tail. **Recommend** starting `β=2, H=120, φ_min=0.1` and
   sweeping β as the new capacity axis.
2. **Exogenous vs endogenous balance.** Confirm the **decay-spot-only / contracted-protected** split
   (my recommendation, resolves retroactive infeasibility), vs decaying contracted too (more
   aggressive, but needs an infeasibility-handling story). Should φ jitter per-arc (lumpy decay,
   like α) or be one network curve?
3. **Express late-tail weight + cutoff coupling.** The express very-late bucket weight (0.25
   proposed, **MRN**) *and* its coupling to the deferred cutoff window (C.1) jointly decide whether
   the contention ever fires. How heavy a late-express tail is realistic, and should the kill-shot
   cell *hard-wire* one captive-express / early-deferred pair to guarantee a non-vacuous signal at
   proof scale?

**Secondary assumptions to flag:** `φ_min=0.1`, the four bucket ranges, and the contracted re-anchor
to $4.2/kg (inherited from doc 01, **INFERRED**) all stand as defaults. The decay primarily reshapes
the **spot** tier; if the user wants contracted contention to be temporal too, that is a larger
change (infeasibility handling) and is **out of scope** for this minimal design.
