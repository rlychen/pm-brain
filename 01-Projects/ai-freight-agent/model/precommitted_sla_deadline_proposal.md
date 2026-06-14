# Methodology change proposal — Pre-committed tier×lane SLA deadlines (supersedes D-F6)

**Status:** ✅ APPROVED & IMPLEMENTED (S37). Decisions §9: (1) adopt — yes; (2) base_transit hardcoded
`[CAL]` now — yes; (3) keep `T^abs = cutoff + buffer` — yes. Folded into `flexibility_model.md` §1/§2.1
(D-F6 → **D-F6 v2**) + `flexibility.committed_deadline` + the arrival generator. 255 tests green.

**Author note:** raised by the user (S37). The trigger was a real defect: under the build-time
geographic graph (candidates resolved at build, not generation), the per-shipment `A_k^min`
derivation produced `Δ_k > T^abs` for some shipments — a soft deadline later than the hard
deadline. The deeper objection: the current rule is not how forwarders actually set deadlines.

---

## 1. What's wrong with the current rule (D-F6)

Today (`flexibility_model.md` §1, §2.1):

```
A_k^min        = earliest feasible arrival over shipment k's OWN routes (predicates 1–8)
min_transit_k  = A_k^min − ready_k
Δ_k            = ready_k + min_transit_k + sla_offset_h(tier)     # = A_k^min + sla_offset(tier)
```

The soft deadline (the *promise*) is defined relative to **each shipment's own best achievable
route**. Two problems:

1. **It's circular / not how SLAs work.** A forwarder does not receive a shipment, compute its
   best possible routing, and *then* decide what it promised. The promise is a **pre-committed
   per-(tier, lane) ETA**, quoted from the network's *capability*, before any routing decision.
   E.g. TPE→LAX: ~96 / 120 / 144h for express / standard / deferred — published, then sold against.

2. **It couples Δ_k to the graph, which broke under build-time geo selection.** Because Δ_k was
   derived from `A_k^min`, and `A_k^min` depends on which airports the shipment may use, the
   deadline became entangled with the candidate-resolution stage. Computing `A_k^min` over a single
   nominal airport (a poor route) inflated `Δ_k` past `T^abs`:

   ```
   nominal-airport A_k^min = 168h → Δ_k = 168 + 100 = 268h  vs  T^abs = 234h   → 268 > 234, invalid
   ```

   The deadline should not depend on the routing graph at all.

---

## 2. Proposed rule — pre-committed tier×lane SLA

```
Δ_k (soft / promise) = ready_k + sla_transit_h(tier, lane_k)
T^abs (hard / drop-dead) = cutoff_k + backstop_buffer_h            # UNCHANGED (planning horizon)
```

