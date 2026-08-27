## simpleaudit.experiment

Batch experiments: run audits across multiple models and compare them.

### Classes

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

