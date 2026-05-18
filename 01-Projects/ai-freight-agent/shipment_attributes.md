# Shipment Attributes

Source of truth for the per-shipment data fields that flow through the system from intake to final delivery. Captures **static attributes** (set at intake, immutable through lifecycle) and **dynamic attributes** (created or updated as the shipment moves), along with the **milestone events** that drive dynamic-attribute updates.

This document is the contract between the **ingestion layer**, the **routing optimizer** (consumes static + current dynamic state), the **execution / lock tracking system** (writes dynamic state), and the **UI / reporting** layers. Cross-references to the air-freight LaTeX model and `data_model.md` are noted per attribute.

---

## 1. Categories

**Static attributes** — set at the moment a shipment is created (push API, manual entry, or demand generator) and immutable for the lifetime of the shipment. Includes the commercial commitment (service product, sell rate), the physical descriptors (weight, volume, dimensions), and the regulatory flags (cargo type, screening status, lithium spec). The routing optimizer reads these as constants.

**Dynamic attributes** — created or updated by milestone events as the shipment progresses. Capture lifecycle state, lock state, observed times, customs state, booking status, realized cost, and current location. The optimizer reads these to know what is fixed and what is still open; the execution system writes them in response to real-world events.

**Hybrid case**: a few fields (e.g., `service_product_id`) can in principle be changed mid-lifecycle (shipper amends contract); current MVP treats these as static and any change creates a new shipment record. Documented per-field below.

---

## 2. Static Attribute Catalog

### 2.1 Identity and tenancy

| Attribute | Type | Required at intake | Notes |
|---|---|---|---|
| `shipment_id` | UUID | yes (system-generated) | Internal identifier |
| `tenant_id` | UUID | yes | Multi-tenancy; RLS enforced |
| `hawb_number` | string | yes (forwarder-assigned at intake) | Forwarder's House AWB number |
| `external_id` | string | optional | Push-API client's reference; idempotency key |
| `ingestion_source` | enum | yes | `push_api` / `manual` / `demand_generator` |
| `created_at` | timestamp (UTC) | yes (system) | Per `air.tex` §2 time-zone convention |
| `created_by` | UUID | yes | Operator or API key origin |

### 2.2 Parties

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `shipper_id` | UUID | yes | Foreign key to `shippers` table |
| `consignee_id` | UUID | yes | Foreign key to `consignees` |
| `notify_party_id` | UUID | optional | Often = consignee |

### 2.3 Origin and destination

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `origin_door_address` | structured address | yes | Pickup location |
| `origin_country` | ISO-3166 alpha-2 | yes | Drives regulatory flags |
| `origin_iata` | 3-letter IATA airport code | yes (derived) | Resolved by ingestion |
| `origin_cfs_preference` | CFS node id | optional | Operator can pin; optimizer treats as soft constraint when present |
| `destination_door_address` | structured address | yes | |
| `destination_country` | ISO-3166 alpha-2 | yes | Drives customs / screening / release type defaults |
| `destination_iata` | 3-letter IATA airport code | yes (derived) | |
| `destination_cfs_preference` | CFS node id | optional | |

### 2.4 Time constraints

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `cargo_ready_early_utc` | timestamp (UTC) | yes | Earliest pickup; `t_k^{rdy,early}` in air model |
| `cargo_ready_late_utc` | timestamp (UTC) | yes | Latest pickup window close; `t_k^{rdy,late}` |
| `deadline_utc` | timestamp (UTC) | yes | Hard delivery deadline; `T_k^{dead}` in air model |

### 2.5 Service product and commercial

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `service_product_id` | reference to `service_products` catalog | yes | Resolves bundle: mode_allow, carrier_allow, T_SLA, etc. (data_model.md §4, air.tex §6.14) |
| `budget_cap_usd` | numeric | optional (default ∞) | `B_k` in air model (P.18 budget cap) |
| `sell_rate_usd_per_kg` | numeric | optional | Quote-engine input; NOT used by routing MILP (cost minimization only); stored for margin reporting |
| `sell_rate_basis` | enum | optional | `per_kg` / `per_shipment_flat` / `per_cbm` |
| `currency` | ISO 4217 | yes | Default USD; FX table in `data_model.md` §8 |

