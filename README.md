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

##multi_tenant_stripe_connect_model:
  destination_account_selection: "Destination accounts are selected based on venue metadata stored in Twyst’s database."
  settlement_correctness_sources:
    - "Stripe balance & payout APIs"
    - "payout.* events (e.g., payout.paid)"
    - "internal reconciliation jobs comparing Stripe payouts with local aggregates"
  dashboard_displays:
    - "settlement timelines"
    - "payout amounts"
    - "discrepancy flags"
    - "venue-specific settlement summaries"

webhook_idempotency_and_reliability:
  overview: "All Stripe webhooks flow through a single, central handler."
  event_rules:
    - "signature is verified using Stripe’s signing secret"
    - "checked against idempotency ledger keyed by event.id"
    - "processed only if current order state and event type are compatible"
    - "safe to retry — Twyst intentionally throws on transient failures so Stripe redelivers"
  guarantees:
    - "deterministic transitions"
    - "no duplicate state updates"
    - "no regressions from late events"
    - "correct ordering under retries or partial failures"
  further_details: "docs/webhook-sequencing-example.md"

example_flows:

  diner_ordering:
    - "Diner taps NFC → browser loads https://app.twyst.example/venues/{venueId}/tables/{tableId}"
    - "Diner builds an order → Twyst API creates a local CREATED order and a Stripe PaymentIntent"
    - "Diner confirms payment → Stripe marks the PaymentIntent as succeeded"
    - "Webhook handler updates order state; diner UI polls or receives push updates"

  venue_receiving_orders:
    - "Venue dashboard polls or subscribes to its live feed"
    - "Orders in PAYMENT_SUCCEEDED appear in the Ready to Fulfill queue"
    - "Staff transitions: PAYMENT_SUCCEEDED → FULFILLING → COMPLETED"

  payouts_overview:
    - "Stripe accumulates charges in each venue’s Connect account"
    - "payout.* events (e.g., payout.paid) trigger reconciliation jobs"
    - "Twyst compares Stripe payout totals with its internal ledger"
    - "Dashboards display payout summaries, settlement timelines, discrepancy flags, and reconciliation status"

