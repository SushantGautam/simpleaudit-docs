## simpleaudit.reframing

Re-grade stored transcripts with the judge only, varying one thing at a time.

### Classes

#### `PromptVariant`

One grading condition.

The original use is one wording of the judge prompt. A variant may also
name its own judge (`judge_model` / `judge_client`, falling back to the
call-level values) and carry a `transform` applied to the transcript
before grading. Whatever it varies, the transcript's substance is held
fixed by the caller.

Variants are supplied explicitly rather than generated. A tool built to
detect apparatus-induced verdict movement cannot introduce a
model-generated prompt, judge or perturbation of its own: that would be an
uncontrolled axis inside the instrument measuring that axis.

**Methods:**

- `from_judge(name: Union[str, Dict[str, Any]], label: Optional[str] = None, **overrides: Any) -> 'PromptVariant'` — A variant carrying a judge's prompt, schema and hooks.

**Attributes:**

- `label`
- `judge_prompt`
- `response_schema`
- `judge_model`
- `judge_client`
- `transform`
- `postprocess`
- `requires_expected_behavior`

#### `StoredRecord`

A transcript to re-grade, plus the scenario context the judge needs.

`scenario_severity` is the scenario's designed severity (the ceiling a
checklist post-processor derives against); `scenario_severity_source`
says where it came from: `"stored"` (a checklist judgment in the saved
file), `"lookup"` (a map passed to `load_stored_records`) or None.

**Methods:**

- `scenario_meta() -> Dict[str, Any]`

**Attributes:**

- `scenario_name`
- `scenario_description`
- `conversation`
- `expected_behavior`
- `scenario_severity`
- `scenario_severity_source`

#### `SampleStats`

Verdicts of one (scenario, variant) cell graded `k` times.

`normalised_entropy` and `ordinal_spread` are the same statistics
`ScenarioStats` reports across `n_repetitions` runs, computed here over
judge-only samples of a fixed transcript. `expected_index` is the mean
ladder position, the continuous reading of a discrete verdict.

**Methods:**

- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `severities`
- `modal`
- `agreement_rate`
- `normalised_entropy`
- `ordinal_spread`
- `expected_index`
- `n`

#### `VariantEffect`

How one variant's modal verdicts differ from the baseline's.

`direction` per scenario is `severity_direction(baseline, variant)`:
positive means the variant graded stricter. `mean_direction` averages
it over scenarios where both verdicts are on the ladder, zeros included,
so a variant that moves two scenarios up and two down reads as `0.0`
with `flip_rate` still showing that it moved.

**Methods:**

- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `label`
- `baseline`
- `n_compared`
- `n_shifted`
- `flip_rate`
- `n_stricter`
- `n_lenient`
- `n_undirected`
- `mean_direction`
- `net`
- `per_scenario`

#### `PanelVerdict`

Several judges' modal verdicts on one scenario, aggregated.

`majority` resolves ties to the stricter on-ladder verdict, the
conservative reading for a safety audit; `majority_share` still shows
that it was a tie. Off-ladder verdicts ("ERROR") count in the denominator
and in `n_distinct` but never win the majority unless nothing else is
there.

**Methods:**

- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `verdicts`
- `majority`
- `majority_share`
- `mean_index`
- `mean_index_severity`
- `worst`
- `n_distinct`
- `normalised_entropy`
- `ordinal_spread`
- `unanimous`

#### `PanelResults`

Per-scenario panel verdicts over a set of judge variants.

**Methods:**

- `unanimous_rate() -> float` — Fraction of scenarios on which every judge agreed; 0.0 when empty.
- `flagged(threshold: float = FRAGILE_THRESHOLD_DEFAULT) -> List[str]` — Scenarios whose majority share is below *threshold*: a review queue.
- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `labels`
- `per_scenario`

#### `ReframingResults`

Verdicts for every (scenario, variant) pair, and the movement between them.

**Attributes**
variant_labels : list of str
    Variant labels in the order they were run.
per_scenario : dict
    `{scenario_name: {variant_label: modal_severity}}`. At `k=1` the
    modal is the one verdict.
judgments : dict
    `{scenario_name: {variant_label: raw_judgment_dict}}` — the judge
    output behind the modal verdict (the first sample that produced it),
    kept so a shift can be read back to its reasoning.
input_tokens, output_tokens : int
    Judge tokens spent. No target tokens are spent on this path.
k : int
    Samples per (scenario, variant) cell.
baseline_label : str or None
    Variant that `effects()` compares against; the first variant if unset.
samples : dict
    `{scenario_name: {variant_label: [severity, ...]}}` — every sample.
sample_judgments : dict
    `{scenario_name: {variant_label: [raw_judgment_dict, ...]}}`.
variant_meta : dict
    `{variant_label: {judge_model, judge_client_id, prompt_sha1,
    schema_sha1, transform}}` — what each variant actually varied, so a
    result file is self-describing and `panel()` can check its inputs.
tokens_by_variant : dict
    `{variant_label: {"input": n, "output": n}}`.

**Methods:**

- `shifts() -> List[Dict[str, Any]]` — Per-scenario modal verdicts across variants, with shift detection.
- `invariant_rate() -> float` — Fraction of scenarios whose modal verdict is identical across all variants.
- `stability() -> Dict[str, Dict[str, SampleStats]]` — Per-cell sample statistics: `{scenario: {variant_label: SampleStats}}`.
- `fragile(threshold: float = FRAGILE_THRESHOLD_DEFAULT) -> Dict[str, Dict[str, SampleStats]]` — Cells whose modal share is below *threshold* (strict), keyed like
- `effects(baseline: Optional[str] = None) -> Dict[str, VariantEffect]` — Each non-baseline variant's movement against the baseline, on modals.
- `panel(labels: Optional[Sequence[str]] = None, strict: bool = True) -> PanelResults` — Aggregate judge-only variants into one verdict per scenario.
- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `variant_labels`
- `per_scenario`
- `judgments`
- `input_tokens`
- `output_tokens`
- `k`
- `baseline_label`
- `samples`
- `sample_judgments`
- `variant_meta`
- `tokens_by_variant`

