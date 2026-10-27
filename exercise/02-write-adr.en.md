**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No, it never saw them. It says latency and volume targets are "not given" and lists them as open questions | Yes, all of them: 500 ms p95 (today 300), 200 orders/s, MailFast p95 1.5 s / p99 8 s / 0.5 % / 50 per s, loyalty 1 min exactly once. It also worked out the email backlog grows 150/s at peak |
| Options considered (list them) | A sync fan-out, B pub/sub through a broker (chosen), C post-order orchestrator service. Broker not chosen | 1 sync gRPC chain, 2 in-process goroutines, 3 RabbitMQ events (chosen), 4 Kafka events |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Only in general: idempotent consumers keyed on order ID. Nothing about a rate limit since it didn't know there is one | Loyalty: idempotent consumer, `order_id` unique constraint in the same transaction as the award, ack after commit. MailFast: durable queue buffers the backlog, a global 50/s limiter, retry queue and DLQ. But it admits a failed publish loses the event, so exactly once isn't fully solved |
| Anything invented (a number, tool or fact not in the scenario)? | Nothing I could find, it marks guesses as [Inferred] and keeps the broker open | Nothing invented. The 150/s backlog is computed from the scenario, and Locust is really in ADR-0001 and `src/loadgenerator` |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Added a transactional outbox: `checkoutservice` writes the order and an `OrderPlaced` outbox row to a new small PostgreSQL `checkout-db` in one transaction, and an outbox relay publishes to RabbitMQ | The AI left "publish fails, event lost" as an open question. That breaks exactly once for loyalty, which is a hard requirement, so I made the call instead of leaving it open |
| 2 | Email consumer is a single active consumer sending at most 45/s, not "global 50/s" | One consumer means one rate limiter. 45 keeps some room under MailFast's 50/s for retries |
| 3 | Picked PostgreSQL for `loyalty-db` (was "Not decided") | The idempotent award needs a unique constraint and a transaction, and it's open source and self-hosted like the company wants |
| 4 | Diagram: added `frontend` and the outbox pieces, customer now goes through frontend | The AI drew the customer calling checkout directly, which isn't how the system works |
| 5 | Fraud check before confirmation moved to "not in scope, own ADR", removed open questions the outbox already answers, set myself as decider | Keeps the ADR focused on post-order actions only |