### 2.6 Commodity and physical

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `commodity_class` | enum | yes | `GEN` / `DGR` / `PER` / `VAL` / `AVI` / `HUM` |
| `commodity_codes_hs` | array of HS codes | yes | Multi-HS allowed for mixed-product FCL (per existing P1 spec) |
| `commodity_description` | text | yes | Required for AWB filing |
| `weight_kg` | numeric | yes | Actual gross weight |
| `volume_m3` | numeric | yes | |
| `chargeable_weight_kg` | numeric (derived) | yes (computed) | `cw_k = max(w_k, v_k × 167)` |
| `piece_count` | integer | yes | |
| `piece_dimensions` | list of (L, W, H) cm | yes | Max-single-piece used for ULD fit check P.17 |
| `stackable` | boolean | yes | Defaults true; false for fragile/floor-load-only |

### 2.7 Cargo-type and regulatory flags

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `temperature_control` | enum | optional | `ambient` / `cool_chain_2_8c` / `frozen` / `pharma_active` |
| `screening_status` | enum | required for PAX_BELLY routings | `known_shipper` / `ccsf_screened` / `ra3_kc3_chain` / `none` (per air.tex §6 screening section) |
| `screening_chain_id` | UUID | optional | Reference to certifying entity (CCSF / RA3 / KC3) for audit |
| `release_type` | enum | yes (destination dwell input) | `express_release` / `original_awb_surrender` / `telex_equivalent` / `freight_collect_release` |

### 2.8 Lithium battery (when applicable)

Applies only when `commodity_class = DGR` and lithium content present. Per air.tex §6.11.

| Attribute | Type | Required if lithium | Notes |
|---|---|---|---|
| `lithium_un_number` | enum | yes | `UN3480` / `UN3481` / `UN3090` / `UN3091` |
| `lithium_pi_code` | enum | yes | `PI965` ... `PI970` |
| `lithium_section` | enum | yes | `IA` / `IB` / `II` |
| `lithium_watt_hours` | numeric | yes | Per cell or per battery |
| `lithium_soc_compliant` | boolean | yes | State-of-charge ≤ 30% per IATA DGR |
| `lithium_ddr` | boolean | yes | Damaged / defective / recalled exclusion |

### 2.9 Special handling

| Attribute | Type | Notes |
|---|---|---|
| `valuables_handling` | boolean | Triggers separate handling chain, escort, vaulted CFS |
| `live_animal_species` | string | AVI cargo; per IATA Live Animal Regulations |
| `oversize_oog` | boolean | Out-of-gauge; aircraft and ULD compat narrows |
| `customs_broker_id` | UUID | optional | Destination customs broker for clearance |

---

## 3. Dynamic Attribute Catalog

### 3.1 Lifecycle state

| Attribute | Type | Source | Notes |
|---|---|---|---|
| `lifecycle_state` | enum | system | 7-state DAG: `unrouted` → `soft_planned` → `firm_deadline` → `firm_planned` → `in_transit` → `destination_planning` → `delivered`. Per `data_model.md` §3.5 and air.tex §6.12 (locks) |
| `lifecycle_state_entered_at` | timestamp (UTC) | system | When did the shipment enter the current state |
| `is_escalated` | boolean | system / operator | Flagged for human exception queue (Tier 3 routing) |
| `is_infeasible` | boolean | system | No feasible route found in latest solve |
| `is_cancelled` | boolean | operator | Cancelled by ops |

### 3.2 Routing decision (output of optimizer; current plan)

| Attribute | Type | Source | Notes |
|---|---|---|---|
| `current_route` | ordered list of arcs | routing optimizer | The chosen path through G(N_k, A_k); active after `soft_planned` |
| `current_uld_assignments` | per-flight (u, count, contract) | routing optimizer | What ULDs on what flights; per (c, u, f) |
| `current_spot_bookings` | per-flight booking record | routing optimizer | When using spot rates |
| `routing_run_id` | UUID | optimizer | The run that produced the current plan; binds to spot/policy snapshots for replay |
| `plan_cost_usd` | numeric | optimizer | Per the plan; `cost(solution)` |
| `plan_margin_usd` | numeric (derived) | quote engine | `sell - plan_cost`; for reporting only |

### 3.3 Lock state (per air.tex §6.12)

