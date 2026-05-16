# Build Plan

*Part of the AI Freight Routing PRD. See [PRD.md](PRD.md) for strategic overview and document map.*

*Engineering reference: tech stack, architecture, data design, build sequence, and testing requirements.*

---

## 1. Tech Stack

### 1.1 Overview

| Layer | Technology | Rationale |
|---|---|---|
| Backend API | FastAPI (Python 3.12, async) | Agent executor is async-native; type-hinted; pairs well with LangGraph |
| Frontend | Next.js 14 (React, TypeScript) | Web-first B2B SaaS; App Router; SSE support for agent progress |
| Auth | Clerk | Native org/tenant model; JWT contains org_id (tenant_id) and org_role — no extra DB round-trip per request |
| Database | PostgreSQL 16 (primary) | Row-level security, jsonb for solver output, mature ecosystem |
| Time-series | TimescaleDB extension | AIS positions, shipment events, agent decisions — hypertables on PostgreSQL; not a separate service |
| Cache + broker | Redis (AWS ElastiCache) | Allocation state cache, Celery job broker, rate limit counters, sailing schedule cache |
| Task queue | Celery + Redis | MILP solver holds the GIL — must run in Celery workers, not the async event loop |
| Agent framework | LangGraph | Planner-validator pattern, LangGraph Postgres checkpointer for HITL state, model-agnostic |
| LLM | Claude (Anthropic SDK) | Wrapped in LangGraph; model-agnostic swap is straightforward |
| MILP solver | HiGHS (`highspy`) | Open-source, Python-native, competitive with commercial solvers on MILP |
| MCP framework | FastMCP | Exposes optimization components as MCP tools callable by agent |
| Observability | LangSmith (agent traces) + Sentry (errors) + Datadog (metrics/alarms) |
| Billing | Stripe metered | Per-routing-decision pricing; usage-based billing events |
| Infra | AWS ECS Fargate (containers) + RDS (Postgres) + ElastiCache (Redis) + S3 + ALB + CloudFront |
| Admin | Retool (not custom-built) connecting directly to Postgres |
| Mobile (deferred) | React Native + Expo | Push notifications + quick actions only; full ops surface stays web |

### 1.2 Backend Services

**FastAPI app (ECS Fargate)**
- Handles all HTTP routes
- Validates requests, sets `app.current_tenant_id` in Postgres session
- Returns 202 + `run_id` immediately for routing jobs; Celery handles the heavy work
- SSE endpoint for agent run progress: `/api/agent-runs/{run_id}/stream`

**Celery workers (ECS Fargate)**
- Priority queues: `routing.priority`, `routing.batch`, `replan`, `analytics`, `notifications`
- MILP solve + LangGraph orchestration run here, never in the async loop
- Worker autoscaling: scale on `routing.priority` and `routing.batch` queue depth

**Next.js frontend (ECS Fargate or CloudFront)**
- Dark sidebar / light content hybrid (see `ui_spec.md §1`)
- Polls `agent_run_steps` or subscribes to SSE for routing progress
- Clerk handles auth; JWT forwarded to FastAPI on every request

### 1.3 Infrastructure Topology

```
Internet → CloudFront (static assets)
         → ALB → FastAPI (ECS Fargate, 2+ tasks)
                → Next.js (ECS Fargate, 2+ tasks)

FastAPI → RDS PostgreSQL 16 (primary + read replica)
        → ElastiCache Redis (Celery broker + app cache)
        → S3 (exports, logs, audit artifacts)
        → LangSmith (agent trace push)
        → Sentry (error capture)

Celery workers (ECS Fargate, autoscaling) → RDS + Redis + HiGHS (in-process)
```

**Estimated cost at launch:** ~$360/month (2 ECS tasks per service, single-AZ RDS db.t4g.medium, small ElastiCache).

---

## 2. Multi-Tenancy and Data Architecture

### 2.1 Multi-Tenant Strategy: Shared Schema + Postgres Row-Level Security

Every table that contains tenant data carries a `tenant_id UUID NOT NULL` column. PostgreSQL Row-Level Security (RLS) enforces isolation as a backstop:

```sql
-- Applied to every tenant-scoped table
ALTER TABLE shipments ENABLE ROW LEVEL SECURITY;
ALTER TABLE shipments FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON shipments
    AS RESTRICTIVE
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**Defense in depth:** The application layer always filters by `tenant_id` in every query. RLS is a second enforcement layer — it catches application bugs, not the primary isolation mechanism.

**Session setup:** FastAPI middleware extracts `org_id` from the Clerk JWT and sets it on the Postgres connection before any query runs:

```python
# In FastAPI middleware, before each request
await conn.execute(
    "SET LOCAL app.current_tenant_id = $1", 
    str(org_id)
)
```

### 2.2 Auth and Identity (Clerk)

- Each forwarder is a Clerk **Organization**. `org.id` = `tenant_id` throughout.
- Users belong to one organization with a role: `ops_planner`, `analyst`, `admin`.
- Clerk JWT fields used: `org_id` → `tenant_id`, `org_role` → permission checks.
- No custom auth code. Clerk handles SSO, MFA, invite flows, and session management.

**Shipper portal:** Shippers are invited as users with a restricted role in the forwarder's Clerk org. They see only their own shipments (filtered by `shipper_id`). The shipper portal is a different product surface — not the forwarder ops dashboard (see `ui_spec.md §3`).

### 2.3 LangGraph Agent State (Postgres Checkpointer)

LangGraph state is persisted to Postgres between agent steps, enabling HITL pause/resume and run inspection. Namespace format: `f"{tenant_id}:{run_id}"` — ensures agent state is scoped to a tenant.

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

checkpointer = AsyncPostgresSaver.from_conn_string(DATABASE_URL)
graph = compiled_graph.with_config(
    configurable={"thread_id": f"{tenant_id}:{run_id}"}
)
```

### 2.4 Redis Use Cases

| Use case | Key pattern | TTL |
|---|---|---|
| Allocation state cache (`rem(s,t)`) | `alloc:{tenant_id}:{string}:{period}` | 5 min |
| Sailing schedule cache | `schedule:{tenant_id}:{hash}` | 1 hour |
| Rate cache | `rate:{tenant_id}:{lane}:{carrier}` | 30 min |
| Celery job broker | (Celery internal) | — |
| Rate limit counters | `ratelimit:{tenant_id}:{endpoint}` | 60 s |
| Agent run status | `agent_run:{run_id}:status` | 24 hours |

