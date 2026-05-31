# Air MILP Spec — 3-Agent Critique (Session 15, 2026-05-23)

**Read this first on resume.** Verbatim findings from the 3-agent critique pass on
`air_milp_spec.md` v1. The fixes are applied in `air_milp_spec.md` v2; the
cluster-by-cluster fix sweep is in `SESSION_LOG.md` Session-15 entry; the
high-level decisions are in `CONTEXT.md` Session-15 critique-decisions block.
This file is the **receipts** — what each agent actually said, with severity tags,
diagnoses, and proposed fixes — so the trail survives even after the spec is
deleted post-Stage-2.

**Total findings:** 61 (25 + 20 + 16) plus Agent-3's base-scale estimate and
instrumentation suite.

**Status:** all CRITICAL and HIGH findings either applied to spec v2 or
explicitly deferred / out-of-scope (see `SESSION_LOG.md` Session-15 for the
triage). The 18 fix-shape clusters in the SESSION_LOG entry summarize what
landed where.

---

## Agent 1 — Notation & Formulation Correctness

**Charter:** catch bugs and inconsistencies in notation, set definitions,
indexing, domain consistency, and equation correctness. **NOT** architectural
design, performance, or linearization technique — those were Agents 2 and 3.

**Returned:** 25 findings. 1 CRITICAL, 8 HIGH, 6 MEDIUM, 8 LOW/NIT.

---

**1. CRITICAL — `D_k` symbol collision with destination node never cleanly disambiguated in constraints**
   Cite: §3.1 (parameter table + the parenthetical clarifier), C.10, C.11.
   Diagnosis: §3.1 introduces both `D_k` (scalar, h — effective soft deadline) and `D_k^{node}` (destination node) and notes they are "disambiguated by superscript." But in C.10 `τ_k ≥ t_k^{D_k^{node}} − D_k` the same letter `D` plays both roles inside a single expression, and the time-indexed variable is `t_k^{D_k^{node}}` which is a stacked-superscript construction that is awkward and easy to misread. The convention only works because the reader memorizes which `D` is which.
   Suggested fix: rename the scalar to `Δ_k` (or `S_k` for "soft deadline") and keep `D_k` exclusively for the destination node — eliminates the collision rather than papering over it.

**2. HIGH — `O_k` and `D_k^{node}` not declared as elements of `N_k`, only as "node"**
   Cite: §3.1 row "`O_k, D_k^{node}` | node | Origin door, destination door nodes."
   Diagnosis: C.1 indexes flow conservation over `n ∈ N_k` and treats `O_k`, `D_k^{node}` as special values of `n`, but §3.1 only types them as "node" (no set membership). For correctness need `O_k ∈ N_k` and `D_k^{node} ∈ N_k` after the Phase-1 pre-filter — otherwise the RHS of C.1 references nodes that may not be in the conservation index.
   Suggested fix: declare `O_k ∈ N_k` and `D_k^{node} ∈ N_k` explicitly in §3.1 (and note that pre-filter step 7 in §4 guarantees this).

**3. HIGH — `c_a^{handle}(k)` used in objective as a function of `k`, declared in §3.2 as a scalar per arc**
   Cite: §3.2 row `c_a^{handle}` ("USD per HAWB or per kg"); §7 objective term `Σ_{a ∈ A^ground, k ∈ K_a} c_a^{handle}(k) · x_{k,a}`.
   Diagnosis: The objective writes `c_a^{handle}(k)` (function-of-`k`) but the parameter table declares `c_a^{handle}` (subscript on arc only) with an ambiguous unit ("per HAWB or per kg" — pick one or split). The mapping from the per-arc declaration to the per-HAWB objective term is unspecified — is it `c_a^{handle}` (constant) when per-HAWB, and `c_a^{handle} · w_k` when per-kg?
   Suggested fix: split into `c_a^{flat}` (USD/HAWB) and `c_a^{kg}` (USD/kg); write the objective term as `(c_a^{flat} + c_a^{kg} · w_k) · x_{k,a}`. Or pick one basis for MVP.

**4. HIGH — `δ^{cust}_k` declared per-HAWB but never linked to a specific arc in C.6**
   Cite: §3.1 (`δ^{cust}_k`), §3.2 parenthetical ("`δ^{cust}_k` is per-HAWB on the customs-dwell arc"), C.6 statement.
   Diagnosis: C.6 says `transit(k,a) = δ_a` for ground arcs and adds parenthetically "(plus `δ^{cust}_k` for the customs-dwell arc)". But there is no formal predicate or set `A^{cust} ⊆ A^{ground}` defined that identifies *which* arc is the customs-dwell arc. The reader cannot evaluate `transit(k, a)` deterministically from the spec.
   Suggested fix: define `A^{cust} ⊆ A^{ground}` (or a flag `is_customs(a)`) and write `transit(k,a) = δ_a + δ^{cust}_k · 𝟙[a ∈ A^{cust}]`.

**5. HIGH — `head(a)`, `tail(a)`, `legs(a)`, `first leg of a` used as primitives but never formally defined**
   Cite: C.1, C.6, C.7, C.9; §2.1 only defines `legs(a)`.
   Diagnosis: `head(a)`, `tail(a)` appear in C.1, C.6, C.7, C.9 without ever being declared in §2 or §3. `first leg of a` in C.9 is English prose, not a defined function. Per the memory rule `feedback_define_notation_before_use.md`, every symbol must be defined before first use.
   Suggested fix: add to §2.1 nomenclature: `tail(a) ∈ N` (origin node of arc `a`), `head(a) ∈ N` (destination node), `first(a) ∈ legs(a)` (first physical flight leg of `a` in time order). Note arc direction convention: `a` goes from `tail(a)` to `head(a)`.

**6. HIGH — C.1 RHS sign convention contradicts standard MCNF and the arrow convention**
   Cite: C.1.
   Diagnosis: C.1 writes `Σ_{head(a)=n} x − Σ_{tail(a)=n} x = −1 if n=O_k, +1 if n=D_k^{node}`. With the standard convention that an arc *enters* its head, the LHS is `(inflow) − (outflow)`. At the source you have outflow but no inflow → LHS = `−1` *only if* the RHS reads `+1 at origin, −1 at destination` (i.e., supply at source = +1, demand at sink = −1) or you flip the LHS to `outflow − inflow`. As written the signs are inverted relative to standard MCNF; this is either a bug or a non-standard convention that requires a note.
   Suggested fix: flip the RHS signs to `+1 at O_k, −1 at D_k^{node}`, OR rewrite LHS as `Σ_{tail(a)=n} x − Σ_{head(a)=n} x` (outflow − inflow), and add a one-line convention note.

**7. HIGH — `pivot_{a,g}` lacks a definitional upper-link constraint when `min` of `r_a · max(CW, π·Ση)` is intended**
   Cite: §5.1 (`pivot_a,g` description "max(CW, π·Ση)"), C.13b, §7.3.
   Diagnosis: C.13b-1/2 give only `pivot ≥ CW` and `pivot ≥ π·Ση`. Since `cost = r_a · pivot_{a,g}` is being minimized and `r_a > 0`, the optimizer drives `pivot` down to the max of the two RHS — this is fine for per-flight when both RHS are activated. But for inactive MAWBs (`z_{a,g}=0`), `CW_{a,g}` is free of an upper link (no C.x forces `CW_{a,g} ≤ M · z_{a,g}`), so `pivot_{a,g}` could be nonzero even when nothing is booked. The only safeguard is C.2b → `z=0` ⟹ no HAWB ⟹ Wt=Wv=0 ⟹ `CW=0` at optimum. But the minimization argument is what closes the gap, not the constraint set itself.
   Suggested fix: add `CW_{a,g} ≤ M · z_{a,g}` (or `Σ_u η_{a,g,u} ≤ N_{a,u} · z_{a,g}` upper-link) to explicitly tie the bucket weights / ULD counts to MAWB activation.

