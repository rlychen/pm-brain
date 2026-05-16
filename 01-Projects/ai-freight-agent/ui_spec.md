# UI Specification

*Part of the AI Freight Routing PRD. See [PRD.md](PRD.md) for strategic overview and document map.*

**Sections:** Look and Feel · Screen Inventory · Persona Views · Agent Action Feed · Mobile Philosophy · Wireframes · Interaction Design · Agent Reasoning Transparency

---

## 1. Look and Feel

### 1.1 Design Language

**Hybrid dark sidebar / light content.** The sidebar navigation is dark (deep navy `#1e2a3a` or equivalent); the main content area is white/light gray (`#f8f9fa`). This is the pragmatic middle ground between the pure "dark ops center" aesthetic (compelling but harsh in dense tables) and full light-mode SaaS (clean but low contrast for operations work). Reference: Portcast uses this split effectively — dark Command Center, lighter analytics.

The product is an **ops tool for people who stare at it for 8 hours**. Legibility and information density beat visual drama.

### 1.2 Color System

| Token | Use | Value (approx) |
|---|---|---|
| `sidebar-bg` | Left navigation background | `#1e2a3a` (dark navy) |
| `sidebar-text` | Nav labels, icons | `#94a3b8` (slate, inactive) |
| `sidebar-active` | Active nav item | `#ffffff` + `#2563eb` accent left border |
| `content-bg` | Main content area | `#f8f9fa` |
| `card-bg` | Cards, panels, tables | `#ffffff` |
| `border` | Dividers | `#e2e8f0` |
| `text-primary` | Body text | `#0f172a` |
| `text-secondary` | Labels, metadata | `#64748b` |
| `status-green` | On track / committed / delivered | `#16a34a` |
| `status-amber` | At risk / Tier 2 / dry-run expiring | `#d97706` |
| `status-red` | Critical / Tier 3 / escalated | `#dc2626` |
| `status-gray` | Pending / unrouted / pre-departure | `#94a3b8` |
| `status-blue` | In transit (ocean) | `#2563eb` |
| `agent-purple` | Agent-originated actions in audit trail | `#7c3aed` |
| `tier1-green` | Tier 1 auto-execute indicator | `#16a34a` |
| `tier2-amber` | Tier 2 recommend indicator | `#d97706` |
| `tier3-red` | Tier 3 escalate indicator | `#dc2626` |

### 1.3 Typography

- **Font:** Inter (web) — clean, legible at small sizes, excellent for data tables
- **Scale:** 12px (table cells, metadata), 14px (body, labels), 16px (section headers), 20px (page titles), 24px (KPI numbers)
- **Weight:** 400 regular (body), 500 medium (labels, table headers), 600 semibold (KPIs, critical values), 700 bold (page titles only)

### 1.4 Component Style

- **Tables:** High-density, thin row separators (`#e2e8f0`), sticky header, hover highlight (`#f1f5f9`)
- **Cards:** White with `1px` border, `4px` border-radius, subtle `box-shadow`
- **Badges/pills:** Rounded pill shape for status, tier, mode indicators. Small and inline.
- **Buttons:** Primary action = `#2563eb` (blue), destructive = `#dc2626`, neutral = `#f1f5f9`
- **Exception queue items:** Left border `4px` colored by severity (red Tier 3, amber Tier 2)
- **Agent action items in feed:** Left border `4px` `#7c3aed` (purple/indigo) to distinguish from carrier events and user actions
- **Autonomy mode pill:** Always visible in UI header — "Co-pilot" (blue), "Supervised" (amber), "Autonomous" (green)

---

## 2. Screen Inventory

A complete freight ops product requires the following screens. Grouped by area.

### Core Operations Workspace

