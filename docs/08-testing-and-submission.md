# 08 — Testing, Submission & Clean-up

All tests run against the **deployed** agent with `agentcore invoke` from `starter/`.

| # | Test | Command (`agentcore invoke '<payload>'`) | Proves | Evidence |
|---|---|---|---|---|
| 1 | Order tracking | `{"prompt": "Can you track order ORD-001?", "customer_id": "CUST-123", "session_id": "t1"}` | Gateway → API Gateway → `order-tracker` Lambda | `test_1_order_tracking.png` |
| 2 | Refund processing | `{"prompt": "I want to return my Kindle Paperwhite (ORD-002). Please initiate a refund.", "customer_id": "CUST-123", "session_id": "t2"}` | Gateway → `refund-processor` Lambda, real amount $139.99 | `test_2_refund_processing.png` |
| 3 | Knowledge Base (RAG) | `{"prompt": "What are the benefits of the Platinum loyalty tier?", "customer_id": "CUST-123", "session_id": "t3"}` | `search_knowledge_base` | `test_3_knowledge_base_rag.png` |
| 4a | Memory — session A | `{"prompt": "Hi, I am Jane. I prefer concise responses.", "customer_id": "CUST-123", "session_id": "s-A"}` | `create_event` | `test_4a_memory_session_A.png` |
| 4b | Memory — session B (after ~30–90 s) | `{"prompt": "Do you remember my name and communication preference?", "customer_id": "CUST-123", "session_id": "s-B"}` | Cross-session recall → "Jane, concise" | `test_4b_memory_session_B.png` |
| 5 | Loyalty discount | `{"prompt": "I am a Gold member with 4250 points. Calculate my discount on a $150 standard order.", "customer_id": "CUST-123", "session_id": "t5"}` | Code Interpreter → $95.00, 4000 pts redeemed, 250 left | `test_5_loyalty_discount.png` |
| 6 | Browser | `{"prompt": "Go to https://www.udacity.com and tell me the page title.", "customer_id": "CUST-123", "session_id": "t6"}` | AgentCore Browser | `test_6_browser_tool.png` |

All screenshots are in [`screenshots/`](../screenshots/).

## Reflection

[`REFLECTION.md`](../REFLECTION.md) — 200–400 words: a design decision (Gateway
transport factory), a challenge (memory hooks with dict messages) and a production
consideration (least-privilege runtime role hidden by fallbacks).

## Submission checklist

- [x] `starter/main.py` — every TODO implemented (no `pass` / `None` placeholders)
- [x] Tests 1–6 evidence (screenshots; test 4 has both sessions)
- [x] Reflection (200–400 words)

## Clean-up (after grading)

Delete, in the AWS Console (or with the CLI):

1. The AgentCore Runtime (`agentcore destroy` from `starter/`, or console → AgentCore → Runtime)
2. The Gateway `CustomerSupportGateway` and its targets
3. The Memory `CustomerSupportMemory`
4. The Knowledge Base `CustomerSupportKB` (+ its vector store) and the S3 bucket
5. The API Gateway REST API and the two Lambda functions
6. The runtime execution role and its inline policy
