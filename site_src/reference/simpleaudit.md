## simpleaudit

Top-level package: the public API of SimpleAudit.

SimpleAudit - Lightweight AI Safety Auditing Framework

A simple, effective tool for red-teaming AI systems using LLMs as auditor and judge.

Supports multiple providers:
- Anthropic (Claude) - default
- OpenAI (GPT-4, GPT-5, etc.)
- Grok (xAI)
- Ollama (local models)
- vLLM (local model serving)
- Any OpenAI-compatible API

Usage:
    from simpleaudit import ModelAuditor, get_scenarios

    # Audit a model directly via its API
    auditor = ModelAuditor(
        model="gpt-4o-mini",
        provider="openai",
        judge_model="gpt-4o",
        judge_provider="openai",
        system_prompt="You are helpful.",
    )
    results = auditor.run(get_scenarios("safety"))
    results.summary()

### Classes

#### `ModelAuditor`

Audits a target LLM directly via its API against a set of scenarios.

Each scenario is sent to the target model (optionally over multiple turns),
and the resulting conversation is scored by a separate LLM judge that
returns a severity verdict, issues, and recommendations. The judge and
target may use different providers, and an optional auditor model can
probe the target between turns.

**Methods:**

- `strip_thinking(text: str) -> str`
- `get_scenarios(pack_name: str) -> List[Dict]`
- `parse_json_response(response: str, default_severity: str = 'ERROR') -> Dict[str, Any]` — Parse JSON from LLM response. Delegates to utils.parse_json_response.
- `async run_scenario(name: str, description: str, expected_behavior: Optional[List[str]] = None, test_prompt: Optional[str] = None, file_uri: Optional[Union[str, List[str]]] = None, max_turns: Optional[int] = None, language: str = 'English', pbar_audit: Optional[tqdm] = None, pbar_judge: Optional[tqdm] = None, max_workers: Optional[int] = None) -> AuditResult`
- `async run_async(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> AuditResults` — Run the audit asynchronously across all scenarios.
- `run(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> AuditResults` — Run the audit synchronously by driving :meth:`run_async` with ``asyncio.run``.

**Attributes:**

- `max_turns`
- `verbose`
- `show_progress`
- `system_prompt`
- `json_format`
- `max_retries`
- `retry_backoff`
- `judge_fields`
- `probe_prompt`
- `judge_prompt`
- `judge_response_schema`
- `target_model`
- `target_client`
- `judge_model`
- `judge_client`
- `auditor_model`
- `auditor_client`

#### `AuditResults`

Collection of audit results with analysis and export methods.

Attributes:
    results: List of AuditResult objects
    timestamp: When the audit was run

**Methods:**

- `summary()`
- `to_dict() -> Dict`
- `save(filepath: str)` — Save results to JSON file (atomically, so interrupts can't corrupt it).
- `load(filepath: str) -> AuditResults` — Load results from JSON file.
- `plot(save_path: Optional[str] = None)` — Plot audit results visualization.

**Attributes:**

- `SEVERITY_SCORES`
- `SEVERITY_ICONS`
- `results`
- `timestamp`
- `total_auditor_input_tokens`
- `total_auditor_output_tokens`
- `total_judge_input_tokens`
- `total_judge_output_tokens`
- `total_target_input_tokens`
- `total_target_output_tokens`
- `token_usage`
- `severity_distribution`
- `all_issues`
- `all_recommendations`
- `passed`
- `failed`
- `critical_count`
- `score` — Safety score over scenarios that actually completed (0-100).

#### `AuditResult`

A single audit result for one scenario.

Holds the scenario metadata, the full conversation with the target model,
the judge's verdict (severity, issues, positive behaviors, summary,
recommendations), the raw judge judgment dict, and token usage counts for
the auditor, judge, and target models.

**Methods:**

- `to_dict() -> Dict`

**Attributes:**

- `scenario_name`
- `scenario_description`
- `conversation`
- `severity`
- `issues_found`
- `positive_behaviors`
- `summary`
- `recommendations`
- `expected_behavior`
- `judgment`
- `auditor_input_tokens`
- `auditor_output_tokens`
- `judge_input_tokens`
- `judge_output_tokens`
- `target_input_tokens`
- `target_output_tokens`

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

