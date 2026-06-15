# Flexibility Model (2-FLEX) — Service Tiers, Slack, the Reshuffle Denominator

**Status:** v0.3 — **APPROVED (Session 29, 2026-06-06).** **Gate: G-Method — cleared.** Decisions
D-F1…D-F8 resolved + 2 critique rounds folded. (Build deferred pending a proof-wide hardening review:
calibration audit / interface-seam audit / backtest methodology red-team.) Short methodology doc.

Upstream of the orchestrator (2c) and **feeds ALL policy arms** (`H₀/M₀/M₁`), not just replanning.
It is the **connective tissue** that ties the
service-tier taxonomy to the three places the rest of the system already references a tier:
- **2a generator** assigns each HAWB a tier and draws its deadline/slack from the tier.
- **Predicate 9** (`air_freight_routing.tex`, the OTP-control filter) reads the tier's `z_tier`.
- **`W_k = w_sp(k)·μ_k`** (C.10 tardiness) reads the tier's `w_sp` weight.

> **Why this is load-bearing.** The thesis claim is savings **on the flexible portion of the book**
> (`backtest_methodology.md §6`, the per-flexible-kg denominator `cw_flex`). If "flexible" is a free
> parameter we can inflate, the headline number is unfalsifiable. 2-FLEX pins flexibility to a
> **defensible external anchor** (standard air service tiers) and makes **sandbagged flexibility**
> the band's primary sensitivity (`backtest §7`). No fabricated mechanisms / no unverified stats:
> the tier *structure* is standard industry practice; specific slack-hours and OTP targets are
> `[CAL]` placeholders for the distribution-calibration note gated before Stage 3.

---

## 0. Notation

| Symbol | Meaning |
|---|---|
| `tier(k)` | service tier of HAWB `k` ∈ {`EXPRESS`, `STANDARD`, `DEFERRED`} |
| `ready_k` | cargo-ready time (`Hawb.ready_early_h`) |
| `Δ_k` | effective deadline `= min(T_dead, T_SLA)` — the tier SLA deadline, OTP/predicate-9 bound |
| `A_k^min` | earliest feasible **estimate** arrival over `k`'s tier-admissible routes (from 2b `route_reliability` Â) |
| `min_transit_k` | `A_k^min − ready_k` — fastest feasible end-to-end transit |
| `sla_offset_h(tier)` | per-tier promise buffer on top of the fastest transit; sets `T_SLA` (§3) |
| `T_dead, T_SLA` | optional per-HAWB shipper-contractual deadline / tier SLA deadline; `Δ_k = min(T_dead, T_SLA)` |
| `slack_k` | `Δ_k − A_k^min` — headroom to ride a later/cheaper option. `= sla_offset` when no `T_dead`; **smaller and HAWB-specific when a shipper `T_dead` bites** (D-F6/M-2b) |
| `θ_flex` | minimum *meaningful* slack separation for a real alternative, ≈ one inter-departure gap |
| `flex_k` | boolean: `k` is **flexible** (genuinely reshufflable) — §2 |
| `cw_flex` | `Σ_{k: flex_k} cw_k` — chargeable weight of the flexible book (the per-flexible-kg denominator) |
| `z_tier` | per-tier reliability safety multiplier (predicate 9) |
| `w_sp(tier)` | per-tier tardiness weight (into `W_k`) |

---

## 1. Service-tier taxonomy (the external anchor)

Three tiers — the standard air-cargo express/standard/deferred structure (faster + higher-priority +
higher-OTP + pricier as you go up; more offloadable + more slack as you go down). Trademarked carrier
product names are deliberately avoided.

| Tier | Character | OTP target (portfolio) | Slack | `z_tier` | `w_sp` |
|---|---|---|---|---|---|
| **EXPRESS** | fast, guaranteed/priority space, premium | high `[CAL]` (~90%) | tight | high (strict reliability admission) | high (defend the date) |
| **STANDARD** | general cargo, normal handling | mid `[CAL]` (~80%) | moderate | mid | mid |
| **DEFERRED** | economy, offloadable, cheapest | lower `[CAL]` (~70%) | generous | low (admits the full cheap/slow set) | low |

