# Air Freight Routing v3 — Real-World Operational Considerations

**Audit target:** `model/air_freight_routing.tex` v3 plus `air_graph_construction.md`.
**Voice:** 15-year forwarder ops lead / consolidation planner / tariff specialist.
**Scope rule:** items already covered in `03-gap-finder.md` (data fields,
rate-structure, carrier-policy nuances, customs, ULD, exception taxonomy,
replan triggers, pricing), the explicit `Deferred P1` list, and the explicit
"What is not a hard constraint" section are **excluded by construction**. So
are math/notation/formulation concerns. Findings below are net-new operational
realities a forwarder ops person would call out on first read.

**Severity ladder:**
- **CRITICAL** — would lose a pilot deal; first conversation with a real
  air-ops director kills the deal at this point.
- **BLOCKING** — model gives a wrong answer in a *common* scenario that ops
  faces weekly; daily reversion to spreadsheets follows.
- **MATERIAL** — affects >10% of shipments in some realistic slice (lane,
  customer segment, season); operator catches it manually most of the time.
- **EDGE** — real, but rare; worth documenting, not blocking MVP.

---

## Executive summary

Across the v3 spec and graph-construction doc, the formulation is
operationally **defensible at the per-shipment, per-batch, per-snapshot
level**. The serious gaps cluster in five themes that aren't picked up by the
existing 03-gap-finder findings:

1. **Truck-to-airport choreography is invisible.** The model has cargo
   "arriving at the origin door at $t_k^{\text{rdy,early}}$" then propagates
   forward through CFS dwell. But the day-of-tender clock is dominated by the
   truck-dispatch backplane: truck departs door → reach CFS → built into ULD
   → tendered to ramp **N hours before DCO**. The model collapses this into
   ground-arc dwells without acknowledging that the truck has to leave the
   shipper before cargo is technically "ready" if the dispatcher is hitting
   a tight cutoff. Two findings here, both CRITICAL.

2. **Booking acceptance is a state, not a moment.** The model treats a chosen
   MAWB-arc as if booking → confirmation → acceptance are one transaction. In
   real life the carrier returns RQ (requested), KK (confirmed), NS
   (not-accepted), or KK-conditional, and the cargo can be bumped at
   acceptance even after KK was issued. The model has no concept of a soft
   commit that needs to harden. BLOCKING for the orchestrator.

3. **Document race conditions are missing.** FWB and FHL must both be filed
   in the right window for the MAWB to clear. The cutoff stack
   ($\text{DCO} / \text{AMS} / \text{ICS2} / \text{ACI}$) is folded into one
   scalar $\text{CO}_a^*$, but real ops loses cargo to FHL-not-filed-yet
   even when the MAWB itself is on time. MATERIAL.

4. **The fallback arc is structurally clean but operationally dangerous.**
   The guarantee that the MILP is always feasible converts pre-solve
   infeasibility into a post-solve "fallback HAWB" set. In a pilot, an
   operator who sees ten fallback HAWBs at 14:00 has hours, not days, to
   rescue them. There's no built-in early-warning that fallback usage is
   *trending* before the optimizer solves. Three pilot-killer scenarios.

5. **Per-shipment cost attribution is deferred but operationally
   load-bearing.** The model returns per-MAWB cost; the deferred-P1 list
   has per-HAWB cost attribution. But quote-desk margin reasoning, customer
   billing reconciliation, and KAM QBR analysis all need per-HAWB
   attribution at MVP — without it, the optimizer's output can't be wired
   to the rest of the forwarder's workflow. CRITICAL for pilot.

**Severity distribution of net-new findings below:**
- 7 CRITICAL (pilot-killer)
- 9 BLOCKING (wrong answer in common case)
- 14 MATERIAL (>10% of shipments)
- 4 EDGE (real but rare)

Total ≈ 34 findings.

---

## Category 1 — Operational realities silently assumed away

### 1.1 The truck has to leave the warehouse N hours before DCO, not at DCO

**Scenario.** TPE export. Cargo ready at the shipper's door at 06:00 Tuesday.
DCO at TPE = 14:00 Tuesday. CFS is 45 min off-airport. Build is 90 min for a
one-MAWB consolidation. ULD tender to ramp by DCO-2h = 12:00. The
dispatcher's deadline to leave the CFS dock for the ramp is **11:15**, not
14:00. To meet the 11:15 dock-out, the truck must arrive at the CFS by
~09:00 (60 min unload + 90 min build + queue contingency). With 45 min
drive + 30 min loading at shipper = the truck must depart the shipper at
**07:45**. That's 6h 15min before DCO, not 2h.

**Model gap.** $t_k^{\text{rdy,early}}$ is the cargo-ready time at the
*origin door*. The model's forward-time-window propagation then accumulates
ground-arc dwells to the POL. But there is **no parameter** for the truck
dispatcher's pickup-window deadline working backward from DCO. The pickup
arc time is a forward-propagated transit, not a backward-binding deadline.
Result: the model will happily admit a routing where cargo is "ready" at
12:30 for a 14:00 DCO via a 45-min drive, because forward propagation says
13:15 < 14:00. In reality the truck is still in queue at the shipper when
the CFS build crew has gone home. (CLAUDE.md "no scope expansion" caveat: I
am not proposing the fix here — just flagging the gap.)

**Severity:** **CRITICAL**. The single most common cause of cutoff miss at
mid-size forwarders. If the model recommends a route the truck dispatcher
literally cannot execute, the first review session with an air-ops director
ends the pilot.

Section: §Forward-time-window propagation (\S\ref{sec:fwd-time-propagation});
§Ground-arc parameters (\S\ref{sec:ground-arc-params}).

### 1.2 Build crew capacity at origin CFS — labor and dock door

**Scenario.** Wednesday CFS-O at TPE has 12 build slots running 06:00–22:00.
Today's plan from the MILP wants four MAWBs built between 14:00 and 16:00
because four flights have 18:00 DCO. The CFS has six build crews and four
dock doors usable for outbound. The fourth MAWB physically cannot be built
in time even though the per-arc CFS-O dwell scalar says 90 min is enough.

**Model gap.** The CFS dwell arc carries a per-arc $\delta_a$ that absorbs
build time as a constant per HAWB. There is no resource constraint on
*concurrent* builds at the same CFS, no labor cap, no dock cap. The "CFS
dock capacity" deferred item only covers receiving-dock contention, not
build-side labor. Aggregate cap of concurrent builds per CFS per time window
is the missing primitive.

**Severity:** **BLOCKING** when the MILP is solving a multi-MAWB build cycle
for the same origin CFS. Acceptable at single-shipment scope. At the
volumes tier-2 forwarders run (50–300 HAWBs/week per coordinator), batch
solves routinely hit this constraint.

Section: §Ground-arc parameters (\S\ref{sec:ground-arc-params}); deferred
item \texttt{cfs-dock-capacity} only covers dock doors.

### 1.3 ULD build sequence dependency (heavy-first, fragile-last)

