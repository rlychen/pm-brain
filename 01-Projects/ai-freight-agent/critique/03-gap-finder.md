# Air Freight Routing v3 — Operational Gap Finder

**Audit target:** `model/air_freight_routing.tex` v3 (2026-05-23), plus
`model/air_graph_construction.md` and the surcharge / ULD / rule subsystems
documented inline.

**Scope rule:** items already on the §Open Items / §Deferred P1 list are
**excluded by construction**. That list covers TT quantile binding, offload
priority, SLA-endpoint / max-hops, split shipment, pairwise DG, temp-band
refinement, FX pinning, multi-tier per-ULD over-pivot, lock-buyout as MILP,
full 3D bin-packing, CFS demurrage cost, probabilistic clearance,
probabilistic transit, ULD availability at CFS, CFS dock capacity, multiple
ULD types per HAWB, alliance slot sharing, spot capacity uncertainty, and
the §"Excluded from MVP" basket. Findings below are **net-new**.

**Out of scope per founder instruction:** math correctness, big-M, LP-gap,
scalability tightness, formulation elegance.

Severity ladder:
- **Blocking** — model cannot be used as a planning tool without it; produces
  wrong cost or unplannable routes routinely.
- **Serious** — model technically runs but operators revert to spreadsheet
  in a meaningful fraction of solves; trust degrades.
- **Nice-to-have** — fix improves quality but no daily reversion.

Fix-size: small ≈ data field + ingestion rule; medium ≈ parameter / pre-filter
change; large ≈ new constraint family or new component.

---

## 1. Data-field gaps (fields the model assumes are clean)

### 1.1 Declared shipment value `value_k` — operator-side completion rate
- **One-liner:** `value_k` is a required ingestion field that drives the
  whole quadratic tardiness weight `W_k = w_sp(k) · μ_k`; in practice
  shipper-declared cargo value is often missing or set to a low default
  (CIF=1.10·invoice convention), and operators infer it from HS code +
  invoice context.
- **Severity:** **serious**. With `value_k` defaulting to the invoice line
  total or a placeholder, `μ_k = value_k / V^ref` collapses cargo-priority
  ordering. The model will spend $X on premium routing for low-actual-value
  shipments and accept lateness on undeclared-but-high-value shipments.
- **Why missed:** the spec treats `value_k` as a clean intake field, but
  forwarder shippers commonly leave declared value blank on the HAWB until
  insurance is bound (or never, on routine FAK lanes). The
  field-completion rate is a property of the customer, not the system.
- **Fix size:** small — add an ingestion validator with three tiers
  (declared / invoice-derived / category-default) and surface the tier in
  output diagnostics so operators see "tardiness ranked using inferred value".

### 1.2 Cargo-ready window flexibility — `t_k^{rdy,late} - t_k^{rdy,early}`
- **One-liner:** the model expects two distinct UTC timestamps, but shippers
  routinely communicate "ready Wed AM" with no late bound, or "ready when
  truck arrives" with no early bound; operators stuff in defaults.
- **Severity:** **serious**. When the window is artificially tight (early=late
  because intake collapsed both to one time), the model loses its key lever
  for cutoff fitting and consolidation timing. When artificially wide
  (default `+72h`), it under-prices the pickup-deferral degree of freedom and
  permits routings that miss real-world pickup constraints.
- **Why missed:** ingestion-layer concern; spec assumes both bounds exist as
  inputs (§Per-HAWB parameters table). Day-in-the-life shows pickup-window
  capture happens via free-text email or WhatsApp, not a calendar field.
- **Fix size:** small — explicit ingestion contract: window must be a
  shipper-confirmed pair or a tenant-default with a `confidence` tag.

### 1.3 Lithium-spec field completeness vs. DDR / SOC / UN-number capture
- **One-liner:** `lithium_spec_k` requires UN number, PI code, section, Wh,
  SOC compliance boolean, DDR boolean — but the day-to-day reality is that
  shippers self-declare "lithium battery" with sub-field gaps; operators
  chase the DG declaration over multiple back-and-forths.
- **Severity:** **serious**. The lithium pre-filter is whitelist-default-deny
  (Eq. lithium-ok), so an incomplete spec rejects an arc that would otherwise
  be feasible, pushing the HAWB onto the fallback arc. Operators will see
  routinely-infeasible-real-arc-sets on lithium HAWBs and stop trusting the
  solver.
- **Why missed:** spec describes the structure of `lithium_spec_k` but not
  what the model does when a sub-field is null. The model's default-deny
  posture is operationally correct for safety but produces a high false-
  negative rate without a parallel "declaration-pending" state.
- **Fix size:** medium — introduce a `lithium_decl_status_k ∈
  {complete, pending, refused}` flag; when `pending`, route HAWB through a
  parallel "soft-deny on lithium-restricted arcs but admit lithium-agnostic
  ones" rule, and surface in operator UI.

