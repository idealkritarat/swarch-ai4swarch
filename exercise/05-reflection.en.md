**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Kritarat Moonmanee / 6732001821
**My role on the review board:** Product owner (role card P)

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I'd trust AI with the "read a lot and find the gaps" work. In Run B it got all 17 dependencies right from the manifests, and on our board the AI review caught the order plus outbox atomicity gap that we humans missed. Drafting an ADR with the right numbers is fine too, if someone checks it.

I'd keep the business side for humans. I was the product owner, and I raised that customers won't see loyalty points right away, that cancelled or refunded orders need to reverse points, and that fraud may need checking before confirmation. The AI review had none of these, and it couldn't, since they weren't in the ADR or the scenario. So AI can say a design is plausible, but only people who know the business can say it's appropriate, and the final call stays with the architect and the board.
