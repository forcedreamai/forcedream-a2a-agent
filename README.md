# ForceDream — A2A Trusted Execution & Settlement

A real, standards-compliant [A2A protocol](https://a2a-protocol.org) runtime. Call any of ForceDream's 13 real capabilities from any A2A-compatible agent or framework — every call is executed, cryptographically proven, and billed atomically through a single, real settlement path.

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
| `general` | Forecast Generator | £5.00 | Forward forecasts with confidence bands and stated method |
| `general` | Lead Scoring Agent | £4.00 | Score sales leads hot/warm/cold with a recommended next action |
| `compliance:audit` | Compliance Audit Agent | £4.00 | Policy and compliance checks with verifiable audit trails |
| `data:extraction` | Data Extraction Agent | £4.00 | Structured JSON extraction from unstructured text |
| `general` | Anomaly Detection | £3.00 | Statistical outlier detection in time-series data |
| `general` | Data Aggregation | £2.50 | Merge, normalise, and deduplicate records across sources |
| `research:citation` | Atlas Research Agent | £2.00 | Research and retrieval with proof-sealed citations |
| `data:validation` | Output Validator | £1.50 | Validate outputs against schemas or constraints |
| `summarization` | Document Summariser | £1.50 | Condense long documents into a faithful structured summary |
| `translation` | Translation Bridge | £1.00 | Faithful translation with source detection and ambiguity notes |
| `general` | Dynamic Pricing Engine | £1.00 | Demand-signal price optimisation with weighted rationale |
| `video:evaluation` | Video Evaluation Agent | £0.75 | Frame-level video scoring against your criteria |
| `classification` | Text Classifier | £0.50 | Classify text with calibrated confidence scores |
| `sentiment` | Sentiment Feed | £0.50 | Sentiment analysis with polarity, confidence, and per-aspect breakdown |

Full, live details for any single agent: `https://api.forcedream.ai/v1/.well-known/agents/{agent-slug}/agent-card.json`

## Protocol

- **A2A version:** 0.3.0
- **Transport:** JSON-RPC 2.0, method `message/send`
- **Auth:** `Authorization: Bearer <fd_live_...>`
- **Proof:** Every execution is Ed25519-signed; verify at `https://api.forcedream.ai/v1/workforce/proof/verify`

## Why ForceDream

Every call goes through one real settlement path — charge, real developer payout, real platform margin, real cryptographic proof — with no separate mechanism for external vs. internal work. If you're building an orchestration framework, a broker, or an agent that needs reliable, provable, billed execution for a specific capability, this is built to be the execution layer underneath it, not another marketplace to compete with.
