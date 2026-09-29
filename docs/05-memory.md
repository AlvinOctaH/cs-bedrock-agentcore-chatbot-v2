# 05 — AgentCore Memory + the MemoryHook

## Concept

**AgentCore Memory** stores conversation **events** (raw turns, short-term) and
extracts **memory records** (long-term) from them with *strategies*:

| Strategy | Name | Namespace | Extracts |
|---|---|---|---|
| Semantic | `customer_facts` | `cs_agent/{actorId}/facts` | Facts ("the customer's name is Jane") |
| User preference | `customer_preferences` | `cs_agent/{actorId}/preferences` | Preferences ("prefers concise responses") |

`{actorId}` is the customer ID, so every customer has their own memory space that
persists **across sessions**. Extraction runs asynchronously (~30–90 s after an event).

## Steps (AWS Console)

1. **Bedrock → AgentCore → Memory → Create memory**, name `CustomerSupportMemory`.
2. Add the two strategies from the table above.
3. Copy the **Memory ID** into `MEMORY_ID` in `main.py`.

## SDK shapes that matter (learned the hard way)

```python
strategies = memory_client.get_memory_strategies(memory_id)      # → a LIST
# each: {"type": "SEMANTIC", "namespaces" or "namespaceTemplates": ["cs_agent/{actorId}/facts"], ...}

records = memory_client.retrieve_memories(                        # → a LIST
    memory_id=..., namespace="cs_agent/CUST-123/facts", query="...", top_k=3)
# each: {"content": {"text": "..."}, ...}

memory_client.create_event(                                       # no namespace argument
    memory_id=..., actor_id=..., session_id=...,
    messages=[(customer_query, "USER"), (agent_response, "ASSISTANT")])
```

## Implementation in `starter/main.py`

| Piece | Lines | What it does |
|---|---|---|
| `get_namespaces()` | 106-122 | Strategy type → namespace template (`namespaceTemplates` or `namespaces`), with a fallback layout |
| `_message_text()` / `_is_tool_message()` | 151-166 | Dict-aware helpers: messages are Converse-format **dicts**, not objects |
| `MemoryHook(HookProvider)` | 169-243 | The hook provider |
| `retrieve_customer_context` | 180-208 | On a plain-text user message: query every namespace, tag results by strategy, prepend "Customer Context:" |
| `save_support_interaction` | 210-239 | After the invocation: find the last user query + assistant reply, `create_event()` |
| `register_hooks` | 241-243 | `MessageAddedEvent` → retrieve, `AfterInvocationEvent` → save |

Pass the hook **in the constructor** — hooks are registered only during `__init__`:

```python
agent = Agent(model=model, tools=tools_list, system_prompt=system_prompt, hooks=[memory_hook])
```

## Test (two sessions, same customer)

```bash
# Session A
agentcore invoke '{"prompt": "Hi, I am Jane. I prefer concise responses.", "customer_id": "CUST-123", "session_id": "s-A"}'
# wait ~30-90 s for long-term extraction, then a NEW session:
agentcore invoke '{"prompt": "Do you remember my name and communication preference?", "customer_id": "CUST-123", "session_id": "s-B"}'
# → "Yes, I remember your name is Jane, and you prefer concise responses."
```