**Scenario.** PMC pallet on the main deck, 3,000 kg of cargo. Build sequence
is heavy-first (machinery at the bottom), apparel on top, fragile last.
Apparel HAWB arrives at CFS at 14:00. Machinery HAWB hasn't arrived yet
(expected 15:00). The CFS supervisor cannot start the build — has to wait
for the heavy piece. This is invisible to the model's CFS dwell scalar.

**Model gap.** Build dwell is a per-arc constant. There is no concept of
intra-MAWB cargo arrival sequencing affecting build start time. In real ops
this routinely causes 1–3h slippage on consolidation builds.

**Severity:** **MATERIAL**. ~20% of multi-HAWB MAWBs at mid-size where one
HAWB is materially heavier or denser than the rest.

Section: §Ground-arc parameters; deferred item \texttt{bin-packing} only
covers fit, not sequence.

### 1.4 Document race condition — FWB filed, FHL not yet

**Scenario.** TPE → JFK. MAWB-arc on CI BSA tender. Coordinator files FWB
(MAWB-level waybill data) on time at 12:00. The build crew is still
finishing the build at 12:30, so the FHL (list of HAWBs under the MAWB)
isn't filed until 13:20. DCO is 14:00. CI's acceptance system requires FHL
before the cargo physically arrives at the ramp; if FHL is missing the ULD
is held in CI's hold area. Cargo physically present, MAWB present, but
no-FHL = no tender. Customs at JFK won't have HAWB-level data, no AMS
status, no per-HAWB inbound for the importer's broker.

**Model gap.** The single $\text{CO}_a^*$ scalar collapses
$\max(\text{DCO}, \text{AMS}, \text{ICS2}, \text{ACI})$ with a per-HAWB
prep-time subtraction. It treats document filing as one synchronized
deadline. In reality FWB and FHL are sequenced; the FHL gates HAWB-level
filings; and FHL preparation has its own per-MAWB labor cost that is
correlated with HAWB count on that MAWB. A 12-HAWB consolidation MAWB has
~30 minutes of FHL prep; a 2-HAWB has ~5 minutes.

**Severity:** **MATERIAL**. Affects every multi-HAWB MAWB. Causes 5–10%
day-of-tender misses at mid-size.

Section: §Air-arc parameters (\S\ref{sec:air-arc-params}); §Cutoffs at
§Time-zone Convention end.

### 1.5 Booking-confirmation status as a state machine, not a transaction

**Scenario.** Coordinator books CI on cargo.one at 09:00 for 18:00 ETD.
cargo.one returns "RQ" (requested) — CI's revenue system has to confirm.
At 11:30 cargo.one updates to "KK" (confirmed). At 13:00 cargo.one updates
to "KK-conditional, pending DGD validation." At 14:00 cargo.one updates to
"NS, equipment swap — your PMC slot is now an AKE." The booking has been
through four states. The cargo is at CFS at 13:30 expecting to ride a PMC
load plan; the build crew has to break and rebuild.

**Model gap.** The model treats the booking as a deterministic "chose arc
$a$ → MAWB exists at $(a, g)$." There is no representation of:
(i) request-to-confirm latency;
(ii) the carrier's right to downgrade ULD type post-acceptance;
(iii) the carrier's right to refuse on policy grounds at any stage between
RQ and physical tender.
Locked-commitments §Booking-confirmation rejection recovery acknowledges
the rejection case as a workflow concern, but the conditional/partial state
between RQ and KK is not modeled at all. The orchestrator gets no early
signal that an arc's booking is still in RQ status when the optimizer
re-solves at 13:00.

**Severity:** **BLOCKING** for the orchestrator design. The "always
feasible" guarantee in the MILP is correct; but in the orchestrator layer,
a booking-status state machine is required before the model is wired into
the operator workflow.

Section: §Locked Commitments (\S\ref{sec:locked-commitments}).

### 1.6 Cargo readiness slip arriving as a WhatsApp message at 15:00

**Scenario.** MAWB build scheduled to start 16:00 Wednesday. At 14:50 the
shipper's logistics agent WhatsApps the forwarder's coordinator: "delay at
factory, cargo will arrive at CFS at 17:30, not 15:00." The model's
$[t_k^{\text{rdy,early}}, t_k^{\text{rdy,late}}]$ window was set at intake
to $[$09:00 Wed, 15:00 Wed$]$. The slip arrives mid-batch; the optimizer
runs at 09:00 every day, not on-demand.

**Model gap.** The forward-time-window propagation enforces the cargo-ready
window at graph build. There is no event for "cargo-ready-window updated"
between solves. 03-gap-finder §7.1 covers this as a replan-trigger gap;
**but** the deeper issue is that the *messaging-agent ingestion path* (the
project's claimed wedge per `messaging-agent-prior-art.md`) needs to write
back to a structured event that the orchestrator can pick up, and the air
model has no schema for "in-flight HAWB attribute change." The 7-state
shipment lifecycle in §Locked Commitments transitions are mostly carrier-
side events; there's no shipper-side event class.

**Severity:** **BLOCKING** for the orchestrator-to-MILP loop. The model
itself is fine, but the integration contract isn't specified.

Section: §Locked Commitments, supply-side lock invalidation; deferred items
\texttt{time-windowed-carrier-rules}.

### 1.7 Mixed-PER ULDs — pharma + floral within "chilled" temperature band

**Scenario.** Two HAWBs sharing $g(k) = (\text{PER, chilled})$. HAWB-1 is
fresh-cut roses (Kenya → Amsterdam, chilled +2 to +8°C, requires no
ethylene-producing co-load). HAWB-2 is vaccines (Mumbai → Frankfurt,
chilled +2 to +8°C, requires GDP-compliant cold chain, no co-load with
floral due to ethylene contamination risk). Same `g`, but operationally
**not** consolidable.

**Model gap.** §Consolidation group $g(k)$ §DGR coarseness paragraph
acknowledges DG coarseness; the same coarseness applies to PER. Pharma vs
floral within "chilled" is a real segregation requirement at the cold-chain
GHA layer (BR / LH Pharma Hub Vienna), and the partition rule treats them as
the same group. The deferred-P1 item \texttt{temp-band-refinement} is
*temperature-band* sub-division (chilled / frozen / pharma); the
**within-PER commodity-type** segregation (pharma vs floral vs seafood
within the same temperature band) is a separate omission.

**Severity:** **MATERIAL**. The forwarder loses pharma freight to the
specialized pharma carriers (LH Pharma Hub, CV PharmaHub) if it doesn't
respect this. A model that consolidates pharma with floral roses on a
single chilled MAWB will produce a recommendation that the pharma quality
manager kills on sight.

Section: §The consolidation group $g(k)$ (\S\ref{sec:g-of-k}); deferred
item \texttt{temp-band-refinement} (insufficient — also need commodity
sub-class within band).

### 1.8 Per-flight aggregate lithium cap — claimed not modelable, but workaround exists

**Scenario.** Wednesday TPE-LAX, the forwarder books two HAWBs of PI965 II
on EVA BR8 freighter (200 kg + 350 kg lithium aggregate = 550 kg). EVA's
per-flight aggregate cap on PI965 II for BR8 is 800 kg. Other forwarders
have booked 400 kg already. EVA accepts the forwarder's first HAWB (200 kg
brings flight to 600 kg, under cap); on the second (350 kg), the flight is
overflow. The forwarder's HAWB-2 is bumped to BR12 next morning.

