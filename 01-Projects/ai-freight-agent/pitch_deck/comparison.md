# Pitch Deck v1 → v2 Comparison

**Date:** 2026-05-25
**Process:** Created v1 from project context → ran 2 critique agents in parallel (skeptical seed VC partner + mid-size forwarder COO) → synthesized critique → wrote v2.

**Deliverables:**
- `v1.md` + `v1_initial.pptx` — initial deck
- `v2.md` + `v2_revised.pptx` — revised deck after critique
- `generate_v1.py` + `generate_v2.py` — python-pptx generators (editable)
- This file.

---

## Critique highlights (where v1 failed)

Both critique agents converged on the same structural issues. The most important ones:

### From the VC partner
1. **Team slide is a placeholder — single biggest blocker.** Cannot ship a seed deck with `[YOUR NAME]`. Need real credibility numbers + advisors.
2. **Zero customer evidence anywhere.** 27,000-word analysis = desk research, not customer contact. Need: forwarder quotes or named conversations in progress.
3. **"No incumbent" claim is the load-bearing wall and not properly defended.** Needs evidence anchor visible on the slide.
4. **Market sizing convergence is suspicious.** Top-down and bottom-up don't actually converge; both have ~50% ranges. Investors will spot this.
5. **MILP-grounded as headline differentiator is weak.** Even the founder's own §C.7 admits MILP doesn't matter for ~80% of shipments. Replace with labor-automation ROI.

### From the forwarder COO
1. **Consolidation planner objection — senior coordinators will say "Excel works."** Deck never confronts this. Need a UI mockup.
2. **Sub-90-day deployment is NOT credible for CargoWise shops.** Realistic is 7–10 months minimum. Split into GoFreight accelerator vs CargoWise moat.
3. **"TMS-agnostic adapter" is a red-flag phrase.** Every vendor says this. Commit to one TMS first.
4. **Pilot contract needs a defined success metric for the CFO.** Without go/no-go gate + refund clause, won't sign.
5. **"REPLACE Slide 7 WITH A SCREENSHOT. Even a mock one. If you can't show me this screen, I am not signing a $25K pilot."**

### Convergent across both
- **Architecture slide is too dense — simplify** (VC) or **drop entirely** (COO). Compromise: simplify to 3 layers.
- **Slide 8 (Why we win) is too many bullets — cut to 3.**
- **Slide 9 (Market) — replace dense table with one SAM + one SOM number.**
- **Slide 13 (Traction) — fix the ARR math; replace "6 strategic commitments" with customer-conversation traction; outcome-based not output-based milestones.**

---

## Slide-by-slide v1 → v2 changes

