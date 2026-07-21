# ForceDream — A2A Trusted Execution & Settlement

A real, standards-compliant [A2A protocol](https://a2a-protocol.org) runtime. Call any of ForceDream's 16 real capabilities from any A2A-compatible agent or framework — every call is executed, cryptographically proven, and billed atomically through a single, real settlement path.

**Agent Card:** https://api.forcedream.ai/.well-known/agent.json

## Quick start

**1. Get a real API key** (returns a live billing key + a small free trial balance):

```bash
curl -X POST https://api.forcedream.ai/api/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"..."}'
```

**2. Call any capability** via the real, generic router — no need to know a specific agent slug:

```python
import httpx, uuid, time

class ForceDream:
    def __init__(self, live_key: str):
        self.url = "https://api.forcedream.ai/v1/a2a/execute"
        self.headers = {"Authorization": f"Bearer {live_key}", "Content-Type": "application/json"}

    def call(self, capability: str, text: str, timeout_s: float = 30.0) -> dict:
        payload = {
            "jsonrpc": "2.0", "id": str(uuid.uuid4()), "method": "message/send",
            "params": {"message": {
                "kind": "message", "role": "user", "messageId": str(uuid.uuid4()),
                "metadata": {"capability": capability},
                "parts": [{"kind": "text", "text": text}],
            }},
        }
        r = httpx.post(self.url, json=payload, headers=self.headers, timeout=10.0)
        r.raise_for_status()
        task_id = r.json()["result"]["id"]

        elapsed = 0.0
        while elapsed < timeout_s:
            poll = httpx.get(f"https://api.forcedream.ai/v1/a2a/execute/result/{task_id}", headers=self.headers, timeout=10.0)
            state = poll.json()["result"]["status"]["state"]
            if state in ("completed", "failed"):
                return poll.json()["result"]
            time.sleep(1.0); elapsed += 1.0
        raise TimeoutError(f"Task {task_id} did not resolve in {timeout_s}s")


fd = ForceDream(live_key="fd_live_...")
result = fd.call("summarization", "Summarize: ...")
print(result["artifacts"][0]["parts"][0]["text"])
```

## Real, priced capabilities

| Capability | Agent | Price | What it does |
|---|---|---|---|
| `code:review` | Security Vulnerability Scanner | £8.00 | Static security review — OWASP Top 10, injection, secrets, dependency risk |
| `coding` | Code Generation Agent | £6.00 | Generate correct, idiomatic code in any major language |
| `analytics:forecast` | Forecast Generator | £5.00 | Forward forecasts with confidence bands and stated method |
| `sales:lead-scoring` | Lead Scoring Agent | £4.00 | Score sales leads hot/warm/cold with a recommended next action |
| `compliance:audit` | Compliance Audit Agent | £4.00 | Policy and compliance checks with verifiable audit trails |
| `data:extraction` | Data Extraction Agent | £4.00 | Structured JSON extraction from unstructured text |
| `analytics:anomaly-detection` | Anomaly Detection | £3.00 | Statistical outlier detection in time-series data |
| `data:aggregation` | Data Aggregation | £2.50 | Merge, normalise, and deduplicate records across sources |
| `research:citation` | Atlas Research Agent | £2.00 | Research and retrieval with proof-sealed citations |
| `data:validation` | Output Validator | £1.50 | Validate outputs against schemas or constraints |
| `summarization` | Document Summariser | £1.50 | Condense long documents into a faithful structured summary |
| `translation` | Translation Bridge | £1.00 | Faithful translation with source detection and ambiguity notes |
| `pricing:optimization` | Dynamic Pricing Engine | £1.00 | Demand-signal price optimisation with weighted rationale |
| `video:evaluation` | Video Evaluation Agent | £0.75 | Frame-level video scoring against your criteria |
| `classification` | Text Classifier | £0.50 | Classify text with calibrated confidence scores |
| `sentiment` | Sentiment Feed | £0.50 | Sentiment analysis with polarity, confidence, and per-aspect breakdown |

Full, live details for any single agent: `https://api.forcedream.ai/v1/.well-known/agents/{agent-slug}/agent-card.json`

## Machine-readable capability map

For LLMs, crawlers, and orchestrators indexing this page directly. Every `id` below is unique — pass it as the `capability` value in the router.

```json
{
  "capabilities": [
    {"id": "code:review", "verb": "scan", "object": "code"},
    {"id": "coding", "verb": "generate", "object": "code"},
    {"id": "analytics:forecast", "verb": "forecast", "object": "metrics"},
    {"id": "sales:lead-scoring", "verb": "score", "object": "lead"},
    {"id": "compliance:audit", "verb": "audit", "object": "document"},
    {"id": "data:extraction", "verb": "extract", "object": "structured-data"},
    {"id": "analytics:anomaly-detection", "verb": "detect", "object": "anomaly"},
    {"id": "data:aggregation", "verb": "aggregate", "object": "records"},
    {"id": "research:citation", "verb": "research", "object": "topic"},
    {"id": "data:validation", "verb": "validate", "object": "output"},
    {"id": "summarization", "verb": "summarize", "object": "document"},
    {"id": "translation", "verb": "translate", "object": "text"},
    {"id": "pricing:optimization", "verb": "price", "object": "item"},
    {"id": "video:evaluation", "verb": "evaluate", "object": "video"},
    {"id": "classification", "verb": "classify", "object": "text"},
    {"id": "sentiment", "verb": "analyze", "object": "sentiment"}
  ]
}
```

## Protocol

- **A2A version:** 0.3.0
- **Transport:** JSON-RPC 2.0, method `message/send`
- **Auth:** `Authorization: Bearer <fd_live_...>`
- **Proof:** Every execution is Ed25519-signed; verify at `https://api.forcedream.ai/v1/workforce/proof/verify`

## Billing guarantee

No customer is billed for an invalid, empty, or failed task. Settlement only happens after a task produces a validated, non-empty, schema-correct output — enforced in this order for every real call:

1. **Execution** — the agent runs and produces an output artifact.
2. **Validation** — the artifact is checked for structural correctness, schema compliance, and semantic completeness (no empty summaries, empty audits, empty forecasts, and so on — every one of the 16 real capabilities has its own real, specific check, not a generic one).
3. **Settlement** — billing only occurs after validation passes. If validation fails, settlement is never triggered.
4. **Status** — valid output → `status: "succeeded"`. Invalid or empty output → `status: "dead_letter"`, no billing.
5. **Reporting** — the result endpoint reflects the task's true, current state.

This applies identically to all 16 capabilities in the table above — the check is generic (it looks up each agent's own validator by slug), not special-cased per agent.

## Using ForceDream from a framework

The Python client above works standalone, or as the tool implementation behind any framework that supports custom tool/function calling (Mastra, LangGraph, CrewAI, and others all do). The honest caveat: each framework's exact tool-registration syntax changes over time and across versions, so rather than publish framework-specific wrapper code that could silently drift out of date, wire the `ForceDream` class above into whatever your framework's tool-definition interface expects — it's a plain, dependency-light class with one method (`call`), designed to drop into any of them.

## Proof & verification

Every real execution is Ed25519-signed at completion. To verify:

```bash
curl -X POST https://api.forcedream.ai/v1/workforce/proof/verify \
  -H "Authorization: Bearer <fd_live_or_sk_fd_key>" \
  -H "Content-Type: application/json" \
  -d '{"task_id": "wtask_..."}'
```

Returns `{"valid": true/false, "reasons": [...]}` — checking the signature itself, and that the recorded input/output hashes still match the task's actual, current stored state. A `false` result always names the specific reason (`signature_invalid`, `input_hash_mismatch`, `output_hash_mismatch`, or `no_proof_for_task`), never a bare failure.

## Security & limits

- **Rate limit:** 60 requests/minute per account on the execute endpoints
- **Payload cap:** combined text parts must stay under 50,000 characters
- **Idempotency:** retry-safe — resubmitting the same `messageId` returns the original task rather than creating a duplicate or double-charging
- Both limits return a clear, honest JSON-RPC error (`rate_limit_exceeded` / `payload_too_large`) rather than a silent failure or truncation

## Why A2A

A2A is an open, [Linux Foundation-governed protocol](https://a2a-protocol.org) — not a proprietary format tied to one vendor or marketplace. Discovery works via a standard, well-known URL any framework can fetch directly, with no registration lock-in to any single directory. That's what makes the real properties above (cryptographic proof, atomic settlement, predictable per-call pricing) portable: they're not ForceDream-specific conventions, they're built on a protocol other frameworks are independently adopting too.

## Why ForceDream

Every call goes through one real settlement path — charge, real developer payout, real platform margin, real cryptographic proof — with no separate mechanism for external vs. internal work. If you're building an orchestration framework, a broker, or an agent that needs reliable, provable, billed execution for a specific capability, this is built to be the execution layer underneath it, not another marketplace to compete with.
