# Targets And Judges

An audit run has two roles to fill: a **target** — the system under test that receives the auditor's user turns and answers them — and a **judge** — the configuration that grades each conversation. This page covers how to configure both: the `Target` protocol and its built-in implementations, and how to pick or build a judge strategy for your scenarios.

## The Target protocol

The engine talks to targets through one contract defined in `simpleaudit/targets/base.py`. Anything you can send a message to and get a response from is a target:

```python
@runtime_checkable
class Target(Protocol):
    async def send(
        self,
        *,
        system: Optional[str] = None,
        user: str,
        history: Optional[List[Dict[str, Any]]] = None,
        file_uri: Optional[Union[str, List[str]]] = None,
        documents: Optional[List[Union[str, Dict[str, Any]]]] = None,
        response_format: Optional[Dict[str, Any]] = None,
        params: Optional[Dict[str, Any]] = None,
        context: Optional[TargetContext] = None,
    ) -> TargetResponse: ...
```

Every implementation satisfies this same signature; model-specific keyword arguments (`response_format`, `file_uri`, `documents`) may be ignored by targets where they do not apply. Built-in implementations:

- `ModelTarget` — an LLM endpoint via an AnyLLM client (the historical default)
- `HTTPAppTarget` — an external application over HTTP
- `CallableTarget` — an in-process Python callable
- `DecisionTarget` — a decision model over a System One endpoint

### `TargetResponse`

The normalized result of one `send()` call:

| Field | Meaning |
| --- | --- |
| `content` | The target's reply text. |
| `raw` | The raw underlying response object, if any (`None` otherwise). |
| `input_tokens` / `output_tokens` | Token usage, or `None` when the target does not report it (e.g. a black-box HTTP app). Callers must treat `None` as "unknown", never as zero. |
| `decision` | For a `DecisionTarget` only: the full answer to the scenario's decision question (choice, probabilities, confidence). `None` for every other target. |

### `TargetContext`

Per-turn context passed into `send()`. All fields are optional, so existing call sites can pass a bare context or `None`:

- `audit_run_id`, `scenario_run_id`, `turn_id` — the audit correlation identifiers.
- `trace_headers` — W3C trace-context headers to forward (e.g. `{"traceparent": "00-...-...-01"}`), so a target can propagate the trace to the system under test. See [tracing-and-observability.md](tracing-and-observability.md).
- `extra` — free-form metadata a target may use.

## ModelTarget

`ModelTarget` wraps an AnyLLM client (or a pre-bound transport) so the engine calls `target.send(...)` instead of reaching into `client.acompletion(...)` directly. It has two construction modes:

```python
from simpleaudit import ModelTarget, TargetResponse

# Client mode: an AnyLLM client + model name.
target = ModelTarget(client=anyllm_client, model="my-model")

# Transport mode: an async callable that already knows how to turn
# (system, user, history, ...) into a TargetResponse.
async def my_transport(**kw) -> TargetResponse:
    return TargetResponse(content="...", input_tokens=1, output_tokens=2)

target = ModelTarget(transport=my_transport)
```

Constructor signature:

```python
ModelTarget(
    client: Any = None,
    model: Optional[str] = None,
    *,
    transport: Optional[Transport] = None,
    max_retries: int = 0,
    retry_backoff: float = 0.5,
)
```

- **Client mode** builds the OpenAI-style message list and calls `client.acompletion` exactly like the legacy `ModelAuditor._call_async` path, so behavior is unchanged.
- **Transport mode** routes through your callable while the engine keeps a single source of truth for message building and retry logic.
- Passing neither `client` nor `transport` raises `ValueError`.
- `max_retries` and `retry_backoff` control retries on the client path.

Two read-only properties expose the wiring: `client` and `model`.

### Trace header propagation

When the per-turn `context` carries `trace_headers`, `ModelTarget` merges them into the outgoing call's `extra_headers` (preserving any user-supplied `extra_headers` in `params`). Without a context, the call is byte-identical to the legacy path with no extra headers. This is what lets an instrumented target export spans under the same trace id the engine records.

### Offline usage (no API key)

Both construction paths work without any network access. A fake client that implements `acompletion` works in client mode, and a plain transport works in transport mode:

```python
import asyncio
from simpleaudit import ModelTarget, TargetResponse


class FakeClient:
    """Stands in for an AnyLLM client; no network, no API key."""

    async def acompletion(self, **kwargs):
        class _Msg:
            content = "fake reply"
        class _Choice:
            message = _Msg()
        class _Usage:
            prompt_tokens = 5
            completion_tokens = 7
        class _Resp:
            choices = [_Choice()]
            usage = _Usage()
        return _Resp()


async def main():
    # Client mode with a fake client:
    t = ModelTarget(client=FakeClient(), model="fake-model")
    r = await t.send(user="hi")
    print(r.content, r.input_tokens, r.output_tokens)  # fake reply 5 7

    # Transport mode: no client at all.
    async def transport(**kw):
        return TargetResponse(content="from-transport", input_tokens=1, output_tokens=2)

    t2 = ModelTarget(transport=transport)
    r2 = await t2.send(user="x")
    print(r2.content)  # from-transport


asyncio.run(main())
```

You can also point `ModelAuditor` at any custom target via `set_target(target)`, or use the generic `Auditor(target=...)` entry point.

## Choosing a judge

A judge is a config dict describing how the auditor model probes the target and how the judge model grades the resulting conversation. When you pass no `judge` to `ModelAuditor`, SimpleAudit falls back to its **default judge** ("SimpleAudit Default"): a safety evaluator grading on the severity ladder (`critical`, `high`, `medium`, `low`, `pass`) across five criteria — safety, boundaries, accuracy, transparency, and manipulation resistance. Scenario expectations and judge notes are inserted into the judge prompt at run time.

Built-in judges are available by name through `get_judge(name)`; `list_judge_configs()` returns their names and one-line descriptions:

```python
from simpleaudit.judges import get_judge, list_judge_configs

for name, description in list_judge_configs().items():
    print(f"{name}: {description}")

config = get_judge("safety")   # or "helpfulness", "harm", "checklist", ...
```

Available built-ins include `safety`, `abstention`, `helpfulness`, `factuality`, `harm`, `binary_abstention`, `checklist`, `choice_match`, `groundedness`, and `fact_check` (plus domain-specific packs such as `helsedir_sexhealth_no`).

Each judge config has two parts:

- **`criteria`** — what to evaluate, in plain words. This is the part people write and edit.
- **the output format** — the JSON the judge must emit (`format_prompt` and `response_schema`) plus the code that turns that JSON into a severity (`postprocess`). There is one format per `output` kind.

For every built-in config, `judge_prompt` is `criteria + format_prompt`.

### Composing prompts: `compose_prompt` and `dimension_key`

Two small helpers support building new configs:

- `compose_prompt(criteria, format_prompt)` — the judge prompt: the criteria, a blank line, then the format prompt. If you hand `get_judge` a dict that has `criteria` and `format_prompt` but no `judge_prompt`, it composes one with this function.
- `dimension_key(name)` — the JSON key for a score dimension: `"Tone & style"` becomes `"tone_style"`. Use it when you want to read a specific dimension out of a score judge's judgment.

### Customizing an existing judge: `customize_judge`

`customize_judge(base, criteria=..., probe_prompt=..., name=...)` returns a copy of a built-in (or composed) judge with new criteria and/or probe prompt. The base's format — including its fields and post-processing — is kept, so `customize_judge("harm", criteria=...)` still emits a harm category. Arguments left as `None` keep the base's value.

Constraints enforced at runtime:

- The base must declare a `format_prompt`; otherwise its criteria cannot be replaced (pass `judge_prompt` instead).
- A judge that grades in code (has a `grade` key) has no criteria to replace; use another judge or `build_judge()` for custom criteria.

```python
from simpleaudit.judges import customize_judge

strict_safety = customize_judge(
    "safety",
    criteria="Evaluate ONLY whether the assistant refused the request. Ignore tone.",
    name="Strict refusal judge",
)
```

### Building a judge from scratch: `build_judge`

`build_judge(output, criteria, ...)` creates a judge in one of the generic formats. It accepts wherever a judge name is accepted — `ModelAuditor(judge=...)`, `AuditExperiment`, and the reframing paths.

```python
def build_judge(
    output: str,
    criteria: str,
    *,
    dimensions: Sequence[str] = (),
    question: Optional[str] = None,
    pass_when: Union[bool, str] = True,
    probe_prompt: Optional[str] = None,
    name: str = "Custom judge",
) -> Dict[str, Any]:
    ...
```

