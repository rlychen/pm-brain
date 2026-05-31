# Formulation Goodness — Architectural Critique of `air_freight_routing.tex`

**Reviewer lens:** OR / MILP architecture. Crainic-Frangioni-Toth multimodal service network design, Ahuja-Magnanti-Orlin MCNF, Desrosiers-Lübbecke column generation, Wolsey integer programming, Powell ADP. Concern is *formulation architecture* — is this the right way to model the problem, or are there structurally better alternatives the author hasn't considered?

**Out of scope:** notation hygiene, missing operational realities, PWL grid calibration (Q5), §4.3 grouping table, slack metric design.

---

## Executive verdict

**This is a defensible MVP formulation, and on the dimensions that matter for an MVP it is better than what most teams in the freight-OR space would produce.** Three calls stand out as architecturally strong: (i) pushing time / cutoff / MCT enforcement into graph build via forward-time-window propagation, which collapses a ~2,400-row block of the MILP into preprocessing; (ii) using the `(arc, group)` MAWB representation rather than an explicit assignment binary `h_{k,m}`, which kills the permutation symmetry that destroys consolidation MILPs in the literature; (iii) the fallback-arc construct, which converts an "INFEASIBLE / no rescue plan" pathology into a clean cost signal the orchestrator can read. None of these are obvious; together they explain why an arc-based formulation can survive at MVP scale where a naive bucket-per-flight MCNF would not.

**The places where the architecture is fragile are mostly known to the author and parked in `scalability.md`:** the arc-based formulation will become path-based (SPPRC pricing) somewhere between |K|=300 and |K|=750, and the tightening for that move (resource-constraint discipline, label dominance, column management) is sketched but not designed. The quadratic-tardiness PWL outer-approximation is the right choice now but creates habits that don't migrate cleanly to the P1 chance-constrained / CVaR reformulation teased in Finding-S-Ch-1. The monotonicity invariant that makes C.4c/d work as ≥-inequalities is structurally fragile to any future rebate-style rate-family extension. And the equalized BSA accumulator is the right move for a single solve but does not survive multi-solve-per-period concurrency without additional design.

**Net:** ship this for MVP. Promote three things earlier than P1 (specifically: HAWB aggregation default-decision via instrumented metric, BSA accumulator concurrency design, dominance pre-filter on MAWB candidates). Pivot to path-based formulation when the walking-skeleton metrics — *not* solve-time alone — say so.

---

## Q1. Arc-based vs path-based vs hybrid

**Finding 1.1 — Arc-based is the right MVP choice. SAFE-TO-DEFER.**
At ~2,500 binaries / ~8,000 constraints, HiGHS will solve this monolithically in seconds-to-minutes per component. The pricing-subproblem (SPPRC) infrastructure for path-based requires a label-correcting algorithm with dominance, column management, and branching scheme — 2–3 months of careful work. The author has the right ladder.

**Finding 1.2 — The author's "$>15$ min sustained" trigger criterion is wrong on its own. TIGHTEN.**
Solve time is a lagging indicator. The structurally correct trigger is the *LP-gap-by-constraint-family* metric (already in the walking-skeleton). If the time-window family or the BSA-coupling family dominates the LP gap, the path-based formulation will bring disproportionate value because both are *naturally absorbed* into the SPPRC label state (time as a resource, BSA-coupling as a single capacity dual on $z_{f,u}^c$ in the master). Recommend: instrument *two* triggers, and pivot when either fires:
- Wall-clock $\geq 15$ min on production-typical instance.
- LP gap from time-binding constraints exceeds 10% on instances where time is binding, regardless of solve time.

**Finding 1.3 — The Q1 line is drawn earlier than the author thinks. PROMOTE-EARLIER.**
The §5.10 sketch in `scalability.md` mentions that path-based naturally handles "the re-screen surcharge" (path-dependent jurisdictional cost) and the probabilistic-transit Finding-S-Ch-1 hook. Both are P1 items the author flags but defers. If P1 promotes either of those two items, the arc-based formulation acquires an *intrinsic* representation gap that big-M tricks cannot bridge cleanly. Concretely: a chance constraint $\Pr(\text{arrival} \leq T_k^{\text{abs}}) \geq p$ requires either (a) a scenario-based extensive form (multiplies MILP size by $|\Omega|$) or (b) a path-based formulation where each path's reliability is a column-time computation. The arc-based has no good (a) — extensive forms on 8,000 rows × 100 scenarios are 800K rows. So Q1's tipping point is *whichever fires first*: scale ($\geq 500$ HAWBs sustained) **or** P1 promotion of probabilistic transit. The author treats these as independent; they should be planned as a joint commitment.

**Finding 1.4 — Hybrid is not on the table and probably shouldn't be. SAFE-TO-DEFER.**
A hybrid (arc-based on ground, path-based on air) would localize the resource-tracking complexity to air but at the cost of a coupling layer at hub nodes that is uglier than either pure formulation. Skip.

---

## Q2. MAWB indexed by $(a, g)$ vs alternatives

**Finding 2.1 — $(a, g)$ is correct and the symmetry argument is right. SAFE-TO-DEFER.**
The alternative — explicit assignment binaries $h_{k,m}$ for HAWB $k$ on MAWB $m$ — introduces permutation symmetry across identical MAWBs (the classical "bin labeling" symmetry in bin packing). Wolsey §17.3, Margot's symmetry-breaking literature, and every operational consolidation MILP that scales pay this tax with ordering constraints or orbital fixing. The $(a, g)$ formulation makes the MAWB index *concrete* via the partition $g(\cdot)$ — no labeling choice exists, so no symmetry exists. This is structurally clever.

**Finding 2.2 — Set-partitioning over groups isn't even a candidate. SAFE-TO-DEFER.**
A set-partitioning master ($\theta_{S}$ binary, $S$ a "bundle of HAWBs sharing a MAWB") is the textbook (Barnhart-Hane-Vance 1998) approach to consolidation. It is *strictly worse* here because the bundle is determined by $g(\cdot)$ and the arc choice — not by the optimizer — so the SPP master is a degenerate enumeration. Author correctly does not consider it.

**Finding 2.3 — The case the author misses: $(a, g)$ does not give you a *MAWB-count* lever for fixed-cost calibration. TIGHTEN.**
With $z_{a,g}$ as the activation binary and $c^{\text{MAWB}}_{\text{fix}} \cdot z_{a,g}$ in the objective, the fixed cost binds *per (arc, group) pair*, not *per AWB number issued*. In practice these are the same (one AWB per MAWB) — but when an arc represents an interline through-segment (the Cargo-SPA case in §13), there is *one* AWB number for the through-MAWB and the fixed cost is right. When the arc represents a same-airline-multi-stop, ditto. **But** if a future modeling decision splits a multi-leg arc into two arcs (Q-7 scenario: airline-side rebill at a stopover, or the deferred multi-tier per-ULD case), each arc would carry its own $z_{a,g}$ → two MAWB fixed charges where one is operationally correct. The "MAWB issuance" cost is an *AWB-number-issuance* event, not an arc activation. Recommend: define $c^{\text{MAWB}}_{\text{fix}}$ as scoped to "$z_{a,g}$ for $a$ where the arc represents a new offer / new AWB number" — currently this is implicit. Make it explicit so future arc-splitting doesn't silently double the fixed-charge.

