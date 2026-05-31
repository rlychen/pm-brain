# Omitted-findings index — Pass A residual

Produced 2026-05-28 after Pass A LaTeX edits landed. Every finding the three
critique agents returned that was **NOT** executed in Pass A is listed below,
grouped by source agent and severity, with the agent's original numbering
preserved. User triages: which become Pass B (act-now LaTeX edits), Pass C
(design notes outside LaTeX), Pass D (defer), or drop.

**What Pass A executed (excluded from this index):**

- Correctness BUG-1 (`min_flat_breaks` big-M) + C.14 BW domain widening
- Correctness BUG-2 (equalized BSA cost ambiguity → cost=0 reading)
- Correctness BUG-3 (forward-time-window per-outbound admission rewrite)
- Correctness TIGHTEN-1 (`η ≤ N · z` link as C.5-act)
- Formulation F15.2 ($C^{\text{fallback}}$ sizing formula in §sec:hawb-params)
- Formulation F12.1 (3 new walking-skeleton metrics: fractional-$x$ at LP root, per-arc activated-$z$ distribution, fractional fallback usage)
- Formulation F6.2 (P-quantile naive-propagation warning added to deferred TT-Service item)
- Notation rows: `POD_k`, `Δ^post_k`, `T_k^dead`, `ETD_f`, `ETA_f`, `ac(f)`, per-flight capability predicates (`dgr`, `per`, `val_capable`, `avi_capable`, `hum_capable`), `carrier^op` / `carrier^mk`, `transit(k, a^fb_k)`
- Notation renames: `C^{pu} → A^{pu}` (17 sites); `g` (MIP gap) → `g^{mip}` in §sec:carrier-policy
- Removed "Renamed from $D_k$" historical parenthetical

---

## A. Correctness + notation agent (`06-correctness-notation.md`) — residual

### Tightening / safety

- **TIGHTEN-2** — `t_k^{D_k^node}` lower bound should be `min_{a ∈ A^last_k} arr_dest(k, a)` not `t_k^{rdy,early}`. Pure LP tightening, no integer change. [Cheap LaTeX edit.]
- **NOTATION-18** — same as TIGHTEN-2 from documentation angle. Same fix resolves both.
- **SAFE-TO-DEFER-2** — `t^hi` propagation through all-ground subgraphs accumulates cargo-ready width (window collapse never fires). Edge case worth a one-line acknowledgment.
- **SAFE-TO-DEFER-3** — pruning-safety argument in §sec:fwd-time-propagation relies on `C^fallback` dominating `W_k (T^abs - Δ_k)^2`. Pass A added the per-tenant sizing formula but did NOT rewrite the pruning-safety prose to reflect the per-HAWB scaling. If the formula is per-HAWB, the safety argument needs to be too.
- **SAFE-TO-DEFER-4** — flow-conservation guarantees exactly one terminal arc activates; the C.10a sum reduces to a single term. One-line note would help readers.

### Notation cleanup (additional)