**8. HIGH — `η_{a,g,u}` not tied to `z_{a,g}` (can claim ULDs on a non-instantiated MAWB)**
   Cite: §5.1 (`η`), C.5, C.5b, C.13b-2.
   Diagnosis: There is no constraint preventing `η_{a,g,u} > 0` when `z_{a,g} = 0`. C.5 sums `η` across `g` against `N_{a,u}` (no `z` link); C.5b puts `η` on the RHS of a capacity bound (so larger `η` is harmless unless the LHS sum is positive); C.13b-2 ties `pivot ≥ π·Ση`. Per-flight settlement cost `r_a · pivot` would drive `η = 0` at optimum when `CW = 0`. But for equalized settlement (per §7.3 the pivot floor "is handled at the contract level") `η` has *no* downward force in the objective, so `η > 0` on an inactive MAWB is not blocked.
   Suggested fix: add `Σ_u η_{a,g,u} ≤ N_{a,u}^{max} · z_{a,g}` (or any sufficiently large multiplier of `z`) to enforce that ULDs only exist on instantiated MAWBs.

**9. HIGH — `over_c` only appears in C.13a; relies on minimization to be tight, but `r_c` is undefined**
   Cite: §3.5 (`A_c` only), C.13a, §7 objective term `r_c · over_c`.
   Diagnosis: The objective references `r_c` (per-contract per-kg overage rate), but §3.5 declares only `A_c`. `r_c` is never defined as a parameter. It is conceptually distinct from `r_a` (per-arc rate from §3.3 `per_uld_pivot`) — multiple arcs share one contract `c`, so `r_a` for all `a` tagged to contract `c` should be the same value, but the spec never establishes that equality nor introduces `r_c` formally.
   Suggested fix: add `r_c` (USD/kg) to §3.5; or state `r_c := r_a` for any `a ∈ A_c^{MAWB}` with a consistency assumption.

**10. HIGH — C.13a uses `A_c^{MAWB}` indexed by `c` but the index `c` is never introduced as a set**
   Cite: §3.5 (parameter `A_c`); C.13a.
   Diagnosis: `c` is used as if it indexes a set of equalized-settlement contracts, but no set `C` (or `C^{eq}`) of contracts is declared in §2.1. The sum `Σ_{c ∈ equalized BSA}` in §7 has the same problem.
   Suggested fix: add `C` (set of contracts), `C^{eq} ⊆ C` (equalized-settlement subset), `A_c^{MAWB} ⊆ A^{MAWB}` (arcs tagged to contract `c`) to §2.1.

**11. MEDIUM — `min_chg_a` in C.4 / §7.1 multiplied by `z_{a,g}` but `min_chg` is per-arc, MAWB is per-`(a,g)`**
   Cite: §3.3 `flat_rate` row (`min_chg_a` USD); §7.1 `cost^{MAWB}_{a,g} = max(min_chg_a · z_{a,g}, m_a · CW_{a,g})`.
   Diagnosis: If two groups `g₁, g₂` both share arc `a` (so `z_{a,g₁} = z_{a,g₂} = 1`), the spec charges `min_chg_a` twice (once per active MAWB on the arc). This is probably the intended semantics (each MAWB is its own document with its own minimum), but the spec never states it. Reader can plausibly read `min_chg_a` as a per-arc one-time fee.
   Suggested fix: add a one-line note: "`min_chg_a` is applied per instantiated MAWB on arc `a`, not per arc — distinct groups consolidate to distinct MAWBs, each independently subject to the minimum."

**12. MEDIUM — `cap_a` interpretation inconsistent across C.5 (no use), C.5c (uses), and `per_uld_pivot` row**
   Cite: §3.3 `flat_rate`, `min_flat_breaks`, `per_uld_pivot` rows ("`cap_a` (kg, optional)"); C.5c.
   Diagnosis: `cap_a` is listed as an attribute for `flat_rate` and `min_flat_breaks` and is optional. C.5c bounds `Σ_k w_k · x_{k,a} ≤ cap_a` with `Σ_k` over `K_a` — but this sums actual weight across all groups on arc `a`, while the bucket cost model rates each group independently. Should `cap_a` bound per-MAWB chargeable weight (sum within `(a,g)`) or per-arc total actual weight (across `g`)? Spec asserts "actual weight per-arc" — fine, but inconsistent with billing units (`cap` quoted in kg but is it chargeable or actual?).
   Suggested fix: clarify `cap_a` as actual-weight per arc (across groups) vs. chargeable-weight per MAWB; pick one and write the constraint accordingly.

**13. MEDIUM — `B_a` defined as "weight-break segments" but its breakpoint structure is never specified**
   Cite: §2.1 (`B_a`), §3.3 (`min_flat_breaks` row with `(break_{a,b}, rate_{a,b})`), §7.2 linearization.
   Diagnosis: `break_{a,b}` is used in §3.3 and §7.2 without §2.1 declaring it. Is `break_{a,b}` the lower-weight threshold for break `b`? In ascending or descending order? §7.2 uses `BW ≥ break_b · γ_b`, which implies `break_{a,b}` is a lower bound on billed weight when that break is selected. The "min-over-flat-breaks" semantics from review item 7 implies each break is `(min-weight, rate)` and cost is `min_b rate_b · max(CW, break_b)`. State this.
   Suggested fix: add to §3.3: "`B_a` is ordered s.t. `break_{a,1} < break_{a,2} < …`; `break_{a,b}` is the minimum billed weight if break `b` is selected; `rate_{a,b}` is the per-kg rate."

**14. MEDIUM — `legs(a)` and `arcs(f)` referenced as inverses; only `legs` defined**
   Cite: §2.1 (`legs(a)`); §2.2 ("its inverse `arcs(f)`").
   Diagnosis: `arcs(f)` is named in §2.2 ("all per-flight capacity coupling lives through `arcs(f)`") but never declared in the nomenclature table. Then C.5 / C.5b / C.5c bind per-arc (not per-flight), so `arcs(f)` is never actually used — making §2.2's claim misleading. Either drop the `arcs(f)` mention or restore a per-flight coupling.
   Suggested fix: delete the `arcs(f)` reference in §2.2 since the spec deliberately drops flight-level capacity (per §3.4 and Q1) — `arcs(f)` is no longer needed.

