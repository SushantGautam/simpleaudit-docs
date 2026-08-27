## simpleaudit.cross_judge

Cross-judge experiments: score the same outputs with different judges and compare.

Cross-judge stability analysis for SimpleAudit.

Provides CrossJudgeExperiment, which orchestrates identical AuditExperiment
runs under multiple judge models to measure how judge version affects severity
ratings and score distributions.

Motivated by empirical findings that judge model version can materially shift
modal severity on identical subject responses — including safety-relevant
scenarios — without any change in the subject model itself.

### Classes

#### `AuditExperiment`

Runs audits across multiple models and compares their results.

Each model is audited with the same scenarios and judge configuration,
optionally repeated ``n_repetitions`` times (with optional adaptive
reruns for low-agreement scenarios). Results can be cached to disk for
resuming interrupted experiments, and per-model completion can be
reported via the ``on_model_done`` callback.

**Methods:**

- `async run_async(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> RepeatedExperimentResults` — Run the experiment asynchronously across all configured models.
- `run(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> RepeatedExperimentResults` — Run the experiment synchronously by driving :meth:`run_async` with ``asyncio.run``.

**Attributes:**

- `models`
- `judge_model`
- `judge_base_url`
- `judge_api_key`
- `judge_provider`
- `auditor_model`
- `auditor_provider`
- `auditor_api_key`
- `auditor_base_url`
- `judge`
- `probe_prompt`
- `judge_prompt`
- `judge_response_schema`
- `json_format`
- `verbose`
- `show_progress`
- `n_repetitions`
- `adaptive_reruns`
- `save_dir`
- `on_model_done`

#### `RepeatedExperimentResults`

Results from running AuditExperiment with n_repetitions > 1.