- **NOTATION-10** — lift `chargeable(c)` to a §sec:variables row (currently defined ad-hoc inside C.13a).
- **NOTATION-11** — move `K^fb` out of the sets-and-indices table to the Post-solve derived block (already exists in §sec:variables); avoids the implication it's an MILP set.
- **NOTATION-13** — drop `G` from the sets table; only `G_a` is ever iterated.
- **NOTATION-14** — `base_freight_k` under consolidation is an approximation, not literal per-HAWB cost share. One-line note in §sec:surcharge Path A.
- **NOTATION-15** — C.12 has no equation; either remove from "active families" list or annotate "placeholder — handled outside the MILP."
- **NOTATION-16** — pivot named "per-flight" but `pivot_{a,g}` is per-MAWB-arc, not per-leg. **Decision needed:** does the formulation actually want one pivot per MAWB-arc (current) or per leg (multi-leg interline arcs)? Affects multi-leg BSA take-or-pay accounting.
- **NOTATION-17** — indicator-function font inconsistency (`\mathbbm{1}` vs `1` in binary RHS). Pick one.
- **NOTATION-19** — `Π` (ULD interchange triples) is graph-build-only; annotate or move to a "Graph-generator inputs (not MILP)" sub-block.
- **NOTATION-21** — Eq.~\ref{eq:flight-uld-surcharge} double-iterates over `a` (outer + inner bind the same index). Clarity fix, math is correct.
- **NOTATION-22** — `T_k^abs` "mandatory-finite" needs a defensive ingestion guard; if a HAWB enters with `T_k^abs` unset, `arr_dest(k, a^fb_k)` and the C.14 `τ_k` bound are undefined. Add data-contract note.
- **NOTATION-23** — `ε` (dunnage) vs `ε^pref` (cost-ceiling slack). Acceptable with superscript; optional rename to `ε^dun` for total disambiguation.
- **NOTATION-24** — `μ_k` (value coefficient, dimensionless) vs `μ_a` (transit time, hours) share glyph. Heavier rename; defer unless skim-hostility is a concern.
- **NOTATION-26** — active-constraint family enumeration appears in abstract, §sec:constraints opening, and §sec:objective monotonicity paragraph; cross-check they all match.

---

## B. Real-world considerations agent (`07-real-world-considerations.md`) — residual

**Severity ladder:** CRITICAL = pilot-killer, BLOCKING = wrong answer in common case, MATERIAL = >10% of shipments, EDGE = real but rare.

### CRITICAL (7)

- **1.1** Truck has to leave warehouse N hours before DCO, not at DCO. Most common cause of cutoff miss; model has no pickup-window-from-DCO parameter. [Earlier framed as Pass B; not executed.]
- **2.2** PI370 Class 3 flammable + PI965 lithium on same MAWB. Model's coarse DGR group consolidates them; airline acceptance fails at ramp; fallback arc silently masks the operational infeasibility.
- **3.1** Lithium acceptance matrix is per-carrier-per-flight-per-PI-per-Section-per-ac_type. Without LLM-assisted ingestion the tenant runs the model with stale matrix → systematically wrong routings.
- **3.2** Embargo records — manual curation is operationally fragile; updates weekly minimum.
- **3.3** BSA contract attributes ($N_{a,u}$, $\pi_a$, $A_c$) are PDF + Excel artifacts; tenant ops can't extract without dedicated parsing tool.
- **4.1** Fallback usage reported per-solve, not as a trend. Operator sees 2→5→9 fallback HAWBs over three days with no alarm; SLA breach by Friday.
- **5.1** Per-HAWB cost attribution promoted from deferred to MVP. MILP returns per-MAWB cost only; quote desk, customer billing, KAM QBR all blocked without per-HAWB. [Earlier framed as Pass B; not executed.]

### BLOCKING (9)

- **1.2** CFS build crew labor + dock door capacity. No resource constraint on concurrent builds at same CFS per time window.
- **1.5** Booking-confirmation as state machine, not transaction. RQ → KK → KK-conditional → NS transitions, equipment swap downgrades post-accept; model has no soft-commit representation.
- **1.6** Cargo-readiness slip arriving as WhatsApp at 15:00 mid-batch. No event for "cargo-ready-window updated" between solves; messaging-agent has no schema for in-flight HAWB attribute change.
- **2.1** PI966 express walkthrough surfaces multiple gaps: lithium matrix `ac_type` keying ingestion contract, per-hub-handler GDP attribute (CV PharmaHub at LUX, not HKG), shipper-reliability prior on $t_k^{rdy,early}$, soft-reserve vs firm-book, push-back loop the model can't initiate.
- **2.4** Equalized-allowance allocation priority. Optimizer doesn't bias toward filling allowance with highest-margin HAWBs first.
- **4.2** Fallback selection has no root-cause attribution. Operator must mentally replay 8 pre-filter predicates + capacity + cutoff to understand rescue.
- **4.3** No counterfactual sensitivity. Operator can't see "if you relax X, fallback cost drops to Y."
- **5.3** Per-HAWB tardiness in hours not first-class diagnostic. Customer-comms workflow blocked.

