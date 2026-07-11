# H₀ Human-Baseline Consolidation Policy — Design Spec

**Status: BUILT S43** (`H0` + `PIH` arms in `src/replay.py`; `tests/test_replay_loop.py`, 304
passing). Specifies the `H₀` arm of the 5-arm replan-savings decomposition
(`model/arrival_only_replan_methodology.md` §4). Open decisions are flagged inline as **[OPEN-n]**
and collected in §9.

## AS BUILT (S43) — deviations from the spec below, and two findings

The intent below is preserved (batch-at-cutoff; standalone routing with no joint optimization;
billing via the engine; H₀ visibly not the optimizer). Three implementation choices differ, and
two findings came out of the build:

- **Routing via per-HAWB singleton solve, not pure-Python path enumeration.** Each cutoff HAWB is
  routed by `solve([k])` — a *singleton* solve (one box ⇒ no consolidation decision possible, so
  the engine makes zero multi-shipment tradeoff). This gets the exact cheapest-standalone route
  with **no second billing code path to drift** from the MILP (the §4 concern), at the cost of one
  small solve per HAWB. The committed book is then billed by one all-pinned solve as specified.
- **Capacity contention handled by fallback-roll, not a Python capacity ledger (§3.5).** When the
  all-pinned billing solve returns `INFEASIBLE` (naive standalone routes over-subscribe a capped
  arc), the marginal batch shipments are **rolled to the fallback arc** (largest-cw first, until
  feasible — the MILP is the feasibility arbiter, so no ledger to drift). This is *harsher* than
  the spec's "roll to the next-cheapest FLIGHT": fallback ⇒ arrives `T^abs` (late) + ~\$40k. **[OPEN-7,
  needs user] roll-to-next-cheapest-flight is a deferred realism refinement** — it matters because
  the roll policy moves H₀'s cost and OTP at tight cells, which feeds L1.
- **Lives in `replay.py` as `_plan_cycle_h0`, not `src/components/h0_planner.py`.** Kept beside the
  other arm planners (`_plan_cycle`) — they are replay glue that needs `build_geo_air_graph`.

- **FINDING — L1 can be ≤ 0 (timing, not a bug).** H₀ batches at **cutoff** (D-A14, more info at
  commit); `M₁'` commits at **reveal** (D-A23, less info). So H₀'s late-commit timing edge can beat
  `M₁'`, making `L1 = C(H₀) − C(M₁') ≤ 0` at tight cells (measured: seed-2 κ=2 tier-coupled, L1 ≈
  −\$258, the mirror of the L2 ≈ +\$258 there). The methodology **anticipates this** (D-A15: the
  H₀-timing/arrival asymmetry are set L2-favorable and are explicitly **not** claimed as a bound),
  but §4 line ~59 still writes the full chain `C(H₀) ≥ C(M₀) ≥ …` as if guaranteed. **[OPEN-8, needs
  user] reconcile:** report L1 honestly (can be ≤0; planning value lives mostly in the within-cycle
  rung `C(M₀) − C(M₁')`), and/or add the **on-arrival H₀** variant (D-A14 upper bracket; commits at
  reveal like `M₁'` ⇒ `L1 ≥ 0` by construction) as a second baseline.
- **π_hind is a lower bound only within `mip_rel_gap` (0.5%).** Each arm's reported cost is a
  gap-stopped solve, so on loose cells π_hind can read ~0.1% above M₁. Exact regret-floor validation
  (`C(M₁) ≥ C(π_hind)` via a tight π_hind solve) is a Stage-3 concern; tests assert the bound within
  the relative gap.

## 0. What H₀ is (and the correction this spec encodes)

H₀ is the "spreadsheet planner" baseline that **anchors L1 = C(H₀) − C(M₁′)** (planning value).
The cost chain it tops:

```
C(H₀) ≥ C(M₀) ≥ C(M₁′) ≥ C(M₁) ≥ C(π_hind)
```

**The correction:** a competent forwarder *always consolidates by hand*. They batch HAWBs,
fill ULDs, and push a consolidation's chargeable weight up to the next IATA break-weight (MFB)
when the higher floor at the cheaper rate bills lower. H₀ does **all** of that. H₀ is **not**
"ship each HAWB standalone" — that would make L1 a consolidation-vs-no-consolidation artifact
(over-stated and attackable), not a planning-myopia number.