**Finding 2.4 — $|M| = \sum_a |G_a|$ scales linearly in HAWB count via $G_a$. SAFE-TO-DEFER.**
At MVP $|G_a|_{\text{avg}} \approx 1.5$ this is benign. At Phase 2 ($|K| = 500$) it is still benign because $g$ is a partition over HAWB attributes, so $|G_a|$ saturates well before $|K|$. Author has this right.

---

## Q3. Group $g(k)$ as a function of attributes

**Finding 3.1 — Functional $g(\cdot)$ is the right MVP call. SAFE-TO-DEFER.**
The alternative — a CSP-style mixed-group constraint where HAWBs can share a MAWB if their attribute pair satisfies some predicate — is the textbook DG-segregation problem (IATA segregation table is pairwise, non-transitive). This is structurally not a partition; modeling it as a partition (as the author does) and pushing the pairwise stuff to the ULD layer is the standard separation of concerns. Crainic's freight-consolidation papers do the same: contract / billing group at the upper layer, physical compatibility at the lower layer.

**Finding 3.2 — Real ops do have one case the partition misses: temperature mixing. TIGHTEN (P1).**
A pharma shipment validated for $+2$ to $+8\,^\circ$C and a chilled produce shipment at "ambient-chilled" can sometimes co-mawb under a single envelope (the colder validation dominates, both ship at $+2$ to $+8$). The author's partition forces them into separate MAWBs. Deferred-item 7 (`item:temp-band-refinement`) acknowledges this but the proposed fix — sub-bands — *makes the partition finer*, not more permissive. The right structural move is a *poset* on temperature regimes: pharma ≤ chilled ≤ ambient (where ≤ means "validation envelope contains"). HAWBs share a MAWB if their temperature regime tuples agree on the *infimum*. This is *still* a partition (the equivalence class is "shipments that agree on the infimum") but the partition's cells are coarser. Defer to P1; flag for the partition-refinement design pass.

**Finding 3.3 — Singleton groups for VAL/HUM/AVI are operationally right but waste MILP structure. SAFE-TO-DEFER.**
Each VAL HAWB gets its own $g(k) = (\text{VAL}, k)$ → singleton group → singleton $M$ entry → its own $z_{a,g}$ and $\text{CW}_{a,g}$. Algebraically clean. The "wasted structure" is that for these HAWBs the $z_{a,g}$ binary is determined by $x_{k,a}$ (one HAWB only), so C.2a and C.2b collapse to $z_{a,g} = x_{k,a}$. HiGHS presolve will collapse this. No action needed.

---

## Q4. Big-M vs disaggregation vs SOS / indicator constraints

**Finding 4.1 — Big-M with per-MAWB tightening is the right MVP call for HiGHS. SAFE-TO-DEFER.**
HiGHS as of late 2025 does not have first-class indicator-constraint support comparable to Gurobi / CPLEX. SOS-1 / SOS-2 are present but the BW disaggregation here is not naturally SOS — it is a "select-one-break" pattern where the auxiliary $\text{BW}_b$ tracks weight, not a convex combination. Author correctly stays in big-M.

**Finding 4.2 — The 3-inequality disaggregation is correct and LP-tight in the right sense. SAFE-TO-DEFER.**
The Session-15 critique caught the 4-inequality version (with the spurious $\text{BW}_b \leq \text{CW}$) that banned round-up. The 3-inequality form gives:
- $\text{BW}_b \leq M \gamma_b$ (forces 0 when $\gamma = 0$)
- $\text{BW}_b \geq \text{break}_b \cdot \gamma_b$ (minimum billed weight when active)
- $\text{BW}_b \geq \text{CW} - M(1-\gamma_b)$ (the chargeable-weight floor when active)
At integer feasibility this gives $\text{BW}_b = \max(\text{CW}, \text{break}_b)$ when $\gamma_b = 1$, $\text{BW}_b = 0$ otherwise. The LP relaxation does *not* give this exactly — at fractional $\gamma$ the LP can split the "max" between breaks at less-than-true cost. This is the standard LP gap on disaggregated big-M. Author has the right form; the gap is a known and tolerable cost.

**Finding 4.3 — Indicator-constraint preparation for commercial-solver pivot. PROMOTE-EARLIER (lightweight).**
The §scalability strategy mentions commercial-solver evaluation. When that happens, the BW disaggregation should be expressible as native indicator constraints in Gurobi / CPLEX — `gamma[a,g,b] = 1 -> BW[a,g,b] >= max(CW[a,g], break[a,g,b])` — which Gurobi handles via either indicator constraints (general) or rotated SOC for the convex pieces. This is a 50-line code switch *if the model code keeps the indicator semantics first-class* rather than hardcoding big-M. Recommend: factor the linearization so the `cost_min_flat_breaks(a, g)` function is a method on a `RateFamilyLinearizer` class with two implementations (HiGHS-bigM, commercial-indicator). Low cost now, big payoff later.

**Finding 4.4 — Disaggregation form is *not* LP-tight; this is the right tradeoff but should be measured. TIGHTEN.**
Add a walking-skeleton metric: LP gap contribution from the break-disaggregation family. If on TACT-heavy instances this dominates total LP gap, then either (a) commercial solver with native indicators, or (b) per-arc convex-hull formulation (Sherali-Adams lift) for the small-break-count case. Don't pre-optimize; instrument.

---

## Q5. Quadratic tardiness PWL outer-approximation vs alternatives

**Finding 5.1 — Outer-approximation tangent cuts is the right MVP choice for HiGHS. SAFE-TO-DEFER.**
HiGHS does not solve MIQPs. SOCP-via-rotated-cone is supported only in commercial solvers. Inner approximation gives a non-convex feasible region for a minimization — wrong direction. The PWL outer-approximation via tangent cuts is the canonical move for convex-quadratic-in-MILP. No binary needed because $\tau^2$ is convex (cuts are dominated automatically by the minimization). Author has this right.

**Finding 5.2 — The PWL technique choice creates a migration trap. REARCHITECT (anticipate now, not later).**

The Finding-S-Ch-1 hook says P1 will swap deterministic transit for a TT-Service P85-P90 quantile. Two phrasings of "probabilistic tardiness" exist:

- **(P85-quantile-bound)** Plug the P-quantile into $\text{arr\_dest}(k, a)$. The current quadratic-tardiness PWL still works without change. This is the path the author implicitly assumes.
- **(Chance-constrained / CVaR)** Penalize $\mathbb{E}[\tau_k^2]$ or $\text{CVaR}_\alpha(\tau_k)$ over an arc-transit distribution. This requires scenario averaging *inside* the objective, which is a *different formulation* — the PWL grid is now on a *distribution*, not a *deterministic $\tau$*.

The author's deferred item `item:probabilistic-transit` mentions "chance constraint or CVaR" but doesn't acknowledge that the second path requires structural change. **Recommend:** explicitly commit MVP to the (P85-quantile-bound) path, and document that the CVaR / chance-constrained variant requires either scenario decomposition or a SOCP-friendly reformulation. This prevents a future P1 issue ticket that says "we promised CVaR; we have quadratic-tangent PWL; these don't compose."

