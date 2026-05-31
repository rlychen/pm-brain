# Air Model Review — Working Notes (Session 13, paused 2026-05-20)

**Purpose:** Resume point for the `air_freight_routing.tex` Draft-v2 review. The review is mid-flight: items 1–7 processed, items 8–19 not yet started. **Start tomorrow from the bucket formulation in §"Item 4" below — that is the agreed item-4 rewrite spec.**

---

## The 19-item review checklist (full list)

Phase A — Scope: (1) abstract, (2) MVP scope bullets, (3) Problem Statement "such that" list.
Phase B — 3 design decisions: (4) per-shipment vs per-MAWB PWL, (5) IATA next-break-down via tier enumeration, (6) hybrid pivot.
Phase C — PWL correctness: (7) worked comparison table, (8) derived-shorthand definitions, (9) `O_f` partition.
Phase D — Variables + constraints: (10) Decision Variables, (11) P.8, (12) P.9, (13) P.2/P.3 + P.4–P.7, (14) P.10, (15) P.17/P.18/P.19/P.21.
Phase E — Objective + consistency: (16) Objective Function, (17) Linearization, (18) Open Items + Tractability, (19) notation sweep.

---

## Items 1–7 — outcomes

**1. Abstract / 5 supply types — OK.** User confirmed.

**2. Currency / FX — LOCKED (Session 14, 2026-05-21).** Currency conversion is specified in `data_model.md §7`: USD canonical; all catalog rates carry a `currency` field; converted to USD at solve time. The air model `.tex` parameter tables just say "USD" and never cross-reference it.

Research outcome (Session 14): air cargo rates are quoted in the **origin country's local currency** per IATA TACT rules — so the `currency` field is genuinely necessary, not optional (an air model spanning multiple origins sees multiple currencies). The industry's own FX mechanism (IATA Rates of Exchange) is itself a *periodically-fixed published rate table*, not live spot — so a static-table approach is the domain norm.

**Decision — minimal:** the MILP does cost comparison for routing, not invoicing/settlement. MVP = USD canonical, **a single FX table, convert at solve time.** No reconciliation log, no CAF, no hedging, **no per-run FX pinning** (that is an audit-reproducibility concern, deferred to Phase 2+; it can ride the `data_model.md §5` per-run snapshot mechanism when that is actually built). **Pending edit:** one plain paragraph in the air model Parameters section — "all rates converted to a USD canonical currency per `data_model.md §7`; conversion applies a single FX table at solve time" — no pinning language. Batch with the item-4 rewrite.

**3. Soft deadline + LINEAR tardiness penalty — LOCKED (Session 14, 2026-05-21).** P.15 (deadline) and P.20 (SLA) flip from hard to soft. Agreed formulation:
- New continuous var `τ_k ≥ 0` (tardiness). `τ_k ≥ t_k(d(k)) − D_k` where `D_k = min(T_k^dead, t_k^{rdy,early} + T^{SLA}_{sp(k)})`; combined with `τ_k ≥ 0` gives `τ_k = max(0, lateness)`.
- P.15 / P.20 change from hard to soft; the tardiness var absorbs the overage.
- Keep one hard backstop: absolute drop-dead `T_k^{abs}` (perishables; `∞` for GEN). Beyond it → genuinely infeasible.
- **Objective gains `+ Σ_k w_{sp(k)} · τ_k` — LINEAR, not quadratic.** Session-14 decision reversed the Session-13 quadratic choice: linear-soft is the genuine MVP — no `q_k` aux var, no tangent-cut grid `τ̂_j`, no PWL apparatus. Plain linear term.
- **Quadratic = DEFERRED refinement.** The quadratic penalty `Σ w·τ_k²` (PWL-linearized via convex tangent cuts `q_k ≥ 2τ̂_j·τ_k − τ̂_j²`) is the upgrade if calibration shows linear gives bad lateness-spreading behavior. Tangent-cut spec preserved here for future-self. Add it only when measured behavior justifies it.
- This supersedes the deferred `§11` item `sla-soft-otp` (linear + deferred) — promote to active, keep linear.
- **Weights `w_{sp(k)}` are an exchange rate, not a relative scale.** `τ_k` is in days; `w` has units USD/day and the term competes directly against routing cost (USD) in the objective. Absolute magnitude sets aggressiveness (overpay to route fast vs. eat delay); inter-tier ratios set priority (premium ≫ economy). Genuine calibration exercise. **MVP uses placeholders flagged `CALIBRATION NEEDED`.**
- Composition with Finding S: item 3 binds `τ_k` against deterministic `t_k(d(k))`; Finding S later swaps that for a P85–P90 Transit Time Service quantile. Walking skeleton stays deterministic until the TT service exists.

