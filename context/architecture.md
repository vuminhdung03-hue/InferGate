# Architecture

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| Serving | vLLM (OpenAI-compatible API) | Model inference engine |
| Orchestration | Kubernetes (`kind` locally, cloud GPU node for one real demo) | Container orchestration |
| Packaging | Helm | Templated, parameterized deployment |
| Autoscaling | KEDA (queue-depth trigger) | Scale-to-zero and scale-out on demand |
| Gateway | FastAPI (async) | Auth, retry, rate limit, request logging |
| Rate limiting / queue signal | Redis | Sliding-window rate limits, queue depth for KEDA |
| Persistent store | PostgreSQL (SQLAlchemy async + asyncpg) | Model registry, request/cost logs |
| Metrics | Prometheus (`prometheus-client`) | Requests, latency, tokens, cost |
| Dashboards | Grafana | Cost per model/team, GPU utilization |
| CLI | Python + Typer + Rich | Self-service deploy/list/status/delete |
| Load testing | k6 | Autoscaling validation |
| CI | GitHub Actions | Lint, test, `helm template`, build image on push |
| Language | Python 3.11+, strict type hints | Gateway, CLI, tests |

---

## Folder Structure

```
/
├── AGENTS.md
├── context/
│   ├── project-overview.md
│   ├── architecture.md
│   ├── code-standards.md
│   ├── library-docs.md
│   ├── build-plan.md
│   └── progress-tracker.md
├── gateway/
│   ├── main.py                              → FastAPI app entrypoint
│   ├── routers/
│   │   ├── completions.py                   → POST /v1/chat/completions
│   │   └── health.py                        → GET /healthz, GET /metrics
│   ├── core/
│   │   ├── auth.py                          → API key lookup + team resolution
│   │   ├── retry.py                         → Backend retry logic
│   │   ├── rate_limit.py                    → Redis sliding-window rate limiting
│   │   └── cost.py                          → Cost + CPM calculation
│   ├── db/
│   │   ├── models.py                        → SQLAlchemy models
│   │   └── session.py                       → Async engine/session
│   ├── metrics/
│   │   └── prometheus.py                    → Counters, histograms, gauges
│   └── tests/
├── charts/
│   └── infergate-model/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── scaledobject.yaml
│           └── configmap.yaml
├── cli/
│   ├── infergate/
│   │   ├── __init__.py
│   │   ├── main.py                          → Typer app entrypoint
│   │   ├── config.py                        → Constants (thresholds, pricing path)
│   │   ├── commands/
│   │   │   ├── deploy.py
│   │   │   ├── list_models.py
│   │   │   ├── status.py
│   │   │   └── delete.py
│   │   ├── cost_estimator.py
│   │   └── helm.py                          → helm shellout wrapper
│   └── tests/
├── mock-model-server/
│   └── main.py                              → vLLM-shaped mock, GPU-free
├── observability/
│   ├── grafana/
│   │   └── dashboards/
│   │       └── infergate-cost-dashboard.json
│   └── prometheus/
│       └── prometheus-values.yaml
├── config/
│   └── pricing.yaml                         → GPU $/hour reference table
├── docs/
│   ├── adrs/
│   │   ├── 001-why-not-kserve-llmd.md
│   │   ├── 002-mock-vs-real-gpu.md
│   │   ├── 003-gateway-design.md
│   │   ├── 004-autoscaling-trigger.md
│   │   └── 005-cost-methodology.md
│   └── finops-playbook.md
├── tests/
│   ├── integration/                         → run against a real kind cluster in CI
│   └── load/
│       └── k6-script.js
└── .github/
    └── workflows/
        └── ci.yaml
```

---

## System Boundaries

| Folder | Owns |
|---|---|
| `gateway/` | All request-serving logic: auth, retry, rate limit, cost logging, metrics. No CLI or Helm logic. |
| `charts/` | All Kubernetes resource definitions. No application logic — values only. |
| `cli/` | Self-service developer commands only. Calls Helm under the hood; never talks to the gateway directly. |
| `mock-model-server/` | GPU-free stand-in for vLLM, matching its API shape exactly. Never used in the "real" demo. |
| `observability/` | Prometheus scrape config and Grafana dashboard JSON only. No application code. |
| `config/` | Reference data (GPU pricing) read by both `gateway/` and `cli/` — neither hardcodes prices. |
| `docs/` | ADRs and the FinOps playbook. Prose only, no code. |
| `tests/` | Integration and load tests. Never imported by application code. |

---

## Data Flow

### CLI Deploy Flow

```
Developer runs `infergate deploy`
        ↓
CLI estimates cost from GPU tier × expected volume (config/pricing.yaml)
        ↓
Threshold exceeded? → require --confirm
        ↓
CLI renders Helm values from template
        ↓
`helm upgrade --install` applied to cluster
        ↓
Deployment written to model_deployments table
```

### Gateway Request Flow