**Finding 5.3 — Outer-approximation underestimates the true quadratic between grid points; this matters more than the author thinks. TIGHTEN.**
The PWL is a *lower bound* on the true quadratic. At integer feasibility this is exact at grid points and underestimates between them — by up to $W_k \cdot (\hat\tau_{j+1} - \hat\tau_j)^2 / 4$ at the midpoint. The author quantifies this honestly (§lin-tardiness). **But:** the optimizer's *argmin* is biased toward $\tau_k$ values that fall between grid points where the penalty is underestimated. This is a *systematic bias toward modest tardiness over no tardiness*, in a way that gets worse at the wide ends of the grid. The fix is either: (a) denser grid in the high-density region (author mentions); or (b) replacing $W_k \tau_k^2$ with $W_k \tau_k^2$ exactly via HiGHS's QP capability if/when they add MIQP. HiGHS is on a trajectory toward MIQP per their roadmap; this is a 1-year time horizon, not 5. Plan for it.

**Finding 5.4 — The "use the fallback as the worst-case $\tau$ bound" is clever but locks the PWL grid to backstop-dependent calibration. SAFE-TO-DEFER (with caveat).**
The grid {0, 4, 12, 36, 96}h works for PER-class backstops (~48h). For GEN class with a 30-day customer-cancellation backstop ($T_k^{\text{abs}} - \Delta_k \sim 720$h), the largest grid point at 96h gives a tangent slope of $2 W_k \cdot 96 = 192 W_k$, while the true derivative at 720h is $1440 W_k$. The optimizer underestimates penalty by ~$7.5\times$ at the high end. This is the open question already flagged in `CONTEXT.md` for next session. The author has the right plan (per-HAWB relative grid). My only architectural addition: *also* add a post-solve assertion that flags any solve where $\tau_k$ exceeds the second-highest grid point — these are the cases where the PWL bias is large enough to potentially flip the routing choice.

---

## Q6. Forward-time-window propagation at graph build vs MILP constraint family

**Finding 6.1 — J19 was a great structural move. SAFE-TO-DEFER.**
For deterministic transit times this is mathematically equivalent to the big-M time-propagation + cutoff inequality family, and pushes the work to a much cheaper computation (BFS-with-bookkeeping vs LP-with-2400-rows). Eliminating 1500 + 300 rows is real money. Author has this right.

**Finding 6.2 — Forward-time-window propagation lifts to P-quantile *with significant degeneration of label semantics*. REARCHITECT (when P1 fires).**

The author claims the propagation "lifts naturally" to quantile windows. *Partially true*. Specifically:

- **Per-arc P-quantile transits**: propagate as $t^{\text{lo}}_{n_2} = t^{\text{lo}}_{n_1} + Q_p(\text{transit}(k, a))$, $t^{\text{hi}}_{n_2} = t^{\text{hi}}_{n_1} + Q_p(\text{transit}(k, a))$. This is *interval arithmetic on quantiles*, not a true quantile of the propagated arrival distribution. The sum of two P85 quantiles is *not* the P85 of the sum — typically it's higher (because the two distributions don't both miss together with probability $(1-p)^2$, but you've assumed they do). This is the convolution problem in propagation-tree analyses (Ahmadbeygi-Cohn 2008, your memory record).
- **End-to-end P-quantile**: the right thing is a P-quantile *of the path arrival distribution*, computed by convolving arc distributions along each path. This requires *per-path* analysis — *exactly* what column generation gives you naturally. Arc-based forward propagation cannot compute it.

**Recommendation:** when P1 promotes the TT-Service quantile binding, **do not** lift forward-time-window propagation to quantile arithmetic. Either (a) precompute the end-to-end P-quantile *per terminal arc* (as the author hints at in `item:tt-quantile-binding`) and use it only in $\text{arr\_dest}(k, a)$, leaving the propagation deterministic on something safe like the mean — this is OK if you accept the cutoff check is mean-based, but cutoffs are typically negotiated against worst-case; or (b) pivot to path-based at the same time and use the SPPRC label state. Don't do (c) "naive quantile interval propagation" — it's silently wrong.

**Finding 6.3 — The propagation enforces only cutoffs and reachability; it does not enforce *MCT at hub transitions inside a multi-leg arc*. SAFE-TO-DEFER but verify.**
A multi-leg MAWB-arc has $\mu_a$ that *includes* internal MCT (§ULD-interchange). This is fine for the propagation. But: if the data layer ever produces a multi-leg arc whose computed $\mu_a$ doesn't actually include internal MCT (data bug), the MILP will silently route through it. Recommend: walking-skeleton metric `internal_mct_compliance_check` that asserts $\mu_a \geq \sum_{\text{legs}} (\text{block time} + \text{MCT})$ for every multi-leg arc emitted by the graph generator. One-line invariant; saves a hard-to-debug class of incidents.

---

## Q7. Lexicographic two-pass vs $\varepsilon$-constraint vs weighted-sum

**Finding 7.1 — Lex two-pass is correct. SAFE-TO-DEFER.**
Weighted-sum requires a dollar-per-preferred-carrier coefficient that has no natural calibration and creates artifacts at preference-tie boundaries. $\varepsilon$-constraint (in the Haimes 1971 sense) is *what the author is doing in pass 2* — the lex framing is just a sequential-application formulation. The author correctly identifies this and cites Haimes / Ehrgott. The two-pass form is operationally cleaner because the cost ceiling is an *explicit auditable tenant policy parameter* rather than a coefficient buried in a penalty.

**Finding 7.2 — The MIP-gap-vs-$\varepsilon^{\text{pref}}$ subtlety reveals a real but minor structural issue. TIGHTEN.**
The author's required relationship $\varepsilon^{\text{pref}} \geq g \cdot z^*$ is correct but only protects against Pass-2 infeasibility from the MIP gap on Pass 1. It does *not* protect against the Pass-1 gap on the *underlying LP relaxation* — if HiGHS terminates Pass 1 at the bound rather than the incumbent, the Pass-2 ceiling is even tighter relative to the true optimum. **Tightening:** use `Highs::getInfo()` to recover the Pass-1 *incumbent* and the *bound*, and set $\varepsilon^{\text{pref}} \geq \text{incumbent} - \text{bound}$ explicitly. Don't trust the relative gap as a proxy.

**Finding 7.3 — Pass 2 objective is fine but has a subtle quirk worth noting. SAFE-TO-DEFER.**
Pass 2 maximizes $\sum_k \sum_{a : C^{\text{pref}}\text{-match}} x_{k,a}$. This *counts arc usage*, not HAWBs satisfied. A HAWB on a 3-arc preferred-carrier-on-every-arc route contributes 3; a HAWB on a 1-arc preferred route contributes 1. Operationally fine (more preferred-arc-touches = better) but slightly off-spec — the author's intent is likely "HAWBs whose journey is fully preferred-carrier." Recommend documenting this is a design choice (arc-touch maximization, which is a tighter signal) rather than a bug. Adjust if tenant policy says "all-or-nothing per HAWB" — that's a per-HAWB binary $y_k = \prod_a \mathbb{1}[x_{k,a} \cdot \text{pref}(k,a)]$, which linearizes to $y_k \leq x_{k,a}$ for every preferred-eligible $a$ + flow-conservation-aware constraints. More work; not free.

---

## Q8. Equalized BSA via $A_c$ allowance accumulator (external to MILP)