**4. Per-MAWB PWL — RESOLVED via the bucket formulation.** See dedicated section below. (User overruled the Session-12 per-shipment decision; per-shipment "makes no sense" — loses density mixing. Then rejected explicit-MAWB-entity formulations because of the count + symmetry problem. Resolution: model the MAWB as an implicit `(flight-leg, supply-offer)` bucket.)

**5. IATA next-break-down via tier enumeration — no comment from user.** Treat as no-objection; the mechanism survives the item-4 rewrite (tier enumeration is now per-bucket, not per-shipment).

**6. Hybrid pivot (`ps`/`pu` cost-basis split) — MOOT.** The `ps`/`pu` split existed only because of per-shipment precomputation. The bucket formulation collapses it (everything rates on a bucket aggregate). No separate review needed; it gets rewritten by item 4. Doc location for reference: `§"Supply Option Catalog (Per Flight)"` — the `cost_basis` attribute row, the "Per-ULD pivot cost" paragraph, the "Multi-segment per-ULD PWL (deferred)" paragraph.

**7. Worked comparison table — REAL BUG FOUND.** The 12 cells are arithmetically correct, **but TACT is computed by a different mechanism than the doc claims.** TACT (`$960/$3,040/$4,800`) uses **min-over-flat-break-rates**: `cost = min_b (rate_b · max(CW, break_b))`. BUC/NAC/BSA-pivot use the cumulative PWL `eq:rate-fn`. The doc's `§"Rate Function"` presents `eq:rate-fn` and claims TACT is a special case of it — it is not (plug TACT tuples into `eq:rate-fn` cumulatively → `$1,052.50` at 200 kg, not `$960`). The `(b_i, m_i)` notation means **flat break rate** for TACT but **cumulative segment marginal rate** for BSA-pivot — two semantics, one notation. **Fix in the item-4 rewrite:** explicitly define two rate-function families — (a) cumulative PWL, (b) min-over-flat-breaks.

**Session-14 amendment (CONFIRMED).** Do not hard-wire family → supply-type *name* (NAC/BUC can be quoted either way by different carriers; the rating mechanism is a property of the contract, not the label). Make **`rate_family ∈ {cumulative_pwl, min_flat_breaks}` an explicit attribute of each offer** in the supply-option catalog, populated from contract terms; the MILP build dispatches on it. Also: the worked comparison table must label each column with its family, and the $1,052.50-vs-$960 gap is shown *as the point* (two mechanisms), not silently corrected.

---

## Item 4 — RESOLVED: the bucket formulation (THE item-4 rewrite spec)

**The MAWB is a bucket, not an object.** A MAWB has no independent identity — it is "the HAWBs tendered together on a given routing under a given rate offer." Index it by what it is.

**Bucket = (flight-leg `f`, supply-offer `o`).** Concrete, enumerable, every one distinct. Nothing to permute.

**Variables:**
- `y_{f,o,k} ∈ {0,1}` — HAWB `k` tendered on flight `f` under offer `o`. **Same variable as the Session-12 model. No `h_{k,m}`.**
- `CW_{f,o} ≥ 0` — bucket chargeable weight (continuous).

**Density mixing (this is where the §3 density-mixing benefit lives):**
- `Wt_{f,o} = (1+δ)·Σ_k w_k · y_{f,o,k}` — aggregate actual weight (`δ` = dunnage factor)
- `Wv_{f,o} = Σ_k v_k·167 · y_{f,o,k}` — aggregate volumetric weight
- `CW_{f,o} ≥ Wt_{f,o}` and `CW_{f,o} ≥ Wv_{f,o}` — bucket chargeable weight = max of the two (dense cargo absorbs light cargo's volumetric slack). Two inequalities, no binary.

**Cost of the bucket = `R_o(CW_{f,o})`** — the concave rate on the aggregate, linearized:
- TACT offer: per-break billed-weight `BW_{f,o}` + break-selection binary `γ_{f,o,b}` (exactly-one over breaks `b`); `BW ≥ CW_{f,o}`; `BW ≥ break_b·γ_b`; cost `Σ_b rate_b·BW_{f,o,b}`.
- Single-pivot BSA: convex (flat floor then one marginal rate) → 2-inequality pivot linearization, no break binary.

**Why this resolves the count + symmetry problem:**
- *How many MAWBs?* — Not a decision. Buckets are `(flight-leg, offer)` pairs; index set fixed and enumerable from supply data.
- *Symmetry?* — None. Bucket `(CI5, TACT)` ≠ bucket `(BR10, TACT)` — concretely distinct, no permutation group.
- *Size blowup?* — None. Decision variables stay `y_{f,o,k}` (same order as Session 12). Added: `CW_{f,o}` aggregates + `γ_{f,o,b}` break-binaries, both indexed on `(flight, offer[, break])` — a few hundred, not `O(|K|²)`.

**Consolidation is still jointly optimized:** two HAWBs are consolidated iff the optimizer puts them in the same `(f,o)` bucket. The concave rate is subadditive (`R(a+b) ≤ R(a)+R(b)`), so the optimizer merges co-routable HAWBs without being told to. Consolidation emerges from the objective.

**What it can't represent (and why that's fine):** two separate MAWBs on the same `(flight, offer)`. With a concave rate the optimizer never wants that — merging is always weakly cheaper. Separate MAWBs on one flight happen for documentary reasons (consignees, paperwork) which don't change cost-optimal routing. MAWB document-splitting is post-optimization output.