where `lane_k = (nominal_origin_airport, nominal_dest_airport)` (the shipment's gateway pair) and
`sla_transit_h(tier, lane)` is an **exogenous, pre-committed** promised end-to-end transit, set from
the lane's capability — **not** from this shipment's own `A_k^min`.

**Parsimonious form (recommended), equivalent to a full tier×lane table:**

```
sla_transit_h(tier, lane) = base_transit_h(lane) + sla_offset_h(tier)
```

- `base_transit_h(lane)` — the lane's pre-committed achievable transit (capability estimate).
- `sla_offset_h(tier)` — the existing per-tier buffer (express tight → deferred generous), keeping
  the tier ordering `slack_EXPRESS < slack_STANDARD < slack_DEFERRED`.

This is a **one-line change to D-F6**: replace the per-shipment `min_transit_k` with the per-lane,
pre-committed `base_transit_h(lane)`. The tier-offset machinery (`sla_offset_h(tier)`) is unchanged.

Worked: `base_transit(TPE→LAX) = 96`, offsets `{EXPRESS:0, STANDARD:24, DEFERRED:48}` →
SLAs `96 / 120 / 144h` — matching the real-world example.

---

## 3. What stays, what's removed

**Removed**
- The per-shipment `A_k^min` computation **in the deadline derivation** (arrival generator pass-2):
  no graph build at generation for deadlines, no FreightNet dependency in pass-2, and the geo
  entanglement (and the `Δ_k > T^abs` defect) simply cannot occur. Generation becomes a table lookup.

**Unchanged**
- `T^abs = cutoff + backstop_buffer_h` (the planning horizon).
- `sla_offset_h(tier)`, tier mix (D-F5), `z_tier`, `w_sp`, predicate 9, the C.10 tardiness PWL —
  all read `Δ_k`; none care how it was derived.
- `A_k^min` still exists for `slack_k = Δ_k − A_k^min` and `flex_k`/`cw_flex` — but that is the
  **flex stage** (2b reliability / F5), computed when flex is wired, not in the generator deadline step.

---

## 4. The invariant holds by construction (no `max()` hack)

```
T^abs − Δ_k = (cutoff − ready_k) + backstop_buffer_h − sla_transit_h(tier, lane)
```

With realistic SLAs (≤ ~144h) and `backstop_buffer_h = 168h`, and `cutoff ≥ ready`, this is
`≥ 168 − 144 = 24h > 0` always. The defect that motivated this change cannot recur. (Rejected
alternative: `T^abs = max(default, Δ_k)` — it satisfies the invariant by setting `span = T^abs − Δ_k
= 0` for the very shipments whose Δ_k is high, which makes them **un-penalizable for lateness** and
gives their fallback a free pass. It destroys the tardiness semantics; not adopted.)

---

## 5. Why this is the right phenomenon

The SLA is a **fixed commitment**; congestion decides whether it's kept. Under tight supply
(high κ) / heavy arrivals, the optimizer cannot route every shipment to its committed `Δ_k` →
tardiness, then fallback at `T^abs`. **Replanning (M1) keeps more commitments than single-pass
(M0) — that gap is the measured value.** The current rule blunts this: defining the promise from
the shipment's own best route makes the promise self-fulfilling in isolation, so misses arise only
from a *relative* slip, not from an absolute commitment the network couldn't honor.

**Born-at-risk shipments (slack_k < 0).** With an exogenous SLA, a shipment whose own
`min_transit_k > base_transit(lane)` (e.g. an unlucky door far from any airport, or a late booking)
is committed-late from birth — realistic, but it inflates baseline tardiness independent of
congestion. Calibration guard: set `base_transit(lane)` to a **generous percentile** of the lane's
achievable transit (e.g. p90), so most shipments *can* make it and misses are congestion-driven, not
birth defects. The flex stage must handle `slack_k ≤ 0` gracefully (`flex_k = False`, on the at-risk list).

**Over-commitment (future, out of scope here).** "Taking on too many shipments vs capacity" is a
forwarder *accept/reject* decision we do not model; today it appears only via the demand / κ knobs.
Noted as a candidate extension, not part of this change.

---

## 6. Proof neutrality (L2)

`Δ_k` is identical across the M0 and M1 arms (same shipment, tier, lane, ready), so:
- **Freight-cost L2 is Δ-independent** (Δ_k does not enter freight cost).
- **Headline (W_k = 0):** the tardiness term is 0 in both arms → L2 unaffected by this change.
- **Tardiness-weighted runs (W_k > 0):** the quadratic tardiness depends on the absolute `Δ_k`, so
  diagnostics shift slightly vs the old per-shipment derivation. This is a definitional change to the
  promise, not a bug — and the new promise is the realistic one.

---

## 7. Calibration (`[CAL]`)

- `base_transit_h(lane)` — start with a **hardcoded realistic lane table** for the TPEB topology
  (≤ 9 lanes), `[CAL]`. Honest "capability-estimated" upgrade path: populate each lane's
  `base_transit` from a percentile (≈ p90) of `A_k^min` for representative shipments on that lane,
  computed **once offline per lane** (≤ 9 estimates), not per shipment.
- `sla_offset_h(tier)` — unchanged `[CAL]` (existing).

---

## 8. Implementation impact (on approval)

1. Revert the S37 pass-2 change (the transient geo-resolve for `A_k^min`) — no longer needed.
2. Generator: `Δ_k = ready_k + base_transit(lane) + sla_offset(tier)` via a lookup; `lane_k` from the
   nominal gateway pair already computed.
3. Add a `base_transit_h(lane)` `[CAL]` table (forwarder config or a generator constant).
4. `flexibility_model.md`: amend D-F6 + §2.1 step 3; note `slack_k ≤ 0` handling.
5. Tests: deadlines are now table-driven (assert `Δ_k = ready + base + offset`); the
   `backstop > Δ_k` invariant test stays green by construction.

---

## 9. Decisions for sign-off

1. **Adopt** pre-committed tier×lane SLA (`Δ_k = ready + base_transit(lane) + sla_offset(tier)`),
   replacing D-F6? (recommended: yes)
2. **`base_transit` source:** hardcoded realistic `[CAL]` lane table now, p90-capability calibration
   later? (recommended) — or compute the p90 offline now?
3. **`T^abs`:** keep `cutoff + backstop_buffer_h`? (recommended) — or make it tier-aware
   (`Δ_k + hard_buffer`)?