### 2.5 TimescaleDB Hypertables

The following tables are TimescaleDB hypertables (partitioned by time on the listed column):

| Table | Partition column | Chunk interval | Retention |
|---|---|---|---|
| `ais_positions` | `recorded_at` | 1 day | 90 days |
| `shipment_events` | `occurred_at` | 1 week | 2 years |
| `agent_decisions` | `decided_at` | 1 week | 3 years |
| `routing_run_steps` | `created_at` | 1 week | 1 year |

All other tables remain standard Postgres tables.

---

## 3. Customer and Tenant Entity Model

Full SQL schemas, entity relationships, user roles, and the shipment lifecycle state machine are in [`data_model.md §3`](data_model.md).

**Summary:**
- **Organization** — the tenant; one per freight forwarder
- **User** — belongs to one Organization; roles: `ops_planner`, `analyst`, `admin`
- **Shipper** — the forwarder's client; scoped to a tenant
- **Carrier** — global reference table (MSC, CMA CGM, etc.)
- **TenantCarrier** — forwarder-carrier relationship + contract metadata
- **CarrierAllocation** — BSA monthly allocation caps per string
- **Shipment** — one shipment request; lifecycle: `unrouted → dry_run → committed → in_transit → delivered`
- **Route** — optimizer output attached to a shipment; stores MILP solution, confidence score, tier, legs as JSONB
- **Booking** — confirmed booking record; created when route exits dry-run state

---

## 4. Key Database Tables

### 4.1 Agent Run Tracking

```sql
CREATE TABLE agent_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES organizations(id),
    run_type        TEXT NOT NULL,   -- batch_routing | single_routing | replan | analytics
    status          TEXT NOT NULL DEFAULT 'queued',
    -- queued | running | completed | failed | cancelled
    trigger_type    TEXT,            -- scheduled | accumulation | urgency | manual | disruption
    shipment_count  INT,
    routed_count    INT,
    escalated_count INT,
    infeasible_count INT,
    total_cost_usd  NUMERIC(12,2),
    celery_task_id  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);

CREATE TABLE agent_run_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES agent_runs(id),
    tenant_id       UUID NOT NULL,
    step_type       TEXT NOT NULL,   -- graph_generation | milp_solve | validation | commit | escalate
    shipment_id     UUID REFERENCES shipments(id),
    status          TEXT NOT NULL,   -- running | completed | failed
    detail          JSONB,           -- step-specific output
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);
```

### 4.2 Audit Log (Immutable)

```sql
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   UUID NOT NULL,
    actor_type  TEXT NOT NULL,  -- agent | user | system
    actor_id    TEXT NOT NULL,  -- user UUID or agent name
    action      TEXT NOT NULL,
    resource    TEXT NOT NULL,
    resource_id UUID,
    detail      JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- INSERT only. No UPDATE or DELETE for app_user role.
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
```

The audit log is partitioned by month and append-only. It is the authoritative record of every agent action and operator override.

### 4.3 Routing Policy Store

```sql
CREATE TABLE routing_policies (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES organizations(id) UNIQUE,
    autonomy_mode           TEXT NOT NULL DEFAULT 'co_pilot',
    -- co_pilot | supervised | autonomous
    default_objective       TEXT NOT NULL DEFAULT 'balanced',
    -- cheapest | balanced | fastest | most_reliable
    confidence_threshold    REAL NOT NULL DEFAULT 0.80,
    cyc_margin_days         REAL NOT NULL DEFAULT 0.5,
    cost_ceiling_usd        NUMERIC(10,2) DEFAULT 10000,
    dry_run_window_min      INT NOT NULL DEFAULT 60,
    urgency_dry_run_min     INT NOT NULL DEFAULT 15,
    max_cost_deviation_pct  REAL NOT NULL DEFAULT 0.30,
    min_otp_economy_pct     REAL NOT NULL DEFAULT 0.80,
    min_otp_standard_pct    REAL NOT NULL DEFAULT 0.90,
    min_otp_express_pct     REAL NOT NULL DEFAULT 0.95,
    alloc_warn_threshold_pct REAL NOT NULL DEFAULT 0.70,
    trigger_schedule        TEXT DEFAULT 'daily_0600',
    trigger_accumulate_n    INT DEFAULT 10,
    routing_paused          BOOLEAN NOT NULL DEFAULT FALSE,
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by              UUID REFERENCES users(id)
);
```

### 4.4 Feature Flags

```sql
CREATE TABLE feature_flags (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_key    TEXT NOT NULL,
    tenant_id   UUID REFERENCES organizations(id),  -- NULL = global default
    enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_key, tenant_id)
);
```

Flag lookup: tenant override takes precedence over global default. Migration path to LaunchDarkly when flag count grows.

---

## 5. Demand Generator

The demand generator produces synthetic but realistic shipment requests over time, enabling agent development and testing without waiting for real customer data.

### 5.1 Configuration Table

