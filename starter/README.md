# E-commerce Support Agent — Submission

Implementation of the customer support AI agent described in the project
README (repo root), built on Amazon Bedrock AgentCore.

## What's implemented in `main.py`

- **Cloud runtime**: `BedrockAgentCoreApp` + async `@app.entrypoint invoke()`, deployed via `agentcore deploy`.
- **MCP Gateway tools**: `MCPClient` loads two Gateway-backed targets — `order-tracker` (API-based) and `refund-processor` (Lambda-based).
- **RAG**: `search_knowledge_base` tool calls the Bedrock Knowledge Base `Retrieve` API, with a guarded fallback when `KB_ID` isn't configured.
- **Cross-session memory**: `MemoryHook` (a `HookProvider`) retrieves relevant memories before each turn and saves the interaction afterward via `memory_client.create_event()`.
- **Code interpreter**: `calculate_loyalty_discount` runs the discount math in a sandboxed `code_session`, with a local fallback if the sandbox is unavailable.
- **Browser tool**: `AgentCoreBrowser` is added to the agent's tool list for live web access.

## Running a test locally

Toggle the CLI entry point at the bottom of `main.py` (comment `app.run()`,
uncomment `main()`), then:

```bash
uv run main.py '{"prompt": "Can you track order ORD-001?", "customer_id": "CUST-123", "session_id": "t1"}'
```

Revert the toggle before deploying (`app.run()` must be active for `agentcore deploy`).

## Test evidence

Screenshots for all 6 test scenarios (`agentcore invoke` against the deployed
agent) — full-size originals are in [`screenshots/`](./screenshots).

### Test 1 — Order Tracking
![Order tracking](./screenshots/test_1_order_tracking.png)

### Test 2 — Refund Processing
![Refund processing](./screenshots/test_2_refund_processing.png)

### Test 3 — Knowledge Base (RAG)
![Knowledge base RAG](./screenshots/test_3_knowledge_base_rag.png)

### Test 4 — Long-Term Memory (both sessions)
Session A (introduce name + preference):
![Memory session A](./screenshots/test_4a_memory_session_A.png)

Session B (recall, new session):
![Memory session B](./screenshots/test_4b_memory_session_B.png)

### Test 5 — Loyalty Discount Calculation
![Loyalty discount calculator](./screenshots/test_5_loyalty_discount.png)

### Test 6 — Browser Tool
![Browser tool](./screenshots/test_6_browser_tool.png)

## Reflection

See [`REFLECTION.md`](./REFLECTION.md) — 200-400 word write-up covering a
design decision, a challenge encountered, and a production consideration.

## Building this from scratch

See the [project README](../README.md) — it's the full step-by-step guide
(AWS console setup + `main.py` TODOs), with a **⚠️ Gotcha** note inlined at
every step that caused a real error while building this submission, so a
fresh build shouldn't hit the same ones.