**What item 4 actually is (a contained change, not the explicit-MAWB restructure):** the Session-12 model with the cost moved from per-shipment `c_o(cw_k)` to per-bucket `R_o(CW_{f,o})`:
- Add `CW_{f,o}` aggregate-weight variables + the two density-mixing inequalities.
- Move the rate function from precomputed-constant to a linearized function of `CW_{f,o}`.
- Add per-bucket break-selection binaries for TACT offers.
- Drop the per-shipment `c_o(cw_k)` precomputation and the `ps`/`pu` cost-basis split (collapses — everything rates on a bucket aggregate).
- No `M`, no `h_{k,m}`, no symmetry-breaking constraints, no column generation for MVP.

**Rejected alternatives (for the record):** explicit MAWB set + `h_{k,m}` assignment (count decision + `|M|!` symmetry); representative/canonical-naming formulation (symmetry-breaking ordering constraints — "band-aid", `O(|K|²)` vars remain); set-partitioning + column generation (correct long-term, too heavy for MVP — stays on the `scalability.md` shelf); rule-based pre-consolidation outside the MILP (loses joint optimization).

**Cross-cutting:** the same principle applies to the LCL container model — container = `(sailing, slot-type)` bucket, not explicit container entities (LCL adds 3D bin-packing on top, but the symmetry-free bucket indexing transfers). The LCL rework inherits this.

**Point 1 honest correction (carried from the discussion):** picking the TACT break IS `K` binaries per bucket (`K ≈ 6`) in a flat exactly-one — the break is folded into the option/bucket index, so it's the same exactly-one structure as the offer choice, not a separate SOS-2 layer. Cheap (assignment polytope, tight LP relaxation) but not zero. Earlier "for free" was an overclaim.

---

### Item 4 — Session 14 additions (2026-05-21)

**Bucket formulation (E) endorsed by user.** Four scrutiny points to fold into the rewrite:

1. **State the monotonicity condition.** `CW ≥ Wt`, `CW ≥ Wv` is exact only because `R_o` is monotone non-decreasing in `CW` (true for both rate families). Add an explicit remark — it is the correctness condition for that relaxation.
2. **TACT cost is a binary×continuous product.** `Σ_b rate_b·BW_{f,o,b}` needs the standard disaggregation (`BW_b ≤ M·γ_b`, `BW_b ≥ break_b·γ_b`, `BW_b ≥ CW − M(1−γ_b)`, `BW_b ≤ CW`). Write the full set; "billed weight + exactly-one binary" understates it.
3. **Empty-bucket handling.** A bucket with no HAWBs must contribute zero cost and select zero breaks. Break selection is "exactly-one IF bucket active, else zero": `Σ_b γ_b = active_{f,o}`, `active_{f,o} ≤ Σ_k y_{f,o,k}`, `active_{f,o} ≥ y_{f,o,k} ∀k`. Otherwise empty buckets book a phantom minimum charge.
4. **Bucket is a rating construct.** One sentence in the rewrite: the `(f,o)` bucket rates cargo; physical compatibility (DGR segregation, ULD fit) is enforced independently at the ULD/flight layer (P.16/P.17).