| Screen | Primary user | Description |
|---|---|---|
| **Operations Dashboard** | Ops Planner | Landing page: KPI strip + exception queue preview + agent status + allocation health |
| **Exception Queue** | Ops Planner | All items requiring human decision, severity-sorted. The primary daily workflow surface. |
| **Routing Activity Log** | Ops Planner, Analyst | Full log of agent decisions for a batch or date range; groupable by carrier/shipper/receiver |
| **Shipment List** | Ops Planner, Analyst | Full filterable/sortable table of all shipments with saved views; map view toggle |
| **Shipment Detail** | Ops Planner, Analyst | Single shipment: route, agent reasoning trace (3 levels), alternatives, override |
| **Map View** | Ops Planner | Geographic overview of active shipments, clustered by risk. Toggleable from Shipment List. |

### Booking and Rate Screens (Phase 2+)

| Screen | Primary user | Description |
|---|---|---|
| **Rate Workspace** | Ops Planner | Carrier rates (contract + spot) searchable by lane, mode, date — read-only in MVP |
| **Quote Builder** | Ops Planner | Build a customer-facing quote with routing options and margin |
| **Booking Confirmation** | Ops Planner | Review and confirm a route after human approval; generates pre-alert |

### Operations Management

| Screen | Primary user | Description |
|---|---|---|
| **Allocation Monitor** | Ops Planner | Carrier string utilization across all strings and periods with alerts at thresholds |
| **Document Management** | Ops Planner, Compliance | Uploaded docs per shipment, required vs. received checklist, version history |
| **Milestone / Event Log** | All | Full audit trail per shipment — carrier events vs. user actions vs. agent actions |

### Agent / AI Screens

| Screen | Primary user | Description |
|---|---|---|
| **Agent Dashboard** | Ops Planner | Agent throughput, escalation rate, current run status, tier distribution over time |
| **Agent Action Feed** | Ops Planner | Real-time feed of agent actions: what the agent processed, escalated, and why |
| **Policy & Guardrails Editor** | Admin | Configure autonomy mode, confidence threshold, carrier priorities, trigger schedule |

### Analytics and Reporting

| Screen | Primary user | Description |
|---|---|---|
| **Performance Dashboard** | Analyst, Management | OTD rate, carrier scorecards, lane performance, exception rate trend, savings attribution |
| **Lane Analytics** | Analyst | Volume by lane, rate trend, transit time distribution, cost vs. benchmark |
| **Carrier Scorecard** | Analyst | Per-carrier: cost, OTD, rollover rate, allocation utilization vs. minimum commitment |
| **Custom Report Builder** | Analyst | Configurable report with export (CSV) and scheduled email delivery |

### Customer-Facing

| Screen | Primary user | Description |
|---|---|---|
| **Customer Portal (Shipper View)** | Shipper's logistics manager | White-label read-only view: shipment status, ETA, milestones, documents. No cost/margin visible. |

### Administration

| Screen | Primary user | Description |
|---|---|---|
| **User Management** | Admin | Roles, permissions, team hierarchy, invite new users |
| **Carrier Network** | Admin | Carrier integrations, preferred/acceptable/blacklisted status, allocation contracts |
| **Alert Configuration** | Admin | Exception rules, thresholds, notification routing (email / Slack / in-app) |
| **API & Integrations** | Admin | API key management, webhook endpoints, TMS integrations |

---

## 3. Persona Views and Separation

**The forwarder ops view and the shipper customer portal are two different products sharing a data model.** They are not filtered versions of the same screen.

| Dimension | Forwarder Ops View | Shipper Customer Portal |
|---|---|---|
| **Cost visibility** | Full (carrier cost, margin, spend analytics) | None |
| **Agent decisions** | Full (tier, confidence score, override options, reasoning trace) | None |
| **Carrier selection** | Visible (which carrier, why, allocation impact) | Not shown |
| **Shipment columns** | MBL/HBL, mode, carrier, document status, ETA, confidence, tier, exception flag | Reference #, status, ETA, last event |
| **Actions available** | Override, approve, escalate, cancel, export | None (read-only) |
| **Notifications** | Exception alerts, dry-run expiry, trust graduation, allocation warnings | ETA change, delay alert, delivery confirmation |
| **Navigation** | Full sidebar (all 20 screens) | Minimal — shipment list, detail, documents |