H₀'s suboptimality vs M₀/M₁′ is **purely myopia** (§5): no cross-gateway joint optimization, no
reshuffle of priors, no lookahead to future arrivals, rule-of-thumb tie-breaks. The forwarder is
good at *local* consolidation arithmetic and bad at *global/temporal* coordination. That gap is
the entire content of L1.

Because M₀ is also myopic-but-MILP (greedy incremental, priors pinned), the **automation rung**
`C(H₀) − C(M₀)` isolates "human rule-of-thumb vs greedy optimizer on the *same myopic information
set*," and the **within-cycle rung** `C(M₀) − C(M₁′)` isolates joint optimization. H₀ must be a
*defensible* human for that split to mean anything.

## 1. Cadence — batch-at-cutoff (confirms D-A14)

H₀ acts **only at cutoff events**, never on every reveal. Per D-A14, batch-at-cutoff H₀ is the
**headline** baseline (an on-arrival H₀ is an upper-bracket sensitivity variant only — see §9
[OPEN-5]). Concretely:

- The replay cadence (`replay._event_times`) is the sorted union of reveals (`known_at`) and
  cutoffs (`tender_at`). The MILP arms re-plan the open book on **every** event.
- **H₀ ignores reveal-only events.** At a reveal, H₀ does nothing (the spreadsheet planner does
  not re-touch the book just because a HAWB landed in the inbox — they wait for the build-up).
- At each **cutoff event** at clock `t`, H₀ runs one **batch pass** over all HAWBs that (a) are
  visible (`known_at ≤ t`), (b) are not yet placed/tendered, and (c) are at or past their own
  cutoff (`tender_at ≤ t`). It places them, then tenders them (they are at cutoff by definition).
- A placed-and-tendered HAWB is **frozen forever** (no reshuffle — that is M₁'s job). H₀ never
  revisits a prior decision.

**Consequence (the myopia mechanism):** because H₀ only acts at a HAWB's own cutoff and freezes
immediately, every batch sees only HAWBs that *must* go now, plus whatever already-committed mass
shares their gateway/lane. It never co-optimizes a HAWB against a later-cutoff HAWB that would
have made a fuller, cheaper consolidation. That is the temporal half of L1.

**Edge case — co-cutoff HAWBs.** Multiple HAWBs sharing a cutoff time *are* in the same batch and
*are* consolidated together (§3) — the human builds one consolidation at the build-up bench from
everything on the dock at that cutoff. So H₀ is not myopic *within* a cutoff batch; it is myopic
*across* cutoffs and *across* gateways.

## 2. Gateway / routing choice per HAWB — cheapest-standalone, no cross-gateway joint opt

For each HAWB in the batch, H₀ picks **one** origin gateway and **one** O→D routing by a
deterministic rule-of-thumb, with **deliberately no cross-gateway or cross-HAWB joint
optimization**.

**Routing rule (recommended default — [OPEN-1]):** **cheapest-standalone feasible route.**
For each HAWB `k`, enumerate the simple O→D paths in its pre-filtered subgraph `A_k`
(`air_graph.subgraphs[k]`, fallback excluded for now), cost each path as if the HAWB shipped
**alone** on it (standalone chargeable weight `cw_k = max(w_k, v_k·167)`, dunnage-uplifted on the
actual-weight side per `_chargeable_weight` / `air_leg_cost_ub` arithmetic), and pick the cheapest.
Ties broken deterministically (§3.4).

- This is exactly the "I know my lane rates, I pick the cheapest way to fly it" heuristic.
- It is **myopic** because the standalone cost ignores who *else* H₀ will put on that arc — the
  human picks the gateway/route first, *then* consolidates whoever else is going the same way. M₀
  by contrast lets the greedy MILP weigh the newcomer against everyone already placed.
- **No cross-gateway pooling decision.** If HAWB `k₁` would route TPE→LAX and `k₂` would route
  TPE→ICN→LAX, H₀ does not consider rerouting `k₁` through ICN to share a fuller MAWB with `k₂`.
  Each HAWB's gateway/route is chosen on its own standalone merits. (The MILP arms *can* make that
  trade; H₀ structurally cannot — this is the gateway half of L1.)

