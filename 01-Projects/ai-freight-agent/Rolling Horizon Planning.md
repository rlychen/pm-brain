---
tags:
  - ai-freight-agent
  - architecture
  - planning
last_synced: '2026-05-07'
---
# Rolling Horizon Planning: Progressive Graph Resolution

*Key architectural concept that differentiates this system from all existing TMS platforms.*

See full PRD: [[PRD]] Section 5

---

## Core Idea

The system maintains a **complete end-to-end plan at all times**. Future legs are resolved at coarse resolution (cost/time envelopes); as a shipment advances and uncertainty decreases, each leg is re-solved on a fine graph with actual carrier schedules, port clearance estimates, and live spot rates.

This is **Model Predictive Control applied to multimodal freight routing**.

---

## Two Graph Resolutions

| Graph | When Used | Arc Weights |
|---|---|---|
| **G_coarse** | Future legs (high uncertainty) | Cost/time envelopes from historical data and ML models. Sufficient for optimization objective but not execution. |
| **G_fine** | Next leg approaching execution | Actual carrier schedules, confirmed spot rates, port-specific clearance estimates. |

---

## Concrete Example: Shenzhen → Phoenix

### Physical Journey
Shenzhen Factory → CNSGH (Shanghai Port) → Ocean Transit (~20 days) → USLAX (Port of LA) → Phoenix Warehouse

### T=0 (Booking)
- **Ocean leg (CNSGH → USLAX):** Precisely committed
  - Carrier: MSC | Vessel: MSC GÜLSÜN
  - Sailing: Jun 12, CNSGH
  - Scheduled ETA: ~Jul 3 (±3 days)
- **Inland leg (USLAX → Phoenix):** Rough estimate on G_coarse
  - Cost range: $400–900 | Transit: 1–3 days
  - No specific carrier, schedule, or mode committed

### Re-Plan Trigger Fires When:
- Vessel 72h from USLAX
- AIS-derived ETA confidence > 90%
- Port clearance window confirmed

### T=2 (Vessel Near USLAX)
- **Ocean leg:** Confirmed via AIS
  - Actual ETA: Jul 4, 06:00 PST
  - Port clearance est: Jul 4, 14:00 PST
- **Inland leg:** Re-solved on G_fine with actual schedules + spot rates
  - [A] Direct Truck (OHL) — depart Jul 4 16:00, arrive Jul 5 09:00 — **$612 ✓ selected**
  - [B] Drayage to Compton + Rail — depart Jul 4 20:00, arrive Jul 6 08:00 — $389
  - [C] Expedite (FedEx Custom Critical) — depart Jul 4 14:30, arrive Jul 4 22:00 — $1,480

---

## Legend (draw.io diagram color coding)
- **Blue** — Precise / Committed
- **Orange dashed** — Rough Estimate / Placeholder on G_coarse
- **Green** — Optimized / Fine-grained (post re-plan)
- **Red** — Re-plan Trigger event

Diagram file: `~/Projects/ai-freight-agent/docs/rolling_horizon_planning.drawio`

---

## Design Principle

The optimizer holds a full door-to-door plan at all times. Future legs are placeholders on G_coarse — enough to make good booking decisions today. As each leg's execution window approaches, it is re-solved on G_fine with real data. The re-plan trigger fires when confidence in the upstream leg's arrival time exceeds a threshold.

**Hard requirement:** Single-horizon planning (plan once, execute) is not acceptable.
