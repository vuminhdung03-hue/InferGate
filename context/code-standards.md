# Code Standards

Implementation rules and conventions for the entire project. The AI agent must follow these in every session without exception. These rules prevent pattern drift across sessions.

---

## Engineering Mindset

- **Think before implementing** — understand what is being built and why before writing a single line. Use the `architect` skill before starting any new feature.
- **Read context files first** — never assume, always verify against `architecture.md` and `project-overview.md`.
- **Scope is sacred** — only build what the current phase in `build-plan.md` requires. Never go beyond scope even if it seems helpful.
- **Every feature must be testable** — if it cannot be verified immediately after implementation (a curl call, a `kubectl get`, a dashboard panel), it is incomplete.
- **Clean over clever** — simple, readable code a junior engineer can follow beats a clever abstraction.
- **One thing at a time** — complete one feature fully, including its test, before touching the next.
- **Failures are expected** — every gateway call, CLI command, and Helm operation is wrapped in error handling; a failure in one request or deploy never crashes the platform.

---

## Python

- Python 3.11+ throughout.
- Type hints on every function signature — parameters and return types. No exceptions.
- Never use bare `except:` — always catch a specific exception or `Exception` with logging.
- Use `pydantic` (v2) models for every request/response shape and every config file schema — never pass untyped dicts across module boundaries.
- Use `ruff` for linting and formatting; run it before every commit.
- Use `mypy --strict` on `gateway/` and `cli/` — fix type errors, never suppress with `# type: ignore` without a comment explaining why.
- Prefer `httpx` (async) over `requests` for outbound HTTP calls — the gateway is async end to end.
- Use `pathlib.Path`, never raw string path concatenation.

---

## FastAPI Conventions

- Routers live in `gateway/routers/` — one file per resource area (`completions.py`, `health.py`).
- Business logic never lives directly in a route handler — route handlers call into `gateway/core/`.
- Every route has an explicit `response_model`.
- Dependencies (DB session, current API key/team) are injected via FastAPI `Depends`, never constructed inline in the handler.

```python
# gateway/routers/completions.py

from fastapi import APIRouter, Depends, HTTPException
from gateway.core.auth import get_current_team
from gateway.core.completions import handle_completion
from gateway.models import CompletionRequest, CompletionResponse

router = APIRouter()

@router.post("/v1/chat/completions", response_model=CompletionResponse)
async def chat_completions(
    body: CompletionRequest,
    team: str = Depends(get_current_team),
) -> CompletionResponse:
    try:
        return await handle_completion(body, team)
    except BackendUnavailableError as e:
        logger.error("[gateway/completions] %s", e)
        raise HTTPException(status_code=503, detail="Model temporarily unavailable")
```

- Every route handler has a try/except around calls to `core/`.
- Errors are logged with a `[module/function]` prefix, matching the pattern below.
- Never return raw internal error messages to the client — always a generic, human-readable message.

---

## CLI Conventions (Typer)

- Commands live in `cli/infergate/commands/`, one file per command.
- Every command returns a non-zero exit code on failure — never exit 0 after a failure.
- Use `rich` for all table/status output — never raw `print()` for structured data.
- Every destructive or cost-incurring action (`deploy`, `delete`) requires an explicit `--confirm` flag once above a defined threshold — never silently proceed.

```python
# cli/infergate/commands/deploy.py

import typer
from infergate.config import COST_CONFIRMATION_THRESHOLD_USD
from infergate.cost_estimator import estimate_monthly_cost
from infergate.helm import render_and_install

app = typer.Typer()

@app.command()
def deploy(
    model: str,
    gpu_tier: str,
    team: str,
    expected_daily_requests: int = typer.Option(1000, help="Used for cost estimate"),
    confirm: bool = typer.Option(False, "--confirm"),
) -> None:
    estimate = estimate_monthly_cost(gpu_tier, expected_daily_requests)
    if estimate.monthly_usd > COST_CONFIRMATION_THRESHOLD_USD and not confirm:
        typer.secho(
            f"Estimated cost ${estimate.monthly_usd:.2f}/mo exceeds threshold. "
            f"Re-run with --confirm to proceed.",
            fg=typer.colors.YELLOW,
        )
        raise typer.Exit(code=1)
    render_and_install(model, gpu_tier, team)
```

---

## Helm / Kubernetes Conventions

