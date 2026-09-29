# 07 — Agent Entrypoint, Deployment and IAM Scoping

## Configuration and initialisation (`starter/main.py:44-91`)

```python
app = BedrockAgentCoreApp()                                  # :52 — exactly one per deployment
os.environ["BYPASS_TOOL_CONSENT"] = "true"                   # :56 — headless tools
GATEWAY_URL, KB_ID, REGION, MEMORY_ID = ...                  # :68-71
model = BedrockModel(model_id="global.amazon.nova-2-lite-v1:0")   # :85
memory_client = MemoryClient(region_name=REGION)             # :88
_bedrock_runtime = boto3.client("bedrock-agent-runtime", region_name=REGION)  # :91
```

## The entrypoint (`starter/main.py:432-527`)

`@app.entrypoint async def invoke(payload, context=None)` runs for every request:

1. Read `prompt`, `customer_id` (default `ANONYMOUS_CUST`) and `session_id`
   (a new UUID if missing).
2. Create a `MemoryHook` for this customer/session.
3. Create the `AgentCoreBrowser`.
4. Tools list: `search_knowledge_base`, `calculate_loyalty_discount`, `browser`.
5. Connect the `MCPClient` to the Gateway and add its tools (in `try/except`).
6. Build the `Agent` with `hooks=[memory_hook]` and a system prompt that says:
   *look up the order first and pass its real total as the refund amount.*
7. Trim long histories and enrich the prompt with long-term memory (stand-out, see 09).
8. `await agent.invoke_async(...)` and return the text; errors return a friendly message.

## Local run vs deployment

| Mode | Bottom of `main.py` | Command |
|---|---|---|
| **Local CLI test** | comment out `app.run()`, uncomment `main()` | `uv run main.py '{"prompt": "...", "customer_id": "CUST-123", "session_id": "s1"}'` |
| **Deploy** | `app.run()` active (as committed) | `agentcore deploy` |

> Revert to `app.run()` before deploying — `agentcore deploy` needs the ASGI server.

## Deploy and invoke

```bash
cd starter
agentcore configure     # first time only (writes .bedrock_agentcore.yaml)
agentcore deploy
agentcore invoke '{"prompt": "Hello, what can you help me with?", "customer_id": "CUST-123", "session_id": "test-1"}'
```

The committed `.bedrock_agentcore.yaml` records the deployment: agent
`ecommerce_support_agent`, direct code deploy, arm64, PUBLIC network, HTTP protocol,
observability enabled.

## IAM scoping for the deployed agent

The first deploy creates an execution role
`AmazonBedrockAgentCoreSDKRuntime-<region>-<suffix>`. It only auto-grants logs,
X-Ray, model invocation, the memory the toolkit itself created and Code Interpreter.
Extend its inline policy with:

| Permission | Resource |
|---|---|
| `bedrock-agentcore:GetMemory`, `CreateEvent`, `RetrieveMemoryRecords`, … | the exact Memory ARN in `MEMORY_ID` |
| `bedrock:Retrieve` | the Knowledge Base ARN |
| `StartBrowserSession`, `StopBrowserSession`, `GetBrowserSession`, `ListBrowserSessions`, `ConnectBrowserAutomationStream`, `GetBrowser`, `ListBrowsers` | `browser/aws.browser.v1` |

Why this matters: **locally the agent runs with your broad credentials**, so
everything "works". In the cloud the same calls fail with `AccessDeniedException`,
and the tools' fallbacks hide the failure behind plausible answers. Always check
what the deployed agent actually does:

```bash
aws logs tail /aws/bedrock-agentcore/runtimes/<agent>-DEFAULT --since 1h
```

If a tool degrades to its fallback once deployed, check IAM before assuming a code bug.
