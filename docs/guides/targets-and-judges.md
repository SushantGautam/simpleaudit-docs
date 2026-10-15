# Targets And Judges

SimpleAudit separates *what is being tested* (the **target**) from *how the
test is graded* (the **judge**). This page covers both sides of that split:

- **Targets** — the `Target` protocol and its built-in implementations, with
  `ModelTarget` as the historical default for auditing an LLM endpoint.
- **Judges** — how to pick a built-in judge config, and how to compose new
  ones with `build_judge`, `customize_judge`, `compose_prompt`,
  `dimension_key`, `binary_postprocess` and `average_postprocess`.

## The Target protocol

Everything the auditor sends messages to must satisfy the `Target` protocol
(`simpleaudit/targets/base.py`). It is a single async method:

```python
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
) -> TargetResponse
```

New targets may ignore the model-specific keyword arguments (`response_format`,
`file_uri`, `documents`) when they do not apply. Four concrete targets ship in
`simpleaudit.targets`:

| Target | Use it for |
| --- | --- |
| `ModelTarget` | An LLM endpoint via AnyLLM (the historical default) |
| `HTTPAppTarget` | An external application over HTTP (black-box) |
| `CallableTarget` | An in-process Python callable (handy for tests) |
| `DecisionTarget` | A decision model over a System One endpoint |

### `TargetResponse`

The normalized result of one `send()` call:

- `content: str` — the reply text.
- `raw: Any = None` — the unprocessed response, if any.
- `input_tokens` / `output_tokens` — `Optional[int]`. These are `None` when the
  target does not report usage (e.g. a black-box HTTP app). Callers must treat
  `None` as "unknown", never as zero, so audit records do not fabricate token
  counts.
- `decision: Optional[Dict[str, Any]] = None` — set only by a decision model's
  full answer to a scenario's decision question; `None` for every other target.

### `TargetContext`

Per-turn context passed into `send()`. All fields are optional, so existing
call sites can pass a bare context or `None`:

- `audit_run_id`, `scenario_run_id`, `turn_id` — the audit correlation
  identifiers.
- `trace_headers: Dict[str, str]` — W3C trace-context headers to forward, e.g.
  `{"traceparent": "00-...-...-01"}`.
- `extra: Dict[str, Any]` — free-form metadata a target may use.

When the engine runs a scenario with tracing enabled, it generates a W3C
`traceparent` per turn and puts it on `context.trace_headers`; see
[Tracing And Observability](tracing-and-observability.md) for the rest of
that machinery.

## Configuring a model target: `ModelTarget`

`ModelTarget` (`simpleaudit/targets/model.py`) wraps an AnyLLM client so the
engine can call `target.send(...)` instead of reaching into
`client.acompletion(...)` directly. It has two construction modes:

```python
from simpleaudit import ModelTarget, TargetResponse

# 1. Client mode — pass an AnyLLM client + model name.
target = ModelTarget(client=my_anyllm_client, model="gpt-4o")

# 2. Transport mode — pass an async callable that already knows how to turn
#    (system, user, history, ...) into a TargetResponse.
async def my_transport(**kw):
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

- You must supply **either** `client` **or** `transport`; constructing
  `ModelTarget()` with neither raises `ValueError`.
- In client mode, `send()` builds the OpenAI-style message list and calls
  `client.acompletion` exactly like the legacy `ModelAuditor._call_async` path,
  so behavior is unchanged. If `context.trace_headers` is present, they are
  merged into `params["extra_headers"]` (preserving any user-supplied
  `extra_headers`) so the outgoing call propagates the trace context.
- In transport mode, `send()` simply awaits your transport with the same
  keyword arguments.
- `max_retries` and `retry_backoff` control retries in client mode.
- The `client` and `model` properties expose what was passed in.

### Wiring a target into an auditor

`ModelAuditor` always has a target: the `auditor.target` property lazily wraps
its `target_client` in a `ModelTarget` (using the auditor's `max_retries` and
`retry_backoff`). To replace it with any other `Target`, use
`ModelAuditor.set_target(target)` — this is the extension point for auditing
non-model systems:

```python
from simpleaudit import ModelAuditor, CallableTarget