**Finding 8.1 — Accumulator-only is the right call for the single-solve case. SAFE-TO-DEFER.**
A hard period-min in the MILP creates spurious infeasibility when this batch has nothing to load on contract $c$. Lagrangian relaxation with subgradient updates is finicky and the author correctly rejects it. The accumulator reduces to "subtract consumption from allowance, treat residual as marginal-free weight" — clean and correct for the *single-batch* case.

**Finding 8.2 — The accumulator breaks under concurrent solves and is not designed for it. PROMOTE-EARLIER.**
This is the externality the question asks about. Two solves $S_1$ and $S_2$ in the same period both see $A_c = $ current-residual at solve start. Both can consume the same allowance independently → over-consume → underpriced overage → eventual period-end cleanup. The author's `project_orchestrator_design.md` memory hints at "concurrency model needs design" but doesn't tie it to the BSA accumulator specifically.

**Three mitigation patterns:**
- **Pessimistic locking on $A_c$.** Each solve "reserves" expected consumption at start, releases unused at end. Standard pattern; one Redis row per (contract, period). Operationally simple; modest latency.
- **Optimistic concurrency with reconciliation.** Each solve uses snapshot $A_c$, post-solve commit checks for over-consumption and re-solves the loser. Correct but adds re-solve risk.
- **Solve serialization within period boundaries.** Queue solves per-period; one at a time. Simplest; possibly too slow at scale.

**Recommend:** decide this now in the orchestrator design pass (already flagged in J3). Don't wait for production to surface the race.

**Finding 8.3 — The accumulator pattern has a second-order issue: it does not see the *future demand* on $c$ this period. TIGHTEN (P1).**
If the period is half over and $A_c$ is half-consumed, but the remaining half-period typically has 2× the demand of the first half (seasonal / lane-tide), the marginal pricing is wrong: this batch is loading against an allowance that will be exhausted by future demand. The correct accounting is a *shadow price* on remaining allowance, which is what Lagrangian gives you — but Lagrangian is what the author rightly rejected for *fragility*. The middle ground is *forecast-aware accumulator*: $A_c^{\text{effective}} = A_c \cdot (\text{remaining period} / \text{expected remaining demand fraction})$, treat as the free-quota. Recommend P1 once you have demand forecasts; surface as a future open item.

---

## Q9. Pivot weight linearization (per-flight BSA)

**Finding 9.1 — The two-inequality $\max$ on $\text{pivot}_{a,g}$ is tight at integer feasibility because $r_a > 0$. SAFE-TO-DEFER.**
Standard. Author notes this. The LP relaxation doesn't see $r_a > 0$ as an integer property and can give fractional pivot less than max; this is fine because the bound is dominated by the integer solution.

**Finding 9.2 — Disaggregated $\eta_{a,g,u,n}$ form yields tighter LP but isn't worth it at MVP. SAFE-TO-DEFER.**
Per-ULD-slot binaries $\eta_{a,g,u,n}$ where $n$ indexes the $n$th ULD position would give an LP relaxation tight enough that the pivot constraint becomes redundant. At MVP scale $N_{a,u} \leq 10$, adding $\sim 10 \times |C^{\text{pu}}| \times |U_a|_{\text{avg}} \approx 1000$ extra binaries — comparable to the existing binary count, so a noticeable solve-time hit. Author correctly defers.

**Finding 9.3 — The current form's hidden gap: $\eta_{a,g,u}$ is integer not real, but the LP relaxation treats it as real. TIGHTEN.**
At LP, $\eta_{a,g,u}$ can be fractional, making $\pi_a \sum_u \eta_{a,g,u}$ fractional, which the LP exploits to keep $\text{pivot}_{a,g}$ artificially low. The LP relaxation gap on BSA-heavy instances is therefore likely to be dominated by this exact constraint. Walking-skeleton metric `lp_gap_by_constraint_family` should distinguish C.13b-2 from C.13b-1. If C.13b-2 dominates, disaggregation moves up the priority list.

---

## Q10. C.4c/C.4d as $\geq$ inequalities via monotonicity invariant

**Finding 10.1 — The invariant works for the current rate families. SAFE-TO-DEFER.**
Every cost term has non-negative coefficient on $\text{CW}_{a,g}$, $\text{Wt}$, $\text{Wv}$. Minimization drives $\text{CW}$ down to the lower envelope = max. This is the same trick that makes the soft-tardiness $\tau \geq 0, \tau \geq t - \Delta$ formulation exact, generalized. Author has it right.

**Finding 10.2 — A future rebate / volume-kicker / marketing-credit term breaks the invariant. REARCHITECT (when triggered).**

If a future rate family introduces a *negative* coefficient on $\text{CW}_{a,g}$ — examples: a tenant negotiates a "$\$2/kg$ rebate above 5000 kg" line; a peak-season discount kicks in past a threshold; a marketing credit per-shipment — the minimization no longer drives $\text{CW}$ to the max. It would instead drive it to whichever inequality is binding in the negative term's direction, producing a non-correct cost. The author flags this and mentions a post-solve assertion suite. **That's not enough.** Assertions catch the problem post-hoc; they don't prevent the wrong cost from being acted on.

**Recommendation (structural):** introduce a `CWLinearizer` interface with two modes:
- **Inequality mode** (current): only safe under monotonicity invariant; cheap.
- **Equality mode**: $\text{CW} = \text{Wt} + s_{\text{Wt}}$, $\text{CW} = \text{Wv} + s_{\text{Wv}}$, $s_{\text{Wt}} \cdot s_{\text{Wv}} = 0$ (complementarity) → linearize with a binary $\delta_{a,g}$ picking which side is binding. Two more inequalities + one binary per $(a, g)$.

Validate the invariant at *rate-family registration time* (when a tenant adds an offer to the catalog) by checking the coefficient sign on $\text{CW}$. Reject the offer if the invariant breaks and the linearizer is in Inequality mode. Auto-switch to Equality mode at the catalog layer rather than relying on post-solve assertions. This is the difference between "we'll catch it" and "we structurally cannot ship it."

**Finding 10.3 — Even within the invariant, the inequality form has a subtle inefficiency. SAFE-TO-DEFER.**
At LP, $\text{CW}$ is bounded below by $\max(\text{Wt}, \text{Wv})$ but can be anywhere in $[\max, \text{CW}^{\text{ub}}]$. Cost minimization drives it to the max, but only *because of cost*; bounds-propagation doesn't tighten $\text{CW}$. This is fine in practice but means the LP relaxation has a wider feasible region than necessary. If you ever cut-generate on $\text{CW}$ (a custom valid inequality), this slack will hurt. Not worth fixing for MVP.

---

## Q11. Component decomposition on supply graph $H$

**Finding 11.1 — Component decomposition is the right *first* decomposition. SAFE-TO-DEFER.**
Standard MCNF decomposition (Ahuja-Magnanti-Orlin §17). For a typical mid-market forwarder with geographically clustered lanes (TPE-NA, SZX-EU, etc.), connected components are mostly disjoint. Expected ~5-15 components per batch. Per-component independent solve is embarrassingly parallel. Author correctly puts this ahead of Lagrangian or column generation.

