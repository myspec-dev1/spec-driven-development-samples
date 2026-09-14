# Solution Architecture: Enterprise AI Gateway & Agentic Policy Enforcement Platform

## 1. Architectural Overview

The solution consists of three primary planes:
1. **Data Plane (High-Performance Proxy)**: Built on Rust (Axum / Tokio) or Go (FastHTTP), maintaining persistent HTTP/2 and SSE connections with sub-10ms processing latency.
2. **Policy & Inspection Engine**: An extensible filter pipeline executing regex heuristics, Presidio PII masks, vector safety embeddings, and MCP tool call verifications.
3. **Control & Telemetry Plane**: NestJS / PostgreSQL backend with Redis Cluster for rate limits, ClickHouse for high-cardinality telemetry, and S3 WORM for immutable audits.

```
                    +----------------------------------------------+
                    |           Client Apps & Agents               |
                    +----------------------------------------------+
                                          | (TLS 1.3 / SSE)
                                          v
+-----------------------------------------------------------------------------------------+
|                              DATA PLANE (Rust / Tokio)                                  |
|                                                                                         |
|  [Auth & Key Check] ---> [Rate Limiter] ---> [Semantic Cache] ---> [DLP & PII Masker]   |
|          |                     |                      |                     |           |
|          v                     v                      v                     v           |
|  [Redis Token Bucket]   [Redis Leaky]           [Qdrant / Milvus]     [Rust Regex/NER]  |
|                                                                                         |
|  [Intelligent Router] ---> [Circuit Breaker / Retries] ---> [Upstream Provider Pool]    |
|                                                              (OpenAI, Anthropic, Bedrock)|
+-----------------------------------------------------------------------------------------+
                                          | (Async Kafka Event Stream)
                                          v
+-----------------------------------------------------------------------------------------+
|                               CONTROL & TELEMETRY PLANE                                 |
|                                                                                         |
|  [Kafka Event Bus] ---> [ClickHouse: Metrics/FinOps] ---> [S3 WORM: Audit Trail Logs]    |
|                                                                                         |
|  [Management Dashboard (React)] <---> [Control API (NestJS)] <---> [PostgreSQL Config]   |
+-----------------------------------------------------------------------------------------+
```

## 2. Component Specifications & Interfaces

### 2.1 Proxy Pipeline Stages
1. **`AuthN/AuthZ Interceptor`**: Validates incoming Bearer token against Redis cache ($O(1)$ lookup, $\le 0.8\text{ms}$). Resolves tenant organization ID, allowed models, and budget limits.
2. **`Dual Leaky Bucket Rate Limiter`**: Evaluates `RPM` and `TPM` against Redis sliding windows.
3. **`Semantic Cache Filter`**: Generates embedding using local ONNX model (all-MiniLM-L6-v2 in $\le 3\text{ms}$) or Fastembed, querying Qdrant/Redis. Short-circuits upstream calls on cache hits.
4. **`Streaming DLP Processor`**: Pre-scans prompts using optimized multi-pattern Aho-Corasick automaton for keywords, credit cards, SSNs, and private keys.
5. **`Dynamic Upstream Dispatcher`**: Translates unified request format into provider-specific schemas (e.g. converting OpenAI tool format to Anthropic tool schema if routing between providers).
6. **`Streaming SSE Demuxer`**: Intercepts SSE chunks, counts tokens in real time, monitors for disallowed toxic phrases, and buffers tool calls.
7. **`Async Telemetry Emitter`**: Pushes structured audit envelope to Kafka/Redpanda topic `ai.gateway.traces`.

## 3. Data Schema & Contracts

### 3.1 Unified Chat Completion Endpoint (`POST /v1/chat/completions`)
```json
{
  "model": "auto-smart",
  "messages": [
    { "role": "system", "content": "You are a financial advisor." },
    { "role": "user", "content": "Analyze customer transaction 1234-5678-9012-3456" }
  ],
  "temperature": 0.2,
  "stream": true,
  "metadata": {
    "organization_id": "org_enterprise_99",
    "cost_center": "wealth_management_eu",
    "trace_id": "trc_849204928402"
  }
}
```

### 3.2 Kafka Audit Log Envelope (`ai.gateway.traces`)
```json
{
  "trace_id": "trc_849204928402",
  "timestamp": "2026-09-14T12:00:00.123Z",
  "organization_id": "org_enterprise_99",
  "api_key_prefix": "msp_gw_a9f1",
  "client_ip_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "model_requested": "auto-smart",
  "model_routed": "claude-3-5-sonnet-20241022",
  "provider": "anthropic",
  "prompt_token_count": 420,
  "completion_token_count": 890,
  "total_cost_usd": 0.01461,
  "ttft_ms": 284,
  "total_duration_ms": 1420,
  "cache_hit": false,
  "dlp_actions_taken": [
    { "type": "CREDIT_CARD", "redacted_count": 1, "action": "TOKENIZED_SURROGATE" }
  ],
  "policy_decision": "ALLOWED",
  "worm_storage_uri": "s3://enterprise-ai-worm-logs/2026/09/14/trc_849204928402.json.gz"
}
```

## 4. MCP Agentic Sandbox Implementation
- When an autonomous agent attempts a tool execution (`tools/call`), the gateway intercepts the JSON payload.
- Rules are evaluated against an Open Policy Agent (OPA) / Rego policy engine:
  ```rego
  package ai.gateway.mcp

  default allow = false

  allow {
      input.tool_name == "sql_query"
      input.arguments.readonly == true
      not contains(lower(input.arguments.query), "drop")
      not contains(lower(input.arguments.query), "delete")
  }
  ```
- If `allow` is false and rule severity is `ESCALATE`, the request enters the HITL approval state machine with WebSocket notifications.

## 5. Security & Deployment Architecture
- **Multi-Region Deployment**: Active-active Kubernetes pods deployed across AWS (us-east-1, eu-central-1) and Azure (East US, West Europe) behind Cloudflare Magic Transit.
- **Mutual TLS (mTLS)**: Enforced between proxy data planes and internal enterprise services.
- **Hardware Security Modules (HSM)**: Upstream API keys are never stored in plaintext on disk; keys are dynamically injected from HashiCorp Vault or AWS Secrets Manager into memory with periodic rotation.