**Forwarder analyst view** is the forwarder ops view with exception queue and override actions hidden. Analytics, reporting, and carrier scorecard screens are the primary surfaces.

---

## 4. Agent Action Feed

The agent action feed is the **trust interface** — the mechanism by which human operators build confidence in autonomous operation over time. It is distinct from the exception queue (which shows only escalations) — it shows the routine work the agent handled autonomously.

### 4.1 Design

The feed appears as a persistent panel on the right side of the Operations Dashboard, or as a dedicated tab in the left nav ("Agent Feed"). It is always visible during active routing runs.

Each entry in the feed:
- Left border: `4px` `#7c3aed` (purple/indigo) — distinguishes agent actions from carrier and user events
- Icon: robot/agent icon (not a checkmark — checkmarks are for human-confirmed actions)
- Timestamp: relative ("2 min ago") during active run, absolute time in history view
- Content: one-line summary + expandable detail

### 4.2 Entry Format

```
●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│ Agent  06:14:02   Routed SHV-008 → MSC Tiger May 18
│        SHA→USLAX  $3,200 total  18d  91% on-time  [Tier 1]
│        Confidence: 0.84  Risk: Low  [Expand reasoning ↓]
●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│ Agent  06:12:15   ⚡ Escalated SHV-001 → Exception Queue
│        CYC margin 0.4d below threshold (0.5d)  [Tier 3]
│        Alternatives computed: 3 options ready for review
●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│ Agent  06:10:00   Batch started — 31 shipments
│        TPEB (24) + FEWB (7), trigger: scheduled 06:00
│        Graph build: 0.8s  MILP decomposed: 3 components
●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 4.3 What the Feed Shows

- Batch start and completion with shipment counts
- Each Tier 1 auto-execute with route summary, cost, confidence, risk level
- Each Tier 2 recommendation surfaced (link to exception queue)
- Each Tier 3 escalation (link to exception queue)
- Re-plan triggers fired (which shipments, which disruption event)
- Market Intelligence alerts (port congestion, spot rate spike)
- Compliance check results (PASS per batch, FAIL details per shipment)
- Dry-run expiry events (Tier 1 routes auto-committed)

### 4.4 Autonomy Mode Indicator

The autonomy mode is a persistent visible pill badge in the top navigation, not buried in settings:

```
[AI FREIGHT ROUTING]  ●  Supervised  |  TPEB: Eligible ↑  FEWB: Building (12/30d)
```

The badge changes color by mode:
- Co-pilot: blue (`#2563eb`)
- Supervised: amber (`#d97706`)
- Autonomous: green (`#16a34a`)

Trust status per lane is visible directly in the header so operators always know where the agent stands.

---

## 5. Mobile Design Philosophy

**Mobile is a push-notification surface and quick-action surface — not a full ops interface.**

No current freight SaaS product treats mobile as a full operations interface. Operators run the full workspace on desktop. Mobile is for:
- Receiving exception alerts (Tier 3 escalations, dry-run expiry warnings)
- Reviewing and approving a single escalated exception (approve / reject / escalate to manager)
- Checking portfolio risk status at a glance
- Viewing a single shipment's current status and ETA

**Mobile is not for:**
- Running routing batches
- Viewing full shipment lists with all columns
- Accessing analytics and reporting
- Configuring routing policy

**Implementation:** React Native with Expo (see `build_plan.md` §1.3). Expo Router mirrors Next.js file-based routing conventions. Mobile app launches to a notification-driven triage view. Full workspace is always a desktop action.

---

## 6. Forwarder Operations Planning UI — Wireframes

*AI-Native operations interface. Built on the autonomy model and guardrails framework defined in `agent_design.md`.*

### 6.1 UI Architecture

Six primary views. The operator's default landing page is the **Operations Dashboard**.

| View | Primary purpose |
|---|---|
| Operations Dashboard | Live status: agent health, today's routing summary, exception queue preview, allocation health |
| Exception Queue | The items requiring human decision — infeasibles, threshold violations, escalations |
| Routing Activity | Full log of agent decisions for a batch or date range; filterable and groupable |
| Policy & Guardrails | Configure routing objectives, carrier priorities, threshold guardrails, trigger schedule |
| Shipment Explorer | Browse all shipments (any status); view agent reasoning per shipment; override |
| Allocation Monitor | Carrier string utilization across all strings and periods |

