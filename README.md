# InferGate

**Self-service LLM inference on Kubernetes, with GPU cost accountability built into the request path.**

Deploy a model with one CLI command instead of hand-written Kubernetes YAML. Every request flows through a gateway that handles auth, retries, and rate limiting — and logs what it actually cost, attributed to the team that made it.

> **Status: in development — Phase 1 of 8.**
> The architecture and build plan are complete; the serving path is being built in order. There is no quickstart yet because the deploy path does not exist yet. See [Build status](#build-status) for exactly where things stand.

---

## The problem

Any team self-hosting an LLM hits the same wall once more than one person wants to use it:

- Nobody wants to hand-write a `Deployment`/`Service`/autoscaler per model.
- Nobody has real GPU cost visibility until the cloud bill arrives — and by then it cannot be attributed to a team.
- Five teams each reinvent their own inference wrapper, with five different retry policies.

InferGate is the paved road: self-service deployment, one shared gateway, and cost accountability that is correct by construction rather than reconstructed from metrics afterwards.

---

## How it works

```
  ┌── DEPLOY PATH ─────────────────────────────────────────────────────┐
  │                                                                    │
  │   $ infergate deploy --model mistral-7b                            │
  │                      --gpu-tier small --team growth-eng            │
  │                     │                                              │
  │                     ▼                                              │
  │        estimate monthly cost ◀────── config/pricing.yaml            │
  │                     │                                              │
  │          over threshold? ──▶ require --confirm ──▶ exit 1          │
  │                     │                                              │
  │                     ▼                                              │
  │            helm upgrade --install                                  │
  │                     │                                              │
  └─────────────────────┼──────────────────────────────────────────────┘
                        ▼
              ┌───────────────────────┐
              │   vLLM pods           │  one Deployment per model,
              │   (OpenAI-compatible) │  every resource carries a
              └───────────────────────┘  `team` label
                        ▲       ▲
  ┌── REQUEST PATH ──────┼───────┼─────────────────────────────────────┐
  │                      │       │                                     │
  │  your app            │       └── retry on unreachable replica      │
  │  (x-api-key)         │                                             │
  │      │               │                                             │
  │      ▼               │                                             │
  │  ┌─────────────────────────────────────┐                           │
  │  │        FastAPI Gateway              │                           │
  │  │  auth ▸ rate limit ▸ proxy ▸ log    │                           │
  │  └────┬──────────┬──────────┬──────────┘                           │
  │       │          │          │                                      │
  │       ▼          ▼          ▼                                      │
  │  PostgreSQL    Redis    Prometheus ──▶ Grafana                     │
  │  requests,     rate      metrics       cost per model / per team,  │
  │  deployments   limits,                 GPU utilization, latency    │
  │  (cost per     queue                                               │
  │   request)     depth                                               │
  │                  │                                                 │
  └──────────────────┼─────────────────────────────────────────────────┘
                     ▼
              KEDA scales replicas on queue depth — not CPU — to zero when idle
```

Three surfaces, no custom web frontend:

| Surface | What it is |
|---|---|
| **CLI** | `infergate deploy / list / status / delete` |
| **Gateway API** | `POST /v1/chat/completions` — OpenAI-compatible |
| **Grafana** | Cost per model and team, GPU utilization, latency |

---

## Why not KServe + llm-d?

Because the goal is not *serve models at maximum efficiency* — it is *give a small platform team a paved road with cost accountability built in*, and those have different centers of gravity.

Cost attribution is a schema decision, not a plugin. Making `team` a required value and `cost_usd` a column on every logged request is only possible when you own the request path. The tradeoff is real and deliberate: InferGate gives up KV-cache-aware routing, multi-runtime support, and multi-cluster deployment to get there.

The full argument, including what this costs and when to abandon it, is in **[ADR-001](docs/adrs/001-why-not-kserve-llmd.md)**.

---

## Build status

| Phase | Features | Status |
|---|---|---|
| 1 — Foundation | Repo, local env, mock model server, raw K8s YAML | 1 / 4 |
| 2 — Real serving path | vLLM on a real GPU | 0 / 1 |
| 3 — Gateway | Skeleton, auth, retry, rate limiting, request logging | 0 / 5 |
| 4 — Helm + CLI | Chart, CLI lifecycle, cost-estimate gate | 0 / 3 |
| 5 — FinOps | Prometheus/Grafana, metrics, CPM calculator, dashboard, playbook | 0 / 5 |
| 6 — Autoscaling | KEDA ScaledObject, k6 load test | 0 / 2 |
| 7 — Chaos | Pod-kill resilience test | 0 / 1 |
| 8 — Polish & OSS | ADRs, README + demo, cold-read test, upstream PR | 0 / 4 |

---

## Success criteria

What "done" means, and which feature delivers each:

| Criterion | Delivered by | Status |
|---|---|---|
| Deploy a model with one CLI command, no raw Kubernetes YAML | Helm chart, CLI | ☐ |
| A gateway request returns a real completion, retrying if a backend pod is down | Proxy + retry | ☐ |
| Grafana shows cost-per-million-tokens verified by hand against published GPU pricing | CPM calculator, dashboard | ☐ |
| A load test visibly triggers KEDA scale-up and scale-down, captured as a graph | KEDA, k6 load test | ☐ |
| Killing a serving pod mid-traffic produces zero client-visible failures | Chaos pod-kill test | ☐ |
| The FinOps playbook makes a concrete, defensible cost recommendation | FinOps playbook | ☐ |
| A cold reader can deploy a model from the README alone | Repo, README, cold-read test | ☐ |
| At least one real PR opened against an external project in this space | Upstream contribution | ☐ |

---

## Engineering competencies demonstrated

| Competency | Where InferGate demonstrates it |
|---|---|
| Kubernetes-native platform design | Helm chart parameterized per model/tier/team; one namespace, label-based isolation |
| Autoscaling for inference workloads | KEDA triggered on request queue depth rather than CPU, with scale-to-zero on idle |
| GPU cost engineering (FinOps) | Cost-per-million-tokens calculator, per-team attribution, FinOps playbook |
| Production API design | Async FastAPI gateway: API-key auth, backend retry, Redis sliding-window rate limiting |
| Observability | Prometheus instrumentation with model/team labels; Grafana cost and utilization dashboards |
| Resilience engineering | Chaos pod-kill test proving zero client-visible failures under pod loss |
| Data modelling for cost attribution | Append-only `requests` table; `team` required at deploy time, never defaulted |
| Technical decision-making | ADRs recording each major tradeoff and the conditions that would reverse it |

---

## Scope

**In scope:** vLLM serving on Kubernetes · GPU-free mock backend for local dev · gateway auth, retry, rate limiting, request logging · Helm chart · CLI with a pre-deploy cost gate · CPM cost calculator · Prometheus + Grafana · KEDA queue-depth autoscaling · load and chaos tests · FinOps playbook · ADRs.

**Out of scope, deliberately:** distributed KV-cache-aware routing (that is llm-d's job) · multi-runtime GPU fleets · a custom web frontend (Grafana is the dashboard) · multi-cluster/multi-region · fine-tuning or training · SSO (API keys only) · billing and invoicing · prompt-injection and PII guardrails.

---

## Documentation

- [ADR-001 — Why not KServe + llm-d](docs/adrs/001-why-not-kserve-llmd.md)
- [Architecture](context/architecture.md) — stack, boundaries, data flows, schema, invariants
- [Project overview](context/project-overview.md) — problem, user flows, success criteria
- [Build plan](context/build-plan.md) — all 25 features in build order

---

## License

[MIT](LICENSE)