### 1.4 Piece dimensions vs. oversized / non-stackable cargo realities
- **One-liner:** `piece_dimensions` is a list of (L, W, H) cm per piece, but
  the model only consumes `max_single_piece` for the ULD-fit pre-filter and
  has no concept of `non_stackable`, irregular shape, or required orientation.
- **Severity:** **nice-to-have** at MVP, but trending toward serious. The
  pre-filter rejects "no single ULD type fits this piece" but admits a
  multi-piece tall thin shipment that would in fact require two ULDs or a
  contour rejection at acceptance.
- **Why missed:** §ULD spec talks about contour, floor load, fill cap, but
  the per-HAWB pre-filter only uses bulk weight and bulk volume; it doesn't
  carry stackability or fragility, despite `shipment_attributes.md`
  documenting `stackable: boolean`. Information is in the data model, not
  in the filter.
- **Fix size:** small — pre-filter extension to also reject when
  non-stackable cargo demands more ULDs than `cap_a` admits.

### 1.5 Net vs. gross weight reconciliation
- **One-liner:** `w_k` is "gross weight," but air-freight invoicing
  distinguishes gross actual weight, gross weight per IATA scale tolerance
  (±0.1 kg at the airline scale), and chargeable weight. The model treats
  gross as a single number; operators routinely see ±5–10% variance between
  shipper-declared and airline-weighed.
- **Severity:** **nice-to-have**. The model's chargeable-weight aggregation
  is correct; the problem is that the planning `w_k` may differ from the
  acceptance `w_k`, which can flip a pivot-floor binding decision after
  booking.
- **Why missed:** spec assumes one weight per HAWB. Reality: shipper-declared
  weight is provisional until airline scale reading.
- **Fix size:** small — `weight_tolerance` parameter in the BSA pivot floor
  test; output diagnostic when planning weight is near a pivot boundary.

### 1.6 HS-code → cargo-type compatibility, not modeled
- **One-liner:** `commodity_codes_hs` is captured but the cargo-type-ok
  pre-filter uses only `cargo_class` (GEN/DGR/PER/...). HS code drives
  destination-side PGA/customs feasibility, screening exemptions, and some
  carrier acceptance rules that the cargo-class enum collapses.
- **Severity:** **nice-to-have**. The five-layer carrier policy cascade can
  absorb HS-keyed rules, but the model doesn't surface this hook.
- **Why missed:** the cascade is keyed on "commodity overlay" abstractly;
  no worked example shows HS-keyed denial.
- **Fix size:** small — extend the carrier-policy schema to allow
  `hs_prefix` as a scope key.

---

## 2. Rate-structure gaps (pricing arrangements not in the catalog)

### 2.1 Fuel surcharge as an index-linked pass-through, not a static surcharge
- **One-liner:** §Surcharge treats FSC as a per-kg amount in the surcharge
  catalog with effective-from/to dates. Real FSCs are index-linked
  (carriers publish monthly fuel-index brackets — IATA Fuel Price Monitor,
  US Gulf Coast Jet Kerosene published by the US EIA — and FSC moves to
  the next bracket within the active period). The model can't represent
  "FSC = bracket A from price ≤ Xc/gal, bracket B from Xc/gal to Yc/gal".
- **Severity:** **serious** in volatile fuel periods (2022, post-Red-Sea
  re-routing). Stale FSC values quote shipments at the wrong cost; if
  bracket has moved one tier in either direction, error is 5–10% of total
  freight cost.
- **Why missed:** surcharge catalog is a flat `(scope, rate, validity)`
  schema; no index-binding mechanism.
