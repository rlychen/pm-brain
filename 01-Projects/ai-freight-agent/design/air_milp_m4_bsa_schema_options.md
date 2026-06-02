# Air MILP slice M4 — `per_uld_pivot` + BSA: schema options + worked numerics

**Status:** OPEN — schema decision NOT made. User to deep-dive next session before M4 build.
**Date drafted:** 2026-06-02 (Session 25).
**Context:** M1/M2/M3 of `src/components/air_milp.py` are done (routing skeleton + flat_rate
density-mixing + min_flat_breaks). M4 adds the third MAWB-eligible rate family,
`per_uld_pivot`, plus block-space-agreement (BSA) settlement. M4 is a bigger commitment
than M2/M3 because it introduces a **BSA contract entity** and a **ULD-type catalog** into
the data model, not just per-arc rate fields. This doc records the options + numeric
examples so the decision can be made cold next session.

Model source: `model/air_freight_routing.tex` (APPROVED) — constraints C.5, C.5-act, C.5b,
C.5c, C.13a/b; bucket cost dispatch in `sec:lin-bucket`; pivot linearization `sec:lin-pivot`.

---

## 0. First, the conceptual confusion to clear up (pivot vs allowance)

During the session the user's mental model was: *"BSA = pre-buy ULDs on fixed flights for a
contract term; everything up to pivot weight is free (sunk); pay only above pivot."*

That intuition is **economically right about a pre-paid block, but attaches the "free up to X"
idea to the wrong parameter.** The model uses TWO distinct levers that pull opposite ways:

### Pivot weight `π` — a per-ULD *minimum charge* (NOT a free allowance)
The minimum billable weight per ULD position. It makes you pay for empty space — the opposite
of "free." Lives in the **`per_flight`** settlement branch:

```
cost^MAWB = r_a · max(CW, π · Ση)
```
Claim 2 LD3 (pivot 1000 kg each → 2000 kg floor), load 1700 kg → pay `3 × 2000 = $6000`.
The 300 kg of empty space costs you. Nothing sunk, nothing free; `π` is a floor that stops you
under-paying when you under-fill a claimed ULD.

### Allowance `A_c` — the take-or-pay sunk threshold (THIS is "free up to X")
Lives in the **`equalized`** settlement branch:
```
over_c = max(0, chargeable(c) − A_c)
cost   = r_c · over_c
```
`A_c` = the block already committed / pre-paid → everything below it is sunk (zero marginal
cost), you pay `r_c` only on the overage. Move 1700+1800 = 3500 kg under the contract,
`A_c = 3000` → first 3000 free, `500 × $3 = $1500`. **This is exactly the user's mental model.**

### The conflation, tabulated

| User's phrase | What they meant | Model parameter | Branch |
|---|---|---|---|
| "pre-buy ULDs, free up to [threshold], pay above" | sunk take-or-pay block | **`A_c`** (allowance, kg) | `equalized` |
| "pivot weight" (used as the threshold label) | — | **`π`** (per-ULD minimum) | `per_flight` |

`A_c` gives free room *below* a threshold; `π` forces a minimum payment *per claimed ULD*.
They are different parameters in different settlement bases.

### The bridge (why the intuition still maps)
The "free up to the pivot weight of the block" intuition is recoverable — set the allowance in
kg to the block's pivot-capacity:
```
A_c  ≈  (pre-bought ULD positions over the period) × π
```
e.g. 4 LD3 pre-bought × 1000 kg pivot → `A_c = 4000 kg` free, overage above at `r_c`. The model
expresses the threshold as one kg number (`A_c`) and **pools it across all the contract's
flights** ("equalized") — because a pre-bought block is fungible across the lanes it covers, so
the free budget is shared, not per-flight. That pooling is the whole reason `equalized` exists.

### Why the model also keeps `per_flight`
`per_flight` is the *other* commercial reality: a contracted **rate** with a per-ULD pivot
minimum, **settled per flight, no sunk block** (money flows each shipment). A "soft BSA"/NAC-style
deal — you didn't pre-buy, you have a negotiated rate + minimum-weight-per-ULD. No free tier,
hence the `$6000` with nothing sunk. Forwarders sign both kinds, so the model carries both.

**Honesty caveat:** real BSAs come in soft/hard variants and sometimes layer pivot-and-over
rating *inside* a take-or-pay block. The model deliberately collapses that variety into these two
settlement bases. It's an abstraction, not the full tariff zoo.