**Model gap.** §Lithium taxonomy "Per-flight aggregate lithium caps —
carrier-side, not modeled" correctly acknowledges single forwarders can't
see other forwarders' loads. **But** there is a workaround that the model
isn't using: the forwarder *can* model an *expected* per-flight lithium
cap as a soft constraint with a tenant-calibrated risk threshold from
historical bumping data. The forwarder knows from past experience that on
BR-freighter flights, lithium quantities >X kg get bumped Y% of the time.
A per-flight expected-cap parameter would let the model avoid concentrating
lithium on one flight when the operator-historical bump rate is high.

**Severity:** **MATERIAL** for any tenant with significant lithium volume
(30–40% of TPE/HKG outbound per the lithium-taxonomy paragraph). The model
correctly says "not deterministically modelable"; the right move is to
model it stochastically with a soft penalty, not to drop it entirely.

Section: §Lithium taxonomy, "Per-flight aggregate lithium caps" paragraph
(\S\ref{sec:lithium}).

### 1.9 Carrier daily BSA cut variance — beyond 03-gap-finder §3.4

**Note:** 03-gap-finder §3.4 covers BSA daily allotment variance. **Net-new
nuance not in §3.4:** the *cause* of daily cuts on PAX-belly aircraft is
correlated across the day — when SQ overbooks SQ226 (TPE-SIN evening
pax-belly), all forwarders' allotments are cut together, and the
forwarders pile onto SQ222 (TPE-SIN afternoon) for that day's overflow.
SQ222's effective capacity tightens as a downstream effect, *not because
SQ222 itself cut allotments*. The optimizer that re-solves after the
SQ226 cut should derate SQ222 stochastically.

**Severity:** **EDGE** (already partially covered by 03-gap-finder §3.4 at
the direct level; nuance is the cross-flight cascade).

Section: complements 03-gap-finder §3.4.

### 1.10 Aircraft equipment swap can change $U_a$ post-booking

**Scenario.** Booked PMC slot on B777F. Carrier swaps to A330F freighter day
of departure (mechanical). A330F doesn't accept main-deck PMC (different
contour). Forwarder's PMC build must be broken down and rebuilt onto two
lower-deck AKEs. Build time, dwell, rate-card all change post-booking.

