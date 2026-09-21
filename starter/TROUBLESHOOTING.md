# Setup Notes / Troubleshooting

Issues hit while building this agent, their root cause, and how to avoid them
if rebuilding from scratch.

## 1. Local CLI testing does nothing / hangs

`main.py`'s `if __name__ == "__main__":` block runs `app.run()` (an ASGI
server, for `agentcore deploy`), not the CLI path. For local testing with
`uv run main.py '{...}'`, comment out `app.run()` and uncomment `main()` —
then **revert it before deploying**, since `agentcore deploy` requires
`app.run()` to be active.

## 2. Knowledge Base `Retrieve` fails with `ResourceNotFoundException`

`KB_ID` was stale/wrong. Confirm the real ID with:
```bash
aws bedrock-agent list-knowledge-bases --region <region>
```

## 3. KB `Retrieve` fails with `vectorSearchConfiguration is not supported for managed knowledge bases`

Check the KB type first:
```bash
aws bedrock-agent get-knowledge-base --knowledge-base-id <id> --region <region>
```
If `knowledgeBaseConfiguration.type` is `MANAGED`, the `retrieve()` call must
use `retrievalConfiguration={"managedSearchConfiguration": {"numberOfResults": 3}}`,
not `vectorSearchConfiguration`.

## 4. Gateway tools never load (`MCP Gateway target could not load tools dynamically`)

`MCPClient` does not take a `gateway_url=` kwarg, and doesn't need to be
wrapped in an `async with streamable_http_client(...)` block you never use.
Construct it with a **transport factory**:
```python
mcp_link = MCPClient(lambda: streamable_http_client(GATEWAY_URL))
gateway_tools = await mcp_link.load_tools()
```

## 5. `get_memory_strategies()` / `retrieve_memories()` errors like `'list' object has no attribute 'get'`

Both methods return a **list** directly, not a dict wrapper:
- `get_memory_strategies(memory_id)` → `List[Dict]`, each with `type` and
  `namespaces`/`namespaceTemplates` (a list — take `[0]`).
- `retrieve_memories(memory_id=..., namespace=..., query=..., top_k=...)` →
  `List[Dict]`, each shaped `{"content": {"text": "..."}, ...}`.

Also use the SDK's actual keyword names — `memory_id`/`namespace`/`query`/
`top_k`, not `memoryId`/`topK`.

## 6. `create_event()` fails / never gets called with the right args

`MemoryClient.create_event()` has no `namespace` parameter — it's:
```python
memory_client.create_event(
    memory_id=..., actor_id=..., session_id=...,
    messages=[(text, "USER"), (text, "ASSISTANT")],  # list of (text, role) tuples
)
```

## 7. Memory hooks (`retrieve_customer_context` / `save_support_interaction`) never fire

`hooks` must be passed into the `Agent(...)` constructor:
```python
agent = Agent(model=model, tools=tools_list, system_prompt=system_prompt, hooks=[memory_hook])
```
Assigning `agent.hooks = registry` *after* construction does nothing — hook
registration only happens during `__init__`.

## 8. Memory hooks fire but never actually read/save anything

`event.agent.messages` items are **plain dicts** in Bedrock Converse format —
`{"role": "user"/"assistant", "content": [{"text": "..."}]}` for text, or
`{"toolUse": {...}}` / `{"toolResult": {...}}` for tool calls — not objects.
`getattr(msg, "role", ...)` / `hasattr(msg, "tool_call_id")` silently do
nothing. Use dict access and inspect the content blocks instead (see
`_message_text()` / `_is_tool_message()` in `main.py`).

## 9. Loyalty discount tool always falls back (`CodeInterpreter.invoke() got an unexpected keyword argument 'code'`)

`session.invoke(method, params)` takes the method name and **one params
dict** — not `code=`, `language=`, `clearContext=` as separate kwargs:
```python
resp = session.invoke("executeCode", {"code": code, "language": "python", "clearContext": True})
```
The result is an event stream, not a plain dict — read stdout from it:
```python
for event in resp["stream"]:
    structured = event.get("result", {}).get("structuredContent", {})
    stdout_output = structured.get("stdout", stdout_output)
```

## 10. Browser tool crashes with `Timeout should be used inside a task`

Calling the agent synchronously (`agent(prompt)`) from inside the async
`invoke()` entrypoint breaks anyio's task-scoped timeouts used by the browser
tool. Use the async call instead:
```python
agent_response = await agent.invoke_async(str(enriched_user_input))
```

## 11. Refund amount comes back as `$0`

The agent didn't look up the order before calling `initiate_refund`, so it
never passed a real `amount` (the field is optional in `lambda_schema`).
Add explicit guidance to the system prompt: look up the order first (e.g. via
a `get_order` tool) and pass its real total as the refund amount. Don't "fix"
this in the Lambda itself — it's pre-built backend infrastructure, not part
of the agent code you're meant to change.

## 12. Everything works locally but fails silently in the deployed (cloud) agent

Local testing uses your own broad AWS credentials; the deployed agent runs
under its own IAM execution role
(`AmazonBedrockAgentCoreSDKRuntime-<region>-<suffix>`), which starts with
**no permissions beyond what `agentcore deploy` auto-grants** (logs, X-Ray,
model invocation, the auto-created memory resource, code interpreter). It
will silently `AccessDeniedException` on anything else and your `@tool`
functions' fallback paths will mask the failure with a plausible-looking
answer. Before assuming a tool "works," check the deployed agent's logs:
```bash
aws logs tail /aws/bedrock-agentcore/runtimes/<agent>-DEFAULT --since 1h
```
and grant the execution role explicit permission for every resource the
agent actually calls: the specific Memory resource ARN in `MEMORY_ID`
(not just whatever memory `agentcore deploy` auto-created), `bedrock:Retrieve`
on the Knowledge Base ARN, and the Browser tool's session actions
(`StartBrowserSession`, `StopBrowserSession`, `GetBrowserSession`,
`ListBrowserSessions`, `ConnectBrowserAutomationStream`, `GetBrowser`,
`ListBrowsers`) scoped to `browser/aws.browser.v1`.
