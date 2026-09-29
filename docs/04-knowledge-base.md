# 04 — Bedrock Knowledge Base (RAG)

## Concept

A **Knowledge Base** is Bedrock's managed RAG: point it at documents in S3 and it
chunks, embeds and indexes them. `bedrock-agent-runtime.retrieve()` returns the most
relevant passages for a question. Here it holds `starter/product_catalog.txt`:
products (specs, warranty, return rules), return & refund policy, the loyalty
program (tiers, earning, redemption, benefits) and order-status definitions.

## Steps (AWS Console)

1. Upload `product_catalog.txt` to an S3 bucket in your account.
2. **Bedrock → Knowledge Bases → Create**, name `CustomerSupportKB`, data source =
   that bucket.
3. **Sync** the data source.
4. Copy the **Knowledge Base ID** into `KB_ID` in `main.py`.
5. Confirm the KB and **its type** before writing the tool:
   ```bash
   aws bedrock-agent list-knowledge-bases --region us-east-1
   aws bedrock-agent get-knowledge-base --knowledge-base-id <id> --region us-east-1
   ```

Console check: KB → **Test** → *"What is the return policy for electronics?"* →
expected: a 15-day return window.

## The retrieval configuration depends on the KB type

| KB type (`knowledgeBaseConfiguration.type`) | `retrievalConfiguration` |
|---|---|
| **MANAGED** (Bedrock-managed store) | `{"managedSearchConfiguration": {"numberOfResults": 3}}` |
| **VECTOR** (self-managed, e.g. OpenSearch Serverless, S3 Vectors) | `{"vectorSearchConfiguration": {"numberOfResults": 3}}` |

Using the wrong one fails with *"vectorSearchConfiguration is not supported for
managed knowledge bases"*. This project's KB is **managed**.

## The tool (`starter/main.py:261-297`)

```python
@tool
def search_knowledge_base(query: str) -> str:
    """Search the Amazon product catalog and support knowledge base. ..."""
    if not KB_ID ...:                       # guard clause (rubric)
        return "Knowledge base not configured. Fallback data: ..."
    response = _bedrock_runtime.retrieve(
        knowledgeBaseId=KB_ID,
        retrievalQuery={"text": query},
        retrievalConfiguration={"managedSearchConfiguration": {"numberOfResults": 3}},
    )
    chunks = [r["content"]["text"] for r in response["retrievalResults"] if ...]
    return "\n---\n".join(chunks)
```

- The **docstring** tells the model when to use the tool (product specs, returns,
  warranty, loyalty, order statuses).
- **Guard clause** returns a descriptive message when `KB_ID` isn't configured.
- If `Retrieve` fails (e.g. a lab IAM restriction), a **grounded fallback** returns
  the relevant catalog rules instead of an error (see [09-standout](09-standout.md)).

Test:
```bash
agentcore invoke '{"prompt": "Is the Kindle Paperwhite waterproof?"}'
```
