# Project Overview

## About the Project

InferGate is an open-source, Kubernetes-native platform for self-service LLM inference. A developer deploys a model with one CLI command instead of hand-writing Kubernetes YAML. Every request to that model flows through a gateway that handles auth, retries, and rate limiting, and logs its real cost in cost-per-million-tokens (CPM). A Grafana dashboard shows spend and GPU utilization per model and per team in real time. The cluster autoscales on request queue depth, not CPU, and recovers automatically if a serving pod dies mid-request.

It is a scoped, single-runtime alternative to heavier platforms like KServe+llm-d — see `docs/adrs/001-why-not-kserve-llmd.md` for the reasoning.

---

## The Problem It Solves

Any team that wants to self-host an LLM hits the same wall once more than one person wants to use it: nobody wants to hand-write Kubernetes Deployment/Service/HPA YAML per model, nobody has real GPU cost visibility until the cloud bill arrives, and nobody wants five teams each reinventing their own inference wrapper with its own auth and retry logic.

InferGate is the paved road: self-service deployment, a shared gateway, and built-in cost accountability, so platform teams stop reinventing this per model and per team.

---

## Interfaces

InferGate has three surfaces instead of pages:

```
CLI                  → infergate deploy / list / status / delete
Gateway API           → POST /v1/chat/completions (OpenAI-compatible)
Grafana Dashboards     → cost per model/team, GPU utilization, request latency
```

There is no custom web frontend. Grafana is the dashboard; the CLI is the control-plane interface.

---

## Core User Flow

### Deploying a model

- Developer runs `infergate deploy --model mistral-7b --gpu-tier small --team growth-eng`
- CLI estimates monthly cost from GPU tier × expected daily requests (asked interactively if not passed as a flag)
- If the estimate is above the configured threshold, the CLI requires `--confirm` before proceeding
- CLI renders the Helm chart with the given values and runs `helm upgrade --install`
- Model comes up behind the gateway, registered under its team label

### Calling a model

- Application sends a request to the gateway, not directly to the model
- Gateway authenticates the API key, checks the team's rate limit, and forwards the request to a healthy backend replica
- If a replica is unreachable, the gateway retries against another replica before failing
- Gateway logs tokens in/out, latency, and computed cost to Postgres and emits Prometheus metrics

### Watching cost and load

- Platform owner opens Grafana: cost per model, cost per team, GPU utilization, idle vs. active GPU-seconds
- Under load, KEDA scales replicas up based on pending request queue depth; scales back down once idle

### Recovering from failure

- If a serving pod is killed or crashes mid-request, the gateway retries against a healthy replica and Kubernetes reschedules the dead pod — no client-visible failure

---

## Data Architecture

### Model Deployment Data

- Lives in the `model_deployments` table (Postgres)
- Written by the CLI on `deploy` / `delete`
- Read by the gateway to know which models exist and where to route

### Request / Cost Data

- Lives in the `requests` table (Postgres) — one row per gateway request
- Written by the gateway after every request, success or failure
- Powers the Grafana cost dashboards
- Append-only — never modified after being written

---

## Features In Scope

- Mock model server matching vLLM's OpenAI-compatible API shape, for GPU-free local dev
- Real vLLM deployment on Kubernetes (validated at least once against a rented GPU)
- FastAPI gateway: auth, retry-on-failure, per-team rate limiting, request logging
- Helm chart for model deployment, parameterized by model name, GPU tier, replica count, team
- CLI (`infergate deploy/list/status/delete`) with a pre-deploy cost estimate and confirmation gate
- Cost calculator using cost-per-million-tokens (CPM), configurable GPU price table
- Prometheus metrics + Grafana dashboard: cost per model/team, GPU utilization, latency, idle GPU-seconds
- KEDA autoscaling on request queue depth
- Load test (k6) demonstrating scale-up and scale-down
- Chaos test demonstrating gateway retry + Kubernetes pod rescheduling
- FinOps playbook document
- Architecture decision records (ADRs) for major tradeoffs
- One real open-source contribution to an existing project in this space (InferCost or LLMKube)

## Features Out of Scope

- Distributed KV-cache-aware routing (that's llm-d's job, not InferGate's)
- Multi-runtime GPU fleet support (NVIDIA/AMD/Apple Silicon) — vLLM only
- A custom web frontend/dashboard — Grafana is the dashboard
- Multi-cluster / multi-region deployment
- Fine-tuning or training workloads — inference only
- SSO / enterprise auth — API keys only
- Billing/invoicing integration — cost visibility only, no payment processing
- Prompt-injection or PII guardrails at the gateway edge

---

## Target User

A platform engineer or ML infrastructure engineer at a company that:

- Self-hosts (or is evaluating self-hosting) at least one open-weight LLM
- Has more than one internal team or application that wants to call that model
- Has no current visibility into GPU cost per model or per team
- Wants a paved-road self-service path instead of every team writing its own Kubernetes YAML

---

## Success Criteria

- A developer can deploy a model with a single CLI command, without touching raw Kubernetes YAML
- A request through the gateway returns a real completion, with retry against a healthy replica if one backend pod is down
- The Grafana dashboard shows accurate cost-per-million-tokens for a known workload, verified by hand against published GPU pricing
- A load test visibly triggers KEDA scale-up and scale-down, captured as a graph
- A chaos test (killing a serving pod mid-traffic) shows zero client-visible failures
- The FinOps playbook gives a concrete, defensible cost-optimization recommendation
- The repo is public and documented well enough that a cold reader can deploy a model from the README alone
- At least one real PR has been merged (or opened) against an external open-source project in this space