### MATERIAL (14)

- **1.3** ULD build sequence (heavy-first, fragile-last); intra-MAWB cargo arrival timing affects build start.
- **1.4** FWB/FHL document race. Cargo + MAWB at ramp + missing FHL = no tender; FHL prep is per-MAWB labor correlated with HAWB count.
- **1.7** Mixed-PER segregation (pharma vs floral within "chilled" temperature band). Coarse `g(k)` for PER.
- **1.8** Per-flight aggregate lithium cap workaround. Model says "carrier-side, not modelable," but tenant historical bump-rate priors enable stochastic soft-cap modeling.
- **1.10** Equipment swap rate-card change. `equipment_swap` in supply-side event taxonomy lists $U_a / N_{a,u}$ changes; rate-card on the affected arc may also change.
- **1.12** AWB stock per-station-per-carrier (not just per-tenant). Deferred item exists but cardinality + cross-period structure unspecified.
- **1.13** Co-load cutoff offset. Master-loader's own consol cutoff is DCO-3h, not DCO; `CO_a^*` doesn't distinguish co-load vs direct.
- **2.5** Hub cancellation cascade. HKG-LAX cancels, cargo at HKG CFS. Storage-demurrage (deferred), screening-cert lapse (in 03-gap-finder), unused-leg payment (not modeled).
- **2.6** PSS mid-cycle margin erosion. Customer quote was pre-PSS; carrier invoice post-PSS. Diagnostic output doesn't flag PSS-driven margin gap.
- **2.7** Cargo-ready window collapsed to a point (shipper omitted late bound). Optimizer's pickup-deferral degree of freedom collapses; minimum window width invariant needed at intake.
- **2.9** One-off carrier exception. Operator gets verbal one-time approval for a denied carrier; cascade has no per-shipment override.
- **3.4** Spot-rate refresh latency varies per carrier × platform (cargo.one / WebCargo / CargoAi). Per-tenant per-platform calibration.
- **3.5** Surcharge catalog version control across tenants. Industry-published rates should be shared catalog with tenant-override.
- **5.2** Hard per-shipment budget cap exclusion is correct, but commercial communication of why is missing → pilot sales blocker.
- **5.4** Flight-level capacity excluded — but no risk flag for high-utilization-history flights. Trans-pacific peak season needs per-flight risk prior.
- **6.1** Carrier-policy cascade can't express "depends on what other HAWBs are on this MAWB" (pharma → all hubs GDP rule).
- **6.3** Preferred carrier set is unordered. Tenant policy frequently has ordered preferences (CI > BR > CX > LH).
- **6.4** Cascade can't express "if HAWB attribute then deny carriers without certification" cross-attribute rules.
- **7.1** Joint hub move with alliance through-AWB. One physical MAWB identity persists across what the model sees as separate arcs after hub equipment swap.
- **7.2** Co-load per-HAWB documentation cost not in catalog. Model under-prices co-load on small (<500 kg) shipments.
- **7.3** Split-shipment counterfactual. Operator-driven splitting at solve-time; model's post-solve diagnostics need "would have routed if split into ≤X kg pieces" sensitivity.

### EDGE (4)

- **1.9** BSA daily cut cross-flight cascade (SQ226 cut → SQ222 derate). Complements 03-gap-finder §3.4.
- **1.11** In-bond / broker-location triangulation. Cargo / broker / consignee may be in 3 different cities.
- **1.14** Pickup vehicle type (van vs 5-ton with liftgate); pickup window granularity is vehicle-typed.
- **2.3** RQ vs KK soft-commit — restated edge case of 1.5.
- **2.8** Contradictory signals (shipper WhatsApp says "late" while CFS receiving log shows on-time). Orchestrator gap, not model.
- **2.10** Late-DGR-declaration at CFS receiving breaks already-built MAWB. Demand-side event class missing from lifecycle taxonomy.
- **6.2** Conditional denies ("deny X if predicate(shipment, supply)"). Edge at MVP, material as rule library grows.
- **7.4** Multi-stop arc combinatorics. Phase 1 step 5 overlapping enumeration policy → $|M|$ could be 5–10× MVP estimate at Phase 2 scale.