**Model gap.** §Locked Commitments §Supply-side lock invalidation lists
`equipment_swap` as an event, but the consequence is described as "same
flight number, different $\text{ac}(f)$, potentially different $U_a /
N_{a,u}$ on the affected arc." This is right at the catalog level. **What's
missing:** the rate-card on the affected arc may *also* change (BSA per-ULD
rates differ between PMC and AKE; the forwarder's contracted PMC rate
doesn't apply to AKE positions on the same lane). The current locked
commitments framework re-runs MILP with the affected arc removed; it
doesn't surface that the forwarder might still want the same physical
flight if the AKE pivot/rate combo is workable.

**Severity:** **MATERIAL** on routes where equipment swap rate is ≥5% (the
TPE-LAX, HKG-LAX, SIN-LAX trans-pacific freighter rotation has historical
swap rate ≥7%).

Section: §Locked Commitments §Supply-side lock invalidation
(\S\ref{sec:locked-commitments}).

### 1.11 In-bond movement when the destination broker is at a different hub

**Note:** 03-gap-finder §4.1 covers in-bond / T&E generally. **Net-new
nuance:** the forwarder's *destination broker* may be located in a third
city. Cargo arriving LAX → cleared by broker in Houston via electronic
filing → physically delivered LAX-area. The broker location influences
which CFS the forwarder uses (the broker often runs its own bonded
warehouse). The model treats destination CFS as one node fixed to the POD;
in real ops the choice is broker-dependent.

**Severity:** **EDGE** (already partially in 03-gap-finder; nuance is the
3-way location triangulation of cargo / broker / final consignee).

Section: §Ground-arc parameters; complements 03-gap-finder §4.1.

### 1.12 AWB stock allocation — origin-station-specific, finite

**Scenario.** Forwarder's TPE office has 200 unused AWB numbers from
CI-prefix MAWB stock allocated by CI for May. By the third week of May,
145 used. The TPE office is running 12 MAWBs/day; will run out around May
27. CI's restock process takes 3–5 business days. If the model recommends
3 additional CI MAWBs on May 27, AWB stock is exhausted; the coordinator
has to use a backup AWB stock (more $$$, different invoicing).

**Model gap.** Deferred-P1 item \texttt{awb-stock-management} acknowledges
this but defers it. **Net-new observation not in 03-gap-finder:** AWB stock
is *per-carrier-per-origin*, not just per-tenant. The deferred item should
specify the cardinality and the cross-period accumulator structure.

**Severity:** **MATERIAL** at tier-2 scale (the forwarder runs few enough
MAWBs per origin that stock exhaustion is a real monthly event); already
acknowledged in deferred list.

Section: Deferred item \texttt{awb-stock-management}.

### 1.13 Co-loader cutoff is earlier than direct cutoff

**Scenario.** TPE-LAX co-load via ECU. ECU's master MAWB is tendered to the
underlying carrier at DCO-3h to give ECU time to build their own MAWB. So
the forwarder's effective cutoff on a co-load arc is DCO-3h, not DCO. The
model uses one $\text{CO}_a^*$ scalar per arc that doesn't distinguish
co-load vs direct.

**Model gap.** $\text{CO}_a^*$ is computed per arc; a co-load arc and a
MAWB-arc on the same underlying flight have *different* effective cutoffs.
The graph-construction doc says co-load arcs are emitted separately from
MAWB-arcs (§5 Phase 1 step 5); the cutoff for co-load needs an additional
scalar offset = master-loader's own consol cutoff lead time.

**Severity:** **MATERIAL**. Co-loading is 15–30% of mid-tier forwarder air
volume per the synthesis F7; mis-computed co-load cutoffs cause routing
errors on every co-load arc.

Section: §Per-arc air parameters; §Procurement types.

### 1.14 Pickup vehicle type affects pickup window granularity

**Scenario.** Two pickups at the same shipper today. HAWB-1 is 200 kg
hand-carry (van pickup, can run multiple pickups). HAWB-2 is a 2,000 kg
pallet (requires a 5-ton truck with liftgate, dedicated to this pickup).
The van can be at the shipper at 08:00 and 10:00 and 12:00; the 5-ton truck
is one round trip per day. Pickup window is operationally vehicle-typed,
not just time-typed.

**Model gap.** Pickup arc time/cost is a function of distance + handling.
There is no concept of vehicle-type matching for the cargo. Pickup
infeasibility from missing the right vehicle type doesn't surface.

**Severity:** **EDGE** for the air-only model (most air pickups are
truck-handled standardized cargo). Material for the trucking model.

Section: §Ground-arc parameters.

---

## Category 2 — Edge cases that would surprise a planner

### 2.1 Express PI966 14:00 booking for next-day dispatch

**Scenario walkthrough.** PRM_AIR_EXP, 3-day SLA, PER (pharma cold-chain),
PI966 lithium declared. Booked 14:00 Day 0 for Day 1 dispatch. Cargo ready
06:00 Day 1. Route TPE → ORD via HKG with one CFS deconsol at HKG.

**What the model decides.** MILP picks the lowest-cost route that meets the
SLA. Forward-time-window propagation admits arcs based on cutoff feasibility.
The model's CO* scalar at TPE absorbs the AMS prep-time. The HKG deconsol
arc has its own $\delta_a$. C.10 quadratic tardiness biases toward the
earliest-arrival route.

**What the planner would actually decide:**
1. PI966 means the lithium is *packed with the equipment*, lower risk than
   PI965 standalone. Carrier acceptance is wider but still narrows the
   list. The planner pulls up the lithium acceptance matrix mentally — but
   also checks the **flight-specific** lithium acceptance, not just
   `lithium_accept_f` — some carriers accept PI966 on certain routings only
   if the underlying flight is freighter (not pax-belly). The model captures
   this if the matrix is keyed on `ac_type`, but most tenant catalogs at
   MVP will key on (PI, Section) and forget `ac_type`. **Gap:** ingestion
   contract for the lithium matrix not specified.
2. The pharma cold-chain on this routing requires HKG to have GDP-compliant
   handling. CV PharmaHub is at LUX, not HKG. CX has limited pharma
   capability at HKG. The planner checks the *handler* at HKG, not just
   the carrier. **Gap:** §Service products has `cargo_type_min` and
   `temperature(k)` but no per-hub-handler attribute. The CFS at HKG may
   or may not be GDP-certified — a per-CFS attribute.
3. 14:00 Day 0 booking → cargo ready 06:00 Day 1 is 16 hours — but the
   shipper hasn't actually given pickup confirmation yet. The shipper said
   "ready Day 1 AM"; the planner's experience says "shipper is consistently
   1–2 hours later than declared." **Gap:** shipper-reliability prior on
   $t_k^{\text{rdy,early}}$.
4. The planner doesn't actually book at 14:00 — they put a *soft reserve*
   on the BSA position pending shipper confirmation, then convert to firm
   at 06:00 Day 1 when cargo is at CFS. **Gap:** soft-reserve vs firm-book
   is the booking-state state-machine point (Finding 1.5).
5. The Day 1 06:00 cargo-ready is unrealistic for a 3-day SLA on
   trans-pacific PER — the planner would push back on the shipper to
   confirm 04:00 readiness, given the 16:00 DCO at TPE. **Gap:** the model
   doesn't ever push back; it accepts the window as input.

**Severity:** **BLOCKING** at the operator-trust level. A planner who runs
this case and sees a recommendation that doesn't acknowledge the GDP-hub /
shipper-reliability / soft-reserve / push-back loops will distrust the
optimizer's other recommendations even when correct.

Section: §Lithium taxonomy; §Service products; §Cargo-ready window.

### 2.2 1,200 kg PI370 Class 3 flammable + 800 kg PI965 lithium on a mixed-DG flight

**Scenario.** TPE → JFK via HKG. CI BSA on TPE-HKG, CX BSA HKG-JFK. The
forwarder has 1,200 kg PI370 (Class 3 flammable liquid, paint/solvent) and
another forwarder has booked 800 kg PI965 lithium on the same CI flight
TPE-HKG. IATA segregation tables say Class 3 and Class 9 (lithium) must be
**separated by 3m** in the same hold, OR loaded in different holds, OR in
different ULDs that themselves are separated.

**What the model decides.** Both shipments' HAWBs survive the lithium /
DGR pre-filter — PI370 lithium predicate is N/A (not lithium); PI965
lithium predicate passes per `lithium_accept_f`. The MILP routes them on
the same MAWB-arc if their `g` matches. PI370 is `g = (DGR, ambient)`;
PI965 is `g = (DGR, ambient)`. Same group. **The model would consolidate
them onto one MAWB.**

**What real ops would do.** The DG operator reviews the IATA segregation
table before consolidating. PI370 and PI965 don't co-load — different
ULDs, separated. The model's coarse DGR group is the gap. Deferred item
\texttt{pairwise-dg-segregation} acknowledges this. **Net-new observation:**
even if the model consolidates them on a MAWB, the airline's acceptance
will fail at the ramp. The model's "always feasible" guarantee then
silently routes the second HAWB through the fallback arc when the build
fails — but the build hasn't happened yet at MILP-solve time, so the
fallback usage isn't triggered until after the build is attempted. The
operator sees no warning until the CFS calls.

**Severity:** **CRITICAL** on any DG-heavy lane (HKG/TPE/SHA outbound). The
fallback arc's "feasibility guarantee" masks the operational infeasibility
at the segregation layer.

Section: §Lithium taxonomy; deferred \texttt{pairwise-dg-segregation};
fallback arc semantics.

### 2.3 Booking-confirmation returns RQ, not KK, for the chosen MAWB-arc

Covered as Finding 1.5. Re-noted here as an edge case the planner asks:
"what does the model do if cargo.one returns RQ at 11:00 for my 18:00 ETD
booking, and the operator has to commit a build sequence at 13:00 without
knowing whether it'll be KK by then?" Answer: the model has no representation
of soft commits, so the operator has to manage RQ status manually — which
is exactly the planning-tool failure mode the project is trying to fix.

### 2.4 Equalized-allowance accumulator hits zero mid-batch

**Scenario.** Forwarder's CI BSA equalized period is calendar quarter. By
mid-March (Q1 end approaching), the accumulator $A_c$ for CI is 800 kg.
Today's batch has three HAWBs eligible to ride CI's BSA: HAWB-1 (500 kg,
TPE-JFK), HAWB-2 (600 kg, TPE-LAX), HAWB-3 (400 kg, TPE-SFO). Total
chargeable weight 1,500 kg; allowance 800 kg. The optimizer charges $r_c$
on the overage = $r_c \cdot 700$ kg.

**What the model does.** C.13a treats $A_c$ as a single allowance against
the period-total chargeable weight tendered on $c$. The optimizer minimizes
overage cost; **it doesn't bias toward filling the allowance with the
highest-margin HAWBs first**. So a low-margin HAWB-1 might consume 500 of
the 800 kg allowance, leaving high-margin HAWB-2 (with a $20/kg sell
margin) paying overage on its 600 kg. The forwarder's CFO wants HAWB-2 to
get the allowance.

**What real ops does.** The allocation manager priorities the allowance
manually based on margin / strategic-account status / shipper relationship.
There is no priority parameter in the model's BSA allocation logic.

**Severity:** **BLOCKING** for forwarders with margin-tiered customer mix.
This isn't a per-period dynamics issue (03-gap-finder §2.5); it's a *within-
batch allocation priority* gap.

Section: §Equalized settlement; C.13a; complements 03-gap-finder §2.5.

### 2.5 Hub cancellation — HKG-LAX cancels with cargo already at HKG CFS

**Scenario.** TPE-HKG-LAX. Cargo cleared TPE, arrived HKG CFS at 03:00.
HKG-LAX flight cancelled at 06:00 due to mechanical. Cargo is sitting at
HKG CFS. The forwarder needs to rebook on next viable HKG-US flight.

**What the model does.** §Locked Commitments treats the cargo as partially
locked. The locked-prefix preprocessing re-points $O_k$ to HKG CFS with
$t_k^{\text{obs}} = $ 03:00 (cargo arrival). Forward propagation from
there. The graph generator emits new HKG-US arcs that the MILP can choose
from.

**What real ops does.** Same — but in real ops, the planner *also* checks:
(i) does HKG CFS have storage capacity for the cargo for 12–24h? (HKG
SuperTerminal storage charges are real OpEx);
(ii) does the cargo need to be re-screened? If TSA CCSP chain was on the
original MAWB, breaking the MAWB at HKG may invalidate the screening
status (03-gap-finder §7.2 covers screening-cert lapse; the deeper issue
is that the HKG cancellation event is the trigger for screening re-check
that isn't in the supply-side event taxonomy);
(iii) does the original tender include an extension fee with CI for the
unused HKG-LAX leg? CI may demand payment even though the leg cancelled.

**Model gap.** Locked-prefix re-pointing is correct; the dwell-cost
accumulation at HKG CFS (storage demurrage) is in deferred item
\texttt{cfs-storage-demurrage}; the screening-cert-lapse trigger is in
03-gap-finder §7.2; the unused-leg payment is not modeled at all.

**Severity:** **MATERIAL**. Hub cancellation rate is 0.5–1.5% per leg;
multi-segment routings hit it 1–2x per week per gateway.

Section: §Locked Commitments; complements deferred storage-demurrage.

### 2.6 PSS (Peak Season Surcharge) starts mid-week, before all current bookings clear

**Scenario.** Carrier announces PSS $0.40/kg effective Monday 00:00 UTC.
Forwarder has 30 bookings for the coming week that were quoted to customers
Wednesday-Friday last week, *before* the PSS was announced. The PSS applies
based on the *flight-departure date*, not the booking date. The quotes to
the customer were at last-week's rate; the carrier's invoice next week will
include PSS.

**What the model does.** Surcharge catalog has `effective_from` /
`effective_to`. The active surcharge resolver uses *current time* (`now`)
at solve. So a re-solve at Tuesday will correctly include PSS for the
Wednesday flights. But the *quote* to the customer was at Friday's
catalog snapshot.

**Model gap.** The model is correct for cost optimization; the *margin*
exposure (customer was quoted pre-PSS, forwarder pays carrier post-PSS) is
silent. This is upstream of the MILP (sell rate is "out of scope") but the
operator-facing tool needs to surface that the gap exists. The
diagnostic output (\S\ref{sec:output-diagnostics}) reports `z^{routing}`
without flagging PSS-driven margin erosion vs the customer-side quote.

**Severity:** **MATERIAL** in seasonal periods (Q4 from October, CNY
February). The margin erosion is 5–15% of base rate on affected lanes.

Section: §Surcharge; §Output and Diagnostics.

### 2.7 Cargo-ready window collapses to a point because the shipper omitted late bound

**Scenario.** Shipper email: "cargo ready Friday 09:00." Ingestion sets
$t_k^{\text{rdy,early}} = $ 09:00 Fri, $t_k^{\text{rdy,late}}$ = 09:00 Fri
(no late bound given). Window is zero-width.

**What the model does.** Forward propagation sets origin window
$[09:00, 09:00]$ — singleton. All pickup must occur at exactly 09:00.

**What real ops does.** Shipper "09:00" means somewhere in the 08:30–12:00
window with high probability, 12:00–18:00 with non-zero probability, after
18:00 with material probability. Ingestion should apply a tenant-default
spread (e.g., 4h forward) unless the shipper explicitly says "exact" — and
flag the tier in the output.

**Model gap.** 03-gap-finder §1.2 covers the cargo-ready window
flexibility ingestion issue. **Net-new nuance:** when the window collapses,
the optimizer's pickup-deferral degree of freedom collapses, and the
forward propagation becomes brittle. A single propagation hiccup makes the
HAWB unroutable (or rather, routable only via fallback). The model needs
a *minimum window width* invariant at intake.

**Severity:** **MATERIAL** (complements 03-gap-finder §1.2).

Section: §Cargo-ready window; complements 03-gap-finder §1.2.

### 2.8 The shipper's contact says "late" but the warehouse receiving record says "on time"

**Scenario.** Shipper logistics person WhatsApps coordinator: "cargo's
delayed, sending you confirmation later." 30 minutes later the CFS-O
receiving log shows the cargo arrived. The WhatsApp message was about a
different SKU within the same PO; the cargo was already in transit.

**Model gap.** This isn't a model gap; it's an orchestrator gap. The
*event ingestion path* needs reconciliation between signal channels. The
model assumes the cargo-readiness slip is a one-way event; reality has
contradictory signals from the same shipper, sometimes within minutes. The
messaging-agent design needs to defer to authoritative sources (CFS scale,
RFID) over chat.

**Severity:** **EDGE** (rare in absolute terms; high reputation cost when
the optimizer reacts to false slip).

Section: §Locked Commitments orchestrator boundary.

### 2.9 First-choice MAWB-arc was chosen because the others were filtered by carrier-policy cascade

**Scenario.** TPE-JFK. Forwarder has CI BSA, BR BSA, CV co-load arc, and
LH spot. Tenant's carrier-policy cascade denies LH on this lane (contractual
shipper restriction). Pre-filter removes the LH arc. MILP optimizes over
CI/BR/CV. CI wins. CI's daily allotment is cut at 13:00. The operator at
13:30 wants the optimizer to consider LH (the original deny may be
negotiable for this specific shipment). The model can't re-include LH
without changing the tenant carrier-policy data.

**What real ops does.** The senior operator calls the shipper, gets a
verbal one-time exception for LH, manually books LH outside the optimizer.

**Model gap.** The carrier-policy cascade is correctly tenant-mutable but
not *per-shipment-mutable* without a policy edit. There's no concept of a
"one-off exception" attribute on a HAWB that overrides the cascade for
that HAWB only.

**Severity:** **MATERIAL**. One-off exceptions are 5–15% of operator
overrides on the lanes where the cascade is restrictive.

Section: §Carrier Policy and Rules Resolution (\S\ref{sec:carrier-policy}).

### 2.10 Late-DGR-declaration breaks the already-built ULD

**Scenario.** HAWB declared GEN at intake (cargo class GEN, no lithium
spec). At CFS receiving, the build crew opens a piece and finds a small
lithium battery (PI967, contained in equipment) that the shipper forgot to
declare. The HAWB is now DGR. The MAWB has already been built around it
with GEN-only manifest. The DG operator has to break the build, re-declare
the MAWB, re-build with proper segregation.

**Model gap.** This is an in-flight HAWB-attribute change event. §Locked
Commitments lists supply-side events; demand-side events are not in the
taxonomy. The model's deterministic group assignment $g(k)$ doesn't admit
attribute changes between batches.

**Severity:** **EDGE** in absolute frequency (~1% of HAWBs); **MATERIAL**
in operational disruption when it happens.

Section: §Locked Commitments; §The consolidation group $g(k)$.

---

## Category 3 — Data-feed bottlenecks

### 3.1 The lithium-acceptance matrix is per-carrier-per-flight-per-PI-per-Section-per-ac_type

**Note.** 03-gap-finder §1.3 covers lithium spec field completeness on the
*HAWB* side. **Net-new gap:** the catalog side. `lithium_accept_f` per
§Per-flight acceptance matrix is a (PI, Section, ac_type) → bool function
that the operator has to populate per *flight*. For a tenant with 50
flights/day across 8 carriers, this is 50 × 6 PIs × 3 Sections × 2 ac_types
= 1,800 booleans, refreshed annually plus on-event. The operational reality
is the matrix is published in PDF form by each carrier (sometimes by
country regulator); structured ingestion is a hand-curation problem. The
deferred-list "annual refresh plus event-driven updates" is unrealistic
without an LLM-assisted ingestion path.

**Severity:** **CRITICAL** for the data layer. Without an industrial
lithium-matrix ingestion mechanism, the tenant will run the model with
stale matrix and produce systematically-wrong routings.

Section: §Lithium taxonomy §Sourcing.

### 3.2 Embargo records as a curated catalog

**Note.** §Sourcing and scope explicitly says "$E$ is a manually-curated
table maintained by tenant ops, seeded from carrier embargo portal pages,
IATA alerts." Real-world embargo records update weekly at minimum (carrier
adds embargo for one route, lifts for another). A tenant ops manual
maintenance is operationally fragile.

