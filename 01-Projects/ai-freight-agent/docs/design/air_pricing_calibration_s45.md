# Air-Freight Pricing & Capacity Calibration — Transpacific Eastbound (S45)

**Purpose:** Sourced calibration of realistic air-freight rates and a finite, increasing-block scarcity
structure for the synthetic-data generator. Step 1 of 2 — this is the NUMBERS + provenance; the model
design agent consumes it next.

**Network:** TPE / PVG / HKG → LAX / ORD / SFO (transpacific eastbound).
**Weight scale:** HAWB 50–1,200 kg; full LD3 ≈ 1,500 kg chargeable; lane weekly demand few-thousand to
tens-of-thousands of kg.

**Data window:** Public reporting Sep 2025 – Apr/May 2026. Air-cargo rates are regime-dependent and
seasonal; all point estimates below are calibrated to a *normal-to-firm* transpacific market (not peak
spike, not trough). Peak/spike behavior is captured in the scarcity curve and fallback multiple.

---

## 1. Summary (the realistic picture)

Transpacific eastbound (Northeast Asia → North America) air cargo cleared at roughly **$5.5/chargeable-kg
spot in spring 2026** (Xeneta: NE-Asia→NA $5.54/kg week ending 26 Apr 2026; WorldACD: China→US ~$5.3/kg
in July, rising to ~$6.6–7.0/kg at the Dec peak). Contracted/BSA capacity sits **below spot** as a
stable, committed level — the spot premium over contract is **regime-dependent, roughly 1.2×–1.5× in
normal-to-firm markets and 2×–3× at peak** (industry rule-of-thumb: spot "2–3× contract during peak";
2026 YoY spot rose +30% vs long-term +18%). Spot is **not wildly volatile week-to-week** — typical
transpac WoW moves in 2025–26 were **±1–4%**, confirming the user's prior of a slow-moving multi-dollar
band rather than daily swings; large moves are episodic (fuel shocks, peak, geopolitics), not noise.
The key new structure: lane spot capacity is **finite and priced in increasing blocks** — the first
chunk clears cheap, the marginal clearing price steps up ~**1.15–1.25× per block** as the lane fills,
and there is a **hard weekly ceiling** (a single origin→gateway lane offers on the order of low-thousands
to ~10k chargeable-kg of *spot* space per week on top of contracted allotments) beyond which cargo rolls
to the next flight or goes to ad-hoc/charter at a **2–4× normal-spot** last-resort price. This lane-level
scarcity curve is **separate** from, and coexists with, the per-shipment IATA GCR weight-break discount
(a single booking's $/kg *decreases* as that one booking gets heavier).

---

## 2. Master calibration table

All rates are USD per **chargeable** kg unless noted. "Point" = recommended generator anchor for a
normal-to-firm transpac market. Confidence: **SOURCED** (public figure), **INFERRED** (derived/best-judgment
from sourced anchors), **MRN** (market research needed — no free public figure).

| Capacity type | $/chargeable-kg (range → point) | Capacity per flight / lane-week | Scarcity behavior | Source(s) + confidence |
|---|---|---|---|---|
| **Contracted / BSA allotment** | $3.5–5.0 → **$4.2** (normal-firm transpac; tracks ~0.7–0.85× of prevailing spot) | A mid-market forwarder BSA: **~1–4 ULD positions per flight** (≈1.5–6 t/flight); a few daily flights → **~10k–40k chargeable-kg/lane-week** of committed allotment. | Fixed/committed block. Priced via **pivot weight** per ULD: under-pivot charged at allotment rate, over-pivot at a lower marginal rate. Hard-BSA = pay-or-fly (pay whether used or not); soft-BSA = cancellable. Capacity is *capped at the contracted block*; overflow goes to spot. | Spot anchor SOURCED (Xeneta/WorldACD, §3); contract-as-fraction-of-spot **INFERRED**; pivot/allotment mechanics SOURCED (IATA TACT; pivot-weight refs); **per-forwarder position count = MRN** (no public figure — INFERRED range given). |
| **Spot (market)** | $4.5–7.0 → **$5.5** (NE-Asia→NA Apr-26 $5.54; China→US ~$5.3 Jul, ~$6.6–7.0 Dec peak; SE-Asia→NA $6.46) | Finite per-lane: **~3k–10k chargeable-kg/lane-week** of *uncommitted* spot space on top of allotments (INFERRED from ~60–67k t/wk whole-trade widebody freighter capacity split across all origin×gateway pairs and BSA pre-allocation). | **Increasing-block / rising marginal price** as lane fills (see §4). WoW volatility modest (±1–4%); episodic spikes +15–40%. | Spot levels & WoW **SOURCED** (Xeneta, WorldACD, IATA); per-lane spot kg ceiling **INFERRED**. |
| **Spot → fallback / ad-hoc / "must-ship-now"** | **2–4× normal spot** → point **2.5×** (≈ **$12–15/kg** at $5.5 base) for marked-up last-minute scheduled space; full charter is priced **per-operation**, effectively much higher per-kg on a single HAWB | Effectively the relief valve once contract+spot exhausted; not a standing weekly quantity — triggered per roll/emergency. | Hard ceiling on normal capacity → roll to next flight, heavily-marked-up spot, or ad-hoc charter. | Express/priority +20–40% **SOURCED**; "spot 2–3× contract at peak" **SOURCED**; charter "per-operation, significantly more expensive" **SOURCED**; the 2.5× last-resort point on *spot* base is **INFERRED**. |
| **Per-shipment GCR weight break** (NOT a separate capacity tier — a discount applied within any booking) | Declining $/kg with booking weight; breakpoints **N / +45 / +100 / +250–300 / +500 / +1000 kg** | n/a (applies to a single MAWB/HAWB booking) | $/kg **decreases** as one booking gets heavier (quantity discount); coexists with §4 lane scarcity. | IATA TACT weight-break structure **SOURCED**; exact per-break $/kg deltas **MRN**. |
| **Volumetric / chargeable-weight convention** | n/a | n/a | Chargeable wt = max(actual, volumetric); volumetric kg = volume(cm³)/6000 = **166.67 kg/m³** | IATA 1:6 / divisor-6000 standard **SOURCED**. Model's 167 kg/cbm is correct (rounds 166.67). Some carriers use 5000 → 200 kg/m³ (variant, not standard). |

---

## 3. Sourced rate anchors (spot, transpacific eastbound)

| Period | Lane | $/chargeable-kg | Source |
|---|---|---|---|
| Wk ending 26 Apr 2026 | **Northeast Asia → North America** (≈ PVG/TPE/HKG → US) | **$5.54** | Xeneta |
| Wk ending 26 Apr 2026 | Southeast Asia → North America | $6.46 | Xeneta |
| Jul 2025 | China → US | ~$5.30 | Air Cargo News / WorldACD |
| Wk 38 (Sep 2025) | Asia-Pacific → US | $4.79 | WorldACD |
| Wk 46 (Nov 2025) | Asia-Pacific → US | $5.51 | WorldACD |
| Wk 49–50 (Dec 2025) | Asia-Pacific → US | $6.32 → **$6.57** | WorldACD |
| Wk 50 (Dec 2025) | Hong Kong → US | $6.92 | WorldACD |
| Wk 50 (Dec 2025) | China → US | $6.96 | WorldACD |
| Dec 2025 (monthly) | Hong Kong → North America | $6.60 (+6.7% MoM, 2025 high) | Air Cargo News / TAC |
| Apr 2026 | Global air-cargo spot (all lanes) | $3.34 (+30% YoY) | Xeneta |
| Dec 2025 | Global air-cargo spot (all lanes) | $2.83 (−4% YoY) | Xeneta/WorldACD |

**Reading:** Transpac eastbound spot runs **roughly 1.8–2.5× the global all-lane average** — it is a
premium headhaul. A normal-firm point of **$5.5/kg** is well supported; the band **$4.5–7.0/kg** spans
shoulder season to the Dec peak. Off-peak troughs can dip toward ~$4.5; sustained sub-$4 transpac spot
is unusual in this window.

**Volatility (the user's $3–5 band prior):** Confirmed in spirit. WoW spot moves on transpac in 2025–26
were **±1–4%** (e.g. Dec-25 spot −1% WoW; WorldACD weekly trends rarely exceed low single digits absent
a shock). So week-to-week the rate sits in a slow-moving multi-dollar band, not a wild swing. The user's
"$3–5/kg band" is a reasonable *normal-market* envelope for the slow component; the firm/peak transpac
level sits **above** that band ($5.5–7), so the generator's normal band is better stated as **~$4.5–6.5/kg
with ±1–4% weekly drift**, plus rare episodic regime jumps. Big moves (+30% YoY in Apr-26, +15–40% peak
spikes) are **episodic regime shifts**, not weekly noise — model them as occasional state changes, not
Gaussian week-to-week.

---

## 4. The lane-level spot supply curve (THE KEY NEW PIECE)

**Concept.** On a given lane-week, spot space is finite and clears in **increasing-price blocks**: airlines
and the market release cheap space first; as the lane fills toward departure, the marginal clearing price
rises and remaining space shrinks. This is the mechanism that makes "tighten contracted capacity → cost
goes up" actually bite, because the spot escape is now finite and rising, not flat-and-bottomless.

**Sourced foundation:**
- Whole-trade widebody-freighter capacity ran **~60,000–67,000 t/week** Asia-Pacific→NA in 2025
  (peak 75k in late-Mar front-loading; SOURCED, Air Cargo News). Belly capacity adds materially on top.
- That total is split across **many origin × US-gateway pairs** and is **largely pre-sold to BSA/allotment
  holders**; only a slice is open uncommitted spot at any time.
- Route density matters: dense lanes (e.g. Shanghai/HKG → LA) price **20–30% cheaper per kg** than thin
  lanes; thin-lane / secondary spikes run **+40–50%** (SOURCED, WebCargo). This is the cross-sectional
  analogue of the within-lane block curve.

**Parameterization for the generator (INFERRED block shape, anchored to SOURCED levels):**

Per single origin→gateway lane-week, define a finite spot supply schedule. Suggested default (tune per lane
by density):

| Block | Width (chargeable-kg) | Marginal $/kg (multiplier on base spot $5.5) | Notes |
|---|---|---|---|
| B0 (cheap release) | first **~5,000 kg** | **1.00×** → $5.5 | early/abundant space |
| B1 | next **~3,000 kg** | **~1.20×** → $6.6 | lane filling |
| B2 | next **~2,000 kg** | **~1.20² ≈ 1.44×** → $7.9 | tight |
| B3 (last space) | next **~1,000–1,500 kg** | **~1.20³ ≈ 1.73×** → $9.5 | near-full, scarce |
| **Hard ceiling** | total spot ≈ **~10,000–12,000 kg/lane-week** | — | beyond this: roll / fallback (§5) |

- **Step multiplier:** the user's ~**1.2× per block** is a reasonable, defensible default. A plausible
  range is **1.15×–1.25×** per block. Compounding 3–4 blocks reaches ~1.7–2.0× base — which lands the
  last cheap-spot space near the bottom of the fallback range, a consistent hand-off.
- **Block widths:** first block widest (cheap, abundant), later blocks narrower (space shrinks as it
  fills) — i.e. width *decreases* as price *increases*. The ~5k / 3k / 2k / 1–1.5k schedule encodes this.
- **Finite ceiling:** ~**10–12k chargeable-kg of spot per lane-week** (on top of contracted allotments)
  is a defensible INFERRED ceiling for a major transpac lane; scale down for thinner lanes. When demand
  on a lane exceeds allotment + this spot ceiling, the model must invoke the §5 fallback (roll/charter).
- **Calibration knob:** the design agent should make the *number of blocks unlocked* and the *ceiling*
  a function of network tightness (κ) so that tightening BSA pushes demand up the block curve and raises
  realized cost — the whole point of the redesign.

**Coexistence with the per-shipment weight break (keep these orthogonal):**
- **Lane scarcity curve (§4):** price **rises** as the *lane* fills across many bookings. Marginal-cost-of-
  capacity effect. Applies to the lane's remaining space.
- **GCR weight break (§2 last row):** price **falls** as a *single booking* gets heavier (N/+45/+100/
  +250–300/+500/+1000 kg). Quantity-discount effect. Applies within one MAWB/HAWB.
- Both are real and simultaneous: a heavy single HAWB gets a better per-kg *base* rate (weight break),
  but the *block* it lands in depends on how full the lane already is (scarcity). Generator should apply
  the weight-break discount to the *base* spot rate, then apply the block multiplier for the lane's fill
  state.

---

## 5. Fallback / ad-hoc / must-ship-now pricing

Once contracted allotment + finite spot are exhausted on a lane-week, the realistic relief valve is:
1. **Roll to next flight** (no extra $/kg, but a transit-time/SLA penalty — handled by the routing model,
   not a price here).
2. **Heavily-marked-up last-minute scheduled space:** **2–4× normal spot**; point estimate **2.5×**
   (≈ **$12–15/kg** at a $5.5 base). Express/priority premiums of **+20–40%** (SOURCED) are the *mild* end;
   genuine must-ship-now scarcity pushes well past that, and "spot 2–3× contract at peak" (SOURCED)
   supports the multiple.
3. **Ad-hoc charter:** priced **per-operation**, not per-kg (SOURCED). For a single HAWB the effective
   per-kg is very high; only rational when consolidating a large block or for AOG/emergency. Treat as a
   capped, expensive backstop rather than a smooth price — i.e. a high fixed-cost option the model can
   invoke, not part of the smooth block curve.

**Generator recommendation:** model fallback as a **single high marginal price at 2.5× base spot** for the
last-resort scheduled tier, with charter as a separate large-fixed-cost option above it. This preserves
the "expensive but feasible" backstop (do **not** prune it on standalone cost — per project memory
*No-Standalone-Cost-Pruning*: under tight supply the expensive option is the feasible fallback that
prevents stranding HAWBs).

---

## 6. Volumetric / chargeable-weight convention

- **IATA standard dimensional factor: 1:6**, i.e. volumetric weight (kg) = volume(cm³) / **6000** =
  **166.67 kg/m³**. The model's **167 kg/cbm is correct** (rounded). SOURCED.
- Chargeable weight = **max(actual weight, volumetric weight)**. SOURCED.
- Carrier variant: some use divisor **5000** → **200 kg/m³** (penalizes low-density cargo more). Not the
  IATA standard; flag as an optional per-carrier knob if low-density realism is needed, otherwise keep 167.

---

## 7. Honesty ledger — sourced vs inferred vs needs-research

**SOURCED (public figure cited):**
- Transpac/NE-Asia→NA spot levels and seasonal path: $4.79 (Sep) → $5.51 (Nov) → $6.57 (Dec) Asia-Pac→US;
  NE-Asia→NA $5.54 and SE-Asia→NA $6.46 (Apr-26); HKG→NA $6.60 Dec; China→US ~$5.3 Jul (WorldACD, Xeneta,
  Air Cargo News / TAC).
- Global all-lane spot: $2.83 (Dec-25), $3.34 (Apr-26, +30% YoY); long-term +18% (Xeneta).
- WoW volatility modest (≈±1–4%; Dec-25 spot −1% WoW) — episodic spikes +15–40% / +30% YoY (WorldACD/Xeneta).
- Spot ≈ 2–3× contract at peak; express/priority +20–40%; charter priced per-operation (industry/WebCargo).
- Whole-trade Asia-Pac→NA widebody freighter capacity ~60–67k t/wk (peak 75k) — Air Cargo News.
- Pivot-weight / allotment mechanics (under-pivot vs over-pivot rate; hard vs soft BSA) — IATA / pivot refs.
- GCR weight-break ladder N/+45/+100/+250–300/+500/+1000 kg — IATA TACT.
- LD3 (AKE) max gross **1,588 kg** (≈1,500 kg chargeable planning number) — SeaRates/DSV.
- IATA volumetric divisor 6000 → 166.67 kg/m³; chargeable = max(actual, volumetric) — IATA standard.

**INFERRED (derived from sourced anchors / best judgment, labelled in-table):**
- Contract/BSA $/kg as ~0.7–0.85× prevailing spot → ~$4.2/kg point (no clean public transpac contract $/kg).
- Spot premium-over-contract ratio ~1.2–1.5× normal, 2–3× peak.
- Per-lane finite spot ceiling ~10–12k chargeable-kg/lane-week; block widths (5k/3k/2k/1–1.5k) and
  1.2× (range 1.15–1.25×) step multiplier; last-resort fallback at 2.5× base spot.
- Mid-market forwarder BSA holding ~1–4 ULD positions/flight (≈10–40k kg/lane-week committed).

**MARKET RESEARCH NEEDED (no free public figure; license required for granularity):**
- Exact transpac **contracted/BSA $/kg** by lane (TAC/Xeneta lane-level contract series are paid).
- Exact per-break **GCR $/kg deltas** by carrier/lane (TACT Rates is licensed).
- Actual **ULD positions per flight** a typical mid-market forwarder's BSA holds on these specific lanes
  (commercial/contract data; no public source — confidentiality bars internal rate cards).
- Lane-resolved (origin→specific US gateway) **spot capacity in kg/week** (derived here from whole-trade
  totals; lane-level split is paid/internal).

---

## 8. Sources

- Xeneta — Global air cargo spot rates hit three-year high in April (NE-Asia→NA $5.54, SE-Asia→NA $6.46,
  global $3.34 +30% YoY, long-term +18%): https://www.xeneta.com/news/global-air-cargo-spot-rates-hit-a-three-year-high-in-april-but-market-fundamentals-will-calm-costs-for-shippers
- WorldACD Weekly Air Cargo Trends 2025 (wk 38 $4.79; wk 46 $5.51; wk 49–50 $6.32–6.57; HK $6.92; China→US $6.96):
  https://www.worldacd.com/trend-reports/weekly/worldacd-weekly-air-cargo-trends-2025-week-50/ ;
  https://www.worldacd.com/trend-reports/weekly/worldacd-weekly-air-cargo-trends-2025-week-38/
- Air Cargo News — transpac widebody freighter capacity ~60–67k t/wk (peak 75k):
  https://www.aircargonews.net/data-news/transpacific-widebody-freighter-capacity-settles-after-market-volatility/1080234.article
- Air Cargo News — HKG→NA $6.60 Dec-25 (+6.7% MoM, 2025 high); China→US ~$5.3 Jul:
  https://www.aircargonews.net/data/2026/01/air-cargo-rates-on-key-trades-end-the-year-on-a-high/
- TAC Index / Baltic Air Freight Index (BAI) — spot/contract index, WoW moves:
  https://www.tacindex.com/ ; https://dashboard.tacindex.com/
- Supply Chain Dive / Xeneta — contract vs spot behavior, spot 2–3× at peak, Q4-25 spot share:
  https://www.supplychaindive.com/news/air-cargo-contract-behavior-shifts-rates-ecommerce-xeneta/809284/ ;
  https://www.supplychaindive.com/news/air-cargo-spot-rates-surge-30-in-april/819243/
- WebCargo — transpac per-kg ($3.5–7.0 Shanghai→Chicago), density 20–30% cheaper, thin-lane +40–50%,
  express +20–40%, fuel 15–30%:
  https://www.webcargo.co/blog/air-cargo-price-per-kg/
- IATA — Air Cargo Tariffs and Rules (GCR weight breaks, TACT):
  https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/air-cargo-tariffs-and-rules-what-you-need-to-know/
- SeaRates / DSV — LD3 (AKE) max gross 1,588 kg:
  https://www.searates.com/reference/uld/ld3/ ; https://www.dsv.com/en-us/our-solutions/modes-of-transport/air-freight/unit-load-devices/ld3-ake-ave-container
- Pivot-weight mechanics (under/over-pivot, allotment): https://presou.com/the-ultimate-guide-to-pivot-weight-air-freight-what-it-means-and-how-it-impacts-cargo-logistics/
- Maersk — chargeable weight / IATA volumetric divisor 6000 (166.67 kg/m³):
  https://www.maersk.com/logistics-explained/transportation-and-freight/2025/03/10/air-cargo-chargeable-weight
- IATA Air Cargo Market Analysis (monthly, global avg context):
  https://www.iata.org/en/iata-repository/publications/economic-reports/air-cargo-market-analysis-december-2025/