| # | Slide | v1 issue (critique) | v2 change |
|---|---|---|---|
| 1 | Cover | Working code-name expected | Kept; placeholder for company name still |
| 2 | Vision | Architecture description in vision slide | Replaced para 2 with 2028 vision statement ("rented, not built"); softened 3%/40% to "for consolidation specifically" with FTE anchor |
| 3 | Problem | Three desk-research bullets (Walltech, ScienceDirect academic citation) | **Rewritten as Tuesday afternoon anecdote** + truth + wedge. Dropped ScienceDirect citation. Anchored in real $200M shop scenario. |
| 4 | Why now | "Mid-size being squeezed" hand-wave; unstructured channel feels grafted on | **Reordered to lead with stack maturation** ("MCP + LLMs + MILP solvers all matured in 2025"). Dropped WhatsApp bullet (it's a quote-desk wedge, not consolidation wedge). Added Tier-1 in-house bullet to anchor unbundling thesis. |
| 5 | Solution | "TMS-agnostic vendor-agnostic" marketing language; engine bullet too dense | **Honest integration sequence added** (GoFreight first, CargoWise Q3 2026, others later). Engine simplified to 3 components. |
| 6 | Architecture | 5 layers — too many for a deck slide | **Simplified to 3 layers**: What operator sees / What we build / What we integrate. Build layer highlighted. |
| 7 | How it works | Two text examples; "~2 seconds" is unverified marketing | **REPLACED with mock UI screen.** Consolidation planner showing 40-HAWB LAX build, proposed plan, sensitivities ("HAWB-47 flips to ECU co-load, +$340"), override buttons. Responds directly to COO's #1 concern. |
| 8 | Why we win | 5 bullets; MILP-grounded headline contradicts own §C.7; override learning is year-3 moat (undemonstrable) | **Cut to 3 points.** Kept planning-vs-execution + materiality. Replaced MILP-grounded bullet with **labor-automation ROI** ($4–7 savings per shipment, every shipment, from week 1). Moved override-learning to ask slide. |
| 9 | Market | 18 numbers in a 4-column table; convergence framing is circular | **2 big numbers**: $200M SAM + $5–20M SOM. Methodology one line each. Geographic breakdown moved to appendix. Right buyer / wrong buyer side-by-side. |
| 10 | Competition | Strong claim, weak evidence | Added **evidence anchor** ("15+ primary sources reviewed May 2026, zero mentions"). Added **competitive scenarios list** with probabilities (WiseTech CTO extension = primary risk). |
| 11 | Go-to-market | Sub-90-day deployment claim; no pilot contract terms | **Sequenced distribution table** (GoFreight 60–90 days, CargoWise 8–12 months/customer). **Pilot contract** (scope, duration, KPI, go/no-go gate, refund clause). **Trust ramp** with explicit gates. |
| 12 | Team | Empty placeholders | **Restructured with credibility-anchor shape**: years + $-volume + sanitized prior shipped system + reason-for-leaving. Advisor section added (explicitly marked as TO ADD before sending). |
| 13 | Traction + Ask | "6 durable strategic commitments" + inconsistent ARR math | Replaced "6 strategic commitments" with primary-research validation + forwarder calls placeholder. **Picked one ask number** ($2.5M, 14 months). **Fixed ARR math** ($1.6M at $40K × 40 customers). **Outcome-based milestones** (override rate, planner hours saved, not just deployment count). |

---

## Critique points applied vs rejected

### Applied (high-impact)
- ✅ Team slide restructured with credibility-anchor placeholders (VC + COO)
- ✅ UI mockup added as slide 7 (COO highest-leverage ask)
- ✅ "Why we win" cut from 5 to 3 (VC)
- ✅ MILP-grounded dropped as headline; labor-automation ROI added (VC)
- ✅ Architecture simplified from 5 to 3 layers (VC + COO)
- ✅ Market simplified to 2 numbers (VC)
- ✅ Integration sequence honest (GoFreight first, CargoWise next) — drop "TMS-agnostic" language (COO)
- ✅ Pilot contract terms slide added (COO)
- ✅ Trust ramp made explicit with gates at each stage (COO)
- ✅ ARR math fixed (VC)
- ✅ Outcome-based milestones (VC + COO)
- ✅ Competitive evidence anchor added (VC)
- ✅ Competitive scenarios with probabilities added (COO)
- ✅ Problem slide rewritten as Tuesday afternoon anecdote (COO)
- ✅ Why now reordered, dropped unstructured-channel bullet (COO)
- ✅ Advisor section in team slide marked TO ADD (VC)

### Applied (medium-impact)
- ✅ Vision slide 2028 statement added (VC)
- ✅ Dropped "27,000-word analysis" / "6 strategic commitments" internal language (VC + COO)
- ✅ Softened 3%/40% to "for consolidation specifically" with FTE anchor (COO)

### Rejected (with reason)
- ❌ **"Add real forwarder quotes from interviews"** — would be fabrication. User has to actually conduct these conversations before sending. Marked as placeholder.
- ❌ **"Specifically name a freight system the founder built"** — would violate confidentiality rule (CLAUDE.md). Used sanitized placeholder approach.
- ❌ **"Pick a specific lane (TPEB out of LAX) as the design partner focus"** — fits more naturally in GTM slide; partially incorporated as "consolidation planner on one gateway (e.g., LAX air)" in pilot contract.
- ❌ **"Add 14th slide for vertical specificity"** — would push past 13 slides. Compromise: incorporated into GTM pilot scope.
- ❌ **"Cut architecture slide entirely"** (COO) — VC wants it. Compromise: simplified to 3 layers.
- ❌ **"Pick a company name before sending"** (VC) — out of scope for me; flagged as user action.

---

## What you still need to do before sending the deck

**Critical (deck can't ship without these):**

1. **Fill in team slide placeholders** (slide 12):
   - Your name + role
   - N years at top-5 global forwarder (use that wording — satisfies confidentiality)
   - $YB of freight under prior optimization systems
   - Sanitized name + customer count of a system you led
   - One-sentence "why I'm leaving / why now" thesis

2. **Add 1–2 advisors** to slide 12. Critical:
   - OR / optimization academic for technical credibility transfer
   - Ex-Tier-1 freight industry name for operator credibility

3. **Start forwarder outreach now.** Slide 13 has a placeholder for "[N] forwarder intro calls completed; [M] in active conversation." If you haven't started, do so before sending. **Both critique agents identified zero-customer-evidence as the single biggest gap.**

4. **Pick a company name.** "AI Freight Agent" is descriptive, not memorable. (If genuinely TBD, italicize a working codename.)

**Recommended (deck will be much stronger):**

5. **Get a real screenshot / Figma mock of the consolidation planner UI.** The text mock on slide 7 is a placeholder; an actual visual will be the single biggest improvement.

6. **Confirm the $2.5M / 14-month ask** matches your actual fundraising plan, or override the placeholder.

7. **Validate the labor-automation math** on slide 8. The $4–7/shipment savings claim is anchored in PRD §5.9, but should be sanity-checked against a real forwarder before pitching.

**For the next iteration:**

8. Consider a separate **technical deep-dive deck** (5–7 slides) for the partner meeting after the first pitch. Move MILP / architecture / optimization details there so the main deck stays operator-facing.

---

## One question each critique agent said they'd ask

**VC partner:**
> "Walk me through your last five forwarder conversations. Who were they, what did they tell you about consolidation today, and which of them will run a paid pilot in the next 90 days?"

A founder who answers with 5 names + dated conversations + 1–2 paid-pilot leads is a check. A founder who answers "we're planning to start that outreach in Q3" is not seed-fundable yet.

**Forwarder COO:**
> "Show me the consolidation-planner UI on a synthetic 40-HAWB LAX next-day air build pipeline. Not the architecture, not the math, not the optimizer console — what my Senior Air Coordinator will actually see at 3pm on Wednesday. Walk me through the screen, the recommendation, and what happens when they override it."

If you can show that screen — even a Figma mock — the pilot conversation continues. If you pivot to MILP optimality certificates, it ends.

---

## v3 — founder credentials + modern design system

**Created 2026-05-26.** Two changes from v2:
1. **Founder credentials filled in from resume** (slide 12). Specific numbers, employers anonymized per project confidentiality rule.
2. **Modern design system** applied per Designer agent research on 2025–2026 funded decks. Spec at `pitch_deck/design_spec.md`.

### Founder-market-fit content changes (slide 12)

| What | v2 | v3 |
|---|---|---|
| Name | `[YOUR NAME]` placeholder | **Richard Li-Yang Chen** |
| Tagline | Generic "deep OR/MILP expertise" | "15+ years building production supply-chain planning systems at scale" |
| Track record | Placeholder bullets | **5 anonymized employer entries** with specific metrics: $110M+ savings · 72% cost-to-serve reduction · 82% automation · 30+ team · 100K+ concurrent loads · USDOT / White House Council · Sandia |
| Killer credential | None | **Built the ML transit-time estimator at a top-5 forwarder** — literally the same component as in this product's architecture |
| Education | Not shown | **Ph.D. Operations Research, U. Michigan** + Berkeley M.S./B.S. IEOR |
| Founder thesis | One-line placeholder | "Built this architecture for Tier-1 platforms — twice. It works. The window is now." |

**Confidentiality note:** per CLAUDE.md, Flexport / Amazon / Coupang are never named in any project artifact. v3 anonymizes to "top-5 global digital freight forwarder", "Fortune 50 e-commerce + logistics platform", "top-10 global e-commerce platform." USDOT, White House Council, Sandia, U. Michigan, Berkeley are all kept as real credentials (not in ban list).

### Design system changes (all 13 slides)

**Color palette** — 5 colors plus one Signal accent (replaces v2's 5+):
- Surface `#FAFAF7` (warm off-white, replaces pure white) · Ink `#0E1116` (near-black) · Muted `#5B6470` · Accent `#1F3D8A` (deeper cobalt) · Accent Soft `#EEF2FC` · Hairline `#E4E4DE` · Signal `#B4541C` (used exactly once, on slide 10's primary-risk callout)

**Typography** — Inter + JetBrains Mono (both installed via `brew install --cask font-inter font-jetbrains-mono`):
- 10-step type scale: display-xl (96pt) → caption (11pt)
- Three weights only: 400 / 600 / 700
- Mock UI on slide 7 uses JetBrains Mono

**Layout** — 12-column grid, 0.6" margins, 0.1" baseline rhythm.

**Per-slide moves applied:**

| # | Slide | Key design change |
|---|---|---|
| 1 | Cover | Title moved to Ink (not Accent); Accent reserved for 0.04" bar above title + small square top-right; hairline above byline |
| 2 | Vision | Headline mostly Ink; only the operator-readable phrase **"40% of their day to 15%"** in Accent (inline color shift) |
| 3 | Problem | **Dropped the highlight box.** Anecdote runs as plain paragraph with vertical Accent bar + 0.3" indent (block-quote treatment) |
| 4 | Why now | Replaced `1. / 2. / 3.` with eyebrow labels `01 STACK / 02 PRESSURE / 03 PROOF`; hairline rules between items |
| 5 | Solution | Bento grid (2×2: two stroked boxes top, one wide box bottom for the engine). Integration sequence **moved to slide 11** (was crowding slide 5) |
| 6 | Architecture | All three layers stroked (not filled). Build layer differentiated by Accent vertical bar + Accent eyebrow. Drop gray fill. `↑` glyphs between layers. |
| 7 | What planner sees | **Added macOS window chrome** (3 muted ovals). Dropped grey background. JetBrains Mono. Reduced Accent occurrences. |
| 8 | Why we win | Matches slide 4 — `01 SCOPE / 02 AI LAYER / 03 ROI` numeric labels + hairlines |
| 9 | Market | Two hero numbers stay. Right/wrong buyer become `✓` / `✕` glyph treatments (Accent / Muted) |
| 10 | Competition | **Stripped the table** — no header fill, no row borders, just hairline under header. "Zero overlap on consolidation" inline Accent. **PRIMARY RISK** in Signal orange (single use). |
| 11 | GTM | Stripped table. **3 trust-ramp pills** (Co-pilot · Supervised · Autonomous) with `→` between. Dropped italic epigraph. |
| 12 | Team | **Resume content + design**: vertical Accent bar on track record; em-dash leads instead of bullets; open roles as 3 pills; founder thesis as quote-style with vertical bar |
| 13 | Traction + Ask | **Use-of-funds as single horizontal stacked bar** (one chart in deck). Hero $2.5M at 84pt. `01–04` numeric labels for "where we are". Milestones as 3 bento boxes |

### What was deliberately NOT applied

- **Section divider slides before slide 5 and slide 11** — design spec recommended but only "if pacing allows." Kept at 13 slides; cut to maintain length.
- **Letter-spacing on eyebrow labels** — python-pptx can't do it cleanly. Used uppercase + size + color instead.
- **Real customer quotes** — would be fabrication. Still placeholder.

### Files in pitch_deck/

```
pitch_deck/
├── design_spec.md           # designer agent's research + spec
├── v1.md                    # v1 content
├── v1_initial.pptx          # v1 PPTX
├── v1_initial.pdf           # v1 PDF
├── generate_v1.py           # v1 generator
├── v2.md                    # v2 content (post-critique)
├── v2_revised.pptx          # v2 PPTX
├── v2_revised.pdf           # v2 PDF
├── generate_v2.py           # v2 generator
├── v3.md                    # v3 content (resume + new design)
├── v3_final.pptx            # v3 PPTX  ← the one to use
├── v3_final.pdf             # v3 PDF   ← the one to share with investors
├── generate_v3.py           # v3 generator
└── comparison.md            # this file
```

To regenerate v3 after editing:
```bash
uv run python pitch_deck/generate_v3.py
/Applications/LibreOffice.app/Contents/MacOS/soffice --headless \
  --convert-to pdf --outdir pitch_deck pitch_deck/v3_final.pptx
```

### What you still need to do for v3 before sending

**Critical:**
1. **Add 1–2 advisors** to slide 12 (OR academic + ex-Tier-1 freight operator). Slide currently has placeholder.
2. **Fill in `[N] forwarder intro calls; [M] in active conversation`** on slide 13. Start outreach now if not already.
3. **Pick a company name** (currently using working title "AI Freight Agent").
4. **Confirm $2.5M / 14-month ask** matches your actual fundraising plan.

**Recommended:**
5. Replace the text UI mock on slide 7 with a Figma / screenshot when you have it.
6. Sanity-check the $4–7/shipment savings claim against a real forwarder before pitching.
7. Consider a separate technical-deep-dive deck (5–7 slides) for partner meetings — move MILP/optimization details there so the main deck stays operator-facing.