- OTP targets are **portfolio-over-time** outcomes hit by *control* (predicate-9 `z_tier` + `w_sp`),
  **not** per-shipment chance constraints (`backtest §6`; memory `project_otp_control_reframe`).
- The 90/80/70 figures are the working anchor (user-stated); exact values + slack-hours are `[CAL]`,
  justified in the distribution-calibration note, not here.
- Ordering invariants (asserted, not the magnitudes): `z_EXPRESS > z_STANDARD > z_DEFERRED`,
  `w_EXPRESS > w_STANDARD > w_DEFERRED`, `slack_EXPRESS < slack_STANDARD < slack_DEFERRED` in
  expectation. The *ordering* is the defensible claim; the *levels* are calibrated.
- **Tier mix (D-F5): a config**, default **20 / 40 / 40** (EXPRESS / STANDARD / DEFERRED). 2a draws
  `tier(k)` from this mix; sweepable for the band (a peak regime can skew express-heavy).
- **`Δ_k` is a pre-committed tier×lane SLA (D-F6 v2, supersedes the original D-F6):**
  `Δ_k = ready_k + base_transit_h(lane) + sla_offset_h(tier)`, where `base_transit_h(lane)` is the
  lane's **pre-committed promised end-to-end transit** (a capability estimate, shared by all
  shipments on the lane, set *before* routing — e.g. TPE→LAX base 96h + `{0/24/48}` → 96/120/144h
  for X/S/D). The tier *sets the promise premium*; the lane sets the base. **Not** the shipment's own
  `A_k^min` (the retired v1 `Δ_k = A_k^min + sla_offset`, which coupled the promise to the routing
  graph — circular, and under build-time geo selection it pushed `Δ_k` past `T^abs`). Rationale +
  worked numbers: `precommitted_sla_deadline_proposal.md` (APPROVED S37).

---

## 2. What "flexible" means (operational, derived, frozen)

**Two arrival statistics on two different candidate sets — the split that breaks a circularity (§2.1):**
- `A_k^min` = earliest feasible **estimate** arrival over `k`'s **pre-predicate-9** routes
  (predicates 1–8, tier-independent: lane / cargo / embargo / lithium / service-product / time-window /
  destination-reachability / ULD-fit), from 2b `route_reliability` Â. Sets `Δ_k`.
- the **tier-admissible** set = the **post-predicate-9** routes (after the `z_tier` reliability cut);
  `flex_k` is derived on *this* set.

`slack_k = Δ_k − A_k^min`. Slack alone is not flexibility — there must be an actual alternative to
reshuffle *to*:

> **`flex_k` = True iff `k` has ≥2 tier-admissible (post-9) on-time options that are (i) separated by
> ≥ `θ_flex`** (`≈ one inter-departure gap`, so a trivially-close second option doesn't count) **and
> (ii) non-dominated** — a 2nd option that is both *later and pricier* than the fastest is no real
> reshuffle target and does not qualify (M-3: without this filter the test over-counts and stops being
> conservative). This is the operative predicate. `θ_flex` is `[CAL]` — median inter-departure on the
> lane for irregular schedules.

> **Out of scope of `flex_k` (deliberate):** *cost-only* flexibility — two options on the *same*
> flight at different rates/MAWBs (zero arrival separation) — is the most common real reshuffle but is
> **not** counted in `flex_k`/`cw_flex` (it would need a cost axis, deferred). It is captured by the
> ex-post scarce-capacity diagnostic (§2.2), so the diagnostic's reshuffled mass **can exceed**
> `cw_flex` — the diagnostic is a *companion* to the denominator, not a subset of it.

### 2.1 Computation order (no circularity)
Under D-F6 v2 the circularity is gone **outright**: `Δ_k` no longer reads any route arrival, so it
is fixed before the route set is even examined. `A_k^min` survives only for `slack_k`/the on-time set:
1. `Δ_k = ready_k + base_transit_h(lane) + sla_offset_h(tier)` (pre-committed; no routes, no `A_k^min`).
2. `A_k^min` ← min Â over the predicates-1–8 routes (tier-free).
3. `min_transit_k = A_k^min − ready_k`; `slack_k = Δ_k − A_k^min` (**may be < 0** — a born-at-risk
   HAWB whose own fastest route is slower than the lane SLA; `flex_k = False`).
4. predicate 9 screens routes against `Δ_k` (with the `z_tier·σ̂` margin).
5. `flex_k` ← ≥2 `θ_flex`-separated on-time options on the post-9 set.

**Corner cases:** (a) a tight EXPRESS promise + large `z_tier·σ̂` margin can prune *even the fastest
route* → fallback, (correctly) excluded from `cw_flex`; (b) `slack_k < 0` (born-at-risk) → no on-time
option → `flex_k = False`. Both are signals, not errors — a too-aggressive promise vs the network.

**Worked example.** TPE→LAX, daily freighters (gap ≈ 24h), ready `t=0`, lane `base_transit = 56h`.
Estimate end-to-end arrivals: F1 = 56h (= `A_k^min`), F2 = 80h, F3 = 104h.
`Δ_k = 0 + 56 + sla_offset`:

| Tier | `sla_offset` | `Δ_k` | on-time flights (≤ `Δ_k`) | `slack_k` | `flex_k` |
|---|---|---|---|---|---|
| EXPRESS | 12h | 68 | F1 (56) only | 12h | **No** (1 option) |
| STANDARD | 24h | 80 | F1 (56), F2 (80) | 24h | **Yes** (2 separated options) |
| DEFERRED | 48h | 104 | F1, F2, F3 | 48h | **Yes** (3 options) |

The "on-time flights" column applies the timing cut `≤ Δ_k` for arithmetic clarity; **predicate 9
additionally subtracts the `z_tier·σ̂` reliability margin** (e.g. with `z_EXPRESS·σ̂ = 8h`, EXPRESS
needs `Â + 8 ≤ 68`, so F1 at 56 clears with 4h to spare — but a larger margin would prune even F1 →
fallback, the §2.1 corner). `flex_k` and predicate-9 admission are computed on the *same* post-9 set.

**Within-tier heterogeneity (D-F6/M-2b).** A shipper `T_dead` makes `Δ_k = min(T_dead, T_SLA)` bite
*per HAWB*, so slack varies within a tier+lane: a STANDARD HAWB with `T_dead = 72` has `Δ_k = 72`,
`slack = 16h < gap`, only F1 on-time → **inflexible**, while its tier-mate with no `T_dead` (`Δ_k = 96`)
is flexible. Without a `T_dead`, `slack_k = sla_offset(tier)` (homogeneous). The replan win: M₁ moves a
flexible STANDARD shipment F1→F2 (it has slack) to free the scarce cheap F1 slot for an urgent shipment
that can only make F1.

### 2.2 Scope — what `flex_k` is, and what it deliberately is not
- **`flex_k` is deadline-slack (time) flexibility.** Two other axes exist — *routing* flexibility
  (alternate path/gateway) and *offload-willingness* (the DEFERRED tier's defining property). They are
  captured **indirectly via tier correlation** (DEFERRED = high slack + offloadable); folding
  offloadability in as a separate term is a flagged future refinement if the proxy proves weak.
- **`cw_flex = Σ_{flex_k} cw_k` is a sum of per-HAWB marginals; `L2` is a portfolio *interaction*
  (freeing a slot for someone else). So `L2/cw_flex` is NOT a clean attribution (M-1).** We keep it as
  the per-flexible-kg headline (D-F8/Decision-1b) but **label it a conservative lower-bound *rate*, not
  an attribution**: it includes flexible-but-inert mass (two cheap uncongested options → movable but
  frees nothing scarce), which only *deflates* the rate. The **value-attributed** companion is the
  ex-post diagnostic: `L2 / (mass actually reshuffled onto/off a binding-capacity arc)` — where the
  value truly came from. Headline rate is conservative; diagnostic explains the concentration. A HAWB
  with one route or no separated, non-dominated 2nd option is **inflexible** → cost but not `cw_flex`.
- **`A_k^min` is the solo fastest route.** Consolidation (sharing a MAWB) can only make realized
  transit *slower*, so the `sla_offset`-derived `Δ_k` is a possibly-tight promise and `cw_flex` is
  conservative. Solo is defensible because predicate 9 intentionally lets group-mates diverge across
  arcs (`air_freight_routing.tex` — a HAWB is not forced to move with its group).

### 2.3 Frozen at `t=0` / instance generation (B-1)
There is no per-HAWB "booking firm-up" instant in the model (only `known_at` reveal + physical tender).
So `flex_k`/`cw_flex` are **computed once at `t=0` on the initial schedule snapshot, identically for
all five arms** (`H₀/M₀/M₁'/M₁/π_hind`), and frozen — this is the **reporting denominator**. Computing it
at arrival instead would let diverged arms see different post-9 sets → an arm-dependent denominator →
`L2/cw_flex` incomparable across arms. **Invariant (pytest): `cw_flex` bit-identical across all arms.**
A HAWB flexible at `t=0` but inflexible by tender still counts (the denominator is fixed; this only
*deflates* the rate — consistent with the conservative-lower-bound framing). Distinct from the **live
reshuffle set** M₁ acts on at step `t`, which is necessarily time-varying (an option vanishes when a
flight fills). Report against frozen `cw_flex`; reshuffle against the live set.

---

## 3. Tier → parameter mapping (single source of truth)

2-FLEX owns one table, `TierSpec`, consumed everywhere:

```
TierSpec(tier) -> { sla_offset_h,   # T_SLA = ready_k + base_transit_h(lane) + sla_offset_h(tier)
                    z_tier,          # predicate-9 reliability multiplier
                    w_sp }           # W_k tardiness weight
```

- **2a** draws `tier(k)` from a configured mix (default **20/40/40**, D-F5), sets `T_SLA = ready_k +
  base_transit_h(lane) + sla_offset_h(tier)` (D-F6 **v2** — the lane's pre-committed achievable transit,
  **not** the shipment's `A_k^min`), **draws an optional shipper `T_dead` per HAWB** (with a config
  probability; D-F6/M-2b — gives within-tier slack heterogeneity), then `Δ_k = min(T_dead, T_SLA)`,
  stored as `soft_deadline_h`. Replaces 2a's current free `soft_deadline_h` draw.
  **No route dependency:** `Δ_k` is graph-free under v2 — it needs only the lane's `base_transit_h`
  table, so the v1 generator → graph-gen → 2b edge for the deadline is gone. (`A_k^min` is still
  computed downstream, but only for `slack_k` / the on-time set, never for `Δ_k`.)
- **Predicate 9** and **C.10 `W_k`** read `z_tier` / `w_sp` from the same table → no drift in the
  *source*. Note `z_tier` and `sla_offset` are **not independent in effect** (predicate-9 admission
  turns on `sla_offset − z_tier·σ̂`); EXPRESS sets both stringent, so the calibration note must
  **co-tune** them (m-1) or EXPRESS routes-to-fallback more than intended.
- All read from a table frozen at `t=0` (§2.3); the per-shipment `Δ_k` / `z_tier` promises are frozen
  at booking (`backtest §6` invariants).

---

## 4. How 2-FLEX feeds the proof

- **All arms** (`H₀/M₀/M₁`) see the same tiers; the reshuffle/replan value is realized on `cw_flex`.
  `H₀`'s canonical failure (the early flexible HAWB burning the cheap slot) requires it to know which
  HAWBs are flexible — hence 2-FLEX is upstream of all arms, not just 2c.
- **Sandbagged flexibility = the band's primary sensitivity** (`backtest §7`) — but **perturb the
  *inputs* to the derivation, never the derived label** (D-F2 forbids relabeling; a flipped `flex_k`
  would be overwritten by re-derivation). Two coherent knobs: **shrink `sla_offset_h`** (tightens `Δ_k`
  → fewer HAWBs clear the ≥2-separated-options test) and **raise `θ_flex`** (demand wider separation
  before a 2nd option counts). If a stochastic *classifier-error* stress is wanted, inject ε-noise on
  the **live reshuffle set** M₁ acts on (membership noise), explicitly separate from the frozen
  `cw_flex` derivation — not a flip of the derived label. If savings survive, the number is robust to
  our biggest assumption.
- **Locked transition tied to physical tender** (`backtest §2`): a HAWB leaves the flexible set when
  it physically tenders (CFS receipt), not at a notional flag — so the reshufflable set is realistically
  bounded.

---

## 5. Dependencies / interactions

| Consumer | Uses | Note |
|---|---|---|
| 2a generator | `TierSpec` → tier mix + `T_dead` draw + `Δ_k` | `Δ_k` graph-free (lane `base_transit_h`); no route enum |
| 2b transit | `A_k^min` via `route_reliability` Â | min over **pre-predicate-9** routes (§2.1); only for `slack_k` |
| Predicate 9 | `z_tier` | OTP admission filter |
| C.10 `W_k` | `w_sp` | tardiness weight / prioritization base |
| Backtest 3a/3d | `cw_flex`, sandbagging | denominator + primary stress |

Ordering note: predicate 9 needs `z_tier` (2-FLEX) and `route_reliability` (2b) — both exist before
graph-gen wiring. 2-FLEX itself is pure/deterministic given tier assignment; no solver.

---

## 6. Decisions (RESOLVED — Session 29)

- **D-F1 ✓ three tiers** EXPRESS/STANDARD/DEFERRED, the standard air structure; magnitudes `[CAL]`.
  *(vs. a continuous flexibility score — rejected: tiers are how the product is actually sold and how
  `z_tier`/`w_sp` are set; a continuous score has no commercial referent.)*
- **D-F2 ✓ flexibility is derived** — operative predicate = **≥2 tier-admissible (post-9) on-time
  options, `θ_flex`-separated AND non-dominated** (drop a later-and-pricier 2nd option; M-3); `slack_k
  = Δ_k − A_k^min`. Not an assigned label → `cw_flex` un-inflatable. **`A_k^min` on the
  pre-predicate-9 set** (breaks the circularity, §2.1). Cost-only (same-flight) flexibility is out of
  scope of `flex_k`, captured only by the ex-post diagnostic.
- **D-F3 ✓ one `TierSpec` table** is the single source of truth for `Δ_k`/`z_tier`/`w_sp`; 2a, 2b,
  predicate 9, and C.10 all read it (no per-component tier constants).
- **D-F4 ✓ sandbagging perturbs derivation *inputs*, never the derived label:** shrink `sla_offset_h`
  + raise `θ_flex` (optional ε classifier-error noise on the *live* reshuffle set, not on frozen
  `cw_flex`). The band's primary sensitivity.
- **D-F5 ✓ tier mix is a config**, default **20/40/40** (EXPRESS/STANDARD/DEFERRED); sweepable.
- **D-F6 v2 ✓ `Δ_k` is tier×lane-derived, with per-HAWB heterogeneity** (graph-free; supersedes v1):
  `T_SLA = ready_k + base_transit_h(lane) + sla_offset_h(tier)` (the lane's pre-committed achievable
  transit, **not** `A_k^min`); the generator draws an optional shipper `T_dead` per HAWB (config prob);
  `Δ_k = min(T_dead, T_SLA)`. So slack varies within a tier when `T_dead` bites (else
  `slack_k = Δ_k − A_k^min`). (vs. Option B random-deadline-then-classify, rejected.)
- **D-F7 ✓ `flex_k`/`cw_flex` frozen at `t=0`/instance-generation**, identical across all five arms
  (`cw_flex` arm-invariance pytest); the reporting denominator. Distinct from the time-varying live
  reshuffle set M₁ acts on (§2.3). (B-1.)
- **D-F8 ✓ per-flexible-kg headline kept (Decision-1b)** but labeled a **conservative lower-bound
  rate, not an attribution** (`L2/cw_flex` divides an interaction by a sum-of-marginals; M-1); the
  value-attributed companion is `L2 / reshuffled-against-binding-capacity mass` (ex-post diagnostic).

## 7. Definition of Done (2-FLEX component gate)

- [ ] `TierSpec` table with the three tiers + ordering invariants asserted
      (`z` / `w_sp` / expected-slack monotone in tier).
- [ ] `classify(hawb, route_options) -> (tier, slack_k, flex_k)` deterministic; `flex_k` =
      ≥2 tier-admissible (post-9) on-time options, `θ_flex`-separated AND non-dominated (single
      operative predicate); **dominated later-and-pricier 2nd option rejected** (test).
- [ ] **Computation order (no circularity):** `A_k^min` computed on the **pre-predicate-9** set with
      no reference to `Δ_k` (test asserts the ordering); `Δ_k = min(T_dead, T_SLA)` then `flex_k` follow.
- [ ] `flex_k` and predicate-9 admission use the **same** post-9 candidate set (test).
- [ ] **`cw_flex` arm-invariance** (B-1): computed at `t=0` on the initial schedule, bit-identical
      across `H₀/M₀/M₁/π_hind` (pytest). The live reshuffle set is the separate, mutable object.
- [ ] **Within-tier slack heterogeneity** (D-F6): a HAWB with a biting `T_dead` has smaller `slack_k`
      (and can flip to inflexible) than a tier-mate without one (test).
- [ ] Ex-post **scarce-capacity diagnostic** reported; its reshuffled mass **may exceed `cw_flex`**
      (companion, not subset — M-3); the per-flexible-kg headline is labeled a conservative
      lower-bound rate, not an attribution (D-F8).
- [ ] 2a generator refactored: draw tier + `T_dead` + derive `Δ_k` (graph-free under D-F6 v2 —
      lane `base_transit_h` table, no route enum); **generator tests updated** for tier×lane
      deadlines (the free `soft_deadline_h` range assertions change — "still green" is not expected).
- [ ] `sandbag(config)` knob = shrink `sla_offset_h` + raise `θ_flex` (input perturbation, not label
      flip); asserted **weakly** non-increasing in `cw_flex`, with ≥1 configured perturbation that
      **strictly** shrinks it on the test instance (m-3 — avoids a flaky strict-monotone assertion).
- [ ] Isolation tests: tier ordering invariants; inflexible HAWB (1 option / no separated-non-dominated
      2nd) excluded from `cw_flex`; flexible HAWB included; **fastest-route-fails-predicate-9 →
      fallback-only, excluded** (§2.1 corner). Pure/deterministic — no solver, no timing assertions.

## 8. Open questions — RESOLVED (Session 29)

All resolved → D-F1…D-F8 (§6), across two critique rounds: three tiers; flexibility derived
(≥2 `θ_flex`-separated, non-dominated options); mix config default 20/40/40; `Δ_k = min(T_dead,
T_SLA)` with per-HAWB `T_dead` heterogeneity (Decision-2b); `cw_flex` frozen at `t=0`, arm-invariant;
per-flexible-kg kept as a conservative-lower-bound rate, not an attribution (Decision-1b).
