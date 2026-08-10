# Library Docs

Project-specific usage patterns for every third-party library and tool in this project. This file only covers how we use each one in InferGate specifically — rules, patterns, and constraints. It does not replace official docs.

Read the relevant section before implementing any feature that touches these tools.

---

## Before Using Any Library or Tool

1. **Check AGENTS.md** at the project root for an installed skill covering that tool.
2. **Check if an MCP server is configured** for that tool (e.g. a Kubernetes or GitHub MCP) — use it before falling back to general knowledge.
3. **Read this file** for InferGate-specific patterns that override general knowledge.

Order of authority:

```
MCP server (real-time docs/cluster state) → Skills via AGENTS.md → This file (project rules) → General training knowledge
```

Kubernetes, Helm, vLLM, and KEDA CRD schemas change frequently between versions — never rely on general training knowledge alone for exact flag names or CRD fields. Confirm the version pinned in `charts/infergate-model/Chart.yaml` before assuming an API shape.

---

## vLLM

**Check first:** confirm the pinned vLLM image tag in `charts/infergate-model/values.yaml` before assuming which flags/features are available — quantization support and other features are version-gated.

### Serving a model

```yaml
# charts/infergate-model/values.yaml (relevant excerpt)
image: vllm/vllm-openai:v0.x.x   # always pin, never `latest`
args:
  - "--model"
  - "{{ .Values.model }}"
  - "--gpu-memory-utilization"
  - "0.9"
```

### API shape (what the gateway proxies to)

vLLM exposes an OpenAI-compatible endpoint:

```
POST /v1/chat/completions
{
  "model": "mistral-7b",
  "messages": [{"role": "user", "content": "..."}]
}
```

Response includes a `usage` object — always read tokens from this, never estimate them client-side:

```json
{
  "choices": [...],
  "usage": {
    "prompt_tokens": 42,
    "completion_tokens": 128,
    "total_tokens": 170
  }
}
```

**Rules:**

- Always read `usage.prompt_tokens` / `usage.completion_tokens` from the vLLM response for cost calculation — never estimate with a local tokenizer unless vLLM is unreachable and you're logging a failed request.
- The mock model server (`mock-model-server/main.py`) must return this exact response shape, including a populated `usage` object with realistic-looking numbers, so gateway code never branches on which backend it's talking to.
- Quantization choices (where supported by the pinned version and GPU) are a deployment-time flag, not gateway logic — document any quantization choice in an ADR, never change it silently.

---

## FastAPI

**Check first:** check AGENTS.md for an installed FastAPI/Python skill.

### Health check pattern

```python
# gateway/routers/health.py
@router.get("/healthz")
async def healthz() -> dict[str, str]:
    return {"status": "ok"}
```

### Dependency injection for auth

```python
# gateway/core/auth.py
async def get_current_team(
    x_api_key: str = Header(...),
    db: AsyncSession = Depends(get_db),
) -> str:
    key_record = await lookup_api_key(db, x_api_key)
    if key_record is None or key_record.revoked_at is not None:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return key_record.team
```

**Rules:**

- `/healthz` never requires auth — Kubernetes liveness/readiness probes hit it directly.
- Every other route requires a valid API key via the `get_current_team` dependency.
- API keys are looked up by hash, never by plaintext comparison.

---

## KEDA

**Check first:** confirm the installed KEDA version supports the scaler type used (`prometheus` scaler for queue depth) — check `charts/infergate-model/templates/scaledobject.yaml` for the pinned `apiVersion`.

### ScaledObject — queue-depth trigger

```yaml
# charts/infergate-model/templates/scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {{ .Release.Name }}-so
spec:
  scaleTargetRef:
    name: {{ .Release.Name }}
  minReplicaCount: 0
  maxReplicaCount: {{ .Values.maxReplicas }}
  cooldownPeriod: 300
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.observability.svc:80
        metricName: infergate_queue_depth
        threshold: "50"
        query: infergate_queue_depth{model="{{ .Values.model }}"}
```

**Rules:**

- Trigger is always `infergate_queue_depth`, never CPU or memory — queue depth is the correct signal for inference workloads.
- `minReplicaCount: 0` is intentional for scale-to-zero on idle models — document any model where this is overridden (e.g. a latency-critical model that should never cold-start) in an ADR.
- `cooldownPeriod` is set high enough (300s) to avoid flapping on bursty traffic — do not lower it without a load-test result showing it's safe.

---

## Helm

**Check first:** run `helm template` locally before every `helm upgrade --install` to confirm rendered output — never assume a values change renders as expected.

### Chart structure

```
charts/infergate-model/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── scaledobject.yaml
    └── configmap.yaml
```

### Values pattern

```yaml
# values.yaml
model: ""              # required, no default — CLI must always pass this
gpuTier: "small"
team: ""                # required, no default — cost attribution depends on it
replicaCount: 1
maxReplicas: 5
resources:
  requests: {cpu: "2", memory: "8Gi"}
  limits: {cpu: "4", memory: "16Gi", "nvidia.com/gpu": "1"}
```

**Rules:**

- `model` and `team` have no default — the chart should fail to render meaningfully if the CLI doesn't pass them, rather than silently deploying an unlabeled/unattributed model.
- `helm upgrade --install` is always used from the CLI — never bare `helm install`, so re-running `infergate deploy` on an existing model is idempotent.
- Chart version bumps whenever `templates/` changes, per `code-standards.md`.

