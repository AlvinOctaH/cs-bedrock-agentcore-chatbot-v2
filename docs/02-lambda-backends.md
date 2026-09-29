# 02 — Deploying the Lambda Backends

The two Lambda functions in `starter/lambda/` are **provided backend infrastructure**.
Deploy them as-is — the project is about integrating with them, not changing them.

| Function | File | How the agent reaches it | Tools / routes |
|---|---|---|---|
| `order-tracker` | `order_tracker.py` | Gateway → **API Gateway REST proxy** → Lambda | `GET /orders/{order_id}`, `GET /customers/{customer_id}/orders`, `GET /customers/{customer_id}` |
| `refund-processor` | `refund_processor.py` | Gateway → **Lambda directly** | `initiate_refund`, `check_refund_status`, `get_return_label` (schema in `lambda_schema`) |

Mock data inside `order_tracker.py`:

| Order | Customer | Status | Item | Total |
|---|---|---|---|---|
| ORD-001 | CUST-123 (Jane Smith, Gold, 4250 pts) | SHIPPED | Wireless Headphones Pro | $89.99 |
| ORD-002 | CUST-123 | DELIVERED (3 days ago) | Kindle Paperwhite | $139.99 |
| ORD-003 | CUST-456 (Bob Johnson, Silver, 890 pts) | PROCESSING | 2 × Echo Dot + Smart Plug | $124.97 |

## Steps (AWS Console)

1. **Lambda → Create function** → *Author from scratch*.
   - Name: `order-tracker`, runtime **Python 3.12+**, architecture default.
   - Execution role: *Create a new role with basic Lambda permissions* (CloudWatch Logs).
2. In the code editor, replace `lambda_function.py` with the contents of
   `order_tracker.py` (or zip and upload it) → **Deploy**.
3. Repeat for `refund-processor` with `refund_processor.py`.
4. Note both **function ARNs** (needed for the Gateway targets).

Quick test from the console (*Test* tab) for `order-tracker`:

```json
{ "httpMethod": "GET", "resource": "/orders/{order_id}",
  "pathParameters": { "order_id": "ORD-001" } }
```

## How the refund Lambda knows which tool was called

With a direct Lambda target, the Gateway passes the tool name in the Lambda client
context under `bedrockAgentCoreToolName`, formatted as `TargetName___toolName`.
`refund_processor.py` strips the prefix and branches on the bare tool name.

## Important: the refund amount

In `lambda_schema`, `amount` is **optional** for `initiate_refund`. If the agent doesn't
pass it, the refund comes back as `$0`. The fix belongs in the **agent's system
prompt** (look up the order first, pass its real total) — not in the Lambda, which
is provided infrastructure. See [07](07-entrypoint-and-deployment.md) and
[10 — troubleshooting #11](10-troubleshooting.md).
