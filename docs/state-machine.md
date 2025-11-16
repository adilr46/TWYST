# Order State Machine

Twyst enforces a strict state machine to ensure that order progression always matches Stripe’s PaymentIntent lifecycle.  
Transitions occur only when both the **current state** and the **incoming event** are valid.

---

## States

- **CREATED** — Order record exists; PaymentIntent initiated but not yet confirmed.  
- **PAYMENT_PENDING** — Payment confirmation is in progress (e.g., `requires_action`).  
- **PAYMENT_SUCCEEDED** — Stripe confirmed funds captured.  
- **FULFILLING** — Venue staff acknowledged the order and is preparing it.  
- **COMPLETED** — Order fulfilled to the diner.  
- **CANCELLED** *(optional)* — Order cancelled before fulfillment.  
- **EXPIRED** *(optional)* — Payment was not completed within the allowed timeframe.  

---

## Transition Table

| Current State       | Triggering Event / Action                             | Next State          |
|---------------------|--------------------------------------------------------|---------------------|
| CREATED             | PaymentIntent requires confirmation                    | PAYMENT_PENDING     |
| CREATED             | `payment_intent.succeeded` (e.g., one-step wallets)    | PAYMENT_SUCCEEDED   |
| PAYMENT_PENDING     | `payment_intent.succeeded`                             | PAYMENT_SUCCEEDED   |
| PAYMENT_PENDING     | `payment_intent.payment_failed`                        | CREATED / CANCELLED |
| PAYMENT_SUCCEEDED   | Venue accepts order (dashboard action)                 | FULFILLING          |
| FULFILLING          | Venue marks order complete                             | COMPLETED           |
| CREATED             | Admin/diner cancels before payment                     | CANCELLED           |
| PAYMENT_PENDING     | Payment not completed within timeout                   | EXPIRED             |
| PAYMENT_SUCCEEDED   | Admin cancels after successful payment *(rare)*        | CANCELLED           |

---

## Stripe Events and Actions

- **payment_intent.created** — Initializes state; generally leaves the order in `CREATED`.  
- **payment_intent.requires_action** — Transitions `CREATED` → `PAYMENT_PENDING`.  
- **payment_intent.succeeded** — Transitions to `PAYMENT_SUCCEEDED` from `CREATED` or `PAYMENT_PENDING`.  
- **payment_intent.payment_failed** — Moves back to `CREATED` or `CANCELLED`.  
- **charge.refunded** *(if supported)* — May trigger a separate refund/adjustment workflow from `COMPLETED`.  

---

## ASCII Lifecycle Diagram

```text
          +----------------+
          |    CREATED     |
          +----------------+
                 |
                 | Payment requires confirmation
                 v
         +---------------------+
         |  PAYMENT_PENDING    |
         +---------------------+
                 |
                 | payment_intent.succeeded
                 v
         +---------------------+
         |  PAYMENT_SUCCEEDED  |
         +---------------------+
                 |
                 | Venue accepts order
                 v
         +---------------------+
         |    FULFILLING       |
         +---------------------+
                 |
                 | Order prepared + served
                 v
         +---------------------+
         |     COMPLETED       |
         +---------------------+

Optional Exits
CREATED → CANCELLED (manual cancel)

PAYMENT_PENDING → EXPIRED (timeout)

PAYMENT_SUCCEEDED → CANCELLED (manual post-payment cancel)
- `PAYMENT_PENDING` → `EXPIRED` (timeout)
- `PAYMENT_SUCCEEDED` → `CANCELLED` (manual post-payment cancel)
