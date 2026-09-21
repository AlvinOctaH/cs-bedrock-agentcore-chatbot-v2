# Reflection

One integration decision worth highlighting is how the agent connects to the MCP
Gateway. I used `MCPClient(lambda: streamable_http_client(GATEWAY_URL))` rather than
passing a URL directly to the client, because Strands' `MCPClient` expects a
zero-argument transport factory it can call to open a fresh streamable-HTTP session on
demand, not a pre-opened connection. This keeps the Gateway connection lifecycle owned
entirely by `MCPClient.load_tools()`, so the two Gateway-backed tools (the API-based
`order-tracker` target and the Lambda-based `refund-processor` target) load reliably on
every invocation without leaking sockets between requests.

The most concrete challenge I ran into was making cross-session memory actually work.
`event.agent.messages` turned out to be plain Bedrock Converse-format dictionaries
(`{"role": ..., "content": [{"text": ...}]}` or `{"toolUse": ...}`), not objects — so
early attribute-based checks like `getattr(msg, "role", ...)` silently returned nothing
and the memory hooks never fired. I fixed this by writing small dict-aware helpers to
extract plain text and detect tool-call messages, and by passing `hooks=[memory_hook]`
directly into the `Agent()` constructor instead of assigning `agent.hooks` after
construction, since hook registration only happens during `__init__`.

For production considerations, the project surfaced a real least-privilege problem: the
deployed runtime's IAM execution role only had permission for one memory resource and
no permission at all for the Knowledge Base's `Retrieve` action or the browser tool's
session APIs. Locally the agent looked fully functional because my own broader AWS
credentials masked the gap; in the cloud, every one of those calls failed with
`AccessDeniedException` and the agent silently fell back to canned responses. That's a
scalability and monitoring risk in a real deployment — a permission gap can hide behind
a graceful fallback and go unnoticed by users or dashboards. The fix was to scope the
execution role's policy explicitly to each resource ARN (memory, knowledge base,
browser) instead of relying on broad access, and to make sure failures are logged
loudly enough that a monitoring alarm would catch them instead of silently degrading.