**Important guardrail (memory `feedback_no_standalone_cost_pruning`):** cheapest-standalone is the
*selection* rule for the human's chosen route, **not** a pruning rule that discards options. H₀
still keeps the fallback available (§6); it never strands a routable HAWB because a route looked
expensive standalone. Under tight κ the cheapest-standalone route may already be capacity-full
(§3.5) — H₀ then steps to its next-cheapest standalone route, never to "give up."

**Region→region (D-A24).** When a HAWB carries `origin_candidates` / `dest_candidates`, the
"gateway choice" *is* the candidate-airport choice; cheapest-standalone naturally ranges over the
per-candidate trucking + air arcs in `A_k`, so no special handling is needed — H₀ just costs each
candidate's standalone O→D path and takes the min.

## 3. Consolidation rule — the core of H₀

This is what makes H₀ a *human forwarder* and not a standalone shipper. After §2 has assigned each
batch HAWB a chosen route, H₀ **groups HAWBs that share a MAWB-eligible air arc and a compatible
consolidation group, fills/packs them, and chases the break-weight.**

### 3.1 Who may combine — `group_key` compatibility

Two HAWBs `k₁, k₂` ride a **shared MAWB** on a MAWB-eligible air arc `a` iff **all** hold:

1. **Same chosen air arc `a`** (from §2), and `a` is MAWB-eligible
   (`arc.carries_mawb`, i.e. `rate_family ∈ {FLAT_RATE, MIN_FLAT_BREAKS, PER_ULD_PIVOT}`).
   Co-load (`COLOAD_PER_KG`) arcs carry **no MAWB object** — co-load HAWBs bill per-kg
   individually (§4), so there is nothing to "group" there beyond riding the same arc.
2. **Same consolidation group** `group_key(k₁) == group_key(k₂)`
   (`air_graph.group_key`, line 1396). This already encodes the human's compatibility rules:
   - `CONSOLIDABLE_CLASSES = {GEN, DGR, PER}` group as `"{cargo_type}:{temperature}"` — so
     **temperature segregates** (ambient/chilled/frozen/pharma never co-consolidate) and **class
     segregates** (GEN/DGR/PER never mix). DGR is coarse (one DGR group) at this layer.
   - `VAL / HUM / AVI` → singleton (`"{cargo_type}:{id}"`): valuables/human-remains/live-animals
     **never consolidate**. They ride their own MAWB.
3. `(a, g)` exists as a MAWB candidate in `air_graph.mawbs` (it will, by construction, since both
   HAWBs are riders of `a` in group `g` — `overlay_mawbs`, line 1433).

This is identical to the partition the MILP optimizes over, so H₀'s consolidation lives in the
**same option space** as M₀/M₁′/M₁ — load-bearing for the §6 chain guarantee.

### 3.2 ULD-fill / pack logic

Within one `(a, g)` candidate, H₀ packs HAWBs onto MAWBs/ULDs greedily:

- **Density-mixing is automatic and free.** The realized chargeable weight of a consolidation is
  `CW = max((1+ε)·Σ w_k, Σ v_k·167)` (`_routed_cw`, C.4). H₀ does *not* model 3-D packing; it just
  pools the cargo and lets `max(Wt, Wv)` fall out — which is exactly the forwarder's "dense cargo
  soaks up light cargo's volumetric slack" intuition and exactly what the billing recompute does.
- **`per_uld_pivot` / BSA arcs.** Fill **integer ULDs**: assign HAWBs to ULD positions
  respecting per-ULD physical caps `W_u` / `V_u` (`UldType`, C.5b) and the contract allotment
  `N_{a,u}` (C.5). The human's rule: **fill each ULD before opening the next** (first-fit-decreasing
  by `cw_k`), open the minimum number of ULDs that holds the group's cargo. Per-flight settlement
  bills `r_a·max(CW, π·Ση)` (pivot floor); equalized bills `r_c·max(0, ΣCW − A_c)` across the
  contract pool. **[OPEN-2]:** does the hand-planner pivot-optimize (open one *extra* ULD to lift
  Ση·π above CW so the pivot floor "pre-pays" for headroom)? Recommended **no** — that is a
  non-obvious optimizer move, not a spreadsheet move; H₀ opens the *fewest* ULDs that fit. Leaving
  the pivot-pre-pay trade to the MILP is a legitimate source of L1.