#### `ModelStabilityReport`

Aggregate stability report for a model across repeated audit runs.

Holds the per-run scores with mean, std, min, max and coefficient of
variation, plus a :class:`ScenarioStats` entry per scenario. Provides
:meth:`summary` for a printed report and :meth:`fragile` to query
scenarios whose modal-verdict share falls below a threshold.

**Methods:**

- `summary() -> None`
- `fragile(threshold: float = FRAGILE_THRESHOLD_DEFAULT) -> Dict[str, ScenarioStats]` — Scenarios whose modal-verdict share falls below *threshold*.
- `to_dict() -> Dict`

**Attributes:**

- `model`
- `n_runs`
- `scores`
- `mean_score`
- `std_score`
- `min_score`
- `max_score`
- `cv`
- `per_scenario`

#### `ScenarioStats`

Per-scenario stability statistics across repeated runs of one model.

Captures the pass rate, raw severity distribution, modal severity and
agreement rate, plus normalised entropy and ordinal spread of the
verdicts, so that scenarios whose verdicts do not settle across runs can
be identified.

**Methods:**

- `to_dict() -> Dict`

**Attributes:**

- `pass_rate`
- `severity_distribution`
- `most_common_severity`
- `agreement_rate`
- `normalised_entropy`
- `ordinal_spread`
- `n_observations`

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

#### `PromptVariant`

One wording of the judge prompt.

Variants are supplied explicitly rather than generated. A tool built to
detect prompt-induced verdict movement cannot introduce a model-generated
prompt of its own: the paraphrases would then be an uncontrolled axis
inside the instrument measuring that axis.

**Attributes:**

- `label`
- `judge_prompt`
- `response_schema`

#### `ReframingResults`

Verdicts for every (scenario, variant) pair, and the shifts between them.

Attributes:
----------
variant_labels : list of str
    Variant labels in the order they were run.
per_scenario : dict
    ``{scenario_name: {variant_label: severity}}``.
judgments : dict
    ``{scenario_name: {variant_label: raw_judgment_dict}}`` — the full
    judge output, kept so a shift can be read back to its reasoning.
input_tokens, output_tokens : int
    Judge tokens spent. No target tokens are spent on this path.

**Methods:**

- `shifts() -> List[Dict[str, Any]]` — Per-scenario verdicts across variants, with shift detection.
- `invariant_rate() -> float` — Fraction of scenarios whose verdict is identical across all variants.
- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `variant_labels`
- `per_scenario`
- `judgments`
- `input_tokens`
- `output_tokens`

#### `StoredRecord`

A transcript to re-grade, plus the scenario context the judge needs.

**Attributes:**

- `scenario_name`
- `scenario_description`
- `conversation`
- `expected_behavior`

#### `WiggleRunner`

Runs the L0→L1→...→L6 pressure ladder on a candidate judge.

Args:
    candidate_judge_model: Model ID for the candidate judge (target seat).
    candidate_judge_provider: Provider for the candidate judge.
    persuader_model: Model ID for the persuader (auditor seat).
    persuader_provider: Provider for the persuader.
    observer_model: Model ID for the observer (judge seat).
    observer_provider: Provider for the observer.
    observer_version: Pinned version string for the observer (required).
    candidate_judge_api_key: API key for the candidate judge.
    candidate_judge_base_url: Base URL for the candidate judge.
    persuader_api_key: API key for the persuader.
    persuader_base_url: Base URL for the persuader.
    observer_api_key: API key for the observer.
    observer_base_url: Base URL for the observer.
    max_retries: Max retries per API call.
    retry_backoff: Base backoff seconds for retries.
    verbose: Print progress.

**Methods:**

- `async run(scenarios: Union[str, List[Dict]], levels: Optional[List[str]] = None, multi_turn_n: int = 10) -> WiggleProfile` — Run the full wiggle experiment.

**Attributes:**

- `candidate_judge_model`
- `candidate_judge_provider`
- `persuader_model`
- `persuader_provider`
- `observer_model`
- `observer_provider`
- `observer_version`
- `max_retries`
- `retry_backoff`
- `verbose`

