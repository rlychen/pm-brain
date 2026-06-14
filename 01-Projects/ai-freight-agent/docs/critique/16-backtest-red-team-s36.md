# Critique 16 — Backtest Methodology Red-Team (Session 36)

**Standing review agent: Backtest Methodology Red-Team.** Read-only audit of the experimental
design as of S35's §13 v4 redesign. The question is **not** "is the code correct" (the other two
standing agents cover that) — it is **"is the experiment honest?"** A flaw here produces a savings
number that looks real but is an artifact of how the simulation was built. Attacked as a hostile
reviewer trying to get the paper rejected.

**Docs audited:** `arrival_only_replan_methodology.md` (§10 / §12 D-A9..16 / §13 v4 D-A18..24),
`backtest_methodology.md` v0.5, `product_thesis.md`, `data/synthetic/air_generator.py` (current),
`src/scenario_db.py` (RNG streams + ledger DDL), `src/components/air_milp.py` (spot/conservation
surface), critiques 11 and 13. **State of the build:** F1 / 2c / arms / scorer are UNBUILT. The
spot CW-cap (D-A19), route-based fallback, supply-freeze-across-arms, and `capacity_ledger`
**writers** do not exist yet. So this red-team attacks the *design's falsifiability*, and flags
which attacks the **not-yet-written** code must defeat with a specific instance/test, not assertion.

---

## Verdict summary (ranked by threat to the headline)

| # | Attack | Verdict | What settles it |
|---|---|---|---|
| R1 | **L2=0-fraction can make the headline a permutation artifact, and §13 set no kill threshold** | **LANDS** | Pre-register an L2=0-fraction ceiling + a distinct-reshuffle-event floor; gate the headline on both |
| R2 | **The loose-corner null is structurally guaranteed to pass → the falsifiability gate is near-vacuous** | **LANDS** | Show the loose corner can produce L2 *meaningfully > 0* on *some* instance, i.e. that the gate has teeth; otherwise it tests nothing |
| R3 | **Conservation is unproven on the new binding corner (integer multinomial → zero-count flights + α-lumpiness)** | **LANDS** (open) | The binding-capacity + mid-tender + 2-arc-reshuffle fixture *at a zero-count/low-α cell*, asserting the global per-step identity |
| R4 | **Supply "frozen across arms" is asserted; no arm-invariance test exists, and M₁ re-running graph-gen is a live re-screen channel** | **PARTIALLY LANDS** | A cross-arm byte-identical capacity-vector test + a test that M₁'s re-screen cannot admit capacity M₀'s screen excluded on the *same* state |
| R5 | **L2 = recourse is asserted, not decomposed; consolidation-timing + re-screen + mix-shift can masquerade as reshuffle** | **PARTIALLY LANDS** | The 3-way split (D-A12) gated, AND an M₁′ arm pinned to `C(M₁′)==C(M₀)` exact, on a binding instance |
| R6 | **Fallback @ 1.5× worst-spot-route can still dominate L2 (fallback-avoidance ≠ reshuffle) and is mis-conditioned in live fixtures** | **PARTIALLY LANDS** | Reshuffle-share floor (≥50%) gated on separated CI; live `FALLBACK_COST` must be the per-instance value, not $1M/$100k |
| R7 | **π_hind floor "holds by construction" hides a consolidation/time-scalar bug class** | **DEFENDED-with-condition** | Floor is sound *iff* D-A13 single-time-scalar holds under region→region; re-verify walk≡scalar on multi-O/D |
| R8 | **Baseline M₀ (D-A23) may be hobbled — or M₁′ control not actually pinned** | **DEFENDED-on-design** | Design is fair; the risk is implementation drift — `C(M₁′)==C(M₀)` per-draw is the tripwire |
| R9 | **Demand/schedule leakage: tripwires exist for the old fixed-lane world, not for region→region multi-O/D** | **PARTIALLY LANDS** | Re-run both tripwires after D-A24 lands; add an airport-choice leakage case |