- Every value that could plausibly differ between deployments lives in `values.yaml` — never hardcoded in a template.
- Image tags are always pinned to a specific version — never `latest`.
- Every container sets both `resources.requests` and `resources.limits` — no unbounded pods.
- Every Deployment carries a `team` label — cost attribution and the Grafana dashboards depend on it.
- Chart version is bumped in `Chart.yaml` any time `templates/` changes.
- `helm template` is run and diffed before every `helm upgrade --install` in CI.

---

## File and Folder Naming

- Folders: kebab-case — `mock-model-server`, `gpu-pricing`.
- Python modules/files: snake_case — `cost_estimator.py`, `rate_limit.py`.
- Pydantic model classes: PascalCase — `CompletionRequest`, `ModelDeployment`.
- Helm chart template files: lowercase, matching the Kubernetes resource kind they render, with no separator — `deployment.yaml`, `service.yaml`, `scaledobject.yaml`, `configmap.yaml`. This is the one exception to the kebab-case folder rule above: it keeps template filenames aligned with `kind:` so a reader can map a rendered resource back to its template at a glance.
- CLI command files: snake_case matching the command name — `deploy.py` for `infergate deploy`.
- Test files: `test_<module>.py`, mirroring the module under test.

---

## Error Handling

- Never use empty `except` blocks — always log or handle.
- Log messages always include a `[module/function]` prefix: `logger.error("[gateway/completions] backend unreachable: %s", e)`.
- User-facing errors (CLI output, gateway HTTP responses) must be human-readable — never a raw stack trace or internal exception message.
- Gateway retries are always logged, even on eventual success — never silent.
- API route errors return a generic message with an appropriate status code — never expose internals.

---

## Testing Standards

- Every `gateway/core/` module has a corresponding `test_*.py` with pytest.
- Every CLI command has a test that runs it against `helm template` (rendered output checked), never against a live cluster in unit tests.
- Integration tests (`tests/integration/`) run against a real `kind` cluster in CI — these are allowed to be slower.
- Load tests (`tests/load/`) are k6 scripts, run manually or in a scheduled CI job — not part of the default `pytest` run.
- Mock the model backend in all gateway unit tests — never call a real model server in `pytest`.

---

## Prometheus Metric Naming

- All InferGate metrics are prefixed `infergate_` — never a bare metric name.
- Counters end in `_total`. Gauges describe a current value with no suffix. Histograms are unit-suffixed (e.g. `_seconds`).
- Every metric that can be attributed to a team or model carries those as labels — never a metric InferGate can't attribute cost to.

---

## Cost Threshold Constant

The cost confirmation threshold is defined once as a constant. Never hardcode this value anywhere else.

```python
# cli/infergate/config.py
COST_CONFIRMATION_THRESHOLD_USD = 500.0
```

Import and use `COST_CONFIRMATION_THRESHOLD_USD` everywhere this value is needed.

---

## Import Conventions

- Absolute imports only within each package (`gateway.core.auth`, `infergate.commands.deploy`) — never relative imports that climb more than one level.
- `gateway/` and `cli/` are separate installable packages — `cli/` never imports directly from `gateway/` internals. If they need to share a type (e.g. the cost estimate shape), it lives in a small shared module or is duplicated deliberately with a comment noting why.

---

## Comments

- No comments explaining what the code does — code should be self-explanatory through naming.
- Comments only for why — a non-obvious tradeoff, a workaround for a vLLM/KEDA quirk, a reference to an ADR.
- Never leave `TODO` comments in committed code — open a GitHub issue instead.

---

## Dependencies

Never install a new package without a clear reason. Before installing anything, check:

1. Does the standard library already do this?
2. Does an already-approved dependency below cover it?
3. Is there a simpler native solution (e.g. a plain `kubectl`/`helm` shellout vs. a heavy SDK)?

**Approved dependencies for this project:**

- `fastapi`, `uvicorn` — gateway
- `pydantic` (v2) — schema validation throughout
- `httpx` — async HTTP client
- `sqlalchemy` (async) + `asyncpg` — Postgres access
- `redis` (redis-py, async) — rate limiting, queue depth signal
- `prometheus-client` — metrics
- `typer`, `rich` — CLI
- `kubernetes` (official Python client) — used only where a `helm`/`kubectl` shellout genuinely isn't sufficient
- `pytest`, `pytest-asyncio`, `httpx` (test client) — testing
- `ruff`, `mypy` — lint/type-check

Do not install any other package without updating this list first.