### Closing-observation patterns from the agent (worth re-reading directly)

- **Pattern 1:** Operator workflow over-trust risk — many findings need diagnostic / sensitivity / attribution output, NOT new MILP constraints.
- **Pattern 2:** Data feeds are the binding constraint, not the math.
- **Pattern 3:** Fallback arc is operationally isolated; needs trend monitoring + root-cause attribution.
- **Pattern 4:** Workflow choreography (trucks, FWB/FHL filing, RQ→KK, chat updates, DG re-declaration) is what the operator manages; model should surface which choreography constraints are binding.

### Agent's recommended Top 5 for pilot + Top 3 for pilot+1

**Top 5 must-fix for pilot:**

1. 5.1 — Per-HAWB cost attribution promoted from deferred.
2. 5.3 — Per-HAWB tardiness reporting in hours.
3. 1.1 — Truck dispatch pickup-deadline parameter.
4. 4.2 — Fallback root-cause attribution in pre-filter.
5. 3.1/3.2/3.3 — Lithium matrix + embargo + BSA ingestion tooling.

**Top 3 for pilot+1:**

6. 1.5 — Booking-confirmation state machine in orchestrator.
7. 6.1 — MAWB composition-conditional rules in cascade.
8. 5.4 — Per-flight risk priors from schedule-reliability.

---

## C. Formulation-goodness agent (`08-formulation-goodness.md`) — residual

### REARCHITECT (3 remaining)

- **F5.2 / F14.1** — Probabilistic-transit migration paths fork. Three structurally distinct options: (a) quantile-bound arrival in `arr_dest` (arc-based-compatible); (b) expected $\tau^2$ over scenarios (requires scenario decomposition or per-arc moment propagation); (c) chance-constrained / CVaR (path-based natural). Pass A added the three-path naming to the deferred TT-Service item; the *MVP commitment* to which path is intended is still open.
- **F10.2** — `CWLinearizer` interface. Any future rebate / volume-kicker / marketing-credit term with a negative coefficient on $\text{CW}_{a,g}$ breaks the C.4c/d $\geq$-inequality form (monotonicity invariant); post-solve assertion catches it AFTER wrong cost is acted on. Proposed structural fix: rate-family-registration-time validation, two-mode interface (Inequality / Equality with complementarity binary).

### PROMOTE-EARLIER (6 remaining)

- **F1.3** — Q1 (path-based pivot) trigger should be joint with P1 promotion of probabilistic transit, not independent. Plan as a joint commitment.
- **F4.3** — Indicator-constraint factoring for commercial-solver pivot. Factor the `min_flat_breaks` linearization as a method on a `RateFamilyLinearizer` class with two implementations (HiGHS-bigM, commercial-indicator). Low cost now, big payoff later.
- **F8.2** — BSA accumulator concurrency design (orchestrator). Three patterns sketched: pessimistic locking on $A_c$, optimistic concurrency with reconciliation, solve serialization within period. Tied to J3 from Session 20.
- **X.2** — Warm-start across rolling-horizon re-solves. Right "warm" data is per-HAWB $x_{k,a}$ from previous solve where locked prefix is unchanged. State-management problem; design now.

### TIGHTEN (~14 remaining)

