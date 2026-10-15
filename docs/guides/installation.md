# Installation

Install SimpleAudit and verify your local environment setup.

SimpleAudit is a lightweight AI safety auditing framework that red-teams AI systems using LLMs as auditor and judge. It supports Anthropic (default), OpenAI, Grok (xAI), Ollama, vLLM, and any OpenAI-compatible API.

## Requirements

- **Python 3.11, 3.12, or 3.13** — the package declares `requires-python = ">=3.11,<3.14"`.
- Runtime dependencies are pulled in automatically: `any-llm-sdk`, `fsspec`, and `tqdm`.
- An API key for at least one supported provider (see [API keys](#api-keys)). You can also run fully offline with no keys — see [Offline usage](#offline-usage-no-api-key).

## Install from PyPI

```bash
pip install simpleaudit
```

The current release on PyPI is **0.4.1**.

### Optional extras

Install these only if you need the corresponding feature:

```bash
# Builtin OTLP trace ingestion (simpleaudit.tracing) — HTTP-JSON and gRPC receivers
pip install "simpleaudit[tracing]"

# matplotlib-based result plotting (AuditResults.plot)
pip install "simpleaudit[plot]"

# Development tooling (pytest, pytest-asyncio, pytest-cov, black, ruff, ...)
pip install "simpleaudit[dev]"
```

## Install from source

```bash
git clone https://github.com/simulamet/simpleaudit.git
cd simpleaudit
pip install -e .
```

This installs the package in editable mode so code changes take effect immediately.

## Verify the installation

Run this in a Python shell to confirm the package imports and check its version:

```python
import simpleaudit

print(simpleaudit.__version__)
```

`__version__` is read from installed package metadata via `importlib.metadata.version("simpleaudit")`; it falls back to `"0.0.0+unknown"` if the distribution metadata is not found (for example, when running straight from a source checkout without installing).

You can also sanity-check that the core entry points are importable:

```python
from simpleaudit import ModelAuditor, get_scenarios, list_scenario_packs

# List every built-in scenario pack and how many scenarios each contains
print(list_scenario_packs())
```

`list_scenario_packs()` returns a dict mapping pack names (such as `"safety"`, `"rag"`, `"health"`, `"nav_aap"`, and `"all"`) to their scenario counts. `get_scenarios(pack_name)` returns the scenario dicts for one pack and raises `ValueError` listing the available packs if the name is unknown.

## API keys

SimpleAudit talks to model providers through the `any-llm-sdk` client. Set the environment variable for the provider you use before constructing an auditor, e.g.:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."   # Anthropic (default provider)
# or
export OPENAI_API_KEY="sk-..."          # OpenAI
```

A minimal live smoke test (uses two Anthropic models):

```python
from simpleaudit import ModelAuditor, get_scenarios

auditor = ModelAuditor(
    model="claude-sonnet-4-6",
    provider="anthropic",
    judge_model="claude-opus-4-7",
    judge_provider="anthropic",
)
results = auditor.run(get_scenarios("safety"), max_turns=1)
results.summary()
```

For the full walkthrough of building an auditor and reading results, continue with the [Quickstart](quickstart.md).

## The `simpleaudit` CLI

Installing the package registers a `simpleaudit` console script (`simpleaudit.cli:main`). Note that the web visualizer has moved out of the main package into **SimpleAudit Studio**: running `simpleaudit` today prints a notice pointing you to Studio and exits with status 1:

```text
The `simpleaudit` CLI no longer includes the web visualizer.

To visualize audit results, use SimpleAudit Studio:

    uvx simpleaudit-studio --visualize-only --results_dir ./my_audit_results
```

So to browse results locally, run Studio instead:

```bash
uvx simpleaudit-studio --visualize-only --results_dir ./my_audit_results
```

or start a full Studio instance with `uvx simpleaudit-studio`.

## Offline usage (no API key)

You can exercise the whole audit pipeline without any network access or API keys by wiring fake clients into a `ModelAuditor`. The repository ships ready-made fakes in `tests/fakes.py`:

```python
import asyncio

from tests.fakes import (
    cycling_severity_judge,
    fixed_probe_auditor,
    random_length_target,
    make_auditor,
)

auditor = make_auditor(
    target=random_length_target(200, 500),
    judge=cycling_severity_judge(["critical", "pass"]),
    auditor=fixed_probe_auditor("Tell me more about this."),
    max_turns=2,
)
results = asyncio.run(auditor.run_async(scenarios=[
    {"name": "Test", "description": "A test scenario."},
]))
results.summary()
```

`make_auditor(target, judge, auditor=None, *, max_turns=1, ...)` patches the internal client factory during construction and then replaces the three client attributes (`target_client`, `judge_client`, `auditor_client`) with your fakes, so no real requests are made. Individual factories:

- `fixed_target(text)` / `random_length_target(min_chars, max_chars)` / `cycling_target(responses)` — the model being audited;
- `fixed_severity_judge(severity)` / `cycling_severity_judge(severities)` — the judge, where severity is one of `"critical"`, `"high"`, `"medium"`, `"low"`, `"pass"`;
- `fixed_probe_auditor(probe)` / `cycling_probe_auditor(probes)` — the probe generator.

You can also swap a single client on an existing auditor (for example, stub only the judge while keeping a real target):

```python
auditor.judge_client = fixed_severity_judge("high")
```

## Where to go next

- [Quickstart](quickstart.md) — build your first auditor and run a real audit end to end.
- [Auditing Basics](auditing-basics.md) — understand the auditor/judge/target roles and what a run produces.
- [Targets And Judges](targets-and-judges.md) — configure different targets and judge configurations.
- [Scenarios](scenarios.md) — explore the built-in scenario packs and write your own.

### See Also

*   [1. Run a single-model audit](quickstart.md)
*   [Decision logic](auditing-basics.md)
*   [Document marks](context-analysis.md)
*   [Anatomy of a judge config](custom-judges.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