The three that genuinely threaten the **publishability of the number** are **R1, R2, R3**. R4–R6 threaten
**attribution** (is it really recourse, is it really reshuffle). R7–R9 are bug-class guards.

---

## R1 — The "L2=0 fraction" can make the headline a permutation artifact, and no kill threshold is set. **LANDS**

§13 / D-A23 says "report the fraction of draws with L2 = 0 as a diagnostic (if large, the arrival
permutation — not the engine — is driving the result)." This is the right instinct and it is **the
single most dangerous honesty hole in the whole design**, because it is named but **not gated**.

The attack: at proof scale `total_N ≈ 10` ULD positions over dozens of flights, with fixed-N demand
(~15–30 HAWBs) on a 7-day horizon (~0.7 HAWB/day/lane, critique-13 N15). On most seeded arrival
permutations there is simply **no contention event** — supply and demand happen not to collide on a
binding flight — so M₀ and M₁ produce identical plans and L2 = 0. The headline mean L2 is then
carried by a **small minority of draws** where a collision happened to occur. That is not "the engine
creates value"; that is "we sampled until a few instances were contentious." The mean over R
replications is then dominated by which permutations the seed happened to draw.

Why it lands as written: §13 reports the fraction but **states no threshold beyond which the headline
is not credible**. A reviewer asks "what L2=0 fraction would falsify the claim that the engine, not
the seed, drives the result?" and the methodology has no answer. A 90%-zero result with a large mean
on the 10% is reported with the *same* headline as a 10%-zero result with a uniform mean. Those are
completely different epistemic objects.

Compounding: the prior rounds already flagged the upstream causes — N15 (thin demand, "if < ~5
distinct reshuffle events it's anecdote"), C5/N14 (waves manufacture contention), M-B7 (one bump ≈
the whole signal at 15–30 HAWBs). §13 v4 did **not** resolve these; it added the diagnostic without
the gate, and the new integer-multinomial supply with **zero-count flights** *increases* the variance
of whether any given seed is contentious (a flight can draw 0 positions, so whether scarcity bites is
now also a supply-draw lottery, not just an arrival lottery).

**What settles it (must be in the proof before the number is published):**
1. **Pre-register an L2=0-fraction ceiling** at the headline cell (proposal: if > ~50% of draws are
   exactly L2=0, the headline is "engine value is intermittent / instance-rare," not a savings rate).
2. **Pre-register a distinct-reshuffle-event floor** (N15: ≥ ~5 distinct cross-cycle reshuffle events
   behind the reported mean, else label it anecdote).
3. **Report the L2 *conditional on a contention event occurring*** alongside the unconditional mean —
   the two numbers tell different stories and the deck must not conflate them.
4. This is the argument for moving the headline to the **forwarder-scale instance** (§11) where
   contention is structural, not a seed lottery. The integer-`total_N≈10` regime is where R1 bites
   hardest.

---

## R2 — The loose-corner null is structurally guaranteed to pass, so the falsifiability gate has no teeth. **LANDS**

§13 / D-A10 (rev v4) **retired the dedicated negative-control cell** (critique-11 C2's mandatory
abundant-capacity × early-arrival construction) and replaced it with: "the abundant-capacity ×
even-supply × early-arrival corner of the (κ,α,λ) sweep — already computed — carries a pre-registered
`|L2| < CI` pass condition." The stated benefit: falsifiability at zero extra construction.

The attack — this is a **weaker** falsifiability test than the one it replaced, and may be vacuous.
A legitimate negative control must be a regime where the engine *could* produce L2 but *should not*.
The constructed C2 cell was adversary-resistant precisely because it was built to be a clean null. The
loose **corner of the sweep**, by contrast, is *guaranteed* to show L2 ≈ 0 **for a structural reason
that has nothing to do with the engine being correct**: at abundant capacity + even supply + early
arrival there is no binding flight, so M₀ and M₁ are trivially identical for the **same** reason R1
identifies — no contention event ever fires. A gate that passes because "nothing ever binds" does not
demonstrate the engine *correctly* finds zero value; it demonstrates the instance is empty. A
*broken* M₁ (one that never reshuffles at all) **also** passes this gate. So `|L2| < CI` at the loose
corner is satisfied by both the true engine and a no-op engine — it cannot discriminate.

