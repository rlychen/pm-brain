# BUILD STATUS — AI Multimodal Freight Routing Agent

> ⛔ **THE SIMULATION WAS BROKEN. THE REBUILD IS UNDER WAY (S60–S63). DO NOT TRUST ANY PRE-REBUILD NUMBER.**
> S56: the generated instance was **physically impossible** — median flight held **197 kg** vs a **500 kg**
> median shipment; **33% of flights held nothing**; cargo readiness **silently zeroed on DB load**; the
> scorer **could not express a missed connection**. Every old headline (L1, L2, OTP, fallback, τ/κ bands,
> decay, **tardiness weights**) is an artifact — re-measure per §11 Q6 before quoting anything.
> **Done: R1 persistence round-trip (S60); defect-contract test deletions (S61); supply-generator SLICES
> 1–4 (S61–S63); the demand-generator §3 forward-timeline rewrite (S63).** **Still no valid headline
> until the scorer is rebuilt and the §3 weight distribution lands, then Q6 re-measures.**

> ⛔ **HARD RULE: TARDINESS PENALTY ALWAYS ON.** Never run any air model with `tardiness_weight_scale=0`.
> The calibrated per-tier weights must be RE-DERIVED after the rebuild (anchored to the broken fallback gap).

> 🗣 **HOW TO WRITE TO THE USER.** Plain English, short, to the point. Define any term as you use it.
> **No notation dumps / symbol soup** (M1/τ/μ/PIH) — the user does not carry symbols across turns. No walls
> of text (500–1000 words = a failure). **Reach for a simple numeric example whenever it helps** (e.g.
> "3000 kg plane, 500 kg shipment ⇒ ~6 fit"). One question at a time, in prose (AskUserQuestion
> multiple-choice is rejected). See CLAUDE.md top rules + `feedback_plain_language_define_terms`,
> `feedback_one_question_at_a_time`.

> ⛔ **LOOSE PERMISSIONS, TIGHT CHECKPOINTS.** Never guess/fabricate — say "unverified" and verify. Stop
> and ask before any directional commitment (model/schema/dependency/calibration). Report failures plainly
> (a green suite testing the wrong thing is a failure). A **calibration-edit hook** blocks edits naming
> `CALIBRATED_TARDINESS_W`/`TIER_SPECS`/`tardiness_weight_scale`; unlock only with explicit user sign-off
> (`touch .claude/ALLOW_CALIBRATION_EDIT`), then re-arm (remove the file).

**Last refreshed:** 2026-07-25 (Session 63 — **Supply-gen slice 4 (instance report-card + load-time
safety check) + the demand-generator §3 forward-timeline rewrite, both BUILT & GREEN in isolation.**
Demand + supply + card tests pass; **45 downstream replay/scorer tests RED on `KeyError: tender_at` =
the accepted clean-cut breakage** (fixed in the replay/scorer rebuild). Committed at this sign-off.).

---

## Current position

- **Phase:** 2 (Component Builds), air slice — **REBUILD UNDER WAY.**
- **Supply generator:** ✓ slices 1–4 done (exogenous per-flight draw · `mu` dial + per-flight cap C.5f ·
  co-load lane-SLA remodel · **S63: report-card + `validate_instance()` load-gate**).
- **Demand generator:** ◐ **§3 forward timeline BUILT (S63).** Remaining: the §3 **weight distribution**
  (lognormal — couples to the τ denominator) and **dead config-knob cleanup**.
- **Scorer / replay:** ✗ — **the next big build.** 45 tests red on the deleted `tender_at`.

### Next action (resume here)

**The replay/scorer rebuild.** Two coupled parts:
1. Repoint the replay commit clock from the deleted `tender_at` → `commit_backstop_at` (clears most of
   the 45 red). `src/replay.py` reads `r["tender_at"]` at ~343/370/388/928/1094.
2. **R4** scorer refuses non-OPTIMAL + **R5** a missed connection is expressible (aircraft never waits).

**Or finish the shipments side first:** the §3 weight distribution (lognormal median 250 / mean 500, cap
`min(3000, 9ρ)`, resample ≤5% — **treat carefully: `_expected_slot_mean` and the τ denominator assume the
current triangular closed form**) + config-knob cleanup (rip out `book_lead_*`, `lambda_compress`,
`tier_coupled_arrival`, `lead_buckets`/`LeadBucketConfig`, `t_dead_prob`, `_T_DEAD_FLOOR_H`,
`_contracted_by_dest_day`, `_categorical`). Then Q6 re-measure, then the sweep.

---

## §11 sign-off — CLOSED (S60). Q1–Q6 all answered; §3/§5 re-derived for the 240/wk book.