**15. MEDIUM — `T^{SLA}_{sp(k)}` and `T^{SLA}_p` notation inconsistency**
   Cite: §3.1 row `T_k^{SLA}` ("= `t_k^{rdy,early} + T^{SLA}_{sp(k)}`"); §3.6 row `T^{SLA}_p`.
   Diagnosis: §3.1 uses `T^{SLA}_{sp(k)}` (subscripted by the product binding of HAWB `k`), §3.6 lists `T^{SLA}_p` as a per-product parameter. These are the same thing under the substitution `p ← sp(k)` but the visual asymmetry (per-HAWB notation in §3.1 introduced before the per-product source in §3.6) is the exact violation `feedback_define_notation_before_use.md` warns against — use is two pages before declaration.
   Suggested fix: move §3.6 row `T^{SLA}_p` to appear *before* §3.1 (or merge §3.6 into §3.1's preamble), so the per-product parameter is declared before the per-HAWB derived quantity uses it.

**16. MEDIUM — C.7 `MCT^{eff}_{a_{in}, a_{out}, k}` not declared**
   Cite: C.7.
   Diagnosis: `MCT^{eff}` appears in C.7 with no entry in §3. The parenthetical says "Detail level of MCT effective values is data-side" but the symbol itself must still be declared with its index set.
   Suggested fix: add a row in §3 (per-arc-pair-per-HAWB) for `MCT^{eff}_{a_{in}, a_{out}, k}` (hours, h), even if its computation is data-side.

**17. LOW — `pivot_{a,g}` written as `pivot_a,g` in §5.1 / §5.2 (missing braces)**
   Cite: §5.1 row 9 (`pivot_a,g`); §5.2 last bullet.
   Diagnosis: Subscripting hygiene — should be `pivot_{a,g}` to match every other doubly-indexed variable. Trivial, but it is exactly the kind of thing that bites in the LaTeX rewrite.
   Suggested fix: write `pivot_{a,g}` everywhere.

**18. LOW — Initial-condition constraint `t_k^{O_k} ≥ t_k^{rdy,early}` stated under both C.6 and C.8**
   Cite: C.6 ("Initial condition…"), C.8 (separately).
   Diagnosis: C.8 standalone repeats what C.6 already states as an initial condition. Harmless duplication, but worth deciding which is canonical.
   Suggested fix: drop the initial-condition line from C.6 (or drop C.8); keep one.

**19. LOW — Nomenclature tables present in §2 and §5 but §3 and §6 do not lead with one**
   Cite: §3.1–§3.6 (tables present but interleaved with prose); §6 (no nomenclature table).
   Diagnosis: `feedback_define_notation_before_use.md` says notation-introducing sections lead with a nomenclature table. §3 splits parameters into six sub-tables (good per-category but not one consolidated index); §6 introduces no new symbols formally but uses several functions (`head`, `tail`, `transit`, `first leg`) that should be in a table.
   Suggested fix: add a consolidated symbol index at the front of §3 (or as an appendix); add `transit(k,a)`, `head`, `tail`, `first(a)` to the §2.1 nomenclature.

**20. LOW — §12 mapping row "P.7 hard BSA take-or-pay min → Removed" but review notes say "removed for equalized only"**
   Cite: §12 row P.7; `air_review_notes.md` item 13 / item 9.
   Diagnosis: Review notes say hard P.7 is "removed for equalized contracts, replaced by the allowance mechanism," and per-flight take-or-pay is "existing P.10 unchanged." §12 says simply "Removed (item 4 / BSA design). Replaced by `A_c` allowance + C.13a." This conflates the two settlement bases. Per-flight take-or-pay still binds via `pivot ≥ π · Ση` (C.13b-2).
   Suggested fix: rewrite the row as "Hard period-minimum removed for equalized (replaced by `A_c` + C.13a); per-flight take-or-pay retained via C.13b."

**21. LOW — §12 row "P.10 pivot weight linearization → C.13b" but review notes call P.10 the "per-flight ULD physical capacity" elsewhere**
   Cite: §12 row P.10; review notes item 13 (P.10 used as "per-flight ULD weight cap"?), C.5b heading ("item 13-A bug fix").
   Diagnosis: There is `P.10` ambiguity in the review thread — §12's row names it "pivot weight linearization" but the C.5b heading calls itself the "item 13-A bug fix," and item 13 in the review describes P.3 (not P.10) as the cw_k → w_k fix. Verify the mapping table matches what the prior LaTeX actually had.
   Suggested fix: cross-check the prior LaTeX's P.3 vs P.10 numbering and align §12 mapping rows; the C.5b heading "item 13-A bug fix" wants to point at "prior P.3" not "prior P.10."

**22. NIT — "consolidable" / "non-consolidable" notation in §2.1 `g(k)` row mixes two formulas without a switch predicate**
   Cite: §2.1 row `g(k)`.
   Diagnosis: The row gives two cases for `g(k)` but the predicate ("consolidable" vs "VAL/HUM/AVI") is not formalized — depends on `cargo_class(k)`. A reader from §2 alone cannot evaluate `g(k)`.
   Suggested fix: write `g(k) = (cargo_class(k), HAWB-id(k))` if `cargo_class(k) ∈ {VAL, HUM, AVI}`, else `(cargo_class(k), screening(k), temperature(k))`.

**23. NIT — `δ` (dunnage) and `δ_a` (arc dwell) and `δ^{cust}_k` (customs dwell) all use the letter δ**
   Cite: §3.2 (`δ_a`), §3.1 (`δ^{cust}_k`), §3.6 (`δ` dunnage).
   Diagnosis: Three different uses of δ — dimensionless dunnage scalar, hours-per-arc, hours-per-HAWB. Distinguishable by subscript/superscript pattern but reads as overloaded.
   Suggested fix: rename the dunnage factor to `ε` or `ρ_{dun}`.

**24. NIT — §5.1 `γ_{a,g,b}` index condition reads "`a ∈ min_flat_breaks`" — set vs predicate**
   Cite: §5.1 `γ_{a,g,b}` row; C.14 same.
   Diagnosis: `min_flat_breaks` is a `rate_family` enum value, not a set. The index condition should read `rate_family_a = min_flat_breaks` or define a set `A^{MFB} = {a ∈ A^{MAWB} : rate_family_a = min_flat_breaks}` and index over that.
   Suggested fix: define `A^{MFB} ⊆ A^{MAWB}` (mirroring `C^{pu}`) and index `γ` over `(a,g) ∈ M, a ∈ A^{MFB}, b ∈ B_a`.

**25. NIT — §6.5 reference in §2.1 (`legs(a)` description) points to a section that doesn't exist**
   Cite: §2.1 row `legs(a)` ("see §6.5 commentary").
   Diagnosis: §6 stops at C.14. There is no §6.5; this is a forward-reference to a section that doesn't exist in the spec.
   Suggested fix: either add the commentary in §6 or change the reference to "§3.4 commentary" (which is where the no-`W_f`/`V_f` rationale lives).

---

**Agent 1 closing note:** Findings 1–10 are the load-bearing defects to address before the LaTeX rewrite. 11–16 are real correctness improvements. 17–25 are hygiene/clarity that compound under LaTeX precision.

---

## Agent 2 — Linearization & MILP Technique

**Charter:** review the MILP technique — linearization formulations, big-M correctness/tightness, LP-relaxation quality, redundancy, and standard MILP pitfalls. **NOT** architectural design or notation hygiene.

**Returned:** 20 findings. **2 CRITICAL** (same root cause: `min_flat_breaks` linearization banned the round-up-to-higher-break case). 9 HIGH. 5 MEDIUM. 4 LOW/NIT.

---

### CRITICAL

**1. CRITICAL — `min_flat_breaks` disaggregation does NOT compute `min_b`; it lets the solver pick the cheapest break unconditionally and is therefore wrong.**
Cite: §7.2 / `Σ_b γ = z`, four `BW` inequalities.
Diagnosis: The IATA "next-break-down" cost is `cost = min_b [rate_b · max(CW, break_b)]` — a *deterministic function* of `CW`, not a free choice. Under §7.2 the solver picks any single `γ_b = 1` and pays `rate_b · max(CW, break_b)`; since the model *minimizes* over `γ_b`, the solver always picks the cheapest break — including breaks **below** `CW`, where the inequality `BW_b ≥ CW − M(1−γ_b)` correctly forces `BW_b = CW` and so pays `rate_b · CW` instead of `rate_b · break_b`. That can be cheaper than the IATA-correct answer (since lower breaks have higher rates, but the rate × CW product can still be lower than higher-break rate × break). Either way the model is not enforcing IATA semantics — it is doing free disjunctive minimization over breaks. In some instances this *underprices* shipments; in others the solver luck-aligns with IATA.
Suggested fix: Either (a) precompute the IATA function offline as a PWL on `CW` and represent it with SOS-2 or convex-hull breakpoint formulation (cleanest — the function is piecewise linear, one segment per break, and is the lower envelope, hence concave-ish), or (b) restrict `γ_b = 1` to be admissible only when `CW ≤ break_{b+1}` (i.e. break `b` is the "next break down" from `CW`) by adding indicator constraints `CW ≤ break_{b+1} + M(1 − γ_b)`. Option (a) is standard and recommended; the underlying function is `f(CW) = min_b rate_b · max(CW, break_b)`, which is concave PWL — represent it with the standard λ-form on its breakpoints. The current 4-inequality form is the wrong tool.

**2. CRITICAL — `Σ_b γ_{a,g,b} = z_{a,g}` combined with `BW_b ≤ CW` and `BW_b ≥ CW − M(1−γ_b)` collapses to `BW_{b*} = CW` for the chosen `b*`, but `BW_b ≥ break_b · γ_b` then forces `BW_{b*} ≥ break_{b*}` — a *contradiction* if `CW < break_{b*}` and the solver picks a higher break.**
Cite: §7.2 inequalities 2 and 3.
Diagnosis: When `γ_{b*} = 1`: inequality (3) gives `BW_{b*} ≥ CW`, inequality (4) gives `BW_{b*} ≤ CW` → `BW_{b*} = CW`. Inequality (2) then demands `CW ≥ break_{b*}`. So the formulation *implicitly* forbids selecting a break above `CW`. That is the opposite of IATA — the whole *point* of break-down rating is that you pay for `break_{b*}` (the higher number) when `CW < break_{b*}` and the higher-break rate × `break_{b*}` is cheaper than current-tier rate × `CW`. Under §7.2 the solver is forced to pick a `b` with `break_b ≤ CW`, and then pays `rate_b · CW` — never the "round up to a higher break" outcome.
Suggested fix: The `BW_b ≤ CW` inequality is the offender — it should be `BW_b ≤ max(CW, break_b) · γ_b`, which is itself nonlinear. The clean reformulation: drop the 4-inequality disaggregation, model `BW_b = max(CW, break_b) · γ_b` directly using `BW_b ≥ CW · γ_b` (via `BW_b ≥ CW − M(1−γ_b)`, `BW_b ≥ 0`) and `BW_b ≥ break_b · γ_b`, with `BW_b ≤ M · γ_b` (force zero when not selected), and **drop `BW_b ≤ CW`**. Combined with the next-break-down admissibility restriction from Finding 1, this is correct. Or use the PWL recommendation from Finding 1, which sidesteps the whole structure.

### HIGH

**3. HIGH — Big-M values for time propagation are flagged as "use tight M" but no formula is given; the spec is one solver-input away from being arbitrarily loose.**
Cite: §8 final paragraph; C.6, C.7, C.9.
Diagnosis: For C.6 `t_{k}^{n'} ≥ t_k^n + transit(k,a) − M(1−x_{k,a})`, the tightest valid M is `T^{horizon} + transit_{max}`, but a per-arc-per-shipment tight value is `M_{C.6, k, a} = max(0, ub(t_k^n) + transit(k,a) − lb(t_k^{n'}))` where `ub(t_k^n) ≤ T_k^{abs} − μ_a` (latest possible time at `n` if `k` is still routable) and `lb(t_k^{n'}) ≥ t_k^{rdy,early}`. For C.9 cutoff: `M_{C.9, k, a} = max(0, ub(t_k^{tail(a)}) − CO_f^*)` — bounded by `T_k^{abs} − μ_a − CO_f^*`-equivalent. For C.7 MCT with `(2−x−x)`: same horizon-scaled value but doubled effective slack.
Suggested fix: Add a §8 sub-table with the per-constraint formula: `M^{C.6}_{k,a} = T_k^{abs} − t_k^{rdy,early}` (per-shipment, valid envelope); `M^{C.9}_{k,a} = T_k^{abs} − t_k^{rdy,early}`; `M^{C.7}_{k,a_{in},a_{out}}` similarly. These are loose but per-shipment, an order of magnitude tighter than horizon-global. Tightest values require LP-bound preprocessing — note as P1.

**4. HIGH — Per-MAWB `BW` big-M is undefined; the natural bound `M = max_k cw_k · |K_a|` is very loose and tanks the LP relaxation of §7.2.**
Cite: §7.2, `BW_b ≤ M · γ_b` and `BW_b ≥ CW − M(1−γ_b)`.
Diagnosis: `BW` is bounded above by `CW_{a,g}`, which is itself bounded by `(1+δ) · Σ_{k ∈ K_a, g(k)=g} max(w_k, v_k · 167)`. Using a global M (e.g. 100,000 kg) makes the LP relaxation set `γ_b ≈ CW/M`, fractional and useless for branching, *and* makes the `BW_b ≥ CW − M(1−γ_b)` constraint slack at the LP root.
Suggested fix: Per-MAWB tight M: `M^{BW}_{a,g} = (1+δ) · Σ_{k ∈ K_a, g(k)=g} max(w_k, v_k · 167)`. Use the same value for both `BW_b ≤ M·γ_b` and `BW_b ≥ CW − M(1−γ_b)`. Replace `M` with `CW^{ub}_{a,g}` (a precomputed scalar) — this is the natural and tight choice.

**5. HIGH — `CW = max(Wt, Wv)` monotonicity argument is correct *for the rate function* but breaks down on the equalized BSA term in C.13a if `r_c < 0` is ever permitted (or if a negative coefficient appears via rebates).**
Cite: §6 / C.4c, C.4d; C.13a.
Diagnosis: The relaxation `CW ≥ Wt, CW ≥ Wv` (instead of equality) is valid *only* because the objective is monotone non-decreasing in `CW_{a,g}` — true for the three rate families since `min_chg`, `m`, `rate_b`, `r_a · max(CW, pivot)` are all weakly increasing. The equalized BSA C.13a, `over_c ≥ Σ CW − A_c`, with cost `r_c · over_c`, *is* monotone in `CW` (more chargeable weight → more overage → more cost), so the monotonicity argument holds across all three families plus the BSA term. **However**: if a future "rebate-above-pivot" or "marketing credit" enters as a negative coefficient on `CW`, the relaxation would let the solver set `CW > max(Wt, Wv)` to harvest the rebate. Flag explicitly: the spec states the condition for one family in passing but doesn't enumerate it across the four cost contributors.
Suggested fix: Add a one-line invariant in §6.4 stating: "all five cost contributors (`flat_rate` aux, `min_flat_breaks` `Σ rate_b·BW_b`, `per_uld_pivot` `r_a·pivot`, equalized BSA `r_c·over_c`, sunk allowance accumulator) have non-negative coefficients on `CW_{a,g}`. If any negative-coefficient term is added later (rebates, marketing credits), C.4c–d must be tightened to equality via `CW = Wt + (Wv − Wt)⁺` PWL."

**6. HIGH — C.13a `over_c ≥ chargeable(c) − A_c` is one-sided; the relaxation lets `over_c` overshoot when no rate-tightening force exists.**
Cite: C.13a.
Diagnosis: The constraint plus `over_c ≥ 0` plus minimizing `r_c · over_c` in the objective is the standard `max(0, ·)` linearization, and is correct *iff* `r_c > 0` (so the solver prefers smaller `over_c`). Verified — `r_c > 0` always. **Tightness**: with `r_c > 0` this is the convex envelope, no improvement possible. Mark as correct. But the spec uses the informal `chargeable(c) = Σ_{a ∈ A_c^{MAWB}, g ∈ G_a} CW_{a,g}` — the indexing relies on `A_c^{MAWB}` being defined elsewhere (it isn't formalized in §2.1). Flag for the notation reviewer; here just note that the linearization is correct given the sum is well-formed.
Suggested fix: Linearization is fine. Formalize `A_c^{MAWB}` and `chargeable(c)` in §2.1.

**7. HIGH — C.3 (`x_{k,a} ≤ z_{a,g(k)}`) is *not* redundant with C.2a in the LP relaxation; keep it for tighter LP bound.**
Cite: §6 / C.2a, C.2b, C.3.
Diagnosis: C.2a is `x_{k,a} ≤ z_{a,g}` for `k` with `g(k) = g` — identical to C.3 modulo the fact that C.2a is *indexed by* `(a, g)` and C.3 is *indexed by* `(a, k)`. If C.2a is generated for every `(a, g, k)` with `g(k) = g`, then yes, C.3 is literally a duplicate. But if C.2a is generated only once per `(a, g)` *aggregated* (which the spec wording suggests — "For each MAWB `(a, g) ∈ M` and each HAWB `k ∈ K_a` with `g(k) = g`: `x ≤ z`") — these are the same constraints written twice. Verdict: **redundant in count, equivalent in LP relaxation strength**. Drop one.
Suggested fix: Drop C.3 — keep the C.2a form (it makes the MAWB-side indexing explicit). LP relaxation unchanged; presolve will remove duplicates anyway, but cleaner spec.

**8. HIGH — C.2b `z_{a,g} ≤ Σ_k x_{k,a}` is a *single aggregated* inequality; the disaggregated form `z_{a,g} ≤ x_{k,a} ∀ k` would be wrong (forces all HAWBs to use the arc when `z=1`), but a *better* form exists.**
Cite: C.2b.
Diagnosis: The aggregated form `z ≤ Σx` is the correct prevent-phantom-activation constraint. It is the *weakest* valid form. LP relaxation: at root, `z` can equal any fractional value up to the sum, which is fine here since C.2a `x ≤ z` is the binding direction. The aggregated form is standard and tight in this context (the convex hull of `z = 1 ⇔ Σx ≥ 1` is exactly `z ≤ Σx` + `x ≤ z` for each x). No improvement available.
Suggested fix: Keep as-is. Standard.

**9. HIGH — `pivot_{a,g}` has no explicit upper bound; combined with the BSA `r_a · pivot` objective term, the LP relaxation is fine, but branching benefits from a bound.**
Cite: C.13b, §5.1 (variable declaration), §6.14 domain.
Diagnosis: `pivot_{a,g} ≥ CW`, `pivot ≥ π · Ση`, minimize `r_a · pivot`. Since `r_a > 0`, LP sets `pivot = max(CW, π · Ση)` exactly — correct linearization. No upper bound needed for correctness. For branching: an explicit `pivot_{a,g} ≤ π · Σ_u N_{a,u}` (max possible if all allotment is used) helps presolve and dual bounds.
Suggested fix: Add domain bound `pivot_{a,g} ≤ max(CW^{ub}_{a,g}, π_a · Σ_u N_{a,u})` in §6.14 for solver-friendliness.

**10. HIGH — `η_{a,g,u}` (integer ULD count) has no per-MAWB upper bound; only the *aggregate* cap C.5 bounds it across groups. Add per-MAWB `η ≤ N_{a,u}` for branching strength.**
Cite: §5.1, C.5, §6.14.
Diagnosis: `η_{a,g,u} ≤ Σ_{g'} η_{a,g',u} ≤ N_{a,u}` is implied by C.5 + non-negativity, but the implied bound is not visible to branching heuristics until the aggregate is binding. An explicit `η_{a,g,u} ≤ N_{a,u}` per MAWB is a redundant valid inequality that tightens the LP relaxation locally and gives presolve a tighter integer bound on each `η`. Cost: |M| × |U| inequalities, cheap.
Suggested fix: Add `η_{a,g,u} ≤ N_{a,u}` as an explicit domain bound in §6.14 (or as a redundant inequality in C.5). HiGHS will use this for variable bounds tightening immediately.

**11. HIGH — C.5b (per-ULD weight/volume sum) is a *per-MAWB* aggregate bound, which is **looser** than per-ULD-instance bin-packing.**
Cite: C.5b-w, C.5b-v.
Diagnosis: The constraint says "total HAWB weight on the MAWB ≤ ULD-type-summed capacity" — but a single 1500 kg HAWB physically must fit in *one* LD3 (≤1588 kg), not split across two LD3s. The current formulation *allows* fractional fit: e.g. 2 LD3 (`Σ W_u·η = 3176`) holds two 1500 kg HAWBs even though each HAWB needs a dedicated LD3 (true) — fine here; but it *also* allows one 3000 kg HAWB on 2 LD3 (`Σ ≤ 3176`), which is physically impossible (cargo can't be split). The spec explicitly notes this is by design (no HAWB-to-ULD assignment), but the looseness is real.
Suggested fix: Document the looseness in §6.5 with a worked counterexample. The fix (HAWB-to-ULD assignment with bin-packing) is heavy — a `y_{k,u,i}` variable for each HAWB, ULD type, ULD instance — and is correctly out of MVP scope. For MVP, add a pre-filter check at graph construction time: any HAWB with `w_k > max_u W_u` or `v_k > max_u V_u` is infeasible on per-ULD-pivot arcs (push to co-load or rescue). This catches the worst case cheaply without a bin-packing MILP.

### MEDIUM

**12. MEDIUM — `flat_rate` `c_{a,g} ≥ min_chg · z` correctly multiplies by `z` (so cost = 0 when bucket inactive), but `c_{a,g} ≥ m · CW` does not need `z` because `CW = 0` when `z = 0`. Confirm.**
Cite: §7.1.
Diagnosis: When `z = 0`, C.2a forces `x_{k,a} = 0` for all `k ∈ K_a` with `g(k) = g`, which via C.4a–b forces `Wt = Wv = 0`, and C.4c–d allow `CW = 0` (the LP relaxation picks `CW = 0` since cost is monotone). So `c_{a,g} ≥ m · CW` correctly gives `c ≥ 0` when bucket is empty. The `c ≥ min_chg · z` gives `c ≥ 0` when `z = 0`. Combined: `c_{a,g} = 0` when empty. Correct.
Suggested fix: No change. Add a one-line confirmation in §7.1: "When `z_{a,g} = 0`, both inequalities yield `c_{a,g} ≥ 0`, so empty buckets contribute zero cost."

**13. MEDIUM — `Σ_b γ = z` for `min_flat_breaks` correctly handles the empty bucket only if the `BW_b` constraints are corrected (Finding 2). Currently, with `z = 0` → all `γ = 0` → `BW_b ≤ 0` and `BW_b ≥ CW − M`. The second is trivially satisfied; the first forces `BW_b = 0`. Cost = 0. Correct.**
Cite: §7.2.
Diagnosis: Empty-bucket behavior is fine in the current §7.2 *if* the broken disaggregation from Finding 2 is repaired. The `Σ_b γ = z` form is the standard "exactly-one IF active" formulation and is correct here.
Suggested fix: After fixing Finding 2, re-verify empty-bucket behavior; the current `Σγ = z` mechanism is correct.

**14. MEDIUM — C.6 time propagation with `M(1−x)` produces an unbounded lower bound on `t_k^n` when `x = 0`; combined with `τ_k ≥ t_k^{D_k^{node}} − D_k`, the LP relaxation could let `t_k^{D_k^{node}} = 0` (its lower bound) and `τ_k = 0`, even when the path is infeasible.**
Cite: C.6, C.10.
Diagnosis: This is normal big-M behavior. When `x_{k,a} = 0`, C.6 becomes vacuous (`t ≥ −M + transit`); the only binding force on `t` is C.8 (`t ≥ t^{rdy,early}`) and downstream C.6 chains. In the LP relaxation with fractional `x`, the bound becomes `t ≥ ... − M(1 − x_frac)`, which is *much* looser than integer — letting `τ_k = 0` even though the actual integer solution has tardiness. This is a known weakness of big-M time propagation; the fix is either disaggregated formulations (per-path) or extended-formulation event-time variables, both heavy.
Suggested fix: Accept the LP slack for MVP. Document that LP relaxation bounds will be optimistic on tardiness for shipments whose path is long; this matters for solver branching but not for the optimal integer solution. If branching becomes slow, P1 upgrade: time-indexed formulation or per-arc dual time variables.

**15. MEDIUM — C.7 MCT `M(2 − x_{in} − x_{out})` is the standard "active only when both arcs used" formulation; it's correct but loose. The disaggregated form is tighter.**
Cite: C.7.
Diagnosis: `M(2 − x_{in} − x_{out})` is slack when *either* arc is unused (M absorbed by one). The disaggregated alternative — two constraints `t_{out} ≥ t_{in} + MCT − M(1 − x_{in})` and `t_{out} ≥ t_{in} + MCT − M(1 − x_{out})` — is equivalent at integer points but tighter at the LP root (each constraint independently active). Standard tradeoff: 2× constraint count vs LP tightness.
Suggested fix: For MVP keep the single constraint. If LP relaxation is loose at scale, swap to the disaggregated form — straightforward.

**16. MEDIUM — `over_c` should have a domain upper bound for solver-friendliness: `over_c ≤ Σ CW^{ub}_{a,g}` over arcs tagged to contract `c`.**
Cite: C.13a, §6.14.
Diagnosis: `over_c` has no upper bound declared. With `r_c > 0`, the LP correctly pulls `over_c` to its lower bound `max(0, chargeable − A_c)`. An explicit upper bound helps presolve and is cheap.
Suggested fix: Add `over_c ≤ Σ_{a ∈ A_c^{MAWB}, g ∈ G_a} CW^{ub}_{a,g}` in §6.14.

### LOW

**17. LOW — `τ_k` has no domain upper bound; with the linear `+w·τ_k` term and `w > 0`, the LP correctly minimizes, but a domain bound `τ_k ≤ T_k^{abs} − D_k` is tighter and free.**
Cite: C.10, §6.14.
Diagnosis: `τ_k` is non-negative and minimized; LP pulls it to lower bound. An explicit upper bound `τ_k ≤ T_k^{abs} − D_k` (or 0 if the SLA exceeds the absolute deadline) is implied by C.11 + C.10 but worth stating.
Suggested fix: Add `τ_k ≤ max(0, T_k^{abs} − D_k)` in §6.14.

**18. LOW — Sign check on objective: all five terms are positive coefficients on non-negative variables, minimization. Correct.**
Cite: §7.
Diagnosis: Bucket cost (positive), co-load (positive `m·cw·x`), ground handling (positive), BSA overage (`r_c · over_c`, both positive), tardiness (`w · τ`, both positive). All summed, minimized. No sign error. The sunk `r_c · A_c` correctly dropped as a constant — does not affect argmin.
Suggested fix: None.

**19. LOW — Per-flight pivot C.13b is a clean two-inequality `max` linearization; correct and tight given `r_a > 0`.**
Cite: C.13b-1, C.13b-2.
Diagnosis: Standard. LP exactly achieves `pivot = max(CW, π · Ση)`. Tight.
Suggested fix: None.

### NIT

**20. NIT — §8 linearization summary table claims `min_flat_breaks` uses "4-inequality disaggregation `BW_b`" — this is the broken form per Findings 1 and 2. Update after fix.**
Cite: §8 table row 3.
Diagnosis: Stale once Findings 1/2 are addressed.
Suggested fix: Rewrite table row to reflect the corrected formulation (PWL or restricted-disaggregation).

---

**Agent 2 closing note:** Findings 1 and 2 are the critical issues — the `min_flat_breaks` linearization as written does not implement IATA next-break-down rating. Everything else is tightness or domain-bound refinement; no other linearization is incorrect.

---

## Agent 3 — Simplification & Tractability at Scale

**Charter:** estimate problem size, identify what binds first at scale, recommend simplifications, dominance, decomposition, instrumentation. **NOT** architecture, notation, or linearization correctness.

**Returned:** 16 findings + base-scale estimate + walking-skeleton instrumentation suite + minimum-viable-subset recommendation.

---

```
[1. HIGH] Variable count for x_{k,a} dominated by per-shipment subgraph survival, but survival is uninstrumented
   Cite: §4, §13.
   Observation: The whole binary budget hinges on |A_k|_avg, yet the spec's only number — "|A_k|_avg ≈ 20" — is a placeholder with no measurement. With 7 pre-filter predicates each cutting 30–60% of arcs (per scalability.md §2.3), realistic survival is 5–20% of |A|. At |K|=100, |A|=500: 5% survival → ~2,500 x binaries; 20% survival → ~10,000. The 4× spread completely changes the solver story (HiGHS handles 2.5k easily; 10k pushes into the "needs presolve to save it" zone).
   Recommendation: Make per-shipment |A_k| and per-predicate drop-rate the first walking-skeleton metric. Log |A_k| histogram and per-predicate survival to a structured `pre_filter_stats.jsonl` on every solve. Refuse to scale up |K| until this number is empirical.
```

```
[2. HIGH] Time-propagation C.6 is the densest constraint family and grows as |K|·|A_k|_avg with one big-M per row
   Cite: §6 C.6, §8.
   Observation: C.6 generates one constraint per (HAWB, arc) pair — ~2,500–10,000 rows at MVP, ~25k–150k at Phase-2 — each a big-M with a single binary. This is typically the LP-relaxation-degrading family in fixed-charge MCNF. The spec correctly flags per-constraint tight M but offers no estimate of how loose the relaxation will be. C.7 hub MCT compounds: it's per (HAWB, hub node, inbound arc, outbound arc) — combinatorial in hub fan-out.
   Recommendation: Add `lp_gap_by_family` instrumentation — break the LP-vs-MIP gap by constraint family after the first solve. If C.6/C.7 dominate the gap (as expected), the tight-M derivation in §8 needs aggressive per-arc, per-shipment tightening (M_{C.6}^{k,a} = T_k^abs − t_k^{rdy,early} − μ_{shortest path} is the tightest defensible). Hub MCT in C.7 should be folded into the deconsol-arc δ_a (the spec already notes this — make it the default, not the exception).
```

```
[3. HIGH] No dominance pre-filter for offers within a (a, g) bucket
   Cite: §13.
   Observation: Multiple arcs can serve the same physical O-D under different offers (TACT/NAC/BSA/spot/co-load) — the spec acknowledges these are not symmetric (different rates) but the optimizer enumerates all of them. With 5 offers/lane typical and many bucket combinations, a substantial fraction are cost-dominated for every reasonable shipment density profile (e.g., a TACT offer with min_chg=$X is dominated by another TACT with same break grid and uniformly-lower rates). The spec's §13 admits `scale-option-dominance` needs re-derivation; no derivation is offered.
   Recommendation: Add bucket-level dominance pre-filter as a hard Phase-1 step (post-survival, pre-MILP). Form: for arcs (a_1, a_2) with same (O, D, rate_family, μ ≤ μ_1, all break_b in B_2 covered by B_1 with rate_{a_2,b} ≤ rate_{a_1,b}, and cap_{a_2} ≥ cap_{a_1}), drop a_1 from A^MAWB for any g that survives a_2. Cite as `scale-bucket-dominance`. Estimate 20–40% offer pruning at MVP.
```

```
[4. HIGH] Aggregation of identical HAWBs (consolidation_mode=preprocess) should be considered for MVP
   Cite: §10 deferred, §13.
   Observation: At Phase-2 (|K|=500–1500), many shipments share identical (g, O, D, window, sp). Two HAWBs with the same g, O, D, deadline tier are interchangeable from the MILP's standpoint — they will route identically. Treating them as a single "super-shipment" with weight w_k₁+w_k₂ collapses |K| dramatically. The spec defers this. At Phase-2 load, this is likely the biggest single tractability win available.
   Recommendation: Promote `consolidation_mode = preprocess` from deferred to walking-skeleton-instrumented. Don't enable by default; add a metric `aggregation_reduction = (|K_aggregated| / |K_raw|)` measured during walking skeleton — if this is ever < 0.7 on realistic batches, ship a config flag and turn it on for Phase-2.
```

```
[5. MEDIUM] z_{a,g} and γ_{a,g,b} count are bounded by |M| but |M| growth is not estimated
   Cite: §2.1, §13.
   Observation: |M| = Σ_a |G_a| ≤ |A^MAWB| · |G|. At MVP with ~8 distinct g values realistic (per graph doc §4.2 worked example), and |A^MAWB| ~250 (half of |A|), |M| could be 200–2000 depending on group concentration. The spec's "|M| ≈ 100–300" is optimistic. γ count is |M_{min_flat_breaks}| · |B_a| ≈ 0.3·|M|·6 ≈ 360–3,600. At Phase-2 this is 5x larger.
   Recommendation: Add `bucket_activation_distribution` metric — log distribution of |G_a| per arc on walking-skeleton runs. If empirical |G_a|_avg > 2, recompute base-scale projections.
```

```
[6. MEDIUM] η_{a,g,u} integer variables are the most expensive variable type but no count estimate provided
   Cite: §5.1, C.5b.
   Observation: General integers (not binaries) make branch-and-bound much harder. Count is |M_{C^pu}| · |U_a|_avg. For per_uld_pivot arcs (BSA), this could be 30–100 buckets × 2–4 ULD types = 60–400 integer vars at MVP. Domain is bounded (η ≤ N_{a,u}), small in practice, but the branching tree grows fast in presence of integers with the ceiling/floor branches HiGHS uses.
   Recommendation: Add explicit per-bucket bound `η_{a,g,u} ≤ N_{a,u}` (currently only Σ_g is bounded by C.5; per-g should be too — tighter LP). Instrument `integer_branch_count` to see if these dominate the search.
```

```
[7. MEDIUM] Decomposition opportunity (Phase-1 graph H) is not addressed
   Cite: §13, scalability.md §3.1.
   Observation: The prior LaTeX had a "supply graph H" decomposition (commodities sharing a flight/contract/ULD form a connected component; disjoint components solve independently). Under the O-D-arc graph this concept still applies — two HAWBs share an "edge" in H iff they share at least one arc a ∈ A_k₁ ∩ A_k₂ AND g(k₁) = g(k₂) (so they could coexist on a MAWB) OR they share a per-contract allotment N_{a,u}. Typical mid-market batches decompose into 3–10 components by geography. This is the cheapest big win at Phase-2 scale.
   Recommendation: Add `component_decomposition_count` as walking-skeleton metric — even though the walking skeleton solves monolithically, log what components *would* be. If consistently > 3 at Phase-2-like batches, decomposition is mandatory before Phase-2 ship.
```

```
[8. MEDIUM] legs(a) is now data-only but the spec still treats it as a live model object
   Cite: §2.1, §3.4.
   Observation: With Q1 resolved (no flight-level W_f, V_f), `legs(a)` is only used for two things: μ_a aggregation (precomputable at graph build) and CO_f* lookup at C.9 (also precomputable as CO_a*). The spec leaves `legs(a)` and `arcs(f)` as set-valued maps in §2.1, which suggests they live in the MILP. They don't — both can be flattened to per-arc constants.
   Recommendation: Push legs(a) and arcs(f) entirely into the data-prep layer. The MILP sees μ_a (a scalar) and CO_a* (a scalar) per arc. `legs(a)` and `arcs(f)` should not appear in §2.1 nomenclature for the MILP — only in the graph-construction doc. Drop |F| from the MILP scale model; only |A| matters at solve.
```

```
[9. MEDIUM] C.7 hub MCT constraint is combinatorial-pair density when arcs aren't folded into deconsol-dwell
   Cite: §6 C.7.
   Observation: The spec says "most MCT is absorbed into the deconsol-arc's δ_a" but C.7 still exists as a constraint family for the residual case (same-MAWB through-connection, interline through-ULD). The residual count is per (HAWB, hub node, inbound air arc, outbound air arc) — at a busy hub with 5+ inbound and 5+ outbound air arcs per shipment subgraph, that's 25+ constraints per HAWB-hub pair. The spec doesn't bound the residual.
   Recommendation: Eliminate C.7 from the MILP body entirely by mandating that all hub transitions go through a deconsol-dwell arc (folding all MCT effects into δ_a, including same-MAWB through-connection MCT as 0 on a synthetic "through-MCT" arc). C.7 becomes redundant with C.6. Saves a constraint family and an Mc7 derivation.
```

```
[10. MEDIUM] C.13a BSA equalized accumulator couples buckets globally, hurting decomposability
   Cite: §6 C.13a.
   Observation: chargeable(c) sums CW_{a,g} across all arcs tagged to contract c. Even shipments with disjoint A_k subgraphs can be in the same component of H purely via shared BSA contracts. This is a real constraint, not a modeling artifact — but it bites decomposition.
   Recommendation: When implementing decomposition, BSA-contract coupling is the typical breaker — measure `bsa_coupling_fraction = (|components with shared BSA| / |total components|)` in the walking-skeleton instrumentation. If high, plan for Lagrangian relaxation of just C.13a as the Phase-2 trick (dualize the global allowance, get per-component independence).
```

```
[11. LOW] No per-shipment top-K offer pruning at the bucket level
   Cite: §13.
   Observation: For a given HAWB k, only the top ~3 offers per arc class (cheapest TACT for this density, cheapest BSA bucket given current g, cheapest co-load) ever appear in the optimal solution. Enumerating all 5 offers per O-D pair × all g groups creates buckets that will never activate.
   Recommendation: Add a per-shipment "candidate offer reduction" step — for each k, restrict A_k further by keeping only top-3 offers per (O, D, rate_family) ranked by c(cw_k) at point-estimate density. This is a heuristic (may cut optima in pathological mixing cases) so make it a flag, not default. Run in shadow mode (compute reduction, don't apply) during walking-skeleton to measure how often the dropped offers actually appear in solutions.
```

```
[12. LOW] No symmetry residue, confirmed — but flag the cross-MAWB equivalence at hubs
   Cite: general.
   Observation: Bucket formulation eliminated MAWB-count symmetry (good). Attribute-tuple g eliminates within-arc bucket symmetry (good). One residual: at a hub with CFS-H, the choice of "which deconsol-dwell arc to traverse" may be a single arc (no symmetry) or, if the data layer creates multiple parallel deconsol-dwell arcs (e.g., one per inbound-outbound MAWB pair), it's symmetric. Confirm graph layer creates exactly one deconsol-dwell arc per hub-CFS.
   Recommendation: Add a unit-test invariant in walking-skeleton: at each (CFS-H, HAWB) pair, |deconsol-dwell arcs in A_k| ≤ 1. If > 1, surface as graph-construction bug.
```

```
[13. LOW] C.3 is admitted-redundant and should be dropped, not "kept for solver friendliness" pending review
   Cite: §6 C.3.
   Observation: C.3 is implied by C.2a per the spec's own admission. Redundant constraints sometimes tighten LP relaxation; just as often they slow presolve and waste rows. With ~2,500–10,000 x binaries, C.3 adds the same number of rows.
   Recommendation: Drop C.3 from the walking skeleton; add `enable_c3_redundant` as a flag for A/B testing once the model runs. Don't ship redundant constraints "just in case" — measure.
```

```
[14. MEDIUM] §13 refresh — concrete forms suggested
   Cite: §13.
   Observation: Spec admits three Tractability items are stale. Concrete refresh:
   - `scale-y-aggregation`: now `scale-hawb-aggregation` — collapse HAWBs with identical (g, O, D, deadline tier, sp) to super-shipments before MILP. See Finding 4.
   - `scale-option-dominance`: now `scale-bucket-dominance` — per-MAWB-candidate dominance pre-filter (Finding 3). Form: drop arc a' ∈ A^MAWB if ∃ a ∈ A^MAWB with same g-set, μ_a ≤ μ_{a'}, cap_a ≥ cap_{a'}, rate_a(w) ≤ rate_{a'}(w) for all w in feasible bucket-weight range.
   - `strat-v2-mawb-rescale`: delete (correctly identified as obsolete).
   Recommendation: Fold these into the LaTeX rewrite §Tractability section verbatim.
```

```
[15. LOW] Cost minimization in C.4c–d ("CW = max" via two ≥) relies on monotone cost — verify in the implementation
   Cite: §6 C.4c–d.
   Observation: Spec correctly notes CW = max(Wt, Wv) at optimum because cost is monotone in CW. True for all four rate families, but only because the cost terms are weakly increasing. If a future rate family introduces a discount-above-threshold (concave-then-flat), this breaks silently — CW could be free-set above max(Wt, Wv) to "buy" a tier without paying for it.
   Recommendation: Add an invariant test in walking-skeleton: post-solve, assert CW_{a,g} == max(Wt_{a,g}, Wv_{a,g}) within tolerance. Catches monotonicity violations on rate-family extensions.
```

```
[16. MEDIUM] Walking-skeleton sequencing — minimum viable subset
   Cite: general.
   Observation: 14 constraint families is too many for the first runnable model. Tractability questions are answered by a subset.
   Recommendation: Walking-skeleton implements:
   - C.1 flow, C.2 MAWB activation (drop C.3), C.4 CW aggregation, C.5c per-offer cap, C.6 time prop, C.9 cutoff, C.10 soft tardiness, C.11 hard backstop, C.14 domain.
   - Rate families: flat_rate and coload_per_kg only. Skip min_flat_breaks (defers γ/BW family) and per_uld_pivot (defers η/C.5/C.5b/C.13).
   - Defer: C.5 (allotment), C.5b (ULD physical), C.7 (fold into δ_a), C.12 (locks — preprocess only), C.13 (BSA accumulator).
   This is ~6 constraint families, ~3 variable families (x, z, CW + Wt + Wv, τ, t_k^n). Tests the core tractability questions (x-binary scaling, C.6 density, LP gap, big-M tightness) without the per_uld_pivot tail.
```

### Base-scale estimate (Agent 3, concrete numbers)

At MVP-production (|K|=100, |A|=500, |M|=200, |F|=200, |A_k|_avg=15 assumed 10% survival, |A^MAWB|=250, |min_flat_breaks arcs|=80, |per_uld_pivot arcs|=40, |U_a|=2.5, |G_a|_avg=1.5):

- **Binaries:** x_{k,a}: 100·15 = 1,500. z_{a,g}: ~300. γ_{a,g,b}: 80·1.5·6 = 720. **Total ~2,500 binaries.**
- **Integers (general):** η_{a,g,u}: 40·1.5·2.5 = 150 integer vars. Domain ≤ N_{a,u} (small, typical ≤10).
- **Continuous:** t_k^n: 100·25 nodes ≈ 2,500. CW + Wt + Wv: 3·300 = 900. BW: 720. pivot, over, τ: ~250. **Total ~4,400 continuous.**
- **Constraints:** C.1: 100·25 = 2,500. C.2a/b: 2·1,500 = 3,000. C.4: 4·300 = 1,200. C.5+C.5b+C.5c: ~500. C.6: 1,500 (densest binary-coupled). C.7: ~500 residual if not folded. C.9: ~300. C.10/C.11: 200. C.13: ~80. **Total ~10,000 constraints.**
- **Expected HiGHS solve time order:** **seconds to a few minutes** on a single-component, well-tuned MIP. At |K|=500 (Phase-2): ~12,500 binaries, ~50k constraints — **minutes to ~1 hour** monolithic; decomposition mandatory. At |K|=1500: monolithic infeasible without column generation.
- **First-binding family:** C.6 time-propagation (LP relaxation quality), followed by C.4 density-mixing if cargo profile is light/bulky-heavy. Memory not a concern until Phase-2.

### Instrumentation to add to walking-skeleton (Agent 3)

- **Per-shipment subgraph survival.** Log |A_k| and per-predicate drop counts (the 7 §4 predicates). Output: `pre_filter_stats.jsonl`, one record per HAWB per solve.
- **Bucket activation distribution.** Log |G_a| histogram on every arc in A^MAWB after Phase-2; log activated bucket count Σz_{a,g} post-solve.
- **LP-vs-MIP gap by constraint family.** After solving LP relaxation and the MIP, compute the per-family relaxation tightness: (z_MIP − z_LP) attributed by removing each family from the LP and re-solving. Output: `lp_gap_breakdown.json`.
- **Big-M slack distribution.** For each time-propagation constraint (C.6, C.7, C.9), log the realized slack in the MIP solution; if median slack > 0.5·M, the M is loose — surface as `bigm_slack_warning`.
- **Component decomposition shadow.** Compute connected components of H (HAWBs sharing arcs or BSA contracts) without acting on them; log count + max component size. Indicator for when to ship decomposition.
- **Aggregation shadow.** Compute super-shipment equivalence classes (identical g, O, D, deadline tier) without acting; log `aggregation_potential = 1 − |classes|/|K|`.
- **Solve-time-by-phase.** HiGHS presolve / LP root / B&B / cuts breakdown via HiGHS callbacks. Tells you where the time actually goes — guides next optimization.
- **Solution invariants.** Post-solve assertion suite: CW = max(Wt, Wv), τ_k = max(0, lateness), Σ_g η ≤ N_{a,u}, no z = 1 with empty bucket. Catches silent monotonicity bugs.

---

## Cross-cutting overlap (clusters)

Agents 1 and 2 independently flagged several of the same defects:

- **`min_flat_breaks` linearization bug** — A2 #1, #2 (CRITICAL on the same root cause). Fix: drop `BW_b ≤ CW`. Applied.
- **Missing upper-link bounds** — A1 #7 (`CW ≤ M·z`), A1 #8 (`η ≤ N·z`), A2 #9 (`pivot` upper bound), A2 #10 (`η ≤ N_{a,u}` per-MAWB), A2 #16 (`over_c` upper bound), A2 #17 (`τ_k` upper bound), A3 #6 (`η ≤ N_{a,u}`). All one cluster, fixed in C.14 domain expansion.
- **Contract set undefined** — A1 #10 (`C`, `C^{eq}`, `A_c^{MAWB}`), A2 #6 (formalize). Same fix.
- **Drop C.3 redundant** — A2 #7, A3 #13. Confirmed by two agents, dropped.
- **`legs(a)` push to data-prep** — A1 #14, A3 #8. Confirmed by two agents, applied.
- **Drop C.7 hub MCT** — A3 #9 (recommendation), implied by A3 #2 (folding into δ_a). Applied.

## Where the fixes landed

- `model/air_milp_spec.md` v2 — all CRITICAL and HIGH applied (with a few exceptions explicitly deferred / out-of-scope; see SESSION_LOG.md Session 15).
- `SESSION_LOG.md` Session-15 entry — cluster-by-cluster fix sweep with each cluster mapped to applied vs deferred.
- `CONTEXT.md` Session-15 critique-decisions block — high-level decision log.
- `TEST_PLAN.md` §10 — walking-skeleton observability section, built from Agent 3's 8-metric instrumentation suite.
- `CONTEXT.md` Stage 4 ladder — refined v1/v2/v3/v4 per Agent 3 finding #16.

## What was not applied (intentional)

- A2 #14 (LP-relaxation slack on C.6 time propagation) — accepted as known weakness of big-M time propagation; heavy fix (time-indexed formulation) deferred to P1.
- A2 #15 (C.7 disaggregated MCT form) — moot, C.7 removed entirely (Cluster J).
- A3 #11 (top-K offer pruning) — recommended as shadow-mode metric only; deferred to walking-skeleton instrumentation phase.
- Agent 1 NITs and stylistic items 22–25 — mostly applied; some folded into the new spec v2 nomenclature reorg.
