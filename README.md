# Project: Building a Production-Grade Customer Support AI Agent with Amazon Bedrock AgentCore

**Udacity — AWS AI Engineering Nanodegree — Course 2**

---

## Overview

In this project you will build a fully functional, production-ready AI customer support agent for a fictional Amazon store. Starting from a simple local chatbot, you will progressively add cloud infrastructure, external tool integration, a knowledge base, persistent memory, a code interpreter, and a browser — finishing with a deployable agent that can handle real customer inquiries end-to-end.

By the end of the project your agent will be able to:

- Answer questions about products, return policies, and loyalty rewards using Retrieval-Augmented Generation (RAG)
- Look up order status and process refunds by calling Lambda functions through the AgentCore Gateway
- Remember customer preferences and conversation history across multiple sessions
- Calculate exact loyalty discounts using a secure code sandbox
- Navigate websites to fetch live information

> 💡 This copy of the instructions has been annotated with **⚠️ Gotcha** call-outs
> at each step where a real implementation ran into trouble, so a fresh build
> can go straight through without repeating the same mistakes.

---

## Learning Objectives

After completing this project you will be able to:

1. Deploy an AI agent to Amazon Bedrock AgentCore
2. Wire up external Lambda tools via the AgentCore Gateway using the Model Context Protocol (MCP)
3. Implement RAG with a Bedrock Knowledge Base
4. Add short-term (session) and long-term (cross-session) memory using AgentCore Memory
5. Use the AgentCore Code Interpreter for precise computation
6. Integrate the AgentCore Browser tool for live web access
7. Monitor and observe agent behaviour with Amazon CloudWatch

---

## Prerequisites

### AWS Account

- An active AWS account with permission to create and manage:
  - IAM roles and policies
  - Lambda functions
  - API Gateway REST APIs
  - Amazon Bedrock Knowledge Bases (with S3 and OpenSearch access)
  - Amazon Bedrock AgentCore resources (Runtime, Gateway, Memory)
  - Amazon CloudWatch
- All resources should be created in **us-east-1** (N. Virginia) unless stated otherwise.

### Local Development Environment

