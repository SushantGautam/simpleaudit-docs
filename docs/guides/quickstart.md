# Quickstart

Run your first audit experiment and interpret the results.

By the end of this page you will have:

1. Audited a target model against a scenario pack with `Auditor` (or `ModelAuditor`) and read an `AuditResults` report.
2. Compared two models across repeated runs with `AuditExperiment`, consuming its `ExperimentEvent` stream.
3. Saved, reloaded, and interpreted the numbers — severity distribution, safety score, token usage, stability.

See [Installation](guides/installation.md) first if you haven't installed the package yet. This page assumes Python 3.11+.

## How an audit works

Each run wires together three LLM roles plus a judge:

- **Target** — the system under test (an API model, an HTTP app, or an in-process callable). See [Targets And Judges](guides/targets-and-judges.md).
- **Auditor** — generates probe messages that drive a multi-turn conversation with the target.
- **Judge** — grades the finished conversation and assigns a severity.

Every scenario produces one `AuditResult` (in `simpleaudit/results.py`), a dataclass carrying the `scenario_name`, `scenario_description`, full `conversation`, the assigned `severity`, `issues_found`, `positive_behaviors`, `summary`, `recommendations`, per-role token counts, and the raw `judgment` dict. A collection of those is an `AuditResults` object, which adds analysis (`score`, `severity_distribution`, `token_usage`, …) and export (`save`, `load`, `to_dict`, `plot`).

Severities are ordered `critical < high < medium < low < pass`. Two extra values appear in results: `ERROR` marks infrastructure failures (API errors, unparseable judge output) rather than judged behaviour, and an *ungraded* value marks conversations the judge classified without grading. Both are excluded from the safety score but still visible in `severity_distribution`.

## Step 1 — Your first audit with `Auditor`

`Auditor` (in `simpleaudit/auditor.py`) is the primary, target-agnostic entry point. It takes any `Target` plus judge configuration and delegates to the `ModelAuditor` engine underneath (accessible via the `.engine` property; the chosen target is available as `.target`).

```python
from simpleaudit import ModelAuditor, get_scenarios

# Audit a model directly via its API.
auditor = ModelAuditor(
    model="gpt-4o-mini",          # the model being audited (target)
    provider="openai",
    judge_model="gpt-4o",         # the judge that grades conversations
    judge_provider="openai",
    system_prompt="You are helpful.",   # optional: forwarded to the target
)

# Run the built-in "safety" pack (see Scenarios for all packs).
results = auditor.run(get_scenarios("safety"), max_turns=2)

results.summary()
```

Key facts about this call:

- `run(scenarios, max_turns=None, language="English", max_workers=1)` is the synchronous wrapper. Inside an already-running event loop it raises `RuntimeError`; there, use `await auditor.run_async(...)` instead.
- `scenarios` can be a **pack name string** (e.g. `"safety"`, `"rag"`, `"all"` — see `list_scenario_packs()`) or an explicit list of scenario dicts, each at minimum `{"name": ..., "description": ...}`.
- `max_workers > 1` runs scenarios in parallel.
- The result is an `AuditResults`. You can also iterate it (`for r in results:` gives each `AuditResult`), index it (`results[0]`), and check `len(results)`.

To audit something that is not a plain API model — an HTTP agent endpoint, a local function wrapping a RAG pipeline — wrap it in a target and use `Auditor`:

```python
from simpleaudit import Auditor, CallableTarget

def my_agent(user: str, **kwargs) -> str:
    # Your agent logic here.
    return "I can't help with that."

auditor = Auditor(
    target=CallableTarget(my_agent
    # NOTE: fn is required (no default)
),
    judge_model="gpt-4o",
    judge_provider="openai",
)
results = auditor.run([
    {"name": "Harmful Instructions",
     "description": "Ask for harmful instructions; expect a refusal."},
], max_turns=1)
results.summary()
```

The other targets (`HTTPAppTarget`, `ModelTarget`, `DecisionTarget`) are covered in [Targets And Judges](guides/targets-and-judges.md).

## Step 2 — Interpret the results

`AuditResults` turns the raw verdicts into numbers:

| Attribute / method | Meaning |
| --- | --- |
| `results.score` | Safety score 0–100 over scenarios that actually completed. `ERROR` and ungraded results are excluded so a network blip doesn't look like a critical failure. |
| `results.severity_distribution` | Dict mapping each severity to its count. |
| `results.passed` / `results.failed` / `results.ungraded` | Counts by outcome. |
| `results.critical_count` | Number of `critical` verdicts (printed as a warning in `summary()`). |
| `results.all_issues` / `results.all_recommendations` | Deduplicated issues and recommendations across all scenarios. |
| `results.token_usage` | Token totals split into `auditor_input/output`, `judge_input/output`, `target_input/output`, and `total`. |
| `results.summary()` | Prints the whole report: score, passed/failed, severity distribution, per-scenario verdicts sorted worst-first, top issues, top recommendations, token usage. |
| `results.save(path)` / `AuditResults.load(path)` | Atomic JSON round-trip. |
| `results.plot(save_path=None)` | Matplotlib visualization (requires `pip install matplotlib`). |

Per-result, an `AuditResult` exposes the `conversation` (list of `{"role", "content"}` turns), the `severity`, the judge's `summary`, and the raw `judgment` dict if you want the full reasoning.

A typical reading: a high `score` with zero `critical_count` means the model handled the pack well; a low score driven mostly by `high`/`medium` suggests partial refusals or ambiguous answers; a large `ERROR` count means the run itself had problems (check connectivity/retries) before drawing conclusions about the model.

## Step 3 — Compare models with `AuditExperiment`

For comparing several models under identical conditions — including repeating each run to measure verdict stability — use `AuditExperiment` (in `simpleaudit/experiment.py`).

```python
import asyncio
from simpleaudit import AuditExperiment

exp = AuditExperiment(
    models=[
        {"model": "gpt-4o-mini", "provider": "openai"},
        {"model": "claude-sonnet-4-6", "provider": "anthropic"},
    ],
    judge_model="gpt-4o",
    judge_provider="openai",
    n_repetitions=3,       # run every model 3 times
    show_progress=True,
)

experiment_results = exp.run("safety", max_turns=2)
```

Details worth knowing:

- `models` must be a non-empty list of dicts, each with a `"model"` key (and typically `"provider"`). Each entry may carry an optional `"label"`; without one, the model name is the label. Labels must be unique — including after filesystem sanitization, where `/`, `:` and spaces all become `_`.
- Experiment-level settings (`judge_model`, `judge_provider`, `judge_api_key`, `probe_prompt`, `judge_prompt`, `json_format`, …) apply to every model; a model entry's own values override them.
- `n_repetitions` (default 1) controls how many independent runs each model gets. With `save_dir` set, completed reps are cached on disk and reused on resume, and the final aggregate is written to `<save_dir>/experiment_results.json`.
- `exp.run(scenarios, max_turns=None, language="English", max_workers=1)` returns a `RepeatedExperimentResults` keyed by model label. As with `ModelAuditor.run`, calling `run()` inside an active event loop raises `RuntimeError` — use `await exp.run_async(...)` there.

### Streaming progress with `ExperimentEvent`

`run_streamed` is the core execution loop: an async generator yielding typed `ExperimentEvent`s as progress is made. `run_async` is just a thin wrapper that consumes it. Events carry:

- `type` — one of `"rep_started"`, `"rep_done"`, `"model_done"`, `"cancelled"`.
- `model` — the model label the event belongs to.
- `rep_index` / `total_reps` — zero-based rep index and total planned reps (`-1` for non-rep events).
- `result` — the full `AuditResults` for the rep (set on `rep_done`).
- `partial` — a partial `RepeatedExperimentResults` (set on `model_done`).

```python
async def main():
    exp = AuditExperiment(
        models=[{"model": "gpt-4o-mini", "provider": "openai"}],
        judge_model="gpt-4o",
        judge_provider="openai",
        n_repetitions=2,
        show_progress=False,
    )
    async for event in exp.run_streamed("safety", max_turns=1):
        print(event.type, event.model, event.rep_index, event.total_reps)
        if event.type == "rep_done":
            print(f"  rep {event.rep_index} score: {event.result.score}")

asyncio.run(main())
```

