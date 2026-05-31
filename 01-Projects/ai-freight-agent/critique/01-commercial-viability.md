# Commercial viability critique — air freight routing model v3

**Date:** 2026-05-27
**Reviewer:** Senior commercial / product critic
**Scope:** model/air_freight_routing.tex v3 (3,503 lines, 22 sections); the O-D-arc graph in model/air_graph_construction.md; the architecture (architecture.md, SYSTEM.md); reality grounding in docs/forwarder-operations-analysis/ and its day-in-the-life roll-up.
**Question:** can this model automate or replace 90%+ of the air planning/replanning work at a typical tier-2 forwarder?
**Out of scope by instruction:** math/formulation/scalability correctness — covered elsewhere.

---

## 1. Headline verdict

No. The model — even fully built and stably solving — covers a real, narrow, and under-served slice of the air planning job. It does *not* automate the majority of what an air export coordinator, allocation manager, consolidation planner, exception handler, or CFS-interface supervisor actually do day-to-day. Best read: **end-to-end no-human-in-the-loop automation at 5–15% of the air planning/replanning hour-share; semi-automation (model recommends, human approves) at an additional 25–40%; permanently out-of-scope at 50–65%.**

The 90% target is the wrong target. The right framing — already implicit in the project's own day-in-the-life rollup — is the "3% of shipments consume 40% of the day" Pareto. This model lives inside that 40%, specifically inside the consolidation-planning sub-block (Persona 2 §9 in the DITL: 2–4 hours/day for a senior coordinator) and the air-exception re-plan sub-block (Persona 4 §8.1: 6–8 hours per cancellation event, ~one per active week). That is a genuine, defensible wedge — but it is not 90% of the air-ops surface area.

---

## 2. What the model actually does well

To be specific about what the v3 LaTeX commits to:

- **Routes HAWBs over an O-D-arc graph with MAWB-as-(arc, group) consolidation logic** — this is the right level of abstraction. The bucket-per-flight-leg formulation that the v3 replaced was structurally wrong; v3 is the right shape for productionizing a consolidation-aware MILP.
- **Encodes the four rate families** (`flat_rate`, `min_flat_breaks`, `per_uld_pivot`, `coload_per_kg`), the IATA next-break-down rule, and BSA settlement with both per-flight and equalized-allowance mechanics. This is a genuinely difficult piece of commercial logic and the doc nails it.
- **Carrier-policy cascade** (tenant blacklist / shipper-lane / service-product / lane preference / commodity overlay) with intersect-allows / union-denies is the right rule resolution. Operators will recognize this immediately.
- **Cargo-type and embargo / lithium gating, screening cost, MCT-folded-into-arc-scalars, locked-commitment preprocessing, quadratic tardiness** — all design choices that show the author has thought hard about the gap between textbook MILP routing and real freight planning.
- **Fallback arc as guaranteed-feasibility mechanism** — this is the single best design decision in the v3. It converts solver INFEASIBLE into a structured rescue signal, which is *exactly* the right shape for putting an MILP into production where operators cannot tolerate "no plan." Most academic MILPs do not do this; most production MILPs hack it in after the first time the solver returns INFEASIBLE in front of a customer.

Scope-wise, what the model competently addresses:

- **P1 multi-shipment grouping** — the consolidation planner's core decision.
- **P2 MAWB-vs-co-load** — the cost/control trade-off the planner makes manually today.
- **P3 cutoff-vs-ready-window** — soft via the tardiness penalty and the cutoff scalar.
- **P4 hub-vs-direct** — falls out of the graph.
- **P5 compatibility-group selection within cost** — handled by the group function and pre-filter.

These are five of the project's stated wedge tasks. The model is genuinely useful here. No incumbent tool at mid-size handles P1–P5 as a joint optimization, per the forwarder-ops research.

---

## 3. Realism for production use — is it usable?

**Operationally, yes — with caveats. Commercially, only inside a narrow slice.**

