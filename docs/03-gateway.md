# 03 — AgentCore Gateway (MCP)

## Concept

The **AgentCore Gateway** turns backend APIs and Lambda functions into **MCP tools**
behind one managed endpoint (`.../mcp`). The agent connects with an MCP client and
**discovers** the tools at runtime — tool code is deployed, versioned and secured
independently of the agent.

## Steps (AWS Console)

1. **Amazon Bedrock → AgentCore → Gateways → Create gateway**, name `CustomerSupportGateway`.
2. Add two **targets**:

   | Target name | Lambda | Integration |
   |---|---|---|
   | `order_tracker` | `order-tracker` | API Gateway REST proxy |
   | `refund_processor` | `refund-processor` | Direct Lambda invocation |

3. For `order_tracker`, create an API Gateway REST API with proxy integration to the
   Lambda and these routes:
   - `GET /orders/{order_id}`
   - `GET /customers/{customer_id}/orders`
   - `GET /customers/{customer_id}`
4. For `refund_processor`, import the tool schema from `starter/lambda/lambda_schema`.
5. Copy the **Gateway URL** (ends with `/mcp`) into `GATEWAY_URL` in `main.py`.

## Verify with MCP Inspector

```bash
npx @modelcontextprotocol/inspector
# Connect to the Gateway URL → "List tools" → both targets' tools must appear
```

## Code: connect with a transport factory

```python
from strands.tools.mcp.mcp_client import MCPClient
from mcp.client.streamable_http import streamable_http_client

mcp_link = MCPClient(lambda: streamable_http_client(GATEWAY_URL))
gateway_tools = await mcp_link.load_tools()
tools_list.extend(gateway_tools)
```

`MCPClient` expects a **zero-argument factory** it can call to open a fresh
streamable-HTTP session on demand — not a URL keyword and not an already-opened
connection. This keeps the connection lifecycle inside `load_tools()` and avoids
leaking sockets between requests. The call is wrapped in `try/except` so the agent
still answers (with its local tools) if the Gateway is unreachable.

In this repo: `starter/main.py:458-463` (inside `invoke()`).