auditor = ModelAuditor(model="gpt-4o-mini", provider="openai",
                       judge_model="gpt-4o", judge_provider="openai")
auditor.set_target(CallableTarget(lambda **kw: "I cannot help with that.")
    # NOTE: fn is required (no default)
,
    # NOTE: fn is required (no default)

    # NOTE: fn is required (no default)
)
assert isinstance(auditor.target, CallableTarget)
```

For non-model targets there is also the generic, target-agnostic entry point
`Auditor`, which takes any `Target` directly and delegates everything elsfrom simpleaudit import Auditor, CallableTarget

auditor = Auditor(
    target=CallableTarget(lambda **kw: "echo: " + kw["user"]
    # NOTE: fn is required (no default)
),
    judge_model="gpt-4o",
    judge_provider="openai",
    judge_api_key="sk-test-dummy",
)
results = auditor.run([{"name": "Demo", "description": "A demo scenario."}])
results.summary()
 "A demo scenario."}])
results.summary()
```

`Auditor` requires a target (`Auditor(target=None)` raises `ValueError`); its
`target` and `engine` properties give access to the target and the underlying
`ModelAuditor`.

## Choosing a judge

Pass a judge to `ModelAuditor` (or `Auditor` via `**kwargs`) with the `judge=`
argument. It accepts either a **named built-in config** or a **config di# Named built-in
auditor = ModelAuditor(..., judge="helpfulness"
    # NOTE: judge_model is required (no default),
    # NOTE: judge_provider is required (no default),
    # NOTE: model is required (no default),
    # NOTE: provider is required (no default)
)

# Named judge + custom probe prompt (probe_prompt overrides the config's)
auditor = ModelAuditor(..., judge="factuality", probe_prompt="Ask about X..."
    # NOTE: judge_model is required (no default),
    # NOTE: judge_provider is required (no default),
    # NOTE: model is required (no default),
    # NOTE: provider is required (no default)
)

# Fully custom prompt (takes precedence over any named judge)
auditor = ModelAuditor(..., judge_prompt="You are a custom judge..."
    # NOTE: judge_model is required (no default),
    # NOTE: judge_provider is required (no default),
    # NOTE: model is required (no default),
    # NOTE: provider is required (no default)
)
amed judge)
auditor = ModelAuditor(..., judge_prompt="You are a custom judge...")
```

List what is available with `list_judge_configs()` (a dict of name →
one-line description), and resolve a name or dict with `get_judge(name)`. The
built-ins include:

- **`safety`** — Constitutional AI safety evaluation; severity ladder
  (`critical | high | medium | low | pass`).
- **`abstention`** — refusal/abstention appropriateness (AbstentionBench
  taxonomy); score 1–10 with abstained/appropriate flags.
- **`helpfulness`** — response quality across four MT-Bench dimensions; score
  1–10 with relevance, accuracy, clarity, completeness sub-scores.
- **`factuality`** — hallucination and factual error detection; score 1–10
  with verdict and error lists.
- **`harm`** — HELM Safety harm categorisation; harmful flag, category,
  severity across six harm types.
- **`checklist`** — evidence-anchored checklist: met/violated/not_applicable
  per `expected_behavior` item with verbatim quotes; code verifies the quotes
  and derives severity, capped by the scenario's designed severity. Declares
  `postprocess` and `requires_expected_behavior` (scenarios without
  expectations fall back to the default judge).
- **`choice_match`** — code-only exact-match grading of a scenario's decision
  question; calls no judge model at all.
- plus domain judges such as `helsedir_sexhealth_no`,
  `helsedir_sexhealth_no_rag`, `binary_abstention`, `groundedness` and
  `fact_check`.

Score-based judges emit a 1–10 score and no severity; the framework derives
`AuditResult.severity` from the score (9–10 pass, 7–8 low, 5–6 medium, 3–4
high, 1–2 critical) so summaries and plots stay meaningful. When no judge is
named at all, `ModelAuditor` falls back to its default safety judge (see
`DEFAULT_JUDGE` in `simpleaudit/judges/default.py`), whose criteria cover
safety, boundaries, accuracy, transparency and manipulation resistance.

Every judge config declares an `output` kind — `"severity"`, `"score"`,
`"binary"`, `"checklist"` or `"grounding"` — along with `criteria` (what to
evaluate) and `format_prompt` (the output contract); its `judge_prompt` is the
two joined. For deeper authoring techniques — regrading stored results,
reframing checks, panels — see [Custom Judges](custom-judges.md).

## Composing judges

`simpleaudit/judges/compose.py` provides the building blocks for writing new
judges without touching the registry.

### `compose_prompt(criteria, format_prompt)`

The simplest helper: the judge prompt is the criteria, a blank line, then the
format prompt.

```python
from simpleaudit import compose_prompt

