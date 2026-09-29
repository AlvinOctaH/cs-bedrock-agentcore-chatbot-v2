# 06 — Code Interpreter (Loyalty Discount) and Browser Tools

## Part A — `calculate_loyalty_discount` (Code Interpreter)

### Why a sandbox?

LLMs are unreliable at arithmetic. The **AgentCore Code Interpreter** runs real Python
in an isolated sandbox, so the discount is computed exactly and the model only
explains the result.

### Business rules (from the product catalog)

| Rule | Value |
|---|---|
| Tier discount | Silver 5% · Gold 10% · Platinum 15% (applied first) |
| Points value | 100 points = $1 |
| Minimum / step | Redeem in multiples of 500 points |
| Cap | Points discount ≤ 50% of the subtotal after the tier discount |

### Implementation (`starter/main.py:317-413`)

1. Build a **self-contained Python code string** with the rules and the customer's
   inputs, which prints a JSON result.
2. Run it in the sandbox:
   ```python
   with code_session(REGION) as session:
       resp = session.invoke("executeCode",
                             {"code": code, "language": "python", "clearContext": True})
       for event in resp["stream"]:
           structured = event.get("result", {}).get("structuredContent", {})
           if structured.get("stdout"):
               stdout_output = structured["stdout"]
   ```
   `invoke(method, params)` takes **one params dict**, and the result is an **event
   stream** — stdout is read from `structuredContent`.
3. **Validate** the output with a Pydantic model (`DiscountOutputSchema`:
   `points_redeemed`, `tier_discount_pct`, `final_total`, `remaining_points`).
4. **Fallback**: if the sandbox is unavailable, compute the same rules in plain Python.

### Worked example (test 5)

Gold member, 4250 points, $150 standard order:

```
tier discount 10%         → subtotal $135.00
50% cap                   → max $67.50 = 6750 points
usable points             → min(4250, 6750) = 4250 → floor to 500s = 4000
points discount           → $40.00
final total               → $95.00      remaining points → 250
```

```bash
agentcore invoke '{"prompt": "I am a Gold member with 4250 points. Calculate my discount on a $150 standard order.", "customer_id": "CUST-123", "session_id": "t5"}'
```

## Part B — Browser tool

```python
from strands_tools.browser import AgentCoreBrowser

agent_browser = AgentCoreBrowser(region=REGION)          # main.py:452
tools_list = [search_knowledge_base, calculate_loyalty_discount,
              agent_browser.browser]                     # main.py:455
```

The **AgentCore Browser** is a managed headless browser; the agent can open a URL and
read the page. `BYPASS_TOOL_CONSENT=true` (main.py:56) suppresses interactive consent
prompts, which would hang a headless deployment.

The agent must be invoked **asynchronously** (`await agent.invoke_async(...)`):
calling it synchronously inside the async entrypoint breaks the browser tool's
task-scoped timeouts (*"Timeout should be used inside a task"*).

```bash
agentcore invoke '{"prompt": "Go to https://www.udacity.com and tell me the page title.", "customer_id": "CUST-123", "session_id": "t6"}'
```

The deployed runtime role needs the browser session permissions — see
[07](07-entrypoint-and-deployment.md#iam-scoping-for-the-deployed-agent).
