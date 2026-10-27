**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | Teerachot | defends the ADR |
| **AI analyst** | Nathawat| reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Kritarat| role card P (business side) |
| **Site reliability engineer (SRE)** | Piranat| role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-use-rabbitmq-for-post-order-action.md` by Teerachot  
**Board members and roles:** 

| Role | Who | Brings |
|---|---|---|
| **Architect** | Teerachot | defends the ADR |
| **AI analyst** | Nathawat| reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Kritarat| role card P (business side) |
| **Site reliability engineer (SRE)** | Piranat| role card S (operations side) |

**Decision:** accept with changes

**Required changes:**
1. Define and demonstrate atomic order-plus-outbox acceptance, relay recovery, and duplicate-publication handling.
2. Set peak duration, consumer capacity, outage allowance, and acceptable backlog; demonstrate the one-minute loyalty objective and the 50 emails/s MailFast limit.
3. Resolve MailFast idempotency and safe retries after ambiguous timeouts.
4. Specify customer-visible loyalty timing and how cancellations and refunds reverse awards.
5. Assign operational ownership and document paging, replay, and tracing; make a separate decision for any fraud check that must happen before order confirmation.

**Who raised what?**

| #   | Concern                                                                                                                   | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review?                                                                  | Could the AI have known? Why or why not?                                                                                                                               |
| --- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Independent queues isolate checkout and the other actions from MailFast latency and its 50 emails/s limit.                | AI review                                               | Yes                                                                                         | Yes. ADR-0002 states the provider latency, rate limit, peak load, and queue decision.                                                                                  |
| 2   | Atomic order-plus-outbox acceptance and relay recovery are unresolved.                                                    | AI review                                               | Yes                                                                                         | Yes. ADR-0002 leaves the order transaction and durable storage choice open.                                                                                            |
| 3   | Peak duration, outage allowance, consumer capacity, and acceptable backlog determine whether loyalty can meet one minute. | AI review                                               | Yes                                                                                         | Yes. ADR-0002 lists these as open questions.                                                                                                                           |
| 4   | Loyalty idempotency must be atomic; MailFast idempotency and safe retry behavior remain unresolved.                       | AI review                                               | Yes                                                                                         | Yes. ADR-0002 states the loyalty requirement and asks whether MailFast supports idempotency.                                                                           |
| 5   | Loyalty may not appear immediately to customers, and refund or cancellation reversals are unspecified.                    | PO                                                      | No                                                                                          | No. These product requirements came from stakeholder context in the role card and are not stated in the ADR or AI review.                                              |
| 6   | Launch readiness and pre-confirmation fraud need consideration.                                                           | PO                                                      | No                                                                                          | No. The deadline and fraud timing came from stakeholder context not present in the ADR or AI review.                                                                   |
| 7   | Paging ownership, replay procedures, and tracing across the asynchronous boundary are unclear.                            | SRE                                                     | Partly: the AI noted operational work and recovery gaps, but not paging or tracing details. | The ADR raises operational ownership and recovery, so the AI could identify those gaps. The specific paging and tracing needs were not stated in the ADR or AI review. |
| 8   | Durable writes may add checkout-path latency, but the impact is not measured.                                             | AI review                                               | Yes                                                                                         | The AI could identify this as a possibility. The ADR does not provide a storage design or measurement to establish the impact.                                         |

**The AI's best finding** (one the humans had missed): The accepted order and outbox event need atomic persistence with a defined relay recovery path; otherwise an accepted order could lose its post-order work.

**The AI's weakest finding** (generic, wrong or irrelevant): Durable writes may add checkout-path latency. This is a reasonable validation question, but the impact is unmeasured and depends on the storage design.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
