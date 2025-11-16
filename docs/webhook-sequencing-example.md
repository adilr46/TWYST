 Webhook Sequencing Example
Scenario Overview
A diner completes payment for order order_test_001. Stripe emits three events. Twyst’s webhook handler must ensure idempotent, ordered application.
Events Received (chronological arrival order)
evt_test_456: payment_intent.succeeded for pi_test_123
evt_test_123: payment_intent.created for pi_test_123 (out-of-order retry)
evt_test_789: charge.succeeded linked to pi_test_123
Processing Steps
Receive payment_intent.succeeded (evt_test_456)
Check idempotency ledger → no existing entry → continue.
Lookup order via PaymentIntent metadata.
Current order state: PAYMENT_PENDING.
Transition to PAYMENT_SUCCEEDED.
Mark event as processed.
Receive payment_intent.created (evt_test_123)
Verify signature, check ledger → not processed yet.
Order is already PAYMENT_SUCCEEDED; sequencing guard detects state mismatch.
Log out-of-sequence arrival, no state change.
Mark event as processed (so future retries are ignored).
Receive charge.succeeded (evt_test_789)
Idempotency check passes.
Order state already PAYMENT_SUCCEEDED; handler records charge details (for reconciliation) without changing state.
Mark processed.
Reliability Behaviors
Every event ID is recorded in the idempotency store, preventing duplicate state changes on retries.
State guards ensure late-arriving events (payment_intent.created) do not regress the order.
Charge events enrich reconciliation data but do not mutate order state once payment finality is reached.