```sql
CREATE TABLE demand_generator_configs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES organizations(id) UNIQUE,
    enabled             BOOLEAN NOT NULL DEFAULT FALSE,
    batch_size_mean     INT NOT NULL DEFAULT 20,
    batch_size_sigma    INT NOT NULL DEFAULT 5,
    -- shipments per generation run
    lane_mix            JSONB NOT NULL,
    -- e.g. {"TPEB": 0.60, "FEWB": 0.30, "OTHER": 0.10}
    commodity_mix       JSONB NOT NULL,
    -- e.g. {"GENERAL": 0.70, "ELECTRONICS": 0.20, "HAZMAT": 0.10}
    service_level_mix   JSONB NOT NULL DEFAULT '{"economy": 0.4, "standard": 0.5, "express": 0.1}',
    lead_time_mean_days INT NOT NULL DEFAULT 30,
    lead_time_sigma_days INT NOT NULL DEFAULT 7,
    -- days from generation to required_delivery
    seasonality_profile JSONB,
    -- monthly multipliers on batch_size_mean, e.g. {"12": 1.3, "1": 0.8}
    auto_trigger_routing BOOLEAN NOT NULL DEFAULT TRUE,
    -- if TRUE, enqueues a routing batch after generation
    generation_schedule TEXT NOT NULL DEFAULT 'daily_0500',
    -- cron-style: daily_HHMM or specific cron expression
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5.2 Generation Logic

```python
# Celery beat task — runs per demand_generator_configs.generation_schedule
@celery.task
def generate_demand(tenant_id: str) -> dict:
    config = get_demand_config(tenant_id)
    rng = np.random.default_rng(seed=None)  # real randomness for generation

    batch_size = max(1, int(rng.normal(config.batch_size_mean, config.batch_size_sigma)))
    shipments = []

    for _ in range(batch_size):
        lane = rng.choice(list(config.lane_mix.keys()), p=list(config.lane_mix.values()))
        origin, destination = sample_od_pair(lane, rng)
        cargo_ready = date.today() + timedelta(days=rng.integers(1, 5))
        lead_days = max(10, int(rng.normal(config.lead_time_mean_days, config.lead_time_sigma_days)))
        required_delivery = cargo_ready + timedelta(days=lead_days)
        volume_cbm = float(rng.uniform(10, 80))
        weight_kg = float(rng.uniform(volume_cbm * 200, volume_cbm * 500))

        shipment = Shipment(
            tenant_id=tenant_id,
            origin=origin,
            destination=destination,
            cargo_ready_date=cargo_ready,
            required_delivery=required_delivery,
            volume_cbm=volume_cbm,
            weight_kg=weight_kg,
            service_level=sample_service_level(config.service_level_mix, rng),
            cargo_type=sample_commodity(config.commodity_mix, rng),
            status="unrouted",
        )
        shipments.append(shipment)

    insert_shipments(tenant_id, shipments)

    if config.auto_trigger_routing:
        route_batch.apply_async(
            args=[tenant_id],
            queue="routing.batch",
        )

    return {"tenant_id": tenant_id, "generated": len(shipments)}
```

### 5.3 Integration with Agent Loop

After generation, if `auto_trigger_routing` is set:
1. Celery enqueues `route_batch` on `routing.batch` queue
2. Routing worker creates an `agent_run` record (status=`queued`)
3. LangGraph orchestrates: graph generation → MILP solve → compliance validation → tier assignment → commit/escalate
4. Each step written to `agent_run_steps`
5. Final state written to `agent_runs` (status=`completed`)
6. Frontend polls or SSE subscription delivers live progress

The forwarder ops planner sees the results in the Operations Dashboard and Exception Queue when they log in next morning.

---

## 6. Peripheral Product Components

### 6.1 Onboarding Wizard

New forwarder accounts complete a 4-step wizard before their first routing run:

| Step | Content |
|---|---|
| 1 — Carriers | Add carrier relationships: select from carrier reference, enter BSA allocation per string per month |
| 2 — Shippers | Add client shippers: name, contact, default service level, carrier preferences |
| 3 — Lanes | Confirm active trade lanes; set default routing objective per lane |
| 4 — Policy | Set autonomy mode (start at Co-pilot), configure threshold guardrails |
| Sandbox run | Optional: generate 5 synthetic shipments and run the agent in Co-pilot mode to see output |

The wizard writes to `organizations`, `tenant_carriers`, `carrier_allocations`, `shippers`, and `routing_policies`.

### 6.2 Stripe Billing (Metered)

```python
# Emit a billing event when a routing decision is committed
async def record_billing_event(tenant_id: str, shipment_id: str) -> None:
    customer_id = await get_stripe_customer_id(tenant_id)
    await stripe.UsageRecord.create(
        subscription_item=get_subscription_item_id(customer_id),
        quantity=1,
        timestamp=int(time.time()),
        action="increment",
        idempotency_key=f"route-commit-{shipment_id}",
    )
```

- Billing trigger: route transitions from `dry_run` to `committed`
- Per-decision pricing. Invoice generated monthly by Stripe.
- Stripe Customer Portal for self-serve subscription management.

### 6.3 Notifications

| Channel | Trigger | Implementation |
|---|---|---|
| In-app (SSE) | Any agent action, exception created, run completed | Postgres LISTEN/NOTIFY → FastAPI SSE endpoint |
| Email | Exception created, run summary (daily digest) | AWS SES + Jinja2 templates |
| Slack webhook (optional) | Exception created, run summary | Forwarder configures webhook URL in Settings |

### 6.4 API Keys (for programmatic access)

```sql
CREATE TABLE api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES organizations(id),
    key_hash    TEXT NOT NULL UNIQUE,  -- bcrypt hash; plaintext shown only at creation
    label       TEXT NOT NULL,
    scopes      TEXT[] NOT NULL DEFAULT '{"routing:read"}',
    last_used_at TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by  UUID NOT NULL REFERENCES users(id)
);
```

API key auth: extract from `Authorization: Bearer sk_...` header → hash → lookup → verify tenant + scope.

### 6.5 Rate Limiting

Per-tenant, per-endpoint rate limits enforced in FastAPI middleware via Redis counters:
- Routing runs: max 10 per hour per tenant (prevent runaway demand generator)
- API keys: max 1,000 requests per hour per key
- Webhook deliveries: max 100 per hour per tenant

### 6.6 Retool Admin Panel

Retool connects directly to Postgres (read-only replica) for internal admin:
- View all tenants, subscription status, billing events
- Manually pause/unpause routing for a tenant
- Inspect agent runs, audit logs, override logs
- Backfill or override feature flags

No custom admin UI is built. Retool is sufficient for internal use until the product scales to require a custom-built admin surface.

### 6.7 S3 Exports

Operators can export:
- Routing activity log (CSV) for any date range
- Carrier allocation utilization (CSV) by month
- Shipment audit trail (JSON) for a specific shipment

Exports are generated as Celery tasks (async), uploaded to S3 with a signed URL, and delivered via email or in-app notification.

### 6.8 Monitoring Stack

| Tool | Purpose |
|---|---|
| LangSmith | Agent trace capture and retrieval; required for Level 3 agent reasoning display |
| Sentry | Error capture with tenant context; performance profiling for slow requests |
| Datadog | Infrastructure metrics (ECS CPU/memory, RDS connections, Redis queue depth, Celery task latency) |
| CloudWatch | AWS-native alarms on RDS, ElastiCache, ALB |

**Alarms that page on-call:**
- `routing.priority` queue depth > 50 for > 5 min
- Celery routing task P99 latency > 120 s
- RDS connection count > 80% of max
- Any 5xx error rate > 1% for 5 min

---

## 7. Agent Execution Architecture

### 7.1 Async Execution Pattern

Routing is not synchronous. The FastAPI endpoint returns immediately; the work runs in Celery.

```
POST /api/routing-runs
  ↓
