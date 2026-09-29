# 00 — Overview: What Was Built and Why

These notes explain the **Customer Support AI Agent** project (Udacity AWS Agentic AI —
Course 2) step by step, so it can be rebuilt from scratch without help.

| File | Contents |
|---|---|
| [00-overview.md](00-overview.md) | Concepts, architecture, request flow (this file) |
| [01-setup.md](01-setup.md) | Tools, credentials, project initialisation, model access |
| [02-lambda-backends.md](02-lambda-backends.md) | Deploying the two provided Lambda functions |
| [03-gateway.md](03-gateway.md) | AgentCore Gateway (MCP) with API-Gateway and Lambda targets |
| [04-knowledge-base.md](04-knowledge-base.md) | Bedrock Knowledge Base + the `search_knowledge_base` tool |
| [05-memory.md](05-memory.md) | AgentCore Memory resource + the `MemoryHook` |
| [06-code-interpreter-and-browser.md](06-code-interpreter-and-browser.md) | Loyalty discount tool (Code Interpreter) + Browser tool |
| [07-entrypoint-and-deployment.md](07-entrypoint-and-deployment.md) | `invoke()` entrypoint, deploy, IAM scoping |
| [08-testing-and-submission.md](08-testing-and-submission.md) | The six functional tests, screenshots, checklist, clean-up |
| [09-standout.md](09-standout.md) | Stand-out extensions |
| [10-troubleshooting.md](10-troubleshooting.md) | Every real error hit during the build, with the fix |

---

## 1. The business problem

An online store (Amazon-style catalog) wants a support assistant that answers product
and policy questions, looks up orders, processes refunds, remembers customers between
conversations and does exact loyalty-discount maths — deployed as a managed,
production-ready service.

## 2. Architecture: one agent, many managed capabilities

Unlike project 3 (multi-agent), this project is a **single Strands agent** whose
power comes from **AgentCore's managed building blocks**:

```
Customer ── agentcore invoke ──► AgentCore Runtime (BedrockAgentCoreApp, main.py)
                                   │  Strands Agent · Amazon Nova 2 Lite
                                   │  hooks=[MemoryHook]
      ┌────────────────┬───────────┼────────────────┬──────────────────┐
      ▼                ▼           ▼                ▼                  ▼
 AgentCore Gateway   Bedrock     AgentCore       AgentCore Code     AgentCore
 (MCP)               Knowledge   Memory          Interpreter        Browser
  ├─ order-tracker   Base (RAG)  (semantic facts (sandboxed Python  (live web
  │  Lambda via      product     + preferences,  loyalty maths)     pages)
  │  API Gateway     catalog     per customer)
  └─ refund-processor
     Lambda (direct)
```

| Component | Purpose in this project |
|---|---|
| **AgentCore Runtime** | Hosts the agent (`BedrockAgentCoreApp`, `@app.entrypoint`), deployed with `agentcore deploy` |
| **Amazon Nova 2 Lite** | The model (`global.amazon.nova-2-lite-v1:0`) |
| **AgentCore Gateway (MCP)** | Exposes backend Lambdas as MCP tools the agent discovers at runtime |
| **Bedrock Knowledge Base** | RAG over `product_catalog.txt` (products, returns, loyalty rules) |
| **AgentCore Memory** | Long-term memory: facts + preferences per customer across sessions |
| **Code Interpreter** | Runs the discount calculation in a secure sandbox (exact arithmetic) |
| **Browser** | Lets the agent read live web pages |

## 3. The tools the agent can use

| Tool | Source | Example question |
|---|---|---|
| Order lookup (`GET /orders/{id}`, customer orders, profile) | Gateway → `order-tracker` Lambda via API Gateway | "Where is order ORD-001?" |
| `initiate_refund`, `check_refund_status`, `get_return_label` | Gateway → `refund-processor` Lambda (direct) | "Refund my Kindle (ORD-002)" |
| `search_knowledge_base` | Local `@tool` → Bedrock KB `Retrieve` | "Is the Kindle waterproof?" |
| `calculate_loyalty_discount` | Local `@tool` → Code Interpreter | "Gold, 4250 points, $150 order?" |
| `browser` | `AgentCoreBrowser` | "What is the title of amazon.com?" |

## 4. Request flow (refund example)

1. `agentcore invoke '{"prompt": "...refund ORD-002", "customer_id": "CUST-123", "session_id": "t2"}'`.
2. `invoke()` builds a `MemoryHook` for `CUST-123` / `t2`, the browser tool, connects
   the `MCPClient` to the Gateway and loads its tools.
3. Long-term memories for the customer are retrieved and prepended as
   "Customer Context".
4. The agent looks up the order first (real total $139.99), then calls
   `initiate_refund` with that amount → `REF-XXXXXXXX`.
5. After the answer, `save_support_interaction` stores the turn as a Memory event;
   AgentCore extracts facts/preferences from it asynchronously.

## 5. Key concepts learned

- **MCP (Model Context Protocol) Gateway**: tools live behind a managed endpoint
  instead of inside the agent code — they can be versioned and shared.
- **Hooks**: code that runs on agent lifecycle events (`MessageAddedEvent`,
  `AfterInvocationEvent`) — used here to read and write long-term memory.
- **Short-term vs long-term memory**: events (raw turns) vs extracted records
  (facts, preferences) in namespaces like `cs_agent/{actorId}/facts`.
- **Least-privilege runtime role**: the deployed agent runs under its own IAM role,
  not your credentials — every resource it touches must be granted explicitly.
