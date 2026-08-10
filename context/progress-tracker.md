# Progress Tracker

Update this file after every completed feature. Any AI agent reading this should immediately know what is done, what is in progress, and what is next.

---

## Current Status

**Phase:** Phase 1 — Foundation
**Last completed:** 01 Repo & Differentiation (2026-08-10)
**Next:** 02 Local Environment — `kind` and `helm` are not installed yet; both need installing before `kind create cluster --name infergate`

---

## Progress

### Phase 1 — Foundation

- [x] 01 Repo & Differentiation
- [ ] 02 Local Environment
- [ ] 03 Mock Model Server
- [ ] 04 Raw Kubernetes YAML (by hand, once)

### Phase 2 — Real Serving Path

- [ ] 05 Real vLLM Deployment

### Phase 3 — Gateway Layer

- [ ] 06 Gateway Skeleton + Health Check
- [ ] 07 Auth (API Keys)
- [ ] 08 Proxy + Retry Logic
- [ ] 09 Rate Limiting
- [ ] 10 Request Logging

### Phase 4 — Helm Chart + CLI

- [ ] 11 Helm Chart
- [ ] 12 CLI — deploy / list / status / delete
- [ ] 13 Cost-Estimate Gate

### Phase 5 — FinOps / Observability

- [ ] 14 Prometheus + Grafana Install
- [ ] 15 Gateway Metrics Instrumentation
- [ ] 16 Cost Calculator (CPM)
- [ ] 17 Grafana Dashboard
- [ ] 18 FinOps Playbook

### Phase 6 — Autoscaling

- [ ] 19 KEDA Install + ScaledObject
- [ ] 20 Load Test

### Phase 7 — Chaos / Resilience

- [ ] 21 Pod-Kill Test

### Phase 8 — Polish & Open Source

- [ ] 22 Remaining ADRs
- [ ] 23 README + Demo Video
- [ ] 24 Cold-Read UX Test
- [ ] 25 External Open-Source Contribution

---

## Decisions Made During Build

**2026-08-05 — `AGENTS.md` lives at the repo root, not in `context/`.**
`library-docs.md` and the folder tree in `architecture.md` both specify the root, and root is the conventional location other coding agents (Codex, Cursor) look for it. Its own read-order paths (`context/project-overview.md`, …) only resolve correctly from the root.

**2026-08-05 — Helm template files are named for the Kubernetes resource kind they render, lowercase with no separator (`scaledobject.yaml`).**
`code-standards.md` previously mandated kebab-case with `scaled-object.yaml` as its example, contradicting `architecture.md` and `library-docs.md`, which both specify `scaledobject.yaml`. Settled on `scaledobject.yaml` — it matches the KEDA CRD kind `ScaledObject`, so a rendered resource maps back to its template at a glance. `code-standards.md` now carries this as an explicit exception to the kebab-case rule for folders.

**2026-08-10 — Folder skeleton is directories + `.gitkeep` only, no stub source files.**
An empty `charts/infergate-model/templates/deployment.yaml` would make `helm template` render nothing silently instead of failing loudly, and empty modules read as work already done. Files get created when the feature that owns them is built.

**2026-08-10 — README ships without a quickstart until Phase 4.**
A quickstart that cannot be followed is worse than none. The README carries a phase-status banner instead; build-plan feature 23 is where the real quickstart lands.

**2026-08-10 — MIT license, copyright holder `vuminhdung03-hue`.**
As specified in `build-plan.md`. Noted at decision time: every project InferGate depends on (Kubernetes, Helm, KEDA, vLLM) is Apache 2.0, which adds an express patent grant that MIT leaves implicit. Relicensing is easy while there is a single author and hard once there are outside contributors — so revisit before accepting the first external PR, if at all.

**2026-08-05 — Skills are symlinked from `agent-skills/` into the local agent tooling directory.**
The agent runtime only discovers skills inside its own config directory, so the five skills AGENTS.md lists as installed would not otherwise load. Symlinks rather than copies keep `agent-skills/` the single source of truth — editing a `SKILL.md` there takes effect immediately with no sync step. That tooling directory is gitignored: it is local setup, not project content. Note for non-macOS/Linux contributors: these are POSIX symlinks; a Windows checkout without symlink support will need copies instead.

---

## Notes

_Add notes here as the build progresses — workarounds, patterns, anything that differs from the context files._
