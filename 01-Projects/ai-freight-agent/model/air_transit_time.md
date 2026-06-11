# Air Transit-Time Methodology — Per-Leg (mean, sd), Single-Draw Realization (Stage 2b)

**Status:** v0.3 — **APPROVED (Session 32, 2026-06-09).** **Gate: G-Method — cleared.** Short
methodology doc, **not** a full LaTeX model — air transit is parametric, not a fitted model.

> **v0.3 amendment (Session 32, user-driven) — air transit is DETERMINISTIC.** Governed by the
> approved `model/arrival_only_replan_methodology.md`. The **only** stochastic process in the air proof
> is the demand-arrival stream; per-leg sampling is "fake" manufactured infeasibility and is removed
> for air. **For air, `s = 0`:** realized transit = the **scheduled block** per leg + ground/dwell at
> the **mean** (D-A3), so the realized arrival `A` equals the deterministic estimate `Â` (§4 walk runs
> with zero dispersion). **Predicate-9 collapses to deterministic deadline feasibility `A ≤ Δ_k`**;
> `z_tier`, `σ̂`, and the chance-flavored admission of §5 are **retired for air**. The entire
> stochastic apparatus below (`σ`, the `s` band, `z_tier`, `σ̂`, the refining `(μ_t, σ_t)` forecast,
> cancellations) is **RETAINED FOR OCEAN** (Stage 4), where transit *is* the value driver — it is the
> *ocean* model, not stale air text. This supersedes the air-stochastic realization language in
> §2/§3/§4/§5/§8 (read those as ocean-applicable / `s=0` for air). Disruption recourse is a **tested
> capability, not a value source** — `arrival_only_replan_methodology.md §6`.

> **v0.2 reframe (Session 29, user-driven).** Two structural corrections to v0.1:
> (1) **No Monte-Carlo per route.** Each shipment is realized **once** — one draw per leg becomes
> the actual; OTP is a **population-over-time** metric (fraction of *all shipments* on-time, tracked
> weekly→monthly→quarterly), not a per-route on-time probability. (2) **Recourse is not a transit
> function.** 2b emits per-leg actuals only; whether a missed connection is *rolled* (baseline) or
> *replanned from current position* (the product, `M₁`) is a policy decision owned by the
> orchestrator (2c). The old "joint MC draws" and "roll lookup" framings are removed.

Downstream of the approved `model/backtest_methodology.md` (v0.3); its §2/§6/§8 are build
constraints on this component. OTP **control** lives in graph generation + the per-shipment penalty
(see `model/air_freight_routing.tex` and `backtest_methodology.md §6–§7`), **not** here and **not**
via chance constraints — this doc supplies the *transit realization* and the *deterministic
reliability scalar* the graph-gen filter consumes.

> **One-line scope.** For **air**, transit is **low-variance and not the value driver** (the value
> driver is demand arrival — `backtest §1`). This model exists so OTP is a *real* number and the
> occasional disruption fires as a break-trigger — not to generate savings. Transit uncertainty,
> a **refining forecast** `(mean_t, sd_t)` that sharpens toward departure, plus **cancellations**
> (blank sailings), is a first-class **ocean** driver, reintroduced in Stage 4.

---

## 0. Notation