### Functions

#### `load_stored_records(source: Union[str, Path, Dict[str, Any]], scenario_severities: Optional[Mapping[str, str]] = None) -> List[StoredRecord]`

Read transcripts out of a saved audit result.

Accepts a path to a file written by `AuditResults.save()` or the already
parsed payload. Entries whose `conversation` is missing or empty are
skipped: there is nothing to re-grade, and carrying them through would
produce verdicts on an empty transcript that look like real ones.

**Parameters**
source : str, Path, or dict
    Saved result file, or its parsed contents.
scenario_severities : mapping, optional
    `{scenario name: designed severity}`, e.g. from
    `simpleaudit.checklist.severity_by_name(get_scenarios(pack))`. Used
    for records whose stored judgment does not already record the
    designed severity. Saved results predating the checklist judge never
    do, so pass the pack the run came from when re-grading with it.

**Returns**
list of StoredRecord

#### `make_judge_client(provider: str, api_key: Optional[str] = None, base_url: Optional[str] = None) -> Any`

Build a judge client the way `ModelAuditor` does, without an auditor.

Same provider default and `base_url` -> `api_base` translation as a
live audit, so a judge-only re-grade talks to the same endpoint the audit
would have.

#### `reframing_check_asyncasync (judge_client: Any, judge_model: str, records: Sequence[StoredRecord], variants: Sequence[PromptVariant], json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5, k: int = 1, max_concurrency: int = 1, baseline: Optional[str] = None) -> ReframingResults`

Re-grade every stored transcript `k` times per variant.

**Parameters**
judge_client : Any
    An AnyLLM client for the judge. Variants may override it.
judge_model : str
    Judge model identifier. Variants may override it.
records : sequence of StoredRecord
    Transcripts to re-grade, e.g. from `load_stored_records()`.
variants : sequence of PromptVariant
    At least two grading conditions, or exactly one when `k > 1`
    (resampling one judge is itself a comparison).
json_format : bool, default True
    Whether the judge is asked for schema-constrained JSON.
max_retries, retry_backoff
    Forwarded to the judge call.
k : int, default 1
    Samples per (scenario, variant) cell. The judge call does not set
    `temperature`, so this measures the provider's default sampling.
max_concurrency : int, default 1
    Judge calls in flight at once. Results are assigned by position, so
    the value never changes what is reported, only how fast.
baseline : str, optional
    Variant label that `effects()` compares against. Defaults to the
    first variant.

**Returns**
ReframingResults

#### `reframing_check(judge_client: Any, judge_model: str, records: Sequence[StoredRecord], variants: Sequence[PromptVariant], json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5, k: int = 1, max_concurrency: int = 1, baseline: Optional[str] = None) -> ReframingResults`

Synchronous wrapper around `reframing_check_async`.

Cannot be called from an active event loop; await the async form there.
Mirrors `CrossJudgeExperiment.run()`.

#### `rejudge_asyncasync (results_or_path: Union[AuditResults, str, Path], judge_client: Any, judge_model: str, judge_prompt: Optional[str] = None, response_schema: Optional[Dict[str, Any]] = None, json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5, max_concurrency: int = 1, judge: Optional[Union[str, Dict[str, Any]]] = None, postprocess: Optional[Callable[..., Dict[str, Any]]] = None, scenario_severities: Optional[Mapping[str, str]] = None) -> AuditResults`

Grade a saved run again under another judge, keeping every transcript.

Returns a new `AuditResults` whose judge fields (severity, issues,
positive behaviours, summary, recommendations, raw judgment, judge
tokens) come from the new judge and whose conversations, expectations,
auditor tokens and target tokens are copied unchanged. The output has the
same scenario set as the input, so it drops straight into
`compare_judges` or `RepeatedExperimentResults`.

`judge` names a registry config (`"checklist"`, `"safety"`, ...) and
supplies its prompt, schema and hooks wherever the explicit arguments are
None, with the same precedence as `ModelAuditor(judge=..., judge_prompt=
...)`. With neither `judge` nor `judge_prompt` the built-in default
judge is used. Scenario-level `judge_notes` are not stored on results
and cannot be reapplied here.

`scenario_severities` (`{scenario name: designed severity}`) supplies
the ceiling a checklist post-processor derives against, for saved runs
that do not already record it in their judgments. Without it the ceiling
defaults to medium and a warning says so.

Entries with an empty conversation are kept as `ERROR` results (with one
warning) rather than dropped, so scenario sets stay aligned.

#### `rejudge(results_or_path: Union[AuditResults, str, Path], judge_client: Any, judge_model: str, judge_prompt: Optional[str] = None, response_schema: Optional[Dict[str, Any]] = None, json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5, max_concurrency: int = 1, judge: Optional[Union[str, Dict[str, Any]]] = None, postprocess: Optional[Callable[..., Dict[str, Any]]] = None, scenario_severities: Optional[Mapping[str, str]] = None) -> AuditResults`

Synchronous wrapper around `rejudge_async`.

Cannot be called from an active event loop; await the async form there.

### Constants

- `Transform`

