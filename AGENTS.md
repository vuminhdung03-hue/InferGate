# AGENTS.md

This file is the entry point for any AI coding agent working on InferGate. Read this first, every session.

---

## Read Order

1. `context/project-overview.md` — what InferGate is and why it exists
2. `context/architecture.md` — stack, folder structure, data flow, invariants
3. `context/code-standards.md` — conventions the agent must follow in every file it touches
4. `context/library-docs.md` — project-specific usage patterns for every third-party library/tool
5. `context/build-plan.md` — the phased feature list, in build order
6. `context/progress-tracker.md` — what's done, what's next — update this after every completed feature

---

## Installed Skills

| Skill | Use for |
|---|---|
| `architect` | Before writing any new feature — think through the design, surface decisions, confirm the plan |
| `review` | After building a feature — verify it matches the plan and the architecture |
| `recover` | When a build breaks — diagnose the failure type before patching |
| `imprint` | After building any CLI command, API route, or Helm template — extract the pattern so future work stays consistent |
| `remember` | End of session — save state; start of session — restore context |

---

## Order of Authority for Library APIs

```
MCP server (real-time docs/cluster state, if configured) → Skills listed above → context/library-docs.md → general training knowledge
```

Kubernetes, Helm, vLLM, and KEDA APIs change often between versions. Never trust general training knowledge alone for exact flag names, CRD fields, or config schema — check `library-docs.md` first, and prefer an MCP/skill if one is configured.

---

## Non-Negotiables

- No hand-written Kubernetes YAML in the deploy path — everything ships through the Helm chart in `charts/`. (Phase 1's hand-written YAML is a deliberate one-time learning step, not a pattern to repeat.)
- Every gateway request is logged with cost data, success or failure. No silent/unlogged inference calls.
- Every agent function (gateway route, CLI command, agent module) returns and logs errors — nothing fails silently, per `code-standards.md`.
- Scope is sacred — build only what the current phase in `build-plan.md` requires.
