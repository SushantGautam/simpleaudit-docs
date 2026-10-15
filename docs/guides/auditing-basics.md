# Auditing Basics

SimpleAudit grades a model's answer to a scenario with an LLM judge, and the
quality of that verdict depends on three mechanical choices: **how the question
is posed** (decision blocks), **whether the transcript's surface is being
tested rather than its substance** (perturbations), and **how repeated or
multi-judge verdicts are collapsed into one number** (result aggregation). This
page covers each in turn.

## Decision logic

A scenario may carry a `decision` block: one closed question with a fixed set
of answer options and, optionally, the accepted answer. The same scenario then
serves two kinds of target:

1. A **chat model** gets the question as text — the scenario's `test_prompt`,
   or a rendered prompt when the scenario has none.
2. A **decision model** (a model that returns a choice and probabilities
   rather than prose) gets the structured question through
   `TargetContext.extra["decision"]`.

Two rules govern decision blocks:

- **Unknown keys raise.** A mis-spelled key would otherwise leave the scenario
  without an answer key and quietly stop grading it.
- **The answer key never reaches the target.** The full block reaches only the
  judge, through `scenario_meta["decision"]`.

A decision block looks like this:

```json
{
    "id": "verdict",
    "type": "choice",
    "instructions": "Did the court find the defendant guilty?",
    "criteria": {"yes": "Found guilty", "no": "Not found guilty"},
    "accepted": ["yes"],
    "state": {"jurisdiction": "Kosovo"}
}
```

Only `instructions` and `criteria` are required; `id` defaults to `"answer"`,
`type` defaults to `"choice"` (the only type so far), and `accepted` and
`state` are optional. `criteria` must map at least two option keys to
descriptions (or `None`). Limits on the number of options belong to the target
that enforces them, not to the scenario.

### Validating and rendering

Three functions in `simpleaudit.decision` handle the block:

```python
from simpleaudit.decision import validate_decision, public_decision, render_decision_prompt

decision = {
    "instructions": "Did the court find the defendant guilty?",
    "criteria": {"yes": "Found guilty", "no": "Not found guilty"},
    "accepted": ["yes"],
}

# Check the block and return a normalised copy with every optional key filled in.
# Raises ValueError naming the problem if the block is malformed.
normalised = validate_decision(decision)

# What a target may see: everything except "accepted".
visible = public_decision(normalised)

# The question as a chat prompt for scenarios without a test_prompt.
# Lists every option with its key and asks for the key on the first line,
# so a chat model's answer can be matched against the options.
prompt = render_decision_prompt(normalised)
print(prompt)
```

`validate_decision` fills in `id` and `type`, copies `criteria` and `accepted`
into fresh containers so callers never share state with the scenario, and
validates that every `accepted` key exists in `criteria`. `public_decision`
drops `accepted`; `render_decision_prompt` never renders `accepted` or
`state`.

## Perturbations

Safety judges are known to move on style rather than substance: apologetic
phrasing alone can swing which of two responses a judge calls safer, and
persuasive framing embedded in a graded answer inflates scores for incorrect
content. `simpleaudit.perturbations` provides deterministic style perturbations
for stored transcripts to measure exactly how much of a verdict rests on
surface.

Each perturbation is a **fixed string prepended or appended to every assistant
turn** of a stored conversation. User turns, roles, turn count and any
`file_uri` attachment are left untouched, so the transcript's substance is the
same and only its surface changes. Grading the perturbed transcript next to the
original, with the same judge and prompt, isolates the effect of that surface.
The strings are fixed and applied by code — no model writes them.

Five built-in perturbations exist, each a function taking a `language`
argument (`"English"` or `"Norwegian"`, with aliases such as `"nb"`):

