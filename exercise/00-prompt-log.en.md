**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Kritarat Moonmanee / 6732001821
**Agent(s) used:** Claude Code (CLI). I ran each task as a new headless session (`claude -p`) in the repo folder, so it loaded CLAUDE.md and AGENTS.md every time. Tools allowed: Read, Write, Glob, Grep only, no terminal.
**Model(s):** Claude Opus 5.5 (`claude-opus-5-5`)
**Plan / tier:** my paid Claude plan

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A, evidence demo.proto | prompts/1-recover-architecture.md, protos/demo.proto | Yes | 3 | Got all 9 gRPC services but 0 dependencies, it put the likely calls in Gaps as [Inferred]. Honest but not much of a diagram |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-…` run B, evidence demo.proto + kubernetes-manifests/ | prompts/1-recover-architecture.md, protos/demo.proto, all 13 files in kubernetes-manifests/ (README, kustomization and 11 yaml) | Yes. It also listed the file names in exercise/outputs/ with Glob but didn't open them | 5 | 12 containers (incl. redis-cart and loadgenerator), 17 deps, every one I checked was right |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-…` run a, context none, R-12 pasted | prompts/2-write-adr.md, docs/adr/template.md | Yes, it did not open scenario.md | 3 | Generic pub/sub ADR, broker not chosen, no numbers |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-…` run b, context scenario + ADR-0001 + checkoutservice/main.go | prompts/2-write-adr.md, docs/adr/template.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, src/checkoutservice/main.go | Yes | 5 | Picked RabbitMQ, used the real numbers and line refs. Left the lost event gap as an open question |
| 5 | 3 | Claude Code + Opus 5.5 | `prompts/3-…` adr: docs/adr/0002-publish-order-placed-events-to-rabbitmq.md | prompts/3-review-adr.md, my ADR 0002, docs/scenario.md | Yes, didn't touch my ADR | 4 | Found the "card charged but order not committed" gap in my outbox design |
| 6 | | | | | | | |
| 7 | | | | | | | |