**Severity:** **CRITICAL** for the data layer; complements §3.1.

Section: §Embargo Gating §Sourcing.

### 3.3 BSA contract attributes — $N_{a,u}$, $\pi_a$, $A_c$ — are PDF-and-Excel artifacts

**Scenario.** The forwarder's CI BSA contract is a 30-page PDF + an Excel
spreadsheet with quarterly $N_{a,u}$ tables. The contracted ULD positions
vary by day-of-week. The pivot weight varies by ULD type and aircraft
type. Equalization clauses are in legalese. There is no standardized
machine-readable format.

**Model gap.** The catalog assumes $(N_{a,u}, \pi_a, r_c, A_c,
\text{settlement}_a)$ are clean per-arc parameters. Real BSA contracts have
day-of-week patterns, ULD-type-specific rates, pivot-by-aircraft, and
equalization clauses that don't map to two settlement modes. The deferred-
P1 multi-tier per-ULD pivot is one example; more important is the schema
gap at ingestion.

**Severity:** **CRITICAL** for the pilot deal. The tenant ops can't extract
the data without a dedicated parsing + validation tool.

Section: §Per-contract BSA parameters.

### 3.4 Spot rate refresh latency on aggregators

**Note.** 03-gap-finder §7.4 covers spot-rate snapshot expiry. **Net-new:**
the actual *refresh latency* on cargo.one / WebCargo / CargoAi varies by
carrier-on-platform. Some carriers refresh every 30 min; some refresh
twice a day. The tenant's snapshot frequency must be calibrated per carrier
×platform, not globally. This affects when the dynamic_spot $\text{expiry}_a$
should fire.