| Symbol | Meaning |
|---|---|
| `ℓ` | an air **leg** (one physical flight); scheduled block `b̄_ℓ = arr_utc_h − dep_utc_h` |
| `(μ_ℓ, σ_ℓ)` | per-leg transit **mean / standard deviation**. `μ_ℓ` = the published/estimate central value; `σ_ℓ` is the dispersion (the *only* uncertainty primitive). |
| `a_ℓ` | the **single realized actual** transit of leg `ℓ` on one shipment realization (one draw from its distribution) |
| `(μ_g, σ_g)` | mean/sd of a **ground/dwell** component (cartage, CFS build-up, hub dwell, customs) |
| `s` | the **dispersion multiplier** for the sensitivity band: `σ → s·σ`. `s=0` ⇒ deterministic (recovers today's graph); `s=1` ⇒ calibrated; `s>1` ⇒ stress. (Single global scalar, MVP.) |
| `r` | a candidate end-to-end **route** for a shipment |
| `Â(r)` | **estimate** end-to-end arrival of route `r` (sum of `μ` along the route + connection/dwell means) |
| `σ̂(r)` | a deterministic end-to-end **dispersion proxy** for `r` (≈ `√Σσ²`; a screen, not exact) |
| `z_tier` | per-tier **reliability safety multiplier** used by the graph-gen filter (§5) |
| `A` | the **realized** end-to-end door arrival of a shipment (one running-clock walk, §4) |
| `Δ_k` | the **effective deadline** OTP and the graph-gen filter score against `= min(contractual T_dead, service-tier T_SLA)` — the *same* `Δ_k` the air model's C.10 tardiness penalizes against (`air_freight_routing.tex §hawb-params`). On-time = `A ≤ Δ_k`. |
| `R` | number of seeded **horizon replications** (for confidence intervals on the savings delta) — *not* per-route draws |

---

## 1. What this model is — and is NOT

**Is:** a parametric generator that (a) yields a **single realized per-leg actual** `a_ℓ` per
shipment, walked into one end-to-end realized arrival `A` (§4), and (b) exposes a **deterministic
end-to-end reliability scalar** `(Â(r), σ̂(r))` per candidate route for the graph-gen OTP filter
(§5). Central tendency `μ` equals the **published schedule** (unbiased-estimator assumption for air),
so "plan on the estimate" = plan on the published timetable already in the air graph.

**Is NOT:**
- **NOT a per-route Monte-Carlo OTP estimator.** No `N` draws per route. One draw per leg per
  shipment is the actual; OTP is counted across the shipment **population** over a period (§2).
- **NOT a recourse engine.** Missed-connection handling (roll vs. replan) is the orchestrator's
  policy, not transit (§4).
- **NOT a fitted model of any real network.** A model calibrated to one network gives one point, not
  the curve the proof needs (`product_thesis.md §2`). OpenSky calibration is deferred (and observes
  neither connection nor customs dwell, with poor trans-Pacific coverage). Numeric parameters here
  are placeholders marked `[CAL]`; their justification is the **distribution-calibration note**
  gated *before Stage 3*, not this doc (no unverified statistics — memory
  `feedback_no_unverified_stats`).

**Structural anchor (not a parameter source).** Air delay propagation is **highly localized** —
trees are mostly short chains; a large fraction of sizeable delays propagate to *zero* downstream
flights (AhmadBeygi, Cohn, Guan & Belobaba 2008, *J. Air Transport Mgmt* 14(5)). Defensive
ammunition for (i) treating air transit as low-variance and (ii) a single-draw realization rather
than a heavy propagation engine.

---

## 2. Estimate / actual split (build constraint from `backtest §2`)

- **Estimate (planning input):** the published schedule — `μ_ℓ` and the scheduled ground/dwell
  means already in the graph. Policies (`H₀/M₀/M₁`) plan on these. **No air-MILP change is required.**
  *(The published ETA itself drifts over time — small for air, large/cancellable for ocean. Air MVP
  treats `(μ_ℓ, σ_ℓ)` as **static through time**; the refining `(μ_t, σ_t)` forecast and
  cancellations are an ocean/Stage-4 generalization, §3.)*
- **Actual (scoring):** **one draw per leg** `a_ℓ ~ dist(μ_ℓ, s·σ_ℓ)`, taken when the leg completes
  in sim time, walked into one realized end-to-end arrival `A` (§4). Each shipment is realized
  **once**.
- **OTP is a population-over-time metric.** `OTP = (# shipments with A ≤ Δ_k) / (# shipments)`
  over the period (weekly→monthly→quarterly). The "many samples" are the **many shipments**, each
  drawn once — not replays of one shipment. Per-shipment on-time is **binary**.
- **CRN + replications.** The per-leg draw seed is shared across policy arms (so `M₀` vs `M₁` see the
  *same* realized actuals for the same shipment — the delta is policy, not noise). Confidence
  intervals on the savings delta come from `R` seeded **horizon replications**, each realizing its
  whole population once.

---

## 3. The transit model — per-leg (mean, sd)

The uncertainty primitive is a per-component **(mean, sd)**; `σ` *is* the dispersion. A draw is

```
a_ℓ ~ Normal(μ_ℓ, (s·σ_ℓ)²),  floored so arrival is not implausibly early   [CAL] σ_ℓ
```

and likewise for ground/dwell/customs components `(μ_g, s·σ_g)`. At `s=0` every realized value
collapses to its scheduled value (deterministic-recovery regression anchor).

- **Air MVP = static (μ, σ) through time.** Air ETA refinement is small, so we do not model the
  `(μ_t, σ_t)` forecast that sharpens toward departure. **No explicit disruption mixture for air** —
  rare lateness is folded into `σ` (kept simple per minimal-design default).
- **Ocean/Stage-4 generalization (documented, not built):** (i) a **refining forecast** `(μ_t, σ_t)`
  with `σ_t` shrinking as departure nears (early = uncertain, late = sharp), so replanning has an
  ETA signal to react to; (ii) **discrete cancellation events** (blank sailings) — a point mass that
  `(μ, σ)` cannot express; (iii) likely **abandoning carrier-published ETA** as the estimate (ocean
  schedules are unreliable). None of this is air MVP.
- **Customs dwell** is per-HAWB (own entry), right-skewed in reality; modeled as `(μ_cust, σ_cust)`
  with `μ_cust = Hawb.customs_dwell_h`. **Ground jitter** (cartage, CFS, hub dwell) is low-variance.

---

## 4. The single realized timeline (one draw per leg)

One realization of a shipment on route `r` = a single sequential running-clock walk; **one** draw
per component, so every component sees the realized upstream time (leg correlation + the
connection-miss cascade survive with a single realization):

```
clock ← cargo_ready_time(k)
for each component c in route r (in time order):
    if c is GROUND/DWELL/CUSTOMS:
        clock ← clock + draw(μ_c, s·σ_c)
    elif c is AIR leg ℓ:
        dep   ← max(dep_utc_h(ℓ), clock)           # cannot depart before cargo is present
        clock ← dep + draw(μ_ℓ, s·σ_ℓ)             # the single realized block a_ℓ
        emit per-leg realized actual = clock
A ← clock                                          # realized end-to-end door arrival
```

- **Emits per-leg realized actuals** (not just an end-to-end boolean) — required so break-triggers
  fire at leg granularity (`backtest §8`).
- **2b does NOT decide recourse.** Whether the running clock implies a **missed connection** is a
  check the **orchestrator** owns (it knows the booked itinerary: `clock + MCT_h > dep(outbound)`),
  and the recourse is **policy-specific**:
  - `H₀ / M₀` — **break-only recourse:** roll the shipment to the next available flight.
  - `M₁` — **replan from current position** with all available options (re-optimize the affected
    shipment from where it physically is). Better disruption recourse is part of air L2.
  2b's job ends at emitting `a_ℓ` and `A`; the connection-made check and the roll/replan live in 2c.
- **No Monte-Carlo loop.** This is one walk per shipment per realization.

---

## 5. OTP control is at GRAPH-GEN + the penalty — NOT a chance constraint

The proof's planner stays on published ETAs (§2); OTP is *controlled* by two external levers
(detailed in `air_freight_routing.tex` / `backtest §6–§7`), and this model only **feeds the first**:

1. **Graph-gen tier-reliability filter (primary OTP control).** When building a shipment's subgraph
   `G(N_s, A_s)`, admit a candidate route `r` only if its **deterministic** reliability margin meets
   the tier floor:

   > admit `r`  ⇔  `Â(r) + z_tier · σ̂(r) ≤ Δ_k`

   `Â(r)`, `σ̂(r)` are the deterministic end-to-end estimate + dispersion proxy this model exposes
   (§6). **No Monte-Carlo, no per-arc quantile propagation in the MILP** (sum of per-arc quantiles ≠
   quantile of the sum — the convolution problem; the model's `item:tt-quantile-binding` is *not*
   built). Express tiers admit only reliable/fast routes; deferred tiers admit the full cheap/slow
   set. The threshold `z_tier` is tuned **closed-loop against measured population OTP**.
   - *Candidate routes* `r` are enumerated by graph-gen over the per-shipment subgraph `A_k` (the
     air per-shipment path set is small); an arc is retained iff it lies on ≥1 tier-admissible route.
   - *Honest caveat 1 — it's a candidate-set prune, not an end-to-end guarantee.* Predicate 9 is an
     **arc** filter, so the MILP can still **recombine** surviving arcs into a path that was never
     itself tier-admissible (arc A kept for fast-route r1, arc B for fast-route r2, but A→B = a slow
     r3). The screen is necessary, not sufficient. The selected path's reliability is backstopped
     **ex-post** by the C.10 tardiness penalty (`W_k`), the closed-loop `z_tier`, and the fallback —
     not guaranteed by the filter. Exact per-path admission is the deferred path-based/column-gen
     migration (`air_freight_routing.tex item:tt-quantile-binding`).
   - *Honest caveat 2 — `σ̂(r) ≈ √Σσ²` is not uniformly conservative.* It errs conservative for
     slack-rich routes, but **under-states** dispersion for **tight-connection** routes (the cascade
     is super-additive — a small upstream slip blows a connection → a discrete downstream jump), so
     it can **false-admit** a fragile route into an express tier. The closed-loop `z_tier` (tuned to
     realized population OTP) exists precisely to cover this; tight-connection routes are exactly
     where the deferred path-based quantile binding would eventually be warranted.

2. **Per-shipment penalty `W_k` (prioritization control input).** Default = a fixed pre-registered
   per-tier ratio; **exposed as an external control signal** to prioritize specific shipments under
   capacity contention (e.g. raise the weight of a customer who is behind on month-to-date OTP so a
   scarce reliable slot breaks toward them). It shapes *who wins scarce capacity*, not just lateness
   magnitude. **Frozen during the L2 savings proof** (fixed weights, no mid-run rebalancing) so the
   headline number is not confounded by tuning the metric we report; demonstrated as a separate
   capability. (Owned by `air_freight_routing.tex` `W_k = w_sp(k)·μ_k`; this doc only notes the
   dependency.)

**Chance constraints / in-MILP quantile binding are NOT in MVP** — explicitly deferred (not built),
reserved only for contractual per-shipment SLAs, thin-tail lanes, and singleton cargo (VAL/HUM/AVI),
where portfolio averaging fails.

---

## 6. Output schema — the FROZEN interface (freeze before 3a)

`backtest §8` makes 3a depend on this; frozen before Stage 3a per the input-seam rule. Two surfaces:

**(A) Realization (scoring side):** `sample_route(arcs, deadline_h, rng, config, ready_h) ->
RouteRealization` (as built, `src/components/air_transit_time.py`):

| Field | Type | Meaning |
|---|---|---|
| `leg_etas` | `list[LegEta]` | per air leg, in route order: `(flight_id, scheduled_dep_h, realized_dep_h, realized_arrival_h)` |
| `end_to_end_arrival_h` | `float` (UTC h) | `A` — realized door arrival (one draw) |
| `on_time` | `bool` | `A ≤ Δ_k` (binary; the population OTP numerator) |

`leg_etas` carries `scheduled_dep_h` vs `realized_dep_h` per leg precisely so the orchestrator can do
the **connection-made check** (`realized_dep_h > scheduled_dep_h` ⇒ slip) — a bare `list[float]` of
arrivals could not. The **connection-made check and recourse are NOT done here** — the orchestrator
derives them from `leg_etas` + the booked itinerary (§4). Routes are passed **time-ordered**
(`air_graph.order_route`), a hard precondition of the running-clock walk.

**(B) Reliability scalar (graph-gen side):** `route_reliability(route) -> (Â, σ̂)` — deterministic,
no draws; consumed by the §5 admission filter. Cacheable (a function of schedule + `(μ, σ)`, not of
the arriving demand), so the rolling-horizon loop reuses it across replan steps without re-sampling.

The **estimate** side reuses existing graph fields verbatim (no new planning-input schema).
`(μ_ℓ, σ_ℓ)`, the dispersion multiplier `s`, and `[CAL]` parameters live in one config object
(mirroring 2a's `GenConfig`), so the `(κ, λ)` sweep can hold transit fixed while varying demand, and
stress transit (raise `s`) independently.

---

## 7. Calibration & provenance (deferred to the pre-Stage-3 note)

- **Forms (this doc, fixed):** per-leg `(μ, σ)`, single-draw realization, static-through-time for
  air, deterministic reliability proxy, single global `s`.
- **Magnitudes (`[CAL]`, deferred):** `σ_ℓ, σ_cust, σ_ground, z_tier, MCT_h`. Justified in the
  **distribution-calibration note** (plan §2a/§2b) vs. public signal, provenance tagged
  real-network vs. synthetic per CLAUDE.md. No unverified statistics enter this doc.
- **OpenSky deferred** to the design-partner validation phase, off the critical path.

---

## 8. Decisions (proposed — confirm at gate)

- **D-T1 — realization:** **one draw per leg = the actual**, single end-to-end running-clock walk;
  OTP is the **population-over-time** on-time fraction; CIs via `R` horizon replications. *No
  per-route Monte-Carlo.*
- **D-T2 — recourse out of scope:** 2b emits per-leg actuals; the **orchestrator** does the
  connection-made check + recourse (`H₀/M₀` roll, `M₁` replan-from-current).
- **D-T3 — dispersion:** primitive is per-leg `σ`; the band sweeps a **single global multiplier `s`**
  (`s=0` recovers the deterministic graph). Per-component split deferred (split only if a stress
  variant needs it).
- **D-T4 — OTP control is external:** graph-gen tier-reliability filter (deterministic, §5) +
  per-shipment penalty control input. **No chance constraints / no in-MILP quantile binding in MVP.**
- **D-T5 — air static, ocean refines:** air `(μ, σ)` static through time, no disruption mixture;
  refining `(μ_t, σ_t)` + cancellations are an ocean/Stage-4 generalization (documented, not built).

---

## 9. Definition of Done (2b component gate)

- [ ] `sample_route(...)` returns schema (A); per-leg `leg_etas` present (scheduled + realized dep).
- [ ] Routes consumed **time-ordered** via `air_graph.order_route` (the running-clock precondition).
- [ ] Single-walk realization reproduces a late-leg → tight-connection slip on a hand-built route at
      a fixed seed (the running clock carries the cascade), with **recourse left to the caller**.
- [ ] `s=0` ⇒ every realized value equals the scheduled value (deterministic-recovery regression).
- [ ] Monotonicity: higher `s` ⇒ (weakly) wider arrival spread + lower population OTP on a fixed book.
- [ ] `route_reliability(route)` deterministic, cacheable, monotone in `z_tier` margin; returns
      `(Â, σ̂)` per schema (B).
- [ ] **Recombination limitation encoded as a test** (§5 caveat 1): a hand-built instance where two
      individually tier-admissible arcs recombine into a non-admissible path, asserting the
      documented ex-post backstop (penalty/fallback dominates), not a false "guaranteed" claim.
- [ ] **Output schema frozen** (§6) before 3a begins.
- [ ] **No Monte-Carlo loop**, **no recourse logic**, **no chance constraint** in this component.
- [ ] Pure sampler/estimator — real HiGHS never invoked here; isolation tests are distribution/seed
      assertions only, no timing assertions (CLAUDE.md).
