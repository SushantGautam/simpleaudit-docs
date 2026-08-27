## simpleaudit.repeated_results

Stability analysis: repeated runs, fragility thresholds, and model stability reports.

Repeated-run results for SimpleAudit.

Holds results from running an AuditExperiment multiple times and provides
stability statistics across runs.

Stability is reported at two levels. The aggregate level — mean, std and CV
over whole-run scores — answers whether a model's overall score has settled.
The per-scenario level answers which individual verdicts have settled, and
that is a different question: a scenario swinging between pass and critical
can sit inside a perfectly steady mean, and a single-run "critical" on it
should not be read the same way as one on a scenario that never moves.

The per-scenario statistics here — modal share, normalised entropy and
ordinal spread — are the reproducibility leg of the validity chain pushed
down to the scenario. They cost nothing to compute: the verdicts were
already produced by the runs the experiment paid for. The choice to derive
fragility from existing disagreement rather than from new perturbation runs
follows Zhao et al., *Jagged Judges: Epistemic Stability Under Silence,
Pressure, and Persistence* (2026), https://arxiv.org/abs/2608.12645, which
identifies baseline jury majority strength as the most effective single-shot
signal for anticipating which items wiggle. Modal share is that quantity as
it already exists here, so the disagreement across ``n_repetitions`` is a
signal the runs have paid for whether or not anyone reads it.

The complementary check, which varies the judge prompt rather than resampling
it, lives in :mod:`simpleaudit.reframing`.

### Classes

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

### Constants

#### `SEVERITY_ORDER`
#### `FRAGILE_THRESHOLD_DEFAULT`

_Value: `0.6`_