FastAPI: validate request, create agent_run record (status=queued)
  ↓ returns 202 + {run_id}
  ↓
Celery task enqueued on routing.batch or routing.priority
  ↓
Celery worker picks up task:
  1. Set agent_run.status = "running"
  2. LangGraph orchestration:
     a. Graph Generator → build G(N, A) for this batch
     b. Ocean Optimizer → MILP solve per decomposed component
     c. Compliance Validator → allocation snapshot check
     d. Tier assignment → commit / recommend / escalate
  3. Write each step to agent_run_steps
  4. Update shipment.status per outcome
  5. Set agent_run.status = "completed" (or "failed")
  6. Emit Postgres NOTIFY for SSE subscribers
  7. Queue notification tasks (email/Slack if configured)
```

### 7.2 Progress States

Frontend displays live progress via SSE subscription to `/api/agent-runs/{run_id}/stream`:

```json
{"event": "step", "data": {"step": "graph_generation", "status": "running", "shipment_count": 31}}
{"event": "step", "data": {"step": "milp_solve", "status": "running", "component": 1, "of": 3}}
{"event": "step", "data": {"step": "milp_solve", "status": "running", "component": 2, "of": 3}}
{"event": "step", "data": {"step": "validation", "status": "running"}}
{"event": "complete", "data": {"routed": 28, "escalated": 3, "infeasible": 0, "cost_usd": 186000}}
```

### 7.3 Failure Handling

| Failure type | Behavior |
|---|---|
| MILP infeasible (single commodity) | Structured report written to `agent_run_steps`; commodity escalated to exception queue; rest of batch continues |
| Celery task timeout (> 10 min) | Task marked failed; `agent_run.status = "failed"`; operator notified; no partial commits |
| LangGraph node exception | Caught at the node boundary; error logged to `agent_run_steps`; run fails cleanly |
| Redis unavailable | Celery broker unavailable — routing runs queue to disk (Celery fallback); rate limit bypassed with warning |
| Database write failure | Transaction rolled back; run marked failed; no partial state |

### 7.4 LangGraph State Schema

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class RoutingState(TypedDict):
    tenant_id: str
    run_id: str
    shipment_ids: list[str]
    policy: dict                    # routing_policies row
    allocation_snapshot: dict       # rem(s,t) at snapshot time
    graph: dict                     # G(N, A) — serializable form
    milp_solutions: list[dict]      # per-component MILP output
    routes: list[dict]              # final route assignments
    escalations: list[dict]         # exceptions for queue
    infeasibles: list[dict]         # structured infeasibility reports
    messages: Annotated[list, add_messages]
```

---

## 8. Data Sources

All data sources are either real network topology or synthetic commercial parameters. Data provenance is documented at the point of use in code.

| Source | Type | Use | License |
|---|---|---|---|
| UN/LOCODE | Real topology | Port and location reference | Free |
| IATA codes | Real topology | Airport reference | Free |
| NOAA AIS (historical) | Real signal | Ocean vessel tracking, transit time training | Free |
| AIS live feed (TBD) | Real signal | Active shipment tracking | Paid (future) |
| Google Maps Routes API | Real signal | Road transit time estimation | Pay-per-use |
| OSRM | Real signal | Road routing (free alternative) | Free |
| OpenSky Network | Real signal | Air freight schedules (historical) | Free |
| Ocean carrier schedules | Real topology | Sailing schedule graph construction | Public / scraped |
| BTS Freight Analysis Framework | Real topology | Trucking lane structure | Free |
| Synthetic rates | Synthetic | Commercial rate parameters for optimization | N/A |
| DAT | — | **NOT licensed. Do not use without explicit license.** | — |
| USLAX / port authority data | Real signal | Terminal throughput, berth schedules, typical clearance windows by terminal | Free / public |
| BNSF / UP intermodal ramp data | Real topology | Inland ramp locations, service days, cutoff times | Public |
| Google Maps Distance Matrix | Real signal | Road distance and historical transit time between inland nodes | Pay-per-use |
| NOAA / NWS weather | Real signal | Weather disruption risk by port and lane | Free |
| Spire Maritime AIS (live) | Real signal | Global vessel position, ETA, port call data beyond US waters | ~$30–80K/year |
| Descartes carrier schedules | Real topology | Licensed ocean carrier schedule feed (replaces scraping at production scale) | ~$20–50K/year |
| Xeneta / Freightos FBX | Real signal | Ocean spot rate benchmarks for rate validation | Paid |

---

## 8.1 External Data Integration Architecture — What a Production TMS Must Connect To

This section maps the full integration surface of a freight forwarder TMS. Each category is a distinct integration domain with its own protocols, data standards, and update cadences. Our system intersects with several of these — understanding the full landscape is necessary to scope what we build vs. what we assume the TMS provides.

---

### 8.1.1 Ocean Carrier Integrations

The most complex and highest-volume integration category. A forwarder works with 5–30 ocean carriers. Each carrier has its own portal, API, and EDI connectivity, with varying degrees of standardization.

**What data flows:**

| Direction | Data | Protocol |
|---|---|---|
| Forwarder → Carrier | Booking request (commodity, weight, volume, port pair, sailing) | EDI IFTMIN or carrier portal API |
| Carrier → Forwarder | Booking confirmation (container number, seal, CY cutoff, vessel, voyage) | EDI IFTMBC or carrier API |
| Carrier → Forwarder | Bill of Lading draft, amendments, final release | EDI IFTMCS or web portal |
| Carrier → Forwarder | Container status milestones (gate-in, loaded, departed, arrived, gate-out) | EDI IFTSTA or carrier tracking API |
| Carrier → Forwarder | Vessel departure/arrival updates, delays, vessel substitution | EDI IFTSTA or carrier push API |
| Forwarder → Carrier | Verified Gross Mass (VGM) declaration | SOLAS mandate — separate EDI message |
| Carrier → Forwarder | Demurrage and detention charges | Carrier invoice EDI or portal |
| Carrier → Forwarder | Sailing schedule (full port rotation, ETD, ETA per port) | Carrier schedule API or INTTRA |

**EDI standards used:**