### What `r` ($/kg) is
`r` is the contracted **per-kilogram price** under the BSA (the worked numbers use `$3/kg` as an
illustrative placeholder, not real data). `π`/`A_c` decide how many kilos you're billed for; `r`
turns those kilos into dollars. `r_a` = per_flight rate (multiplies `pivot`); `r_c` = equalized
**overage** rate (multiplies `over_c`).

---

## 1. The shared scenario (identical across all three schema options)

Forwarder holds a Cathay (CX) BSA covering **two parallel arcs out of HKG**:

| | arc a1 = HKG→LAX | arc a2 = HKG→ORD |
|---|---|---|
| offer id | `AIR:cx_hkglax` | `AIR:cx_hkgord` |
| allotment `N_{a,LD3}` | 2 LD3 positions | 2 LD3 positions |

ULD type **LD3**: `W_u = 1500 kg`, `V_u = 4.5 m³`. Pivot `π = 1000 kg/ULD`, BSA rate `r = $3/kg`.

### Shared mechanics (same in every option) — how `η` is determined
One GEN MAWB on a1 carries actual weight **1700 kg**, volume **3.0 m³** (ε = 0 for clean numbers):

- **C.5b-w** (enough ULDs by weight): `1700 ≤ 1500·η` → `η ≥ 2`
- **C.5b-v** (by volume): `3.0 ≤ 4.5·η` → `η ≥ 1` → weight binds → **η = 2 LD3**
  (NB: C.5b uses `w_k` ACTUAL weight, not `cw_k` — item 13-A fix in the tex.)
- **C.5** (allotment): `η = 2 ≤ N = 2` ✓
- **C.5-act** (LP tightening): `η ≤ N·z` — no phantom ULD claims on inactive MAWBs
- **C.5c** (per-offer cap, if specified): `Σ w_k x ≤ cap_a`
- **CW** (billing weight): `max(Wt=1700, Wv=167·3.0=501) = 1700`

### Settlement = `per_flight` → $6000
```
pivot_{a,g} = max(CW, π·Ση) = max(1700, 1000·2) = 2000 kg   (C.13b-1, C.13b-2)
cost^MAWB   = r_a · pivot    = 3 · 2000          = $6000
```
Purely per-arc. Linearized (`sec:lin-pivot`): `pivot ≥ CW`, `pivot ≥ π·Ση`, cost `= r_a·pivot`
(two-inequality max, tight because `r_a > 0`).

### Settlement = `equalized` → $1500
Allowance `A_c = 3000 kg` pre-paid (sunk), overage rate `r_c = $3/kg`. a1 MAWB CW 1700, a2 MAWB CW 1800:
```
cost^MAWB(a1) = cost^MAWB(a2) = 0                ← no per-MAWB cost (sec:lin-bucket equalized)
chargeable(c) = Σ CW over BOTH contract arcs = 1700 + 1800 = 3500   (C.13a)
over_c        = max(0, 3500 − 3000)          = 500
cost          = r_c · over_c                 = 3 · 500 = $1500
```
**The cost depends on a sum across arcs that belong to the same contract.** That cross-arc
grouping is the structurally new thing the schema must support; per_flight does not need it.

---

## 2. The three schema options

### Option 1 — Contract entity (faithful to the tex)
The contract is a first-class object; arcs point into it.

```python
UldType(code="LD3", w_kg=1500.0, v_cbm=4.5)

BsaContract(
    id="cx_bsa",
    settlement="equalized",                       # or "per_flight"
    arcs={"AIR:cx_hkglax", "AIR:cx_hkgord"},      # A_c^MAWB
    allotment={"AIR:cx_hkglax": {"LD3": 2},
               "AIR:cx_hkgord": {"LD3": 2}},      # N_{a,u}
    pivot=1000.0, r_a=3.0,                         # per_flight terms
    A_c=3000.0,  r_c=3.0,                          # equalized terms
)
```
Equalized C.13a falls out directly — the sum literally iterates `contract.arcs`:
```python
chargeable_c = sum(CW[(a, g)] for a in contract.arcs for g in groups_on(a))  # 1700 + 1800
over_c >= chargeable_c - contract.A_c                                         # ≥ 500
```
One `over_c` var per contract. No reconstruction. **Cost:** more types up front
(`UldType`, `BsaContract`). **Benefit:** clean single injection point for the future Layer-3
`capacity_manager.md` controller, which supplies `A_c` and `N_{a,u}` exogenously per solve.

