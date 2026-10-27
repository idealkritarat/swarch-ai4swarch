**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| frontend → checkoutservice | `frontend.yaml` (`CHECKOUT_SERVICE_ADDR`) | ✅ | `src/frontend/main.go:136` maps the env var and `src/frontend/handlers.go:354` calls `PlaceOrder` |
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go:116` and `Charge` at line 370 |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | `main.go:380` `SendOrderConfirmation`, and if it fails it only logs a warning |
| checkoutservice → cartservice | `checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | `main.go:325` `GetCart` and line 333 `EmptyCart` |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | `src/recommendationservice/recommendation_server.py:131` reads the address |
| cartservice → redis-cart | `cartservice.yaml` (`REDIS_ADDR`) | ✅ | `src/cartservice/src/Startup.cs:29` reads `REDIS_ADDR` and uses Redis as the cart cache |
| frontend → shippingservice | `frontend.yaml` (`SHIPPING_SERVICE_ADDR`) | ✅ | `src/frontend/rpc.go:88` calls `GetQuote` for the cart page |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): I didn't find any false dependency, all 17 match the code. Missing: `shoppingassistantservice` exists in `src/` and the frontend calls it over HTTP (`frontend/main.go:139`, `handlers.go:464`), but it has no manifest, so B could only list it under Gaps. It's optional though. Questionable: `loadgenerator` is drawn as part of the system, but it's a Locust test tool and it's commented out in `kustomization.yaml`, so I wouldn't put it in the real container diagram. Also every language is "not evidenced", the code shows Go (frontend, checkout, catalog, shipping), C# (cart), Python (recommendation, email), Node.js (currency, payment) and Java (ad).

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

   From the proto alone it couldn't know who calls who, there's no frontend, no Redis and no ports, because a proto only says what each service offers, not who uses it. So A had 0 dependencies and B had 17. The `*_SERVICE_ADDR` env vars in the manifests are basically the wiring, that's what made the difference.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

   Putting `loadgenerator` inside the system boundary. A new person would think it's a real service that has to run in prod, but it's just a test tool and it's not even in the kustomization. Not knowing about the shopping assistant is less bad since it's optional.
