# 10 — Troubleshooting (every problem actually hit)

Issues hit while building this agent, their root cause, and how to avoid them when
rebuilding from scratch.

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Local CLI test does nothing / hangs | `__main__` runs `app.run()` (ASGI server for deploy), not the CLI | For `uv run main.py '{...}'`, comment out `app.run()` and uncomment `main()` — **revert before deploying** |
| 2 | KB `Retrieve` → `ResourceNotFoundException` | Stale / wrong `KB_ID` | `aws bedrock-agent list-knowledge-bases --region us-east-1` and copy the real ID |
| 3 | KB `Retrieve` → *vectorSearchConfiguration is not supported for managed knowledge bases* | Wrong retrieval config for the KB type | `get-knowledge-base`; if `type` is `MANAGED` use `managedSearchConfiguration`, otherwise `vectorSearchConfiguration` |
| 4 | *MCP Gateway target could not load tools dynamically* | `MCPClient` has no `gateway_url=` kwarg; an unused `async with streamable_http_client(...)` block | `MCPClient(lambda: streamable_http_client(GATEWAY_URL))`, then `await mcp_link.load_tools()` |
| 5 | `'list' object has no attribute 'get'` from memory calls | `get_memory_strategies()` / `retrieve_memories()` return **lists**, not dict wrappers | Iterate the list; namespaces are in `namespaces`/`namespaceTemplates` (take `[0]`); records are `{"content": {"text": ...}}`; use `memory_id`/`namespace`/`query`/`top_k` keyword names |
| 6 | `create_event()` fails | It has **no** `namespace` parameter | `create_event(memory_id=..., actor_id=..., session_id=..., messages=[(text, "USER"), (text, "ASSISTANT")])` |
| 7 | Memory hooks never fire | Hooks assigned after construction (`agent.hooks = ...`) | Pass them in the constructor: `Agent(..., hooks=[memory_hook])` — registration happens only in `__init__` |
| 8 | Hooks fire but never read/save anything | `event.agent.messages` are **dicts** (Converse format), not objects — `getattr(msg, "role")` silently returns nothing | Dict access + helpers `_message_text()` / `_is_tool_message()` |
| 9 | Discount tool always falls back: *CodeInterpreter.invoke() got an unexpected keyword argument 'code'* | `invoke(method, params)` takes one params dict | `session.invoke("executeCode", {"code": ..., "language": "python", "clearContext": True})` and read stdout from `resp["stream"]` → `structuredContent` |
| 10 | Browser tool: *Timeout should be used inside a task* | Synchronous `agent(prompt)` inside the async entrypoint breaks anyio task-scoped timeouts | `await agent.invoke_async(...)` |
| 11 | Refund amount comes back as `$0` | The agent didn't look up the order, so it never passed `amount` (optional in `lambda_schema`) | System prompt: look up the order first and pass its real total. Don't change the Lambda (provided infrastructure) |
| 12 | Everything works locally but fails silently when deployed | The runtime's own IAM role lacks permissions; tool fallbacks mask `AccessDeniedException` | `aws logs tail /aws/bedrock-agentcore/runtimes/<agent>-DEFAULT --since 1h`; grant the Memory ARN, `bedrock:Retrieve` on the KB and the browser session actions (see [07](07-entrypoint-and-deployment.md#iam-scoping-for-the-deployed-agent)) |