### Option 2 — Flat per-arc fields + a thin contract side-table
Arcs carry a `contract_id` *tag*; the equalized scalars still need a small home (they are genuinely
contract-level, not per-arc).

```python
RateCatalog(
    per_uld_pivot={
        "AIR:cx_hkglax": PerUld(r=3.0, pivot=1000.0, settlement="equalized",
                                contract_id="cx_bsa", allotment={"LD3": 2}),
        "AIR:cx_hkgord": PerUld(r=3.0, pivot=1000.0, settlement="equalized",
                                contract_id="cx_bsa", allotment={"LD3": 2}),
    },
    uld_types={"LD3": UldType(w_kg=1500.0, v_cbm=4.5)},
    contracts={"cx_bsa": ContractTerms(A_c=3000.0, r_c=3.0)},  # ← can't fully flatten this
)
```
Equalized requires a **reconstruction step** at solve time — group arcs by tag, then sum:
```python
arcs_of = defaultdict(set)
for a, pu in rates.per_uld_pivot.items():
    if pu.settlement == "equalized":
        arcs_of[pu.contract_id].add(a)            # rebuild A_c^MAWB from tags
for cid, arcs in arcs_of.items():
    chargeable_c = sum(CW[(a, g)] for a in arcs for g in groups_on(a))   # 1700 + 1800
    over_c >= chargeable_c - rates.contracts[cid].A_c                     # ≥ 500
```
`per_flight` needs no side-table (everything is per-arc: `r`, `pivot`, `allotment`). The contract
concept becomes implicit (a tag + a side-dict) rather than explicit. Lighter types; the contract
re-materializes only for equalized.

### Option 3 — `per_flight` only this slice (smallest correct step)
No contract entity, no `A_c`, no `over_c`, no settlement enum.

```python
RateCatalog(
    per_uld_pivot={
        "AIR:cx_hkglax": PerUld(r=3.0, pivot=1000.0, allotment={"LD3": 2}),  # per_flight implied
    },
    uld_types={"LD3": UldType(w_kg=1500.0, v_cbm=4.5)},
)
```
Builds only the per_flight branch → the `$6000` example works. The `$1500` equalized example is
**not expressible** — `chargeable(c)`/`over_c`/cross-arc pooling all deferred to a follow-up M4b.
Postpones the only genuinely new structural piece (cross-arc aggregation) + the "where do
`A_c`/`r_c` live" question.

---

## 3. Comparison

| | per_flight ($6000) | equalized ($1500) | new types | cross-arc grouping |
|---|---|---|---|---|
| **Opt 1 contract entity** | ✓ via `contract.pivot/r_a` | ✓ native, iterate `contract.arcs` | `UldType`, `BsaContract` | explicit |
| **Opt 2 flat + tag** | ✓ all per-arc | ✓ but reconstruct from `contract_id` + side-table | `PerUld`, `UldType`, `ContractTerms` | implicit (rebuilt at solve) |
| **Opt 3 per_flight only** | ✓ | ✗ deferred to M4b | `PerUld`, `UldType` | none yet |

Cross-cutting: per the tex, both `A_c` and `N_{a,u}` are supplied **exogenously per solve** by the
Layer-3 consumed-weight accumulator (future `capacity_manager.md`). Opt 1 gives that controller one
clean injection point per contract; Opt 2 scatters the values across per-arc records + a side-table;
Opt 3 sidesteps it for now.

---

## 4. Drafter's recommendation (for the user to accept/reject)

**Opt 3 now → Opt 1 for M4b.** Rationale: get `per_uld_pivot` + per_flight + C.5/C.5b/C.5c correct
and tested first (smallest correct step, project's minimal-design default); add the contract entity
when equalized lands, by which point it earns its keep and gives the `capacity_manager` a clean seam.
Opt 2 tends to be the worst long run — you build the contract concept anyway, just implicitly and
harder to hand to the controller.

**Counter-consideration:** if equalized BSA is core to the MVP pitch (consolidation savings story),
splitting M4/M4b delays the most commercially-resonant number. If so, go straight to Opt 1.

---

## 5. Open question for next session
1. Pick a schema option (1 / 2 / 3) — or Opt 3-now-then-Opt-1.
2. If deferring equalized (Opt 3): confirm M4b is tracked, not dropped.
3. Source values: `π`, `r_a`/`r_c`, `A_c`, `N_{a,u}`, `W_u`/`V_u` are all `MARKET RESEARCH NEEDED`
   / synthetic placeholders today — flag which need real-ish calibration for the TPEB instance.