- **`flat_rate` / `min_flat_breaks` arcs.** No ULD integrality — a single MAWB object holds the
  whole group's pooled `CW`. H₀ puts the entire compatible group on **one** MAWB per `(a, g)`
  (one consolidation per arc per group — the human does not split a group across two MAWBs on the
  same flight without reason). Splitting to dodge a cap is handled in §3.5.

### 3.3 The break-weight chase (MFB) — the obvious human move

On a `min_flat_breaks` arc, the realized cost is `min_b rate_b · max(CW, break_b)`
(`_validate_billing`, line 1274 — **H₀ reuses this exact closed form**). The human's move:

> If bumping the billed weight up to the next break floor `break_b` lets me bill at the cheaper
> rate `rate_b` for a lower *total*, do it.

This is **already the cost-minimizing closed form** — `min_b` over breaks. So H₀'s "chase" is not
a separate heuristic; it is **evaluating the same `min_b rate_b·max(CW, break_b)` the billing code
evaluates** and recording that the consolidation bills at the round-up break. Two consequences:

- H₀ gets the break-weight economics **exactly right for a fixed consolidation** — it is the
  *which-HAWBs-to-pool* and *which-arc* decisions that H₀ does myopically, not the per-MAWB
  rating arithmetic. Good: it keeps H₀ a competent hand-planner, so L1 is myopia, not arithmetic.
- **[OPEN-3] — aggressive vs passive chase.** A truly aggressive planner would *pull a
  later-cutoff HAWB forward* into this batch to reach a break floor with real cargo (rather than
  pay for dead-weight round-up). That is **lookahead** and H₀ does not do it (§5). Recommended:
  H₀ chases the break **only with cargo already in the batch** — it computes `min_b` over breaks
  for the consolidation it has, and never reaches across cutoffs to fill a break. This keeps the
  "fill the break by pulling future demand forward" value firmly inside L2/M₁. **Confirm.**

### 3.4 Deterministic tie-breaks and ordering

Everything must be reproducible (two runs byte-identical, like the MILP arms). Fix the order:

1. **Batch HAWB processing order:** ascending `(tender_at, hawb_id)`.
2. **Route enumeration per HAWB:** paths emitted in sorted-arc order; cheapest standalone cost
   wins; ties broken by lexicographically smallest ordered arc-id tuple (`order_route` then string
   compare).
3. **ULD first-fit:** HAWBs sorted descending by `cw_k`, then ascending `hawb_id`; ULD types in
   sorted code order; positions filled low-index first.
4. **Break selection:** `min_b` over breaks; on a cost tie pick the **lower** break floor
   (smaller `threshold`) — deterministic and matches "don't pay for headroom you don't need."

### 3.5 Capacity awareness (the human reads the allotment board)

H₀ is **myopic but not capacity-blind** — a real planner knows when a contracted flight/ULD block
is full and bumps cargo to the next option. H₀ respects the live ledger (`ReplayState.free(arc)`):

- When the chosen `(a, g)` consolidation would exceed a `per_uld_pivot` allotment
  (`free` ULD positions on `"{offer}:{uld}"`) or a `spot_cap` chargeable-weight ceiling, H₀ fills
  to the cap, then **spills the remainder to that HAWB's next-cheapest standalone route** (§2),
  re-running §3 on the spilled HAWBs. Iterate until placed or exhausted → fallback (§6).
- This mirrors how the MILP sees capacity (`_build_spot_cap` C.5d, `_build_c5_allotment` C.5), so
  H₀'s feasibility coincides with the MILP feasible set — again load-bearing for §6.