The retired C2 cell was stronger because the construction could include a *latent* opportunity the
engine must correctly decline; the sweep corner has no such latent opportunity by construction. §13
traded a discriminating null for a non-discriminating one and called it "restores falsifiability." It
does the opposite at the corner.

Worse: the regret floor `C(π_hind) ≤ C(M₁)` was **demoted to "labeled self-check, holds by
construction"** (D-A10 rev), explicitly **"not the falsifiability guard."** So §13 removed the
constructed control AND demoted the floor, leaving the **only** falsifiability guard a corner gate that
a no-op engine passes. The thesis can no longer demonstrably fail in a way that distinguishes a working
engine from a dead one.

**What settles it:**
1. **Keep the loose-corner gate but add a discriminating positive control**: a *constructed* small
   instance with a known reshuffle opportunity and a hand-computed L2* > 0, asserting the engine
   recovers it (within tie-break tolerance). Falsifiability needs both a null the engine passes
   **and** a positive the engine must hit — the corner gives only the former, and a degenerate one.
2. Demonstrate the corner gate has **teeth**: show that *perturbing* the corner toward tightness makes
   L2 cross the CI (i.e. the gate is on a real gradient, not a flat dead zone). If L2 is structurally
   pinned to ~0 across a wide neighborhood of the corner, the `|L2|<CI` pass is meaningless.
3. If the loose corner cannot exhibit L2 ≠ 0 for *any* reachable perturbation, the gate is vacuous and
   the constructed C2 control must be reinstated — §13's "zero extra construction" saving is false
   economy.

---

## R3 — Conservation is unproven on the new binding corner the integer supply creates. **LANDS (open)**

`backtest_methodology.md` §6 is explicit and correct: a capacity double-spend is "a phantom saving
indistinguishable from a real one," the invariant is the **per-arc, per-step conservation identity**,
and — its own words — **"this invariant has never been exercised"** because the Stage-0.5 spikes had no
binding capacity and re-solve from full capacity each step. Critique-11 C5 sharpened it: conservation
must be a **global per-step identity across all arcs** + a **per-shipment move-journal**, because the
DEFERRED bump is a *cross-arc atomic move* (release on arc(d*), claim on arc(d*+1)) and a per-row
CHECK cannot see the phantom-free intermediate state.

The attack, sharpened by §13 v4: the new **integer multinomial** supply (`Multinomial(total_N,
Dirichlet(α))`) deliberately produces **zero-count flights** and, at **low α**, severe lumpiness — i.e.
it manufactures exactly the **binding corners** where the conservation identity is most likely to be
violated and the violation is most valuable (a phantom slot on a contended arc is precisely where L2 is
generated). Yet:
- The `capacity_ledger` table exists as **DDL with zero writers** (critique-13 N3). Nothing advances
  the ledger today.
- The conservation identity has been verified on **no** instance — let alone one with a zero-count
  flight forcing all demand onto a single binding contracted arc while later HAWBs are still arriving.
- The 2-arc reshuffle fixture (C5) is **unbuilt**.

So the load-bearing correctness invariant of the entire savings claim is currently **asserted in prose
and enforced nowhere**, and the supply model that S35 introduced makes the untested corner *more*
reachable, not less. A double-spend here is not a crash — it is a **silent inflation of L2** that looks
exactly like reshuffle value.