**MVP build decision — two consolidation modes behind a config toggle `consolidation_mode`:**
- `bucket` (option E) — exact. MILP commodity unit = individual HAWB; `y_{f,o,k}`; joint consolidation + routing. The correctness path; isolation optimality tests run against this.
- `preprocess` (option C) — fast/approximate. A heuristic pre-groups HAWBs; MILP commodity unit = pre-formed group. **Same bucket MILP, grouped input** — `y_{f,o,g}` with `|g| < |K|`, smaller and faster. Suboptimal (grouping frozen before routing is known). Roles: fast path for end-to-end / agent / orchestrator testing; baseline to measure E's consolidation gain; scale fallback.
- C's pre-grouping rule (CONFIRMED Session 14): two HAWBs are groupable iff they share at least one valid origin-CFS→destination-CFS path. **Not implemented pairwise** — `O(|K|²)` and wrong ("shares a path" is non-transitive; a group needs one *common* path, not pairwise-common paths). Implementation: (1) hash-bucket by categorical pre-filter key (origin gw, dest gw, cargo-type class, carrier/screening/embargo policy class) — cross-bucket pairs are provably incompatible, never examined; (2) within a bucket the categorical subgraph is identical, only the `[ready, deadline]` window varies → sub-bucket by coarse time window; (3) one feasibility check per candidate group via the existing subgraph constructor on intersected constraints + tightest window `[max ready, min deadline]`; split empty groups. Cost `O(|K| log|K| + |groups|·subgraph-build)`.
- C is not E's correctness oracle (different, worse objective). C tests = feasibility + preprocessing-rule correctness; E tests = optimal value.
- The toggle is a code/build concern, not a math-model element — record it in the build plan, not as a model parameter.

**Model doc — options A–E section (combined rewrite).** Add a "Consolidation: alternatives considered" section documenting all five — A explicit `h_{k,m}` (count + `|M|!` symmetry), B canonical naming (band-aid, `O(|K|²)`), C rule-based preprocessing (frozen grouping, lossy, fast), D set-partitioning + branch-and-price (correct large-scale answer), E bucket (chosen). **D gets the fullest treatment** as the scalability path — set-partitioning master, SPPRC-style pricing — cross-referencing `scalability.md`. Write this as part of the combined rewrite so it uses the final bucket notation; not before.

---

### BSA cost modeling — converged design (Session 14, 2026-05-21)

A deep-dive with the user (research on real BSA contract structure + a design walkthrough) produced the converged design below for the `per_uld_pivot` rate family. **This is the item-4 spec for BSA cost.** Replaces the earlier loose "2 vs 3 families" framing.

**Research basis.** A BSA is contracted per (carrier, O-D airport pair, recurring flight slot), typically one IATA season (~26 wks; range a few months to a year). Fixed allotment per flight (N ULD positions). Hard BSA = take-or-pay; soft BSA = walk away free. Take-or-pay is a *weight* commitment (pivot / commitment weight), settled **per-flight OR period-equalized** — the **equalization clause** averages the pivot over a period (typically monthly). Sources: Cargo Flowers, Ethiopian Cargo, Xeneta, Amaruchkul (game-theoretic allotment), Hellermann (flexible carrier–forwarder contracts).

**1. Three rate families.** `rate_family ∈ {cumulative_pwl, min_flat_breaks, per_uld_pivot}` — a per-offer catalog attribute. `cumulative_pwl` + `min_flat_breaks` rate on the bucket aggregate `CW_{f,o}` (TACT/SCR/NAC/spot). `per_uld_pivot` = BSA. This folds the old `ps`/`pu` `cost_basis` attribute into `rate_family` — supersedes item 6's "ps/pu collapses."

**2. BSA cost structure.** Per claimed ULD, pivot `π`, contract rate `r_c`, chargeable weight `w`:
`cost = r_c·max(w, π) = r_c·π` **[SUNK]** `+ r_c·max(0, w−π)` **[MARGINAL at r_c]**.
Sunk up to the pivot; marginal at `r_c` above it. Soft BSA → `hard_c = 0`, no pivot floor (pay actual only).