- **[OPEN-4] — spill granularity.** Does the human split a single consolidation across two arcs
  when one is partly full, or move whole HAWBs? Recommended: **move whole HAWBs** (the human
  doesn't shave a HAWB across two flights); spill the marginal HAWB(s) by ascending `cw_k` so the
  fullest cheap consolidation is preserved. Confirm.

## 4. Cost computation — reuse the MILP billing closed-forms verbatim

H₀ must be costed on the **same accounting basis** as the MILP arms or L1 is a cross-engine
artifact. **It is, and for free**, because of how scoring already works:

`score_run` (`replay.py:597`) costs *any* arm by taking the arm's committed routes
(`result.tendered_routes`), pinning every `(hawb_id, arc_id)` (`_route_pins`), and calling
`solve(ag, resolved, rates, MilpParams(), pinned=pins).total_cost` — **a single all-pinned solve
with no free decision variables**. That degenerate solve does nothing but **bill the fixed
routing** through the exact same objective + `_validate_billing` / `_validate_bsa` closed forms the
optimizing arms use:

- co-load / per-kg: `Σ m_a^cl · cw_k · x` (`_set_objective`)
- `flat_rate`: `max(min_chg, m·CW)` via `c_flat` (`_validate_billing:1270`)
- `min_flat_breaks`: `min_b rate_b·max(CW, break_b)` (`_validate_billing:1274`) — the break chase
- `per_uld_pivot`: `r_a·max(CW, π·Ση)` (`_validate_bsa:1204`)
- equalized BSA: `r_c·max(0, ΣCW − A_c)` (`_validate_bsa:1212`)
- MAWB fixed charge `c^MAWB_fix·z`, ground/dwell/fallback `Σ cost_a·x`, Path-A/B surcharges.

**So H₀ does not compute cost itself.** H₀ produces a feasible `tendered_routes` dict
(per-HAWB ordered arc list + the implied MAWB/ULD consolidation), and `score_run` bills it. There
is **zero** separate H₀ cost code, hence zero risk of a cross-engine accounting drift — the
headline-clean property the methodology insists on for L2 carries to L1 by the same mechanism.

**One subtlety to confirm in implementation:** the all-pinned billing solve must reproduce H₀'s
*intended consolidation*, not a cheaper re-grouping. It does, because `group_key` is a deterministic
partition: once each HAWB's air arc is pinned, the `(a, g)` MAWB memberships are fully determined
(every rider of `a` in group `g` is on that MAWB), CW is `max(Wt, Wv)` of exactly that set, and the
MILP's only remaining freedom (break selection, ULD count, pivot) is the **same `min`/`max` the
human computed by hand** in §3.3. So the billed consolidation == H₀'s planned consolidation.
**Edge to verify:** `per_uld_pivot` ULD *count* `η` is a free integer the billing solve minimizes;
if H₀'s hand-pack opened more ULDs than the cost-minimal count, the billing solve would bill the
cheaper count and under-state H₀ (good direction for the chain — see §6, but flag for the test).

## 5. Where H₀ is deliberately myopic — the source of L1

Enumerated exactly. H₀ does **NOT**:

1. **Jointly optimize across HAWBs within a cutoff batch's gateway/route choice.** It picks each
   HAWB's route cheapest-*standalone* (§2), then consolidates whoever collided. M₀ lets the greedy
   MILP weigh each newcomer against everyone already placed; M₁′ jointly optimizes the whole batch.
2. **Optimize across gateways.** No "reroute `k₁` through ICN to fill `k₂`'s MAWB" (§2). The
   gateway/route is fixed per-HAWB before consolidation.
3. **Reshuffle priors.** Once placed+tendered, frozen forever. M₁ relaxes prior pins (open book);
   that recourse is **L2**, structurally absent from H₀.
4. **Look ahead to future arrivals.** No pulling a later-cutoff HAWB forward to fill a ULD or a
   break floor (§3.3 [OPEN-3]); no holding cargo for an anticipated fuller consolidation. H₀ sees
   only the current cutoff batch + already-committed mass.
5. **Re-optimize per newcomer.** Within a batch, H₀'s consolidation is one greedy first-fit pass
   (§3.2), not a re-solve as each HAWB is added (that *is* M₀'s loop, `_plan_cycle` lines 356–363).

What H₀ **does** do well (so L1 is myopia, not incompetence): per-MAWB break-weight rating (§3.3),
density mixing (§3.2), within-batch / within-gateway consolidation (§3.1), integer-ULD fill (§3.2),
capacity-aware spill (§3.5). These are the hand-arithmetic a competent forwarder nails.

**L1 anatomy this produces:**
`C(H₀) − C(M₀)` = item 1's *greedy-vs-rule-of-thumb* gap on the same myopic info + item 5.
`C(M₀) − C(M₁′)` = within-cycle joint optimization (items 1–2 fully).
`C(M₁′) − C(M₁)` = L2 (items 3–4: reshuffle + open-book recourse, **not** in H₀ at all).

## 6. Feasibility + chain guarantee `C(H₀) ≥ C(M₁′)`

**Feasibility — H₀ always produces a feasible plan.** Every HAWB has a fallback arc
(`fallback_arc`, always in `A_k`, exempt from all predicates), so:

- A HAWB H₀ cannot place on any real route (all routes capacity-full §3.5, or empty real subgraph)
  is routed to its **fallback arc** → arrives at `T^abs` (late by definition, counted as a miss in
  OTP, billed at the fallback dollar cost). This is the **same** `FallbackPolicy` escape the MILP
  uses. H₀ never produces an infeasible/empty assignment.
- So H₀'s plan is always a member of the routing feasible set over the per-HAWB subgraphs.

**Chain `C(H₀) ≥ C(M₁′)` holds by construction.** M₁′ is the **optimum** (min-cost) of the
single-pass pinned feasible set — the set of all feasible route assignments where tendered priors
are pinned. H₀'s plan is a **feasible member of that same set** (same subgraphs, same `group_key`
partition, same capacity constraints, same billing). Therefore `C(M₁′) ≤ C(H₀)`. ∎

This requires three things, all satisfied by design:
1. H₀ routes only over arcs in `A_k` (§2 enumerates `air_graph.subgraphs[k]`). ✔
2. H₀'s consolidation respects `group_key` + capacity (§3), the same constraints M₁′ obeys. ✔
3. H₀ is **billed by the same solve** (§4), so the costs are comparable point-for-point. ✔

**Edge cases where the inequality could *appear* violated — and why it doesn't / how to guard:**

- **Time-limited / gap-stopped M₁′ incumbent (BLK-1, methodology §4 note).** M₁′ is solved to
  `mip_rel_gap=0.005`, not proven-exact. A within-gap M₁′ incumbent could transiently read up to
  0.5% above true optimum, and *in principle* nip above a lucky H₀. This is the **already-accepted**
  per-draw chain caveat in the methodology (track `status`/`mip_gap`); it is not specific to H₀.
  Guard: report H₀'s `mip_gap=0` (its billing solve is degenerate/instant, always exact) and flag
  any cell where `C(H₀) < C(M₁′)` for inspection — almost certainly an M₁′ gap artifact, not an H₀
  bug.
- **H₀ opens more ULDs than cost-minimal (§4 subtlety).** The billing solve minimizes `η`, so it
  bills H₀'s consolidation at the *cheaper* count — this can only **lower** `C(H₀)`, never raise it,
  so it cannot break `C(H₀) ≥ C(M₁′)` (it tightens it). But it means H₀'s *billed* cost slightly
  understates a literal hand-pack; acceptable and conservative for L1. Flag in the test plan.
- **Fallback over-pricing.** `FallbackPolicy` sizes fallback to dominate any real route × margin,
  so H₀ taking a fallback is genuinely expensive — correct, it should be (the human stranding a
  HAWB *is* costly). No violation.

**Net:** `C(H₀) ≥ C(M₁′)` is structural modulo the pre-existing M₁′ solver-gap caveat. No
H₀-specific way to violate it.

## 7. Implementation shape — standalone rule-planner (recommended)

**Recommendation: (a) a standalone rule-planner that does NOT call `solve()` for *planning*.**
H₀ computes its route assignment by the §2–§3 procedure in plain Python; `solve()` is invoked
**only** by `score_run` afterward to *bill* the frozen plan (§4). Justification:

- **Visibility.** The user's requirement is that H₀ visibly is *not* "the optimizer with pins."
  Option (b) — `solve()` with a degenerate objective — would make H₀ a MILP call and blur exactly
  the H₀-vs-M₀ distinction L1 measures. A standalone rule-planner is auditable line-by-line as a
  human procedure.
- **No accounting drift.** Because billing is still the all-pinned `solve()` (§4), the standalone
  planner gets the same cost basis without *being* the optimizer. Best of both.
- **Determinism.** Pure Python first-fit with the §3.4 tie-breaks is trivially reproducible; no
  dependence on HiGHS incumbent selection.

### 7.1 Where it slots into `run_replay`

`run_replay` (`replay.py:372`) dispatches on `arm ∈ {M0, M1p, M1}` via `_plan_cycle`
(`replay.py:333`). Add `"H0"` to `_ARMS` and a branch in `_plan_cycle` (or a sibling
`_plan_cycle_h0`) that:

- **On a reveal-only cycle:** returns the prior plan unchanged (no re-plan). Detect "is this a
  cutoff cycle" by `any(tender_at[i] <= t for i in newly-eligible)`; if none, skip planning.
- **On a cutoff cycle:** run the §1–§3 batch procedure over the eligible batch, write
  `placed_routes[k]` for each placed HAWB (as `order_route(...)`, same as the other arms), leave
  priors frozen (H₀ never reshuffles, so just carry `placed_routes` forward untouched).
- Returns `(ag, resolved, sol)` in the same shape the other arms return. **But `sol` here is not
  an optimizer result** — it is a lightweight `AirSolution`-shaped object carrying H₀'s
  `routes`/`active_mawbs` so `_record_snapshot` and `_reconcile_cycle` work unchanged. Two options:
  - **(7.1a)** Build a real `AirSolution` by hand (fill `routes`, `active_mawbs`,
    `mawb_chargeable_weight` from `_routed_cw`, `mawb_uld_counts` from the hand-pack). The ledger
    reconcile (`_solve_claims`) then works verbatim. `total_cost` can be left 0 / NULL on the
    *snapshot* (the authoritative cost is `score_run`'s billing solve). **Recommended** — keeps
    the snapshot/ledger plumbing identical.
  - **(7.1b)** Run a single all-pinned `solve()` *per cutsycle* purely to populate the snapshot
    `AirSolution` (route fixed = H₀'s, so it only bills). Simpler code, but reintroduces a per-cycle
    solve. Use only if (7.1a)'s hand-built `AirSolution` proves error-prone.

### 7.2 Function signature sketch

```python
# replay.py, sibling to _plan_cycle
def _plan_cycle_h0(
    sim, rates, params, build_geo_air_graph,
    visible_ids, hawb_by_id, placed_routes, tender_at, t,
    ledger: ReplayState,
) -> tuple[AirGraph, list[Hawb], AirSolution]:
    """H₀ batch-at-cutoff rule-planner (no MILP solve). Places the cutoff batch by
    cheapest-standalone route (§2) + group_key consolidation + MFB break chase (§3),
    respecting the live ledger (§3.5); freezes placements; fallback for the unplaceable
    (§6). Returns an AirSolution-shaped record (routes/active_mawbs/CW/ULD-counts) so the
    existing _record_snapshot / _reconcile_cycle plumbing is unchanged. Cost is NOT
    computed here — score_run bills the frozen plan via the all-pinned solve (§4)."""
```

A dedicated rule-planner module is cleaner than burying this in `replay.py`. **[OPEN-6]:**
suggested `src/components/h0_planner.py` exposing
`plan_batch(air_graph, hawbs, rates, free_cap) -> dict[HawbId, list[ArcId]]` (pure, testable in
isolation per the component-isolation gate), with `_plan_cycle_h0` as the thin replay adapter that
wires the ledger and builds the `AirSolution`. Confirm the module name/location.

### 7.3 Scoring

**Unchanged.** `score_run` (`replay.py:597`) scores H₀ exactly as it scores the MILP arms:
deterministic arrival walk (`route_reliability`), OTP vs frozen `booking_promise`, and realized
cost = the all-pinned billing `solve()`. No H₀-specific scoring path. The `metrics.l1` field is
filled at the **sweep level** (Stage 3) by differencing `C(H₀) − C(M₁′)` across the scored arms,
exactly as `l2_*` are.

### 7.4 Isolation tests (component gate)

Per `CLAUDE.md` the planner doesn't reach integration until isolation tests pass. Minimum set:
- **Happy path:** small fixture, two GEN/ambient HAWBs same arc → one shared MAWB, CW = density
  mix, billed cost = `min_b` break = hand-computed value.
- **Segregation:** one PER/frozen + one PER/chilled same arc → two MAWBs (different `group_key`).
- **Break chase:** a HAWB just under a break floor → billed at the round-up break, asserted equal
  to `min_b rate_b·max(CW, break_b)`.
- **Capacity spill:** allotment full → marginal HAWB spills to next-cheapest route, then fallback.
- **Chain witness:** on the same fixture, `C(H₀) ≥ C(M₁′)` (run both arms, assert).
- **Determinism:** two runs byte-identical routes.
- **Myopia witness (the L1 point):** a fixture where pulling a later-cutoff HAWB forward would fill
  a break/ULD → assert H₀ does *not* do it and `C(H₀) > C(M₁)` strictly (so L1 + L2 > 0).

## 8. One-paragraph summary

H₀ is a **batch-at-cutoff rule-planner** (D-A14 headline): at each cutoff it takes the must-go
batch, routes each HAWB by **cheapest-standalone** route over `A_k` (no cross-gateway/cross-HAWB
joint opt), **consolidates** compatible HAWBs onto shared MAWBs/ULDs by `group_key` (temperature +
class segregation, VAL/HUM/AVI singleton), **density-mixes** and **fills integer ULDs** greedily,
**chases the MFB break** via the exact `min_b rate_b·max(CW, break_b)` closed form, respects the
**live capacity ledger** (spill to next route, else fallback), and **freezes** every placement.
It is deliberately myopic: no reshuffle, no lookahead, no cross-gateway/within-batch joint
optimization, no per-newcomer re-solve — that myopia *is* L1. It is implemented as a **standalone
Python rule-planner** (not `solve()`-with-pins), and **billed by the same all-pinned `solve()`**
`score_run` already uses, so its cost is on the identical accounting basis as the MILP arms and the
chain `C(H₀) ≥ C(M₁′)` holds by feasible-set membership.

## 9. Open decisions for the user

| # | Decision | Recommendation |
|---|---|---|
| OPEN-1 | Gateway/routing rule: cheapest-standalone vs nearest/default gateway vs cheapest-on-dominant-flow | **Cheapest-standalone feasible route** — most defensible "competent human," ranges over D-A24 candidates naturally. Confirm. |
| OPEN-2 | `per_uld_pivot` pivot pre-pay: open an extra ULD to lift the pivot floor? | **No** — fewest ULDs that fit; pivot-pre-pay is an optimizer move, leave to MILP (→ L1). |
| OPEN-3 | Break-weight chase aggressiveness: chase with current-batch cargo only, or pull future demand forward? | **Current-batch only** — pulling future demand forward is lookahead → keep in L2/M₁. |
| OPEN-4 | Capacity spill granularity: split a consolidation across arcs, or move whole HAWBs? | **Whole HAWBs**, spill marginal by ascending `cw_k`. |
| OPEN-5 | On-arrival H₀ variant (D-A14 upper bracket) — build now or defer? | **Defer** — batch-at-cutoff is the headline; on-arrival is a later sensitivity arm. |
| OPEN-6 | Module location/name for the rule-planner | `src/components/h0_planner.py` with pure `plan_batch(...)`; `_plan_cycle_h0` adapter in `replay.py`. Confirm. |

### Cross-references
- Methodology arms / chain / L1-L2: `model/arrival_only_replan_methodology.md` §4, D-A14.
- Rate families + billing closed-forms: `src/components/air_milp.py` — `FlatRate`/`Break`/`BsaContract` (82–137), `_set_objective`, `_validate_billing` (1239), `_validate_bsa` (1168), `_routed_cw` (1219).
- `group_key` + MAWB overlay: `src/components/air_graph.py:1396` (`group_key`), `:1433` (`overlay_mawbs`), `CONSOLIDABLE_CLASSES:1393`.
- Fallback: `air_graph.fallback_arc` / `FallbackPolicy` (`:772`–`:872`).
- Replay integration: `src/replay.py` — `_plan_cycle:333`, `run_replay:372`, `_record_snapshot:511`, `_reconcile_cycle:498`, `score_run:597`, ledger `ReplayState` (`:48`).
