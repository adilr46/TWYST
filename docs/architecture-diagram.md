Architecture Diagram Outline

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
        | (3) SQL/ RPC          | (5) HTTPS API calls (create PaymentIntent, attach metadata)
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
Legend:
Numbered vertical arrows (1), (2), (3), (5) represent synchronous HTTPS requests.
Arrow (4) represents asynchronous webhook delivery from Stripe to Twyst.
Arrows (6) and (7) represent internal asynchronous processing and resulting state updates consumed by the dashboard.
