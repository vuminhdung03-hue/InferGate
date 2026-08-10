# Build Plan

## Core Principle

Every feature must produce something runnable and verifiable — a curl response, a `kubectl get` result, a rendered Helm template, a dashboard panel — before moving to the next. No invisible phases where multiple features get built before anything is checked.

---

## Phase 1 — Foundation

### 01 Repo & Differentiation

**Setup:**

- Public GitHub repo, MIT license
- Folder skeleton per `architecture.md`
- ADR-001: why a scoped single-runtime vLLM + FastAPI gateway instead of KServe+llm-d or LLMKube
- README with pitch, architecture diagram, qualifications-to-features mapping table

---

### 02 Local Environment

**Setup:**

- Docker, `kind`, `kubectl`, `helm`, Python 3.11+ installed
- `kind create cluster --name infergate`
- ADR-002: mock model server vs. real GPU strategy for local dev

**Verify:** `kubectl get nodes` shows a healthy node.

---

### 03 Mock Model Server

**Logic:**

- FastAPI app in `mock-model-server/main.py` returning vLLM-shaped responses (including a populated `usage` object) with a randomized/artificial delay
- Dockerized, loaded into `kind` via `kind load docker-image`

**Verify:** `kubectl port-forward` to the pod, curl `/v1/chat/completions`, get a valid mock completion back.

---

### 04 Raw Kubernetes YAML (by hand, once)

**Logic:**

- Hand-write Deployment + Service YAML for the mock model server
- Apply directly with `kubectl apply -f`
- Deliberately NOT abstracted yet — understanding what the Helm chart needs to generate later depends on having done this manually once

**Verify:** pod running, service reachable via `kubectl port-forward` + curl.

---

## Phase 2 — Real Serving Path

### 05 Real vLLM Deployment

**Logic:**

- Pull `vllm/vllm-openai`, deploy a small model (1B–3B params) against a rented GPU node (RunPod/Lambda, one session)
- Confirm the real response shape matches the mock server's shape exactly

**Verify:** real completion returned via curl; screen recording saved as the "proof of life" artifact.

---

## Phase 3 — Gateway Layer

### 06 Gateway Skeleton + Health Check

**Logic:**

- FastAPI app in `gateway/main.py`, `/healthz` route
- Deployed into `kind` as its own Deployment + Service

**Verify:** curl `/healthz` returns `{"status": "ok"}` from inside the cluster.

---

### 07 Auth (API Keys)

**Logic:**

- `api_keys` table in Postgres (key_hash, team, created_at, revoked_at)
- `get_current_team` FastAPI dependency, per `library-docs.md`
- Missing/invalid key → 401

**Verify:** request without a key is rejected; request with a valid key succeeds.

---

### 08 Proxy + Retry Logic

**Logic:**

- Gateway forwards `/v1/chat/completions` to the model service
- On backend failure, retries against another replica before returning an error
- Unit tests mock a dead backend and assert retry behavior

**Verify:** pytest passes; manually kill a mock-server replica and confirm the gateway still returns a success.

---

### 09 Rate Limiting

**Logic:**

- Redis-backed sliding window, scoped per team, per `library-docs.md`
- Exceeding the limit returns 429

**Verify:** load a team's key past its limit, confirm 429s start appearing.

---

### 10 Request Logging

**Logic:**

- `requests` table in Postgres — every request logged with tokens in/out, latency, status, cost (real cost calculation lands in Phase 5; log a placeholder/zero cost for now)
- Logged in a `finally` block so failed requests are captured too

**Verify:** query the `requests` table after a burst of curl calls, confirm rows exist for both successes and induced failures.

---

## Phase 4 — Helm Chart + CLI

### 11 Helm Chart

**Setup:**

- Convert Phase 1's hand-written YAML into `charts/infergate-model/`, parameterized per `library-docs.md`
- `helm template` output diffed against the original hand-written YAML to confirm equivalence

**Verify:** `helm upgrade --install` deploys a model correctly from chart values alone.

---

### 12 CLI — deploy / list / status / delete

**Logic:**

- Typer app per `library-docs.md`, commands calling `helm upgrade --install` / `helm uninstall` under the hood
- `infergate list` reads from the `model_deployments` table
- `infergate status` reads live pod state via the Kubernetes Python client