Navigation is persistent left sidebar. Exception Queue badge shows pending count in real time.

---

### 6.2 Screen Designs

#### 6.2.1 Operations Dashboard

The daily starting point. One screen shows everything the operator needs to decide: are things running, what needs attention, are allocations healthy?

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║  AI FREIGHT ROUTING                      [Tue 13 May 2026]   ⚠ 3 exceptions        ║
╠═══════════════╦══════════════════════════════════════╦══════════════════════════════╣
║ TODAY         ║ AGENT STATUS                         ║ ALLOCATION HEALTH            ║
║               ║                                      ║                              ║
║  47  Routed   ║ ● Ocean Planner       running        ║ MSC Tiger (TPEB)             ║
║   3  Exceptions║● Compliance Validator running       ║ ████████░░  78%  88 TEU rem  ║
║  12  In-flight ║● Execution Monitor   watching 12   ║                              ║
║   0  Overrides ║● Market Intelligence last: 2h ago  ║ CMA CGM AEX-1 (TPEB)        ║
║               ║                                      ║ ████░░░░░░  40% 240 TEU rem ║
║ $186k spend   ║ LAST BATCH  ·  06:00 this morning    ║                              ║
║ 16.8d avg     ║ 31 shipments submitted               ║ ONE NE5 (TPEB)               ║
║ 89%  on-time  ║ 28 routed and committed              ║ ██░░░░░░░░  20% 320 TEU rem ║
║               ║  3 escalated to exception queue      ║                              ║
║               ║ [View batch →]  [Trigger run now]    ║ MSC Silk (FEWB)              ║
║               ║                                      ║ ██████░░░░  60% 160 TEU rem ║
╠═══════════════╩══════════════════════════════════════╩══════════════════════════════╣
║ EXCEPTION QUEUE — 3 items requiring decision                     [View all →]       ║
║                                                                                      ║
║ ⚡ SHV-001   CYC risk — margin 0.4d (threshold 0.5d)   SHA→USLAX    [Review →]     ║
║ ✕  SHV-009   No feasible path — Jun 2 deadline, Jun 5 earliest   [Review →]        ║
║ ✕  SHV-021   No feasible path — Jun 1 deadline, Jun 4 earliest   [Review →]        ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║ ROUTING ACTIVITY — last 2 hours                                   [Live ●] [All →] ║
║                                                                                      ║
║ 06:14  ✓ Routed SHV-008 → MSC Tiger May 18  $3,200  18d  balanced policy           ║
║ 06:14  ✓ Routed SHV-007 → MSC Tiger May 18  $3,200  18d  balanced policy           ║
║ 06:13  ✓ Routed SHV-006 → CMA AEX-1 May 21  $3,400  21d  carrier preferred         ║
║ 06:12  ⚡ SHV-001 escalated — CYC margin 0.4d below 0.5d threshold                ║
║ 06:11  ✓ Routed SHV-005 → ONE NE5 May 24  $3,100  22d  allocation-aware routing    ║
║ 06:10  Batch started — 31 shipments, TPEB+FEWB, trigger: scheduled 06:00           ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

#### 6.2.2 Exception Queue — Item Detail

