# 09 — Stand-out Extensions

| Extension | Where | What it adds |
|---|---|---|
| **Structured output validation (Pydantic)** | `DiscountOutputSchema`, `starter/main.py:317-321`; validated at `:385-387` | The Code Interpreter's stdout is parsed and validated against a typed schema (`points_redeemed: int`, `tier_discount_pct: float`, `final_total: float`, `remaining_points: int`) before it reaches the model. Malformed sandbox output falls through to the deterministic fallback instead of producing a wrong number. |
| **Conversation summarisation / history trimming** | `invoke()`, `starter/main.py:480-487` (search `Condensing conversational state`) | When the message history exceeds 15 entries, it keeps the first message plus the last four — protecting the token budget and latency in long conversations. |
| **Grounded fallbacks** | `search_knowledge_base` `:290-297`, `calculate_loyalty_discount` `:391-413` | If the KB `Retrieve` or the sandbox is blocked (e.g. a lab IAM deny), the tools answer from the catalog's actual rules instead of erroring. The same rules as the primary path, so answers stay correct. |
| **Safe memory-context enrichment** | `invoke()`, `starter/main.py:489-515` | Long-term memories are retrieved up front for the user's prompt and prepended as "Customer Context", in addition to the hook — each namespace in its own `try/except`, so one failing namespace never breaks the request. |

## Caveat worth remembering

Fallbacks improve availability but can **hide** a broken integration: in the cloud,
IAM gaps made every tool silently fall back while answers still looked plausible
(see [10-troubleshooting](10-troubleshooting.md) #12). In production, pair fallbacks
with loud logging and an alarm on fallback frequency.