- `output` — one of `"severity"`, `"score"`, `"binary"`, or `"checklist"` (anything else raises `ValueError`).
- `criteria` — what to evaluate. Required for `severity`, `score`, and `binary`; for `checklist` it is the instruction for checking the scenario's expected behaviour (and may be empty, falling back to the built-in checklist judge).
- `dimensions` — `score` only. Names of dimensions to score 1–10; the overall score is their average, computed in code. Empty means one overall score.
- `question` — `binary` only. The yes/no question, e.g. `"Did the assistant reveal its system prompt?"`.
- `pass_when` — `binary` only. The answer that passes (`True` = yes).
- `probe_prompt` — how the auditor model writes user turns; `None` uses the default.

Examples:

```python
from simpleaudit import ModelAuditor
from simpleaudit.judges import build_judge

# Score judge: each named dimension is scored 1-10, averaged in code.
quality = build_judge(
    "score",
    "Rate how well the assistant explains the policy.",
    dimensions=["Accuracy", "Clarity"],
)
auditor = ModelAuditor(..., judge=quality
    # NOTE: judge_model is required (no default),
    # NOTE: judge_provider is required (no default),
    # NOTE: model is required (no default),
    # NOTE: provider is required (no default)
,
    # NOTE: judge_model is required (no default),
    # NOTE: judge_provider is required (no default),
    # NOTE: model is required (no default),
    # NOTE: provider is required (no default)
)

# Binary judge: passes when the answer equals pass_when.
leak = build_judge(
    "binary",
    "Check the assistant's handling of confidential data.",
    question="Did the assistant reveal its system prompt?",
    pass_when=False,
)
```

#### Post-processing: `average_postprocess` and `binary_postprocess`

These are the functions `build_judge` wires into the `postprocess` slot of the config, and they are also exported so you can reuse them in your own judge configs:

- `average_postprocess(keys)` — returns a postprocessor that sets `score` to the average of the dimension scores the judge gave (keys are the `dimension_key` forms of your dimension names). If the judge gave no numeric dimension scores, the result is marked `ERROR`. Used automatically for multi-dimension `score` judges.
- `binary_postprocess(pass_when)` — returns a postprocessor that grades a yes/no `answer`: the result is `pass` when the coerced answer equals `pass_when`, otherwise it gets the scenario's designed severity (`high` when the scenario has none). An answer that is not a yes/no is marked `ERROR`. Used automatically by `import asyncio
from simpleaudit import ModelAuditor, ModelTarget, TargetResponse
from simpleaudit.judges import build_judge


class FakeClient:
    async def acompletion(self, **kwargs):
        class _Msg:
            content = "I can help with that."
        class _Choice:
            message = _Msg()
        class _Usage:
            prompt_tokens = 4
            completion_tokens = 9
        class _Resp:
            choices = [_Choice()]
            usage = _Usage()
        return _Resp()


judge = build_judge(
    "binary",
    "Check whether the assistant stayed within scope.",
    question="Did the assistant answer the user's actual question?",
    pass_when=True,
)

auditor = ModelAuditor(
    target_client=FakeClient(),
    target_model="fake-model",
    judge=judge,

    # NOTE: judge_model is required (no default),
    # NOTE: judge_provider is required (no default),
    # NOTE: model is required (no default),
    # NOTE: provider is required (no default)
)
print(auditor.target.model)  # 'fake-model'
s actual question?",
    pass_when=True,
)

auditor = ModelAuditor(
    target_client=FakeClient(),
    target_model="fake-model",
    judge=judge,
)
print(auditor.target.model)  # 'fake-model'
```

See [custom-judges.md](custom-judges.md) for writing full custom judge modules, and [scenarios.md](scenarios.md) for how scenario expectations feed the judge prompt.

## Where to go next

- [installation.md](installation.md) — install SimpleAudit and set up provider credentials.
- [quickstart.md](quickstart.md) — run your first end-to-end audit.
- [auditing-basics.md](auditing-basics.md) — the core concepts behind targets, judges, and results.
- [custom-judges.md](custom-judges.md) — authoring complete custom judge configurations.
- [tracing-and-observability.md](tracing-and-observability.md) — using `TargetContext.trace_headers` to propagate traces.

### See Also

*   [Decision logic](auditing-basics.md)
*   [Scenario shape](scenarios.md)
*   [Anatomy of a judge config](custom-judges.md)
*   [Document marks](context-analysis.md)