**What settles it (the single most important test in the proof — backtest §6 says so itself):**
1. A **binding-capacity + mid-horizon-tender** instance constructed at a **low-α / zero-count-flight**
   cell: one contracted flight draws all positions for a (dest, day), neighbors draw 0, a HAWB tenders
   on the binding flight while a later HAWB is still arriving and contends for it.
2. Assert the **global per-step identity across all arcs** (not per-row) + the **per-shipment
   move-journal** each step, for both M₀ and M₁, with M₁ executing an actual d*→d*+1 bump.
3. The 2-arc reshuffle fixture must show the slot is conserved *as it moves* — no step in which the slot
   is in two ledgers or in neither.
4. Until this passes on the binding corner, **no L2 number is trustworthy** regardless of how the rest
   of the design reads.

---

## R4 — "Supply frozen across arms" is asserted; no arm-invariance test exists, and M₁'s per-step re-screen is a live capacity-leak channel. **PARTIALLY LANDS**

§13 "Unchanged invariants": the drawn integer network capacity vector is "**bit-identical** across
H₀/M₀/M₁/M₁′/π_hind, computed once in generation, persisted, read-only (no arm recomputes it)." Good in
principle. Two live concerns:

**(a) No test enforces arm-invariance of the capacity vector.** The generator draws `supply_counts`
once per scenario (`_draw_network_supply` on the `supply` stream) — correct. But the *arms* (2c,
unbuilt) each load the scenario and run the MILP; nothing yet asserts the capacity vector each arm
*sees at solve time* is byte-identical after a sequence of tenders/decrements. D-A16's freeze is a
generation-time property; the **per-arm ledger** (N3, unbuilt) is where it can silently diverge — e.g.
if M₁'s re-solve reads a ledger state M₀ never reached. The CRN-across-axes test (vary κ/α ⇒ demand
byte-identical) is the *upstream* guard and is sound; the *downstream* guard (capacity vector identical
across arms at matched sim-time) is unspecified.

**(b) M₁ re-runs graph generation each replan step — a declared re-screen (backtest §4) — and §13's
region→region routing makes the admitted route set genuinely larger and schedule-dependent.** The
methodology *defines* re-screening into L2 on purpose, which is defensible. But the **schedule
lookahead tripwire** (backtest §5) was written for the **static-schedule, fixed-lane** world. Under
D-A24 the optimizer now *chooses the origin and dest airport*, so M₁'s re-screen on fresher state can
admit an **airport-pair route** M₀'s once-at-commit screen did not consider — and if that route's
capacity was *also* re-derived rather than read from the frozen vector, M₁ would be spending capacity
M₀ structurally couldn't see. The tripwire that would catch this (inject a future-only flight; assert
bit-identical plan) does **not** currently perturb the **airport-choice** channel.

**What settles it:**
1. A **cross-arm capacity-vector byte-identity test**: after any tender sequence, assert every arm
   reads the same frozen `N_f` per flight at the same sim-time (the per-arm ledger only *subtracts*
   from the frozen vector, never re-draws it).
2. A test that M₁'s per-step re-screen on identical state admits **identical** capacity to M₀'s — i.e.
   re-screen changes the *admissible-route set as schedules move*, never the *capacity* on a given arc.
3. **Re-run both lookahead tripwires after D-A24 lands**, adding an **airport-choice leakage case**
   (inject a future-only flight on an *alternate* origin airport; assert the plan at t is bit-identical).
   The existing demand-row tripwire cannot see a schedule/airport leak.

---

## R5 — "L2 measures cross-cycle reshuffling" is asserted, not decomposed. **PARTIALLY LANDS**

§13 / D-A23 claims `L2 = C(M₀) − C(M₁)` "measures cross-cycle reshuffling (= replanning), not
within-batch consolidation a naive greedy would leave on the table." The mechanism is right (M₀ optimally
consolidates *within* a cycle; only cross-cycle reshuffle differs). But "measures" is a **design claim
the proof must demonstrate**, and several non-reshuffle effects ride the same `C(M₀) − C(M₁)` channel:

- **Consolidation-timing artifact.** M₁'s open-book solve can co-consolidate two HAWBs revealed in
  *different* cycles that M₀ (which only consolidates *each cycle's* newly-revealed set, never
  disturbing priors) cannot. That is real replan value — but it is **consolidation timing**, not the
  canonical "bump DEFERRED to free a slot for EXPRESS" reshuffle the thesis sells. The two are
  conflated unless decomposed.
- **Re-screen mix-shift.** M₁ re-screens each step (R4b); a refreshed admissible set can lower cost via
  a route that became feasible, independent of any reshuffle.
- **Tardiness-penalty leakage.** Critique-11 C4 / D-A12: if the C.10 quadratic penalty leaks into
  `realized_cost`, L2 is dominated by avoided-penalty (objective-steering, not cash). Critique-13 §D
  reports the penalty is currently excluded — good — but this must be **asserted on the scorer**, not
  trusted.

§13's only decomposition instrument is the D-A12 3-way split (`L2_reshuffle` /
`L2_fallback_avoidance` / tardiness-delta) and the M₁′ control arm. **Both are unbuilt.** As written,
the claim "L2 = recourse value" is asserted; the test that would separate reshuffle from timing/screen/
penalty is deferred.

**What settles it:**
1. **Gate the headline on `L2_reshuffle` with a separated CI** (D-A12), not on raw `L2`, with the
   pre-registered reshuffle-share floor (≥50%).
