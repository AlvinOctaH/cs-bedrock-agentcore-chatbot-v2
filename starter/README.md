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
agent) are in [`screenshots/`](./screenshots):

1. [Order tracking](./screenshots/test_1_order_tracking.png)
2. [Refund processing](./screenshots/test_2_refund_processing.png)
3. [Knowledge base RAG](./screenshots/test_3_knowledge_base_rag.png)
4. Long-term memory recall — [session A](./screenshots/test_4a_memory_session_A.png) / [session B](./screenshots/test_4b_memory_session_B.png)
5. [Loyalty discount calculator](./screenshots/test_5_loyalty_discount.png)
6. [Browser tool](./screenshots/test_6_browser_tool.png)

## Reflection

See [`REFLECTION.md`](./REFLECTION.md) — 200-400 word write-up covering a
design decision, a challenge encountered, and a production consideration.

## Troubleshooting / setup notes

See [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md) for the errors hit while
building this (bad `KB_ID`, wrong Gateway/Memory API usage, IAM permission
gaps in the deployed runtime, etc.) and how to avoid them from a fresh setup.
