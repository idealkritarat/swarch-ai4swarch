Reviewer: Claude Code (Claude agent), model Claude Opus 5.5 (`claude-opus-5-5`)

# Review of ADR-0002: "order placed" events to RabbitMQ through an outbox

Evidence: `docs/adr/0002-publish-order-placed-events-to-rabbitmq.md` (the ADR) and `docs/scenario.md`. All numbers come from the scenario.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | Shoppers | 200 `PlaceOrder`/s | 11.11 peak, MailFast at p95 1.5 s / p99 8 s | Order commits (order + outbox row) without calling MailFast | `PlaceOrder` p95 ≤ 500 ms |
| S2 Availability | RabbitMQ, `emailservice` or `loyaltyservice` | Down for 10 min | During the peak | Orders still succeed; events wait in the outbox or a queue, then drain | 0 orders failed because of them; 0 events lost after recovery |
| S3 Correctness | Outbox relay or broker | Delivers the same `OrderPlaced` twice | Peak, with the email backlog growing | Loyalty awards points once | Awarded ≤ 60 s after the order; 0 double awards |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | The order and outbox row are written **after** `chargeCard` and `shipOrder` (Decision, "Producer"). If `checkout-db` is down, or the pod crashes before the commit, the card is charged, shipping is booked and the order then fails, with no event and no loyalty points. | The ADR lists "`checkout-db` down, orders fail" as a risk but leaves out that the customer has **already paid**. The outbox only protects orders whose commit succeeded. |
| Risk | The ADR says the email lag "may be minutes". With 200 orders/s in and 45 emails/s out, the backlog grows by **155/s** (the ADR uses 150/s, against 50/s). A 10-minute peak leaves about 93,000 emails, roughly 35 min to drain. A 1-hour peak leaves about 558,000, more than 3 hours. | The claim is likely wrong for any peak longer than a few minutes. Customers may see the lag as "order not confirmed". |
| Risk | A single active consumer at 45/s, against a MailFast p95 of 1.5 s and p99 of 8 s, needs about 70 to 360 sends in flight (Little's law). The ADR does not say whether the consumer sends concurrently or what its prefetch is. | If it sends one email at a time, it reaches less than 1 email/s, not 45/s. |
| Risk | The relay runs "inside `checkoutservice` or its own deployment" (undecided). If it runs in every `checkoutservice` replica, the replicas compete for the same rows unless they claim them (row locks or a leader). | Mass duplicate publishes. Loyalty handles these, but the emails get duplicated too (see the MailFast idempotency open question). |
| Sensitivity point | Relay polling interval and batch size, plus its throughput (it must sustain ≥ 200 events/s). | These add directly to the time to award loyalty points (1 min budget). The ADR leaves them out. |
| Sensitivity point | Email consumer concurrency/prefetch, and the 45/s cap. | Email lag and MailFast rejections depend directly on them. The ADR names the 50/s limit but not the in-flight concurrency. |
| Trade-off point | The new `checkout-db` on the order path: durability of the event (reliability) against a new synchronous dependency (availability) and commit latency (performance). Synchronous PostgreSQL replication makes this sharper. | It shows up as S1 against S2. The replication mode is not chosen. |
| Trade-off point | RabbitMQ rather than Kafka: team fit and per-message retries/DLQ (operability) against replay for backfilling new consumers (modifiability). | Next year's recommendations consumer cannot replay past orders, a limit the ADR admits. RabbitMQ Streams was not considered as a middle path. |
| Non-risk | The loyalty idempotent consumer: a unique `order_id` in the same transaction as the award, acknowledged after commit. | A sound way to get an exactly-once effect from at-least-once delivery (S3). |
| Non-risk | One durable queue per action on a topic exchange. | The loyalty path is never stuck behind the email backlog, and new actions need no change to `checkoutservice`. |
| Non-risk | Removing the in-line `sendOrderConfirmation` from `PlaceOrder`. | MailFast's 1.5 s / 8 s latency leaves the order path, so S1 is likely met **[Inferred]** (the ADR plans a load test to confirm). |

## 3. Missing or weak

- **Options not considered:** writing a "pending" outbox/order row **before** charging the card; publishing directly with publisher confirms and no outbox (simpler; it loses the event only when the broker is down); change data capture (CDC) from the outbox instead of a polling relay; RabbitMQ Streams for replay.
- **Scope creep:** storing a full **order row** is a new responsibility (an order store) that R-12 does not need. Only the outbox row is needed. The ADR does not justify the order row.
- **Consequences not stated:** when RabbitMQ is down for more than 1 minute, loyalty **breaks its 1-minute target**. The outbox gives "eventually", not "within 1 minute". The broker's queue storage needed for the backlog (up to hundreds of thousands of messages) is not stated.
- **Event contract:** there is no schema or versioning for `OrderPlaced` (e.g. protobuf, which the gRPC team already knows). Yet modifiability for SMS, fraud scoring and recommendations depends on that contract.
- **PII:** customer email and address are now stored in `checkout-db` and in the broker, but no retention policy is stated.

## 4. Verdict: **Accept with changes**

1. **Close the "charged but not committed" gap.** Write a pending row before `chargeCard`, or define a refund path, and drop the order row unless it is justified.
2. **Correct the email backlog maths** (155/s), get a maximum acceptable email delay from the business, and size the email consumer (concurrency, prefetch, queue storage). If the delay is too long, consider asking MailFast for a higher limit.
3. **Fix the relay design:** its own deployment or row claiming, a polling interval budgeted inside the 1-minute loyalty target, ≥ 200 events/s, and explicit behaviour when the broker is down for more than 1 minute.
