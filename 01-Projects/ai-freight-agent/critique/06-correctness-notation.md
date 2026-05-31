# Air freight MILP — correctness and notation critique

Scope: `model/air_freight_routing.tex` (v3, 2026-05-23). Math correctness + notation hygiene only.
Out of scope: architecture, scope, formulation choice, real-world coverage.

---

## Executive summary

- **One real bug (BUG-1):** the big-M for the `min_flat_breaks` break-disaggregation is set to
  `CW^ub_{a,g}`, which can be strictly smaller than the highest `break_{a,b}`. This makes the
  "round-up-to-higher-break" case — exactly the case the 3-inequality fix was designed to
  enable — infeasible whenever `break_{a,b} > CW^ub_{a,g}`. The Session 15 worked example
  (90 kg, breaks {45, 100}) happens to pass because both breaks fit under CW^ub for that
  instance; a single-HAWB instance with a higher break (e.g. 280 kg vs breaks {45, 100, 300,
  500, 1000} — the TACT card in §appendix-tact) silently bans the cost-optimal selection.
- **One semantic ambiguity (BUG-2 / NOTATION):** the bucket-cost dispatch for `per_uld_pivot`
  + `equalized` reads as if the per-MAWB cost is both "aggregated into chargeable(c)" and a
  bucket-cost contributor. As written it is ambiguous whether `cost^MAWB_{a,g}` is zero or
  `r_a · CW_{a,g}` on equalized arcs. Either reading is internally inconsistent with the
  rest of §sec:bsa-equalized-settlement.
- **One propagation-equivalence gap (BUG-3):** §sec:fwd-time-propagation propagates a single
  `[t^lo, t^hi]` window per (k, n). The flight-cutoff and t^hi-collapse rules depend on the
  *outbound* arc chosen at n (since each outbound flight has its own CO*_a). The window
  formulation as written is under-specified for nodes with multiple outbound flights — the
  claim of "mathematical equivalence to a big-M time-propagation family" is only true if
  the implementation does per-(node, outbound-arc) admission, which the text does not say.
- **~12 NOTATION defects of various severity.** Most important: `POD_k`, `Δ^post_k`,
  `T_k^dead`, `ETD_f`, `transit(k, a^fb_k)`, `ac(f)` are used in equations without ever
  appearing in any nomenclature row; `C^pu` is an arc set but uses the `C` (contract)
  prefix; `g` is overloaded as both consolidation group and MIP gap inside
  §sec:carrier-policy.
- **One mild looseness (TIGHTEN):** `η_{a,g,u}` has no `≤ N_{a,u} · z_{a,g}` link to MAWB
  activation; the LP relaxation will permit phantom ULD claims on inactive MAWBs.
- **Monotonicity invariant verified:** all CW-coefficient terms in the current objective are
  non-negative, so C.4c/d as `≥` inequalities is sound.

---

## Findings

### BUG-1 — Big-M for break-disaggregation is too tight; bans the round-up case
- **Severity:** BUG
- **Location:** §sec:lin-bucket (line 2768–2778); §sec:bigm-tightening table (line 2916–2923);
  §sec:domain C.14 line 2568 (BW domain).
- **Problem.** The break-disaggregation uses
  `BW_{a,g,b} ≤ M^BW_{a,g} · γ_{a,g,b}` with
  `M^BW_{a,g} = CW^ub_{a,g} = (1+ε) · Σ_k max(w_k, 167·v_k)`. The 3-inequality form is supposed
  to allow `BW_b = max(CW_{a,g}, break_b)` when `γ_b = 1`, including the round-up-to-higher-
  break case where `break_b > CW_{a,g}`. But if `break_b > CW^ub_{a,g}`, the upper-link bound
  `BW_b ≤ M` blocks `BW_b ≥ break_b`, making `γ_b = 1` infeasible.