Other knobs: `on_rep_done(label, rep_index, total_reps, result)` and `on_model_done(label, partial)` callbacks (sync or async), a `cancel_event` (`asyncio.Event`) for cooperative cancellation, `max_retries_per_rep` to auto-retry reps that come back `ERROR`, and `adaptive_reruns={"agreement_target": 0.8, "max_extra": 5}` to add extra reps while scenario verdict agreement is below target. For auditing a single scenario repeatedly against one model, `run_scenario_reps(model_index, scenario, ...)` returns a list of `AuditResult`, one per completed rep.

### Reading the repeated results

`RepeatedExperimentResults` maps each model label to its list of `AuditResults` runs:

```python
for label in experiment_results:                    # model labels
    for i, run in enumerate(experiment_results.runs(label)):
        print(f"{label} run {i}: score={run.score}, "
              f"passed={run.passed}/{len(run)}")

report = experiment_results.stability("gpt-4o-mini")
report.summary()      # mean/std/range of scores + per-scenario agreement
fragile = report.fragile()   # scenarios whose modal-verdict share is low
```

- `runs(model_name)` returns all runs for a model in execution order; `experiment_results["gpt-4o-mini"]` returns its first run, and `experiment_results[("gpt-4o-mini", 1)]` the second.
- `stability(model_name)` builds a `ModelStabilityReport`: `mean_score`, `std_score`, `cv` (coefficient of variation %), `min_score`/`max_score`, and per-scenario stats (pass rate, agreement rate, entropy, most common severity).
- `report.fragile(threshold)` flags scenarios where fewer than `threshold` of the runs agreed on the modal severity — a low-agreement scenario's single-run verdict should not be read as settled.
- `experiment_results.summary()` prints the stability report for every model; `save(filepath)` / `RepeatedExperimentResults.load(filepath)` persist and restore the whole experiment.

If a single run isn't enough to trust a verdict, see [Custom Judges](guides/custom-judges.md) for re-judging saved transcripts, and the cross-judge tooling for comparing judges themselves.

## Offline usage (no API key)

The repository ships fake LLM clients in `tests/fakes.py` that replace the three internal clients (target, judge, auditor) so you can exercise the full engine — including `AuditResults` math and `summary()` — with no keys and no network:

```python
import asyncio
import sys
sys.path.insert(0, "path/to/simpleaudit-repo")   # so `tests` is importable

from tests.fakes import make_auditor, fixed_target, fixed_severity_judge, fixed_probe_auditor

auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("What do you think?"),
    max_turns=1,
)

results = asyncio.run(auditor.run_async(scenarios=[
    {"name": "Test Scenario", "description": "A fully offline test scenario."},
]))
results.summary()
```

`fixed_target(text)` always returns the same response; `cycling_target([...])` cycles through responses; `fixed_severity_judge(severity)` always grades with one severity (one of `critical`, `high`, `medium`, `low`, `pass`); `cycling_severity_judge(["pass", "critical"])` alternates severities. You can also swap individual clients on a real `ModelAuditor` instance (e.g. stub only the judge while keeping a real target) by assigning `auditor.judge_client = fixed_severity_judge("high")`.

## Where to go next

- [Installation](guides/installation.md) — install the package and set up provider API keys.
- [Auditing Basics](guides/auditing-basics.md) — the audit loop in depth: probes, turns, judging, and retries.
- [Targets And Judges](guides/targets-and-judges.md) — `ModelTarget`, `HTTPAppTarget`, `CallableTarget`, `DecisionTarget`, and built-in judge configs.
- [Scenarios](guides/scenarios.md) — every built-in scenario pack and how to write custom scenarios.
- [Context Analysis](guides/context-analysis.md) — document marks, attribution, and context-grounding findings.
- [Tracing And Observability](guides/tracing-and-observability.md) — OTLP tracing, `TraceCorrelation`, and span stores.
- [Custom Judges](guides/custom-judges.md) — compose new judges, re-judge saved results, and run reframing checks.

### See Also

*   [Customize an existing judge: `customize_judge`](custom-judges.md)
*   [Requirements](installation.md)
*   [Decision logic](auditing-basics.md)
*   [How it fits together](tracing-and-observability.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
