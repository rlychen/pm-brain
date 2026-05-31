# Pitch Deck v3 — Design Spec

**Audience:** seed-stage investors for a B2B vertical AI startup (freight forwarding optimization).
**Format:** 13 slides, 16:9 (13.33" × 7.5"), delivered as PPTX via python-pptx.
**Brief in one line:** **Inter on near-white, one disciplined accent (deep cobalt), a 12-column grid, ruthless restraint.** Modern but not gimmicky. Investor-grade, not brand-reveal.

---

## Section 1 — Research synthesis: what's "current" in 2026

The 2026 funded-deck look has converged on a recognizable set of moves. The 2020 "polished corporate" template — gradient hero backgrounds, stock photography, two-color callouts, bullet-heavy slides — now reads as dated. What replaced it:

**Pattern 1 — Near-monochrome with one disciplined accent.** Black-or-near-black on off-white (or inverse, dark mode). Exactly one accent color, used sparingly. Linear, Stripe, Notion, Vercel all converge here. The accent appears on roughly one element per slide — a key number, a section eyebrow label, a CTA. Multi-color palettes signal "amateur deck made in Canva." ([Ink Narrates — fonts and colors in real pitch decks](https://www.inknarrates.com/post/best-fonts-and-colors-for-pitch-deck); [Design System Analysis: Linear](https://getdesign.md/linear.app/design-md))

**Pattern 2 — Inter (or its peers) as the workhorse, paired with a display weight in the same family.** Inter is the de facto SaaS sans-serif (Notion, Linear, Shopify) — but its ubiquity has become a critique: "when everyone uses Inter, typography stops being a differentiator." 2026 funded decks either (a) double down on Inter at extreme weight contrast (300 vs 800) to create hierarchy, or (b) reach for Geist (Vercel's open-source release), GT America, or Söhne for differentiation. For a deck where the engineer's constraint is python-pptx system fonts, Inter is the correct call — but used with discipline. ([SaaS Typography Playbook — FullStop](https://fullstop360.com/blog/insights/branding/saas-typography-playbook-what-leading-companies-use); [Best Font for Presentation — Whitepage 2026](https://www.whitepage.studio/blog/the-ultimate-guide-for-using-fonts-in-decks-presentations))

**Pattern 3 — Oversized headline numerals.** A single 72–120pt number, with a small caption underneath. Market slides, traction slides, and "the ask" slides routinely reduce to one number on the page. Sequoia-coached decks have done this since 2018; 2026 decks lean harder. The v2 deck already does this on slide 9 ($200M / $5–20M) and slide 13 ($2.5M) — keep that and extend it.

**Pattern 4 — Eyebrow + headline + lede, three-tier vertical hierarchy.** Small uppercase eyebrow label (e.g. "PROBLEM"), large display headline, body paragraph or supporting elements below. This is the dominant slide structure in Artisan, Profound, MAI, and most YC W25/S25 demo-day decks. The v2 deck already uses this pattern (`add_text(..., "VISION", ...)` + `add_section_title(...)`) — refine it. ([Artisan Series A breakdown — Best Pitch Deck](https://bestpitchdeck.com/artisan-series-a); [Failory — B2B pitch decks 2025](https://www.failory.com/pitch-deck/b2b))

**Pattern 5 — The bento grid for content-dense slides.** Apple's bento-box layout has crossed into investor decks in 2025–2026. Disparate facts — a chart, a quote, a metric, a screenshot — sit in distinct rounded rectangles on one canvas. Bento works *only* when content is genuinely heterogeneous; do not use it as a default for text-only slides (it makes you look like Canva). For this deck: solution slide (slide 5) and architecture slide (slide 6) are bento candidates. ([Pitch Deck Design Trends 2026 — Pitchworx](https://pitchworx.com/pitch-deck-design-trends-dominating-usa-boardrooms-in-2026/); [Visible.vc — 11 trends 2026](https://visible.vc/blog/startup-presentation-design-trends/))

**Pattern 6 — Generous whitespace and slow visual rhythm.** 2026 decks breathe. Margins are deeper, line-heights are higher (1.4–1.6 on body), there's less ink per square inch. Investors flip through faster than founders expect — a slide must land in 3 seconds. Whitespace is the cheapest way to buy that. ([Visible.vc trends 2026](https://visible.vc/blog/startup-presentation-design-trends/))

**Pattern 7 — Tables stripped of borders, replaced with horizontal rules.** No 1px gridlines. No alternating-row shading by default. The table is just text on a rhythm with one or two horizontal rules separating header from body. Pitch.com and Linear's investor materials both use this.

**Pattern 8 — Restrained iconography via Unicode glyphs or simple geometric shapes.** No icon libraries. No SVG illustrations of clouds and arrows. A single → for a "leads to" relation, a ✓ for ready, a ⚠ for risk — and that's it. The v2 deck already does this (`✓ ready`, `⚠ density var`) — extend that pattern.

**What's out (signals an amateur / dated deck):**
- Multi-color palettes (3+ accents)
- Gradient backgrounds, especially purple/blue gradients
- Stock photography of "diverse teams in office"
- Drop shadows on shapes
- Curved corner radii inconsistent across slides
- Center-aligned body text (use it for hero numbers only)
- Bullet points using `•` as the marker (use indentation + spacing, or `→`)
- More than 2 font weights in the body
- Footers with social handles / website / phone number — investors don't care
- Page numbers in the format "Page 3 of 13" (use just `03 / 13` with the slash, or omit)

---

## Section 2 — Recommended design system for v3

### 2.1 Color palette

Five colors. No more.

| Role | Hex | Use |
|---|---|---|
| **Surface** (background) | `#FAFAF7` | All slide backgrounds. A warm off-white — not pure `#FFFFFF`, which reads cold and unfinished. Closer to paper / Stripe's `#F6F9FC` family. |
| **Ink** (primary text) | `#0E1116` | Body text, headlines. Near-black with a slight cool cast. Pure `#000000` is too harsh on off-white. |
| **Muted** (secondary text) | `#5B6470` | Eyebrow labels, captions, footnotes, deemphasized body. Single muted color — do not also keep `#5A5A5A` and `#A0A0A0` floating around. |
| **Accent** (primary accent) | `#1F3D8A` | The deep cobalt. One step deeper and slightly cooler than v2's `#1A4B8B` — reads more contemporary, less "corporate blue 2015." Used on: hero numbers, section eyebrows, one element per slide. |
| **Accent Soft** (accent surface) | `#EEF2FC` | Quiet wash for highlight boxes (anecdote on slide 3, two surface boxes on slide 5, "build" layer on slide 6). Replaces v2's `#DAE8FC` — same role, lower saturation, less Canva-blue. |
| **Hairline** (border / divider) | `#E4E4DE` | The single line color. Use for horizontal rules under section headers and between table rows. **Replaces** v2's `#E8E8E8` (gray box on slide 6) — and rectangles with this border should always be 0.75pt, never 1.5pt. |
| **(Optional) Signal** | `#B4541C` | A second accent reserved for one purpose only: warnings, "primary risk" callouts. Used at most once or twice in the entire deck (slide 10's "primary risk"). Do not use for decorative emphasis. |

**Why no green / no other colors.** Green for "good" / red for "bad" is reflexive but adds two more colors to defend. The accent does double duty (it's the brand color and the affirmative signal). Reserve `#B4541C` for the rare moment we genuinely need to flag risk.

### 2.2 Typography

**Primary: Inter** (heading + body, single family, variable weights).
Inter is on macOS 14+ by default and on most investor machines via Office. Falls back gracefully.

**Monospace (UI mock on slide 7 only): JetBrains Mono → SF Mono → Menlo.**
JetBrains Mono ships with most dev machines; SF Mono is macOS-native; Menlo is the universal fallback (which v2 currently uses — keep as fallback but prefer JetBrains Mono for the visual signal it's a real product mock).

**System-font fallback chain in python-pptx:**

```python
FONT_SANS = "Inter"          # set this on the run
FONT_SANS_FALLBACK = "Helvetica Neue"  # macOS
FONT_MONO = "JetBrains Mono"
FONT_MONO_FALLBACK = "Menlo"
```

Note: python-pptx sets only one font name per run. If the investor opens the deck on Windows without Inter installed, PowerPoint will substitute. For maximum portability, the safe fallback is `Helvetica Neue` on macOS and `Arial` on Windows — the deck will still look correct, just less distinctive. **Recommend bundling Inter as an embedded font** if the engineer can do so via `python-pptx`'s font embedding (or distribute as PDF, which renders Inter regardless of viewer environment).

**Type scale (pt, in python-pptx `Pt(n)`):**

| Token | Size | Weight | Used for |
|---|---|---|---|
| `display-xl` | 96 | 700 | Cover title only |
| `display-l` | 72 | 700 | Hero numbers (slide 9 market, slide 13 ask) |
| `display-m` | 44 | 700 | Section headlines (slide 2 vision) |
| `display-s` | 32 | 700 | Standard section title (slides 3–13 main headline) |
| `heading` | 22 | 600 | Bento box titles, subsection heads |
| `subheading` | 17 | 600 | Inline emphasis ("First:", "RIGHT BUYER" labels) |
| `body` | 15 | 400 | Default body text |
| `body-s` | 13 | 400 | Compact bullets, table cells, mock UI |
| `caption` | 11 | 400 | Footnotes, page numbers, source citations |
| `eyebrow` | 11 | 700 | Small uppercase section labels, letter-spaced |

Eyebrow labels should be uppercase with positive tracking (letter-spacing). python-pptx does not expose letter-spacing directly — workaround: use a space between each letter (e.g. `"P R O B L E M"`) sparingly, or just use uppercase without tracking and accept that loss. Recommend: uppercase without tracking. The size + Muted color + uppercase reads correctly.

**Weight discipline.** Use only 400 (regular), 600 (semibold), 700 (bold). No light, no medium, no extrabold. Three weights total in the entire deck.

### 2.3 Layout grid

**12-column grid on the 13.33" canvas:**
- Outer margins: 0.6" left / right (matches v2 — keep)
- Top margin: 0.5" (eyebrow row sits here)
- Bottom margin: 0.4" (page number sits here)
- Live area: 12.13" × 6.6"
- Column width: ~0.97" with 0.15" gutters (or treat as a 6-column grid with 0.25" gutters for chunkier bento layouts — pick one and stick to it; recommend 12-col)

**Vertical rhythm:** baseline grid of 0.1" (~7.2pt). All vertical positions snap to multiples of 0.1". This is the source of v2's slightly-off feeling (positions like `Inches(4.55)` and `Inches(4.7)` are not on a shared rhythm).

**Spacing system (4pt base, render in inches):**

| Token | Inches | Use |
|---|---|---|
| `space-1` | 0.05" | tight inline (after eyebrow) |
| `space-2` | 0.1" | between paragraph and supporting caption |
| `space-3` | 0.2" | between major slide regions |
| `space-4` | 0.4" | between hero element and supporting block |
| `space-5` | 0.6" | margins |
| `space-6` | 0.8" | dramatic separation (only on cover) |

### 2.4 Slide background

**Off-white `#FAFAF7` solid fill on every slide.** No gradients, no textures, no patterns.

Why off-white and not pure white: warmer, less clinical, easier on eyes during a 30-minute pitch. Matches the paper-like aesthetic of Stripe's investor materials. Pure white is for **dark-mode tech decks** (Linear, Vercel) — that's a different stylistic direction not chosen for this deck because (a) investor leave-behinds are read on Adobe Reader / Preview which can render dark mode unevenly when printed, and (b) the freight-forwarding category reads better as "trustworthy infrastructure" than "consumer-cool tech."

### 2.5 Visual hierarchy rules

1. **One hero per slide.** Each slide has exactly one element that the eye lands on first — the headline number, the section title, or the mock UI block. Everything else is secondary.
2. **Three text-color levels maximum:** Ink, Muted, Accent. Never four. If you find yourself reaching for a fourth color, you have a layout problem, not a color problem.
3. **No more than two text-weight levels in body content.** Heading weight is its own thing.
4. **Whitespace is content.** If a slide has empty real estate, that's the design working. Do not fill it.
5. **Borders are 0.75pt or 1.5pt — never both on the same slide.** v2 mixes these; consolidate.

### 2.6 Iconography / glyphs

Use **Unicode characters only** (rendered in the current text font, no separate icon font needed). No SVG, no PNG icons.

| Glyph | Unicode | Use |
|---|---|---|
| `→` | U+2192 | "leads to" / sequence / flow |
| `▸` | U+25B8 | bullet replacement when you really want a marker |
| `✓` | U+2713 | ready / done / affirmed |
| `⚠` | U+26A0 | warning (use Signal color) |
| `↳` | U+21B3 | sub-item / consequence |
| `·` | U+00B7 | inline separator ("X · Y · Z") — already in v2, keep |
| `—` | U+2014 | em dash — for inline emphasis. Never use hyphens for emphasis. |

**Do not** use `★`, `►`, `■`, `◆`, emoji, or any Wingdings character. These render inconsistently across viewers.

### 2.7 Data viz / table style

**Tables:** drop all cell borders. Header row is bold Ink (not white-on-accent). One Hairline horizontal rule under the header, no row borders, no alternating shading. Cell padding: 8pt top / 12pt left-right.

Why drop the v2 white-on-accent header: the filled accent header is the single most "PowerPoint 2015" element in v2. Modern decks use typography weight + a hairline rule.

**Numbers and percentages:** when there's a single dominant number on a slide, render it at `display-l` (72pt) in Accent. Supporting caption in Muted body-s. Right-align number columns in tables. Use `−7.2%` (Unicode minus U+2212), not `-7.2%`.

**Charts:** if any chart is added later (not in current v2), use line charts with a single Accent stroke and a Hairline x-axis only — no y-axis line, no gridlines, no legend if there's only one series. Bar charts use Accent fill, no border.

### 2.8 Cover slide pattern

Single composition on left third of canvas (no centering). Display-XL product name in Ink (not Accent — the accent goes on the underline or eyebrow). Below it: 24pt tagline in Muted. Bottom-left corner: 11pt Muted line with name · role · round · date. Top-right corner: small Accent square (0.4" × 0.4") as brand mark — a placeholder for future logo without committing to one.

### 2.9 Section divider pattern (new — v2 doesn't have these)

Recommend adding **two section dividers** to give the 13-slide flow more shape:

- Before slide 5 (Solution): "What it is" — full-bleed slide, eyebrow label + one sentence.
- Before slide 11 (GTM): "How we get there" — same pattern.

Keep dividers if final length budget allows; otherwise skip. Each divider = ~10 seconds in a 20-minute pitch. Worth it if it helps reset attention; cut if pacing is tight.

Pattern: solid Accent Soft fill on full slide, Display-M headline in Accent, eyebrow above in Muted, no other elements.

### 2.10 Page footer / number

Right corner, 0.3" from bottom edge, `caption` size (11pt) in Muted, format `03 / 13` (with spaces around slash). No "page" prefix. No company name in footer. The product name doesn't need to appear on every slide — the investor knows what deck they're reading.

---

## Section 3 — Per-slide layout recommendations

For each slide: **Current state** (what v2 does), **Changes** (specific design moves), **New element** (if any).

### Slide 1 — Cover

- **Current:** Center-left "AI Freight Agent" 72pt in Accent, tagline 24pt, byline at bottom. All left-margin-aligned.
- **Changes:**
  - Move the title to start at `space-5` from left (0.6") and `space-6` from top (~2.6"). Add a 0.04" Accent horizontal bar above the title, 0.8" wide — a visual "brand anchor."
  - Title in **Ink**, not Accent. Tagline in **Muted**. Accent reserved for the bar above and a small Accent square (0.4") in the top-right corner.
  - Byline at the bottom-left, 14pt Muted, single line, with `·` separators (already correct).
  - Add a 1pt Hairline horizontal rule above the byline, full width of live area.
- **New element:** the Accent bar above title + Accent square top-right corner. Both serve as future logo slots without committing to logo design.

### Slide 2 — Vision

- **Current:** Eyebrow "VISION", 44pt headline in Accent, 18pt body, 12pt footnote.
- **Changes:**
  - Move the headline to **Ink**, not Accent. Apply Accent only to the operator-readable phrase **"from 40% of their day to 15%"** (inline run-level color change in python-pptx — yes this is supported).
  - Body paragraph: reduce to 17pt, Ink, with line-spacing 1.5.
  - Bold the operator-readable phrase **"Operators handle exceptions; the system handles the rest."** inline. Two emphasis points on this slide — that's all.
  - Footnote moves to 11pt Muted.
- **Rationale:** if the entire headline is Accent, nothing is. Color emphasis must do work.

### Slide 3 — Problem (Tuesday afternoon scene)

- **Current:** Eyebrow, section title, full-width anecdote in highlight box with Accent border, "THE TRUTH" and "THE WEDGE" sublabels below.
- **Changes:**
  - **Drop the colored box around the anecdote.** Replace with: anecdote runs as paragraph text directly on the surface, 17pt Ink with 1.5 line spacing, indented 0.4" from the left margin (gives it the feel of a pulled quote without the box).
  - To the left of the anecdote, a vertical Accent bar (0.04" wide, full height of anecdote). This is the only graphical treatment — it reads as "block quote."
  - "The plan that ships to CFS is the third version. No artifact survives the day. Tomorrow they start over." — break onto its own line, 19pt **Ink bold**, not Accent. The bold + line break does the work.
  - "THE TRUTH" / "THE WEDGE" — keep the eyebrow style, but render their content at 15pt Ink (currently 15pt — keep).
- **Why:** the v2 highlight box with Accent border is the most dated element on the slide. A vertical bar + indent is the 2026 move.

### Slide 4 — Why now

- **Current:** Three numbered items, each with bold heading + Muted body.
- **Changes:**
  - Replace numbered `1. / 2. / 3.` with eyebrow labels: `01 STACK`, `02 PRESSURE`, `03 PROOF`. Same Accent color, same size, but more meaningful.
  - Tighten the body — Muted at 14pt is fine, but bump heading to 19pt for stronger hierarchy.
  - Add a Hairline horizontal rule (full live-area width) between each item. Three items, two rules.
- **New element:** the two hairline rules. Gives the slide a sense of a list without bullets.

### Slide 5 — Solution

- **Current:** Two surface boxes (Quote desk, Consolidation planner) in Accent Soft, integration sequence as bullets below.
- **Changes:** **This is a bento candidate.**
  - Use a 2-column × 2-row bento grid:
    - Top row (taller): Quote Desk (left), Consolidation Planner (right) — keep as is but switch boxes from filled Accent Soft to **stroked** boxes (0.75pt Hairline, white interior). Headings stay Accent.
    - Bottom row (shorter): one wide box spanning both columns containing the engine description ("MILP optimizer + ML feasibility predictor + LLM orchestration agent") with the engine words in slightly larger Ink (17pt).
  - Move the integration sequence (`First / Second / Then`) out of this slide entirely. It belongs on the GTM slide (slide 11), where the partner-application timing is already discussed. Removing it cleans up slide 5 dramatically.
- **Why drop integrations from slide 5:** the slide is "what is the product." Integration sequencing is "how we get to market." Conflating them is the main reason slide 5 feels cluttered.

### Slide 6 — Architecture (3 layers)

- **Current:** Three stacked horizontal layer boxes with different fills (neutral / highlight / integrate-gray), middle one highlighted as "build."
- **Changes:**
  - Keep the three-layer structure — it's conceptually right.
  - Replace fill colors: **all three layers** get stroked boxes (0.75pt Hairline, white interior). The "build" layer is differentiated by (a) a **0.04" vertical Accent bar on its left edge**, and (b) its eyebrow `WHAT WE BUILD` in Accent rather than Muted.
  - The arrows between layers (`▲` in v2's monospace block — but v2's current implementation uses rectangles, no arrows): add small `↑` glyphs (U+2191) at the boundary between layers, centered, in Muted, 14pt. Two arrows, between three layers.
  - Drop the gray fill (`COLOR_INTEGRATE = #E8E8E8`) entirely — the v2 use of three different fills makes it look busy.
  - Bottom-of-slide italic centered line ("We do not replace the TMS...") — change to **left-aligned**, regular weight (not italic), Muted, 13pt. Centered italic in a deck is dated.

### Slide 7 — What the planner sees (mock UI)

- **Current:** Monospace UI mock in a grey-fill bordered box.
- **Changes:**
  - **Drop the grey fill (`COLOR_MONO_BG = #F5F5F5`).** The mock UI gets a stroked box (1pt Hairline) on the surface (`#FAFAF7`). Cleaner.
  - Change the font from Menlo to **JetBrains Mono** if available, fallback Menlo.
  - Slightly larger monospace text — 14pt rather than 13pt. Pitches are read at distance.
  - Color: line-by-line color hierarchy in v2 is good (Accent for headings / cost / CTA, Ink for body). Keep it. But reduce Accent occurrences from 3 to 2 — the `Tomorrow's outbound build...` title line in Accent + `[ APPROVE PLAN ] ...` in Accent. The `COST vs LAST WEEK BASELINE: -$1,840 (-7.2%)` line stays bold Ink (not Accent — the data should land via weight, not color).
  - Add a small **screen chrome** above the mock UI: a 0.25"-tall thin strip with three tiny circles on the left (the macOS window controls — render as three small `MSO_SHAPE.OVAL` shapes in Muted). This costs almost nothing to add and significantly increases the "this is a real product" perception. The circles do not need to be red/yellow/green — all Muted is fine and reads as more sophisticated than the traffic-light treatment.
  - Below the mock UI, the closing sentence ("Sensitivities are the thing Excel cannot produce...") should be **non-bold** Muted, 14pt. Bold + Muted (current state) is a contradiction.
- **New element:** screen chrome (three Oval shapes).

### Slide 8 — Why we win

- **Current:** Three points with `→` glyph, bold heading 22pt + body 15pt in Muted.
- **Changes:**
  - Replace the `→` glyph in v2 (already Accent) with an Accent numeric label `01 / 02 / 03` (matches slide 4's revised treatment).
  - Add Hairline rules between items (matches slide 4).
  - Tighten heading from 22pt to 20pt; body stays at 15pt but switch from Muted to Ink — these are the proof points, they should land at full readability.
- **Consistency:** slides 4 and 8 should look like siblings — same eyebrow numeric label + hairline rules pattern.

### Slide 9 — Market

- **Current:** Two giant numbers ($200M / $5–20M) at 72pt Accent, right/wrong buyer blocks below.
- **Changes:**
  - Keep the two big numbers. Drop them down 0.2" so they breathe.
  - Add `space-2` after each number, then label in Muted bold-eyebrow style (currently 18pt Muted "SAM (global)" — change to 11pt eyebrow `SAM · GLOBAL` for stylistic consistency, then a 14pt Ink line `2,500 mid-size forwarders × $30–50K ACV`).
  - Right buyer / Wrong buyer blocks: drop the headers `RIGHT BUYER` / `WRONG BUYER` (eyebrow Accent / Muted). Replace with a simple two-column layout: `✓ Right buyer: ...` (Accent ✓, Ink body) and `✕ Wrong buyer: ...` (Muted ✕ at U+2715, Muted body).
  - Drop the parenthetical at the bottom — appendix references don't need to live in the body of a deck slide. Move to footer if at all.

### Slide 10 — Competition

- **Current:** Eyebrow + section title + table with Accent header (white-on-accent) + competitive scenarios as bullets.
- **Changes:**
  - **Strip the table.** Header row becomes Ink bold, no fill. Hairline rule below header row. Hairline rules between data rows are optional — recommend no body rules, just header rule. Cell padding ~8pt vertical.
  - First-column carriers (cargo.one, CargoWise CTO, etc.) stay bold.
  - The phrase "Zero overlap on consolidation" in the cargo.one row — make this run-level Accent. One accented phrase in the table.
  - Competitive scenarios: replace bullets with the same numeric label pattern (`01 / 02 / 03`). For the WiseTech scenario (the primary risk), append the phrase **"primary risk"** in Signal color (`#B4541C`) — single, deliberate use of the second accent.
- **Why:** the v2 table with the dark-blue header is the most retail-PowerPoint element in the deck. Stripping it is the highest-leverage single change.

### Slide 11 — Go-to-market

- **Current:** Distribution table, design partner contract bullets, trust ramp text.
- **Changes:**
  - Same table treatment as slide 10 — strip the header fill.
  - Pull in the integration sequence from slide 5 (3 items: GoFreight, CargoWise, Magaya/Riege) — but only as a single line of subtitle under the section header, not as a separate block, since the table below covers the same information.
  - Trust ramp: render as **3 horizontal pill shapes** (rounded rectangles, `MSO_SHAPE.ROUNDED_RECTANGLE`, stroked Hairline, 0.5" tall, 3 in a row across the live area). Each pill contains the phase label (`Co-pilot ≥4w`, `Supervised ≥8w`, `Autonomous per-lane`). Small `→` glyph between pills.
  - Closing line "Autonomy is earned, not claimed." — 14pt Ink bold, left-aligned below the pills.
- **New element:** the three pill shapes for trust ramp. Strong visual upgrade over the current paragraph text.

### Slide 12 — Team

- **Current:** Name, founder bullets, advisor placeholders, open-role placeholders.
- **Changes:**
  - Move `[YOUR NAME] — Founder & CEO` to display-s (32pt) **Ink**, not Accent. The name is the hero; Accent reserved for one credential.
  - Bullets: drop the `•` marker. Use a leading 14pt em-dash (`—`) and 0.2" indent. Reads as plain prose, which is more senior.
  - Add a 0.04" Accent vertical bar to the left of the founder block (matches slide 3 anecdote treatment).
  - Advisors section: change `(to add before sending)` parenthetical to a small Muted eyebrow label.
  - Open roles: render as three small stroked pill shapes (matches slide 11 trust ramp), each labeled with the role title. Subverts the "list of bullets" feeling.
- **Note:** since this is a single-founder deck with placeholders, the slide will feel sparse. That's correct. Sparse + confident reads better than padded with weak content.

### Slide 13 — Traction & ask

- **Current:** Two-column layout: "where we are" bullets left, "$2.5M seed" + use-of-funds right, milestones below.
- **Changes:**
  - "$2.5M seed" — keep at display-l (72pt would be better than current 42pt — bump up). Accent.
  - Right column: replace `THE ASK` eyebrow → `RAISE`. Replace `14-month runway` → smaller Muted caption directly under the $2.5M.
  - Use-of-funds: render as a horizontal stacked bar chart (single bar, 0.4" tall, full live-area width, divided into four segments with width ∝ percentage). Each segment in a different value of the Accent (50% → full Accent, 25% → Accent at 75% opacity, 15% → 50%, 10% → 25%). Labels above each segment with the role + %. This is the **single chart in the deck** — and it's a critical one (where the money goes).
  - "Where we are": numeric label pattern (`01 / 02 / 03 / 04`) instead of bullets, matches slides 4 and 8.
  - Milestones: 3 horizontal pills (matches slide 11), labeled `M6 / M9 / M14`, each with the milestone text inside or below.
- **New element:** the stacked horizontal bar chart for use-of-funds. Highest single visual upgrade on this slide.

---

## Section 4 — python-pptx implementation notes

**What's easy (just style swaps):**
- Color tokens: redefine the 5 `RGBColor()` constants. Done.
- Font name swap (Helvetica Neue → Inter): change the `font="Helvetica Neue"` defaults. Inter must be installed on the rendering machine; if not, falls back to system default. **Verify Inter is installed** before rendering or accept the Helvetica fallback.
- Removing fills: change `box.fill.solid(); box.fill.fore_color.rgb = COLOR_HIGHLIGHT` to `box.fill.background()` (no fill) or just don't fill.
- Switching border colors / widths: already parameterized in v2.
- Eyebrow uppercase labels: already implemented.
- Hairline rules between items: add `slide.shapes.add_connector(MSO_CONNECTOR.STRAIGHT, x1, y, x2, y)` and style the line.

**What needs clever workarounds:**

- **Letter-spacing on eyebrow labels:** python-pptx does not expose `tracking` / `letter-spacing`. Workaround: live with the un-tracked uppercase, or insert thin space characters (`U+2009`) between letters — ugly but works. Recommend: skip tracking, accept default kerning.

- **Inline color/weight changes within a paragraph:** add multiple `Run`s with different `r.font.color.rgb` and `r.font.bold`. The v2 code already does this in slide 3. Use for slide 2 vision (color the operator-readable phrase) and slide 10 competition table (color "Zero overlap on consolidation").

- **Variable opacity for stacked bar segments (slide 13):** python-pptx's `fore_color.rgb` is opaque. Workaround: pre-compute four mixed colors (Accent + Surface at 50% / 25% / etc. in RGB space) and use those as solid fills. Don't reach for transparency.

- **Three-circle macOS window chrome (slide 7):** add three `MSO_SHAPE.OVAL` shapes, ~0.12" diameter, 0.15" apart, top-left of the mock UI box. Fill Muted, no border.

- **Bento grid spanning rows of different heights:** there's no grid primitive — just compute positions manually. Use a helper `bento_cell(slide, row, col, row_span, col_span, ...)` that maps (row, col, span) to (x, y, w, h) on the 12-column grid. Worth writing once.

- **Pill shapes for trust ramp / open roles:** `MSO_SHAPE.ROUNDED_RECTANGLE` with maximum corner radius (set `adjustments[0]` to 0.5). Stroked, no fill. Text centered.

- **Vertical accent bar (slide 3, slide 6, slide 12):** add a thin `MSO_SHAPE.RECTANGLE` (0.04" wide, full height of adjacent content), solid Accent fill, no border. Position to the left of the content with a 0.2" gap.

- **Numeric labels `01 / 02 / 03`:** just text in eyebrow style (11pt, 700, Accent, uppercase). No special primitive.

- **Horizontal stacked bar (slide 13):** four `MSO_SHAPE.RECTANGLE` shapes in a row, each width proportional to percentage, fills in the four Accent-shades. Labels above as separate text boxes.

**Patterns to drop / simplify:**

- **No SVG, no embedded images of logos, no PNG icons.** Stick to shapes + Unicode glyphs. If a real partner logo is needed later (e.g., CargoWise / GoFreight on slide 5), it must be an image asset — but for v3, leave them as text references.

- **No drop shadows.** v2 already disables shadow (`box.shadow.inherit = False`). Continue this — apply to **every** shape.

- **No charts via `chart.add_chart()`.** python-pptx chart support is finicky and chart styling defaults look like Office 2010. For the one chart needed (slide 13 use-of-funds bar), construct it from rectangles manually. Far more control over appearance.

- **No table cell borders.** Set each cell border to invisible via the underlying XML (python-pptx doesn't expose this cleanly — use a helper that pokes the `a:tcPr` element to set `a:lnL`, `a:lnR`, `a:lnT`, `a:lnB` to `<a:noFill/>`). One-time helper; worth writing.

**A reusable helper kit the engineer should build first:**

```python
def hairline(slide, x1, y, x2):                 # horizontal rule
def eyebrow(slide, x, y, text):                 # uppercase 11pt 700 Muted
def section_title(slide, text):                 # display-s, Ink
def hero_number(slide, x, y, text):             # display-l, Accent
def vertical_accent_bar(slide, x, y, h):        # 0.04" Accent rectangle
def pill(slide, x, y, w, h, label):             # rounded rect, stroked
def bento_box(slide, x, y, w, h, *, fill=None): # 0.75pt Hairline border
def remove_table_borders(table):                # XML poke
```

Build these first, then refactor each slide to use them. This is the single highest-leverage thing the engineer can do — without these helpers, v3 will accumulate the same inconsistencies as v2 (mixed border widths, color drift, off-grid positions).

---

## Section 5 — Brand voice and tonal recommendations

**Capitalization:** sentence case for headlines and body. Title Case only for slide *labels* (eyebrows) — but render those uppercase, so the casing of the source string doesn't matter. Avoid Title Case in headlines ("Why We Win" → "Why we win"). Sentence case reads more confident and contemporary; Title Case reads as corporate / consulting.

**Bold:** use for one phrase per slide, max two. Bold is a single tool — don't dilute it. If multiple things are bold, none of them are. The v2 deck currently uses bold *very* liberally; tighten to roughly half its current frequency.

**Italic:** avoid almost entirely. Italic in PowerPoint reads dated. The two acceptable uses: book titles (none in this deck) and parenthetical asides (v2 has these — *(Placeholders to fill in...)*). Convert those to regular weight Muted with a leading em-dash instead.

**One bold word inline:** strong move when used twice in the entire deck. Examples: "We compress the consolidation planner's worst 3% from 40% of their day to **15%**." and "**$4–7 savings** per shipment, every shipment, from week 1." Reserve for the moments where a single number or phrase is doing the persuasive work.

**Sentence rhythm:** the v2 copy is already strong — short declarative sentences, em-dashes for parentheticals, no filler. Preserve all of it. No copy edits in v3 unless something is genuinely broken; this is a design pass, not a copy pass.

**One thing to remove from v2 copy in v3 (a design issue masquerading as a copy issue):**
- Slide 11: drop the trailing line *"Autonomy is earned, not claimed."* on the trust-ramp block — or, if kept, render plainly, not italic. Italic + epigraph in a B2B deck reads as "trying."

---

## Sources

- [Artisan Series A Pitch Deck — Best Pitch Deck](https://bestpitchdeck.com/artisan-series-a)
- [Top 50 Pitch Decks from B2B Startups — Failory](https://www.failory.com/pitch-deck/b2b)
- [Top 50 Pitch Decks from AI Startups — Failory](https://www.failory.com/pitch-deck/ai)
- [Pitch Deck Design Trends in USA Boardrooms 2026 — Pitchworx](https://pitchworx.com/pitch-deck-design-trends-dominating-usa-boardrooms-in-2026/)
- [11 Presentation Design Trends for Startup Pitch Decks in 2026 — Visible.vc](https://visible.vc/blog/startup-presentation-design-trends/)
- [Fonts and Colors We Use in Real Pitch Decks — Ink Narrates](https://www.inknarrates.com/post/best-fonts-and-colors-for-pitch-deck)
- [SaaS Typography Playbook — FullStop](https://fullstop360.com/blog/insights/branding/saas-typography-playbook-what-leading-companies-use)
- [Linear Brand Guidelines](https://linear.app/brand)
- [Design System Analysis: Linear — getdesign.md](https://getdesign.md/linear.app/design-md)
- [Design System Analysis: Vercel — getdesign.md](https://getdesign.md/vercel/design-md)
- [Best Font for Presentation 2026 — Whitepage Studio](https://www.whitepage.studio/blog/the-ultimate-guide-for-using-fonts-in-decks-presentations)
- [Perplexity Pitch Deck Review — Upmetrics](https://upmetrics.co/pitch-deck-examples/perplexity)
- [Investor Pitch Deck Guide 2026 — MasterRV](https://www.masterrvdesigners.com/blog/investor-pitch-deck-guide-examples-structure/)
- [Seed Round Pitch Deck Guide 2026 — Visible.vc](https://visible.vc/blog/seed-round-pitch-deck/)
- [python-pptx autoshapes documentation](https://python-pptx.readthedocs.io/en/latest/user/autoshapes.html)