- **Fix size:** medium — add `index_basis` field on surcharge records with
  a `fuel_index_lookup(date) → bracket` resolver run at ingestion (not in
  MILP); the catalog then resolves to a concrete per-kg value per solve.
  Sources: IATA Jet Fuel Price Monitor
  ([iata.org/en/publications/economics/fuel-monitor/](https://www.iata.org/en/publications/economics/fuel-monitor/));
  US EIA Gulf Coast Jet Kerosene weekly index.

### 2.2 IATA Cargo Account Settlement Systems (CASS) timing — not modeled
- **One-liner:** Air-freight settlement runs through IATA CASS (Cargo Account
  Settlement Systems) with bi-monthly billing cycles per country
  ([iata.org/.../cargo-account-settlement-systems-cass/](https://www.iata.org/en/services/finance/cass/)).
  Forwarders' working-capital cycle is keyed to CASS billing dates; choosing
  a carrier whose CASS market has favorable settlement timing has real cash
  cost the model can't see.
- **Severity:** **nice-to-have** for routing-cost minimization, but
  **serious** for any MVP that claims to support quote-desk margin reasoning.
- **Why missed:** the model is structurally cost-minimization; CASS settlement
  is a cash-timing concern that doesn't fit the routing objective.
- **Fix size:** small — out of MVP scope but flag at intake so operators
  know it isn't modeled (avoid silent wrong margin numbers downstream).

### 2.3 Multinational (MNC) master contracts spanning origins
- **One-liner:** Big shippers (Apple, Samsung, Pfizer) sign master rate
  agreements that fix a specific carrier × lane bundle for the entire
  contract year regardless of forwarder's local spot view. The model's
  shipper-lane allow/deny captures the negative case (this shipper denies
  carrier X) but not the positive contract-rate case where a specific
  shipper × lane requires a specific carrier at a specific rate even when
  cheaper alternatives exist.
- **Severity:** **serious** on accounts of any size. Operators will reject
  the optimizer's recommendation in favor of the contract carrier — and
  the model has no way to know the chosen carrier was contractually
  mandated, not preferred.
- **Why missed:** §Carrier policy treats this as the soft-preference layer
  (lexicographic Pass 2). Pass 2 has an `ε_pref` cost ceiling, so the
  contracted carrier is selected only if it is within ceiling of optimal.
  Hard-contract carriers ride outside that ceiling.
- **Fix size:** small — add a `contract_mandate` field to the
  shipper-lane layer (escalates the carrier from `pref` to `allow ∩
  carrier=X`), narrowing the allowed-carrier set.

### 2.4 Charter-fragment buys (partial-charter participation)
- **One-liner:** When peak demand spikes (CNY pull, semiconductor outage,
  pharma launch), forwarders buy fragments of an ad-hoc charter — e.g.
  one PMC + two AKEs on a 747F charter someone else organized. Excluded
  from MVP list says "charter, broker-of-record, alliance slot sharing"
  are out of scope.
- **Severity:** **serious** in spike periods, **nice-to-have** in steady
  state. On routes like TPE-EU during the 2024 Red Sea air-converse
  spike, 20–40% of mid-market forwarder air capacity moved on
  charter-fragment buys. Excluding them silently produces "no capacity"
  answers when the operator has real capacity in their email.
- **Why missed:** explicitly excluded in §Excluded from MVP — but the
  exclusion is broader than charter-as-broker-of-record. Charter-fragment
  participation is the operator's option to buy into a charter manifest
  organized by a co-loader, structurally similar to a co-load arc with
  a hard cap.
- **Fix size:** small — model as a `coload_per_kg` arc with a hard
  `cap_a^cl` (the seats on the fragment), no new constraint family.

### 2.5 Allocation-pull-forward (use-it-or-lose-it monthly BSA mechanics)
- **One-liner:** Hard BSAs typically have *monthly* utilization minimums
  enforced via `if you didn't hit P_{c,t-1} last month, your t+1 month
  allotment is cut by X%`. The equalized-allowance mechanism in v3
  models the *sunk* allowance against the current period but doesn't
  model the *forward consequence* of under-utilization on the next
  period's allotment.
- **Severity:** **serious** for the allocation manager persona. They
  routinely pull cargo forward into the current period to keep next
  period's allotment intact (or push back to defer a cut). The model
  treats each batch as independent of inter-period carry-over.
- **Why missed:** §Equalized settlement explicitly says "BSA period
  take-or-pay as a hard count minimum" was dropped to avoid spurious
  infeasibility. The fix is correct *for that batch* but loses the
  cross-period feedback loop.
- **Fix size:** medium — needs a rolling allocation-state input (this
  month's run-rate; next month's expected allotment delta) that gets
  baked into the marginal rate `r_c` as a shadow cost. Out of MVP body,
  but should be a clearly named hook.

### 2.6 GSA override rates (negotiated below TACT)
- **One-liner:** §Supply types lists GSA as feeding `min_flat_breaks` or
  `flat_rate`, but real GSA quotes routinely include silent below-TACT
  overrides on specific commodity codes or shipper segments. The catalog
  schema doesn't carry an explicit "override identity" — multiple GSAs
  may quote the same lane at structurally similar but commercially
  different rates.
- **Severity:** **nice-to-have**. Operators today work around this by
  manually editing rates in the catalog. The risk is that a stale GSA
  rate persists unnoticed.
- **Why missed:** the catalog treats each offer as opaque post-resolution;
  there's no provenance / sourced-by field.
- **Fix size:** small — add `source_party` to rate-offer schema; surface
  in solver output for operator review.

---

## 3. Carrier-policy nuances (beyond the 5-layer cascade)

### 3.1 Time-windowed and scenario-dependent rules — explicitly deferred but operationally critical
- **One-liner:** §Carrier policy §Out of MVP scope says "Time-windowed
  rules, conditional rules (`prefer freighter when commodity is
  electronics`), ML preference weights from override history — all
  deferred." The exclusion is comprehensive enough to call out: peak-
  season carrier shifts (avoid X carrier during Hajj; avoid Y during
  monsoon) are time-windowed by definition.
- **Severity:** **serious** in seasonal lanes. Without time-windowed rule
  support, the operator overrides 20–40% of solver recommendations during
  peak. The override-rate is the KPI; this gap directly hits it.
- **Why missed:** acknowledged as deferred. The gap-finding observation is
  that this isn't an edge case — it is the predominant rule shape during
  Q3/Q4 and at religious-holiday periods.
- **Fix size:** medium — extend rule schema with `effective_from`,
  `effective_to`, plus a `recurring_window` JSONB. Sources: carrier
  embargo portals (e.g., Emirates SkyCargo embargo page) routinely
  publish time-windowed rules in this exact structure.

### 3.2 Alliance allow-list with operating-vs-marketing nuance under codeshare
- **One-liner:** §Carrier policy has the op-vs-mk distinction for
  individual carrier names, but in practice shippers say "any Star
  Alliance carrier is allowed on this lane" — which is an alliance-level
  allow, not a carrier-level allow. The model has no `alliance` index.
- **Severity:** **nice-to-have**. Operators can manually enumerate the
  alliance members into the allow-list, but it's brittle when alliance
  membership changes (Asiana joining KE post-merger; Air Italy exit;
  alliance ULD-pool churn).
- **Why missed:** scope decision — operating carrier identity is the
  modeled scope. Alliances are referenced for ULD interchange but not
  for policy.
- **Fix size:** small — add `alliance_allow` and `alliance_deny` as
  layer-3 rule attributes; resolver expands at solve time.

### 3.3 Embargo-by-route nuance (embargoed for terminating cargo only, not transit)
- **One-liner:** §Embargo gating is leg-level; the same carrier on the
  same leg is either embargoed or not. Real embargoes routinely
  distinguish "no cargo *terminating* at city X" from "transit allowed."
  An embargo for terminating-cargo would block all of POL → POD = X
  flights and all POL → X → POD interlines where city X is the POD;
  the model can express the first but not the second cleanly.
- **Severity:** **nice-to-have**. The 5-layer scope rule (`hub(e)` field)
  partially solves this when `hub` field is set. But the model is
  structurally arc-level; "embargoed only if X is the endpoint" requires
  reasoning over the full path the cargo will take, which is decision
  output, not embargo input.
- **Why missed:** embargoes are evaluated per-arc, but the embargo's
  semantics may depend on the position of the embargoed node in the path.
  Workaround: pre-filter the *interline through-arc* differently from the
  *segment-arc* — currently both are treated identically.
- **Fix size:** medium — add an `embargo_role` field (`terminating` /
  `transit` / `any`) and update the graph generator's pre-filter logic to
  reason about the embargoed node's role in each arc.

### 3.4 Per-flight carrier-side capacity allowance (NOT flight `W_f, V_f`)
- **One-liner:** §Problem statement explicitly excludes flight-level
  physical capacity, correctly noting forwarders can't see other parties'
  bookings. **But** carriers do publish per-forwarder, per-flight cargo
  allowances on top of the BSA — e.g., "you have 1 PMC + 2 AKE on this
  flight today, regardless of your monthly BSA" — and these flow as
  daily/weekly limits.
- **Severity:** **serious**. The model's BSA `N_{a,u}` is a per-arc
  allotment that does not flex with day-of-week availability. Operators
  routinely get a 50% capacity cut on a specific day due to passenger
  cargo overload (PAX-belly aircraft are first-call for catering and
  baggage; cargo positions are released after pax-load close).
- **Why missed:** the BSA structure is modeled at the recurring-contract
  level; daily allotment variance is a real but not-modeled property.
- **Fix size:** medium — `N_{a,u}` becomes time-varying or an additional
  `daily_overlay_cap` parameter is added; either way the catalog schema
  has to admit a date-indexed value.

---

## 4. Customs interactions (points where customs affects routing)

### 4.1 In-bond / T&E movement under one customs envelope
- **One-liner:** US in-bond (T&E = Transportation and Exportation) and
  IT (Immediate Transportation) lets cargo enter a US port and continue
  to another US port under one customs entry, with the entry filed at
  destination. The model treats every cross-border move as a "customs
  clearance dwell at destination CFS" — it has no concept of in-bond
  movement that defers the entry.
- **Severity:** **serious** for any routing through US gateway hubs
  (LAX → ORD → EWR; JFK → ATL; etc.). The dwell at the intermediate
  hub is roughly half what the model assumes when in-bond is used.
- **Why missed:** §Ground-arc parameters folds `δ_cust_k` into
  destination-side dwell; in-bond is not surfaced. The §"In-transit hub
  customs" paragraph says T&E folds into the deconsol-arc dwell, but
  that's only when the hub has a CFS-H — and US T&E is most useful
  precisely at hubs without a CFS-H (carrier-side hub connections).
- **Fix size:** medium — graph generator emits an `in_bond_eligible`
  boolean on hub arcs; pre-filter uses it to inflate the carrier-side
  connection dwell down when in-bond is feasible.
  Source: CBP regulations 19 CFR 18
  ([cbp.gov/.../in-bond-process](https://www.cbp.gov/trade/cargo-security/cargo-control/in-bond-process)).

### 4.2 Post-Brexit GB ↔ EU complexity
- **One-liner:** GB and EU are now separate customs territories. Cargo
  flowing GB → EU requires a full export declaration GB-side + import
  declaration EU-side, plus possible UK Border Target Operating Model
  (BTOM) checks at the GB outbound airport. The model's
  `δ_cust_k` is a single per-HAWB scalar keyed on destination country;
  it can't represent the *origin-export* customs filing burden GB-side.
- **Severity:** **serious** for any tenant with significant LHR / STN /
  MAN outbound. The dwell math underestimates the GB-side delay
  routinely.
- **Why missed:** §Ground-arc parameters scope `δ_cust_k` to import side.
  Export-side filing folds into the POL effective cutoff (per
  §Graph construction), but BTOM physical-check holds are post-tender
  delays, not pre-cutoff.
- **Fix size:** small — additional `δ_cust_export_k` scalar applied at
  origin CFS when origin is a customs-territory-crossing departure.
  Sources: HMRC BTOM
  ([gov.uk/.../border-target-operating-model-august-2023](https://www.gov.uk/government/publications/the-border-target-operating-model-august-2023)).

### 4.3 Customs-broker capacity at destination
- **One-liner:** Per-HAWB `δ_cust_k` is computed by lookup on
  `(destination_country, ctype_k, commodity_class)` — it has no concept
  of the destination customs broker's capacity. A broker who is sitting
  on 60 entries today will clear shipments slower than one with 6.
- **Severity:** **nice-to-have**. The lookup table absorbs the
  *expected* dwell, but per-day variance from broker workload is
  invisible.
- **Why missed:** §Ground-arc parameters explicitly says the lookup is
  tenant-calibrated from historical broker performance — it's a static
  per-broker mean, not a daily-load-adjusted value.
- **Fix size:** medium — broker-capacity input becomes a daily parameter
  on the customs-dwell arc.

### 4.4 PGA hold pre-filing as a routing differentiator
- **One-liner:** FDA / USDA / EPA PGAs admit advance filings (FDA
  Prior Notice, USDA Prior Notification, EPA TSCA pre-import). Cargo
  with PGA pre-filed has a much lower exam-hold probability and shorter
  hold duration than cargo where PGA is filed on entry. The model has
  no `pga_pre_filed_k` attribute.
- **Severity:** **serious** for shippers in food / pharma / chemicals
  (a huge fraction of TPE / HKG / SHA / PVG outbound by HAWB count).
- **Why missed:** spec acknowledges PGA dwell is "destination-country
  and commodity-class dependent" but does not distinguish pre-filed vs
  filed-on-entry within the same commodity.
- **Fix size:** small — `pga_pre_filed_k` boolean as input; lookup table
  keys on `(destination_country, ctype_k, commodity_class, pga_pre_filed_k)`.
  Sources: FDA Prior Notice
  ([fda.gov/.../prior-notice-imported-foods](https://www.fda.gov/food/importing-food-products-united-states/prior-notice-imported-foods)).

---

## 5. ULD operations edge cases

### 5.1 ULD demurrage on returned ULDs (forwarder-side liability)
- **One-liner:** When the forwarder pulls a ULD from the carrier (BSA
  ULD-pool position), they have a contracted return window (typically
  48–96 hours after flight ETA at destination) and pay demurrage past
  that. ULD demurrage is ~$50–$200/ULD/day and a real OpEx line for
  CFS operators. Model has no ULD-return arc / no return-window state.
- **Severity:** **nice-to-have** for individual routing, **serious** in
  aggregate (it shapes which ULD types are economically attractive on
  return-flow-deficit lanes).
- **Why missed:** ULDs are modeled as a per-MAWB claim (`η_{a,g,u}`),
  not as a borrowed asset with a return obligation.
- **Fix size:** medium — needs a per-ULD-pull cost adjustment based on
  expected return-leg dwell; out of MVP scope but should be on the
  deferred list.
  Source: ULD CARE references repair/loss cost ~$400M/y industry-wide
  ([uldcare.com/uld-explained-book/uld-basics/](https://www.uldcare.com/uld-explained-book/uld-basics/)).

### 5.2 Weight-and-balance limits (CG envelope) on the ULD claim
- **One-liner:** §ULD spec carries `MGW_u`, `tare_u`, `W_u` per ULD type,
  but the per-flight weight-and-balance envelope (the aircraft's CG
  constraint) means certain combinations of position × weight are
  forbidden even when each ULD position is within its own limit.
- **Severity:** **nice-to-have**. The carrier handles the W&B at
  acceptance; from forwarder side this is silent. But it routinely
  produces "your booking is confirmed at acceptance but the rear AKE
  was downgraded to forward, so your cargo slipped to next flight."
- **Why missed:** W&B is a per-flight aircraft-level concern; not a
  forwarder-side observable. Acknowledged as out-of-scope (carrier
  responsibility) but worth surfacing as a documented assumption.
- **Fix size:** out-of-scope; document only.

### 5.3 ULD shape-aware pre-filter (LD3 contour vs PMC main-deck)
- **One-liner:** Pre-filter step 8 checks `w_k > max_u W_u` and
  `v_k > max_u V_u`. Real ULD contour rejection happens when the
  combination of `(height, footprint)` of the cargo exceeds the ULD
  contour profile — and the contour data lives in the catalog as a
  string code (per §ULD spec), not as a 3D envelope.
- **Severity:** **nice-to-have** at MVP (pre-filter catches the
  egregious cases); **serious** for oversized-cargo segments.
- **Why missed:** §ULD spec explicitly defers full 3D bin-packing
  (item 11 on deferred list). What's new here: contour-fit isn't 3D
  bin-packing — it's a 2D envelope vs. cargo silhouette. Cheaper than
  full packing but not in MVP.
- **Fix size:** medium — contour-envelope check as a pre-filter
  refinement, separate from the bin-packing deferral.

### 5.4 ULD interline-pool ownership disputes (Star vs SkyTeam vs oneworld pool churn)
- **One-liner:** §Through-ULD policy enumerates the three alliance ULD
  pools as if they are stable sets. They are not. Bilateral ULD-pool
  agreements come and go (SQ left Star Alliance Cargo pool but not
  passenger Star Alliance; CV operates a unilateral pool on some lanes).
- **Severity:** **nice-to-have**. The set Π enumerates pool memberships;
  it just needs version control.
- **Why missed:** spec assumes Π is a tenant-managed static input.
- **Fix size:** small — version Π with effective dates; surface
  membership-change warnings to operators when a previously-used pool
  membership has expired.

---

## 6. Exception / disruption taxonomy gaps

### 6.1 Offload propagation to downstream MAWBs
- **One-liner:** When a HAWB is offloaded mid-journey (cargo too heavy
  for the day's weight-and-balance; LD3 swapped to AKE and one piece
  doesn't fit), the offload propagates to *all* downstream MAWBs that
  expected the offloaded cargo. The supply-side lock-invalidation
  taxonomy (§Locked commitments) covers flight cancellation, equipment
  swap, allocation pull, cutoff shift — but not partial-cargo offload.
- **Severity:** **serious**. Partial offload of a multi-HAWB MAWB
  cascades: the downstream MAWB that was planned at 1200 kg now has
  one of its constituent HAWBs missing, which changes the chargeable
  weight, which can flip the next-break-down rule, which can trigger
  cost recomputation that the locked-commitment preprocessing does not
  handle.
- **Why missed:** lock invalidation is leg-level, not HAWB-membership-
  level within a MAWB.
- **Fix size:** medium — extend supply-event taxonomy with
  `partial_offload(MAWB, HAWB-set)`; orchestrator recomputes downstream
  HAWBs the same way as a flight cancellation.

### 6.2 Alliance-rebooking through partner carrier (not just direct re-book)
- **One-liner:** When carrier X cancels, the rescue path is often "rebook
  on partner Y within X's alliance under X's AWB number." This is a
  routing option (alliance fallback) that the model never explores
  because the alliance structure is only used at graph-build to emit
  through-MAWB arcs, not at re-solve time.
- **Severity:** **serious** at re-solve time. The rescue workflow
  (§Locked commitments §Supply-side lock invalidation) just removes the
  affected arc and re-runs — it does not actively seek alliance partner
  alternatives because the graph generator already encoded one such
  alternative (or didn't) at the first build.
- **Why missed:** alliance structure is "absorbed at graph build" — but
  this means re-solve loses access to the alliance-as-rescue-channel
  reasoning that an operator does.
- **Fix size:** medium — graph generator emits alliance-partner-fallback
  arcs eagerly for high-value carriers; the orchestrator uses them on
  re-solve.

### 6.3 AOG (Aircraft on Ground) diversion as a flight-equivalent rescue
- **One-liner:** AOG events (engine swap, mechanical) typically cause
  next-flight roll, but on freighters they can produce a multi-day
  hole until a replacement aircraft can be repositioned. The supply
  event set lists `flight_cancellation` and `equipment_swap` but not
  `aircraft_on_ground_extended` — a different operational reality with
  different rescue tactics (charter, co-load to partner) and a
  different time horizon.
- **Severity:** **nice-to-have**. The orchestrator can model AOG as a
  flight cancellation; the difference is the cost-of-rescue, which the
  model never sees.
- **Why missed:** lock-invalidation taxonomy is operationally compressed.
- **Fix size:** small — taxonomy expansion only; downstream behavior
  follows the cancellation pattern.

### 6.4 Cargo damage post-CFS receipt (split, recovery, claim path)
- **One-liner:** Cargo arriving damaged at the CFS (forklift damage,
  rain damage, drop) is a real event class. The model doesn't represent
  it because it represents the cargo as a single HAWB with weight and
  volume — once damaged, the rest of the HAWB needs to continue while
  the damaged portion enters a claims path.
- **Severity:** **nice-to-have**. Real-world handling is to re-issue a
  split HAWB and re-route the salvageable portion; the model's lack of
  split-HAWB support (split is on the deferred list) covers this case
  obliquely.
- **Why missed:** intersection of `split shipment` (deferred) and
  `partial damage` (not explicitly listed).
- **Fix size:** subsumed by the deferred split-shipment item; no separate
  action.

### 6.5 Network-level event (port strike, weather closure) as a portfolio-wide signal
- **One-liner:** Per `forwarder-operations-analysis/04-exceptions-replanning.md`
  §4.7, events like port strikes or Red Sea closure affect hundreds of
  shipments simultaneously. The model's re-solve is per-HAWB-batch
  (`K` is one solve's batch); there is no concept of "this event
  invalidates a set of arcs across the catalog for all tenants/HAWBs."
- **Severity:** **serious** in event periods, **nice-to-have** in steady
  state. The orchestrator can in principle invalidate arcs in the
  catalog and re-run, but the gap is the absence of a structured
  network-event input.
- **Why missed:** scope decision — the MILP is per-batch; portfolio
  events are an orchestration concern.
- **Fix size:** small — orchestrator hook; not a MILP change.

---

## 7. Replan-trigger gaps (events that should trigger a re-solve)

### 7.1 Cargo-readiness slip past `t_k^{rdy,late}`
- **One-liner:** When the shipper signals "actually cargo will be ready
  Thursday not Wednesday," the model's `[t_k^{rdy,early}, t_k^{rdy,late}]`
  window assumption is invalidated. The locked-commitment taxonomy
  covers carrier-side events; shipper-side readiness slips are not
  listed as a replan trigger.
- **Severity:** **serious**. Shipper-readiness slip is the most common
  pre-departure exception per `forwarder-operations-analysis/02-network-ops.md`
  §3.1 A6 ("cutoff-miss re-tender") and the day-in-the-life narrative.
- **Why missed:** §Locked commitments §Supply-side lock invalidation
  enumerates only supply (carrier) events; demand-side (shipper) events
  aren't in the taxonomy.
- **Fix size:** small — add `cargo_readiness_slip` event to the
  orchestrator event taxonomy; on receipt, update `t_k^{rdy,early}` and
  re-enqueue.

### 7.2 Screening-certification lapse / re-screen requirement
- **One-liner:** §Screening dropped screening as an arc-eligibility
  predicate (modeled as cost). But when cargo's screening status lapses
  in transit (TSA CCSP chain-of-custody break at a tender between two
  IACs), the cargo must be re-screened — and re-screening may not be
  available at the next station. This event should trigger a re-solve.
- **Severity:** **nice-to-have**. The cost-not-eligibility decision is
  defensible at MVP. The trigger gap is for the rare but high-impact
  case.
- **Why missed:** the §Screening simplification rules out the entire
  predicate, so the system has no mechanism to surface "re-screen
  required at intermediate station."
- **Fix size:** medium — re-screening events feed back into a
  carrier-side dwell adjustment on the next available CFS arc.
  Source: TSA CCSP
  ([tsa.gov/.../cargo-screening-program](https://www.tsa.gov/for-industry/cargo-screening-program)).

### 7.3 Customer-side ETA renegotiation
- **One-liner:** Customer informs forwarder "this cargo can now arrive
  by D+5 instead of D+3." This relaxes `T_k^abs` and possibly
  `T_k^SLA`, which should trigger a re-solve in the *favorable*
  direction — find a cheaper route now that the deadline is looser.
- **Severity:** **nice-to-have**. Operators today reroute manually only
  when they remember to.
- **Why missed:** the model is built to react to constraint
  *tightening*, not to constraint *loosening*. No structured event.
- **Fix size:** small — orchestrator event for `deadline_relaxed`;
  re-enqueue with new `T_k^abs`.

### 7.4 Spot rate snapshot expiry mid-execution
- **One-liner:** `DYNAMIC_SPOT` offers carry `expiry_a`; the validator
  is at graph-build time. But the spot rate the model selected at
  T-72h may expire before the operator books at T-12h, leaving the
  routing decision unbacked. The model has no notification mechanism.
- **Severity:** **serious** on dynamic-spot-heavy lanes (TPE, HKG
  outbound during peak). Operators bind plan to expired rates and only
  discover at booking time.
- **Why missed:** spot-rate validity is data-layer (per
  §Procurement types §Dynamic spot), not part of any constraint or
  trigger.
- **Fix size:** small — orchestrator monitors spot expiries on
  selected arcs; auto-trigger re-solve when expiry approaches.

---

## 8. Pricing-scenario gaps

### 8.1 Spot-vs-snapshot drift on platform-sourced rates
- **One-liner:** Rates pulled from cargo.one / WebCargo / CargoAi
  refresh hourly to daily depending on carrier. The model uses a
  per-run snapshot (per §Pricing layer storage), but doesn't detect or
  surface when the live spot has moved materially from the snapshot
  that drove the optimization.
- **Severity:** **serious**. A 5–10% drift between snapshot and live
  spot is common on volatile lanes; the operator quotes against the
  snapshot value and discovers the gap at booking confirmation.
- **Why missed:** the snapshot mechanism is correct for solve
  reproducibility but doesn't include drift-detection at commit time.
- **Fix size:** small — orchestrator-side drift check between snapshot
  rate and live aggregator rate at booking time; flag if >X%.

### 8.2 Volume-discount kickers and rebates (negative-coefficient terms)
- **One-liner:** Some shipper master contracts include
  `volume_milestone` rebates ("at 500 MT/quarter on the lane, rebate
  $0.05/kg retroactive"). These are negative-coefficient terms in
  `CW_{a,g}` which the model's monotonicity invariant
  (§CW density mixing) explicitly says it cannot represent.
- **Severity:** **serious** for large-volume shippers; **nice-to-have**
  otherwise. The model would systematically under-route through the
  rebated lane during the early period of a quarter (when threshold
  is not yet hit) and over-route at quarter-end (when threshold-cross
  becomes possible).
- **Why missed:** acknowledged that monotonicity breaks under negative
  coefficients (§ Why the inequality relaxation is exact); the
  consequence is that the model can't represent rebates.
- **Fix size:** medium — add a parallel "rebate-aware" mode that
  switches to PWL for `CW_{a,g}` when a rebate applies; for MVP this
  is a manual operator override.

### 8.3 NAC (Negotiated Air Cargo) commitment pull-forward at contract-end
- **One-liner:** NAC contracts often have annual minimum commitments.
  Late in the contract year, forwarders pull cargo onto under-utilized
  NAC lanes to hit commitment. The model's allowance mechanism
  (`A_c`) covers per-period BSA but not NAC year-end pull-forward.
- **Severity:** **nice-to-have**. NAC at MVP is treated as a
  `min_flat_breaks` rate; commitment dynamics are out of scope.
- **Why missed:** NAC commitment structure is not modeled separately
  from rate.
- **Fix size:** medium — NAC-specific allowance parameter or a
  generalization of `A_c` to cover non-BSA contracts.

---

## 9. Top 10 ranked gaps (across categories)

Ranked by combined severity × operator-impact × frequency. Higher = more
blocking.

| Rank | Gap | Category | Severity | Fix |
|------|-----|----------|----------|-----|
| 1 | 3.4 Per-flight carrier daily allowance variance (BSA daily cut) | Carrier policy | Serious | Medium |
| 2 | 3.1 Time-windowed / seasonal carrier rules | Carrier policy | Serious | Medium |
| 3 | 7.1 Cargo-readiness slip past `t_k^{rdy,late}` as a replan trigger | Replan trigger | Serious | Small |
| 4 | 2.5 Allocation pull-forward (cross-period BSA dynamics) | Rate structure | Serious | Medium |
| 5 | 4.1 In-bond / T&E movement at US hubs | Customs | Serious | Medium |
| 6 | 1.3 Lithium-spec field completeness vs. default-deny pre-filter | Data field | Serious | Medium |
| 7 | 6.1 Partial-offload propagation through downstream MAWBs | Exception taxonomy | Serious | Medium |
| 8 | 2.3 MNC hard-contract carrier mandates (vs soft preference) | Rate structure | Serious | Small |
| 9 | 2.1 Fuel surcharge index-linked pass-through | Rate structure | Serious | Medium |
| 10 | 7.4 Spot rate snapshot expiry mid-execution | Replan trigger | Serious | Small |

**Honorable mentions (just below the cut):** 6.2 alliance-partner
rebooking; 4.4 PGA pre-filing; 4.2 GB↔EU post-Brexit dwell; 2.4
charter-fragment buys; 1.1 declared-value completion rate.

**Severity distribution across all 30+ findings:**
- 0 blocking (the v3 model is structurally usable)
- ~18 serious (operator will revert to manual ≥20% of the time in
  affected scenarios)
- ~13 nice-to-have (improves output quality; no daily reversion)

**Pattern observation.** The serious gaps cluster in three areas:
(a) **time-and-state**: the model is per-batch and per-snapshot, which
loses cross-period BSA dynamics, daily capacity variance, mid-execution
rate drift, and shipper-side state changes (gaps 1, 3, 4, 5, 10).
(b) **rule-engine richness**: the 5-layer carrier cascade is well-formed
but doesn't support time windows, alliance-level allows, or
hard-contract mandates (gaps 2, 8). (c) **non-monotone cost terms**:
the monotonicity invariant rules out rebates and any negative-
coefficient term, including all volume-discount mechanisms that exist in
real contracts (gap §8.2). These three thematic clusters are the
candidates for the next round of LaTeX rewriting.
