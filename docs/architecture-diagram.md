# Architecture Diagram (Text Outline)

```text
[Diner: NFC Tap + Browser]
        |
        | (1) HTTPS request to load ordering UI
        v
[Next.js Frontend (Static + SSR)]
        |
        | (2) HTTPS API calls (order create/update)
        v
[Twyst Backend API] ------------+
        |                       |
        | (3) SQL / RPC         | (5) HTTPS API calls (create PaymentIntent, attach metadata)
        v                       v
[Supabase / Database]      [Stripe (PaymentIntents + Connect)]
                                    |
                                    | (4) Asynchronous webhooks (payment events)
                                    v
                               [Webhook Handler]
                                    |
                                    | (6) Internal event dispatch
                                    v
                             [Order State Machine]
                                    |
                                    | (7) Database updates + notifications
                                    v
                             [Venue Dashboard UI]
Legend
(1), (2), (3), (5) — Synchronous HTTPS or DB/RPC requests

(4) — Asynchronous Stripe webhook delivery

(6), (7) — Internal async processing + state updates used by the dashboar
