# S51 — Booking-Curve Capacity Decay: Research + Grounded Model

**Status: ADOPTED (S51).** Grounds the §14.2 capacity-decay booking curve in external data, replacing the
S49 gentle-linear form that had been reverse-engineered to hit a (since-retired) fallback target.

**Method:** three independent research agents were tasked with the *same* neutral brief — research how
bookable air-cargo spot capacity depletes toward the cargo cutoff, and propose a decay model from data —
**without seeing the incumbent model** (to avoid anchoring). Two returned complete reports (A, C); the
third died on an API error. A and C converged on structure and diverged only on the central cutoff fraction;
the user resolved that divergence toward the headhaul-specific low value.

Sourcing tags: **SOURCED** (public URL), **INFERRED** (derived from a sourced anchor), **ASSUMPTION**
(no source — flagged).

---

## 1. The quantity

For a flight, the *bookable spot / space-available capacity* is not constant — it fills as departure
approaches. Define the availability curve `A(τ)` = fraction of a flight's sellable spot capacity still
bookable with `τ` time left before the **cargo cutoff** (cutoff ≈ 3–6h before STD = `τ=0`). `A→1` far out
(empty), falling to **`A_cut`** at cutoff. `A_cut` — the residual free space when cargo closes — is the
load-bearing quantity: it sets how much capacity a late-committing shipment actually finds.

---

## 2. Convergent findings (agents A + C, independent)

| Dimension | Finding | Tag |
|---|---|---|
| **Curve shape** | Convex / back-loaded; **both reject linear and logistic**. Exponential `A(τ)=A_cut+(1−A_cut)(1−e^{−λτ})` | SOURCED (shape) |
| **Depletion timing** | Bulk of fill in the final ~7–14 days; `λ ≈ 0.08–0.20`/day | SOURCED |
| **Load-factor anchor** | Use **dynamic (volume) LF ~57–65%** at departure, not weight CLF ~46% ("cube out before weigh out") | SOURCED |
| **Regime** | `A_cut → ~0` on tight/peak headhaul; 0.4–0.65 on soft/backhaul | SOURCED (direction), INFERRED (values) |
| **Heterogeneity** | Per-flight `A_cut` is **right-skewed** (Beta, mass near 0), not symmetric | INFERRED |
| **Deck** | Freighter `A_cut` (~0.20) **< belly** (~0.38): freighters run hotter, cube out | SOURCED (LF split) |

**Divergence — central `A_cut`:** Agent A **≈0.12–0.18** (explicitly transpacific-conditioned); Agent C
**≈0.30** (global blend across all directions, which *includes* slack backhaul). Both INFERRED — no public
per-flight residual-at-cutoff dataset exists. The gap is a direction-attribution choice, not a contradiction.

---

## 3. Key sources (consolidated evidence table)