The realism is unusually high for an academic-shaped MILP: UTC-normative time handling with SSIM ingestion notes; the cutoff stack (DCO + AMS + ICS2 + ACI) folded into a per-arc scalar; per-HAWB document-prep time; locked commitments handled at preprocessing rather than as MILP constraints; surcharges as a separate per-arc resolution layer; lithium PI gating and embargo as pre-filter predicates. This is the level of detail you only get from someone who has actually shipped this kind of system before.

The model's biggest "production-readiness" question is **data, not math**. Specifically:

1. **The rate catalog and BSA contract feed.** The MILP requires per-arc `rate_family`, `pivot`, `cap`, `allotment`, `surcharges`, `expiry`, and per-contract `A_c` allowance-remaining values. Forwarders today maintain this in CargoWise rate modules, spreadsheets, GSA emails, and senior-coordinator heads. There is no clean API surface for it. The MILP is only as good as this feed, and the feed is mostly manual. The model assumes the catalog exists; in reality the catalog *is* the project, and it lives upstream.
2. **The schedule + cutoff + MCT data per arc.** The model assumes the graph generator has folded all this in. In practice, schedule reliability varies 57–90% by alliance (Sea-Intelligence), MCT is published per hub per carrier per cargo type, and "carrier acceptance cutoff" is famously fuzzy (T-2h to T-6h depending on relationship, day of week, and operational state). The model's per-arc scalar is a clean abstraction; producing the scalar reliably is a nontrivial data engineering problem the LaTeX doesn't own.
3. **The pre-filter predicates** (cargo-type, embargo, lithium PI, screening, carrier eligibility cascade) require carrier-by-carrier operator-variation tables. IATA DGR + operator variations is real, knowable, but unmaintained at mid-size today. This is a "buy or build" decision the model presupposes is already answered.

**Net:** the model is realistic enough that, given the data feeds, an operator would not laugh at the output. But the data feeds are *the* delivery challenge, and they live outside the LaTeX.

---

## 4. Automation %, broken down

The DITL data lets us score this precisely. Time-allocation tables are `Inferred:` upstream but the shape is defensible.

### 4.1 What the model can automate end-to-end (no human in loop) — 5–15% of air ops hours

**Where this realistically lands:**