- **Q1** repair fork → grow the book (120→240/wk; φ=0.25; BSA held ~36).
- **Q2** reservation pricing → C.5e usage floor + ψ=0.35 no-show; co-load = per-(lane,day) lane-SLA channel.
- **Q3** BSA shape → ~36 positions/wk, does NOT scale with the book.
- **Q4** warm-up → 30 d + a 59 d control cell.
- **Q5** pinned constants → tier mix 20/55/25, weekday weights, W=24 h, T_max=96 h, K=3 (M5 threshold).
- **Q6** the re-measure list (14 items) — a consequence, run after the rebuild.

Governing doc: `docs/design/simulation_design_s56.md` (v3). The design gate is clear.

---

## The S56 finding (why the rebuild) + defects

Root cause (one line): `S = tau · n_hawbs · E[cw]` split across every flight-day ⇒ capacity was an artifact
of schedule size and **τ could not bind**. Defects D1–D7 (all measured):

| | Defect | Status |
|---|---|---|
| D1 | `ready_early_h` dropped on load ⇒ replay planned with ready=0 | ✓ fixed (S60 R1 round-trip guard) |
| D2 | `ac_type` hardcoded FREIGHTER on load | ✓ fixed (S60) |
| D3 | MILP has no per-flight capacity | ✓ fixed (S62 C.5f `_build_c5f_flight_cap`) |
| D4 | `score_run` never checks `sol.status` | ✗ — scorer rebuild (R4) |
| D5 | scorer holds the aircraft for late cargo (a miss isn't expressible) | ✗ — scorer rebuild (R5) |
| D6 | `bsa-hard` = one horizon-global pooled take-or-pay | tex row C.13c written S59; code pending |
| D7 | horizon truncation | pending |

**Why ~90 review agents missed it:** 21 critique docs, 0 data-vs-reality. **The S63 report card + the
`validate_instance()` load-gate are the structural fix** — a broken world now shows in a commit diff and
fails loudly on load.

---

## Gates cleared

| Gate | Item | Status |
|---|---|---|
| Phase-0 | PRD | ✓ approved |
| G-LaTeX | Air optimizer model (`model/air_freight_routing.tex`) | ✓ approved; S59 amendments reviewed (S60). C.5f matches code (S62); C.5e + C.13c still to build |
| **G-Sim** | Simulation design v3 §11 | ✓ **CLOSED (S60)** — the design gate is clear |
| G-Method | Arrival-only replan methodology | ⚠ reconcile to the new timeline (`known_at ≡ book_at`, endogenous cutoff, `commit_backstop`) — folds into the replay rebuild |
| G-Isolation | Air graph / MILP / **supply gen** / **demand gen** | ◐ **supply + demand + card GREEN in isolation.** 45 downstream (replay/scorer) RED = accepted clean-cut on the deleted `tender_at` |
| G-Review | Standing agents | ✗ replaced by data-path gates: `validate_instance()` raises in `load()`, round-trip in `persist()`, **diffed `instance_card.md`** |
| G-LaTeX | Ocean FCL / LCL / Trucking | ☐ drafted, not approved |

---

## Component status

Legend: ✓ done · ◐ in progress · ☐ not started · ⏸ suspended · ✗ invalid

| Component | Status | Notes |
|---|---|---|
| Simulation design v3 §11 | ✓ | CLOSED S60. |
| Air MILP model (tex) | ◐ | C.5f built; C.5e reservation + C.13c lane-week BSA re-key still to build. |
| **Supply generator** | ✓ | Slices 1–4. S63 slice 4 = report-card + load-gate. Scope lock held ("draw + selection only"). |
| **Instance report card** | ✓ | `data/synthetic/instance_card.py` (planes-side, final) + `validate_instance()` in `scenario_io`. Written to the scenario folder on save; diffable. Demand/scorer metrics = explicit "pending" placeholders. |
| **Demand generator / timeline** | ◐ | **§3 forward timeline BUILT (S63):** book_at=Poisson(240/wk), ready=book+gap, deadline=ready+base+W+offset, T^abs=Δ+96, `commit_backstop` from graph, `born_dead`. **Remaining: §3 weight distribution (lognormal) + dead config-knob cleanup.** |
| Persistence seam (`scenario_io`/`scenario_db`) | ◐ | R1 round-trip + C.5f caps + **S63: dropped `tender_at`, added `commit_backstop_at`+`born_dead`; `validate_instance()` in `load()`**. |
| Air MILP (`air_milp.py`) | ◐ | C.5f built. Still: C.5e reservation + no-show; C.13c lane-week re-key. |
| **Scorer (`replay.score_run`)** | ✗ | **The next big build.** D4 (never checks status) + D5 (a miss isn't expressible). Reads the deleted `tender_at` ⇒ 45 red. |
| Cadence parameter · CapDecay · 2D reservation/ratchet | ✓/◐/⏸ | survive; co-load excluded from decay; reservation keying+cost owned by the tex (REV 5). |
| **Thesis number (sweep)** | ✗ | BLOCKED on scorer + weight distribution + Q6. |
| Ocean / LCL / Trucking / MCP / agent / UI | ☐ | Later stages. |

---

## Quality state

- **Green in isolation:** demand (17: `test_arrival_stream` 13 + `test_arrival_persistence` 4) + supply
  (`test_exogenous_catalog` 9 + `test_coload_lane_sla` 10) + card (`test_instance_card` 10). Ruff clean on
  all edited files (`air_generator.py` keeps its documented pre-existing E501s).
- **45 RED (accepted clean-cut):** `test_replay_loop` (24) · `test_scorer_metrics` (8) · `test_scenario_db`
  (1) · `test_soft_bsa_cliff` (1) + others — all `KeyError: tender_at`. Cleared by repointing the replay
  clock to `commit_backstop_at` in the scorer/replay rebuild. **NOT a valid-world headline.**
- **Report-card numbers (μ=2.5, the current true values):** τ_v ladder **0.71 / 1.04 / 1.65** at
  μ=1.5/2.5/4.5 (default in [1.0,1.3]); demand-independent (n_hawbs 12 vs 240 → identical). The recorded
  0.72/1.06/1.68 predated the slice-3 co-load remodel.
- **Born-dead check:** 42% at a 7-day flight horizon vs **0% at the §3 30-day horizon** — the long deferred
  gaps (up to ~19 d) need the wider flight visibility. `commit_backstop ∈ [book, deadline]` always.
- Real HiGHS, never mocked. `mip_rel_gap=0.005` planning / 0 scoring. PIH-to-0.005 re-check pending.

---

## Deferred / parked

- **§3 weight distribution** (lognormal median 250 / mean 500, cap `min(3000, 9ρ)`, resample ≤5%) — the
  current triangular draw feeds `_expected_slot_mean` / the τ denominator; switching to lognormal breaks
  that closed form. Its own slice.
- **Dead config-knob cleanup** — `book_lead_*`, `lambda_compress`, `tier_coupled_arrival`, `lead_buckets`/
  `LeadBucketConfig`, `t_dead_prob`, `_T_DEAD_FLOOR_H`, `_contracted_by_dest_day`, `_categorical` (left in
  place this session to keep the change reviewable; not read on the new path).
- **Remaining R-items:** C.5e reservation + no-show; C.13c lane-week BSA re-key; R6 clock-gated arcs;
  R7 retirement predicate; R8 fallback only at backstop; R9 pre-registered headline.
- ⏸ S54 2D spot reservation (REV 5) · reputation/allowance ratchet · disruption/recourse (v1 out of scope)
  · booking revision/cancel (v1 out).
- **Re-derive after the rebuild (§11 Q6):** every headline, τ/κ bands, decay params, belly split, **the
  C.10 tardiness weights**, the S45 L2 decomposition.
- **Scratch** in `scripts/_*.py|.err` — uncommitted, disposable.

---

## Locked decisions — where the durable ones live

CONTEXT.md retired S62; BUILD_STATUS is THE pointer. Do not re-litigate without new evidence:
- **Rebuild decisions (§11 Q1–Q6):** the §11 section above + `docs/design/simulation_design_s56.md`.
- **Air MILP formulation:** `model/air_freight_routing.tex` (keying, cost, C.5e/C.5f/C.13c/co-load).
- **Durable product/strategy/architecture:** the `project_*` memory files (`MEMORY.md` index).
- **Working-style / guardrails:** the `feedback_*` memory files + CLAUDE.md hard rules.
- **Tech stack:** CLAUDE.md § Tech stack (HiGHS / FastMCP / LangGraph / uv / pytest).

---

## Doc map

| Doc | Role |
|---|---|
| **`BUILD_STATUS.md`** (this file) | THE pointer — current position, what's next, gates, locked decisions. Read first. |
| **`docs/design/simulation_design_s56.md` (v3)** | THE governing doc — §11 CLOSED; §3/§5 re-derived for 240/wk. |
| **`model/air_freight_routing.tex`** | Approved MILP + S59 amendments (C.5e/C.5f/C.13c/co-load). User compiles the PDF. |
| `data/synthetic/instance_card.py` | The diffable report card (S63) — the regression guard that replaced the standing review agents. |
| `SESSION_LOG.md` (top entry) | Last-session detail — the append-only archive. |
| `.claude/commands/signoff.md` | The `/signoff` end-of-session checklist (authoritative). |