2. **Build the M₁′ arm** (priors pinned to M₀'s feasible set) and assert `C(M₁′) == C(M₀)` per draw —
   any gap is solver/tie-break leakage netted out (D-A11). This is the instrument that proves the delta
   is *recourse freedom*, not formulation noise.
3. Report **consolidation-timing value separately** from slot-reshuffle value, or explicitly fold both
   under a renamed "cross-cycle replan value" and stop selling it as the slot-bump story specifically.

---

## R6 — Fallback @ 1.5× worst-spot-route can still dominate L2, and live fixtures are mis-conditioned. **PARTIALLY LANDS**

§13 / D-A19 sets fallback = `1.5 × [top-spot-rate · CW · max-air-legs + ground]`, graph-derived,
well-conditioned (replacing $1M / 2×). The conditioning improvement is real and correct (critique-13
N1). Two residual attacks:

**(a) Fallback-avoidance is not reshuffle.** Even at 1.5×, a single avoided fallback is ~1.5× a full
route ≈ several × a marginal reshuffle saving. If a tight/lumpy cell's L2 is driven by M₁ *avoiding a
fallback* M₀ incurred (rather than reshuffling within real routes), the headline is
capacity-rescue value, not the reshuffle thesis. D-A12's split exists to catch this; it is **unbuilt
and ungated**. Until `L2_reshuffle` carries the headline with a separated CI, the number can be
fallback-avoidance wearing a reshuffle label.

**(b) Live fixtures still feed the wrong fallback to HiGHS.** The generator's `compute_fallback_cost`
now computes a per-instance value (good), but `tpeb_air_instance.FALLBACK_COST = 100_000.0` and the
**tests hardcode `1_000_000.0`** (`test_air_milp.py:43`, `test_air_graph.py:417`). Any 2c path that
seeds from the static module constant, or any e2e that inherits the test constant, reintroduces the N1
conditioning defect: at $100k–$1M incumbent, the `mip_rel_gap=1e-4` absolute slack swamps the $50–$300
routing decisions whose *difference* is L2. **L2 is a difference of two MIP objectives**, so a
conditioning error the size of the signal corrupts it silently.

**What settles it:**
1. Gate the headline on `L2_reshuffle` (separated CI, ≥50% share) — same instrument as R5.
2. Ensure **every** path that reaches HiGHS in the proof uses the per-instance `compute_fallback_cost`,
   not the module constant or the test literal; add a guard asserting `fallback_cost < ~1e4` at MILP
   build on proof instances.

---

## R7 — π_hind floor "holds by construction" — sound, conditional on the single-time-scalar invariant surviving region→region. **DEFENDED-with-condition**

§13 / D-A10 (rev): `C(π_hind) ≤ C(M₁)` is "a labeled self-check, holds by construction" because π_hind
has the superset full-information feasible set. The reasoning is **sound** — π_hind sees all demand at
t=0, is free of the tender lock, and is physically-feasible only, so its optimum cannot exceed any
online policy's. As a floor it is correct and cannot hide a *false-thesis* bug.

But critique-11 C6 / D-A13 identified the bug class it *can* hide: if the **time scalars diverge**
between graph-build, the MILP `arr_dest`, and the scorer walk, then under MAWB consolidation the scorer
walk can *beat* `arr_dest`, M₁ can score below π_hind, and `Reg < 0` breaks the chain — i.e. the floor
"holding by construction" can be **violated by a time-scalar drift**, not by a thesis failure.
Critique-13 §D reports D-A13 walk≡scalar was **empirically verified (0 mismatches)** — but **on the
single-gateway, single-POD fixed-lane instance.** §13 v4's D-A24 introduces **multi-O/D subgraphs and
airport-pair-specific arcs**, and critique-13 N7 already flags `Δ^post` is summed over the *whole*
subgraph dest-chain (correct only for single-POD HAWBs). Region→region routing is exactly the
multi-POD generalization that N7 says breaks the walk≡scalar identity.

So: the floor is sound, but its "by construction" status **rests on D-A13, which was verified in a
world D-A24 leaves behind.** Demoting it from a falsifiability guard to a self-check (R2) is acceptable
*only if* the self-check still fires when the time-scalar drifts.

**What settles it:**
1. **Re-verify walk ≡ scalar on a region→region multi-O/D instance** (fix N7: `Δ^post` per terminal arc
   along its actual dest tail) before trusting the floor on the D-A24 grid.
2. Keep `C(π_hind) ≤ C(M₁)` asserted **per draw on a binding-capacity instance** (backtest §11 DoD) —
   a per-draw violation is the tripwire for a time-scalar bug, which is the only thing the floor catches.

---

## R8 — Baseline M₀ fairness / M₁′ pinning. **DEFENDED-on-design (implementation-risk only)**

D-A23 makes M₀ a **competent single-pass baseline**: optimal within-cycle consolidation under a
deterministic `(tender_at, tier, shipment_id)` tie-break, never disturbing priors. This is a **fair**
baseline by design — it is *not* the hobbled "naive greedy that leaves consolidation on the table"
strawman that would inflate L2. The design explicitly closes that gap (the whole point of D-A23 over the
old "incremental-greedy" M₀). I cannot land an attack on the *design*.

The residual risk is **implementation drift**: the proof that M₀ is fair is the M₁′ control arm with
`C(M₁′) == C(M₀)` **exact per draw** (D-A11). If that equality is only approximate (tie-break leakage,
solver nondeterminism — note N2: `PYTHONHASHSEED=0` documented but unenforced), then M₀ is silently
*not* the pinned feasible set of M₁, and any gap leaks into L2 as fake savings. So the design is
defended, but the headline is only honest **if `C(M₁′)==C(M₀)` is asserted exact and green** — and that
requires N2 (hashseed enforcement + cross-process determinism test) which critique-13 lists as still
unfixed.

**What settles it:** `C(M₁′) == C(M₀)` exact per-draw, under enforced `PYTHONHASHSEED=0` + a subprocess
determinism test. If the equality is only "within ε," ε is an L2 noise floor that must be reported and
netted out.

---

## R9 — Leakage tripwires were written for the fixed-lane world; D-A24 opens a new channel. **PARTIALLY LANDS**

Covered mechanically under R4b. Stated as its own attack because it is the cleanest "the experiment
became dishonest when the scope expanded" finding: critique-11/13 certified no-lookahead **for the
fixed-lane, static-schedule instance**. §13 v4's D-A24 (origin/dest airport now an optimizer decision
via a per-airport trucking matrix) is a **scope expansion that invalidates the prior leakage
certification** until re-run. The demand tripwire perturbs demand rows; the schedule tripwire perturbs
flights — neither currently perturbs the **airport-choice** dimension, which is new state the optimizer
now reads. A leak here (M₁ pre-positioning toward an airport because of un-revealed future demand or a
future flight) is, per backtest §5, "indistinguishable from a real saving."