- **Worked counter-example.** Single-HAWB MAWB with `w_k = 80`, `v_k ≈ 0`, ε = 0.05:
  CW^ub ≈ 84. Suppose the TACT card has breaks (45, $10), (100, $8), (300, $5). The IATA
  next-break-down minimum over breaks is `min(84·10, 100·8, 300·5) = min(840, 800, 1500) = 800`
  at γ_100. That selection survives because 100 ≤ M ≈ 84 is false — wait: break_100 = 100,
  M = 84, so 100 > M, so BW_100 ≤ 84 conflicts with BW_100 ≥ break · γ_100 = 100. **γ_100 = 1
  is infeasible.** The model can only pick γ_45, paying $840 instead of $800. Even worse
  with break_300: $5/kg × 300 = $1,500 is the lowest only when no closer break dominates,
  but the model can never even consider it.
  Actually the simpler check: as soon as **any** break in B_a exceeds CW^ub, the model
  bans that break, silently. For a 280 kg single-HAWB MAWB facing the §appendix-tact card
  (breaks 45, 100, 300, 500, 1000): CW^ub ≈ 294. Breaks 500 and 1000 are both unreachable.
  The appendix worked example shows the optimum at break 300 ($1260). Lucky here — 300 ≤ 294
  is false too (294 < 300), so **even the appendix's own worked example is infeasible under
  the current big-M.** The model would pick break 100 at $1344, contradicting the
  next-break-down rule the §appendix-tact section was added to validate.
- **Fix.** Use `M^BW_{a,g} = max(CW^ub_{a,g}, max_{b ∈ B_a} break_{a,b})`. Equivalently,
  pre-compute per-arc `B^max_a = max_b break_{a,b}` and use `max(CW^ub_{a,g}, B^max_a)` as
  the big-M on BW. Also update the C.14 domain on BW to the same widened bound.
- **Worked re-check post-fix.** 280 kg single-HAWB, breaks {45, 100, 300, 500, 1000}, M_new
  = max(294, 1000) = 1000. γ_300 = 1 ⇒ BW_300 ∈ [300, 1000] ⇒ optimizer sets BW_300 = 300,
  cost = 300·4.20 = $1260. Matches appendix. γ_500 = 1 ⇒ BW_500 = 500, cost = 500·3.80 =
  $1900. γ_1000 = 1 ⇒ BW = 1000, cost = 1000·3.20 = $3200. Minimum over γ = $1260 at break
  300. Correct.

### BUG-2 — `cost^MAWB_{a,g}` is undefined for `per_uld_pivot` + `equalized`
- **Severity:** BUG (ambiguity that admits two contradictory readings)
- **Location:** §sec:lin-bucket "per_uld_pivot, equalized settlement" paragraph (line 2814–
  2819); objective Σ over M (line 2641–2642); §sec:bsa-equalized-settlement (line 1170–1185).
- **Problem.** The text says: *"In-MAWB cost contribution is r_a · CW_{a,g} aggregated into
  chargeable(c) in C.13a. The pivot floor is handled at the contract level by A_c and over_c,
  not per-MAWB; C.13b is not generated for equalized arcs."* The objective then sums
  `Σ_{(a,g) ∈ M} cost^MAWB_{a,g}` over **all** MAWBs in M, including equalized ones.
  Two readings:
  - **Reading A:** `cost^MAWB_{a,g} = r_a · CW_{a,g}` for equalized arcs. Then the bucket-
    cost sum charges full `r_a · CW_{a,g}` per MAWB **plus** `r_c · over_c` for the
    overage — double-charges the overage portion.
  - **Reading B:** `cost^MAWB_{a,g} = 0` for equalized arcs (cost charged only through
    `r_c · over_c`). This matches the §sec:bsa-equalized-settlement text "Weight 0→A_c is
    free (sunk: already paid)" but contradicts the §lin-bucket sentence above.
- **Fix.** Choose Reading B and write it explicitly: "`cost^MAWB_{a,g} = 0` for equalized
  `per_uld_pivot` arcs; all cost on equalized arcs is realized through `r_c · over_c` in
  the objective and the C.13a allowance overage." Remove or rewrite the "r_a · CW_{a,g}
  aggregated into chargeable(c)" sentence — `chargeable(c)` is a constraint accounting
  variable (sum of CW), not a cost.

### BUG-3 — Forward-time-window propagation is under-specified at multi-outbound nodes
- **Severity:** BUG (under-specification; equivalence claim does not hold without
  per-outbound-arc state)
- **Location:** §sec:fwd-time-propagation (line 383–443), rules "Flight tail (POL) node" and
  "Flight head node"; the equivalence claim at line 437–438.