| Attribute | Type | Source | Notes |
|---|---|---|---|
| `locked_arcs_on` | set of arcs | execution layer | `A_k^{loc}`; committed-to-use |
| `locked_arcs_off` | set of arcs | execution layer | `A_k^{loc-off}`; committed-not-to-use (e.g., flights passed over) |
| `locked_uld_assignments` | per-flight committed `y` | execution layer | Sticks once ULD built or AWB filed |
| `locked_spot_bookings` | per-flight committed `b` | execution layer | Sticks once spot booking confirmed |
| `lock_horizon_utc` | timestamp | rolling-horizon controller | Horizon through which current locks are enforced |

### 3.4 Observed times (from milestone events)

For each node `n` in the realized route, `observed_node_times[n]` records the actual arrival/departure timestamp (UTC) from milestone events. Populates `\bar{t}_k(n)` in the model.

| Field on the record | Type | Notes |
|---|---|---|
| `node_id` | reference | The graph node (origin door, CFS, airport, etc.) |
| `event_type` | enum | See §4 below |
| `observed_at_utc` | timestamp | Actual time |
| `source_event_id` | UUID | Which milestone event produced this record |
| `variance_minutes` | numeric (derived) | Observed − planned |

### 3.5 Customs state

| Attribute | Type | Notes |
|---|---|---|
| `customs_state` | enum | `not_yet_required` / `filing_pending` / `filed` / `held` / `released` |
| `customs_filed_at_utc` | timestamp | When AMS / ENS / ACI / etc. was filed |
| `customs_released_at_utc` | timestamp | Drives destination dwell calculation |
| `customs_hold_reason` | text | When `held`; for ops review |

### 3.6 Booking status (per flight)

| Attribute | Type | Notes |
|---|---|---|
| `booking_status_per_flight[flight_id]` | enum | `KK` (confirmed) / `RQ` (requested) / `NS` (no space) / `cancelled` |
| `booking_rejection_reason` | text | When `NS`; feeds re-optimization trigger |
| `booking_confirmed_at_utc` | timestamp | When carrier confirmed `KK` |

### 3.7 Realized cost (post-invoice)

| Attribute | Type | Notes |
|---|---|---|
| `realized_cost_usd` | numeric | Sum of actual invoiced charges |
| `realized_cost_variance_usd` | numeric (derived) | `realized - plan_cost` |
| `invoice_reconciled_at_utc` | timestamp | When accounting closed the shipment |

### 3.8 Current location and disruption

| Attribute | Type | Notes |
|---|---|---|
| `current_location_node` | node reference | Where the cargo physically is right now |
| `last_milestone_event_id` | UUID | Most recent event; latest_event_at_utc |
| `reroute_count` | integer | Number of re-plans this shipment has gone through |
| `disruption_flags` | array of enums | `flight_cancelled` / `equipment_swap` / `customs_hold` / `cargo_damaged` / `cutoff_miss` |

---

## 4. Milestone Event Taxonomy

Milestone events are the system of record for dynamic-attribute updates. Each event is immutable, ordered by `observed_at_utc`, and carries the shipment_id + node_id + event_type + payload.

### 4.1 Origin-side events

| Event | Updates |
|---|---|
| `cargo_ready` | Sets `observed_node_times[origin_door]` = cargo-ready time |
| `truck_dispatched` | Pickup arc activated |
| `gate_out` | Cargo departed origin door |
| `cargo_at_origin_cfs` | Arrives at origin CFS; `observed_node_times[origin_cfs]` |
| `screening_completed` | If applicable; records `screening_chain_id`, screen result |
| `mawb_filed` | Carrier-side; AWB on the cargo |
| `uld_built` | At CFS; ULD type and contents recorded |
| `cargo_at_airline_terminal` | Tendered to airline cargo terminal; passes `CGC_f` check |

### 4.2 Air-leg events

| Event | Updates |
|---|---|
| `cargo_loaded` | On aircraft; lock first air arc |
| `flight_departed` | `observed_node_times[origin_airport]` = ETD-actual |
| `flight_arrived` | `observed_node_times[destination_airport]` = ETA-actual |
| `transit_uld_handed_over` | At hub; through-ULD or transferred under ULD interchange |
| `uld_broken_down_at_hub` | Re-ULDing performed |
| `uld_rebuilt_at_hub` | Onward ULD ready |

### 4.3 Destination-side events

