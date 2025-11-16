# TWYST NFC Tap-to-Order Dining Platform
Introduction
Twyst enables diners to tap NFC tags at restaurant tables and instantly launch a web ordering flow. The system orchestrates order capture, payment, and fulfillment using Stripe’s PaymentIntent lifecycle, ensuring deterministic behavior across retries and partial failures.
Core Concepts
NFC-triggered ordering: Each table links to a venue-specific ordering surface hosted in a browser.
Event-driven backend: Stripe PaymentIntent events drive order progression through local state.
Deterministic state machine: Orders transition only through validated states to preserve consistency.
Multi-tenant isolation: Each venue operates within its own Stripe Connect account.
Operational observability: Venue dashboards surface live orders, fulfillment status, and payouts.
High-Level Architecture
Frontend: Next.js application rendered in diners’ browsers after tapping NFC tags.
Backend API: Receives client requests, orchestrates orders, and persists state in Supabase.
Stripe Integration: PaymentIntent lifecycle events inform order state transitions and payouts.
Webhook Handler: Central ingestion point for Stripe events; enforces idempotency and sequencing.
Venue Dashboard: Venue operators review and manage incoming orders alongside payout snapshots.
Order → Payment → Fulfillment Lifecycle
Order creation: Frontend calls Twyst API to create a CREATED order tied to a Stripe PaymentIntent (metadata includes venue/table).
Payment confirmation: Browser confirms PaymentIntent; Stripe emits payment_intent.succeeded.
Backend transition: Webhook handler receives events, verifies idempotency, and transitions order to PAYMENT_SUCCEEDED.
Fulfillment: Venue acknowledges the order (FULFILLING → COMPLETED) using dashboard actions or fulfillment apps.
Post-payment reconciliation: Stripe payout information syncs with local venue records to reconcile revenue share and fees.
Multi-tenant Stripe Connect Model
Each venue corresponds to a distinct Stripe Connect account.
Twyst creates PaymentIntents in the platform account with transfer_data[destination] pointing to the venue account.
Destination accounts are selected based on venue metadata stored in Supabase.
Connect payouts are monitored via Stripe’s balance and payout APIs; reconciliation jobs compare Stripe data against local aggregates.
Webhook Idempotency & Reliability
All Stripe webhooks flow through a single handler.
Each event is verified using Stripe’s signature header.
Events write to an idempotency ledger keyed by event.id; processed events are skipped.
Sequencing guards ensure state transitions occur only when the current order state matches expectations, defending against out-of-order retries.
Webhook processing is designed to be retry-safe: transient failures allow Stripe to re-send until processing succeeds.
Example Flows
Diner Ordering
Diner taps NFC tag → browser loads https://app.twyst.example/venues/{venueId}/tables/{tableId}.
Diner composes order → Twyst API creates order + PaymentIntent.
Diner confirms payment → Stripe redirects back or confirms via JS → PaymentIntent transitions to succeeded.
Webhooks update Twyst order state; diner UI polls or receives push updates.
Venue Receiving Orders
Venue dashboard subscribes to secure websocket or polling endpoint.
When order state moves to PAYMENT_SUCCEEDED, dashboard highlights the order in “Ready to Fulfill”.
Venue staff marks order as FULFILLING, then COMPLETED once handed to diner.
Payouts Overview
Stripe accumulates charges in each destination Connect account.
Stripe payout events (e.g., payout.paid) trigger reconciliation jobs.
Twyst compares payout amounts against internal ledgers; discrepancies raise alerts to ops.
Dashboard surfaces settlement timelines and payout summaries per venue.