- **UN/EDIFACT** — older but still dominant for large carriers (MSC, CMA CGM, Hapag-Lloyd). Message types: IFTMIN (booking), IFTMBC (booking confirmation), IFTMCS (instructions), IFTSTA (status), COPARN (container release), COARRI (container discharge), CODECO (container gate-in/out).
- **DCSA / DCSA T&T API** — newer JSON REST standard pushed by the Digital Container Shipping Association. Maersk, MSC, CMA CGM, Hapag-Lloyd, ONE, Evergreen are members. Gradually replacing EDI for newer integrations.
- **INTTRA (E2open)** — a neutral booking and documentation network that sits between forwarders and carriers. Many forwarders use INTTRA instead of direct carrier APIs to avoid maintaining 20+ individual carrier connections. INTTRA handles the translation to each carrier's native format.

**Major carriers and their integration maturity:**

| Carrier | Direct API | INTTRA | EDI |
|---|---|---|---|
| Maersk | Strong (Maersk API) | Yes | Yes |
| MSC | Growing | Yes | Yes (primary) |
| CMA CGM | Good (CMA CGM API) | Yes | Yes |
| COSCO / OOCL | Moderate | Yes | Yes |
| Hapag-Lloyd | Good | Yes | Yes |
| ONE (Ocean Network Express) | Moderate | Yes | Yes |
| Evergreen | Basic | Yes | Yes |
| Yang Ming | Basic | Yes | Yes |

**GoFreight:** Integrates with 125+ carriers including MSC, Maersk, CMA CGM, COSCO, ONE, Hapag-Lloyd for tracking and booking data. Uses both direct carrier APIs and EDI for document exchange. Does not perform routing optimization against these feeds — it uses them for milestone tracking and booking execution only.

**Our system:** We need ocean carrier schedule data (sailing schedule, port rotations, ETD/ETA) to build G(N, A). At prototype scale, public schedules suffice. At production scale, this requires a licensed feed (Descartes, INTTRA) because carrier schedule scraping is unstable — carriers change schedules frequently and block scrapers. We do not need full carrier operational EDI (booking, B/L) — that remains in the forwarder's TMS. Our interface is: read schedules to build the graph, and eventually write routing decisions back to TMS to trigger the booking workflow.

---

### 8.1.2 Air Carrier and GSSA Integrations

Air freight has a different and less standardized integration landscape than ocean.

**What data flows:**

| Direction | Data | Protocol |
|---|---|---|
| Forwarder → Carrier/GSSA | Air Waybill (AWB) booking | IATA Cargo-XML / e-AWB |
| Carrier → Forwarder | AWB confirmation, flight booking | IATA Cargo-XML or CHAMP/IBS system |
| Carrier → Forwarder | Flight status, ETD, ETA | Airline departure control system (DCS) feed |
| Forwarder → Carrier | Dangerous goods declaration | IATA DGD XML |
| Carrier → Forwarder | Weight and balance data | Carrier load control system |

**Key standards:**

- **IATA Cargo-XML** — IATA's XML messaging standard for air cargo. Covers booking request (XFWB), booking response (XFZB), AWB data (XFBL), status updates (XFSU). Replacing the older Cargo-IMP (text-based, 1970s).
- **e-AWB** — electronic replacement for paper Air Waybill. Now mandatory for most IATA-member airlines on major trade lanes. GoFreight supports e-AWB via EDI.
- **IATA ONE Record** — next-generation data sharing standard. Single data model for a shipment shared across all parties (airline, forwarder, shipper, ground handler, customs). Being adopted gradually as of 2025–2026. Not yet production-critical.
- **CHAMP Cargo Systems / IBS Software Cargo** — the two dominant cargo management systems (CMS) used by airlines internally. A forwarder connecting to an airline may interface through CHAMP rather than directly.

**GSSA complexity:** Many airlines sell belly cargo capacity through General Sales and Service Agents rather than directly. A forwarder booking cargo on Emirates may book through an independent GSSA that uses a different system from Emirates' own cargo portal. Adds one integration hop.

**Integrator networks (DHL Express, FedEx, UPS):** These operate closed proprietary networks. DHL Express has a direct API for rate quotes, booking, and tracking. FedEx Ship Manager API. UPS Developer Kit. All three have well-documented public APIs — easier to integrate than commercial airlines.

---

### 8.1.3 AIS / Vessel Tracking

AIS (Automatic Identification System) — every vessel over 300 gross tonnage is legally required to transmit AIS signals continuously: vessel identity (MMSI, IMO number, call sign), position (lat/lon), speed, course, destination, and ETA.

**How AIS data reaches a TMS:**

Two reception methods:

**Terrestrial AIS:** Ground-based receivers at ports, coast guard stations, and private networks pick up VHF AIS signals from vessels within approximately 50 nautical miles of shore. Dense coverage in European, North American, and Southeast Asian port areas. No coverage in the open ocean.

**Satellite AIS (S-AIS):** Low Earth Orbit satellites with AIS receivers capture transmissions anywhere on the ocean. Essential for trans-Pacific and trans-Atlantic tracking. Key providers:
- **Spire Maritime** (~$30–80K/year): dense satellite constellation, sub-minute update frequency, global coverage, API delivery
- **Orbcomm / exactEarth:** earlier entrant, now merged into Orbcomm; strong coverage
- **MarineTraffic (Kpler):** consumer-facing but also B2B API; aggregate of terrestrial + some satellite
- **VesselFinder / VesselTracker:** similar aggregate services

**What NOAA provides:** Historical terrestrial AIS data for US waters only (within ~50nm of the US coastline). Useful for transit time model training on USLAX, USLGB, USNYC arrivals. Not useful for trans-Pacific leg tracking.

**What AIS does NOT tell you:**
- Container-level status (which containers are on which vessel is from carrier EDI, not AIS)
- Vessel delay cause (AIS shows speed/position, not whether a delay is weather vs. congestion vs. mechanical)
- ETA accuracy (AIS ETA is captain-entered, not computed — often inaccurate; computed ETAs from ML models are more reliable)

**Integration pattern in a TMS:** The TMS subscribes to an AIS data feed (websocket or polling API), maps MMSI/IMO numbers to scheduled voyages in the carrier schedule database, detects position deviations from expected route, and updates ETA predictions. This feeds the exception alerts and re-plan triggers.

**Our system:** AIS positions feed the Rolling Horizon Controller. When a vessel's computed ETA deviates materially from the scheduled ETA, the controller fires a re-plan assessment. NOAA historical data for the prototype; Spire Maritime (or equivalent) for production global coverage.