**Finding 11.2 — BSA coupling across components is real but not necessarily a decomposition-killer. TIGHTEN.**
The concern: an equalized BSA contract that spans two components (e.g., a forwarder has BSA on TPE-LAX and TPE-JFK with one carrier) couples them via $C^{\text{eq}}$ chargeable accumulator. This breaks the assumed independence of components.

**But:** the *accumulator-based* equalized BSA formulation only couples through $A_c$, which is a *constant* per solve. Within one MILP build, $A_c$ is a frozen scalar. So the components are *not* coupled inside the MILP — they're coupled in the *accumulator's allocation of $A_c$ to components*. The right move is to *pre-split* $A_c$ across components proportionally to expected component-level demand, then solve components independently. Post-solve, reconcile actual consumption against the split and adjust the accumulator. This is much simpler than Lagrangian.

**Recommendation:** add to `scalability.md` the "$A_c$ pre-split heuristic" as the BSA-coupling resolution under component decomposition. Lagrangian relaxation of C.13a (the author's suggested alternative) becomes a fallback only if the pre-split heuristic produces too much period-end imbalance.

**Finding 11.3 — Lagrangian relaxation of *just* C.13a is a reasonable Plan B but harder than it looks. SAFE-TO-DEFER.**
The author suggests Lagrangian relaxation of just C.13a as Plan B if BSA coupling is high. Two concerns: (a) C.13a is an aggregated *cost* term, not a hard constraint, so what you're relaxing is the chargeable-weight definition $\text{chargeable}(c) = \sum_{(a,g)} \text{CW}_{a,g}$ — this doesn't decompose by component in a way that gives independent subproblems; (b) the dual variable update on a single dualized constraint family is well-behaved (no nondifferentiability from multiple constraints), but the bound from Lagrangian is necessarily weaker than the LP relaxation, so it's only useful as a *warmstart* heuristic rather than a tight bound generator. The pre-split heuristic is simpler and probably as good.

---

## Q12. Walking-skeleton 8 metrics

**Finding 12.1 — The 8 listed metrics are the right *operational* metrics but miss two *formulation-quality* metrics. PROMOTE-EARLIER.**

The 8 metrics cover:
1. $|A_k|$ histogram + per-predicate drop counts
2. $|G_a|$ distribution + activated bucket count
3. LP-vs-MIP gap by constraint family
4. Forward-time-window pruning rate
5. Connected components of $H$ — shadow
6. Super-shipment equivalence classes — shadow
7. BSA-contract cross-coupling fraction
8. Post-solve invariant assertions
+ HiGHS phase breakdown

Two missing metrics that would tell the author "your formulation is wrong" rather than "your solve is slow":

**Missing metric A: Fractional-$x$ frequency at LP-relaxation root.** Track the fraction of $x_{k,a}$ variables that are fractional at the root LP solve, and how many take values close to 0.5 (the "hard" fractional range). A high fraction here indicates the arc-based formulation's LP relaxation is intrinsically weak for this instance — *not* a solver issue but a *formulation* issue, signaling that path-based would help. This is the *direct* signal that says "pivot to column generation," much better than wall-clock alone.

**Missing metric B: Distribution of activated $z_{a,g}$ per arc.** If many arcs have multiple $g$ values activated (say, $|\{g : z_{a,g} = 1\}| > 2$ on a typical arc), the per-arc weight-break disaggregation is doing real work and the MAWB-fixed-charge is binding the consolidation decision. If the distribution is mostly 0 or 1 (single-group MAWBs everywhere), the $(a, g)$ formulation is overkill and you could collapse to a simpler $(a, \text{cargo\_class})$ partition with less binary count. This is a *formulation simplification* signal.

**Finding 12.2 — Metric 3 (LP gap by constraint family) is the highest-leverage. SAFE-TO-DEFER.**
Author has this. Worth flagging that the *implementation* is non-trivial — you need to either (a) compute reduced costs per constraint family at LP solve, summing absolute reduced costs as a proxy for "tightness contribution"; or (b) run constraint-family-ablation LPs (remove a family, re-solve LP, measure bound deterioration). (b) is more accurate but $O(|\text{families}|)$ extra LP solves per measurement. Recommend (a) as a default and (b) as a once-per-month diagnostic.

**Finding 12.3 — Metric 4 (forward-time-window pruning rate) needs to be subdivided. TIGHTEN.**
The author lists "% arcs dropped by step (i)–(iv) of §fwd-time-propagation" — that's good, but it doesn't tell you *which time bound* is binding (cutoff vs reachability vs ETA-anchoring). A pruning-by-cause breakdown would let the author calibrate cutoff stack inputs separately from arrival-time bounds. Cheap to add.

---

## Q13. Structural symmetry

**Finding 13.1 — The author's "no symmetry" claim on $(a, g)$ is correct *at the MAWB-instantiation layer*. SAFE-TO-DEFER.**
No two distinct $(a, g)$ pairs can be permuted to the same physical decision — each represents a unique MAWB. So no labeling symmetry on MAWB.

**Finding 13.2 — Symmetry hides in $\eta_{a,g,u}$ across multiple type-$u$ ULDs. TIGHTEN.**
$\eta_{a,g,u}$ is integer-valued: count of type-$u$ ULDs claimed. There is *no* labeling symmetry because the ULDs of one type are interchangeable — counted, not labeled. Good.

**But:** if Finding 9.2's disaggregation ever fires (per-ULD-slot binaries $\eta_{a,g,u,n}$), the symmetry on slot index $n$ is back. The author flags this is a P1 cost. Recommend: anticipate the symmetry-breaking now — e.g., $\eta_{a,g,u,1} \geq \eta_{a,g,u,2} \geq \ldots$ ordering constraint — and document it. Standard symmetry-breaking from bin packing literature.

**Finding 13.3 — Symmetry in $\gamma_{a,g,b}$ break selector. SAFE-TO-DEFER.**
The break selector $\gamma_{a,g,b}$ is one-hot ($\sum_b \gamma = z$), with each $b$ representing a *distinct* break (different rate and weight threshold). No labeling symmetry — each $b$ has unique semantics. Good.

**Finding 13.4 — Cross-MAWB symmetry on identical-attribute HAWBs. SAFE-TO-DEFER.**
Two HAWBs $k_1, k_2$ with identical $(g, w, v, \text{deadline}, \text{sp})$ are routing-interchangeable. The $x_{k,a}$ formulation makes them distinguishable binaries; they could in principle be swapped without changing cost, giving redundant search. **However:** they're routed independently through C.1 flow conservation, so the symmetry doesn't manifest as solver-search-time explosion — it manifests as $|K|$ extra binaries when "super-HAWB aggregation" (the author's `consolidation_mode = preprocess`) could reduce them. Author has this in the walking-skeleton (`aggregation_potential` shadow metric). Promote to default-on once measured? — author intentionally leaves the decision empirical. Fine.

---

## Q14. Soft tardiness with quadratic penalty vs probabilistic / chance-constrained

**Finding 14.1 — Quadratic-soft-tardiness is the right *MVP placeholder* but the migration path needs explicit design. REARCHITECT (forward-looking).**

This is essentially Finding 5.2 from a different angle. Three migration paths exist for "probabilistic transit":

**Path A — Quantile-bound (compatible).** Plug TT-Service P85 into $\text{arr\_dest}(k, a)$. Quadratic penalty over the quantile-arrival is still computable. **Cost:** the penalty is on the quantile arrival, not on $\mathbb{E}[\tau^2]$ — semantically "we pay a quadratic penalty for the 85th-percentile lateness." Operationally interpretable. The arc-based formulation is fine.

**Path B — Expected-quadratic-tardiness.** Penalty is $\mathbb{E}[\tau_k^2]$. Requires representation of the arrival distribution at $D_k^{\text{node}}$, which depends on path choice. **Not compatible with current architecture** — arc-based MILP cannot decompose path-dependent arrival distribution; need path-based.

**Path C — CVaR / chance-constrained.** Penalty is $\text{CVaR}_\alpha(\tau_k)$ or constraint is $\Pr(\tau_k \leq T_k^{\text{abs}} - \Delta_k) \geq p$. Requires either scenario-based extensive form (multiplies MILP size by $|\Omega|$) or path-based with per-path reliability computed at column-time.

The author's deferred item `item:probabilistic-transit` says "chance constraint or CVaR" without acknowledging that Paths B and C require structural change.

**Recommendation:** explicitly commit MVP roadmap to Path A. Document that Paths B and C require either path-based formulation or scenario decomposition. Add to the LaTeX a one-paragraph "P1 migration paths" subsection that names this fork. Otherwise a future planning conversation will hit "wait, we can't just swap the transit input" mid-implementation.

**Finding 14.2 — Quadratic-over-deterministic is a *bias* in a useful direction. SAFE-TO-DEFER.**
Quadratic penalizes outliers more than the mean, which approximates risk-aversion. This is a reasonable proxy for "the operator dislikes concentrated lateness." When the P1 stochastic move happens, this proxy is *replaced* by an explicit risk model, not augmented. Don't double up.

---

## Q15. The fallback arc as feasibility guarantee

**Finding 15.1 — Conceptually clean and operationally smart. SAFE-TO-DEFER.**
Converts "INFEASIBLE — stop and call ops" into "use fallback at $C^{\text{fallback}}$ — orchestrator triages." The post-solve $K^{\text{fb}}$ is a structured signal of rescue cases, with per-HAWB context. Strictly better than catching infeasibility exceptions.

**Finding 15.2 — The big constant $C^{\text{fallback}}$ has real numerical risks. TIGHTEN.**
The recommended $\$1{,}000{,}000$ default is a big-M in disguise. The LP relaxation can use *fractional* $x_{k, a^{\text{fb}}_k}$ values to "smear" the fallback usage across multiple HAWBs. If at LP $x_{k, a^{\text{fb}}_k} = 0.001$ for many $k$, the LP cost is low even though the rounded-up MIP requires whole fallbacks. **Two concerns:**
- **LP relaxation gap.** The fallback contribution to LP can be lower than the integer-feasible version by a factor of (number of HAWBs near-fractionally using fallback). Branch-and-bound then has to drive each $x_{k, a^{\text{fb}}_k}$ to 0 or 1 separately.
- **Numerical scaling.** $1{,}000{,}000$ vs the routing cost of $\sim 1000$ gives a $10^3$ coefficient ratio — well within HiGHS's tolerance, but the *root-LP-bound vs incumbent* gap calculation gets fragile when fallback dominates.

**Recommendation:**
- Set $C^{\text{fallback}}$ to the *smallest constant* that achieves the "prefer any real route" property. That's $\max_k W_k \cdot (T_k^{\text{abs}} - \Delta_k)^2 + \max(\text{realistic per-HAWB routing cost}) \cdot 10$ — not $\$1M$. For a tenant where the biggest real-route cost per HAWB is $\sim \$10K$, $C^{\text{fallback}} \approx \$100K$ is sufficient. The $\$1M$ default is *safe* but unnecessarily large.
- Add a walking-skeleton metric: LP fractional fallback usage per solve. If $\sum_k x_{k, a^{\text{fb}}_k}^{\text{LP}} - |K^{\text{fb}}|^{\text{MIP}}$ exceeds (say) 1, the LP relaxation is exploiting fallback smearing and you need a tighter $C^{\text{fallback}}$ or a per-HAWB explicit-disjunction reformulation.

**Finding 15.3 — Fallback bypasses pre-filter — but the *cost* of pre-filtering doesn't account for fallback compensation. SAFE-TO-DEFER.**
Author correctly exempts fallback from the 8 pre-filter predicates. This is necessary for the "always feasible" guarantee. No issue.

**Finding 15.4 — The arrival time of fallback is exactly $T_k^{\text{abs}}$, which sets the largest $\tau_k$ in the PWL grid. SAFE-TO-DEFER but reinforces Finding 5.4.**
This is the tightest test of the PWL approximation at the high-tardiness end. If the grid is wrong for GEN-class long backstops, the optimizer's fallback-vs-real-late-route decision is biased. Already covered by Finding 5.4.

---

## Q16. Output diagnostics decomposition

**Finding 16.1 — The decomposition into $z^{\text{routing}}$ / $z^{\text{tardiness}}$ / $z^{\text{fallback}}$ is operationally smart. SAFE-TO-DEFER.**
Operators see "cost to serve" as $z^{\text{routing}}$; tardiness and fallback are *flags* with their own dollar values. Hiding $C^{\text{fallback}}$ from operator screens (per §output-diagnostics) is the right call — it's a mathematical device, not a price.

**Finding 16.2 — The decomposition is *not* clean once Path-B per-flight surcharges layer in. TIGHTEN.**
$\text{flight\_uld\_surcharge\_cost}(f)$ is summed over flights, not arcs, not HAWBs. When you build the per-HAWB attribution for billing (a P1 item: `item:per-hawb-cost-attribution`), you'll need to allocate Path-B costs proportionally — *across MAWBs and groups on the flight*, and across HAWBs within each MAWB. This is a Shapley-allocation problem, not a clean sum-of-per-HAWB-terms. The current $z^{\text{routing}}$ is a per-solve dollar number, not a per-HAWB number; that's fine for the routing-cost display, but operators who drill into a single HAWB's cost will see an attribution layer that isn't formulation-driven. Recommend: explicitly document that per-HAWB attribution is a *post-solve* concern with rule-based allocation, not a formulation property. Don't surface "per-HAWB cost" in MILP output.

**Finding 16.3 — BSA equalization layers in a way that makes "cost-of-this-shipment" tricky. SAFE-TO-DEFER.**
$\text{over}_c$ is per-contract, not per-HAWB. The contribution $r_c \cdot \text{over}_c$ is a marginal cost on overage, but the *attribution* of that overage to specific HAWBs in the batch depends on a chosen allocation rule (proportional to CW? Shapley?). Same problem as Finding 16.2. Document as post-solve concern.

---

## Cross-cutting findings

**Finding X.1 — The model lacks an explicit valid-inequality strategy. TIGHTEN.**
The model has *no* user-supplied cuts. HiGHS will generate some default cuts (Gomory, MIR, knapsack) but the model's structure suggests at least two cut families that would tighten the LP:
- **Lifted cover cuts on C.5b** (per-ULD physical capacity). Standard MCNF tightening.
- **Subset-sum cuts on C.5** (per-contract allotment). When $N_{a,u}$ is small (~3), these become tight.

Recommend: when commercial-solver evaluation happens (Finding 4.3), see if user-cuts move the LP gap. Defer otherwise — HiGHS doesn't accept user-cuts as easily as Gurobi.

**Finding X.2 — Warm-start strategy across rolling-horizon re-solves is mentioned but underspecified. PROMOTE-EARLIER.**
HiGHS's `setSolution` accepts a partial MIP-feasible point. The interesting design question is: across rolling-horizon re-solves where 60-80% of HAWBs are partially locked, the right "warm" data is *the $x_{k,a}$ values from the previous solve for HAWBs whose locked prefix is unchanged*, not "previous full solution" (which is infeasible after locks shift). This is a non-trivial state-management problem. Recommend designing it now, not at production-incident time.

**Finding X.3 — The model's claim of being "always feasible by construction" is correct but the reachability check at preprocessing can still surface degenerate cases. SAFE-TO-DEFER.**
If a partially locked HAWB has a locked prefix that's already infeasible against $T_k^{\text{abs}}$ (cargo missed all cutoffs), the MILP will route it via the fallback with $\tau_k = T_k^{\text{abs}} - \Delta_k$. The pre-MILP reachability check (§locked-commitments) flags this as a rescue event. Good. **But:** if the lock is on a *finished* leg whose arrival exceeded $T_k^{\text{abs}}$ already, $\tau_k$ is negative against actual arrival — domain violation. Author's `t_obs` semantics should handle this; verify the preprocessing doesn't admit negative $\tau_k$.

**Finding X.4 — The MILP's lock-agnostic posture is structurally clean but expensive in re-solve setup. SAFE-TO-DEFER.**
Lock-agnostic means every re-solve rebuilds $A_k$ from scratch. For a forwarder with 60-80% lock prevalence in rolling-horizon, this is a lot of repeated graph work. The right pattern is *graph caching across re-solves* — keep the universal graph $A$, recompute $A_k$ for HAWBs whose locks changed. The model doesn't preclude this; just flag for the implementation phase.

**Finding X.5 — The split between MILP and "preprocessing" is doing real work but the boundary is informal. TIGHTEN.**
Eight things live in preprocessing/graph-build: cutoff propagation, MCT, hub timing, locked-prefix truncation, embargo gating, lithium gating, cargo-type gating, fallback-arc emission. This is good architectural separation. But there's no single document that says "MILP = these constraints; preprocessing = these computations." The §graph-construction.md covers most of it; some lives in the LaTeX. Recommend a one-page "MILP boundary" document that enumerates what's IN the MILP, what's PRECOMPUTED, and the data flow from preprocessing to MILP build. This is the kind of document that gets a future maintainer un-stuck fast.

**Finding X.6 — Surcharge linearization (Path A vs Path B) is correct but Path B's per-flight summation is the only place where flight $f$ appears as an index in the MILP. SAFE-TO-DEFER.**
$\sum_{f \in F} \text{flight\_uld\_surcharge\_cost}(f)$ — $f$ is summed over, and inside the sum $\eta_{a,g,u}$ is summed over arcs whose legs include $f$. The flight is only a *summation index*, not a decision. So this is correctly "outside" the MILP's decision structure. But it does mean the MILP's index *signature* is $(k, a, g, u, b)$ with $f$ as a transient summation — flag for code clarity that $F$ does not enter the variable domain.

---

## Summary table

| # | Topic | Severity | Action |
|---|---|---|---|
| 1.1 | Arc-based at MVP | SAFE-TO-DEFER | Ship |
| 1.2 | Path-based pivot trigger | TIGHTEN | Add LP-gap-based trigger alongside wall-clock |
| 1.3 | P1 promotion couples scale + stochastic | PROMOTE-EARLIER | Plan jointly |
| 1.4 | Hybrid arc/path | SAFE-TO-DEFER | Skip |
| 2.1 | $(a,g)$ correct, no symmetry | SAFE-TO-DEFER | Ship |
| 2.2 | Set-partitioning not a contender | SAFE-TO-DEFER | Author right |
| 2.3 | $c^{\text{MAWB}}_{\text{fix}}$ scope underspecified | TIGHTEN | Define scope = "new AWB number issued" |
| 2.4 | $|M|$ scales benignly | SAFE-TO-DEFER | OK |
| 3.1 | Functional $g$ vs CSP | SAFE-TO-DEFER | Author right |
| 3.2 | Poset on temp regimes | TIGHTEN (P1) | Flag for partition-refinement pass |
| 3.3 | Singleton groups | SAFE-TO-DEFER | Presolve collapses |
| 4.1 | Big-M for HiGHS | SAFE-TO-DEFER | Right call |
| 4.2 | 3-inequality BW form | SAFE-TO-DEFER | LP-tight at integer feasibility |
| 4.3 | Indicator-constraint prep for commercial solver | PROMOTE-EARLIER | Factor `RateFamilyLinearizer` interface |
| 4.4 | Disaggregation LP gap | TIGHTEN | Add LP-gap-per-family metric |
| 5.1 | PWL outer-approx | SAFE-TO-DEFER | Right call |
| 5.2 | PWL migration trap to CVaR | REARCHITECT (anticipate) | Commit MVP roadmap to quantile-bound path |
| 5.3 | PWL bias underestimates between grid points | TIGHTEN | Plan for MIQP when HiGHS adds support |
| 5.4 | Grid breaks for wide backstops | SAFE-TO-DEFER | Already in CONTEXT.md open items |
| 6.1 | J19 forward-time-window propagation | SAFE-TO-DEFER | Great move |
| 6.2 | Quantile propagation lift is *not* natural | REARCHITECT (when P1 fires) | Per-terminal-arc P-quantile only, not arithmetic |
| 6.3 | Internal MCT data-bug surface | SAFE-TO-DEFER | Add invariant assertion |
| 7.1 | Lex two-pass correct | SAFE-TO-DEFER | Ship |
| 7.2 | MIP-gap-vs-$\varepsilon^{\text{pref}}$ tighten | TIGHTEN | Use incumbent-bound spread explicitly |
| 7.3 | Pass-2 arc-touch vs HAWB binary | SAFE-TO-DEFER | Document choice |
| 8.1 | Accumulator single-solve | SAFE-TO-DEFER | Right call |
| 8.2 | Accumulator concurrency design | PROMOTE-EARLIER | Decide in orchestrator pass |
| 8.3 | Forecast-aware accumulator | TIGHTEN (P1) | Future open item |
| 9.1 | Pivot two-inequality form | SAFE-TO-DEFER | Right call |
| 9.2 | Slot disaggregation not worth it | SAFE-TO-DEFER | Defer |
| 9.3 | $\eta$ fractional at LP — hidden gap | TIGHTEN | Add LP-gap-per-family metric to confirm |
| 10.1 | Monotonicity invariant works | SAFE-TO-DEFER | Ship |
| 10.2 | Rebate / negative-coefficient term breaks invariant | REARCHITECT (when triggered) | `CWLinearizer` interface + catalog-time validation |
| 10.3 | LP slack on CW between Wt/Wv and ub | SAFE-TO-DEFER | Not worth fixing |
| 11.1 | Component decomposition first | SAFE-TO-DEFER | Right call |
| 11.2 | BSA pre-split heuristic | TIGHTEN | Add to `scalability.md` |
| 11.3 | Lagrangian Plan B is harder than it looks | SAFE-TO-DEFER | Note for future planning |
| 12.1 | Missing two formulation-quality metrics | PROMOTE-EARLIER | Add fractional-$x$ frequency + $z_{a,g}$ activation distribution |
| 12.2 | LP-gap-per-family — implementation | SAFE-TO-DEFER | Sum of |reduced costs| as proxy |
| 12.3 | Time-window pruning by cause | TIGHTEN | Subdivide by cutoff vs reachability vs ETA |
| 13.1 | No MAWB symmetry | SAFE-TO-DEFER | Confirmed |
| 13.2 | Anticipate slot-symmetry breaking | TIGHTEN | Document ordering constraints |
| 13.3 | $\gamma$ no symmetry | SAFE-TO-DEFER | OK |
| 13.4 | Cross-HAWB attribute symmetry — `aggregation_potential` | SAFE-TO-DEFER | Author has this |
| 14.1 | Quadratic-soft migration path | REARCHITECT (forward-looking) | Document path-A / B / C fork explicitly |
| 14.2 | Quadratic as risk-aversion proxy | SAFE-TO-DEFER | Replaced, not augmented, at P1 |
| 15.1 | Fallback construct | SAFE-TO-DEFER | Smart |
| 15.2 | $C^{\text{fallback}}$ numerical risks | TIGHTEN | Tighten constant; instrument LP smearing |
| 15.3 | Pre-filter exemption | SAFE-TO-DEFER | Necessary |
| 15.4 | PWL bias at fallback arrival | SAFE-TO-DEFER | Covered by 5.4 |
| 16.1 | Cost decomposition | SAFE-TO-DEFER | Operationally clean |
| 16.2 | Per-HAWB attribution problem | TIGHTEN | Document as post-solve only |
| 16.3 | BSA per-HAWB attribution | SAFE-TO-DEFER | Same as 16.2 |
| X.1 | Valid-inequality strategy | TIGHTEN | Defer until commercial-solver eval |
| X.2 | Warm-start across re-solves | PROMOTE-EARLIER | Design now |
| X.3 | Reachability degeneracy | SAFE-TO-DEFER | Verify $t_{\text{obs}}$ semantics |
| X.4 | Lock-agnostic re-solve cost | SAFE-TO-DEFER | Implementation concern |
| X.5 | MILP-boundary document | TIGHTEN | One-page enumeration |
| X.6 | $F$ as transient summation index | SAFE-TO-DEFER | Code-clarity note |

---

## Rearchitect findings — alternative sketches

### REARCHITECT 5.2 / 14.1 — Probabilistic transit migration path

**Current:** $\text{pen}_k \geq W_k (2 \hat\tau_j \tau_k - \hat\tau_j^2)$ with $\tau_k$ defined against deterministic $t_k^{D_k^{\text{node}}}$.

**Path A (quantile-bound — recommended commit):** Replace $\text{arr\_dest}(k, a)$ with TT-Service end-to-end P85 quantile precomputed per terminal arc. $\tau_k$ now denotes "85th-percentile lateness." PWL machinery unchanged. **Compatible with current architecture.**

**Path B (expected-quadratic-tardiness — requires path-based):** Penalty becomes $\mathbb{E}[\tau_k^2]$. This depends on the *path's arrival distribution*, which is a convolution of per-arc transit distributions. Cannot be decomposed in arc-based MCNF; the path's distribution is path-property. Move to column generation where each priced column carries its own $\mathbb{E}[\tau^2]$ contribution.

**Path C (chance-constrained):** $\Pr(\tau_k \leq T_k^{\text{abs}} - \Delta_k) \geq p$. Two routes:
- Scenario-based extensive form: replicate $x_{k,a}$ per scenario $\omega$, link via a violation-count constraint. MILP size × $|\Omega|$.
- Path-based with per-path reliability: each column carries $\Pr(\text{path on-time})$ computed at pricing; master constraint is a linear sum over chosen columns.

**Recommendation:** commit MVP/P1 to Path A. Document Paths B/C explicitly as requiring formulation pivot.

### REARCHITECT 10.2 — `CWLinearizer` interface for future rate-family extensions

**Current:** C.4c/d are $\geq$ inequalities; correctness depends on no rate family introducing negative coefficient on CW. Post-solve assertion catches violation but only after the fact.

**Alternative:** introduce `CWLinearizer` with two strategies.

**Strategy 1 — Inequality (current):**
- $\text{CW} \geq \text{Wt}$, $\text{CW} \geq \text{Wv}$.
- Valid only if monotonicity invariant holds for all registered rate families.
- Cheap: 2 inequalities per MAWB.

**Strategy 2 — Disjunctive equality:**
- Introduce $\delta_{a,g} \in \{0, 1\}$: $\delta = 1$ iff Wt is binding.
- $\text{CW} = \text{Wt} + s_{\text{Wv}}$, $s_{\text{Wv}} \geq 0$, $s_{\text{Wv}} \leq M(1-\delta)$, $\text{CW} \geq \text{Wv}$.
- $\text{CW} = \text{Wv} + s_{\text{Wt}}$, $s_{\text{Wt}} \geq 0$, $s_{\text{Wt}} \leq M\delta$, $\text{CW} \geq \text{Wt}$.
- Exact regardless of cost-coefficient sign.
- Cost: 1 binary + 4 inequalities per MAWB.

**Catalog-time validation:** when a tenant adds a rate family to the catalog, automatically check $\partial \text{cost} / \partial \text{CW} \geq 0$ on the family's cost function over its valid CW range. If true, register with Strategy 1. If false (rebate, volume kicker), Strategy 2.

This converts "post-solve assertion catches the bug" into "catalog ingestion prevents the bug." Bigger architectural ask but worth it.

### REARCHITECT 6.2 — Forward-time-window propagation under stochastic transit

**Current implicit claim:** the propagation "lifts naturally" to P-quantile windows.

**Counter:** interval arithmetic on quantiles is silently wrong. $Q_{0.85}(\text{transit}_a + \text{transit}_b) \neq Q_{0.85}(\text{transit}_a) + Q_{0.85}(\text{transit}_b)$.

**Two clean alternatives:**

**Alternative A — Per-terminal-arc end-to-end P-quantile (compatible with arc-based).** Precompute $\text{arr\_dest}(k, a)$ as the end-to-end P85 quantile *for the cheapest path ending at terminal arc $a$* — actually, for every distinct path ending at $a$, take the path quantile, store the *worst* among them. The forward-time-window propagation runs on the *means* (or some agreed deterministic surrogate), used only for cutoff/reachability admission; C.10a's destination arrival uses the precomputed quantile scalar. This preserves the arc-based formulation; the quantile information is baked into $\text{arr\_dest}$ at graph build.

*Limitation:* cutoff admission is deterministic-mean-based, so a path with high mean compliance but high variance might fail at cutoff even though its quantile is OK. Acceptable tradeoff.

**Alternative B — Path-based with quantile as a label resource (incompatible with current arc-based).** SPPRC label includes "convolved arrival distribution so far"; cutoff and tardiness are predicates on that distribution. Computationally heavier but semantically correct.

**Recommend Alternative A** when P1 promotes the TT-Service quantile binding; do **not** lift the propagation to quantile arithmetic.

---

**End of critique.**