**Severity:** **MATERIAL** (extends 03-gap-finder §7.4 with per-platform
refresh calibration).

Section: §Dynamic spot — rate validity; complements 03-gap-finder §7.4.

### 3.5 Surcharge catalog version control across tenants

**Scenario.** Tenant A and Tenant B both use surcharges from the same
GHA at LAX (GHA-published THC). The GHA updates THC; Tenant A's ops sees
the update via direct email, Tenant B's ops doesn't see it for 2 weeks. The
two tenants now have different surcharge values for the *same* GHA arc.

**Model gap.** §Surcharge §Storage says "tenant-specific catalog,
versioned via append-only." There is no mechanism for *shared* surcharge
sources (industry-published rates that should be common across tenants).
A shared-catalog layer with tenant-override would catch the consistency
gap.

**Severity:** **MATERIAL** for multi-tenant deployment (cross-tenant
inconsistency erodes operator trust).

Section: §Surcharge parameters §Storage.

---

## Category 4 — Fallback arc masking risks

### 4.1 Fallback usage is reported per-solve, not as a trend

**Scenario.** Monday's solve has 2 fallback HAWBs. Tuesday's solve has 5.
Wednesday's solve has 9. The trend is the signal; each individual day's
fallback count looks manageable. By Wednesday's solve at 09:00, the
operator has 9 HAWBs needing rescue by end of day. Friday's expected
fallback count is 15+. No alarm fires until the day-of.

**Model gap.** §Output and Diagnostics reports $|K^{fb}|$ per solve. There
is no orchestrator-level *trend monitoring* on fallback usage. The fallback
arc design assumes the orchestrator inspects per-solve; in practice the
operator needs a *trend dashboard*.

**Severity:** **CRITICAL** for pilot. The "always feasible" guarantee is a
mathematical statement; the operational interpretation requires trend
visibility. A pilot operator who sees fallback usage triple over a week
without warning will lose faith in the system.

Section: §Output and Diagnostics; orchestrator interface.

### 4.2 Fallback selection is silent on which constraint was binding

**Scenario.** HAWB routed via fallback. The operator wants to know: was
this because pre-filter killed all arcs (and which predicate)? Because
capacity was binding? Because cutoff propagation failed? The operator has
to manually trace through 8 pre-filter predicates + capacity check +
cutoff check to understand the root cause.

**Model gap.** The pre-solve warning ("HAWB k has zero real arcs surviving
pre-filter") is logged but doesn't say which predicate ate the arcs. The
fallback selection at MILP-time has no provenance.

**Severity:** **BLOCKING** for the orchestrator's rescue workflow. Without
root-cause attribution, the rescue path is "operator manually replays the
pre-filter mentally" — defeats the point of automation.

Section: §Per-shipment subgraph pre-filter (\S\ref{sec:prefilter}); §Output
and Diagnostics.

### 4.3 Fallback arc cost obscures the "would have been cheap" failure mode

**Scenario.** Optimizer routes 3 HAWBs via fallback because of a 2h cutoff
shift on the chosen flight. Without the cutoff shift, the routing cost
would have been $4,000 total. With fallback, reported $z^{\text{fallback}}
= 3M$. The operator sees a huge cost; the *delta* between optimal-real and
fallback-real isn't surfaced.

**Model gap.** §Output and Diagnostics reports the three cost components
separately but doesn't compute the *counterfactual* cost (the cost a
slight-relax of constraints would yield). Operators need to know "if you
relax X, your fallback cost drops to Y" — sensitivity reporting.

**Severity:** **BLOCKING** for operator trust. Without sensitivity, the
fallback is a black box: "12 HAWBs failed, I don't know why or how close
they were to succeeding."

Section: §Output and Diagnostics.

---

## Category 5 — MVP scope gaps that would cost a pilot deal

### 5.1 No per-shipment cost attribution

**Scenario.** The MAWB carries 6 HAWBs at $4,800 total cost. The
operator-facing tool needs to bill each HAWB separately. Per the deferred
list, "MILP returns per-MAWB cost only; per-HAWB billing allocation is
downstream." But the *downstream* tool doesn't exist in the project scope.
The quote desk can't quote, the customer-billing reconciliation can't
reconcile, and the KAM can't QBR without per-HAWB cost.