Provides:
- Backward-compatible dict interface (returns first run's AuditResults)
- .runs(model) — all runs for a model, in execution order
- .stability(model) — mean/std/CV and per-scenario pass rates
- .stability(model).fragile(threshold=...) — scenarios whose verdict
  disagrees across runs
- .summary() — prints stability reports for all models
- .save() / .load() — JSON serialization

**Methods:**

- `runs(model_name: str) -> List[AuditResults]` — Return all runs for the given model, in execution order.
- `keys()`
- `values()` — Return the first run for each model (backward compat).
- `items() -> List[Tuple[str, AuditResults]]` — Return (model, first_run) pairs (backward compat).
- `all_runs() -> Dict[str, List[AuditResults]]` — Return a dict mapping each model to its full list of runs.
- `stability(model_name: str) -> ModelStabilityReport` — Compute stability statistics for a single model across N runs.
- `summary() -> None` — Print stability reports for all models.
- `to_dict() -> Dict`
- `save(filepath: str) -> None` — Save all runs to a JSON file (atomically, so interrupts can't corrupt it).
- `load(filepath: str) -> RepeatedExperimentResults` — Load repeated experiment results from a JSON file.

#### `CrossJudgeResults`

Results from CrossJudgeExperiment — one RepeatedExperimentResults per judge.

Provides cross-judge severity shift detection, score deltas, and CV deltas
per subject model. Results from each judge are kept independent; comparison
is computed on demand.

Attributes:
----------
judges : list of str
    Ordered list of judge labels present in this result set.

**Methods:**

- `score_summary() -> Dict[str, Dict[str, Dict[str, Any]]]` — Per-subject, per-judge score statistics.
- `severity_shifts(subject_label: str) -> List[Dict[str, Any]]` — Per-scenario modal severity across all judges, with shift detection.
- `to_dict() -> Dict[str, Any]` — Serialize all results to a JSON-compatible dict.

**Attributes:**

- `judges` — Ordered list of judge labels.

#### `CrossJudgeExperiment`

Orchestrate AuditExperiment runs across multiple judge models.

Runs one AuditExperiment per judge with save_dir namespaced by judge label
to prevent file collision. Reuses PR #20's resume logic: each judge
maintains its own run cache under ``save_dir/<judge_label>/``.

Parameters:
----------
models : list of dict
    Subject models to evaluate. Same format as ``AuditExperiment.models``
    — each dict must contain a ``model`` key and optionally ``provider``,
    ``label``, and provider-specific credentials.
judge_models : list of dict
    Judge models to compare. Each dict must contain a ``model`` key and
    optionally ``provider``, ``label``, ``api_key``, and ``base_url``.
    If ``label`` is omitted it is derived from the model string.
    Must contain at least two entries.
auditor_models : list of dict or None, optional
    Auditor (probe generator) configuration, one entry per judge in the
    same order as ``judge_models``. Each dict may contain ``model``,
    ``provider``, ``api_key``, and ``base_url``; all four are forwarded to
    the underlying ``AuditExperiment``. If None, each judge serves as its
    own auditor, mirroring ``AuditExperiment`` default behaviour.
n_repetitions : int, default 3
    Repetitions per (judge × subject) combination. Passed through to each
    ``AuditExperiment``.
save_dir : str, Path, or None, optional
    Parent directory for saved results. Each judge's runs land under
    ``save_dir/<judge_label>/<subject_label>/run_N.json``. Pass the same
    ``save_dir`` on a resumed call to continue interrupted runs.
*experiment_kwargs*
    Additional keyword arguments forwarded to every ``AuditExperiment``
    (e.g. ``probe_prompt``, ``judge_prompt``, ``json_format``,
    ``show_progress``).

Examples:
--------
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

- `async run_async(scenarios: Union[str, List[Dict[str, Any]]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> CrossJudgeResults` — Execute all judge variants and return combined results.
- `run(scenarios: Union[str, List[Dict[str, Any]]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> CrossJudgeResults` — Synchronous wrapper around run_async.

**Attributes:**

- `models`
- `judge_models`
- `auditor_models`
- `n_repetitions`
- `save_dir`
- `judge_labels` — Labels of all configured judges, in order.

### Functions

#### `severity_direction(severity_a: Any, severity_b: Any) -> Any`

Position of *severity_b* relative to *severity_a* on ``SEVERITY_ORDER``.

Returns ``idx_b - idx_a``; positive means *severity_b* is the stricter
verdict. Returns None when either value is off the canonical ladder
(e.g. "ERROR"), so callers can tell "no direction" apart from "no shift".

#### `compare_judges(results_a: RepeatedExperimentResults, results_b: RepeatedExperimentResults, subject_label: str, label_a: str = 'judge_a', label_b: str = 'judge_b') -> Dict[str, Any]`

Compare two RepeatedExperimentResults for a given subject model.

Computes per-scenario modal severity shifts, score delta, and CV delta.
Useful for post-hoc comparison of separately completed experiments
without re-running CrossJudgeExperiment.

Args:
    results_a: First judge's repeated results.
    results_b: Second judge's repeated results.
    subject_label: Model label to compare. Must exist in both results.
    label_a: Display name for the first judge (default ``"judge_a"``).
    label_b: Display name for the second judge (default ``"judge_b"``).

Returns:
    dict with keys:

    - ``subject``: subject_label
    - ``judge_a``, ``judge_b``: the label strings
    - ``score_delta``: mean_b − mean_a (positive = judge_b scores higher)
    - ``cv_delta``: cv_b − cv_a (positive = judge_b is more variable)
    - ``stats_a``, ``stats_b``: ``{mean, std, cv, min, max, n_runs}``
    - ``severity_shifts``: list of dicts for scenarios where modal
      severity differs, each with ``scenario``, ``modal_a``, ``modal_b``,
      and ``direction`` (positive = stricter under judge_b).
    - ``n_shifted``: count of shifted scenarios
    - ``n_total``: scenario count for the reference judge (results_a)
    - ``n_compared``: scenarios present in BOTH judges and actually
      compared (``<= n_total``; the correct denominator for a shift rate)

Raises:
    KeyError: If subject_label is not found in either results object.

### Constants

#### `SEVERITY_ORDER`

_Values: `pass, low, medium, high, critical`._