| Claim | Value | Tag | Source |
|---|---|---|---|
| Capacity booked 2 weeks before departure | **<40%** | SOURCED | McKinsey, *Ahead of the curve* (2023) — mckinsey.com/industries/logistics/our-insights/ahead-of-the-curve-getting-cargo-revenue-management-right-as-the-cycle-turns |
| Booking is a last-minute process; yield volatility highest in final week | qualitative | SOURCED | McKinsey (same) |
| Weight-based CLF 2024–25 | ~45.9–47.5% | SOURCED | IATA Air Cargo Market Analysis — iata.org/en/pressroom/2025-releases/2025-01-29-02/ |
| International **freighter** CLF | **63.0–63.7%** | SOURCED | IATA, Dec-2025 analysis |
| **Belly** CLF | **41.8%** | SOURCED | IATA, Dec-2025 analysis |
| **Dynamic (volume+weight) load factor** | **~57% Jan-25 / ~62–67% Nov–Dec** | SOURCED | CLIVE / Xeneta — xeneta.com/news (dynamic-loadfactor methodology + monthly notes); aircargonews.net CLIVE coverage |
| Dynamic LF ~35% above weight CLF; flights "cube out before they weigh out" (volume binds) | +35% | SOURCED | CLIVE / Xeneta |
| Cargo cutoff (general cargo) | ~3h physical / 4h docs before STD | SOURCED | cargoenter.com/cargo-tools/air-cargo-cutoffs |
| Spot bookable window | ~10–14 days ahead | SOURCED | American Airlines Cargo / Delta Cargo product pages |
| Allotment vs spot capacity split | spot ~46% / allotment ~54% | SOURCED (vendor) | Future Market Insights — air-cargo-capacity-and-allotment-marketplace |
| Allotment bid 6–12 mo ahead, firmed days before; washout returns unused space to spot near cutoff | qualitative | SOURCED | Amaruchkul & Topaloglu, *Cargo Capacity Management with Allotments and Spot Market Demand*, Oper. Res. — people.orie.cornell.edu/huseyin/publications/allotments.pdf |
| Long-lead (7+ d) bookings 2–3× more likely to cancel than last-minute | 2–3× | SOURCED (blog) | WebCargo — webcargo.co/blog/holiday-air-freight-booking-strategy |
| RM state variable is (remaining capacity, time-to-departure) | methodological | SOURCED | Eren & Li, *Data-Driven Revenue Management for Air Cargo*, arXiv:2405.11000 |
| **Transpacific Asia→US = the tight HEADHAUL**; capacity inflexible, structurally tight | qualitative | SOURCED | IATA 2024 year-end; Air Cargo Week |
| **`A_cut` (headhaul) central** | **~0.15** (band 0.12–0.30) | INFERRED | dynamic LF residual − unusable trim, headhaul-conditioned (agent A); backhaul excluded (we model none) |
| **`λ`** | **~0.10–0.16**/day | INFERRED | solved so <40% booked at τ=14d + same-week dominance |

---

## 4. Adopted model (LEAN)

```
  φ_a(τ) = A_cut,a + (1 − A_cut,a) · (1 − e^(−λ_a·τ)),   τ = (cutoff_a − t) in days, clamped ≥ 0
```

- **`A_cut,a ~ Beta`, right-skewed, deck-differentiated:** freighter `Beta(1.3, 8.7)` (mean **0.13**) **<**
  belly `Beta(1.8, 6.2)` (mean **0.225**); freighter-heavy fleet ⇒ network central **≈ 0.15**.
- **`λ_a ~ U(0.10, 0.16)`/day.**
- Anchored at the **cargo cutoff** (`τ=0`), not STD.
- Drawn once per flight on the `cap_decay` RNG sub-stream (CRN: never reads demand; varying any
  demand/supply knob leaves the curve byte-identical).
- The free-pool / firm-floor / hard-BSA-untouched machinery (§14.2) is **unchanged** — only the curve's
  *shape and parameters* change.

### Why headhaul → low `A_cut` (the user's resolution of the A-vs-C divergence)

Agent A's ~0.15 was already transpacific-conditioned; agent C's ~0.30 was a global blend that averages in
the slack **backhaul** (US→Asia). **Our network is 100% headhaul** (all flights Asia→US; no return leg is
modelled), so the backhaul slack never applies — `A_cut` belongs at the tight end (~0.15), and the deck
split places freighters below / belly above it.

---

## 5. Consequences + open items

- **Higher, regime-dependent fallback than the S49 fit** — the honest grounded result. The §14.5 "2–8%
  normal" fallback target was a phantom (fit-to-target, never sourced) and is **retired**; report the
  fallback the grounded model produces, per τ regime.
- **`A_cut` central is INFERRED** (±~0.07) — no public per-flight residual dataset exists. This is the
  irreducible uncertainty; revisit if flight-level load-factor-at-cutoff data is ever licensed.
- **Deferred to v2 (FULL):** regime-couple `A_cut` to lane tightness `τ_ij` (agent C's `A_cut(κ)`), an
  explicit allotment-washout bump near cutoff, and a Beta-CDF empirical curve if flight-level data is
  obtained. LEAN keeps `A_cut` tightness-independent (the `τ` dial already supplies nominal scarcity).
- **Third agent (B) died** on an API error; A and C are the two independent reads. A third would land in
  the 0.15–0.30 band without resolving the inherent uncertainty.