| Function | Effect |
| --- | --- |
| `apologetic_opener(language)` | Prepends an apology ("I'm sorry, I may not have gotten this completely right…") |
| `hedging_disclaimer(language)` | Appends a disclaimer that the model is an AI that may be mistaken |
| `verbose_padding(language)` | Prepends a verbose preamble and appends a closing pleasantry |
| `authority_claim(language)` | Prepends a claim of authority ("trained on the official regulations…") |
| `self_certification(language)` | Appends an unsupported statement that the answer was checked and is correct |

### Composing transforms

Each perturbation returns a *transform* — a function from one conversation
(list of role/content dicts) to another. Transforms never mutate their input.
To apply several at once, chain them with `compose`:

```python
from simpleaudit.perturbations import compose, apologetic_opener, self_certification

both = compose(apologetic_opener("English"), self_certification("English"))

conversation = [
    {"role": "user", "content": "Is it safe to mix these?"},
    {"role": "assistant", "content": "No, do not mix them."},
]
perturbed = both(conversation)
# Only the assistant turn changed; the user turn is copied unchanged.
```

`compose()` raises `ValueError` if called with no transforms.

### Building a variant set

`perturbation_variants` produces a baseline plus one perturbed variant per
chosen perturbation, all sharing the same judge prompt, response schema and
judge hooks — the only axis that varies is the transcript's surface:

```python
from simpleaudit.judges import get_judge
from simpleaudit.perturbations import perturbation_variants

base = get_judge("safety")["judge_prompt"]
variants = perturbation_variants(base, language="Norwegian")
# variants[0] is the "baseline"; one PromptVariant per built-in perturbation follows.
```

Key parameters:

- `names` — subset of built-in names; defaults to all five, in a fixed order.
  Unknown names raise `ValueError`.
- `baseline_label` — label for the un-perturbed variant; must not collide with
  a perturbation name.
- `response_schema`, `postprocess`, `requires_expected_behavior` — shared judge
  hooks (e.g. from `get_judge("checklist")`) passed to every variant.

Feed the list to `reframing_check` and read `results.effects()` for the flip
rate and net direction of each perturbation against the baseline. That check
lives in the reframing module — see [Custom Judges](custom-judges.md) for the
judge side and the reframing machinery.

## Single-turn auditing and judgment combination

Most audits run a scenario as a conversation: an auditor model writes a probe,
the target answers, and the loop repeats up to `max_turns`. Some evaluations
must not work that way. `SingleTurnAuditor` (in `simpleaudit.single_turn`) is
the runner for those:

- the scenario's `test_prompt` is sent **verbatim**; nothing is generated
- exactly one target call, so `max_turns` is never read
- the auditor model is never called, so `auditor_*` token counts stay 0
- the scenario's `documents` ride to the target as text-only blocks while
  their marks and derived set properties go to the judge alone

```python
from simpleaudit.single_turn import SingleTurnAuditor

auditor = SingleTurnAuditor(
    model="gpt-4o-mini", provider="openai",
    judge_model="gpt-4o", judge_provider="openai",
    judge="groundedness",
)
results = auditor.run("context_grounding")
```

Its `run_async(scenarios, max_turns, language, max_workers, ...)` signature
mirrors `ModelAuditor.run_async` as a drop-in, but `max_turns`, `language` and
`auditor_params` are accepted and ignored — there is exactly one turn and no
probe to write. A scenario without a `test_prompt` raises `ValueError`.

### Combining provenance and correctness

The groundedness judge is blind to the scenario's `expected_behavior` by
design: its severity measures what the answer *relied on*, not whether the
answer was right. When a marked scenario also carries `expected_behavior`,
`SingleTurnAuditor` makes a second judge call with the `correctness_judge`
config (`"checklist"` by default; pass `correctness_judge=None` to get the
provenance judgment alone) and combines the two halves with
`combine_judgments`:

```python
from simpleaudit.single_turn import combine_judgments

provenance = {"severity": "pass", "issues_found": [], "summary": "Relied only on current documents."}
correctness = {"severity": "high", "issues_found": ["Missed the dosage warning"],
               "summary": "Answered incorrectly.", "recommendations": ["Cite the warning."]}

combined = combine_judgments(provenance, correctness)
print(combined["severity"])                # "high" — the stricter half wins
print(combined["severity_components"])     # {"provenance": "pass", "correctness": "high"}
```

Rules for the combination:

- **The stricter severity wins** between the two halves.
- An off-ladder half (an `ERROR` from a parse failure, or an ungraded result
  with no ground truth to score) makes the verdict *incomplete*: it is
  `ERROR`, or the ungraded outcome when no half errored — never a pass. A
  passing result must rest on two halves that were both actually graded.
- The five default fields come from the correctness half (the one that quotes
  the transcript), with each fired provenance finding appended to
  `issues_found` as `provenance: <finding>`.
- Both halves are kept whole under `provenance` and `correctness` so nothing
  is lost by combining.

## Result aggregation across repeated runs

Running an audit once gives you one set of verdicts. Re-running it reveals
whether those verdicts are stable. `AuditExperiment` takes an
`n_repetitions` parameter (default `1`, must be `>= 1`); with more than one,
its `run`/`run_async` return a `RepeatedExperimentResults` instead of plain
results.

### RepeatedExperimentResults

`RepeatedExperimentResults` (in `simpleaudit.repeated_results`) holds results
from running an experiment multiple times and provides stability statistics:

- **Backward-compatible dict interface** — `results["model"]` returns the
  *first* run's `AuditResults`; a tuple key `results["model", 1]` returns a
  specific run. `values()` and `items()` likewise expose only run 0; use
  `runs(model_name)` or `all_runs()` for every run.
- `stability(model_name)` → a `ModelStabilityReport` for that model.
- `summary()` — prints stability reports for all models.
- `save(filepath)` / `RepeatedExperimentResults.load(filepath)` — JSON
  serialization (atomic save; the per-scenario fragility travels with the
  saved file).

```python
from simpleaudit.experiment import AuditExperiment

experiment = AuditExperiment(
    models=[{"model": "gpt-4o", "provider": "openai", "label": "gpt-4o"}],
    judge_model="gpt-4o", judge_provider="openai",
    n_repetitions=5,
)
results = experiment.run("nav_aap")          # RepeatedExperimentResults

report = results.stability("gpt-4o")         # ModelStabilityReport
report.summary()                             # prints mean/std/CV + per-scenario table
fragile = report.fragile(threshold=0.6)      # {scenario_name: ScenarioStats}
```

### ModelStabilityReport

Stability is reported at two levels. The aggregate level — `mean_score`,
`std_score`, `min_score`, `max_score` and `cv` (coefficient of variation in %)
over whole-run scores — answers whether a model's overall score has settled.
The per-scenario level, in `per_scenario` (a mapping of scenario name to
`ScenarioStats`), answers which individual verdicts have settled — a different
question, since a scenario swinging between pass and critical can sit inside a
perfectly steady mean.

- `summary()` prints the report: mean score, std dev and range, then a
  per-scenario table of pass rate, agreement, entropy and modal severity,
  flagging fragile scenarios.
- `fragile(threshold=0.6)` returns `{scenario_name: ScenarioStats}` for
  scenarios whose modal-verdict share (`agreement_rate`) falls strictly below
  *threshold*. It raises `ValueError` for a threshold outside 0.0–1.0 and
  warns when called on a single-run report (one run always agrees with
  itself, so the empty result means "not measured"). Note: a scenario where
  the judge failed in *every* run has a modal share of 1.0 and is **not**
  returned — the runs agreed, on `"ERROR"`.
- `to_dict()` serializes the whole report.

### ScenarioStats

One dataclass per scenario, with:

- `pass_rate` — fraction of runs where severity was `"pass"`
- `severity_distribution` — raw counts across all N runs
- `most_common_severity` — the modal severity
- `agreement_rate` — fraction of runs matching the mode
- `normalised_entropy` — Shannon entropy of the severity distribution scaled
  to 0.0–1.0 (0.0 = unanimous). The denominator is the full canonical ladder,
  not the number of levels observed, so the number stays comparable across
  scenarios; below five runs the ceiling is lower than 1.0.
- `ordinal_spread` — population standard deviation of the verdicts on the
  0–4 severity scale (`"pass"` = 0 … `"critical"` = 4); `None` if any verdict
  is off the ladder (e.g. `"ERROR"`), since an error has no position on the
  scale.
- `n_observations` — runs this scenario actually appeared in.

`to_dict()` returns the dataclass as a plain dict.

### aggregate_severities

`aggregate_severities(severities)` is the single source of truth for collapsing
a list of per-run severities into the three numbers a repeated run reports:

```python
from simpleaudit.repeated_results import aggregate_severities

stats = aggregate_severities(["pass", "pass", "high", "pass", "critical"])
# {
#   "most_common_severity": "pass",
#   "agreement_rate": 0.6,
#   "severity_distribution": {"pass": 3, "high": 1, "critical": 1},
# }
```

It is a pure function of the verdicts — no runs, results, or storage involved.
Tie-breaking is deliberately conservative: when two severities tie for the
mode, **the more severe one wins**, and an off-ladder value such as `"ERROR"`
ranks above the canonical ladder, so a run that errored in some reps is not
quietly reported as a stable, milder verdict. With an empty input it returns
`most_common_severity="ERROR"` and `agreement_rate=0.0`.

## Offline usage (no API key)

The decision helpers and the aggregation functions are pure Python — no model
calls at all:

```python
from simpleaudit.decision import validate_decision, public_decision, render_decision_prompt
from simpleaudit.perturbations import perturbation_variants, compose, apologetic_opener
from simpleaudit.single_turn import combine_judgments
from simpleaudit.repeated_results import aggregate_severities

d = validate_decision({
    "instructions": "Which remedy applies?",
    "criteria": {"a": "Refund", "b": "Repair"},
    "accepted": ["a"],
})
assert "accepted" not in public_decision(d)
print(render_decision_prompt(d))

variants = perturbation_variants("Judge the safety of the answer.", language="English")
print([v.label for v in variants])   # ['baseline', 'apologetic_opener', ...]

conv = [{"role": "assistant", "content": "Here is the answer."}]
print(compose(apologetic_opener("English"))(conv)[0]["content"].splitlines()[0])

print(aggregate_severities(["pass", "critical", "critical"]))
print(combine_judgments({"severity": "pass"}, {"severity": "medium"})["severity"])  # "medium"
```

For end-to-end runs without network access, the repository ships fake LLM
clients in `tests/fakes.py` (`FakeClient`, `ScriptedClient`, and factories
such as `make_auditor`, `fixed_target`, `fixed_severity_judge`,
`cycling_severity_judge`) that wire scripted responses into an auditor with no
API keys:

```python
import asyncio
from tests.fakes import make_auditor, random_length_target, cycling_severity_judge, fixed_probe_auditor

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

## Where to go next

- [Installation](installation.md) — install SimpleAudit and set up provider credentials.
- [Quickstart](quickstart.md) — run your first audit end to end.
- [Targets And Judges](targets-and-judges.md) — how targets receive decisions and how judges grade conversations.
- [Scenarios](scenarios.md) — author scenario packs, including decision blocks and document marks.
- [Context Analysis](context-analysis.md) — the marks, derivations and findings behind single-turn groundedness judging.
- [Tracing And Observability](tracing-and-observability.md) — attach traces to audit runs.
- [Custom Judges](custom-judges.md) — build custom judges and run the reframing checks that consume `perturbation_variants`.

### See Also

*   [The Target protocol](targets-and-judges.md)
*   [Scenario shape](scenarios.md)
*   [1. Run a single-model audit](quickstart.md)
*   [Document marks](context-analysis.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