- **Problem.** The propagation maintains one window `[t^lo_n, t^hi_n]` per (k, n). The flight-
  tail rule reads "Admit a only if `t^lo_{n_1} + transit(k, a) ≤ CO*_a`. If admitted:
  `t^lo_{n_2} = ...`; `t^hi_{n_2}` collapses to the flight's effective cutoff." But:
  - `CO*_a` belongs to the *outbound* air arc leaving `n_2`, not to the inbound arc `a`
    indexed in the rule (the rule's `a = (n_1, n_2)` is the inbound arc). Either the rule
    means "for each outbound air arc `a'` at `n_2`, admit `a'` if `t^lo_{n_2} + 0 ≤ CO*_{a'}`"
    or it conflates inbound/outbound. The wording mixes them.
  - "`t^hi_{n_2}` collapses to the flight's effective cutoff" assumes a single outbound
    flight. If POL has K outbound flights with cutoffs CO*_{a'_1} < ... < CO*_{a'_K}, each
    outbound's admission criterion is different, and the t^hi-collapsed value differs per
    outbound. A single-window state cannot represent this without losing arcs.
- **Why the equivalence claim breaks.** The corresponding MILP big-M time-propagation family
  would have `t_k^{n_2} ≥ t_k^{n_1} + transit(k, a)` plus an outbound-arc-specific cutoff
  `t_k^{n_2} ≤ CO*_{a'} + M(1 - x_{k,a'})`. Each outbound arc gets its own cutoff inequality.
  The single-window graph-build representation is equivalent only if the propagation
  enforces admission per *outbound* arc (with a per-outbound t^lo check) — which the text
  does not require explicitly.
- **Fix.** Rewrite the rule as: "For each candidate outbound air arc `a' = (n_2, n_3)` leaving
  a POL node `n_2`, admit `a'` only if `t^lo_{n_2} ≤ CO*_{a'}`. Each admitted outbound arc
  yields its own downstream propagation; the per-node window is reset per outbound for the
  downstream subgraph rooted at `a'`." Equivalent to: propagate per (k, path-prefix) rather
  than per (k, node). The graph generator can implement this via BFS over arcs not nodes,
  and the resulting `A_k` is the union of all admissible paths.
- **Related notation defect.** In the same paragraph, `a` is used for both inbound ground
  arc and the index whose `CO*_a` is checked. NOTATION.

### TIGHTEN-1 — `η_{a,g,u}` not tied to `z_{a,g}` (phantom-ULD opportunity)
- **Severity:** TIGHTEN
- **Location:** C.14 domain (line 2564); C.5 (line 2394).
- **Problem.** `η_{a,g,u} ∈ [0, N_{a,u}]` with no `· z_{a,g}` factor. At LP relaxation, the
  optimizer can set η > 0 on a MAWB with `z_{a,g} = 0` (no HAWBs assigned). The C.13b-2
  pivot bound would then force pivot ≥ π · Ση cost on a phantom MAWB.
  In practice the integer optimal still sets η = 0 when z = 0 (since pivot cost is positive
  and there's no benefit), but the LP relaxation is loosened.
- **Fix.** Add `η_{a,g,u} ≤ N_{a,u} · z_{a,g}` to C.5 or C.14. Tightens LP, no integer
  change.

### TIGHTEN-2 — `t_k^{D_k^node}` lower bound is `t_k^{rdy,early}`, not `t_k^{rdy,early} +
min path transit`
- **Severity:** TIGHTEN (LP looseness; safe)
- **Location:** C.14 (line 2573).
- **Problem.** The lower bound on destination arrival is the cargo-ready time, which is far
  below any realizable arrival (real routes plus fallback all add transit). Tighter:
  `t_k^{D_k^node} ≥ min_{a ∈ A^last_k} arr_dest(k, a)`.
- **Fix.** Replace lower bound with `min_{a ∈ A^last_k} arr_dest(k, a)`. Tightens the LP
  envelope of `τ_k = max(0, t_k - Δ_k)` on the lower end.

### NOTATION-1 — `POD_k` used but never defined
- **Severity:** NOTATION
- **Location:** C.10a (line 2459, 2473); abstract / nomenclature.
- **Problem.** `POD_k` (the HAWB-specific port of discharge) is used in `A^last_k` and in
  the air-arc terminal case of `arr_dest`. It is never introduced in any nomenclature row
  or §sec:hawb-params table. `POD` appears as a generic node type (line 288, 312) but
  `POD_k` as a per-HAWB symbol is implicit.
- **Fix.** Add a row to §sec:hawb-params or §sec:variables: `POD_k ∈ N` — the HAWB's port-
  of-discharge node (terminal POD on the chosen route). Or rewrite C.10a using
  `head(a) = D_k^{POD}` with an explicit definition.

### NOTATION-2 — `Δ^post_k` defined inline in C.10a, not in any nomenclature
- **Severity:** NOTATION
- **Location:** C.10a (line 2474–2476).
- **Problem.** `Δ^post_k` (sum of post-POD ground transits) is introduced in a parenthetical
  inside a bullet. No nomenclature row. Used only here, but the per-HAWB nature is
  load-bearing for `arr_dest(k, a)`.
- **Fix.** Add a one-row note in §sec:hawb-params or the §sec:constraints symbol table.

### NOTATION-3 — `T_k^dead` used in Δ_k definition, never tabulated
- **Severity:** NOTATION
- **Location:** §sec:hawb-params line 783 ("Effective soft deadline Δ_k = min(T_k^dead,
  T_k^SLA) where T_k^dead is the shipper-contractual deadline if any; otherwise...").
- **Problem.** `T_k^dead` appears in the same row that defines Δ_k but has no row of its
  own; semantics ("shipper-contractual deadline") are entirely buried in that prose.
- **Fix.** Add a dedicated row for `T_k^dead` in §sec:hawb-params before Δ_k.

### NOTATION-4 — `ETD_f` and `ETA_f` used; only `ETA_a` is in any table
- **Severity:** NOTATION
- **Location:** Eq.~\ref{eq:active-embargoes} (line 1555); §sec:graph-construction line 292
  (`ETA_a`); §sec:tz-convention line 229; many §screening / §locked-commitments mentions.
- **Problem.** Per-flight ETD and ETA scalars are used in embargo activeness and elsewhere
  without an entry in any nomenclature table. They're plumbed as flight metadata (line
  956–960) but not formally enumerated as parameters.
- **Fix.** Add `ETD_f`, `ETA_f` to the flight-metadata description block in §sec:air-arc-
  params or §sec:variables.

### NOTATION-5 — `ac(f)`, `ac_type(f)`, `dgr(f)`, `per(f)`, `val_capable(f)`,
`avi_capable(f)`, `hum_capable(f)` used in eq:cargo-type-ok, never tabulated
- **Severity:** NOTATION
- **Location:** Eq.~\ref{eq:cargo-type-ok} (line 2222–2234); embargo §; lithium §
  (`ac_type(f)` row exists in §lithium but as "Defined: §sec:air-arc-params" where it is
  not actually defined).
- **Problem.** Per-flight capability predicates are used in pre-filter predicates but never
  introduced anywhere. The §sec:lithium nomenclature row points to §sec:air-arc-params for
  `ac_type(f)`, but §sec:air-arc-params's "Flight metadata" paragraph names "aircraft type"
  in prose without a formal symbol.
- **Fix.** Add a "Per-flight predicates" block to §sec:air-arc-params: `ac(f)`, `ac_type(f)`,
  `dgr(f)`, `per(f)`, `val_capable(f)`, `avi_capable(f)`, `hum_capable(f)` as boolean
  flight attributes (or `ac(f) → enum` with derived predicates).

### NOTATION-6 — `transit(k, a^fb_k)` undefined in the LaTeX
- **Severity:** NOTATION (correctness depends on the graph-construction MD filling the gap)
- **Location:** §sec:variables transit definition (line 1404) — covers ground + air;
  §sec:fallback-arc (line 459–473) defines arr_dest but not transit; the graph-construction
  MD line 537 supplies `transit(k, a^fb_k) = T_k^abs - t_k^rdy,early`.
- **Problem.** §sec:fwd-time-propagation uses `transit(k, a)` to advance windows for all
  arcs including the fallback arc, but the LaTeX never says what the fallback's transit
  scalar is. The MD has it; the LaTeX should mirror.
- **Fix.** Add a bullet to §sec:fallback-arc per-arc scalars: `transit(k, a^fb_k) = T_k^abs
  - t_k^rdy,early` (so forward propagation lands `t^lo_{D_k^node} = T_k^abs`).

### NOTATION-7 — `C^pu` uses contract-prefix `C` but is an arc set
- **Severity:** NOTATION
- **Location:** §sec:variables line 1412.
- **Problem.** `C` denotes BSA contracts; `C^eq` is the equalized-contract subset of `C`;
  but `C^pu = ⋃_c A_c^MAWB ⊆ A^MAWB` is a set of *arcs*. Prefix collision with the contract
  family. The naming pattern `A^pu` (subset of A^MAWB) would be consistent with `A^MFB`,
  `A^MAWB`, `A^coload`, `A^ground`, `A^cust`.
- **Fix.** Rename `C^pu → A^pu` (or `A^pup` for per-ULD-pivot). Touch all uses (C.5, C.5b,
  C.13b, C.14, §lin-bucket, §lin-pivot).

### NOTATION-8 — `g` overloaded as consolidation group and MIP gap
- **Severity:** NOTATION
- **Location:** §sec:carrier-policy line 2040 (`g` = Pass-1 relative MIP gap); §sec:variables
  line 1413–1414, 1416 (`g` / `g(k)` / `G_a`).
- **Problem.** Inside §sec:carrier-policy's "MIP-gap interaction" paragraph, `g` is the
  HiGHS relative MIP gap. Outside it, `g` is the consolidation group. Local clarity does
  not save a reader who skims.
- **Fix.** Rename the gap to `g^{mip}` or `ε^{gap}` inside §sec:carrier-policy; or restructure
  so consolidation `g` does not appear in any equation alongside the gap discussion.

### NOTATION-9 — `T^SLA_p` vs `T_k^SLA`: clean, but flag `D_k` vs `D_k^node` historical clutter
- **Severity:** NOTATION (clean now; defensive)
- **Location:** §sec:hawb-params (line 783 "Renamed from D_k to disambiguate from the
  destination node D_k^node").
- **Problem.** The rename note is a legacy artifact; future readers will trip on the inline
  history. The "destination door node" is now `D_k^{node}` and the soft deadline is `Δ_k`.
  Fine in current text.
- **Fix.** Drop the "Renamed from D_k..." parenthetical from the production version.

### NOTATION-10 — `chargeable(c)` is defined ad-hoc inside C.13a
- **Severity:** NOTATION
- **Location:** C.13a (line 2536–2538).
- **Problem.** `chargeable(c) ≜ Σ_{(a,g) : a ∈ A_c^MAWB, g ∈ G_a} CW_{a,g}` is defined
  inline as an aux symbol but never appears in §sec:variables or in the §sec:constraints
  nomenclature table. It's a derived quantity, but it is referred to as if a parameter.
- **Fix.** Either lift `chargeable(c)` to a §sec:variables row (derived/continuous) or fold
  the sum into C.13a directly without naming it.

### NOTATION-11 — `K^fb` is "post-solve derived" but used in objective accounting too
- **Severity:** NOTATION (minor)
- **Location:** §sec:variables (line 1422 "Post-solve set..."); objective (line 2664–2666)
  uses `Σ_k C^fallback · x_{k, a^fb_k}` (so the cardinality is implicit, not via K^fb);
  §output-diagnostics line 2954 uses `|K^fb|`.
- **Problem.** Fine in practice — `K^fb` is reporting-side. But the §sec:variables row
  reads as if it's a model symbol; it's purely diagnostic.
- **Fix.** Move `K^fb` to a separate "Post-solve derived quantities" sub-table (already
  exists at line 1488–1499); remove from the sets-and-indices table at line 1422 to avoid
  the implication it's an MILP set.

### NOTATION-12 — `C^pu` row says "Per-ULD-pivot arcs" but the LaTeX symbol is rendered with
contract semantics elsewhere
- **Severity:** NOTATION (covered by NOTATION-7 fix)
- **Location:** §sec:variables row 1412; appendix-tact appendix uses TACT which is
  `min_flat_breaks` not per_uld_pivot.
- Subsumed by NOTATION-7.

### NOTATION-13 — `chargeable(c)` summation range is over `g ∈ G_a` — fine, but the global G
is undefined
- **Severity:** NOTATION (very minor)
- **Location:** §sec:variables row "G" (line 1413: "Set of consolidation groups (range of
  g(·))").
- **Problem.** `G` is defined as the range of g. In practice this is only used implicitly
  via `G_a`. The row is technically correct but adds clutter; `G` is never directly
  iterated.
- **Fix.** Optional: drop `G` from the sets table; keep only `G_a`.

### NOTATION-14 — `base_freight_k` for percent_of_freight surcharges
- **Severity:** NOTATION / potential modeling gap (re-cast as NOTATION since math is
  consistent if base_freight_k is taken at face value)
- **Location:** §sec:surcharge Path A line 1247 ("percent_of_freight: rate · base_freight_k
  with percent_of_field = base_freight, pre-computed per arc").
- **Problem.** Under MAWB consolidation, the per-HAWB base freight is not well-defined
  before solve (density mixing depends on co-loaded HAWBs). Treating `base_freight_k` as
  a precomputed constant is an approximation, not the literal per-HAWB cost share. The
  text does not name the approximation.
- **Fix.** Add a one-line note: "`base_freight_k` is the per-arc per-HAWB freight under
  the assumption HAWB k is shipped alone on a same-rate MAWB; under consolidation the
  actual attributed share may differ. P1: solve-time attribution."

### NOTATION-15 — C.12 listed in "active families" but has no equation
- **Severity:** NOTATION
- **Location:** §sec:constraints opening sentence (line 2282); §sec:constraints C.12 (line
  2517–2523).
- **Problem.** C.12 is listed as active but its body is "Locks are resolved at preprocessing;
  the MILP body is lock-agnostic" — i.e., no constraint. Calling C.12 an active family is
  misleading; it is a placeholder for documentation continuity.
- **Fix.** Either remove C.12 from the active-family list, or annotate "C.12 — placeholder
  (handled outside the MILP)".

### NOTATION-16 — pivot is named "per-flight" but is per-MAWB-arc in the formulation
- **Severity:** NOTATION (potentially BUG depending on the operational claim)
- **Location:** §sec:bsa-params line 1158 ("the pivot binds on each flight of arc a
  independently"); C.13b (line 2547–2553).
- **Problem.** The prose says the pivot binds *per flight* (each leg). But `pivot_{a,g}` is
  one variable per (a, g) — one per MAWB-arc, not per leg. For a multi-leg MAWB-arc
  (interline through-MAWB, same-carrier through-cargo without rate split), the formulation
  charges one pivot floor, not one per leg.
- **Reading.** If multi-leg MAWB-arcs are by construction one-pivot offers (the carrier
  prices them as one rate against one pivot), the formulation is correct and the prose is
  loose. If multi-leg MAWB-arcs realistically face per-leg pivot floors, the formulation
  under-charges the take-or-pay.
- **Fix.** Either clarify "per offer (one pivot per MAWB-arc by definition; multi-leg arcs
  use one consolidated pivot from the contract)" or migrate `pivot_{a,g}` to `pivot_{a,g,f}`
  indexed by leg f, summed in the cost.

### NOTATION-17 — `transit(k, a)` indicator notation
- **Severity:** NOTATION (very minor)
- **Location:** §sec:variables line 1404.
- **Problem.** `transit(k, a) = δ_a + δ^cust_k · 1[a ∈ A^cust]` uses `\mathbbm{1}[·]` —
  fine, but `1` here is also used for binary RHS values in C.1. Consistency: `\mathbb{I}`
  or `\mathbf{1}` everywhere.
- **Fix.** Pick one indicator-function font and use consistently.

### NOTATION-18 — Cargo-ready window upper bound not enforced on `t_k` in the MILP
- **Severity:** NOTATION (documentation precision)
- **Location:** C.14 line 2573–2582.
- **Problem.** The text says "The cargo-ready upper bound on origin time is enforced at
  graph build via the forward-time-window propagation... no MILP variable `t_k^{O_k}`
  exists." Fine. But it then says `t_k^{D_k^node} ∈ [t_k^rdy,early, T_k^abs]`. The lower
  bound `t_k^rdy,early` is the *earliest* cargo-ready time; the destination arrival cannot
  realistically be that early. The bound is *valid* (any arrival ≥ cargo-ready) but
  uninformative.
- **Fix.** Either drop the lower bound (HiGHS infers from arr_dest and x), or tighten to
  `min_{a ∈ A^last_k} arr_dest(k, a)` (TIGHTEN-2).

### NOTATION-19 — `Π` (ULD interchange triples) defined in graph-construction nomenclature
but never used in the MILP
- **Severity:** NOTATION (low-risk; documentation)
- **Location:** §sec:graph-construction nomenclature line 298; §sec:through-uld line 1323.
- **Problem.** `Π` is documented as if a model symbol; it is consumed only by the graph
  generator and never appears in MILP equations.
- **Fix.** Move the `Π` row to a "Graph-generator inputs (not MILP)" sub-block, or annotate
  "graph-build only".

### NOTATION-20 — Indicator `\mathbbm{1}[a ∈ A^cust]` requires `\usepackage{bbm}`
- **Severity:** NOTATION (typographic safety; will compile fine since bbm is in preamble)
- **Location:** §sec:variables line 1404.
- Already loaded (line 10).

### NOTATION-21 — Eq.~\ref{eq:flight-uld-surcharge} double-iterates over `a`
- **Severity:** NOTATION (clarity, math is correct)
- **Location:** Eq.~\ref{eq:flight-uld-surcharge} (line 1257–1263).
- **Problem.** The outer sum binds `a` (line 1261: `Σ_{a ∈ A^MAWB : f ∈ legs(a)}`); the
  inner sum re-binds `a` implicitly inside `(a,g) ∈ M` — the `a` in the inner is meant to
  match the outer. As written it is technically a sum over the same `a` indexed twice.
- **Fix.** Rewrite as `Σ_{a : f ∈ legs(a)} Σ_{g ∈ G_a} Σ_{u ∈ U_a} η_{a,g,u}`. Same value,
  no double-binding.

### NOTATION-22 — `T_k^abs` "mandatory-finite" — what if the input is missing?
- **Severity:** NOTATION (data-contract clarity)
- **Location:** §sec:hawb-params (line 781); §sec:fallback-arc (line 469).
- **Problem.** Text says `T_k^abs` is mandatory-finite "ingestion applies a tenant-configured
  default (e.g. t_k^rdy,early + 30 d) when the shipper does not specify". But the model
  body assumes finiteness without any defensive check. If ingestion fails to populate and a
  HAWB enters the MILP with `T_k^abs = -∞`, the fallback `arr_dest` is undefined and the
  C.14 bound on `τ_k` (`max(0, T_k^abs - Δ_k)`) becomes nonsensical.
- **Fix.** Add a "Pre-MILP guard" note: ingestion asserts `T_k^abs` is finite and `≥ Δ_k`;
  failures surface as structured errors, not silent corruption.

### NOTATION-23 — `ε` overloaded: dunnage factor vs `ε^pref` cost-ceiling slack
- **Severity:** NOTATION
- **Location:** §sec:cw-density-mixing (line 656); §sec:carrier-policy line 2038, 2117.
- **Problem.** `ε` (no superscript) is the dunnage factor (~0.05, dimensionless multiplier).
  `ε^{pref}` is the Pass-2 cost-ceiling slack (USD). Different domains, different roles,
  similar glyph. Acceptable with the superscript distinguisher, but flag.
- **Fix.** Optional: rename dunnage to `ε^{dun}` for total disambiguation, or rename Pass-2
  slack to `Δ^{pref}`.

### NOTATION-24 — `μ_k` (value coefficient) vs `μ_a` (transit time): same glyph, different
roles
- **Severity:** NOTATION
- **Location:** §sec:hawb-params line 785 (`μ_k = value_k / V^ref`, dimensionless);
  §sec:air-arc-params line 891 (`μ_a` in hours).
- **Problem.** Two distinct quantities share the glyph distinguished only by subscript.
  Acceptable but mildly hostile to skimmers — especially because `μ_k` flows into
  `W_k = w_{sp(k)} · μ_k` in the objective while `μ_a` appears in transit propagation;
  both are "value-like" in different senses.
- **Fix.** Rename one. `ρ_k` for the value coefficient (rho = scaling factor) or `t_a` /
  `τ_a` for arc transit (but τ is taken for tardiness — see NOTATION-25). Cheapest
  rename: `μ_a → t^{air}_a`. (Cost: re-flow §graph-construction prose.)

### NOTATION-25 — `τ_k` (tardiness) consistent; no clash, just note
- **Severity:** OK
- `τ_k` is used only for tardiness throughout. No issue.

### NOTATION-26 — Active-constraint list inconsistency
- **Severity:** NOTATION (minor)
- **Location:** Abstract; §sec:constraints opening (line 2282); cover prompt mentions C.5,
  C.5b, C.5c separately; the opening lists "C.5/C.5b/C.5c" (combined). The §sec:objective
  monotonicity paragraph also lists "Wt, Wv, pivot, over, τ, pen" non-negativity invariant
  without naming the constraint families they belong to.
- **Fix.** Cross-check the active-family enumeration against the actual constraints
  defined in §sec:constraints.

### SAFE-TO-DEFER-1 — PWL grid {0, 4, 12, 36, 96} h known stale
- **Severity:** SAFE-TO-DEFER (per prompt; user has open question)
- **Location:** §sec:lin-tardiness (line 2857).
- The midpoint-gap quantification is mathematically correct: max gap is `((b-a)/2)^2 = 900`
  for the [36, 96] interval. The text correctly identifies this. Grid calibration is
  out-of-scope for this review.

### SAFE-TO-DEFER-2 — `t^hi_{n_2}` propagation through non-flight nodes uses cargo-ready
window width, but the window collapses at every flight head
- **Severity:** SAFE-TO-DEFER (collapses at first air arc, so impact is bounded)
- **Location:** §sec:fwd-time-propagation "Non-flight node" + "Flight head node" rules.
- After the first flight, `t^lo_{n_2} = t^hi_{n_2} = ETA_a`. So any window width carried
  from the cargo-ready window vanishes at the first flight head. For all-ground routes (no
  air arcs — unusual but the fallback arc is the only one), window collapse never
  happens and the t^hi propagation accumulates the cargo-ready width. Edge case worth
  acknowledging.

### SAFE-TO-DEFER-3 — `C^fallback` must dominate worst-case real cost + worst-case tardiness
- **Severity:** SAFE-TO-DEFER (calibration; flagged in the model as `CALIBRATION NEEDED`)
- **Location:** §sec:hawb-params (line 829–833).
- The text says `C^fallback` "should exceed `W_k · (T_k^abs - Δ_k)^2` plus any realistic
  sum-of-arc routing cost by at least an order of magnitude". For very high-value HAWBs,
  `W_k = w_p · value_k / V^ref` can be large; with `(T_k^abs - Δ_k)` of, say, 720 h,
  `W_k · 720^2 ≈ W_k · 5e5`. If `w_p = $1/h^2` and `value_k = $10M` (V^ref = $10K),
  `W_k = $1000/h^2` and `W_k · 720^2 = $5.2e8`. Default `C^fallback = $1M` does NOT dominate.
  The pruning argument in §sec:fwd-time-propagation ("Arcs whose end-to-end arrival > T_k^abs
  are dominated by the fallback") then fails for high-value HAWBs.
- **Fix.** Make `C^fallback` per-HAWB-scaled: `C^fallback_k = max(C^fallback_base, α · W_k ·
  (T_k^abs - Δ_k)^2 + β · max_a real_cost(k, a))`. Alternative: drop the dominance
  argument and let the model directly prune via path enumeration.

### SAFE-TO-DEFER-4 — destination-arrival sum requires "exactly one terminal arc activates"
- **Severity:** SAFE-TO-DEFER (implicit from flow conservation when POD_k is on the route)
- **Location:** C.10a (line 2466–2470).
- Flow conservation at `POD_k` guarantees exactly one inflow arc when POD_k is visited.
  Fallback arc is the alternative (skips POD_k entirely). So `|{a ∈ A^last_k : x_{k,a}=1}|
  = 1` always. The C.10a sum reduces to a single term. Correct, but a one-line note would
  help.

---

## Cross-checks (no issue found)

- **Monotonicity invariant** holds for all objective terms involving `CW_{a,g}`. C.4c/d as
  `≥` is sound.
- **C.5b uses w_k, not cw_k** — verified, prior bug fixed (line 2419 comment).
- **C.5b-v is independent of C.5b-w** — verified; light-bulky cargo binds on volume.
- **Eq.~\ref{eq:cw-ub}** is the tightest valid upper bound from C.4a/b/c/d (sum of max
  per HAWB times dunnage). Tight.
- **Pivot upper bound `max(CW^ub, π·ΣN)` in C.14** matches the two `≥` inequalities in
  C.13b. Tight.
- **Tardiness PWL outer-approximation** algebra (line 2845–2847) checks: `f̂(τ) =
  2τ̂·τ - τ̂²` is the correct tangent of `τ²` at `τ̂`.
- **`flat_rate` two-inequality `max`** is correct; `z=0 ⇒ CW=0 ⇒ both RHS = 0`.
- **C.13a equalized one-sided `max(0, ·)`** is the standard `over ≥ chargeable - A`,
  `over ≥ 0` pair.
- **Domain `CW_{a,g} ≤ CW^ub · z`** correctly enforces empty-bucket-zero.
- **Fallback arc + C.10a** correctly handle the rescue case via explicit union in
  `A^last_k`; no double-counting; no flow-conservation issue.

---

## Suggested next actions, ranked

1. **Fix BUG-1.** One-line change to `M^BW`. Verify with the §appendix-tact worked example
   (280 kg, break 300 winning at $1260) as the regression test.
2. **Resolve BUG-2** (equalized `cost^MAWB_{a,g}`) — pick the zero-bucket-cost reading and
   write it explicitly.
3. **Tighten BUG-3** propagation language — rewrite the flight-tail and t^hi rules to be
   per-outbound-arc, or restate the equivalence claim with the qualification that admission
   is per outbound.
4. Notation pass: add nomenclature rows for `POD_k`, `Δ^post_k`, `T_k^dead`, `ETD_f`, `ETA_f`,
   `ac(f)` and the per-flight capability predicates, `transit(k, a^fb_k)`. Rename `C^pu →
   A^pu`. Disambiguate `g` (group vs MIP gap) in §sec:carrier-policy.
5. Add TIGHTEN-1 (`η ≤ N · z`) — one row in C.5 or C.14.
6. Clarify NOTATION-16 (pivot scope: per offer, not per leg) — depending on which
   reading is intended.