**3. Two settlement bases.** New contract attribute `settlement_basis ∈ {per_flight, equalized}`.
- `per_flight` — pivot binds each flight independently. = **existing P.10 unchanged** (`C_{f,u,c} ≥ max(Σ actual, π·z)`, cost `r_c·C_{f,u,c}`); self-contained in one solve. The "easy" case.
- `equalized` — pivot averaged over the period (equalization clause). `cost = r_c·max(W_{c,t}, P_{c,t})`, where `W_{c,t} = Σ_{f∈F_c(t)} w_{c,f}` (period chargeable weight tendered; `w_{c,f}` = the BSA bucket's chargeable weight on flight `f` = `CW_{f,o}` for the BSA offer), and `P_{c,t} = π·M_{c,u,t}` (period commitment weight).

**4. Per-flight ≠ equalized re-indexed.** Per-flight `= Σ_f max(actual_f, π·z_f)` (sum of maxes); equalized `= max(Σ_f actual_f, π·Σ_f z_f)` (max of sums). `Σmax ≥ maxΣ` — genuinely different formulations, not one with a re-indexed sum.

**5. Equalized = a per-solve sunk allowance — NOT a price, NOT an in-MILP period constraint.** The MILP solves per-batch and cannot see future flights, so the period `max(Σ_period)` cannot live in one solve. Decompose by **remaining sunk allowance**:
- `r_c·max(W_{c,t}, P_{c,t}) = r_c·P_{c,t}` [constant, sunk — booked outside the MILP, doesn't affect argmin] `+ r_c·max(0, W_{c,t}−P_{c,t})` [marginal].
- The controller passes each solve the **remaining sunk allowance** `A_c = P_{c,t} − (chargeable weight consumed on contract c so far this period)`, in kg.
- The MILP sees BSA contract `c` as a **2-piece cost**, one breakpoint at `A_c`: weight `0→A_c` is **free** (sunk, already paid), weight above `A_c` costs `r_c`/kg.
- **Exact, not an approximation.** Processing solves in time order with `A_c` updated from actual consumption, the per-solve 2-piece costs sum exactly to the true period cost `r_c·max(W_{c,t}, P_{c,t})`.
- **A flat effective-rate / Lagrangian-price scalar `r̃` was considered and rejected** — it cannot represent the sunk/marginal kink: `r̃=0` wrongly frees over-commitment weight (and over-concentrates cargo onto whichever contract got the low rate); `r̃=r_c` wrongly charges sunk weight. Counterexample: 200 kg, two BSAs each with `A=100` → the allowance model fills both free bands 100/100; a flat-price model dumps all 200 onto the `r̃=0` contract. The allowance represents the kink exactly.
- `A_c` is per-contract; within a solve it is one free-band capacity shared across that contract's flights in the window.

**6. The "controller" is a consumed-weight accumulator — not a system.** Its whole job: keep a running total of chargeable weight consumed per contract this period, and emit `A_c = P_{c,t} − consumed` to each solve. No price, no Lagrangian, no subgradient, no tuning, no forecast in the cost loop. It IS the "upstream period-commitment model" the air model already references for `M_{c,u,t}` — not new scope, and far smaller than first framed. (A demand forecast remains useful as an operator *alert* — "projected to underuse `P_{c,t}`; take-or-pay loss coming" — but that is an alert, not part of the cost mechanism.)

**7. Capacity is always the full true allotment — never throttled.** The MILP always sees every ULD position the contract gives, at full physical capacity, on every flight. No `α`, no soft sub-cap. "Can use more" = capacity (a standing precondition, always at max); "will use more" = price `r̃` (the only lever). Overflow beyond the allotment → spot offer (already modeled). No over-utilization penalty: over-*commitment* weight is just `r_c`; over-*allotment* is spot.

**8. ULD physical capacity is a hard bound on recoverable commitment.** One LD3 holds ≈752–1,587 kg chargeable weight (volume-bound → weight-bound, cargo-density-dependent); LD7 ≈1,854–4,626; PMC ≈1,253–6,804. If remaining physical capacity < remaining commitment, `P_{c,t}` is missed and the take-or-pay shortfall is paid — unavoidable, no lever fixes it; the controller's forecast job is to see it coming and warn the operator. Physical capacity is already enforced by P.2 (volume) / P.3 (weight).

**9. P.7 implication — resolve in the rewrite.** P.7 (period minimum utilization, hard `Σ_{F_c(t)} z ≥ M_{c,u,t}`) *is* the equalized take-or-pay. The converged design handles equalized take-or-pay via the exogenous per-solve allowance `A_c`, not a hard in-MILP period constraint — and a hard period-minimum inside a per-batch solve risks spurious infeasibility (not enough cargo in the window). **So hard P.7 should be removed for equalized contracts, replaced by the allowance mechanism.** P.6 (period allocation cap — upper bound) is also a cross-solve coupling constraint; review for the same per-run-residual treatment, but upper bounds aren't infeasibility-dangerous → lower priority. P.4 / P.5 are per-flight → unaffected.

**10. Net for the rewrite / Chunk 1.** The equalized BSA adds **no in-MILP period constraint, no slack, no binary**. It adds one exogenous parameter per (contract, period): the remaining sunk allowance `A_c` (kg), supplied per solve. BSA contract `c` is modeled as a **2-segment offer** — free up to `A_c`, then `r_c`/kg — which drops directly into the existing `O_f` offer catalog. Per-flight take-or-pay = existing P.10 unchanged. `settlement_basis` selects. The air model declares `A_c` an exogenous parameter; the consumed-weight accumulator is a trivial separate component.

---

## Finding S — Service-product SLA (P.20) design critique [parallel session, 2026-05-20]

A separate session reviewed the §service-products / P.20 design and ran web research. Captured here verbatim-in-substance so it is not lost.

> **DECISION (Session 14, 2026-05-21): implement Change 1 only. Changes 2 and 3 deferred.**
> - **Change 1 — ACTIVE in the combined rewrite.** P.20 soft is already delivered by item 3 (linear tardiness). The rewrite adds: a documented hook that `t_k(d(k))` becomes a P85–P90 Transit Time Service quantile once the TT Service is integrated (MVP stays deterministic); and language framing the SLA as a *planning bound*, not a contractual guarantee. No new variable/constraint beyond item 3.
> - **Change 2 (offload priority into the MILP) — DEFERRED.** Not implemented in MVP. Recorded as a new air-model Open Item: a `supply_class ∈ {confirmed, best_effort}` per-offer catalog attribute + a `confirmed_only` product attribute + a new subgraph pre-filter predicate (express ⟹ confirmed-capacity options only). Probabilistic offload-cost modeling is a further P1.
> - **Change 3 (A2A/D2D SLA-endpoint attribute; `max_hops`/`direct_only`) — DEFERRED.** Recorded as a new air-model Open Item.
> - **Recording:** during the combined rewrite, Changes 2 and 3 are written as entries in the air model's "Open Items and Future Extensions" (§sec:deferred) — keeps the deferred analysis with the model. The full Finding S critique below stays in this doc as the source rationale.

**Verdict:** the catalog *structure* is right; the *hard transit-time SLA as the primary lever* is wrong. Air freight is sold as named service tiers — but the tier primarily buys **offload / booking priority** ("rides as booked"), not a contractual door-to-door clock. The model elevated the weak lever to a hard MVP constraint and demoted the real one.

**Research confirmation:**
- Named tiers are real and standard. exFreight (a real forwarder) sells four named tiers — Deferred / Standard / Express / Express Direct, pre-defined products with individualized pricing. Lufthansa Cargo sells td.Basic / td.Pro / td.Flash / td.Zoom. The `sp(k)` FK-into-a-catalog pattern and tenant-scoping are correct.
- **But the tier differentiator is NOT a transit SLA.** exFreight gives no transit-time guarantees — it uses probability language ("very likely / extremely likely to ride as booked"). Tiers differ on (1) **offload priority** — likelihood cargo flies as booked vs. gets bumped for higher-paying freight; (2) **routing structure** — Express = direct, Standard = 1–2 stops, Deferred = multiple. Transit time is a *consequence*, quoted as an estimate.
- Hard guarantees exist but narrowly — Forward Air "Money Back Guarantee", Alaska Air Cargo Priority (refund if >6h late). Carrier/integrator products, airport-to-airport, money-back remedy — not how a mid-market forwarder's general air product works.

**The 5 critique points:**
1. **Wrong lever elevated.** P.20 makes tier ≈ a hard transit-time bound. Reality: tier ≈ booking priority + routing quality; transit time is estimated. Standard forwarder air products carry no contractual transit guarantee.
2. **Real lever buried.** `handling_tier` (priority/standard/economy) — the field that actually corresponds to "rides as booked" — is shunted to the rolling-horizon scheduler and excluded from the MILP; soft-OTP deferred to P1. The genuine differentiator sits outside the optimizer; the weak one is inside as a hard constraint.
3. **Determinism mismatch → false precision.** P.20 binds `t_k(d(k))` (from deterministic time-propagation P.11–P.13 over scheduled flight times) against the SLA. Air cargo is probabilistic — that is *why* tiers exist. A hard constraint on point-estimate arc times will (a) declare infeasible routes ops would run with buffer, and (b) give false comfort. P.20 should bind against a **P85–P90 transit-time quantile from the Transit Time Service**, not a deterministic sum.
4. **D2D-from-cargo-ready hard-coded.** The SLA clock is fixed door-to-door from `t_k^{rdy,early}`. Lufthansa td products are airport-to-airport; many forwarder products quote A2A with door legs as add-ons. SLA endpoints (A2A vs D2D) should be a **product attribute**, not a fixed assumption.
5. **Routing structure not expressible.** "Express Direct = no hub" is a real product attribute; a `max_hops` / `direct_only` product field would match how the tier is sold.

**Recommendation — keep / change:**
- **Keep:** the catalog, the `sp(k)` FK binding, tenant scoping, the `min(SLA, deadline)` composition, the mode/carrier/ac_type pre-filter predicates.
- **Change 1 — P.20 soft or quantile-bound.** Treat the transit SLA as soft with an OTP penalty in MVP (matches reality — *already in motion via review item 3*, soft deadline + quadratic tardiness). If kept hard anywhere, bind against a P85–P90 TT-Service quantile and document it as a *planning bound*, not a contractual guarantee.
- **Change 2 — offload priority into the model.** A product field the MILP can see — e.g., express tier ⟹ only confirmed/guaranteed-capacity supply options admissible (no spot/best-effort space). This is the lever the industry actually sells.
- **Change 3 — SLA-endpoint attribute (A2A / D2D)** added to the product tuple; optionally `max_hops` / `direct_only`.

**Connection to item 3:** review item 3 already makes P.15/P.20 soft with a quadratic tardiness penalty — so "Change 1, soft" is in motion. Finding S *adds*: bind tardiness against a TT-quantile (not deterministic times); bring offload priority into the MILP; add A2A/D2D + `max_hops` product attributes.

**~~OPEN DESIGN QUESTION~~ — RESOLVED (Session 14):** Change 2 is deferred (see decision block at the top of this section). The proposed mechanism — a `supply_class` per-offer attribute + `confirmed_only` product attribute + pre-filter predicate — is recorded as a deferred Open Item, not implemented in MVP.

**Sources:** exFreight air freight service levels; FreightAmigo express/standard/deferred; Lufthansa Cargo td.Pro / td.Flash / td.Zoom; Air Cargo News on the Lufthansa Cargo portfolio; Forward Air Service Conditions (Guaranteed Service Plus); Alaska Air Cargo Priority.

---

## Items 8–19 — review outcomes (Session 14, 2026-05-21)

Items 8, 10, 11, 12, 14, 16, 17, 19 — **rewritten by item 4 / done post-rewrite.** No standalone findings; review after the combined rewrite lands (8 = shorthands change; 9 = `O_f` partition changes; 19 = notation sweep last). Items **13, 15, 18 reviewed in full** — findings below feed the combined rewrite.

**Item 13 — P.2/P.3 + P.4–P.7:**
- **13-A — REAL BUG, fix outright.** P.3 (ULD weight capacity) uses `cw_k` (chargeable weight) but `W_u` is a physical payload limit — binds on actual mass. `cw_k = max(w_k, v_k·167)` is a billing construct; using it double-counts light-bulky cargo against both P.2 (volume) and P.3 (weight). Worked case LD3 (`W_u≈1588`, `V_u≈4.5`): dense 1400 kg/1 m³ + light 30 kg/3 m³ → vol 4 ✓, actual wt 1430 ✓ (loadable), but P.3-with-`cw` = 1400 + 501 = 1901 > 1588 → wrongly rejected. **Fix: P.3 → `Σ_k w_k · y ≤ W_u · z`.** Independent of PWL. P.2 correctly uses `v_k`.
- **13-B — minor.** P.5 sums `z` over contracts only; spot/NAC options (`uld_o = ⊥`) carry no `z`, not counted against `P_{f,u}`. P.2's comment claims P.5 "bounds all loads jointly" — overclaim. Fix the comment (or confirm spot = airline's capacity problem).
- P.2/P.4/P.6/P.7 sound; `y→z` linkage survives the bucket rewrite.

**Item 15 — P.17/P.18/P.19/P.21:**
- **15-A → RESOLVED: REMOVE P.18 ENTIRELY** (+ the `B_k` parameter). A hard per-shipment budget cap can make a *committed, must-serve* shipment infeasible — operationally wrong; an accepted shipment cannot be declined. The objective already minimizes cost — that is all valid cost control for a committed shipment. A soft budget penalty would be redundant with cost minimization. Budget is a **quoting-layer** concern (accept / price / product choice, upstream), not a routing constraint. Removing P.18 dissolves the cost-attribution problem entirely. **Corollary:** the model's "tenant-level run-total ceiling" escape hatch has the same flaw (whole accepted book must be served) — remove it, or relabel explicitly as a quoting-time advisory, not a solve constraint. **Cross-model:** ocean FCL `P.4 budget cap`, LCL, trucking carry the same defect — revisit on their rework (not this rewrite).
- **15-B — fix forced by item 3.** P.19's pre-solve lock-feasibility check lists P.15 deadline + P.20 SLA as hard-conflict sources. Item 3 makes those soft → a lock "violating" them yields a tardiness penalty, not infeasibility. Check keeps only genuinely hard sources: P.2–P.7 capacity, P.11 cutoff, absolute backstop `T_k^abs`.
- **15-C — housekeeping.** P.21 domain gains `τ_k ∈ ℝ≥0` (item 3) and `CW_{f,o}, BW_{f,o,b} ∈ ℝ≥0`, `γ_{f,o,b}, active_{f,o} ∈ {0,1}` (item 4). P.17 + pickup-window bound clean.
- **Renumbering:** removing P.18 → renumber P.19/P.20/P.21, or leave a "P.18 removed" gap. Rewrite decides.

**Item 18 — Open Items + Tractability:**
- **18-A.** Delete deferred item `sla-soft-otp` (item 3 promotes it to active). Migrate its sourced calibration anchor ("$50–$500 per shipment per day late, tiered") into the active item-3 text as the `CALIBRATION NEEDED` reference range. Reconcile `σ_k`-slack vs. `τ_k` notation to one form.
- **18-B.** Three Tractability items written against the pre-bucket formulation — refresh in the rewrite: `scale-y-aggregation` (premised on `|O_f|≈15–20` with TACT tiers as options — false under bucket, `|O_f|≈5`); `scale-option-dominance` (per-shipment `c_o(cw_k)` dominance — now per-bucket); `strat-v2-mawb-rescale` (**now obsolete** — assumes a future `h_{k,m}` MAWB restructure; the bucket formulation *is* that restructure, with no `h`). Re-derive the base instance scale estimate for the bucket formulation.
- Other Open Items: genuine deferrals, no issues.

---

## Review status

**FULL 19-ITEM REVIEW COMPLETE (Session 14, 2026-05-21).** Outcomes locked in the sections above.
- 1 OK · 2 LOCKED (single FX table, no per-run pinning) · 3 LOCKED (linear-soft tardiness, quadratic deferred, placeholder weights) · 4 RESOLVED (bucket formulation + C/E config toggle + A–E doc section) · 5 no-objection · 6 moot · 7 confirmed (two rate families, `rate_family` per-offer attribute).
- 8/10/11/12/14/16/17/19 rewritten by item 4 / post-rewrite — no standalone findings.
- 13 — 13-A real bug (P.3 `cw_k`→`w_k`), 13-B minor (P.5 comment).
- 15 — 15-A resolved by **removing P.18 entirely** (+ `B_k`); 15-B (P.19 pre-solve check); 15-C (P.21 domain).
- 18 — 18-A (delete `sla-soft-otp`), 18-B (3 stale Tractability items).

**Next: execute the combined rewrite.** No review items outstanding.

## Pending user inputs

1. **Item 3:** per-service-product tardiness weights `w_{sp(k)}` — accepted as `CALIBRATION NEEDED` placeholders for MVP; real values still to come. (Not blocking — the rewrite proceeds with placeholders.)

## Next-session plan — EXECUTE THE COMBINED REWRITE

Review is complete. The rewrite of `air_freight_routing.tex` bundles:
- **item 2** — currency: one cross-ref paragraph to `data_model.md §7` (single FX table, convert at solve, no per-run pinning).
- **item 3** — soft deadline + **linear** tardiness (`τ_k`, `+ Σ w·τ_k`); hard backstop `T_k^abs`; placeholder weights `CALIBRATION NEEDED` ($50–$500/shipment/day anchor from 18-A).
- **item 4** — bucket formulation (MAWB = `(flight-leg, supply-offer)` bucket); 4 scrutiny points (CW monotonicity remark, TACT binary×continuous disaggregation, empty-bucket handling, bucket-is-rating-construct note); A–E "alternatives considered" section (D fullest); **BSA cost per the "BSA cost modeling — converged design" section above** (3 rate families, `settlement_basis`; equalized → per-solve sunk allowance `A_c`, BSA as a 2-segment offer; hard-P.7 removal; controller = consumed-weight accumulator).
- **item 7** — two rate-function families (cumulative PWL / min-over-flat-breaks); `rate_family` per-offer catalog attribute.
- **item 13-A** — P.3 bug fix: `cw_k` → `w_k` (actual weight). 13-B: fix P.5 comment.
- **item 15** — **remove P.18 + `B_k` entirely**; remove/relabel the run-total-ceiling escape hatch; fix P.19 pre-solve check (drop soft P.15/P.20); extend P.21 domain; renumber P.19/P.20/P.21.
- **item 18** — delete deferred `sla-soft-otp`; refresh 3 stale Tractability items (`scale-y-aggregation`, `scale-option-dominance`, obsolete `strat-v2-mawb-rescale`); re-derive base scale estimate.
- **Finding S — Change 1 only.** P.20 soft is delivered by item 3; add the quantile hook (`t_k(d(k))` → TT-Service P85–P90 quantile, deferred) + "planning bound, not contractual guarantee" framing. **Changes 2 and 3 are deferred** — write them as new entries in the air model's §sec:deferred Open Items, not implemented.

Then: critique-pass the rewrite (Session-11-style 3-agent math review); rebuild the walking-skeleton MILP on the bucket formulation (E + C toggle).

**Open user input (not blocking):** real item-3 tardiness weights — rewrite proceeds with `CALIBRATION NEEDED` placeholders.

**Cross-model TODO (not this rewrite):** ocean FCL `P.4 budget cap` / LCL / trucking carry the same P.18 defect — revisit on their rework.
