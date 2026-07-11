# L2 Decomposition — what `L2 = C(M₁′) − C(M₁)` is physically made of (S45)

**Status:** EMPIRICAL MEASUREMENT (read-only probe). Decides whether the increasing-block spot
redesign (`air_pricing_capacity_redesign_s45.md`) will revive `∂L2/∂κ > 0` and zero the loose
corner, or whether the existing L2 is a capacity-independent consolidation artifact the curve
cannot touch.

**Method.** For each cell, ran the current code for arms **M1p** and **M1** through `run_replay`,
took each arm's final committed plan (the all-pinned `solve` the scorer bills — identical to
`res.cycles[-1].total_cost`), and classified every committed air arc by rate family:
spot = arc ∈ `rates.spot_cap` (coload / flat / min_flat_breaks); contracted = `PER_ULD_PIVOT`
BSA arc; fallback = `ArcType.FALLBACK`. Per-HAWB chargeable weight `cw_k = max(w_k, v_k·167)`
attributed to each arc on the route. `n_hawbs=12, days=4, tier_coupled_arrival=True`,
`PYTHONHASHSEED=0`. Classification verified against actual arc ids and `RateCatalog` contents.

---

## Headline finding (all three cells)

**Total spot kg is byte-identical between M1p and M1. Contracted = 0. Fallback = 0. ULD
positions used = 0.** The entire book rides spot in every cell; the contracted block is *priced
and present* (total_N = 8 / 4 / 16 ULD positions across cells) but **never used** because spot is
flat-priced and effectively unbounded (~150k kg available vs a ~6–7.5k kg book — exactly the
48× over-supply critique-20 A1 names). So:

- **L2 is NOT (i) M1 routing less total spot volume** — total spot kg delta = **0.0** in all cells.
- **L2 is NOT (ii) M1 avoiding fallback** — neither arm ever touches fallback.
- **L2 IS (iii)/(iv): how the *same* total spot volume is consolidated and distributed** — MAWB
  count, MAWB density-mixing into cheaper `min_flat_breaks` weight-breaks, and per-(lane,day)
  placement.

---

## Per-cell tables

### Cell A — seed 1, κ=1, L2 = $1055.7 (the big one)

| metric | M1p | M1 | M1p − M1 |
|---|---:|---:|---:|
| spot kg | 7452.6 | 7452.6 | **0.0** |
| contracted kg | 0 | 0 | 0 |
| fallback kg | 0 | 0 | 0 |
| active MAWBs | 7 | 5 | **+2** |
| ULD positions | 0 | 0 | 0 |

Per-lane TOTAL spot kg (week granularity): **TPE→LAX 3067.5 vs 3067.5 (Δ0), TPE→ORD 4385.1 vs
4385.1 (Δ0)** — equal per lane.
Per-(lane,day): TPE→LAX day1 2975.6 vs 2667.6, day4 91.9 vs 399.9 (M1 spreads ±308 kg across days,
nets to 0 on the lane).

**Attribution.** L2 = $1055.7 is entirely consolidation. M1 packs the same cargo into **5 MAWBs
vs M1p's 7** → −$100 in `mawb_fix_cost` (2×$50). The remaining **~$956** is `min_flat_breaks`
density-mixing: M1 builds fatter MAWBs (e.g. a 2576.8 kg consolidation vs M1p splitting the same
mass into 426+576+652+1364 kg), reaching cheaper IATA next-break-down rates. **Same lane, same
total kg, fewer & fatter consolidations.** 100% capacity-independent consolidation channel.

### Cell B — seed 2, κ=2, L2 = $257.7

| metric | M1p | M1 | M1p − M1 |
|---|---:|---:|---:|
| spot kg | 6198.4 | 6198.4 | **0.0** |
| contracted / fallback kg | 0 | 0 | 0 |
| active MAWBs | 8 | 8 | **0** |

Per-lane TOTAL: HKG→LAX 424 vs 369 (**+55**), TPE→LAX 2833 vs 2888 (**−55**), TPE→ORD equal.
A single ~55 kg HAWB shifts corridor; total spot book identical.

**Attribution.** Equal MAWB count, equal total volume. L2 = $257.7 is pure intra-MAWB
re-mixing of the chargeable-weight distribution across the 8 consolidations (which CW lands on
which MAWB → which weight-break rate), plus one small cross-lane HAWB move. No fixed-cost delta,
no volume delta, no fallback. Capacity-independent.

### Cell C — seed 0, κ=0.5, L2 = $318.9 (loose cell)

| metric | M1p | M1 | M1p − M1 |
|---|---:|---:|---:|
| spot kg | 6230.0 | 6230.0 | **0.0** |
| contracted / fallback kg | 0 | 0 | 0 |
| active MAWBs | 9 | 9 | **0** |

Per-lane TOTAL: PVG→LAX 0 vs 446 (**−446**), TPE→LAX 2043.1 vs 1597.1 (**+446**), others equal.
M1 routes one 446 kg HAWB via a *different origin gateway* (PVG vs TPE) to LAX. Total spot book
identical.