---

### 8.1.4 Port and Terminal Systems

Ports are not monolithic — a single major port (USLAX/Long Beach) involves the port authority, multiple marine terminals (APM, TRAPAC, TTI, Pier J), a rail terminal, and dozens of trucking companies. Each has separate data systems.

**Port Community Systems (PCS):** Many major ports operate a PCS — a neutral digital hub that connects all port stakeholders (shipping lines, terminals, forwarders, customs, trucking, port authority) and standardizes data exchange.

| Port | PCS |
|---|---|
| Rotterdam | Portbase |
| Antwerp | Nextpcs / NxtPort |
| Singapore | PORTNET (MPA) + TradeNet (customs) |
| Hamburg | Dakosy |
| Felixstowe (UK) | Community Port Portal |
| Australia (multiple ports) | PortConnect |
| Taiwan (Kaohsiung, Keelung) | Single Window (Customs Administration) |

**What port/terminal data is relevant:**

| Data | Source | Use |
|---|---|---|
| Container gate-in confirmation | Terminal gate system | Confirms cargo arrived before CY cutoff |
| Container loaded on vessel | Terminal bay plan / COARRI EDI | Triggers B/L release process |
| Vessel departed (ATD) | Port authority or terminal | Starts transit time clock |
| Vessel arrived (ATA) | Port authority or terminal | Updates ETA; triggers destination leg planning |
| Container gate-out at POD | Terminal gate system | Confirms container picked up; triggers drayage |
| Free time expiry alert | Terminal / carrier D&D system | Demurrage and detention clock |
| Terminal congestion status | Port authority / anecdotal | Adjusts CFS cutoff buffers and dwell time estimates |

**Our system:** We consume ATA/ATD from AIS (vessel-level) and carrier milestone EDI (container-level). For the prototype, carrier EDI gives us the container milestones we need without requiring direct terminal integration. Direct terminal/PCS integration is a production-scale requirement.

---

### 8.1.5 Customs and Government Systems

Every international shipment crosses at least two borders, each with a mandatory filing. A forwarder TMS must either natively file customs entries or integrate with a licensed customs broker software.

**US import customs:**

| Filing | System | What it is | Timing |
|---|---|---|---|
| AMS (Automated Manifest System) | CBP/ACE | Ocean manifest listing all containers and commodities on the vessel | 24 hours before loading at origin port |
| ISF (Importer Security Filing) | CBP/ACE | 10+2 importer security data (supplier, manufacturer, country of origin, HS code, buyer, consignee) | 24 hours before vessel loading |
| Entry filing (Formal or Informal) | CBP/ACE | Import declaration — HS classification, valuation, duty calculation | Before or within 15 days of arrival |
| C-TPAT / PGA flags | CBP + Partner Government Agencies | FDA, FWS, EPA flags trigger additional hold/exam | Derived from HS code at entry filing |

**US export:**

| Filing | System | What it is |
|---|---|---|
| EEI (Electronic Export Information) | AES / ACE | Export declaration for shipments >$2,500 or controlled items |

**Key other markets:**

| Market | System | Notes |
|---|---|---|
| EU | ICS2 (Import Control System 2, launched 2023–2024) | Entry Summary Declaration; phased rollout by mode |
| China | GACC via CIFER | Customs filing; increasingly required pre-arrival |
| Taiwan | Customs Administration Single Window | EDI filing via approved customs broker |
| Japan | NACCS (Nippon Automated Cargo and Port Consolidated System) | Tightly integrated; AFR (Air Freight Risk) pre-alert for air cargo |
| Singapore | TradeNet | National single window; mandatory for all imports/exports |
| Australia | ICS (Integrated Cargo System) | DIBP/ABF system; stricter biosecurity holds |

**HS code databases:** Every customs filing requires HS (Harmonized System) classification. Forwarders maintain HS code libraries for their customers' products. Misclassification triggers holds, penalties, or seizure. Modern TMS platforms (CargoWise, GoFreight) include HS code lookup and classification assistance — GoFreight explicitly markets AI-assisted HS code classification.

**Our system:** We use HS codes from the shipment demand model (they exist in the `shipments` table). Customs filing is not in our scope — it remains in the forwarder's TMS. However, HS code risk tiers affect transit time (inspection dwell at POD) and are inputs to the customs inspection model (Phase 1).

---

### 8.1.6 Inland Transportation / Road and Rail

**Road trucking:**

| Integration | System / API | What it provides |
|---|---|---|
| Drayage rate card | Trucking company rate API or manual input | Door-to-CY or POD-to-door rate |
| Transit time | Google Maps Routes API / OSRM | Road distance + travel time (with traffic) |
| Carrier safety scores | FMCSA (Federal Motor Carrier Safety Administration) | Carrier vetting; safety rating before tendering |
| ELD (Electronic Logging Device) | ELD provider APIs (Samsara, KeepTruckin/Motive) | Real-time driver hours of service, estimated arrival |
| Load boards | DAT, Truckstop | Spot truck availability (requires license — DAT NOT licensed for us) |

**Intermodal rail:**

| Integration | Source | Notes |
|---|---|---|
| Ramp locations and service days | BNSF / UP public data (BTS FAF) | Free; ramp addresses, intermodal service days |
| Intermodal transit times | BNSF Price and Transit API, UP transit calculator | Published schedules; API access varies |
| Ramp availability | Not publicly available in real-time | Best effort using published schedules |

**Our system:** Google Maps Routes API (or OSRM as free fallback) for road transit times. BTS FAF for lane structure. BNSF/UP public data for intermodal ramp locations and service days. These feed the Graph Generator's trucking arc construction.

---

### 8.1.7 Rate and Market Data Feeds

**Ocean spot rates:**

| Provider | What it is | Cost |
|---|---|---|
| Freightos Baltic Index (FBX) | Weekly spot rate index by trade lane (TPEB, FEWB, etc.) — 12 trade lanes | Free (public index) |
| Xeneta | Real-time contracted + spot rate platform; benchmark your rates against the market | Paid ($30K+/year) |
| Drewry WCI (World Container Index) | Weekly spot rate composite | Free (weekly update) |
| Freightos | Rate comparison and quoting platform; has API | Paid |

**Air rates:**

| Provider | What it is |
|---|---|
| IATA TACT Rates | Official air cargo rate tariffs (IATA TACT subscription) |
| Aircargonews / CLIVE Data Services | Air cargo market intelligence, rate trends |

