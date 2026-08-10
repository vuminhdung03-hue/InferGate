# ADR-001 — A scoped single-runtime gateway instead of KServe + llm-d or LLMKube

**Status:** Accepted
**Date:** 2026-08-10

---

## Context

A team self-hosting an open-weight LLM hits three problems the moment a second team wants to call it:

1. Every model needs hand-written Kubernetes `Deployment`/`Service`/autoscaler YAML, copy-pasted and drifting.
2. Nobody knows what any single model or team actually costs until the cloud bill arrives — and by then it is unattributable.
3. Each calling team writes its own inference wrapper, with its own auth, its own retry logic, and its own idea of a timeout.

Mature projects already exist in this space. Building another one needs a reason, and the reason has to survive contact with someone who knows those projects well.

## Options considered

### Option 1 — KServe + llm-d

KServe provides general-purpose model-serving CRDs (`InferenceService`) spanning many runtimes and frameworks. llm-d adds LLM-specific distributed inference: KV-cache-aware routing, prefill/decode disaggregation, and cache-locality scheduling.

**For:** production-grade, broad ecosystem backing, and genuinely the right answer at scale. llm-d's cache-aware routing is a real efficiency win that InferGate does not attempt to match.

**Against:** the operational surface is large, and much of it exists to support generality InferGate does not need — multiple runtimes, multiple frameworks, multiple serving topologies. More decisively, **cost attribution is not a first-class concept in either.** You get serving; you reconstruct per-team cost afterwards from metrics, which means attribution is only as good as your labels happened to be at request time.

### Option 2 — LLMKube

A lighter Kubernetes-native operator for deploying LLMs — much closer in spirit to InferGate.

**For:** simple, focused, low operational overhead.

**Against:** it solves the deployment axis, not the gateway axis. There is no shared request path, so the auth/retry/rate-limit duplication across teams remains, and per-team cost attribution has nowhere to live.

### Option 3 — Scoped single-runtime vLLM + FastAPI gateway (chosen)

One runtime (vLLM), one gateway that every request passes through, one Helm chart, and cost as a column on every logged request.

---

## Decision

**Option 3.**

The problem InferGate targets is not *"serve models at maximum efficiency."* It is *"give a small platform team a paved road with cost accountability built in."* Those two goals have different centers of gravity, and optimizing for the second is what justifies not adopting the first.

Three things follow from that:

**Cost attribution is a schema decision, not a plugin.** Making `team` a required, defaultless Helm value and `cost_usd` a column on every row of `requests` is only possible when you own the request path. A general serving stack can be instrumented for cost, but attribution then depends on labels being right at request time and stays reconstructive rather than authoritative.

**A single runtime removes an entire axis of abstraction.** vLLM's OpenAI-compatible API *is* the contract. There is no runtime-agnostic indirection layer, which is why the mock server and real vLLM are a one-line config swap rather than a code branch.

**Operability is the feature.** A FastAPI gateway plus one Helm chart can be read end-to-end by one engineer in an afternoon. For a platform team of one to three people, that matters more than peak throughput.

---

## Consequences

### What this costs us

- **No KV-cache-aware routing.** Under multi-turn load, time-to-first-token will be worse than llm-d. This is the single largest efficiency concession and it is deliberate.
- **vLLM only.** No AMD, no Apple Silicon, no TensorRT-LLM. A team standardized on another runtime cannot use InferGate.
- **No multi-cluster or multi-region.** Single-cluster by design.
- **Re-implementing solved problems.** Retry and rate limiting are things a service mesh would provide. Owning them is the price of owning the request path — and the request path is what makes cost attribution authoritative.
- **A migration cliff exists.** If requirements grow toward large-scale multi-tenant serving, the answer is to move to llm-d, not to grow InferGate into it. This is a bet on scope, and bets on scope can be lost.

### What we get

- Cost per team and per model correct **by construction**, not reconstructed after the fact.
- A system small enough to operate, debug, and reason about without specialized knowledge.
- Backend swap (mock ↔ real vLLM) as a config change, enabling GPU-free local development.

### When to revisit this decision

Reopen this ADR if any of the following becomes true:

- More than roughly ten models are deployed concurrently.
- KV-cache locality, not GPU count, becomes the measured bottleneck.
- A second serving runtime becomes a hard requirement.

At that point the correct move is migration to llm-d, and this ADR should be superseded rather than amended.
