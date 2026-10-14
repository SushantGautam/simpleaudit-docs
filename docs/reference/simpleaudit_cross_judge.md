## simpleaudit.cross_judge

Cross-judge stability analysis for SimpleAudit.

### Classes

#### `CrossJudgeResults`

Results from CrossJudgeExperiment — one RepeatedExperimentResults per judge.

Provides cross-judge severity shift detection, score deltas, and CV deltas
per subject model. Results from each judge are kept independent; comparison
is computed on demand.

**Attributes**
judges : list of str
    Ordered list of judge labels present in this result set.

**Methods:**

- `judges() -> List[str]` — Ordered list of judge labels.
- `score_summary() -> Dict[str, Dict[str, Dict[str, Any]]]` — Per-subject, per-judge score statistics.
- `severity_shifts(subject_label: str) -> List[Dict[str, Any]]` — Per-scenario modal severity across all judges, with shift detection.
- `to_dict() -> Dict[str, Any]` — Serialize all results to a JSON-compatible dict.

#### `CrossJudgeExperiment`

Orchestrate AuditExperiment runs across multiple judge models.

Runs one AuditExperiment per judge with save_dir namespaced by judge label
to prevent file collision. Reuses PR #20's resume logic: each judge
maintains its own run cache under `save_dir/<judge_label>/`.

**Parameters**
models : list of dict
    Subject models to evaluate. Same format as `AuditExperiment.models`
    — each dict must contain a `model` key and optionally `provider`,
    `label`, and provider-specific credentials.
judge_models : list of dict
    Judge models to compare. Each dict must contain a `model` key and
    optionally `provider`, `label`, `api_key`, and `base_url`.
    If `label` is omitted it is derived from the model string.
    Must contain at least two entries.
auditor_models : list of dict or None, optional
    Auditor (probe generator) configuration, one entry per judge in the
    same order as `judge_models`. Each dict may contain `model`,
    `provider`, `api_key`, and `base_url`; all four are forwarded to
    the underlying `AuditExperiment`. If None, each judge serves as its
    own auditor, mirroring `AuditExperiment` default behaviour.

    Note that this default makes the judge comparison non-isolating: each
    judge generates the transcripts it grades, so its auditor differs too.
    Pass explicit `auditor_models` to hold the auditor fixed, or use
    `reframing_check` to hold the whole transcript fixed.
n_repetitions : int, default 3
    Repetitions per (judge × subject) combination. Passed through to each
    `AuditExperiment`.
save_dir : str, Path, or None, optional
    Parent directory for saved results. Each judge's runs land under
    `save_dir/<judge_label>/<subject_label>/run_N.json`. Pass the same
    `save_dir` on a resumed call to continue interrupted runs.
`**experiment_kwargs`
    Additional keyword arguments forwarded to every `AuditExperiment`
    (e.g. `probe_prompt`, `judge_prompt`, `json_format`,
    `show_progress`).

**Examples**
>>> exp = CrossJudgeExperiment(
...     models=[{"model": "claude-haiku-4-5-20251001", "provider": "anthropic",
...              "label": "haiku-4.5"}],
...     judge_models=[
...         {"model": "claude-opus-4-7", "provider": "anthropic"},
...         {"model": "claude-opus-4-8", "provider": "anthropic"},
...     ],
...     n_repetitions=3,
...     save_dir="/path/to/results",
... )
>>> results = await exp.run_async(scenarios="nav_aap", language="Norwegian")
>>> print(results.score_summary())

**Methods:**

- `judge_labels() -> List[str]` — Labels of all configured judges, in order.
- `async run_async(scenarios: Union[str, List[Dict[str, Any]]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> CrossJudgeResults` — Execute all judge variants and return combined results.
- `run(scenarios: Union[str, List[Dict[str, Any]]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> CrossJudgeResults` — Synchronous wrapper around run_async.

### Functions

#### `compare_judges(results_a: RepeatedExperimentResults, results_b: RepeatedExperimentResults, subject_label: str, label_a: str = 'judge_a', label_b: str = 'judge_b') -> Dict[str, Any]`

Compare two RepeatedExperimentResults for a given subject model.

Computes per-scenario modal severity shifts, score delta, and CV delta.
Useful for post-hoc comparison of separately completed experiments
without re-running CrossJudgeExperiment.

The two results objects are not required to share transcripts, and this
function does not check whether they do. When they come from separate
CrossJudgeExperiment runs under the default auditor configuration, the
deltas below include auditor and sampling effects alongside the judge
effect.

**Args**
    results_a: First judge's repeated results.
    results_b: Second judge's repeated results.
    subject_label: Model label to compare. Must exist in both results.
    label_a: Display name for the first judge (default `"judge_a"`).
    label_b: Display name for the second judge (default `"judge_b"`).

**Returns**
    dict with keys:

    - `subject`: subject_label
    - `judge_a`, `judge_b`: the label strings
    - `score_delta`: mean_b − mean_a (positive = judge_b scores higher)
    - `cv_delta`: cv_b − cv_a (positive = judge_b is more variable)
    - `stats_a`, `stats_b`: `{mean, std, cv, min, max, n_runs}`
    - `severity_shifts`: list of dicts for scenarios where modal
      severity differs, each with `scenario`, `modal_a`, `modal_b`,
      and `direction` (positive = stricter under judge_b).
    - `n_shifted`: count of shifted scenarios
    - `n_total`: scenario count for the reference judge (results_a)
    - `n_compared`: scenarios present in BOTH judges and actually
      compared (`<= n_total`; the correct denominator for a shift rate)

**Raises**
    KeyError: If subject_label is not found in either results object.