| Tool | Version |
|------|---------|
| Python | 3.14+ |
| [uv](https://docs.astral.sh/uv/) | Latest |
| AWS CLI | v2 |
| AgentCore CLI (`agentcore`) | Installed via the starter-toolkit |
| Node.js (for MCP Inspector) | 18+ |

### Model Access

Enable the following models in the Amazon Bedrock console under **Model access**:

- **Amazon Nova Lite** (or the current Nova Lite generation available in your account — `main.py` pins the exact `model_id`)

---

## Project Structure

```
project/
├── INSTRUCTIONS.md          ← this file
├── RUBRIC.md                ← grading criteria
├── starter/
│   ├── main.py              ← your starting point (fill in the TODOs)
│   ├── README.md             ← submission overview + links to test evidence
│   ├── screenshots/          ← test 1-6 evidence
│   ├── REFLECTION.md         ← written reflection
│   └── lambda/
│       ├── order_tracker.py     ← provided; deploy as-is
│       └── refund_processor.py  ← provided; deploy as-is
└── solution/                ← reference implementation (do not copy)
    ├── main.py
    ├── product_catalog.txt
    ├── pyproject.toml
    ├── lambda/
    │   ├── order_tracker.py
    │   ├── refund_processor.py
    │   └── lambda_schema       ← JSON schema for Gateway tool registration
    └── step-by-step/           ← one file per build step (for reference)
```

---

## Part 1 — AWS Infrastructure Setup

Complete these steps **before** writing any agent code.

### Step 1.1 — Project Initialisation

```bash
# Create a new Python project managed by uv
uv init customer-support-agent
cd customer-support-agent

# Install core dependencies
uv add strands-agents strands-agents-tools
uv add bedrock-agentcore bedrock-agentcore-starter-toolkit
```

### Step 1.2 — Deploy the Lambda Functions

The two Lambda functions (`order_tracker.py` and `refund_processor.py`) are provided in `starter/lambda/`. Deploy them to AWS Lambda before proceeding.

1. In the AWS Lambda console, create two new functions (Python 3.12+ runtime):
   - `order-tracker`
   - `refund-processor`
2. Paste the contents of each file into the inline code editor (or zip and upload).
3. Attach an execution role with basic Lambda permissions (CloudWatch Logs).
4. Note the ARN of each function — you will need them in the next step.
5. **Do not modify these two files.** They are pre-built backend infrastructure the
   agent calls as external tools — the project is about integrating with them
   correctly, not changing their behavior.

### Step 1.3 — Set Up the AgentCore Gateway

The Gateway exposes your Lambda functions as MCP tools that the agent can call.

1. Open the **Amazon Bedrock** console → **AgentCore** → **Gateways**.
2. Create a new Gateway named `CustomerSupportGateway`.
3. Add two **Lambda targets**:

   | Target Name | Lambda Function | Integration |
   |---|---|---|
   | `order_tracker` | `order-tracker` | API Gateway REST proxy |
   | `refund_processor` | `refund-processor` | Direct Lambda invocation |

4. For `order_tracker`, configure API Gateway routes:
   - `GET /orders/{order_id}`
   - `GET /customers/{customer_id}/orders`
   - `GET /customers/{customer_id}`

5. For `refund_processor`, import the tool schema from `solution/lambda/lambda_schema`.

6. Copy the **Gateway URL** (ends with `/mcp`) — paste it into `GATEWAY_URL` in your `main.py`.

**Verify with MCP Inspector:**
```bash
npx @modelcontextprotocol/inspector
# Connect to your Gateway URL and confirm all tools are listed.
```

> ⚠️ **Gotcha — connecting `MCPClient` to the Gateway.** `MCPClient` does not
> take a `gateway_url=` kwarg, and you don't need to open your own
> `async with streamable_http_client(...)` block. Construct it with a
> **transport factory** instead, and call the async `load_tools()`:
> ```python
> mcp_link = MCPClient(lambda: streamable_http_client(GATEWAY_URL))
> gateway_tools = await mcp_link.load_tools()
> ```
> Getting this wrong shows up as `MCP Gateway target could not load tools
> dynamically` in the logs, with both Gateway tools silently missing from
> the agent's tool list.

### Step 1.4 — Create the Knowledge Base

1. Upload `solution/product_catalog.txt` to an **S3 bucket** in your account.
2. In the Bedrock console → **Knowledge Bases**, create a new Knowledge Base:
   - Name: `CustomerSupportKB`
   - Data source: the S3 bucket from above
   - Embeddings model: Amazon Titan Embeddings v2
   - Vector store: Amazon OpenSearch Serverless (auto-created), **or** a Bedrock-managed vector store
3. **Sync** the data source.
4. Copy the **Knowledge Base ID** — paste it into `KB_ID` in your `main.py`.

**Verify:**
```bash
# In the console, use the Knowledge Base "Test" tab
# Query: "What is the return policy for electronics?"
# Expected: 15-day return window for electronics
```

> ⚠️ **Gotcha — wrong or stale `KB_ID`.** If `Retrieve` fails with
> `ResourceNotFoundException`, double-check the ID against what's actually
> live in your account:
> ```bash
> aws bedrock-agent list-knowledge-bases --region us-east-1
> ```
>
> ⚠️ **Gotcha — `vectorSearchConfiguration is not supported for managed
> knowledge bases`.** Check the KB type first:
> ```bash
> aws bedrock-agent get-knowledge-base --knowledge-base-id <id> --region us-east-1
> ```
> If `knowledgeBaseConfiguration.type` is `MANAGED`, your `retrieve()` call
> must use `retrievalConfiguration={"managedSearchConfiguration": {"numberOfResults": 3}}`,
> not `vectorSearchConfiguration`. Getting this wrong makes `search_knowledge_base`
> silently fall back to canned text instead of a real KB answer — it still
> "works" from the user's point of view, which is exactly why it's easy to miss.

### Step 1.5 — Create the AgentCore Memory Resource

1. In the Bedrock console → **AgentCore** → **Memory**, create a new Memory resource:
   - Name: `CustomerSupportMemory`
2. Add two **Memory Strategies**:

   | Strategy | Name | Namespace |
   |---|---|---|
   | Semantic extraction | `customer_facts` | `cs_agent/{actorId}/facts` |
   | User preference | `customer_preferences` | `cs_agent/{actorId}/preferences` |

3. Copy the **Memory ID** — paste it into `MEMORY_ID` in your `main.py`.

> ⚠️ **Gotcha — `get_memory_strategies()` / `retrieve_memories()` response
> shapes.** Both return a **list** directly, not a dict wrapper — code like
> `response.get("strategies", [])` or `resp.get("memories", [])` will crash
> with `'list' object has no attribute 'get'`. Each strategy dict has `type`
> and `namespaces`/`namespaceTemplates` (a list — take `[0]`); each memory
> record is shaped `{"content": {"text": "..."}, ...}`. Also use the SDK's
> real keyword names — `memory_id`/`namespace`/`query`/`top_k`, not
> `memoryId`/`topK`.
>
> ⚠️ **Gotcha — `create_event()` signature.** There is no `namespace`
> parameter. Call it as:
> ```python
> memory_client.create_event(
>     memory_id=..., actor_id=..., session_id=...,
>     messages=[(customer_query, "USER"), (agent_response, "ASSISTANT")],
> )
> ```

### Step 1.6 — Grant the Deployed Agent's Execution Role the Permissions It Actually Needs

> ⚠️ **Gotcha — everything works locally, then fails silently once deployed.**
> Local testing runs under your own broad AWS credentials. The agent's actual
> execution role (`AmazonBedrockAgentCoreSDKRuntime-<region>-<suffix>`,
> created the first time you run `agentcore deploy`) starts with **no
> permissions beyond what the toolkit auto-grants**: logs, X-Ray, model
> invocation, whichever memory resource the toolkit itself created, and Code
> Interpreter. It will silently throw `AccessDeniedException` on anything
> else — and because `@tool` functions in this project have graceful
> fallbacks, the agent keeps answering with plausible-looking (but fake)
> data instead of visibly failing.
>
> Before trusting a "working" cloud test, check the deployed agent's logs:
> ```bash
> aws logs tail /aws/bedrock-agentcore/runtimes/<agent>-DEFAULT --since 1h
> ```
> Then extend the execution role's inline policy to explicitly cover every
> resource the agent calls that isn't covered by the toolkit's defaults:
> - `bedrock-agentcore:{GetMemory,CreateEvent,RetrieveMemoryRecords,...}` on
>   the **exact** Memory resource ARN in `MEMORY_ID` (if it differs from
>   whichever memory the toolkit auto-created and already granted).
> - `bedrock:Retrieve` on the Knowledge Base ARN.
> - Browser tool session actions — `StartBrowserSession`, `StopBrowserSession`,
>   `GetBrowserSession`, `ListBrowserSessions`, `ConnectBrowserAutomationStream`,
>   `GetBrowser`, `ListBrowsers` — scoped to `browser/aws.browser.v1`.

---

## Part 2 — Building the Agent

Open `starter/main.py`. It contains scaffolding and `# TODO` comments marking every section you need to implement. Work through the TODOs in order.

The step-by-step reference files in `solution/step-by-step/` show the state of the code after each section is complete — consult them if you get stuck, but try to implement each section yourself first.

### Section 1 — Configuration and Initialisation

Fill in your resource IDs and set up:
- `BedrockAgentCoreApp`
- `BedrockModel` with Amazon Nova Lite
- `MemoryClient` and `boto3` Bedrock runtime client

### Section 2 — Knowledge Base Tool

Implement `search_knowledge_base(query)`:
- Call the Bedrock Knowledge Base Retrieve API
- Join result chunks with `"\n---\n"`

See the KB gotchas under Step 1.4 above (`managedSearchConfiguration` vs.
`vectorSearchConfiguration`, and verifying `KB_ID`).

**Test:**
```bash
agentcore invoke '{"prompt": "Is the Kindle Paperwhite waterproof?"}'
# Expected: mention of IPX8 rating
```

### Section 3 — Long-Term Memory Hook

Implement `MemoryHook` with two methods:
- `retrieve_customer_context` — query all memory namespaces and prepend results to the user message
- `save_support_interaction` — save the completed (user, assistant) turn after each response

> ⚠️ **Gotcha — hooks never fire.** Pass hooks into the `Agent(...)`
> constructor, not by assigning `agent.hooks` afterward:
> ```python
> agent = Agent(model=model, tools=tools_list, system_prompt=system_prompt, hooks=[memory_hook])
> ```
> Hook registration only happens during `Agent.__init__`; setting
> `agent.hooks = registry` after construction silently does nothing.
>
> ⚠️ **Gotcha — hooks fire but never read/save anything.** `event.agent.messages`
> items are **plain dicts** in Bedrock Converse format —
> `{"role": "user"/"assistant", "content": [{"text": "..."}]}` for text, or
> `{"toolUse": {...}}` / `{"toolResult": {...}}` for tool calls — not
> objects. `getattr(msg, "role", ...)` and `hasattr(msg, "tool_call_id")`
> silently return nothing useful. Write small dict-aware helpers instead —
> one to pull text out of the `content` blocks, one to detect whether a
> message is a tool call/result rather than plain text — and use those in
> both `retrieve_customer_context` and `save_support_interaction`.

### Section 4 — Loyalty Discount Tool (Code Interpreter)

Implement `calculate_loyalty_discount(loyalty_points, tier, order_total, product_category)`:
- Build a Python code string containing the discount logic
- Execute it with `code_session()` and return the JSON result
- Include a fallback for when the Code Interpreter is unavailable

> ⚠️ **Gotcha — `CodeInterpreter.invoke() got an unexpected keyword argument
> 'code'`.** `session.invoke(method, params)` takes the method name and
> **one params dict**, not separate `code=`/`language=`/`clearContext=`
> kwargs:
> ```python
> resp = session.invoke("executeCode", {"code": code, "language": "python", "clearContext": True})
> ```
> The result is an event stream, not a plain dict — read stdout from it:
> ```python
> for event in resp["stream"]:
>     structured = event.get("result", {}).get("structuredContent", {})
>     stdout_output = structured.get("stdout", stdout_output)
> ```
> Skipping this makes the tool *always* hit its fallback path, even though
> nothing looks obviously broken from the agent's replies.

**Test:**
```bash
agentcore invoke '{"prompt": "I am a Gold member with 4250 points. Calculate my discount on a $150 order.", "customer_id": "CUST-123", "session_id": "s1"}'
```

### Section 5 — Main Entrypoint

Implement the `invoke(payload, context)` function:
- Extract `prompt`, `customer_id`, and `session_id` from the payload
- Instantiate `MemoryHook` and `AgentCoreBrowser`
- Connect to the Gateway via `MCPClient` and load gateway tools
- Build the `Agent` with all tools and hooks and return its response

> ⚠️ **Gotcha — browser tool crashes with `Timeout should be used inside a
> task`.** Calling the agent synchronously (`agent(prompt)`) from inside the
> async `invoke()` entrypoint breaks anyio's task-scoped timeouts used by the
> browser tool. Since `invoke()` is already `async def`, call the agent the
> same way:
> ```python
> agent_response = await agent.invoke_async(str(enriched_user_input))
> ```
>
> ⚠️ **Gotcha — refund amount comes back as `$0`.** The agent has no way to
> know an order's real price unless it looks it up — `amount` is an
> *optional* field in the refund tool's schema, so the model can (and will)
> omit it. Add explicit guidance to the system prompt: look up the order
> first (e.g. via a `get_order` tool) and pass its real total as the refund
> amount, rather than guessing or leaving it out. Don't try to "fix" this by
> changing the Lambda — it's pre-built backend infrastructure (Step 1.2).

### Section 6 — Deploy to AgentCore

> ⚠️ **Gotcha — local CLI testing hangs / does nothing.** The bottom of
> `main.py` runs `app.run()` (an ASGI server, for deployment) by default.
> For `uv run main.py '{...}'` to work, comment out `app.run()` and
> uncomment `main()` — then **revert it before deploying**, since
> `agentcore deploy` requires `app.run()` to be active.

```bash
# Configure the AgentCore CLI (first time only)
agentcore configure

# Deploy the agent
agentcore deploy

# Invoke the deployed agent
agentcore invoke '{"prompt": "Hello, what can you help me with?", "customer_id": "CUST-123", "session_id": "test-1"}'
```

If a tool that worked locally stops working once deployed (or degrades to
its fallback), see the **Step 1.6** gotcha above before assuming it's a code
bug — check the execution role's permissions first.

---

## Part 3 — Functional Testing

Run the following test scenarios and verify the expected behaviour. Include screenshots or copy the terminal output in your submission.

### Test 1 — Order Tracking

```bash
agentcore invoke '{"prompt": "Can you track order ORD-001?", "customer_id": "CUST-123", "session_id": "t1"}'
# Expected: shipping status, tracking number TRK987654321, carrier UPS, estimated delivery
```

### Test 2 — Refund Processing

```bash
agentcore invoke '{"prompt": "I want to return my Kindle Paperwhite (ORD-002). Please initiate a refund.", "customer_id": "CUST-123", "session_id": "t2"}'
# Expected: refund ID, APPROVED status, real order total as the refund amount, 3-5 business days message
```

### Test 3 — Knowledge Base (RAG)

```bash
agentcore invoke '{"prompt": "What are the benefits of the Platinum loyalty tier?", "customer_id": "CUST-123", "session_id": "t3"}'
# Expected: free same-day shipping, 15% discount, priority support
```

### Test 4 — Memory (Long-Term)

```bash
# Session A — introduce yourself
agentcore invoke '{"prompt": "Hi, I am Jane. I prefer concise responses.", "customer_id": "CUST-123", "session_id": "s-A"}'

# Session B (new session) — verify recall
agentcore invoke '{"prompt": "Do you remember my name and communication preference?", "customer_id": "CUST-123", "session_id": "s-B"}'
# Expected: agent recalls "Jane" and "concise responses"
```

Long-term extraction runs asynchronously — if session B doesn't recall
anything yet, wait ~30-90 seconds after session A before invoking session B.

### Test 5 — Loyalty Discount Calculation

```bash
agentcore invoke '{"prompt": "I am a Gold member with 4250 points. Calculate my discount on a $150 standard order.", "customer_id": "CUST-123", "session_id": "t5"}'
# Expected: points redeemed, tier discount 10%, final total, remaining points
```

### Test 6 — Browser Tool

```bash
agentcore invoke '{"prompt": "Go to https://www.amazon.com and tell me the page title.", "customer_id": "CUST-123", "session_id": "t6"}'
# Expected: page title retrieved from live Amazon.com
```

---

## Part 4 — CloudWatch Monitoring (optional)

> This part is **not required by the current grading rubric** (see
> Submission Checklist below) — it's kept here as a good production
> practice if you want to go further, not as a submission requirement.

1. In the AWS console, navigate to **CloudWatch** → **Log Groups**.
2. Find the log group for your AgentCore Runtime (named after your deployment).
3. Create a **metric filter** on `ERROR` log entries.
4. Create a **CloudWatch Alarm** that triggers when the error count exceeds 5 in a 5-minute window.

---

## Submission Checklist

- [ ] Completed `main.py` with all TODO sections implemented (no `pass` or `None` placeholders remaining)
- [ ] Screenshots or terminal output for Test 1 — Order Tracking
- [ ] Screenshots or terminal output for Test 2 — Refund Processing
- [ ] Screenshots or terminal output for Test 3 — Knowledge Base (RAG)
- [ ] Screenshots or terminal output for Test 4 — Long-Term Memory (both sessions)
- [ ] Screenshots or terminal output for Test 5 — Loyalty Discount Calculation
- [ ] Screenshots or terminal output for Test 6 — Browser Tool
- [ ] Written reflection (200–400 words) covering a design decision, a challenge, and a production consideration

---

## Helpful References

- [Amazon Bedrock AgentCore Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html)
- [Strands Agents Documentation](https://strandsagents.com)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- [uv Package Manager](https://docs.astral.sh/uv/)
