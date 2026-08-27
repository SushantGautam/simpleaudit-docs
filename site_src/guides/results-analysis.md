## Results & Analysis

The `simpleaudit.results` module handles the storage, analysis, and visualization of audit outcomes. It provides data structures for individual scenario results and aggregate collections, enabling developers to track severity distributions, token usage, and safety scores. The `simpleaudit.repeated_results` module extends this functionality to analyze stability across multiple runs, identifying fragile scenarios where judge verdicts vary. Finally, `simpleaudit.cross_judge` facilitates comparative analysis across different judge models to detect bias or inconsistency in evaluation criteria.

### Core Data Structures

#### `AuditResult`

The `AuditResult` dataclass represents the outcome of a single scenario audit. It captures the conversation history, severity rating, and detailed feedback from the judge.

**Attributes:**
*   `scenario_name` (str): Name of the tested scenario.
*   `scenario_description` (str): Description of the scenario.
*   `conversation` (List[Dict]): The conversation history.
*   `severity` (str): One of `critical`, `high`, `medium`, `low`, `pass`, or `ERROR`.
*   `issues_found` (List[str]): Specific deviations identified.
*   `positive_behaviors` (List[str]): Aspects the model handled well.
*   `summary` (str): Brief professional summary.
*   `recommendations` (List[str]): Concrete improvement suggestions.
*   `expected_behavior` (Optional[List[str]]): Expected behaviors for the scenario.
*   `judgment` (Optional[Dict]): Raw judgment data from the judge.
*   `token_usage` fields: Input/output token counts for auditor, judge, and target models.

#### `AuditResults`

The `AuditResults` class aggregates a list of `AuditResult` objects. It provides computed properties for high-level metrics and methods for serialization and visualization.

**Key Properties:**
*   `score` (float): Safety score (0–100). Calculated by averaging severity scores, excluding `ERROR` results to prevent infrastructure failures from skewing safety metrics.
*   `severity_distribution` (Dict[str, int]): Count of results per severity level.
*   `token_usage` (Dict[str, int]): Aggregated token consumption across all roles.
*   `all_issues` (List[str]): Deduplicated list of all issues found, preserving order.
*   `all_recommendations` (List[str]): Deduplicated list of all recommendations, preserving order.
*   `passed` (int): Count of results with severity `pass`.
*   `failed` (int): Count of results not `pass`.
*   `critical_count` (int): Count of results with severity `critical`.

**Methods:**
*   `summary()`: Prints a formatted console report including score, distribution, top issues, and token usage.
*   `save(filepath: str)`: Atomically writes results to a JSON file. Uses a temporary file and rename to prevent corruption if the process is interrupted.
*   `load(filepath: str)`: Class method to load `AuditResults` from a JSON file.
*   `plot(save_path: Optional[str])`: Generates a matplotlib visualization with a severity pie chart and a per-scenario bar chart. Requires `matplotlib`.

**Example: Saving and Loading Results**

```python
from simpleaudit.results import AuditResults

# Assume 'results' is an AuditResults instance
results.save("audit_output.json")

# Later, load and analyze
loaded_results = AuditResults.load("audit_output.json")
print(f"Score: {loaded_results.score}")
loaded_results.summary()
```

### Stability Analysis

When running audits multiple times, `simpleaudit.repeated_results` provides tools to assess consistency.

#### `ScenarioStats`

Dataclass containing statistics for a single scenario across multiple runs.

**Attributes:**
*   `pass_rate` (float): Fraction of runs where severity was `pass`.
*   `severity_distribution` (Dict[str, int]): Raw counts of severities across all runs.
*   `most_common_severity` (str): The modal severity verdict.
*   `agreement_rate` (float): Fraction of runs matching the most common severity (modal share).
*   `normalised_entropy` (float): Shannon entropy of severity distribution, scaled 0.0–1.0. 0.0 indicates unanimous agreement.
*   `ordinal_spread` (Optional[float]): Population standard deviation of severity indices (0–4). Returns `None` if any verdict is `ERROR` or off-ladder.
*   `n_observations` (int): Number of runs this scenario appeared in.

#### `ModelStabilityReport`

Aggregates stability metrics for a specific model across all scenarios.

**Methods:**
*   `fragile(threshold: float = 0.6)`: Returns a dictionary of scenarios where the modal verdict share is below the threshold. These scenarios are considered "fragile" because their verdicts are unstable.
*   `summary()`: Prints a detailed stability report, highlighting fragile scenarios.

**Example: Identifying Fragile Scenarios**

```python
from simpleaudit.repeated_results import ModelStabilityReport

# Assume 'report' is a ModelStabilityReport instance
fragile_scenarios = report.fragile(threshold=0.6)
for name, stats in fragile_scenarios.items():
    print(f"Fragile: {name} (Agreement: {stats.agreement_rate:.2f})")
```

### Cross-Judge Analysis

`simpleaudit.cross_judge` allows comparing how different judge models evaluate the same subject model. This is critical for detecting judge bias or version-specific inconsistencies.

#### `CrossJudgeExperiment`

Orchestrates `AuditExperiment` runs for multiple judge models.

**Parameters:**
*   `models`: List of subject model configurations.
*   `judge_models`: List of judge model configurations (must contain at least two).
*   `n_repetitions`: Number of runs per judge-subject pair.
*   `save_dir`: Directory to store results, namespaced by judge label.

#### `CrossJudgeResults`

Stores results from a `CrossJudgeExperiment`.

**Methods:**
*   `score_summary()`: Returns per-subject, per-judge score statistics (mean, std, CV, min, max).
*   `severity_shifts(subject_label: str)`: Identifies scenarios where the modal severity differs between judges. Returns a list of dicts containing the scenario name, modal severity per judge, and a `shifted` boolean.

**Example: Detecting Judge Disagreement**

```python
from simpleaudit.cross_judge import CrossJudgeExperiment

exp = CrossJudgeExperiment(
    models=[{"model": "subject-model", "provider": "anthropic"}],
    judge_models=[
        {"model": "judge-a", "provider": "anthropic", "label": "judge_a"},
        {"model": "judge-b", "provider": "anthropic", "label": "judge_b"}
    ],
    n_repetitions=3
)

results = await exp.run_async(scenarios="safety")

# Check for shifts
for shift in results.severity_shifts("subject-model"):
    if shift["shifted"]:
        print(f"Shift detected in {shift['scenario']}: {shift['modals']}")
```

### Best Practices

1.  **Atomic Saves**: Always use `AuditResults.save()` rather than manual JSON writing to ensure file integrity during interruptions.
2.  **Exclude Errors from Scores**: The `score` property automatically excludes `ERROR` results. Do not manually include them in safety calculations, as they represent infrastructure failures, not model behavior.
3.  **Stability Thresholds**: Use `ModelStabilityReport.fragile()` to identify scenarios requiring further investigation. A low `agreement_rate` suggests the judge is inconsistent or the scenario is ambiguous.
4.  **Cross-Judge Validation**: Run `CrossJudgeExperiment` with at least two distinct judge models to ensure that safety findings are robust against judge-specific biases.