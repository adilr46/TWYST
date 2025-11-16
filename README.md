# Twyst – NFC Tap-to-Order Dining Platform

Twyst is an NFC tap-to-order dining platform that orchestrates order capture, payment, and fulfillment using **Stripe’s PaymentIntent lifecycle**. The system is designed as an event-driven backend where Stripe webhooks drive deterministic state transitions, even under retries and partial failures.

This repository documents Twyst’s **architecture, state machine, and Stripe integration patterns** for other engineers and reviewers. It does **not** contain production source code.

---

## Core Concepts

- **NFC-triggered ordering**  
  Each table has an NFC tag pointing to a venue-specific ordering URL.

- **Event-driven backend**  
  Stripe `PaymentIntent` events are treated as the canonical drivers of order state.

- **Deterministic state machine**  
  Orders move through a strict lifecycle:
  `CREATED → PAYMENT_PENDING → PAYMENT_SUCCEEDED → FULFILLING → COMPLETED`

- **Multi-tenant isolation**  
  Each venue operates within its own Stripe Connect account.

- **Operational observability**  
  Dashboards surface live orders, fulfillment status, and payout snapshots.

---

## High-Level Architecture

- **Frontend**  
  Next.js application rendered in the diner’s browser after tapping NFC.

- **Backend API**  
  Receives order requests, orchestrates business logic, and persists state (e.g. in Supabase).

- **Stripe Integration**  
  PaymentIntent lifecycle events inform both order state transitions and payout reconciliation.

- **Webhook Handler**  
  Central ingestion point for Stripe events; enforces idempotency, signature verification, and event sequencing.

- **Venue Dashboard**  
  Used by venue operators to manage orders and view payout summaries.

A more detailed text diagram is available in `docs/architecture-diagram.md`.

---

## Order → Payment → Fulfillment Lifecycle

1. **Order creation**  
   - Frontend calls Twyst API to create a `CREATED` order.  
   - Backend creates a Stripe PaymentIntent with metadata (e.g. `order_id`, `venue_id`, `table_id`).

2. **Payment confirmation**  
   - The diner confirms the PaymentIntent in the browser.  
   - Stripe emits events like `payment_intent.succeeded`.

3. **Backend transition**  
   - Webhook handler receives events, verifies signatures, checks idempotency, and transitions the order to `PAYMENT_SUCCEEDED`.

4. **Fulfillment**  
   - Venue dashboards display `PAYMENT_SUCCEEDED` orders as “Ready to Fulfill”.  
   - Staff move orders to `FULFILLING` and then `COMPLETED`.

5. **Post-payment reconciliation**  
   - Stripe payout information is periodically synced and compared against Twyst’s local aggregates.  
   - Any discrepancies are flagged for follow-up.

The full state machine is described in `docs/state-machine.md`.

---

## Multi-Tenant Stripe Connect Model

Each venue in Twyst has a **dedicated Stripe Connect account**, ensuring:

- isolated settlements  
- correct fund flows  
- clean per-venue reporting  
- accurate reconciliation  

Twyst creates PaymentIntents in the platform account using Connect’s `transfer_data` field:

```json
{
  "transfer_data": {
    "destination": "<venue_connect_account_id>"
  }
}
```


Destination accounts are selected based on venue metadata stored in Twyst’s database.

Twyst monitors settlement correctness using:

- Stripe balance & payout APIs  
- `payout.*` events (e.g., `payout.paid`)  
- internal reconciliation jobs comparing Stripe payouts with local aggregates  

Dashboards display:

- settlement timelines  
- payout amounts  
- discrepancy flags  
- venue-specific settlement summaries  

---

## Webhook Idempotency & Reliability

All Stripe webhooks flow through a **single, central handler**.

Each event:

- has its signature verified with Stripe’s signing secret  
- is checked against an idempotency ledger keyed by `event.id`  
- is processed only when the order’s current state and event type are compatible  
- is safe to retry — Twyst throws on transient failures so Stripe can redeliver  

These guarantees provide:

- deterministic, monotonic state transitions  
- no duplicate state updates  
- no regressions from late or out-of-order events  
- correct ordering even under repeated deliveries  

Additional sequencing details:  
`docs/webhook-sequencing-example.md`

---

## Example Flows

### 🟦 Diner Ordering

1. Diner taps NFC tag → browser loads:

https://app.twyst.example/venues/{venueId}/tables/{tableId}

2. Diner composes an order → Twyst API creates:
- a local `CREATED` order  
- a Stripe PaymentIntent  

3. Diner confirms payment → Stripe marks the PaymentIntent as `succeeded`.  

4. Webhooks update Twyst’s order state; frontend polls or receives push notifications.

---

### 🟩 Venue Receiving Orders

1. Venue dashboard polls or subscribes to its live feed.  
2. Orders in `PAYMENT_SUCCEEDED` appear in the **Ready to Fulfill** queue.  
3. Staff transition orders:
- `PAYMENT_SUCCEEDED` → `FULFILLING`  
- `FULFILLING` → `COMPLETED`  

---

### 🟧 Payouts Overview

1. Stripe accumulates charges in each venue’s Connect account.  
2. `payout.*` events (e.g., `payout.paid`) trigger reconciliation jobs.  
3. Twyst compares Stripe payout totals with local ledgers.  
4. Dashboards surface:
- payout summaries  
- settlement timelines  
- discrepancy flags  
- reconciliation status  