| Event | Updates |
|---|---|
| `cargo_at_destination_terminal` | Arrives airline terminal at destination |
| `customs_filed` | Sets `customs_state = filed`, `customs_filed_at_utc` |
| `customs_held` | Sets `customs_state = held`; surfaces in exception queue |
| `customs_released` | Sets `customs_state = released`, `customs_released_at_utc` |
| `cargo_at_destination_cfs` | At destination CFS |
| `awb_release_obtained` | Per `release_type`: express, original surrender, telex equivalent |
| `cargo_broken_down_dest_cfs` | Destination CFS breakdown complete |
| `final_delivery_dispatched` | Final delivery leg active |
| `delivered` | Final state; sets `lifecycle_state = delivered` |

### 4.4 Disruption / exception events

| Event | Updates |
|---|---|
| `flight_cancelled` | Supply-side lock invalidation (per air.tex §6.12 supply locks) |
| `equipment_swap` | `ac(f)` change; may invalidate ULD-position commitments |
| `cargo_damaged` | Triggers escalation; potential rerouting |
| `cutoff_missed` | Cargo rolled; triggers next-flight re-plan |
| `re_plan_triggered` | Rolling-horizon controller queued a re-solve for this shipment |

---

## 5. Source-of-Truth Mapping

Where each class of attribute is canonically stored.

| Attribute class | System of record | Notes |
|---|---|---|
| Identity, parties, origin/destination, time constraints, service product, commodity, regulatory flags | `shipments` table (Postgres) | Set at intake; immutable thereafter (changes create new shipment) |
| `service_product_id` resolution | `service_products` table (data_model.md §4 via generic policy framework) | Per-tenant catalog |
| Sell rate | `shipments.sell_rate_*` columns | Read by quote engine, NOT by routing MILP |
| `lifecycle_state` and timestamps | `shipments.lifecycle_state` column + `lifecycle_state_transitions` audit table | Updated by execution layer in response to milestone events |
| Current routing plan | `routing_runs` table + `routing_run_outputs` per-shipment | One row per (shipment, run); historical record of plans |
| Locked arcs / locked assignments | Derived from `bookings` + `milestone_events` at solve time | Not stored explicitly; resolved per air.tex §6.12 |
| Observed node times | `milestone_events` table | Append-only event log |
| Customs state | `shipments.customs_*` columns | Updated by customs broker events |
| Booking status per flight | `bookings` table | One row per (shipment, flight) booking attempt |
| Realized cost | `invoices` + reconciliation; rolls up to `shipments.realized_cost_usd` after close | Lag of days-to-weeks after delivery |
| Disruption flags | `shipments.disruption_flags` + `milestone_events` | Both queryable |

---

## 6. Attribute Lifecycle by Phase

When each dynamic attribute becomes meaningful:

| Lifecycle phase | Dynamic attributes active |
|---|---|
| `unrouted` | none yet — only static attributes |
| `soft_planned` | `current_route`, `current_uld_assignments`, `plan_cost_usd`, `routing_run_id` |
| `firm_deadline` | dry-run window timestamps; no new attributes |
| `firm_planned` | `locked_arcs_on`, `locked_uld_assignments`, `locked_spot_bookings`, `booking_status_per_flight` (initial KK confirmations) |
| `in_transit` | `observed_node_times` populated leg-by-leg; `current_location_node` advances; `customs_state` becomes active when destination customs filed |
| `destination_planning` | All air arcs in `locked_arcs_on`; destination ground arcs still open; `customs_state` reaches `released`; destination dwell time accrued (per `release_type`) |
| `delivered` | `realized_cost_usd` (after invoice reconciliation); shipment closed |

---

## 7. Out of Scope (deferred to P1 or later)

- **Amendable static attributes** — current MVP treats static fields as immutable; amendments create a new shipment. P1: explicit amendment semantics with audit chain.
- **Multi-piece per-piece tracking** — current schema captures aggregate piece_count and max-piece dimensions. P1: per-piece records with individual piece IDs (relevant for partial deliveries, claims).
- **Shipper-level forecast attributes** — e.g., "shipper typically books 50 shipments/month on TPE-JFK". Forecasting is a Phase 5 concern, separate from per-shipment attributes.
- **ML-derived attributes** — e.g., predicted disruption risk score per shipment, predicted customs hold probability. Phase 5 constraint learning.