**Model gap.** Deferred item \texttt{per-hawb-cost-attribution} is on the
list. **Net-new observation:** this should be promoted to MVP. Without it,
the optimizer is unusable in the forwarder's billing pipeline. The
proportional-to-CW attribution rule is trivial to add and unblocks the
pipeline.

**Severity:** **CRITICAL** for pilot deal. Promote from deferred to MVP.

Section: Deferred item \texttt{per-hawb-cost-attribution}.

### 5.2 Hard per-shipment budget cap exclusion is operationally correct, but pilot decision-maker doesn't know that

**Scenario.** Sales-side at the pilot forwarder asks: "what if customer X
has a $5/kg budget cap they won't exceed?" The model says: not modeled,
no hard cap. The sales person hears: "the model doesn't respect customer
budget" — and walks away.

**Model gap.** The exclusion is correct (per the spec's reasoning), but
the *commercial communication* of why is missing. A pilot deal often
hinges on this exact question. The orchestrator-layer answer ("the model
warns when expected cost > customer budget; the operator decides to
accept tardiness, push back to shipper, or escalate") needs to be in the
spec, not just implied.

**Severity:** **MATERIAL**. The exclusion is correct; the documentation
gap is the pilot risk.

Section: "What is not a hard constraint" §Per-shipment budget cap.

### 5.3 Hard SLA exclusion + fallback arc — the operator can't see "is this still feasible to my customer's SLA?"

**Scenario.** Customer SLA on PRM_AIR_EXP is 72h door-to-door. The
optimizer routes via fallback with arrival at $T_k^{\text{abs}} = $ 30
days. Quadratic tardiness contributes a huge penalty in the objective.
The operator sees fallback usage but doesn't see "this HAWB is now 28 days
beyond customer SLA." The tardiness value $\tau_k$ is post-solve but isn't
surfaced per-HAWB in the dashboard.

**Model gap.** §Output and Diagnostics surfaces $z^{\text{tardiness}}$ as
an aggregate. Per-HAWB tardiness magnitudes are derivable but not
prominently surfaced. The customer-SLA-breach quantification is implicit.

**Severity:** **BLOCKING** for the customer-comms workflow. The operator
can't draft a customer notification without per-HAWB tardiness in hours.

Section: §Output and Diagnostics.

### 5.4 Flight-level capacity is excluded — but the exclusion needs to surface a confidence flag

**Scenario.** Optimizer recommends BR227 TPE-LAX freighter, today's flight
1,500 kg total weight in the model's view. In reality, BR227 is overbooked
by 3,500 kg today (other forwarders' loads + carrier overbook). The
forwarder's recommendation isn't bumped at booking, but at acceptance the
flight is full and the forwarder's cargo rolls.

**Model gap.** §"What is not a hard constraint" §Flight-level physical
capacity correctly excludes flight-level capacity (forwarder can't observe
others' bookings). **Net-new gap:** the model doesn't surface a *risk
flag* indicating that a particular flight is operating at high utilization
historically. A simple per-flight "high-utilization risk" prior from
schedule-reliability data would let the operator know to double-book or
have backup.

**Severity:** **MATERIAL** on trans-pacific peak-season trade lanes
(BR/CI/CV TPE-LAX freighters Sep–Dec).

Section: "What is not a hard constraint"; complements synthesis F6
(schedule-reliability priors).

---

## Category 6 — Carrier policy cascade gaps

### 6.1 Rule scope can't express "depends on what other HAWBs are on this MAWB"

**Scenario.** Tenant rule: "if any HAWB on the MAWB is pharma, the
forwarder requires GDP-certified handling at all hubs." This is a rule
that conditions on **the consolidated MAWB's composition**, not on a
single HAWB's attributes.

**Model gap.** The 5-layer cascade scopes rules on (carrier, origin,
destination, hub, ac_type, cargo_type, commodity, dgr_class, lithium_pi).
None of these are "OR over the HAWBs on the same MAWB." The cascade can
encode "for pharma HAWBs, require GDP hubs"; it can't encode "for HAWBs
*sharing a MAWB with* a pharma HAWB, require GDP hubs."

**Severity:** **MATERIAL** on pharma-mixed consolidation. The model would
consolidate a pharma HAWB with an ambient HAWB on a non-GDP hub, which
the pharma quality manager would reject.

Section: §Carrier Policy.

### 6.2 Rule with "OR" semantics across deny lists

**Scenario.** Tenant rule: "deny X carrier OR Y carrier on this lane if
neither has DGR certification valid through arrival." Currently a deny is
unconditional; combining "deny X if condition Z" requires a logical
combinator the cascade doesn't have.

**Model gap.** Deny is a flat set; no conditional denies. The cascade
needs an expression layer for "deny X iff predicate(shipment, supply)."
03-gap-finder §3.1 covers time-windowed rules; this is a related but
distinct gap (conditional-on-supply-state).

**Severity:** **EDGE** at MVP; **MATERIAL** as the rule library grows.

Section: §Carrier Policy.

### 6.3 Preferred carrier set has no per-HAWB ranking

**Scenario.** Tenant prefers carriers in order: CI > BR > CX > LH. The
cascade's $C_k^{\text{pref}}$ is a *set*, not a ranking. Pass-2
lexicographic optimization maximizes the *count* of preferred carriers
used, not the *rank-weighted preference*.

**Model gap.** Preferred carrier set is unordered. A ranked preference
list with weights would let pass-2 prefer CI strictly over BR when both
are feasible.

**Severity:** **MATERIAL**. Tenant policy frequently has ordered
preferences (contractual best-tier carrier first); the model's
set-membership coarsens this.

Section: §Carrier Policy §Pass 2.

### 6.4 Cascade can't express "if A then deny B" cross-attribute rules

**Scenario.** "If shipment is pharma, deny carriers that don't have IATA
CEIV Pharma certification." This is a conditional rule on (HAWB attribute,
carrier attribute). The cascade scopes rules on shipment attributes (cargo
type) and applies denies to carriers; it doesn't express
*conditional-on-shipment* attribute matching on the carrier side.

**Model gap.** The cascade's deny semantics are unconditional within a
scoped rule. Conditional carrier-attribute matches need explicit support.
03-gap-finder §3.1 (time-windowed rules) is closest but distinct.

**Severity:** **MATERIAL** on regulated commodity lanes (pharma, perishable,
DG). Tenants would build the rule and find it doesn't express what they
need.

Section: §Carrier Policy.

---

## Category 7 — MAWB structure gaps

### 7.1 Joint hub move — one physical MAWB across two arcs (alliance code-share fleet)

**Scenario.** TPE → HKG → JFK. CI flight TPE-HKG, then CX flight HKG-JFK.
Both are oneworld alliance members with active interline. The carriers
issue **one** AWB number that covers both legs (interline through-AWB).
The graph generator (per §Through-ULD policy / §6 case A4a/b) emits one
MAWB-arc spanning both carriers. So far so good.

**But:** at hub HKG, the alliance partner runs the cargo on a *different*
flight number than originally scheduled. The interline through-AWB doesn't
re-issue. The model's one-MAWB-arc representation assumes the original
flight path stays intact. Equipment swap on the second leg invalidates
the through-arc — but the AWB number is still one document.

**Model gap.** The "one MAWB per (arc, g)" abstraction is correct for the
common case; in the joint-hub-move alliance code-share case, the AWB
*identity* persists across what the model sees as different arcs. Mapping
back to a single physical AWB for billing reconciliation is non-trivial.

**Severity:** **MATERIAL** on alliance-heavy lanes (Star Alliance Cargo:
LH/NH/UA; SkyTeam: AF/KL/DL/KE). Affects ~15-25% of long-haul interline
freight.

Section: §MAWB = (MAWB-arc, group); §Through-ULD policy.

### 7.2 Co-load arc can't carry forwarder-side HAWB-level surcharges that are MAWB-level on direct

**Scenario.** Forwarder's per-MAWB customs filing fee (AMS, ENS) is normally
billed per MAWB. On a co-load arc, there is no forwarder MAWB; the
master-loader's MAWB is the AMS-filing entity. So the forwarder bills the
shipper for *per-HAWB* documentation costs (proportional share) and not
for the per-MAWB cost. The model's MAWB fixed-charge
$c^{\text{MAWB}}_{\text{fix}}$ doesn't apply on co-load arcs (correctly),
but the equivalent per-HAWB doc cost on co-load isn't in the catalog
either.

**Model gap.** §Surcharge handles per-arc/per-shipment/per-AWB bases; the
co-load arc with HAWB-level doc charge that's *higher per kg* than the
MAWB-level cost isn't surfaced. Operators see "co-load is always cheaper"
in the model, missing the per-HAWB doc-cost surcharge that may flip the
decision on small shipments.

**Severity:** **MATERIAL** on small-HAWB co-load routes. The model
under-prices co-load on shipments below ~500 kg.

Section: §Surcharge parameters; §Procurement types — coloader.

### 7.3 Split-shipment deferred but operationally common at peak

**Note.** Deferred-P1 item \texttt{split-shipment} acknowledges this and
proposes virtual-sub-HAWBs as the preferred path. **Net-new observation
not in the deferred entry:** the *trigger* for split is often the
operator at solve-time, not at intake. The MILP's per-HAWB indivisibility
forces a binary route-or-fallback decision; the operator might prefer to
manually split the HAWB after seeing the optimizer's result. The model's
post-solve diagnostics need to flag "this HAWB would have routed if split
into two ≤ X kg pieces" — a counterfactual sensitivity surface.

**Severity:** **MATERIAL** during capacity-constrained peaks.

Section: Deferred \texttt{split-shipment}; §Output and Diagnostics
counterfactual.

### 7.4 MAWB-arc emission policy on multi-stop flights produces a combinatorial explosion

**Scenario.** Per §5 Phase 1 step 5 "overlapping enumeration policy": for
a 3-stop flight `a → b → c → d`, the generator emits segment arcs
$\{a\!\to\!b,\, b\!\to\!c,\, c\!\to\!d\}$ AND the through arc $a\!\to\!d$
AND the 2-stop arcs $a\!\to\!c,\, b\!\to\!d$. That's 6 arcs from one
physical flight. With multiple shipments having overlapping segments and
the carrier offering rates on most contiguous combinations, the arc
count grows quickly.

**Model gap.** The policy is correct for completeness, but the
**MAWB instantiation overlay (Phase 2)** then enumerates one $(a, g)$ per
arc per distinct group present — multiplying the MAWB count. For a tenant
with many multi-stop flights (Pacific Rim: TPE-ANC-JFK; TPE-GUM-HNL;
SIN-HKG-LAX), the $|M|$ at base scale could be 5-10x the estimate.

**Severity:** **EDGE** at MVP scale (100 HAWBs); **MATERIAL** at Phase 2
scale (500-1500 HAWBs). Should be measured in walking-skeleton
instrumentation.

Section: §Graph construction; §Tractability roadmap.

---

## Closing observations

**Pattern 1 — Operator workflow over-trust risk.** Several findings
(1.1, 1.4, 2.1, 2.2, 4.1, 4.2, 5.1, 5.3) point at the same risk: the
model produces an answer the operator can't *interpret* without manual
re-derivation of operational reality. The right response is not to add
constraints (the formulation is already correct); it's to add diagnostic
output, sensitivity reports, and per-HAWB attribution. None of these are
formulation changes.

**Pattern 2 — Data feeds are the binding constraint.** Findings 3.1, 3.2,
3.3 highlight that the model's correctness depends on data feeds whose
freshness and structure aren't industrial-grade. The MVP needs an
LLM-assisted ingestion layer for the lithium acceptance matrix, embargo
records, BSA contract terms, and per-flight surcharges, more than it
needs more MILP constraints.

**Pattern 3 — The fallback arc is a clever device but operationally
isolated.** Findings 4.1-4.3 say: the math is right, but the
orchestrator-side instrumentation needs to be richer. The "always
feasible" property is only useful if the operator can act on the
fallback set before SLA breach. That requires trend monitoring, root-
cause attribution, and counterfactual sensitivity — all orchestrator
features, not MILP features.

**Pattern 4 — Workflow choreography matters more than constraints.**
Findings 1.1, 1.4, 1.5, 1.6, 2.1, 2.2 all point at how the *real
day-of-tender choreography* (trucks, FWB/FHL filing, RQ→KK, last-minute
chat updates, DG re-declaration) is what the operator manages. The model
captures the *outcome* of that choreography (cargo arrives at POL by
$\text{CO}_a^*$) but not the *process* the operator orchestrates to make
it happen. The MILP can't replace the choreography; what it can do is
surface, per-HAWB, *which* choreography constraints are binding so the
operator can prioritize.

**Top 5 must-fix for pilot:**
1. **Per-HAWB cost attribution promoted from deferred to MVP** (Finding
   5.1; trivial fix; unblocks billing pipeline).
2. **Per-HAWB tardiness reporting in hours** as first-class diagnostic
   output (Finding 5.3; trivial fix; unblocks customer comms).
3. **Truck dispatch pickup-deadline parameter** alongside cargo-ready
   window (Finding 1.1; medium fix; prevents the most common cutoff-miss
   class).
4. **Fallback root-cause attribution** in the pre-filter (Finding 4.2;
   medium fix; unblocks the rescue workflow).
5. **Lithium acceptance matrix + embargo + BSA contract ingestion
   tooling** (Findings 3.1, 3.2, 3.3; large fix but unblocks every
   tenant pilot).

**Top 3 for pilot+1 (next quarter):**
6. **Booking-confirmation state machine** in the orchestrator (Finding
   1.5; medium fix; the model is correct but the orchestrator needs the
   RQ/KK/NS state).
7. **MAWB composition-conditional rules** in the cascade (Finding 6.1;
   small fix; pharma + GDP rule is a common tenant request).
8. **Per-flight risk priors** from schedule reliability data (Finding
   5.4; small fix; surfaces high-utilization risk on flights the model
   currently treats as available).