prompt = compose_prompt("Rate the assistant's tone.", "Respond with JSON only.")
# 'Rate the assistant's tone.\n\nRespond with JSON only.'
```

This is exactly how `get_judge` assembles `judge_prompt` from a dict that has
`criteria` and `format_prompt` but no `judge_prompt`.

### `dimension_key(name)`

Maps a human-readable dimension name to its JSON key:
`dimension_key("Tone & style")` → `"tone_style"`. Score dimensions are keyed
this way, so names need at least one letter or digit and must be distinct once
keyed.

### `build_judge(output, criteria, ...)`

Builds a judge from scratch in one of the generic formats:

```python
build_judge(
    output,          # "severity" | "score" | "binary" | "checklist"
    criteria,        # what to evaluate, in plain words
    *,
    dimensions=(),   # "score" only: dimension names scored 1-10
    question=None,   # "binary" only: the yes/no question
    pass_when=True,  # "binary" only: the answer that passes (True = yes)
    probe_prompt=None,
    name="Custom judge",
)
```

- **`severity`** — SimpleAudit's standard severity ladder and fields.
- **`score`** — a 1–10 score; with `dimensions`, each named dimension is
  scored and the overall score is their average, computed in code.
- **`binary`** — a yes/no `question` about the assistant's turns; the result
  passes when the answer equals `pass_when`.
- **`checklist`** — met/violated per expected behaviour, with verified quotes.

Both helpers return a config dict accepted wherever a judge name is:

```python
from simpleaudit import build_judge, ModelAuditor

judge = build_judge(
    "score",
    "Rate how well the assistant explains the policy.",
    dimensions=["Accuracy", "Clarity"],
)
auditor = ModelAuditor(
    model="gpt-4o-mini", provider="openai",
    judge_model="gpt-4o", judge_provider="openai",
    judge=judge,
)
```

### `customize_judge(base, ...)`

A copy of a built-in (or composed) judge with new criteria and/or probe
prompt — the base's format, fields and post-processing are kept, so
`customize_judge("harm", criteria=...)` still emits a harm category:

```python
from simpleaudit import customize_judge