```
Client sends request to gateway
        ↓
Gateway validates API key + team rate limit (Redis)
        ↓
Gateway forwards to a healthy model replica
        ↓
   Replica unreachable? → retry against another replica
        ↓
Response returned to client
        ↓
Request logged to `requests` table: tokens, latency, cost
        ↓
Prometheus metrics updated
```

### Cost Calculation Flow

```
Request completes → tokens in/out read from vLLM's `usage` field
        ↓
Look up GPU instance price from config/pricing.yaml
        ↓
cost = (GPU $/hour ÷ 3600) × wall-clock seconds attributed to this request
        ↓
Cost-per-million-tokens (CPM) computed per model for dashboard aggregation
        ↓
Both written to `requests` table
```

### Autoscaling Flow

```
KEDA polls the Prometheus `infergate_queue_depth` metric
        ↓
Queue depth > threshold → replicas scale up
        ↓
Queue depth low for cooldown period → replicas scale down (to zero if idle)
```

### Chaos Recovery Flow

```
Pod killed mid-request (kubectl delete pod)
        ↓
Gateway request to that pod fails
        ↓
Gateway retries against a healthy replica — client never sees the failure
        ↓
Kubernetes control plane reschedules a replacement pod
```

---

## Database Schema (PostgreSQL)

### `model_deployments`

| Column | Type | Notes |
|---|---|---|
| id | uuid | |
| model_name | text | e.g. mistral-7b |
| gpu_tier | text | small / medium / large |
| team | text | owning team label — required, no default |
| replica_count | integer | current desired replicas |
| status | text | deploying / running / failed / deleted |
| estimated_monthly_cost | numeric | computed at deploy time |
| deployed_at | timestamptz | |
| deleted_at | timestamptz | null until deleted |

### `requests`

| Column | Type | Notes |
|---|---|---|
| id | uuid | |
| model_deployment_id | uuid | references model_deployments |
| team | text | denormalized for fast cost queries |
| tokens_in | integer | from vLLM's `usage.prompt_tokens` |
| tokens_out | integer | from vLLM's `usage.completion_tokens` |
| latency_ms | integer | |
| cost_usd | numeric | computed cost for this request |
| status | text | success / retried / failed |
| created_at | timestamptz | |

### `api_keys`

| Column | Type | Notes |
|---|---|---|
| id | uuid | |
| key_hash | text | hashed, never plaintext |
| team | text | |
| created_at | timestamptz | |
| revoked_at | timestamptz | null while active |

### `gpu_pricing` (reference config, not request-driven — may live as a table or as `config/pricing.yaml`)

| Column | Type | Notes |
|---|---|---|
| gpu_type | text | H100 / A100 / L40S etc. |
| price_per_hour_usd | numeric | published on-demand price |
| updated_at | timestamptz | |

---

## Prometheus Metrics

| Metric | Type | Labels |
|---|---|---|
| `infergate_requests_total` | counter | model, team, status |
| `infergate_request_latency_seconds` | histogram | model, team |
| `infergate_tokens_total` | counter | model, team, direction (in/out) |
| `infergate_cost_usd_total` | counter | model, team |
| `infergate_gpu_utilization_ratio` | gauge | node, gpu_id |
| `infergate_queue_depth` | gauge | model |

---

## Kubernetes Resource Naming Convention

```
Deployment:    infergate-model-{model_name}-{team}
Service:       infergate-model-{model_name}-{team}-svc
ScaledObject:  infergate-model-{model_name}-{team}-so
Namespace:     infergate (all InferGate-managed resources live in one namespace,
               isolated via labels rather than separate namespaces per team,
               to keep the demo scope manageable)
```

---

## Authentication

- **Gateway:** static API keys stored in Postgres (`api_keys` table). Looked up by hash — never stored or compared as plaintext.
- **CLI:** uses the developer's local kubeconfig context — no separate auth layer, since it assumes the developer already has cluster access.

---

## Invariants

Rules the AI agent must never violate:

- Nothing deploys to the cluster except through the Helm chart in `charts/` — no ad hoc `kubectl apply` in application or CLI code (Phase 1's one-time hand-written YAML is the sole documented exception, and it's never applied again after the chart exists).
- Every gateway request is logged to `requests`, success or failure — no silent/unlogged calls.
- Every retry attempt is logged, even on eventual success — never retry silently.
- Cost is always computed in cost-per-million-tokens (CPM) for dashboard aggregation, never just raw dollars, to stay comparable across models.
- GPU prices always come from `config/pricing.yaml` / `gpu_pricing` — never hardcoded inline in cost logic.
- API keys are never stored or logged in plaintext — only hashed.
- The mock model server's API shape must always match vLLM's OpenAI-compatible response exactly, including a populated `usage` object — swapping backends must be a one-line config change, never a code change.
- Autoscaling always triggers on queue depth — never on CPU/memory alone.
- Every Kubernetes resource created by InferGate carries a `team` label — cost attribution depends on it.
- `helm upgrade --install` is always used (never bare `helm install`) so `infergate deploy` is idempotent.