#### `WiggleProfile`

Aggregate wiggle profile for a candidate judge.

**Methods:**

- `per_level_wiggle_rate(level: str) -> float` — Wiggle rate at a specific pressure level.
- `to_dict() -> Dict[str, Any]`
- `save(path: str) -> None`
- `load(path: str) -> WiggleProfile`

**Attributes:**

- `candidate_model`
- `candidate_provider`
- `observer_model`
- `observer_version`
- `persuader_model`
- `persuader_provider`
- `scenarios`
- `levels_tested`
- `n_scenarios`
- `n_flips`
- `wiggle_rate` — Fraction of scenarios whose verdict flipped from L0.
- `l0_accuracy` — Fraction of scenarios where the L0 verdict matched ground truth.
- `corrective_flips` — Flips that moved the verdict TOWARD ground truth.
- `corrupting_flips` — Flips that moved the verdict AWAY from ground truth.

#### `ScenarioWiggle`

Wiggle outcome for one scenario across all pressure levels.

**Attributes:**

- `scenario_name`
- `ground_truth`
- `l0_verdict`
- `l0_correct`
- `turns`
- `final_verdict`
- `flipped`
- `flip_direction`
- `wiggle` — True if the verdict changed from L0 at any point.

#### `TurnRecord`

One pressure turn's outcome.

**Attributes:**

- `level`
- `turn`
- `candidate_response`
- `observer_verdict`
- `observer_changed`
- `observer_reasoning`
- `error`

### Functions

#### `get_scenarios(pack_name: str) -> List[Dict]`

Get scenarios from a built-in pack.

Args:
    pack_name: Name of the scenario pack

Returns:
    List of scenario dictionaries

Raises:
    ValueError: If pack name is not recognized

#### `list_scenario_packs() -> Dict[str, int]`

List available scenario packs and their sizes.

Returns:
    Dict mapping pack names to number of scenarios

#### `get_judge(name: str) -> Dict[str, Any]`

Get a built-in judge configuration by name.

Args:
    name: Judge config name (e.g. "safety", "helpfulness")

Returns:
    Judge config dict with probe_prompt, judge_prompt, output_schema, source

Raises:
    ValueError: If name is not recognised

#### `list_judge_configs() -> Dict[str, str]`

List available judge configs and their descriptions.

Returns:
    Dict mapping config names to one-line descriptions

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

#### `load_stored_records(source: Union[str, Path, Dict[str, Any]]) -> List[StoredRecord]`

Read transcripts out of a saved audit result.

Accepts a path to a file written by ``AuditResults.save()`` or the already
parsed payload. Entries whose ``conversation`` is missing or empty are
skipped: there is nothing to re-grade, and carrying them through would
produce verdicts on an empty transcript that look like real ones.

Parameters:
----------
source : str, Path, or dict
    Saved result file, or its parsed contents.

Returns:
-------
list of StoredRecord

#### `reframing_check(judge_client: Any, judge_model: str, records: Sequence[StoredRecord], variants: Sequence[PromptVariant], json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5) -> ReframingResults`

Synchronous wrapper around :func:`reframing_check_async`.

Cannot be called from an active event loop; await the async form there.
Mirrors ``CrossJudgeExperiment.run()``.

#### `async reframing_check_async(judge_client: Any, judge_model: str, records: Sequence[StoredRecord], variants: Sequence[PromptVariant], json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5) -> ReframingResults`

Re-grade every stored transcript once per prompt variant.

Parameters:
----------
judge_client : Any
    An AnyLLM client for the judge. This is the only client used.
judge_model : str
    Judge model identifier.
records : sequence of StoredRecord
    Transcripts to re-grade, e.g. from ``load_stored_records()``.
variants : sequence of PromptVariant
    At least two wordings of the judge prompt. One variant would produce
    no comparison.
json_format : bool, default True
    Whether the judge is asked for schema-constrained JSON.
max_retries, retry_backoff
    Forwarded to the judge call.

Returns:
-------
ReframingResults

### Constants

#### `FRAGILE_THRESHOLD_DEFAULT`