**Verify:** full lifecycle — `infergate deploy` → `infergate list` shows it → `infergate status` shows it running → `infergate delete` removes it.

---

### 13 Cost-Estimate Gate (the differentiator feature)

**Logic:**

- `cost_estimator.py` computes estimated monthly cost from GPU tier × expected daily requests, using `config/pricing.yaml`
- If above `COST_CONFIRMATION_THRESHOLD_USD`, `deploy` requires `--confirm` and exits non-zero otherwise

**Verify:** deploying an expensive tier without `--confirm` is blocked with a clear message; with `--confirm` it proceeds.

---

## Phase 5 — FinOps / Observability

### 14 Prometheus + Grafana Install

**Setup:**

- Install both via their official Helm charts into an `observability` namespace

**Verify:** Grafana UI reachable via port-forward, default Prometheus data source connected.

---

### 15 Gateway Metrics Instrumentation

**Logic:**

- `gateway/metrics/prometheus.py` per `library-docs.md` — requests, latency, tokens, cost counters/gauges
- Metrics endpoint exposed at `/metrics`, scraped by Prometheus

**Verify:** `infergate_requests_total` and friends visible in the Prometheus UI after generating traffic.

---

### 16 Cost Calculator (CPM)

**Logic:**

- Computes cost per request from GPU price × wall-clock time attributed to that request
- Also computes cost-per-million-tokens (CPM) per model — the headline FinOps metric
- Heavily unit tested — this is the easiest place to be subtly wrong

**Verify:** unit tests pass; a hand-calculated expected cost for a known workload matches the reported cost within a small margin.

---

### 17 Grafana Dashboard

**Setup:**

- Dashboard JSON committed to `observability/grafana/dashboards/`
- Panels: cost per model, cost per team, GPU utilization, idle vs. active GPU-seconds, request latency

**Verify:** dashboard shows live, correct data after a load-generating script runs.

---

### 18 FinOps Playbook

**Setup:**

- `docs/finops-playbook.md` — right-sizing guidance, spot vs. on-demand tradeoffs, scale-to-zero for spiky traffic, referencing the CPM metric from real dashboard data

---

## Phase 6 — Autoscaling

### 19 KEDA Install + ScaledObject

**Setup:**

- Install KEDA via its Helm chart
- `charts/infergate-model/templates/scaledobject.yaml` per `library-docs.md`, triggered on `infergate_queue_depth`

**Verify:** manually push queue depth up via Redis and watch replica count increase.

---

### 20 Load Test

**Logic:**

- `tests/load/k6-script.js` per `library-docs.md`, ramp up then down
- Grafana graph captured showing replica scale-up and scale-down

**Verify:** saved graph/screenshot showing the full ramp cycle.

---

## Phase 7 — Chaos / Resilience

### 21 Pod-Kill Test

**Logic:**

- While a steady request stream runs, `kubectl delete pod <model-pod>`
- Confirm zero client-visible failures and that Kubernetes reschedules the pod

**Verify:** recorded logs/screen capture showing the kill, the retry, and the recovery.

---

## Phase 8 — Polish & Open Source

### 22 Remaining ADRs

**Setup:**

- ADR-003 (gateway design), ADR-004 (autoscaling trigger choice), ADR-005 (cost methodology)

---

### 23 README + Demo Video

**Setup:**

- Architecture diagram, quickstart, qualifications-mapping table, ~3 min demo video: deploy → request → cost dashboard → autoscale → chaos recovery

---

### 24 Cold-Read UX Test

**Verify:** someone unfamiliar with the project follows only the README to deploy a model; fix whatever confuses them.

---

### 25 External Open-Source Contribution

**Setup:**

- One real PR (docs fix, small feature, or bug fix) opened against InferCost or LLMKube

---

## Feature Count

| Phase | Features |
|---|---|
| Phase 1 — Foundation | 4 |
| Phase 2 — Real Serving Path | 1 |
| Phase 3 — Gateway | 5 |
| Phase 4 — Helm + CLI | 3 |
| Phase 5 — FinOps/Observability | 5 |
| Phase 6 — Autoscaling | 2 |
| Phase 7 — Chaos | 1 |
| Phase 8 — Polish/OSS | 4 |
| **Total** | **25** |
