## simpleaudit.repeated_results

Repeated-run results for SimpleAudit.

### Classes

#### `ScenarioStats`

_No docstring._

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

_No docstring._

**Methods:**

- `summary() -> None`
- `fragile(threshold: float = FRAGILE_THRESHOLD_DEFAULT) -> Dict[str, 'ScenarioStats']` — Scenarios whose modal-verdict share falls below *threshold*.
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
- `load(filepath: str) -> 'RepeatedExperimentResults'` — Load repeated experiment results from a JSON file.

### Functions

#### `aggregate_severities(severities: List[str]) -> Dict[str, Any]`

Aggregate a scenario's per-run severity verdicts into summary stats.

This is the single source of truth for collapsing a list of per-run
severities (one per repetition) into the three numbers a repeated run
reports: the modal severity, the share of runs that agreed with it, and
the raw severity distribution. It is a pure function of the verdicts —
no runs, results, or storage involved — so both the in-process stability
report and out-of-process callers (e.g. a web studio that runs reps
itself) can share the exact same aggregation instead of each hand-rolling
a modal computation.

Tie-breaking is deliberately *conservative*: when two severities are tied
for the mode, the more severe one wins. A run that swings between "high"
and "critical" should surface "critical", not whichever happened to be
seen first — and a verdict that is "ERROR" (the judge failed) is treated
as the most severe of all, so a run that errored in some reps is not
quietly reported as a stable, milder verdict. This differs from a plain
`Counter.most_common(1)`, whose tie-break is insertion order and is
therefore arbitrary.

**Args**
    severities: One severity string per run, in execution order. May
        contain off-ladder values such as "ERROR" or a custom judge
        vocabulary; those are counted in the distribution and ranked
        above the canonical ladder for tie-breaking.

**Returns**
    A dict with:
        - `most_common_severity` (str): the modal severity under the
          conservative tie-break above. "ERROR" when *severities* is empty.
        - `agreement_rate` (float): fraction of runs matching the mode;
          0.0 when *severities* is empty.
        - `severity_distribution` (dict): `{severity: count}` over all
          runs, in first-seen order.

### Constants

- `FRAGILE_THRESHOLD_DEFAULT`

