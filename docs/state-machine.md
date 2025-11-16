Order State Machine
Twyst enforces a strict state machine to guarantee that order progression matches Stripe’s PaymentIntent lifecycle. Transitions occur only when both the current state and incoming event are valid.
States
CREATED: Order record exists; PaymentIntent initiated but not confirmed.
PAYMENT_PENDING: Payment confirmation is in progress (e.g., requires_action).
PAYMENT_SUCCEEDED: Stripe confirmed funds captured.
FULFILLING: Venue staff acknowledged order and is preparing it.
COMPLETED: Order fulfilled to diner.
CANCELLED (optional): Manually cancelled before fulfillment.
EXPIRED (optional): Payment was never completed within acceptable timeframe.
Transition Table
Current State
Triggering Event / Action
Next State
CREATED
Frontend detects PaymentIntent requires confirmation
PAYMENT_PENDING
CREATED
payment_intent.succeeded (e.g., one-step wallets)
PAYMENT_SUCCEEDED
PAYMENT_PENDING
payment_intent.succeeded
PAYMENT_SUCCEEDED
PAYMENT_PENDING
payment_intent.payment_failed
CREATED or CANCELLED
PAYMENT_SUCCEEDED
Venue accepts order (dashboard action)
FULFILLING
FULFILLING
Venue marks order complete
COMPLETED
CREATED
Admin or diner cancels before payment
CANCELLED
PAYMENT_PENDING
Payment not completed within timeout
EXPIRED
PAYMENT_SUCCEEDED
Admin cancels post-payment (rare exception)
CANCELLED

Stripe Events and Actions:
payment_intent.created: initializes state; no transition unless metadata indicates auto-capture.
payment_intent.requires_action: transitions CREATED → PAYMENT_PENDING.
payment_intent.succeeded: transitions to PAYMENT_SUCCEEDED.
payment_intent.payment_failed: returns to CREATED or moves to CANCELLED, depending on policy.
charge.refunded: (if supported) may move from COMPLETED to a separate refund tracking workflow.
ASCII Lifecycle Diagram
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
Optional exits:
- `CREATED` → `CANCELLED` (manual cancel)
- `PAYMENT_PENDING` → `EXPIRED` (timeout)
- `PAYMENT_SUCCEEDED` → `CANCELLED` (manual post-payment cancel)