Each exception is a structured decision with context and options. The agent provides the reasoning; the operator chooses from pre-computed alternatives (cheapest / fastest / most reliable) plus any override options.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║ EXCEPTIONS  [1 of 3]                            [← Prev]  [Skip]  [Next →]         ║
╠═════════════════════════════════════╦════════════════════════════════════════════════╣
║ ⚡ CYC RISK — SHV-001               ║ SHIPMENT DETAIL                               ║
║                                     ║ SHA → USLAX (TPEB)                            ║
║ Assigned sailing:  MSC Tiger May 18 ║ Client: Acme Corp                             ║
║ CY cutoff:         May 14  06:00    ║ 32 CBM  /  5,000 kg  /  2 FEU                ║
║ Cargo ready:       May 14           ║ Ready: May 14   Deadline: Jun 3               ║
║ Pre-carriage est:  1.2 days (mean)  ║ Service level: Standard (≥90% on-time req)    ║
║ CYC margin:        0.4 days  ✕      ║ Budget cap: none                              ║
║                    threshold: 0.5d  ║                                               ║
║                                     ╠═══════════════════════════════════════════════╣
║ AGENT REASONING                     ║ ALTERNATIVES                                  ║
║ "MSC Tiger May 18 is optimal under  ║                                               ║
║ balanced policy. CYC margin is 0.4d ║ A  MSC Tiger May 18    $3,200  18d  87%      ║
║ (0.1d below threshold). Escalating  ║    ⚡ CYC margin 0.4d — risk accepted         ║
║ per guardrail. Next sailing (CMA    ║                                               ║
║ AEX-1 May 21) has 2.8d margin and  ║ B  CMA AEX-1 May 21    $3,600  21d  92%  ●   ║
║ meets Standard service level        ║    ✓ CYC margin 2.8d  ✓ 92% ≥ 90% req        ║
║ (92% ≥ 90%). Cost delta: +$400.    ║    Agent recommendation                       ║
║ No budget cap constraint applies."  ║                                               ║
║                                     ║ C  ONE NE5 May 24      $3,100  22d  96%      ║
║ ROUTING GUIDE                       ║    ✓ CYC margin 5.2d  ✓ 96% ≥ 90% req        ║
║ CMA AEX-1 is preferred carrier #2  ║    Within deadline (Jun 3)                    ║
║ for TPEB. MSC Tiger is preferred #1 ║                                               ║
║ but has CYC risk on this shipment.  ║ [Confirm B — CMA AEX-1] [Choose A anyway]    ║
║                                     ║ [Choose C]  [Escalate to manager]            ║
╚═════════════════════════════════════╩════════════════════════════════════════════════╝
```

*Keyboard: `b` = Confirm B, `a` = Choose A, `c` = Choose C, `→` = Next exception*

---

#### 6.2.3 Routing Activity — Flexible Grouping

The full log of what the agent did in a batch. Groupable by carrier, shipper, or receiver. This is the audit surface — not a task list, a ledger.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║ ROUTING ACTIVITY — Batch 2026-05-13 06:00                    28 routed  3 escalated ║
║ Group by: [Carrier ●] [Shipper ○] [Receiver ○]     Status: [All ▼]   [Export CSV]  ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                      ║
║ ▼ MSC TIGER TPEB  ·  ETD May 18  ·  SHA/NGB→USLAX/USLGB      11 ships  64 TEU     ║
║   Allocation consumed: 64 TEU  |  Remaining after batch: 24 TEU  (MSC Tiger May)   ║
║   ─────────────────────────────────────────────────────────────────────────────     ║
║   SHV-007  Acme Corp     SHA→USLAX  32 CBM  2 FEU  $3,200  18d  91%  ✓ committed   ║
║   SHV-008  GlobalTech    SHA→USLAX  48 CBM  2 FEU  $3,200  18d  91%  ✓ committed   ║
║   SHV-012  Acme Corp     NGB→USLGB  15 CBM  1 TEU  $1,800  18d  89%  ✓ committed   ║
║   SHV-001  Acme Corp     SHA→USLAX  32 CBM  2 FEU     —    —    —    ⚡ escalated   ║
║   ...  (7 more)                                                [Expand all]         ║
║                                                                                      ║
║ ▶ CMA CGM AEX-1  ·  ETD May 21  ·  SHA/SZX→USLAX              9 ships  52 TEU     ║
║                                                                                      ║
║ ▶ ONE NE5  ·  ETD May 24  ·  SHA/NGB→USSEA                     8 ships  44 TEU     ║
║                                                                                      ║
║ ✕ INFEASIBLE / ESCALATED  (3 shipments — no agent action taken)                     ║
║   SHV-001  Acme Corp     ⚡ CYC risk — pending human decision                       ║
║   SHV-009  GlobalTech    ✕ No feasible path — Jun 5 earliest vs Jun 2 deadline     ║
║   SHV-021  Acme Corp     ✕ No feasible path — Jun 4 earliest vs Jun 1 deadline     ║
║                                                          [Review all exceptions →]  ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

*Switching to Group by Shipper:*

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║ Group by: [Carrier ○] [Shipper ●] [Receiver ○]                                     ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                      ║
║ ▼ ACME CORP  —  14 shipments  ·  $47,800 total  ·  3 on MSC Tiger, 8 on CMA, 3 exc ║
║   SHV-007  SHA→USLAX  MSC Tiger May 18  $3,200  18d  91%  ✓ committed              ║
║   SHV-012  NGB→USLGB  MSC Tiger May 18  $1,800  18d  89%  ✓ committed              ║
║   SHV-001  SHA→USLAX  —                 —       —    —    ⚡ CYC risk (pending)     ║
║   SHV-021  SHA→USLAX  —                 —       —    —    ✕ no feasible path        ║
║   ...                                                                               ║
║                                                                                      ║
║ ▶ GLOBALTECH  —  9 shipments  ·  $29,400 total  ·  1 exception                     ║
║ ▶ TECHWORLD   —  5 shipments  ·  $16,200 total  ·  0 exceptions                    ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

#### 6.2.4 Shipment Detail — Agent Reasoning + Override

Accessible from any view by clicking a shipment. Shows the full agent reasoning chain for any committed or pending decision.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║ SHV-007  ·  Acme Corp  ·  SHA → USLAX                    ✓ Committed  [Override]   ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║ ROUTE                             ║ AGENT DECISION TRACE                           ║
║                                   ║                                                ║
║ Pre-carriage:  SHA door → SHA     ║ 06:11:42  Ocean Planner                       ║
║   $240  ·  1.1d mean              ║   Batch: 2026-05-13-0600                      ║
║                                   ║   Input: v=32 CBM, w=5000 kg, n_k=1          ║
║ Ocean:  MSC Tiger May 18          ║   Policy: balanced (cost+reliability)         ║
║   SHA → USLAX                     ║                                               ║
║   $3,200  ·  18d mean  ·  91%     ║   Subgraph: 4 feasible sailings               ║
║   2 FEU (40'HC)                   ║   Mix: f*=2, t*=0 (cost $3,200)              ║
║                                   ║   Decomp component: TPEB-1 (12 commodities)  ║
║ Dwell:  USLAX  ·  3.5d mean       ║                                               ║
║                                   ║   MILP solve: 0.3s, 12 vars, OPTIMAL         ║
║ Inland:  USLAX → Phoenix AZ       ║   Assigned: MSC Tiger May 18                 ║
║   $480  ·  1.0d mean              ║   Objective: $3,920 total (balanced score)    ║
║                                   ║                                               ║
║ ────────────────                  ║ 06:11:43  Compliance Validator                ║
║ TOTAL  $3,920  ·  23d  ·  88%     ║   Carrier: MSC — allowed ✓                   ║
║                                   ║   Allocation: 2 TEU consumed, 86 rem ✓       ║
║ Deadline: Jun 3  ✓ (10d slack)    ║   CYC margin: 0.9d ✓                        ║
║                                   ║   Routing guide: MSC Tiger preferred #1 ✓    ║
║                                   ║   Result: PASS                               ║
║                                   ║                                               ║
║                                   ║ 06:11:44  Committed to dry-run state          ║
║                                   ║ 06:51:44  Auto-committed (window expired)     ║
╠═══════════════════════════════════╩════════════════════════════════════════════════╣
║ ALTERNATIVES  (pre-computed at solve time)                                          ║
║ A Cheapest     ONE NE5 May 24  $3,600 total  22d  96% on-time                     ║
║ B Fastest      CMA AEX-1 May 16  $4,100 total  20d  90% on-time                  ║
║ C Most reliable  ONE NE5 May 24  same as A on this shipment                       ║
║                                              [Override to A]  [Override to B]     ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

*If override is selected:*
- Reason field (required): `[________________________]`
- Reason is logged to `logs/overrides.jsonl` for constraint learning
- Override triggers a re-validation pass through Compliance Validator

---

#### 6.2.5 Policy & Guardrails Editor

The operator's primary lever for governing agent behavior. Changes take effect on the next routing run.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║ ROUTING POLICY & GUARDRAILS                        [Save]  [Discard]  [Audit log]  ║
╠══════════════════╦══════════════════════════════════════════════════════════════════╣
║ AUTONOMY MODE    ║ HARD STOPS  (non-configurable)                                  ║
║                  ║ These cannot be disabled. Violations always escalate.           ║
║ ○ Co-pilot       ║                                                                 ║
║   Agent prepares ║  ✓ Allocation cap exceeded → block + escalate                  ║
║   Human approves ║  ✓ No feasible path within deadline → escalate                 ║
║   all decisions  ║  ✓ Blacklisted carrier on route → block                        ║
║                  ║  ✓ Hazmat / OOG / reefer cargo → reject + escalate             ║
║ ● Supervised     ║  ✓ Validator fails after 3 cycles → escalate                   ║
║   Tier 1: auto   ║  ✓ Agent confidence < 0.50 → always escalate                   ║
║   Tier 2+: queue ║                                                                 ║
║                  ╠═════════════════════════════════════════════════════════════════╣
║ ○ Autonomous     ║ THRESHOLD GUARDRAILS  (configurable)                            ║
║   All tiers auto ║                                                                 ║
║   Tier 3 → queue ║  Confidence threshold (Tier 1 auto-execute): [0.80]  (0.70–0.95)║
║                  ║  CYC margin minimum:                          [0.5] days        ║
║ Trust status     ║  Cost ceiling (auto-escalate):                [$10,000]/shipment ║
║ TPEB  ● Eligible ║  Dry-run window (auto-commit):                [60] min          ║
║ FEWB  ○ Building ║  Urgency dry-run window:                      [15] min          ║
║       (12/30d)   ║  Min P(on-time): Economy [80]% Std [90]% Exp [95]%             ║
║ [Review trust →] ║  Max cost deviation from lane median:         [+30]%            ║
║                  ║  Allocation warn threshold (soft):            [70]%             ║
╠══════════════════╬═════════════════════════════════════════════════════════════════╣
║ ROUTING          ║ AUTONOMOUS ROUTING TRIGGERS                                     ║
║ OBJECTIVE        ║                                                                 ║
║                  ║  ● Run automatically   ○ Manual trigger only                   ║
║ Default:         ║                                                                 ║
║ ○ Cheapest       ║  Scheduled:  [Daily at 06:00 ▼]                                ║
║ ● Balanced       ║  Accumulate: [✓ Run when ≥ 10 unrouted on a lane]              ║
║ ○ Fastest        ║  Urgency:    [✓ Route if CYC within 48h — cannot disable]       ║
║ ○ Most reliable  ║                                                                 ║
║                  ║  [● All lanes active]                                           ║
║ Per-client:      ║  [Pause TPEB]  [Pause FEWB]  [Pause all agents]                ║
║ Acme  Fastest    ║                                                                 ║
║ GTech Cheapest   ╠═════════════════════════════════════════════════════════════════╣
║ [+ Add rule]     ║ CARRIER PRIORITIES                                              ║
║                  ║                                                                 ║
║                  ║  TPEB:  1. MSC Tiger ●  2. CMA AEX-1 ●  3. ONE NE5 ●          ║
║                  ║  FEWB:  1. MSC Silk ●   2. CMA FAL1 ●                          ║
║                  ║  [Edit rankings]                                                ║
╚══════════════════╩═════════════════════════════════════════════════════════════════╝
```