- **F1.2** — Add LP-gap-based path-based-pivot trigger alongside wall-clock (10% LP gap on time-binding instances).
- **F2.3** — Define $c^{\text{MAWB}}_{\text{fix}}$ scope as "new AWB number issued," not "arc activation," so future arc-splitting doesn't silently double the fixed charge.
- **F3.2** — Poset on temperature regimes (pharma ≤ chilled ≤ ambient) for partition-refinement P1. Coarser cells than sub-band refinement.
- **F4.4** — LP-gap contribution from break-disaggregation as walking-skeleton metric.
- **F5.3** — PWL bias underestimates between grid points; plan for HiGHS MIQP support (1-year horizon).
- **F7.2** — Use incumbent-bound spread explicitly for $\varepsilon^{\text{pref}}$, not relative MIP gap as proxy.
- **F7.3** — Pass-2 objective counts arc-touches, not HAWBs satisfied. Document as a design choice or migrate to per-HAWB binary.
- **F8.3** — Forecast-aware accumulator: $A_c^{\text{effective}} = A_c \cdot (\text{remaining period} / \text{expected remaining demand fraction})$. P1.
- **F9.3** — Walking-skeleton `lp_gap_by_constraint_family` should distinguish C.13b-2 from C.13b-1 (pivot LP-gap source).
- **F11.2** — BSA `A_c` pre-split heuristic across components in component decomposition. Avoids Lagrangian for the common case.
- **F12.3** — Subdivide forward-time-window pruning rate by cause (cutoff vs reachability vs ETA-anchor).
- **F13.2** — Anticipate slot-symmetry breaking with ordering constraints on $\eta$ slots if disaggregation is ever promoted.
- **F16.2** — Document per-HAWB attribution as post-solve concern with rule-based allocation (proportional-to-CW / Shapley); do NOT surface "per-HAWB cost" in MILP output.
- **X.1** — User-supplied valid inequalities (lifted cover on C.5b, subset-sum on C.5). Defer until commercial-solver evaluation.
- **X.5** — One-page "MILP boundary" document enumerating what's IN the MILP vs PRECOMPUTED and the data flow. Maintainer-onboarding doc.

### Cross-cutting safe-to-defer worth knowing

- **F6.3** — Walking-skeleton invariant: $\mu_a \geq \sum_{\text{legs}}(\text{block time} + \text{MCT})$ for every multi-leg arc emitted. One-line data-bug guard.
- **X.3** — Reachability degeneracy at lock prefix; verify `t_obs` semantics don't admit negative $\tau_k$.
- **X.4** — Graph caching across re-solves (recompute $A_k$ only for HAWBs whose locks changed). Implementation note.
- **X.6** — $F$ enters only as transient summation index in Path-B surcharges, not in any variable domain. Code-clarity flag.

### SAFE-TO-DEFER (~25, ship as-is)

The remaining findings confirm formulation choices are right: arc-based at MVP, $(a, g)$ MAWB indexing, functional $g$, big-M for HiGHS, PWL outer-approx, J19 forward-time-window propagation, lex two-pass, accumulator single-solve, two-inequality pivot, slot-disaggregation deferred, monotonicity invariant within current rate families, component decomposition first, no MAWB symmetry, singleton VAL/HUM/AVI groups, fallback construct, pre-filter exemption, cost-decomposition operationally clean, etc. All explicitly validated. Read `08-formulation-goodness.md` for the full list with reasoning if a particular choice comes up later.

---

## Suggested triage prompts for the user

1. **From section B (real-world)** — accept the agent's "Top 5 for pilot" as Pass B targets? Or pick / drop / re-prioritize?
2. **From section C REARCHITECT** — for F5.2/14.1 (probabilistic transit), commit MVP to (a) quantile-bound only? Document (b)/(c) as separate P2 paths? Or leave open?
3. **From section C REARCHITECT** — for F10.2 (`CWLinearizer`), is this worth a design note now, or wait for the first negative-coefficient rate family to actually appear?
4. **From section C PROMOTE-EARLIER** — F8.2 BSA accumulator concurrency was already J3 in Session 20; ready to design now, or still wait?
5. **From section A NOTATION-16** — pivot per-flight vs per-MAWB-arc. Needs your read on whether multi-leg interline MAWB-arcs face one pivot floor (current) or per-leg pivot floors (under-charged today if so).
6. **From sections A and C TIGHTEN** — any specific items to fold into a "Pass A.5" cheap-LaTeX-edit batch, or all of these go in Pass D (defer to next round)?
