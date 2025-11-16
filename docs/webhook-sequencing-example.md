# Webhook Sequencing Example

## Scenario Overview

A diner completes payment for `order_test_001`.  
Stripe emits multiple events related to the same PaymentIntent, and Twyst’s webhook handler must enforce **idempotency** and **correct ordering** to ensure a consistent final state.

---

## Events Received (Arrival Order)

1. `evt_test_456` — `payment_intent.succeeded` for `pi_test_123`  
2. `evt_test_123` — `payment_intent.created` for `pi_test_123` *(late / retry)*  
3. `evt_test_789` — `charge.succeeded` linked to `pi_test_123`  

---

## Processing Steps

### 1. `payment_intent.succeeded` (`evt_test_456`)
- Check idempotency ledger → **no existing entry**  
- Lookup order via PaymentIntent metadata  
- Current order state: `PAYMENT_PENDING`  
- Transition: `PAYMENT_PENDING` → `PAYMENT_SUCCEEDED`  
- Mark `evt_test_456` as processed  

---

### 2. `payment_intent.created` (`evt_test_123`)
- Verify signature, check ledger → **not processed yet**  
- Order state already `PAYMENT_SUCCEEDED`  
- Sequencing guard detects invalid transition  
- Log out-of-sequence event  
- Do **not** change state  
- Mark `evt_test_123` as processed (prevents retry loops)  

---

### 3. `charge.succeeded` (`evt_test_789`)
- Idempotency check passes  
- Order remains in `PAYMENT_SUCCEEDED`  
- Record charge details for reconciliation  
- Mark `evt_test_789` as processed  

---

## Reliability Behaviors

- Every `event.id` is recorded in the idempotency store to prevent replay-based duplication.  
- Sequencing guards ensure late events (e.g., `payment_intent.created`) **cannot regress** the order.  
- Charge-level events enrich reconciliation data without mutating the core order state once payment final