- Consolidation planner's eligibility-filter + bucket-assembly step (Persona 2 §5.5, 25% of the planner's day) → MILP runs unattended on batch cadence. **But:** the planner is a 2–4 hr/day hat, not a full FTE — so 25% × 2–4 hr ÷ 8 hr workday ≈ 6–12% of one senior coordinator's clock.
- Routine carrier ranking inside an obvious-winner BSA pull (Persona 2 §A1 booking, 25% of the air coordinator's day) → only a fraction is truly hands-off; commitment-first BSA logic plus carrier blacklist is the easy part, and even cargo.one's October 2025 AI Quoting tool still produces "ready-to-share customer quotes that consistently match the rate selections that human experts prefer themselves" — i.e., it recommends; the human still ships. The MILP can do better than gut on the routing math, but it does not replace the human commit step at MVP trust levels.
- Pre-commit batch re-solves driven by rate-catalog refresh, schedule updates, allocation deltas — fully unattended once the deployment ladder reaches "autonomous per-lane" (per architecture.md §10, which itself gates on sustained <8% override).

**Hard ceiling on end-to-end automation:** the override-rate-as-KPI memory and the project's own deployment ladder explicitly require a co-pilot → supervised → autonomous progression with minimum-weeks gates. The MILP *can* run unattended at MVP scale, but the project's own architecture says it *shouldn't* until trust is earned per lane. That's a strategic choice, not a model limitation — but it caps real-world end-to-end automation in year-one well below what the math could in principle deliver.

### 4.2 What it can semi-automate (model recommends, human approves) — 25–40%

This is where the model earns most of its keep:

- **Consolidation plan publication.** The planner reviews the MILP output, tweaks where their tribal knowledge says the solver is wrong, commits. The hour-share captured here is roughly the planner's full 25%+20%+10%+10% = 65% of their decision time (eligibility, MAWB-vs-co-load, hub-vs-direct, cutoff-vs-ready). Compressed from "afternoon spreadsheet block" to "review solver output," which is genuine, defensible value.
- **Single-shipment disruption re-route.** When a flight cancels and 8 HAWBs need rebooking (Persona 4 §8.1 walk-through), the MILP re-solves the affected subgraph and the exception handler approves. Today: 4–8 hours of focused work. With MILP + LLM explanation: plausibly 30–90 minutes. This is a 75–85% time reduction on a high-leverage event class.
- **Carrier re-tender ranking** under allocation, BSA, blacklist constraints — A6 cutoff-miss re-tender in Persona 2 §A6 (10–15% of the air coordinator's heavy-cutoff day).
- **Re-plan trade-off generation** — the model produces top-N alternates with costs; LLM explains; operator picks. Architecture.md §7.2 already names this as MVP capability 4.

### 4.3 What is permanently out of scope (always needs humans) — 50–65%

This is the largest bucket and it's important to be clear about what's here. From the DITL persona time tables:

- **Comms-handling** is the dominant time-sink in 12 of 17 surveyed roles (DITL §2). The MILP routes; it does not draft customer emails, parse partner WhatsApps, handle phone calls, or process voice notes. The LLM agent layer addresses some of this; the *MILP itself* addresses none of it.
- **Cutoff coordination with CFS and GHA** — air coordinator's 15–30% of day. This is phone+portal work happening *after* the plan is committed; MILP is upstream of it.
- **MAWB issuance / FWB / FHL EDI** — 15% of the air coordinator's day. Pure TMS workflow.
- **Pre-alert generation and destination handoff** — 10% of the air coordinator's day. TMS workflow.
- **CFS supervisor's floor-walk, build supervision, screening handoff, ramp tender coordination** — 70%+ of the CFS supervisor's day. Physical, in-warehouse, MILP-irrelevant.
- **Carrier acceptance variation negotiation** (lithium SoC, oversize on different aircraft, temperature-band cool-chain, pharma corridor SME calls). Judgment + relationship.
- **Customer relationship + commercial decisions** (account manager: do we eat the cost or charge the customer? do we accept SLA breach or spend $2,800 on a premium reroute? do we commit to a high-value customer's "ship as usual" WhatsApp with no shipment ID?).
- **Damage / claim / force-majeure judgment** (Persona 4 §4.7, §6.9).
- **Network-event portfolio decisions** at the director level (Red Sea, ILA strike — multi-customer, multi-mode, days of senior judgment).
- **WhatsApp / voice / partner free-text** — explicitly the project's *separate* AI-agent wedge per the synthesis F3 finding; not addressed by this MILP at all.
- **Customs / PGA hold response, CF-28/29 drafting, HS classification** — Persona 3, entirely out of scope for this model.

The DITL §6 explicitly identifies that *the consolidation planner's tool stack is the thinnest of all 18 surveyed roles* — Excel, WhatsApp threads, and tribal knowledge. That's exactly the gap the MILP fills. But filling it does not extend the MILP into the 16 other roles, and most operational hours sit in those 16 roles.

---

## 5. Top 5 blockers to 90%+ end-to-end automation, with lift estimates

### 5.1 The model only owns the routing math; it does not own the data feeds it depends on

**What's missing:** a continuously refreshed rate catalog (BSA, NAC, GSA, TACT, spot — 5-tier per architecture.md §8), a schedule + cutoff + MCT feed per arc, a carrier-eligibility table including lithium operator variations and embargoes, and a consumed-weight accumulator for equalized-settlement BSA contracts.

**Lift to close:** large — 6–12 months of data engineering per design partner, with ongoing maintenance burden. cargo.one / WebCargo / CargoAi cover ~70% of rate catalog needs via aggregator APIs at a per-booking cost; the BSA / NAC contract side is per-tenant and manual. The "5-tier rate strategy" in architecture.md is the right plan; the execution is non-trivial. Until this exists, the MILP is a paperweight.

**Why this is #1:** the LaTeX assumes all this data exists in the graph generator's output. In real deployments at mid-size, *the data does not exist in the required shape*. Most of the project's actual delivery work will be data engineering, not solver tuning.

### 5.2 Trust ladder explicitly caps autonomous coverage in year-one

**What's blocking:** the project's own architecture.md §10 specifies a 4-week co-pilot → 8-week supervised → per-lane autonomous progression gated on sustained <8% override. This is the *correct* discipline (per the 95% accuracy trap and override-rate-as-KPI memory), and it directly precludes 90% end-to-end automation in the first year of any customer engagement.

**Lift to close:** not a "close" — it's a deliberate design constraint. The right question is "how much can be autonomous per lane after 6 months of operation," and the honest answer is "the obvious-winner BSA pulls and the steady-state routine consolidations on stable lanes" — maybe 20–30% of the planner's decisions on a mature deployment, never 90%.

### 5.3 The "is this material?" question is not a MILP question

**What's missing:** materiality assessment (architecture.md §7.2 MVP capability 3; synthesis F8) is a load-bearing AI judgment, not a MILP output. The MILP can re-solve when triggered; deciding *whether* to re-solve given a 6-hour ETA shift requires customer SLA + downstream commitments + cost context. Today: senior judgment. Tomorrow: LLM agent reading TMS context, customer profile, and downstream commitments.

**Lift to close:** medium to large. Building a materiality scorer with the right features (per-customer SLA, downstream production schedule where known, lane-volatility prior, value coefficient) is a 3–6 month ML+LLM build, and ground truth is hard to come by (the DITL flags "severity-assessment ground truth" as item 4 in its open empirical questions). Without this, the MILP has to re-solve on every trigger, which floods operators with low-value re-plans and erodes trust.

### 5.4 Carrier eligibility, lithium operator variations, and embargo lists are unmaintained at mid-size

**What's missing:** per-carrier operator variations for IATA DGR, especially PI965/PI967 lithium battery SoC requirements (post-Jan 2026 mandate per memory `project_air_screening_decision.md`), per-carrier embargo lists, per-aircraft-type capacity constraints (777F vs 767F for oversize), and the chain-of-custody-aware screening status table. The pre-filter requires all of these.

**Lift to close:** medium. Vendor data exists (IATA DGR is licensed; operator variations are formally additions to DGR per Lion Technology). DG-checker tools (IATA DG AutoCheck, CHAMP eDGD) cover most of this. The lift is integration + tenant-side data hygiene, not algorithm work. Probably 3–4 months for a competent integration engineer per tenant if the source data is licensed.

### 5.5 The "I told you so" loop — handling override patterns

**What's missing:** the project's KPI is override rate. When operators systematically override the MILP for reasons not encoded in the model — "carrier X chronically late on Tuesdays out of CGK," "this customer always wants KE not OZ regardless of cost," "DG 1A is informally segregated harder than IATA requires at this CFS" — the model needs to learn. The override log (logs/overrides.jsonl with structured reason codes per architecture.md §9) is the right shape; but the *feedback loop* from override → rule update / rate update / preference update is not yet specified.

**Lift to close:** medium. The data plumbing is straightforward; the hard part is the policy-update workflow (who reviews, who approves, when does a one-off override become a tenant rule, how do you avoid encoding biased operator preferences). This is a 6–12 month operational maturity question, not a 6-month build question. Without this loop, override rate climbs and the deployment ladder regresses from autonomous → supervised → co-pilot — the trust degradation pattern the memory specifically warns about.

---

## 6. Upside surprises — capabilities I didn't expect

Three places where the model is meaningfully better than I expected from reading the abstract:

1. **The fallback arc is a genuinely sophisticated design choice.** Most academic MILPs of this kind would just let HiGHS return INFEASIBLE and call it a day. The fallback arc with finite cost, finite arrival at `T_k^abs`, exempt-from-pre-filter, and post-solve structured-rescue inspection is exactly the production-grade design that makes the difference between "MILP that wins a paper" and "MILP that survives a Monday morning at a forwarder." It also dovetails cleanly with the LLM agent's materiality-assessment role — a fallback selection *is* a materiality signal.

2. **The quadratic tardiness penalty with PWL linearization and value-coefficient scaling.** Combining `w_sp(k)` (service-product tier weight) with `μ_k = value_k / V_ref` (per-HAWB shipment-value coefficient) is sharp. It naturally prevents the optimizer from concentrating slip on one shipment and gives a clean knob for the service-product team to tune Express vs Standard vs Economy without re-architecting. Most forwarders don't think this hard about their own service tiers; the model could surface inconsistencies in tenant tier definitions as a side benefit.

3. **The MAWB-as-(arc, g) abstraction with no `h_{k,m}` explicit assignment variable and no permutation symmetry.** This is the right move and it's not obvious. Most consolidation MILPs from the OR literature carry explicit MAWB-count decisions and pay heavily in symmetry-breaking. The v3 design — group is a single-valued function of (cargo_class, temperature_regime); HAWB rides MAWB iff `x_{k,a}=1` and `g(k)=g` — gets you the consolidation semantics for free. This will solve faster than a naive formulation by a noticeable margin at scale and will produce more interpretable solutions for operators.

---

## 7. Risks of false confidence — where the model looks like it handles things but doesn't

These are the places where an enthusiastic demo would mislead a buyer:

### 7.1 The model "handles" capacity but actually doesn't see physical flight capacity

The problem-statement is explicit: "Flight-level physical capacity $W_f, V_f$ ... The forwarder cannot observe other parties' bookings on the same flight; carrier overbooking and offload is the airline's responsibility." This is *operationally correct* — forwarders genuinely don't see this — but a customer demo where the MILP confidently allocates 12 ULDs to a flight that the airline then bumps because they sold the belly space to passengers will look like the MILP failed. The pitch needs to clearly position this as "the MILP plans against your contracted allocations and the airline owns flight-level capacity." Misrepresent this and operator trust collapses on the first bump.

### 7.2 The model "handles" cutoff but the cutoff time is a soft, relationship-dependent number

The model folds DCO + AMS + ICS2 + ACI into a per-arc scalar `CO_a*` and binds against it as a hard constraint (C.9). In reality, "carrier acceptance cutoff" varies T-2h to T-6h based on relationship, day of week, and how much the carrier wants the freight. Encoding it as a hard constraint will produce conservative plans that leave money on the table when the relationship would have absorbed a late tender. The fallback-arc safety net handles the catastrophic case; the day-to-day "we routinely cutoff 30 min late on this lane with this rep" requires either tenant-tunable cutoff scalars or operator override capability. The model doesn't currently expose this knob.

### 7.3 Locked-commitments preprocessing assumes the lock state is clean

C.12 says locked HAWBs are removed from K or their origin re-pointed. But real locks are messier — partial physical movement (cargo at CFS-O but not tendered), conditional commitments (carrier accepted pending screening), MAWB-issued-but-cancellable-with-penalty. The "fully locked vs partially locked" binary is too clean. Production deployments will encounter a tail of edge-case lock states that the preprocessing layer will fail to classify, and the failure mode is silent (HAWB enters MILP with wrong state). This is the kind of thing that surfaces only after 3–6 months of real deployment data.

### 7.4 The screening cost-not-eligibility decision

The model removed screening as a grouping key and an arc filter (per memory `project_air_screening_decision.md`), treating it as a per-kg origin cost only. This is defensible at MVP and explicitly punts on re-screen handling. But screening status loss in transit (chain-of-custody break per TSA CCSP) is real and material — a HAWB that loses screening status at a hub forces 30–45 min/HAWB re-screen at the next handoff (Persona 2 §4.4 walk-through). The MILP doesn't see this; the per-arc scalar approach misses chain-of-custody dependencies entirely. Plans that look feasible will fail on the floor.

### 7.5 "Output the plan" is not the same as "execute the plan"

The MILP produces `x_{k,a}`, `z_{a,g}`, `CW_{a,g}` and a cost. Operators then have to:
- Book MAWBs in carrier portals (or via WebCargo/cargo.one/CargoAi APIs).
- Issue FWB/FHL.
- Coordinate with CFS on build sheets matching the MILP's plan.
- Notify partners.
- Update CargoWise milestone fields.

The model is silent on all of this. The execution gap is large enough that a fully-correct MILP plan can still produce zero operational lift if the execution layer (Bucket A workflow per the synthesis) isn't built underneath. The "we build" / "we integrate" boundary in architecture.md §11 is correct; the work to do it is large; the model does not by itself imply the execution layer.

### 7.6 The 90% target itself

The most important false-confidence risk: pitching "AI replaces 90% of air planner work" sets up the project to lose. Even ten years of incremental improvement on this MILP wouldn't reach 90% because >50% of the air planning hour-share is comms, physical CFS work, customer relationship judgment, and unstructured-channel inference — which this model architecturally does not address. The honest pitch — "we compress your worst 3% of shipments from 40% of the day to 15% of the day" — is more defensible and matches what the deal economics actually need. Don't oversell.

---

## 8. The commercial frame — what this model is actually worth

Strip away the technical depth and ask: what's the realistic per-tenant value capture?

**The wedge is real.** The consolidation planner role has the thinnest tooling stack of any of the 18 surveyed forwarder roles (DITL §6). No mid-market product addresses P1–P5 jointly. cargo.one's March 2026 AI-native OS targets booking/quoting/rate management/CS — not multi-shipment consolidation MILP. project44's Booking Connect for Ocean re-books within "pre-configured rules," not optimization. The competitive window exists; it is shrinking; this MILP is competently aimed at it.

**But the ROI is not "we save 90% of planner time."** It's:

- 30–60% time compression on the consolidation planning afternoon block (the senior coordinator's 2–4 hr/day hat) — measurable, defensible.
- 60–80% time compression on the air-exception re-plan event (Persona 4 §8.1's 6–8 hour single-cancellation case) — high-leverage, lumpy, hard to sell on per-hour ROI but easy to sell on customer-experience-of-disruption ROI.
- Latent value in *better* plans (higher consolidation ratio, better MAWB-vs-co-load split, fewer co-load defaults from "we ran out of time to plan") — hard to measure pre-deployment, large in aggregate.
- Zero direct ROI on cutoff coordination, MAWB issuance, FWB/FHL, pre-alert, CFS floor work, customer comms, customs filings, claims. These remain the operator's job.

A reasonable elevator pitch: "we compress your consolidation planner's afternoon from 4 hours to 90 minutes and your air-cancellation re-plan from a workday to a coffee break. We don't touch the 60% of the day that's comms, CFS floor work, and customer relationship — and we shouldn't." That pitch is honest, defensible, and matches what the model actually does.

---

## 9. Recommendations (for the user, not the deliverable)

1. **Re-frame the automation target.** Stop using 90% as the goalpost. The DITL's "3% of shipments consume 40% of the day" Pareto is the right framing. Target: compress that 40% to 20% within Year 1 of a customer deployment.
2. **Treat the data pipeline as a first-class deliverable**, not an afterthought. Rate catalog, BSA accumulator, carrier-eligibility table, schedule + cutoff feed are *the* delivery risk. Allocate engineering effort accordingly — probably 50% data engineering, 30% MILP code, 20% LLM agent layer for MVP.
3. **Land the LLM agent's materiality assessment alongside the MILP**, not after. Without it, the MILP triggers too often and erodes trust. With it, the loop closes.
4. **Be explicit in the pitch about what the MILP doesn't do** — and route execution-layer work (TMS integration, EDI generation, carrier-portal booking) through CargoWise integration rather than building it. The architecture already commits to this; the pitch and the sales motion should match.
5. **Track override rate from day one, per-lane.** Architecture.md §10 already commits to this. Honor it. Don't compress the deployment ladder under sales pressure.

---

## Executive verdict

The v3 air freight MILP is a sharp, production-shaped piece of optimization work that competently addresses a real, under-served wedge — multi-shipment consolidation planning and air-exception re-routing at mid-size forwarders. It will not automate 90% of air planner work; that target is wrong. The honest envelope is 5–15% true end-to-end automation, 25–40% semi-automation (model recommends, human approves), 50–65% permanently out of scope (comms, CFS floor work, customer relationship, customs, unstructured-channel inference, execution-layer plumbing). The model's biggest delivery risks are not in the math but in the rate-catalog / schedule / carrier-eligibility data feeds it depends on. Reframe the pitch as "compress your worst 3% from 40% of the day to 15%," ship the materiality-assessment LLM alongside, and the wedge is defensible. Overselling the 90% number is the failure mode.