**Our system:** We use synthetic rate distributions for the prototype (calibrated to FBX ranges). For production, a Xeneta or Freightos rate feed validates that our synthetic rates are market-calibrated. We do not need real-time rate execution for the optimizer — the MILP uses rates from the forwarder's contracted tariff already in the system.

---

### 8.1.8 Financial and Banking Integration

**Trade finance:**
- Letter of Credit (L/C) — the TMS tracks L/C terms (latest ship date, port restrictions, document requirements) because L/C compliance affects routing choices. A shipment with an L/C requiring a specific POD cannot be rerouted through a different port without risking document discrepancy.
- Trust Receipt financing — some forwarders offer short-term financing for duty payment.

**Duty payment:**
- ACH (US) or direct debit to customs authority for duty and tax payment.

**Accounting sync:**
- QuickBooks, Xero, Sage, NetSuite — two-way sync of invoice/payment data.
- Multi-currency FX rates (Open Exchange Rates or equivalent) for USD/NTD/EUR/CNY reconciliation.

**Our system:** Not directly in scope. Financial data (payment terms, L/C restrictions) may appear as routing constraints in the rules engine — e.g., L/C restrictions on POD should constrain arc selection.

---

### 8.1.9 Document Exchange Networks

Beyond bilateral carrier EDI, several neutral document networks route freight paperwork between parties.

| Network | What it does |
|---|---|
| **INTTRA (E2open)** | Neutral ocean booking and documentation hub. Forwarder sends one INTTRA booking; INTTRA routes to the correct carrier in their native format. Covers most major ocean carriers. The most important neutral document exchange for ocean forwarding. |
| **CargoSmart** | Container tracking, schedule, and document exchange. Part of the Global Shipping Business Network (GSBN) — Maersk, COSCO, Evergreen, ONE collaboration. |
| **GT Nexus (Infor Nexus)** | Supply chain platform connecting shippers, forwarders, and financial institutions. Used more by shippers than forwarders. |
| **Bolero / essDOCS** | Electronic bill of lading transfer networks. Enables paperless B/L negotiation and title transfer. Used for trade finance. |
| **CCN (Cargo Community Network) / ACCS** | Air cargo community systems at specific airports. Handles ground handling bookings, dangerous goods notifications, cargo release. |

---

### 8.1.10 Partner / Agent Network Integration

Freight forwarders operate through a global network of agents — local companies in each country that handle origin pickup, customs at destination, and last-mile delivery on behalf of the forwarder. These agents need access to the TMS to update shipment status and share documents.

**Integration methods:**
- EDI EDIFACT for large, sophisticated agents
- TMS portal access (role-based, limited to their assigned shipments)
- API webhooks for modern agents with their own systems
- Email with structured document templates (lowest common denominator)

**GoFreight:** Supports overseas agent EDI settlement and document exchange. Agents can be granted portal access with limited visibility.

**Our system:** Agent network integration is out of scope for MVP. Routing optimization decisions do not depend on agent data — agents execute the leg, they don't affect the routing choice.

---

### 8.1.11 Integration Summary — What We Build vs. What Stays in the TMS