judge = customize_judge(
    "helpfulness",
    criteria="Focus only on whether the answer cites its sources.",
    name="Sourcing check",
)
```

Arguments left as `None` keep the base's value. Two guards apply: a base
without a `format_prompt` cannot have its criteria replaced (pass
`judge_prompt` instead), and a base that grades in code (e.g. `choice_match`)
has no criteria to replace.

### Post-processing hooks: `binary_postprocess` and `average_postprocess`

These factories produce the `postprocess(judgment, *, conversation,
expected_behavior, scenario_meta)` callables that turn raw judge JSON into a
graded judgment. They are wired automatically by `build_judge`, but you can
use them directly when hand-rolling a config:

- **`binary_postprocess(pass_when)`** — grades a yes/no `answer`: the result
  gets `"pass"` when the answer equals `pass_when`; otherwise it gets the
  scenario's designed severity (falling back to `"high"` when the scenario has
  none). An answer that is not a yes/no becomes `severity: "ERROR"`. It also
  sets `passed` and defaults `summary` from `reasoning`.
- **`average_postprocess(keys)`** — sets `score` to the average of the given
  dimension scores (rounded to 2 decimals). If the judge gave no numeric
  dimension scores, the result becomes `severity: "ERROR"`. It defaults
  `summary` from `feedback`.

Inside a `score` judge built with `dimensions`, the keys are derived with
`dimension_key` — so `dimensions=["Tone & style"]` produces a JSON field
`"tone_style"` and `average_postprocess(["tone_style"])` averages it.

## Offline usage (no API key)

You can exercise targets and judges without any network or API key.

**Judge composition is pure code** — no model involved:

```pytimport asyncio
from simpleaudit import CallableTarget, ModelTarget, TargetResponse

# A callable target: sync or async, returning str or TargetResponse.
t = CallableTarget(lambda **kw: "echo: " + kw["user"]
    # NOTE: fn is required (no default)
)
print(asyncio.run(t.send(user="hi")).content)  # echo: hi

# A ModelTarget in transport mode: no client, no model, no network.
async def transport(**kw):
    return TargetResponse(content="from-transport", input_tokens=1, output_tokens=2)

mt = ModelTarget(transport=transport)
r = asyncio.run(mt.send(user="x"))
print(r.content, r.input_tokens, r.output_tokens)  # from-transport 1 2
or a
`ModelTarget` transport import asyncio
from simpleaudit import CallableTarget, ModelTarget, TargetResponse

# A callable target: sync or async, returning str or TargetResponse.
t = CallableTarget(lambda **kw: "echo: " + kw["user"]
    # NOTE: fn is required (no default)
)
print(asyncio.run(t.send(user="hi")).content)  # echo: hi

# A ModelTarget in transport mode: no client, no model, no network.
async def transport(**kw):
    return TargetResponse(content="from-transport", input_tokens=1, output_tokens=2)

mt = ModelTarget(transport=transport)
r = asyncio.run(mt.send(user="x"))
print(r.content, r.input_tokens, r.output_tokens)  # from-transport 1 2
s, r.output_tokens)  # from-transport 1 2
```

For a fully offline end-to-end audit (fake target, judge and probe clients),
the test suite ships ready-made fakes in `tests/fakes.py`:

```python
import asyncio
from tests.fakes import make_auditor, fixed_target, fixed_severity_judge, fixed_probe_auditor

auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("Tell me more about this."),
    max_turns=2,
)
result = asyncio.run(auditor.run_scenario(name="Test", description="desc"))
print(result.severity)  # pass
```

See [Quickstart](quickstart.md) for running these fakes against real
scenario packs, and [Auditing Basics](auditing-basics.md) for how the
auditor drives the target turn by turn.

## Where to go next

- [Installation](installation.md) — get SimpleAudit installed and providers configured.
- [Quickstart](quickstart.md) — your first audit end to end.
- [Auditing Basics](auditing-basics.md) — scenarios, turns and results in detail.
- [Scenarios](scenarios.md) — the scenario packs you feed to a target.
- [Context Analysis](context-analysis.md) — grounding and document-mark based judging.
- [Tracing And Observability](tracing-and-observability.md) — how `TargetContext.trace_headers` feeds trace correlation.
- [Custom Judges](custom-judges.md) — reframing, rejudging and judge panels built on the configs from this page.

### See Also

*   [Decision logic](auditing-basics.md)
*   [Anatomy of a judge config](custom-judges.md)
*   [Scenario shape](scenarios.md)
*   [1. Run a single-model audit](quickstart.md)
