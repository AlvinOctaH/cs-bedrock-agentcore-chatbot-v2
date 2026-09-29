# 01 — Setup: Tools, Credentials, Project Initialisation

## 1. Local prerequisites

| Tool | Version | Check |
|---|---|---|
| Python | 3.14+ (per `pyproject.toml`) | `python --version` |
| [uv](https://docs.astral.sh/uv/) | latest | `uv --version` |
| AWS CLI | v2, region **us-east-1** | `aws --version` |
| AgentCore CLI (`agentcore`) | from `bedrock-agentcore-starter-toolkit` | `agentcore --help` |
| Node.js | 18+ (only for MCP Inspector) | `node --version` |

## 2. AWS account and credentials

The account needs permission to create IAM roles/policies, Lambda functions, API
Gateway REST APIs, Bedrock Knowledge Bases (+ S3), AgentCore resources (Runtime,
Gateway, Memory) and CloudWatch. Everything is created in **us-east-1**.

```bash
aws configure                       # or paste temporary Udacity lab credentials
aws sts get-caller-identity         # must print your Account ID
```

Udacity lab credentials are temporary (access key + secret + session token) and
expire after a few hours — reconfigure them when you see `ExpiredToken`.

## 3. Model access

Bedrock console → **Model access** → enable the current **Amazon Nova Lite**
generation. `main.py` pins the exact model id: `global.amazon.nova-2-lite-v1:0`.

## 4. Project initialisation

```bash
uv init customer-support-agent
cd customer-support-agent
uv add strands-agents strands-agents-tools
uv add bedrock-agentcore bedrock-agentcore-starter-toolkit
```

In this repo the dependencies are already declared in
[`starter/pyproject.toml`](../starter/pyproject.toml) and pinned in `starter/uv.lock`:

```bash
cd starter
uv sync
```

## 5. Configuration values in `main.py`

Four values are collected while building the infrastructure (docs 02–05) and
pasted at the top of `starter/main.py`:

| Constant | Where it comes from | Format |
|---|---|---|
| `GATEWAY_URL` | Gateway console ([03](03-gateway.md)) | `https://<alias>.gateway.bedrock-agentcore.us-east-1.amazonaws.com/mcp` |
| `KB_ID` | Knowledge Base console ([04](04-knowledge-base.md)) | 10 alphanumeric characters |
| `REGION` | — | `us-east-1` |
| `MEMORY_ID` | Memory console ([05](05-memory.md)) | `CustomerSupportMemory-XXXXXXXXXX` |

## 6. Build order

```
01 setup → 02 Lambdas → 03 Gateway → 04 Knowledge Base → 05 Memory
        → 06 tools in main.py → 07 entrypoint + deploy + IAM → 08 tests
```

Infrastructure (02–05) comes **before** agent code, because the code needs the IDs.
