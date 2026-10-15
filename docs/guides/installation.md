# Installation

SimpleAudit is a lightweight AI safety auditing framework: it red-teams an AI system (the **target**) using LLMs as auditor and judge. This page walks through installing the package, wiring up your local environment, and verifying that everything works — including a fully offline check that needs no API keys at all.

## Requirements

From the project metadata (`pyproject.toml`):

- **Python 3.11, 3.12, or 3.13** (`requires-python = ">=3.11,<3.14"`)
- Runtime dependencies (installed automatically with the package):
  - `any-llm-sdk >= 1.14.0` — the multi-provider LLM client used for target, judge, and auditor models
  - `fsspec >= 2023.1.0`
  - `tqdm >= 4.66.0` — progress bars during runs

Supported providers include Anthropic (Claude), OpenAI, Grok (xAI), Ollama (local models), vLLM (local model serving), and any OpenAI-compatible API.

## Install from PyPI

```bash
pip install simpleaudit
```

The package is published on [PyPI](https://pypi.org/project/simpleaudit/) under the name `simpleaudit`.

### Optional extras

| Extra | Installs | For what |
|---|---|---|
| `tracing` | `aiohttp`, `grpcio`, `opentelemetry-proto` | The builtin OTLP trace receiver in `simpleaudit.tracing`. The core audit engine works with no tracing; install this extra only if you want to capture spans from instrumented targets. See [Tracing And Observability](tracing-and-observability.md). |
| `plot` | `matplotlib` | Result plotting via `AuditResults.plot()`. |
| `dev` | `pytest`, `pytest-asyncio`, `pytest-cov`, `pytest-xdist`, `pytest-testmon`, `black`, `ruff` | Running the test suite and linting. |
| `all` | `plot` + `dev` | Everything except `tracing`. |

```bash
# Example: install with plotting support
pip install "simpleaudit[plot]"
```

## Install from source

```bash
git clone https://github.com/simulamet/simpleaudit.git
cd simpleaudit
pip install -e .
```

An editable install with the dev extras is convenient if you plan to run the tests:

```bash
pip install -e ".[dev]"
```

## Verify the installation

Run this snippet to confirm the package imports and its scenario packs are available:

```python
import simpleaudit
from simpleaudit import get_scenarios, list_scenario_packs

print("version:", simpleaudit.__version__)

packs = list_scenario_packs()
print("scenario packs:", sorted(packs))

scenarios = get_scenarios("safety")
print(f"safety pack: {len(scenarios)} scenarios")
assert len(scenarios) > 0
assert all("name" in s and "description" in s for s in scenarios)
print("OK")
```

Notes:

- `simpleaudit.__version__` is read from installed package metadata (it falls back to `"0.0.0+unknown"` when the package metadata is not found, e.g. running straight from a checkout without installing).
- `list_scenario_packs()` returns a mapping of pack name to scenario count. Packs include `safety`, `rag`, `health`, `system_prompt`, `helpmed`, `ung`, `nav_aap`, `skatteetaten`, `helfo`, `lanekassen`, `nb_kryss_ordning`, `skatteetaten_legitimasjon`, `toll_reisegodskvote`, `arbeidstilsynet_arbeidstid`, the `human_rights_*` packs, `healthbench_behaviours`, and `all`.
- `get_scenarios(pack_name)` loads a pack's scenarios; passing an unknown pack name raises `ValueError`. See [Scenarios](scenarios.md) for how to use them.

## Configure your environment

When you create a `ModelAuditor`, you pass provider credentials explicitly — there is no hidden env-var lookup inside SimpleAudit itself. Each role (target, judge, auditor) can have its own key and endpoint:

```python
from simpleaudit import ModelAuditor, get_scenarios

auditor = ModelAuditor(
    model="gpt-4o-mini",          # the model being audited
    provider="openai",            # default provider is "openai"
    api_key="sk-...",             # optional: omit to let the SDK resolve it
    base_url=None,                # optional: custom / OpenAI-compatible endpoint
    judge_model="gpt-4o",         # the model that grades conversations
    judge_provider="openai",
    judge_api_key="sk-...",       # defaults to the target's key if omitted
    system_prompt="You are helpful.",
)

results = auditor.run(get_scenarios("safety"))
results.summary()
```

Key parameters of `ModelAuditor.__init__`:

- `model`, `provider` — the target model. `provider` defaults to `"openai"` when left unset.
- `api_key`, `base_url` — optional explicit credentials/endpoint for the target client.
- `judge_model`, `judge_provider`, `judge_api_key`, `judge_base_url` — the judge model; judge credentials fall back to the target's when omitted.
- `auditor_model`, `auditor_provider`, `auditor_api_key`, `auditor_base_url` — the probe-generator model; these fall back to the judge's configuration.
- `max_turns` (default `5`), `max_retries` (default `2`), `retry_backoff` (default `0.5`), `json_format` (default `True`).

For local models, point `provider`/`base_url` at Ollama or vLLM instead of a hosted provider. A full walkthrough of building your first auditor and reading results lives in the [Quickstart](quickstart.md); the different ways to plug in non-model systems (HTTP apps, callables, decision endpoints) are covered in [Targets And Judges](targets-and-judges.md).

## Offline usage (no API key)

You can verify the whole audit pipeline locally without any API keys or network calls by wiring fake clients into a `ModelAuditor`. The repository ships `tests/fakes.py` for exactly this purpose; when working from a source checkout you can import it directly:

```python
import asyncio

from tests.fakes import make_auditor, fixed_target, fixed_severity_judge

auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    max_turns=1,
)

result = asyncio.run(auditor.run_scenario(name="Test", description="A smoke-test scenario."))
print(result.severity)  # pass
```

`make_auditor(target, judge, auditor=None, *, max_turns=1, ...)` builds a real `ModelAuditor` whose three internal clients (target, judge, probe generator) are replaced with fakes — no API keys, no network. Available helpers:

- `fixed_target(text)` — target that always returns the same response.
- `random_length_target(min_chars=200, max_chars=500)` — target returning random filler text.
- `cycling_target(responses)` — target cycling through a fixed list of responses.
- `fixed_severity_judge(severity="critical")` — judge that always assigns one severity (`critical`, `high`, `medium`, `low`, or `pass`).
- `cycling_severity_judge(severities)` — judge cycling through severities, one per judged conversation.
- `fixed_probe_auditor(probe)` / `cycling_probe_auditor(probes)` — probe generators.
- `FakeClient(response_fn)` / `ScriptedClient(responses)` — lower-level drop-in replacements for an AnyLLM client, usable by assigning `auditor.target_client = ...` etc. on an existing instance.

A slightly larger offline run against a scenario dict:

```python
import asyncio

from tests.fakes import (
    make_auditor,
    random_length_target,
    cycling_severity_judge,
    fixed_probe_auditor,
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

If this runs and prints a summary, your environment is ready: Python version, dependencies, and the audit engine all work end-to-end.

## The `simpleaudit` command

Installing the package also registers a `simpleaudit` console script, whose entry point is `main()` in `simpleaudit/cli.py`. Note that the CLI no longer includes the web visualizer — running it just points you to SimpleAudit Studio:

```console
$ simpleaudit
The `simpleaudit` CLI no longer includes the web visualizer.

To visualize audit results, use SimpleAudit Studio:

    uvx simpleaudit-studio --visualize-only --results_dir ./my_audit_results
...
```

So to browse results, use Studio instead:

```bash
uvx simpleaudit-studio --visualize-only --results_dir ./my_audit_results
```

This starts a local web server (no audit worker) that browses a folder of SimpleAudit JSON results and can export a standalone HTML file.

## Troubleshooting

- **`ModuleNotFoundError: No module named 'simpleaudit'`** — the package isn't installed in the active interpreter. Re-run `pip install simpleaudit` (or `pip install -e .` from a checkout) and make sure you're using the same environment.
- **Wrong Python version** — SimpleAudit requires Python 3.11–3.13. Check with `python --version`; use a virtualenv or conda environment if needed.
- **API/auth errors at run time** — credentials are passed to `ModelAuditor` explicitly (`api_key`, `judge_api_key`, `auditor_api_key`); a missing key surfaces when the corresponding client is created. Run the offline check above first to rule out environment problems.
- **Tracing features missing** — the OTLP receiver requires the `tracing` extra: `pip install "simpleaudit[tracing]"`.

## Where to go next

- [Quickstart](quickstart.md) — build your first auditor and run a real audit.
- [Auditing Basics](auditing-basics.md) — understand the auditor/target/judge roles and how a run flows.
- [Targets And Judges](targets-and-judges.md) — connect different kinds of targets and pick a judge.
- [Scenarios](scenarios.md) — explore the built-in scenario packs and write your own.
- [Tracing And Observability](tracing-and-observability.md) — capture and inspect traces with the OTLP extras.

### See Also

*   [1. Run a single-model audit](quickstart.md)
*   [Decision logic](auditing-basics.md)
*   [Document marks](context-analysis.md)
*   [Anatomy of a judge config](custom-judges.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
