# References — Two-stage reserve/assign in air-cargo allotments

Verified 2026-07-09. These are the four papers behind the S52 grounding claim that
the standard academic framing of air-cargo capacity is *two-stage*: **stage 1 =
reserve the block before demand is known; stage 2 = decide what fills it after
demand materializes.** See `docs/design/air_cargo_reserve_assign_grounding_s52.md`
and memory `reference_air_cargo_reserve_assign_practice`.

Every author list, journal, volume, and page range below was checked against
publisher / RePEc metadata. Abstracts are marked **[verbatim]** (pulled from the
source) or **[paraphrase]** (summarized from an indexer — pull the PDF before
quoting).

> **⚠️ SCOPING — these are procurement/contract-side papers, NOT the routing
> engine.** Every paper here solves some version of *how much BSA/allotment to
> procure* (stage 1 = the block-space quantity decision), or the carrier-side
> allocation/contract design. **Our air planning problem takes BSA as already
> procured and does real-time routing/assignment against it.** In their two-stage
> language, our problem maps to the **second stage only** — `Q(x, ξ)` with the
> allotment `x` held **fixed** — and even that is thinner than ours, because their
> stage-2 demand materializes in one shot per day (no arrival stream, no booking
> lead, no commit timing). Do **not** cite these as models for the routing engine
> or import their headline results (newsvendor critical ratio, "current policy
> over-reserves 3×," the 54% cost gap) — those answer the procurement question,
> which is upstream of and out of scope for our router. Use them only as (a)
> evidence the reserve-early/assign-late split is standard practice, and (b)
> characterization of the upstream procurement problem that sets the `x` our
> engine treats as given.

> **Two corrections to earlier internal citations (read before reusing):**
>
> 1. **"INFORMS" applies to only one of these four.** Only Levin, Nediak &
>    Topaloglu is an INFORMS paper (*Operations Research*). The three Amaruchkul
>    works are **not** INFORMS venues: TR-E is Elsevier, ICORES is SciTePress,
>    ITOR is Wiley (for IFORS). If a prior note grouped all four under "INFORMS,"
>    that label is wrong for the Amaruchkul three.
>
> 2. **The 2011 TR-E paper is Amaruchkul & Lorchirachoonkul — not "Cooper &
>    Gupta."** `docs/academic-literature-references.md` and
>    `references/air-cargo-allotment-contracts.md` both attribute TR-E 47(1):30–40
>    to "Amaruchkul, Cooper & Gupta (2011)." RePEc/Elsevier metadata for that
>    exact volume/issue/pages lists **Kannapha Amaruchkul & Vichit
>    Lorchirachoonkul**. The likely source of the error is conflation with the
>    real Amaruchkul–Cooper–Gupta air-cargo RM paper (*Transportation Science*
>    2007). Those two existing docs should be corrected.

---

## 1. Levin, Nediak & Topaloglu (2012) — the two-stage carrier problem [INFORMS]

**Citation.** Levin, Y., Nediak, M., & Topaloglu, H. (2012). Cargo Capacity
Management with Allotments and Spot Market Demand. *Operations Research*, 60(2),
351–365.

**DOI.** https://doi.org/10.1287/opre.1110.1023

**Author PDF (manuscript OPRE-2008-08-420.R3).** https://people.orie.cornell.edu/huseyin/publications/allotments.pdf

**Abstract [verbatim].** "We consider a problem faced by an airline that operates a
number of parallel flights to transport cargo between a particular origin
destination pair. The airline can sell its cargo capacity either through allotment
contracts or on the spot market where customers exhibit choice behavior between
different flights. The goal is to simultaneously select allotment contracts among
available bids and find a booking control policy for the spot market so as to
maximize the sum of the profit from the allotments and the total expected profit
from the spot market. We formulate the booking control problem on the spot market
as a dynamic program and construct approximations to its value functions, which can
be used to estimate the total expected profit from the spot market. We show that
our value function approximations provide upper bounds on the optimal total
expected profit from the spot market and they allow us to solve the allotment
selection problem through a sequence of linear mixed integer programs with a special
structure. Furthermore, the value function approximations are useful for
constructing a booking control policy for the spot market with desirable monotonic
properties. Computational experiments show that the proposed approach can be scaled
to realistic problems and provides well performing allotment allocation and booking
control decisions."

**Two-stage evidence (verbatim, §1 intro).** "The airline faces two problems that
interact tightly with each other. The first problem is to determine what contracts,
if any, should be signed with potential allotment customers… After setting up the
allotment contracts, the airline faces the second problem that determines which
booking requests to accept on the spot market." Departure-time uncertainty: "…the
fact that not all of the booked requests show up at the departure time…"

---

## 2. Amaruchkul & Lorchirachoonkul (2011) — carrier allocation to many forwarders

**Citation.** Amaruchkul, K., & Lorchirachoonkul, V. (2011). Air-cargo capacity
allocation for multiple freight forwarders. *Transportation Research Part E:
Logistics and Transportation Review*, 47(1), 30–40.

**DOI.** https://doi.org/10.1016/j.tre.2010.07.006

**Link (publisher).** https://www.sciencedirect.com/science/article/abs/pii/S1366554510000797
· **RePEc.** https://ideas.repec.org/a/eee/transe/v47y2011i1p30-40.html

**Abstract [paraphrase — verify against PDF before quoting].** Examines how a single
air-cargo carrier can optimally distribute cargo space among multiple freight
forwarders. The carrier's revenue depends on actual usage of allocated capacity
during the booking period. Uses a discrete Markov chain to model usage-probability
distributions and dynamic programming to solve the allocation; introduces two
heuristics for large-scale instances, validated computationally.

**Relevance.** Carrier-side allotment decision — the stage-1 "reserve" mechanism as
seen from the carrier allocating blocks across competing forwarders.

---

## 3. Amaruchkul (2018) — game-theoretic allotment contract

**Citation.** Amaruchkul, K. (2018). Game-theoretic Analysis of Air-cargo Allotment
Contract. In *Proceedings of the 7th International Conference on Operations Research
and Enterprise Systems (ICORES 2018)*, pp. 47–58. SciTePress.

**DOI.** 10.5220/0006551800470058 *(as recorded on the locally-saved PDF; an
indexer returned a transposed variant — the pages-consistent form is used here.)*

**Link.** https://www.scitepress.org/publishedPapers/2018/65518/pdf/index.html
· **Local PDF.** `references/Amaruchkul 2018 - Game-theoretic Analysis of Air-cargo Allotment Contract (ICORES).pdf`

**Abstract [verbatim].** "Consider an air-cargo carrier and a freight forwarder,
which may establish an allotment contract at the start of the season. The allotment
size needs to be determined, before their stochastic demands materialize. The
forwarder hopes to receive a discount rate, lower than the spot rate. The carrier
hopes to increase capacity utilization by handling not only its own direct-ship
demand but also the forwarder's demand. We formulate a Stackelberg game, in which
the carrier is the leader and offers contract parameters such as the wholesale
price and the refund rate for the unused allotment as well as the minimum allotment
utilization. Given the carrier's offer, the forwarder decides how much to book as an
allotment, in order to maximize its own expected profit. We analyze the game and
identify conditions, in which an equilibrium contract coordinates the air-cargo
chain. We show that the minimum allotment utilization is needed to construct a
coordinating contract. In our numerical examples, we illustrate how to apply our
approach to the case study of one of the biggest forwarders in Thailand. The
contract can improve both parties' profits, compared to the scenario without any
contract, where the forwarder purchases all capacity from the spot market."

**Relevance.** Full contract mechanics (wholesale price, unused-allotment refund,
minimum utilization) already mapped to the project in
`references/air-cargo-allotment-contracts.md`. The "allotment size determined before
stochastic demands materialize" line is the cleanest single-sentence statement of
stage 1.

---

## 4. Amaruchkul (2025) — explicit two-stage stochastic + robust formulation ★

**Citation.** Amaruchkul, K. (2025). Capacity management of forwarder with multiple
carriers under uncertain flight travel time and stochastic shipment demand.
*International Transactions in Operational Research*, 32(6), 3619–3666.

**DOI.** https://doi.org/10.1111/itor.13613

**Link (Wiley).** https://onlinelibrary.wiley.com/doi/10.1111/itor.13613

**Abstract [paraphrase — verify against PDF before quoting].** A forwarder obtains
cargo space from multiple carriers: some capacity reserved long-term via allotment
contracts, some booked short-term daily. Proposes a unified program for both
horizons under two kinds of uncertainty — demand modeled by a probability
distribution (stochastic program) and flight travel times by an uncertainty set
(robust optimization). **Stage 1:** allotments are determined. **Stage 2:** shipment
demand materializes (travel time still unknown) and the forwarder sets the daily
allocation among flights given the stage-1 allotment. Objective: minimize expected
total cost across the two stages. Derives a closed-form solution for a special case,
proposes a heuristic for the general case, and evaluates it on historical records
from one of the largest Thai forwarders.

**Relevance.** The most directly aligned paper: forwarder-side, explicitly
two-stage, with the reserve-then-fill decision structure the project's soft-plan →
commit lifecycle mirrors. Also listed in `docs/academic-literature-references.md`
(§1) as a top-priority read.

---

## Related open-access read (not one of the four)

**Amaruchkul (2024) — two-stage stochastic program, full text free.**
Amaruchkul, K. (2024). Dynamic Network for Air Freight Forwarder's Stochastic
Capacity Management. *Science & Technology Asia*, 29(3), 36–56.
NIDA/NRCT open-access PDF: https://doi.nrct.go.th/admin/doc/doc_657187.pdf
Saved as `papers/Amaruchkul-2024-Dynamic-Network-Air-Freight-Forwarder-Stochastic-Capacity-SciTechAsia.pdf`.

**Abstract [verbatim].** "A freight forwarder, a key player in the air cargo service
chain, collects individual packages from shippers and transports consolidated
shipments to air carriers, some of which have long-term block space agreements with
the forwarder. If, on any day, the total demand from the consolidated shipment
exceeds the allotment specified in the block space agreement, the forwarder may need
to purchase additional space in the spot market, where the freight rate is often
higher. Alternatively, the forwarder may opt to delay some shipments, storing them
overnight at a warehouse and incurring inventory holding costs. On the other hand, if
the total demand is less than the allotment, the forwarder is required to pay at least
the minimum charge. For each destination and each day of the week, demand exhibits
significant week-to-week variation, while the capacity supply on each day of the week
remains fixed over the contract duration. In the long-term problem, the forwarder
must decide on the allotment before knowing the random daily demand. In the
short-term problem, it determines how to allocate the realized demand to multiple
carriers with different freight rates. The problem is formulated as a two-stage
stochastic program, embedding the multi-day network of time-varying demand. The
proposed solution is compared to the current approach in a case study, utilizing
historical demand data from one of Thailand's largest forwarders. Based on the top
four destinations from April 2020 to July 2021, our proposed solution yields
significant cost savings."

**Why it's useful.** The single best *free, full-text* statement of the two-stage
structure: stage 1 = pick the allotment before daily demand is known; stage 2 =
allocate realized demand across carriers. Same author as #2/#3/#4; likely the
working/companion version of the 2025 ITOR paper (#4), which adds robust flight-time
uncertainty on top.

**Scope caveat (read the top banner).** This is a **BSA procurement/sizing** model —
the decision variable is *how much block space to buy*. That is **not our problem**:
our air planner takes BSA as already procured and routes in real time against it.
Only this paper's **fixed-`x` second stage** (`Q(x, ξ)`: allocate realized demand
across BSA + non-BSA + overnight spill) overlaps our engine, and even that abstracts
away the arrival stream / booking-lead / commit-timing dynamics that are the core of
our problem. Its stage-1 machinery (newsvendor critical ratio `1 − r_B/r_S`, the
"reserve max" incumbent at +209% weight / +54% cost) is procurement-side and does
**not** transfer to routing.

---

## Cross-references

- `docs/academic-literature-references.md` — broader forwarder consolidation/routing
  canon (contains #4 above; **has the #2 author error noted at top**).
- `references/air-cargo-allotment-contracts.md` — deep model notes on #3 (Amaruchkul
  2018) and Gupta (2008); **also has the #2 author error**.
- `docs/design/air_cargo_reserve_assign_grounding_s52.md` — the S52 grounding doc
  these citations support.