| Integration category | Who owns it | Our dependency |
|---|---|---|
| Ocean carrier booking + B/L | TMS (forwarder's CargoWise / GoFreight) | Read: sailing schedules for graph. Write: routing decision triggers booking in TMS |
| Air carrier booking + AWB | TMS | Same as ocean |
| AIS / vessel tracking | Our AIS Adapter (§9) | Real-time ETA for Rolling Horizon Controller |
| Port / terminal milestones | TMS (via carrier EDI) | We read ETA updates from TMS or AIS |
| Customs filing | TMS or customs broker software | HS codes from shipment demand; dwell time model |
| Road transit time | Our Road Routing Adapter (§9) | Google Maps / OSRM API |
| Intermodal rail schedules | Our Graph Generator (§9) | BNSF/UP public data |
| Ocean carrier schedules | Our Graph Generator (§9) | Licensed feed (Descartes) at production; public at prototype |
| Rate data | TMS holds contracted rates; we read them | Rates are inputs to MILP objective function |
| Accounting / billing | TMS | Out of scope |
| Document generation | TMS | Out of scope |
| Agent network | TMS | Out of scope |

**CargoWise integration is the critical path item.** Most Tier 2 mid-market forwarders run on CargoWise. We must read shipment and rate data from CargoWise to auto-populate our optimizer inputs, and write routing decisions back to CargoWise to trigger the booking workflow. Without this, the forwarder runs two parallel systems — a non-starter. CargoWise partner program: 4–12 weeks enrollment approval + 6–9 months per-customer integration.

---

## 9. Components Inventory

Each component is independently buildable and testable. No stitching until each component passes isolation tests.

| Component | Description | Mode(s) |
|---|---|---|
| Graph Generator | Constructs G(N, A) from network data sources. Port nodes get terminal throughput, customs clearance windows, anchorage wait distributions; city/inland nodes get intermodal ramp locations (BNSF, UP), road distance matrices, and historical road transit time distributions. | Ocean + Trucking |
| Ocean Transit Time Model | ML model: distribution over transit time per ocean arc | Ocean |
| Trucking Transit Time Model | ML model: distribution over transit time per trucking arc | Trucking |
| Ocean Optimizer | MILP: Binary Multi-Commodity Network Flow (P.1–P.5 in `model/ocean_fcl_routing.tex`). Commodity-specific subgraphs, optimal FEU/TEU container mix, per-sailing vessel slot cap (P.2), monthly string allocation cap (P.3), decomposition before solve, structured infeasibility reports. | Ocean |
| Trucking Optimizer | MILP: selects optimal trucking route given demand and constraints | Trucking |
| Multimodal Stitching Layer | Assembles mode-specific solutions into a coherent end-to-end plan | Both |
| Rolling Horizon Controller | Monitors shipment state, fires re-plan triggers, manages G_coarse/G_fine resolution | Both |
| Rules Engine | Evaluates routing guide compliance, carrier restrictions, business rules | Both |
| AIS Tracking Adapter | Ingests AIS data, maps vessel positions to shipment milestones, produces ETA updates | Ocean |
| Road Routing Adapter | Calls Google Maps / OSRM for road transit time and distance | Trucking |
| MCP Server | Exposes all components as MCP tools callable by the agent layer (`src/server.py`) | Both |
| Planning Agent | LLM agent (LangGraph) that orchestrates tools to answer routing queries | Both |
| Validation Agent | Reviews planning decisions before surfacing to user; two deterministic functions | Both |
| Execution Monitor Agent | Watches active shipments, detects exceptions, triggers re-plans | Both |
| Agent Interaction Logger | Logs all agent queries and responses to `logs/agent_interactions.jsonl` | Both |
| Routing Policy Store | Persists routing objectives, carrier priorities, per-client rules, and threshold guardrails | Both |
| Dry-run State Store | Holds committed-but-not-booked routing decisions during dry-run window; auto-commits on expiry | Both |
| Override Log | Appends operator override events to `logs/overrides.jsonl`; primary training signal for constraint learning | Both |
| LangSmith Trace Retrieval | Fetches structured LangSmith trace for a given decision ID; required for Level 3 agent reasoning in Shipment Detail view | Both |
| Allocation Snapshot Service | Snapshots `rem(s,t)` at routing run start; held fixed throughout batch to prevent concurrent run races | Ocean |
| Routing Run Log | Batch-level metadata: trigger type, shipment counts, total cost, timestamp. Feeds dashboard and audit trail. | Both |
| Demand Generator | Celery beat task producing synthetic shipment requests per tenant config; auto-triggers routing (see §5) | Both |
| Instance Generator | Produces synthetic geographically realistic problem instances for solver testing and model validation. Uses UN/LOCODE, GeoNames, public sailing schedule data, and Haversine-based transit time estimation. Joint session required — do not implement independently. | Ocean + Trucking |
| LCL Consolidation Optimizer | MILP: given LCL shipments on the same lane, determine optimal grouping into shared containers and assignment to sailings. Combined bin-packing + routing. Distinct from FCL optimizer. Requires NVOCC consolidation schedules and LCL rate data. **(Deferred)** | Ocean (deferred) |
| Customs Inspection Model | P1. Commodity-specific dwell time model for POD nodes. Computes inspection probability from HS code risk tier, importer C-TPAT status, country of origin, consignee history. MVP uses fixed port-level dwell constants. **(P1)** | Ocean (P1) |

---

## 10. Build Sequence

Phases are gates. Each phase requires explicit approval before the next begins.

**Phase 0 — PRD** ← CURRENT
Define problem scope, commercial model, data sources, component inventory, agent capabilities, and open questions.

**Phase 1 — Formal Models (LaTeX)**
One model per component. Each approved individually before code starts for that component.
- `model/ocean_fcl_routing.tex` — Draft v2 (not yet approved)
- Remaining models: Trucking Optimizer, Graph Generator, Transit Time Models

**Phase 2 — Component Builds**
Order: Graph Generator → Transit Time Models → Mode Optimizers → Rules Engine → Adapters → Stitching Layer → Rolling Horizon Controller

Each component: build → isolation test → pass gate → next component.

**Phase 3 — MCP Server**
Expose all verified components as tools (`src/server.py`). FastMCP.

**Phase 4 — Agent Layer**
Planning Agent → Validation Agent → Execution Monitor. LangGraph orchestration. LangSmith observability.

**Phase 5 — Product Layer**
FastAPI backend, Next.js frontend, Clerk auth, Postgres RLS, Celery workers, Demand Generator, Stripe billing, onboarding wizard, notifications.

**Phase 6 — Integration and End-to-End Testing**
Full stack integration tests. Load testing on realistic batch sizes.

**Phase 7 — Iterate**
Add air mode, improve models, extend agent capabilities, mobile app.

---

## 11. Unit Testing Requirements (Per Component)

**Gate rule:** A component is not integration-ready until its isolation tests pass. Isolation test = component tested standalone against synthetic data, without calling any other component.

### What to test per component type

**Graph Generator**
- Correct subgraph construction from a known synthetic sailing schedule: verify arcs included, arcs excluded, node sets
- Infeasible commodity: deadline too tight → `A^k = ∅` returned with structured infeasibility report
- Reachability: no dangling arcs after reachability sweep
- Pre-carriage trim: POL with no feasible ocean arcs is removed from subgraph
- CYC compliance: arc excluded when `τ_k(i) > CYC_{ij}`
- Decomposition: TPEB + FEWB mixed batch produces two disconnected components in commodity-supply graph H

**Ocean Optimizer (MILP)**
- Trivial feasibility: one commodity, one feasible arc, solver returns that arc at expected cost
- Infeasibility handling: commodity with no feasible arc is rejected before MILP build; structured report returned
- Vessel cap binding (P.2): batch that exactly fills one sailing's slot cap routes overflow to next sailing
- String allocation binding (P.3): batch that exhausts `rem(s,t)` on a string is correctly blocked
- Budget cap (P.4): commodity with tight `B_k` forced to cheaper route even if slower
- Container mix correctness: verify `(f*, t*, cost, slots)` for specific (volume, weight, ρ) inputs against manually computed values
- Optimal value bound: solution cost on a small synthetic instance is within 5% of manually computed lower bound
- Decomposition independence: two independent subproblems produce same result whether solved jointly or separately

**Transit Time Models**
- Point estimate: known (origin, destination) → transit days within ±10% of published carrier schedule
- Std dev calibration: sigma ≈ CV × mean for configured CV fraction

**Rules Engine**
- Carrier restriction: blacklisted carrier is never selected
- Routing guide: preferred carrier is selected when feasible; fallback logic fires when not
- Allocation cap: carrier at 100% utilization is excluded from selection

**MCP Tools (FastMCP)**
- Schema validation: tool rejects malformed input with structured error (not exception)
- stdout cleanliness: no `print()` output on any tool call path (FastMCP protocol constraint)
- Correct tool output schema on all happy paths

### Test conventions
- All tests in `tests/components/`, one file per component matching `src/components/` structure
- Fixtures in `tests/conftest.py` for shared synthetic instances (small, deterministic, fixed seed)
- Never mock the MILP solver — tests must call HiGHS on real small instances
- Never mock the database or graph state — use in-memory or temp-file fixtures
- Synthetic test instances: ≤5 commodities, ≤10 sailings
- Test names: `test_{component}_{scenario}` — e.g., `test_ocean_optimizer_vessel_cap_binding`
- Every component must have at least one infeasibility test
- Performance tests are deferred — correctness is the gate, not solve time