*Autonomy mode and trust graduation:* The system evaluates each active lane against the three trust graduation criteria (see `agent_design.md` §1.4) and surfaces eligibility in the "Trust status" panel. Operator must explicitly approve a mode upgrade — it is never automatic.

---

#### 6.2.6 Allocation Monitor

Persistent visibility into carrier string utilization. Operator can see exposure before it becomes a problem.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║ ALLOCATION MONITOR — May 2026                                     [← Apr]  [Jun →] ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║ STRING              ALLOC    USED    COMMITTED   REM    UTILIZATION                 ║
║ ─────────────────────────────────────────────────────────────────────────────────── ║
║ MSC Tiger  TPEB     400 TEU  312 TEU  64 TEU    24 TEU  ████████████████░  94%  ⚠  ║
║                              (booked) (dry-run)          ── warn threshold (70%) ── ║
║                                                                                      ║
║ CMA AEX-1  TPEB     400 TEU  160 TEU   0 TEU   240 TEU  ████░░░░░░░░░░░░   40%     ║
║                                                                                      ║
║ ONE NE5    TPEB     400 TEU   80 TEU  44 TEU   276 TEU  ███░░░░░░░░░░░░░   31%     ║
║                                                                                      ║
║ MSC Silk   FEWB     400 TEU  240 TEU   0 TEU   160 TEU  ██████░░░░░░░░░░   60%     ║
║                                                                                      ║
║ ─────────────────────────────────────────────────────────────────────────────────── ║
║ ⚠ MSC Tiger at 94% utilized. Remaining: 24 TEU. Next sailings: May 25, Jun 1.      ║
║   Recommend: route TPEB overflow to CMA AEX-1 (240 TEU remaining) until Jun reset. ║
║   [View agent recommendation]  [Update carrier priority for TPEB]                  ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 7. Interaction Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Primary landing page | Operations Dashboard | Operator should see health and exceptions in one glance, not a shipment list |
| Default group-by in Activity | By Carrier | Forwarders think in vessel bookings; carrier grouping maps to how bookings are submitted |
| Available group-by options | Carrier, Shipper, Receiver | Shipper view for per-client reporting; receiver view for consignee queries |
| Options shown per shipment | 3 (cheapest, fastest, most reliable) | Covers the main trade-off axes without overwhelming; auto-select is the agent's choice, not the operator's |
| Override requires reason | Yes | Reason feeds constraint learning; also creates an audit record if client queries later |
| Dry-run window | 60 min default | Long enough to review; short enough not to delay bookings. Urgency shipments: 15 min |
| Keyboard shortcuts in exception queue | Yes (`b`, `a`, `c`, `→`, `←`) | High-volume operators should not need a mouse to resolve exceptions |
| Kill switch | Global + per-lane | Global for emergencies; per-lane for disruption management without stopping everything |
| Agent reasoning visible by default | Collapsed (click to expand) | Operators don't need it for clean decisions; it's there when something looks wrong |
| Inline actions on exception cards | Yes | Approve/reject/escalate from the card without navigating to detail view |
| Audit trail agent action color | Purple/indigo | Distinguishable from carrier events (blue) and user actions (dark) at a glance |
| Autonomy mode | Persistent header pill, color-coded | Operators must always know what mode the system is in without going to settings |

---

## 8. Agent Reasoning Transparency — Design Requirements

The system must surface agent reasoning at three levels of depth:

**Level 1 — Activity Feed (one line):** What happened and why, in plain language.
> `06:14  ✓ Routed SHV-008 → MSC Tiger May 18  $3,200  18d  balanced policy`

**Level 2 — Exception Context (one paragraph):** What the agent considered, what rule triggered escalation, what alternatives exist.
> *"MSC Tiger May 18 is optimal under balanced policy. CYC margin is 0.4d (0.1d below threshold). Next sailing CMA AEX-1 May 21 has 2.8d margin and meets Standard service level at +$400."*

**Level 3 — Full Decision Trace (structured log):** Every step the agent took — input parameters, subgraph construction, MILP solve summary, validator checks, policy rules evaluated. Accessible from any shipment detail view.

**Implementation requirement:** All LangGraph state transitions and tool calls are captured in LangSmith. The UI must be able to retrieve and render the LangSmith trace for any shipment decision in the shipment detail view. This is not optional — it is the primary trust-building mechanism with operators.