---

## Prometheus (Python client)

**Check first:** check AGENTS.md for an installed Prometheus/observability skill.

```python
# gateway/metrics/prometheus.py
from prometheus_client import Counter, Histogram, Gauge

REQUESTS_TOTAL = Counter(
    "infergate_requests_total", "Total gateway requests", ["model", "team", "status"]
)
REQUEST_LATENCY = Histogram(
    "infergate_request_latency_seconds", "Request latency", ["model", "team"]
)
COST_TOTAL = Counter(
    "infergate_cost_usd_total", "Cumulative cost in USD", ["model", "team"]
)
```

**Rules:**

- Every metric name is prefixed `infergate_`, per `code-standards.md`.
- Every metric that can be attributed to a model or team carries those as labels — the Grafana dashboards join on these labels.
- Metrics are incremented inside `gateway/core/`, never inside route handlers directly, so unit tests can assert on them without spinning up FastAPI.

---

## Redis

**Check first:** check AGENTS.md for an installed Redis skill.

### Rate limiting (sliding window, per team)

```python
# gateway/core/rate_limit.py
async def check_rate_limit(redis: Redis, team: str, limit: int, window_seconds: int) -> bool:
    key = f"ratelimit:{team}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window_seconds)
    return count <= limit
```

### Queue depth signal (for KEDA)

```python
# incremented when a request is accepted, decremented when it completes
await redis.incr(f"queue_depth:{model}")
...
await redis.decr(f"queue_depth:{model}")
```

**Rules:**

- Rate limit keys are always scoped by `team`, never globally — one noisy team must not throttle another.
- Queue depth increments/decrements are always paired in a try/finally — a request that errors must still decrement, or the queue depth gauge drifts and autoscaling misbehaves.

---

## PostgreSQL (SQLAlchemy async + asyncpg)

**Check first:** check AGENTS.md for an installed Postgres/SQLAlchemy skill.

```python
# gateway/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine(DATABASE_URL, pool_size=10)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSession(engine) as session:
        yield session
```

```python
# Insert a request log row
async def log_request(db: AsyncSession, record: RequestLog) -> None:
    db.add(record)
    await db.commit()
```

**Rules:**

- Always use the async engine/session — the gateway is async end to end, a sync call blocks the event loop.
- Every write to `requests` happens exactly once per gateway request, in a `finally` block, so failed requests are logged too.
- Never query `requests` without a `team` or `model_deployment_id` filter when computing per-team/per-model cost — full table scans defeat the purpose of the dashboard.

---

## Typer (CLI)

**Check first:** check AGENTS.md for an installed Typer skill.

```python
# cli/infergate/main.py
import typer
from infergate.commands import deploy, list_models, status, delete

app = typer.Typer(help="InferGate — self-service LLM inference on Kubernetes")
app.add_typer(deploy.app, name="deploy")
app.add_typer(list_models.app, name="list")
app.add_typer(status.app, name="status")
app.add_typer(delete.app, name="delete")

if __name__ == "__main__":
    app()
```

**Rules:**

- Every command module exposes its own `typer.Typer()` instance and is registered in `main.py` — never define commands loose in `main.py` directly.
- Use `rich.table.Table` for any tabular output (`infergate list`) — never manually pad strings.

---

## kind (local Kubernetes)

**Check first:** check AGENTS.md for an installed Kubernetes/kind skill.

```bash
kind create cluster --name infergate --config kind-config.yaml
kind load docker-image infergate-gateway:dev --name infergate
kind load docker-image mock-model-server:dev --name infergate
```

**Rules:**

- All local images are loaded with `kind load docker-image` — never pushed to a public registry during local dev.
- `kind-config.yaml` is committed to the repo so cluster config (node count, port mappings) is reproducible.

---

## k6 (load testing)

**Check first:** check AGENTS.md for an installed k6 skill.

```javascript
// tests/load/k6-script.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 200 },
    { duration: '30s', target: 0 },
  ],
};

export default function () {
  const res = http.post(
    'http://localhost:8000/v1/chat/completions',
    JSON.stringify({ model: 'mistral-7b', messages: [{ role: 'user', content: 'hi' }] }),
    { headers: { 'Content-Type': 'application/json', 'x-api-key': __ENV.API_KEY } },
  );
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

**Rules:**

- Ramp stages always include a ramp-down (`target: 0`) — the point of the test is proving scale-down works, not just scale-up.
- Run against the mock model server for repeatable CI runs; run once against real vLLM for the demo recording.

---

## Kubernetes Python Client

**Check first:** check AGENTS.md for an installed Kubernetes client skill. Prefer shelling out to `kubectl`/`helm` for anything a human would normally type — reserve the Python client for programmatic reads the CLI needs.

```python
# cli/infergate/commands/status.py
from kubernetes import client, config

config.load_kube_config()
apps_v1 = client.AppsV1Api()
deployment = apps_v1.read_namespaced_deployment(name=deployment_name, namespace="infergate")
```

**Rules:**

- `infergate deploy`/`delete` shell out to `helm` — never construct raw Kubernetes objects via the Python client for these.
- `infergate status`/`list` may use the Python client for read-only queries, since it's faster than parsing `kubectl` text output.
