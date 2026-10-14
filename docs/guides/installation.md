# Installation

SimpleAudit is a lightweight AI safety auditing framework that red-teams AI systems using LLMs as auditor and judge. This page covers installing the package from PyPI (or GitHub) and verifying that both the Python import and the `simpleaudit` CLI entry point work.

## Requirements

- **Python 3.11, 3.12, or 3.13** — the project declares `requires-python = ">=3.11,<3.14"` in `pyproject.toml`.
- Runtime dependencies are pulled in automatically: `any-llm-sdk`, `fsspec`, and `tqdm`.

## Install from PyPI (recommended)

```bash
pip install -U simpleaudit
```

With plotting support (for `AuditResults.plot()`):

```bash
pip install -U simpleaudit[plot]
```

Optional extras available in `pyproject.toml`:

| Extra | Adds |
| --- | --- |
| `simpleaudit[tracing]` | OTLP trace ingestion (`aiohttp`, `grpcio`, `opentelemetry-proto`) for the builtin OTLP receiver in `simpleaudit.tracing` |
| `simpleaudit[plot]` | `matplotlib` for result charts |
| `simpleaudit[dev]` | pytest, black, ruff, and other development tools |
| `simpleaudit[all]` | `plot` + `dev` |

## Install from GitHub (latest development features)

```bash
pip install -U git+https://github.com/simulamet/simpleaudit.git
```

## Verify the installation

### 1. Check the version

The package exposes `__version__`, which is read from installed package metadata via `importlib.metadata.version("simpleaudit")` (falling back to `"0.0.0+unknown"` if the distribution is not found):

```python
import simpleaudit

print(simpleaudit.__version__)
```

### 2. Check the CLI entry point

The console script `simpleaudit` is declared in `pyproject.toml` as:

```toml
[project.scripts]
simpleaudit = "simpleaudit.cli:main"
```

It points at [`simpleaudit.cli.main`](https://github.com/simulamet/simpleaudit/blob/main/simpleaudit/cli.py), defined in `simpleaudit/cli.py` as `def main()`. Running it verifies the entry point resolves and executes:

```bash
simpleaudit
```

Expected output (the web visualizer has moved to SimpleAudit Studio, so `main()` prints a pointer and exits with status code 1):

```text
The `simpleaudit` CLI no longer includes the web visualizer.

To visualize audit results, use SimpleAudit Studio:

    uvx simpleaudit-studio --visualize-only --results_dir ./my_audit_results

This starts a local web server (no audit worker) that browses a folder
of SimpleAudit JSON results, and can export a standalone HTML file.

See: https://github.com/SimulaMet/SimpleAuditStudio
```

Seeing this message confirms the `simpleaudit` command is on your PATH and wired to `simpleaudit.cli:main`. The non-zero exit status is intentional — the CLI's only remaining job is to redirect you to [SimpleAudit Studio](https://github.com/SimulaMet/SimpleAuditStudio) for visualization.

### 3. Smoke-test the public API

A quick import of the top-level names confirms the package structure is intact:

```python
from simpleaudit import ModelAuditor, get_scenarios, list_scenario_packs

print(list_scenario_packs())
```

`list_scenario_packs()` returns a dict mapping built-in pack names to their scenario counts (e.g. `"safety": <n>`), and `get_scenarios(pack_name)` loads any of them by name.

## Offline usage (no API key)

You can exercise the full audit pipeline without any API keys or network access by wiring fake LLM clients into `ModelAuditor`. The repository ships fakes in `tests/fakes.py` (used throughout the test suite and demonstrated in `examples/fake_audit.py`). From a checkout of the repo:

```python
import asyncio

from tests.fakes import fixed_probe_auditor, fixed_severity_judge, fixed_target, make_auditor

auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("Tell me more about this topic."),
    max_turns=1,
)

result = asyncio.run(auditor.run_scenario(name="Test", description="A smoke-test scenario."))
print(result.severity)
```

`make_auditor(target=..., judge=..., auditor=...)` patches client creation during `ModelAuditor.__init__` so no network call is ever made; the three fakes stand in for the target model, the judge, and the probe generator respectively. For a multi-scenario run, pass a list of `{"name": ..., "description": ...}` dicts to `auditor.run_async(scenarios=[...], max_turns=1)` and call `results.summary()` on the returned `AuditResults`.

## Where to go next

- [Quickstart](guides/quickstart.md) — run your first real audit with `ModelAuditor`.
- [Auditing Basics](guides/auditing-basics.md) — understand scenarios, severities, and how results are graded.
- [Targets And Judges](guides/targets-and-judges.md) — configure providers, targets, and judge models.
- [Tracing And Observability](guides/tracing-and-observability.md) — capture spans with the `simpleaudit[tracing]` extra.

### See Also

*   [How an audit works](quickstart.md)
*   [Decision logic](auditing-basics.md)
*   [Span verification](context-analysis.md)
*   [Customize an existing judge: `customize_judge`](custom-judges.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