**Attribution.** Equal MAWBs, equal volume. L2 = $318.9 = a cross-gateway re-route of one HAWB
(446 kg TPE→LAX → PVG→LAX) plus intra-MAWB CW re-mixing on the remaining consolidations. Even at
the *loose* corner (κ=0.5) L2 is a clean $318.9 of consolidation/gateway reshuffle with zero
spill-volume or fallback component — which is exactly why the loose corner does NOT zero today.

---

## Does the increasing-block curve amplify any of this?

The redesign's premium is a convex function of **total kg per lane pool**. It amplifies L2 **only
if M1 and M1p put different total kg into the same pool.** Applying the proposed §10.1 ladder
`[(5000,1.00),(3000,1.20),(2000,1.44),(1250,1.73)]×$5.5` to the measured spill (premium-above-base,
overflow→2.5× fallback):

| pooling | ceiling | A (L2=1056) | B (L2=258) | C (L2=319) |
|---|---:|---:|---:|---:|
| **per-lane-WEEK** (spec §10.1) | 11250 kg | **+$0.0** | **+$0.0** | **+$0.0** |
| **per-lane-DAY** (OPEN-1 rec, ÷7) | 1607 kg | +$2541 | **−$454** | +$2136 |

- **Per-lane-WEEK: the curve adds exactly $0 of L2 amplification on all three cells.** Each lane's
  weekly spill (≤7.5k kg) sits inside block B0 (≤5000 kg, rate = base) or barely into B1, and the
  per-lane TOTALS are byte-identical between arms — so both arms pay the identical premium and the
  delta is zero. The week-granularity pool is too coarse to see the day-level reshuffle that
  generates today's L2.
- **Per-lane-DAY: the curve does react, but erratically and with the wrong sign in B.** The 1607 kg
  daily ceiling is *below* typical daily lane spill (2600–4400 kg), so cargo is shoved deep into
  upper blocks and fallback, and the day-level reshuffle (cell A's ±308 kg across days) now changes
  the premium. But the sign is inconsistent: **+$2541 (A), −$454 (B), +$2136 (C)** — in B the curve
  would make M1 *costlier* than M1p, inverting the C(M1p) ≥ C(M1) chain.

Caveat on the day-level numbers: this is a *static* re-pricing of the current routing. Under the
real redesign the MILP re-optimizes against the new cost, so M1/M1p would route differently and the
−$454 would not literally persist (the solver would avoid it). But the static probe is the right
diagnostic for *where the existing L2 lives*, and it shows the existing L2's physical carrier
(intra-MAWB CW re-mix + cross-day/gateway HAWB moves at **equal per-lane-week total volume**) is
**invisible to a per-lane-week curve and only accidentally visible to an over-tight per-lane-day
curve.**

---

## VERDICT

**For every cell, the L2-relevant split is: spill-volume / fallback difference = $0 (0%);
consolidation-reshuffle-at-equal-spill = 100% of L2.**

- Cell A ($1056): $0 volume, $0 fallback; ~$100 MAWB-count + ~$956 density-mix consolidation.
- Cell B ($258): $0 volume, $0 fallback; 100% intra-MAWB CW re-mix.
- Cell C ($319): $0 volume, $0 fallback; 100% cross-gateway + CW re-mix — *at the loose corner*.

**Bottom line — this is a REDESIGN-BLOCKING finding for the stated goal.** The existing L2 is a
**capacity-independent consolidation/CW-redistribution artifact** at byte-identical total spot
volume. The increasing-block per-lane curve, as specified (per-lane-week, OPEN-1's recommended
÷7-per-lane-day still keys the *premium* on total lane kg), **prices total kg per lane pool, which
is exactly the quantity M1 and M1p hold equal.** At week granularity it amplifies L2 by **$0**. The
only way it moves L2 here is to crush the daily ceiling so far below daily spill that the
*within-lane day-shuffle* starts crossing block boundaries — and that produces erratic,
wrong-signed deltas, not a clean monotone `∂L2/∂κ > 0`.

The redesign's own thesis (§2: "M₁ re-packs the cheap finite block; M₁′ cannot") presumes the two
arms *differ in which cargo occupies the scarce cheap capacity at different total volumes.* In this
backtest they don't: they route identical total spot kg and differ only in **consolidation
structure**, which the lane block curve — a function of total lane kg, not of who-sits-where — is
structurally blind to. **Making spot finite will not, by itself, revive κ-sensitivity of L2 as long
as L2's physical carrier is consolidation reshuffle at equal spill volume.**

**What would actually be needed** (not in scope here, flagged for design): for the block curve to
bite, M1 and M1p must be *forced to route different total spot volumes* — which requires the
*contracted* cheap capacity to actually be used and binding (so reshuffling moves cargo between
contracted and spot, changing total spot kg), or the spot ceiling to bind so hard that arms differ
in total spilled-to-fallback kg. Today contracted is never used (spot is cheaper and unbounded), so
there is no contracted↔spot reshuffle margin at all. The redesign must first make **contracted
capacity attractive-and-scarce enough to be used and fought over** (Mechanism B's κ-tied ceiling
*plus* re-anchoring contracted below spot so it's worth filling), or L2 will remain the
consolidation artifact measured here.