**What settles it:** Re-run both tripwires post-D-A24 with an airport-choice case; treat the prior
no-lookahead certification as **lapsed** until then.

---

## Minimum test/instance set before the (κ,α,λ) number is publishable

Ranked. The first three are non-negotiable; without them the number is not a number.

1. **Binding-corner conservation proof (R3).** Binding-capacity + mid-tender + 2-arc-reshuffle fixture
   at a **low-α / zero-count-flight** cell; global per-step identity + per-shipment move-journal
   asserted for M₀ and M₁. *This is backtest §6's own "single most important missing test."*
2. **L2=0-fraction + distinct-reshuffle-event gates (R1).** Pre-registered ceiling on the zero-fraction
   and floor on distinct reshuffle events; report L2 conditional-on-contention beside the unconditional
   mean. Strongly consider moving the headline to the forwarder-scale instance where contention is
   structural, not a seed lottery.
3. **A discriminating positive control (R2).** The retired C2 null is non-discriminating at the sweep
   corner (a no-op engine passes it). Add a constructed instance with a hand-computed L2* > 0 the engine
   must recover, AND show the corner gate sits on a real gradient (perturbing toward tightness moves L2
   across the CI). Without this, the design cannot demonstrably fail in an engine-discriminating way.
4. **`L2_reshuffle`-gated headline (R5, R6a).** Build the D-A12 3-way split; gate the headline on
   `L2_reshuffle` with a separated CI and ≥50% reshuffle-share floor; assert the C.10 penalty is
   excluded from `realized_cost` on the scorer.
5. **M₁′ exact-pin (R5, R8).** `C(M₁′) == C(M₀)` exact per draw under enforced `PYTHONHASHSEED=0` +
   subprocess determinism test; report any ε as an L2 noise floor.
6. **Cross-arm capacity-vector byte-identity (R4a)** + **re-screen-cannot-add-capacity test (R4b)**.
7. **walk ≡ scalar re-verified on region→region multi-O/D (R7)** (fix N7 first); `C(π_hind) ≤ C(M₁)`
   per draw on a binding instance.
8. **Both lookahead tripwires re-run post-D-A24 with an airport-choice case (R9)**.
9. **Fallback conditioning swept clean (R6b):** every HiGHS path in the proof uses per-instance
   `compute_fallback_cost`; guard `fallback_cost < ~1e4` at MILP build.

---

## One-paragraph bottom line

The §13 v4 redesign fixed a real circularity (supply no longer derived from demand) and that is a
genuine improvement to experimental honesty. But two of its *simplifications* — **retiring the
constructed negative control for a sweep-corner gate (R2)** and **adding the L2=0 diagnostic without a
kill threshold (R1)** — weakened falsifiability at exactly the small-scale, integer-`total_N≈10` regime
where the proof currently lives, a regime where whether *any* contention fires is a seed-and-supply
lottery. Combined with the **never-once-exercised conservation identity (R3)** on the new binding
corner the integer multinomial manufactures, the current design can produce a positive mean L2 that is
**not distinguishable** from a permutation artifact, a phantom double-spend, or fallback-avoidance
mislabeled as reshuffle. None of these are fatal to the thesis — they are all settleable with the test
set above — but **as the design stands, the number is not yet publishable**, and the loose-corner gate
is not a sufficient guard against a hostile reviewer making exactly these three points.
