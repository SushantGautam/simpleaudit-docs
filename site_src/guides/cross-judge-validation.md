## Cross-Judge Validation

The `cross_judge` module in SimpleAudit provides tools for evaluating the stability of audit results across different judge models. In LLM-based auditing, the judge model (the evaluator) can significantly influence severity ratings and score distributions, even when the subject model (the target being audited) remains constant. This module allows developers to run identical audit experiments under multiple judge configurations to measure consistency, detect severity shifts, and validate that results are robust to judge version changes.

### Core Concepts

The primary components are `CrossJudgeExperiment` and `CrossJudgeResults`.

*   **`CrossJudgeExperiment`**: Orchestrates the execution of `AuditExperiment` instances for a list of subject models against a list of judge models. It ensures that each judge's results are saved in a namespaced directory to prevent collisions and supports resuming interrupted runs.
*   **`CrossJudgeResults`**: A container for the results of a `CrossJudgeExperiment`. It provides methods to compare scores and severity ratings across judges, identifying discrepancies and shifts in modal severity.

### CrossJudgeExperiment

The `CrossJudgeExperiment` class manages the setup and execution of multi-judge audits.

#### Parameters

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `models` | `List[Dict[str, Any]]` | List of subject model configurations. Each dict must contain a `model` key. |
| `judge_models` | `List[Dict[str, Any]]` | List of judge model configurations. Must contain at least two entries. Each dict must contain a `model` key. |
| `auditor_models` | `Optional[List[Dict[str, Any]]]` | Optional list of auditor (probe generator) configurations. If provided, must match the length of `judge_models`. If `None`, the judge model acts as its own auditor. |
| `n_repetitions` | `int` | Number of repetitions per (judge × subject) combination. Default is `3`. |
| `save_dir` | `Optional[Union[str, Path]]` | Parent directory for saving results. Results for each judge are stored under `save_dir/<judge_label>/`. |
| `**experiment_kwargs` | `Any` | Additional keyword arguments passed to each underlying `AuditExperiment` (e.g., `probe_prompt`, `judge_prompt`). |

#### Example Usage

```python
from simpleaudit.cross_judge import CrossJudgeExperiment

# Define subject models to audit
subject_models = [
    {
        "model": "claude-haiku-4-5-20251001",
        "provider": "anthropic",
        "label": "haiku-4.5"
    }
]

# Define judge models to compare
judge_models = [
    {"model": "claude-opus-4-7", "provider": "anthropic", "label": "opus-4.7"},
    {"model": "claude-opus-4-8", "provider": "anthropic", "label": "opus-4.8"}
]

# Initialize the experiment
experiment = CrossJudgeExperiment(
    models=subject_models,
    judge_models=judge_models,
    n_repetitions=3,
    save_dir="./cross_judge_results"
)

# Run the experiment asynchronously
import asyncio
results = asyncio.run(experiment.run_async(scenarios="nav_aap", language="Norwegian"))
```

### CrossJudgeResults

The `CrossJudgeResults` object is returned after running a `CrossJudgeExperiment`. It stores `RepeatedExperimentResults` for each judge and provides analysis methods.

#### Attributes

*   `judges`: A list of judge labels present in the results.

#### Methods

##### `score_summary()`

Returns a nested dictionary containing score statistics for each subject and judge.

**Returns:**
A dictionary structured as:
```json
{
  "subject_label": {
    "judge_label": {
      "n_runs": 3,
      "mean": 0.85,
      "std": 0.02,
      "cv": 0.023,
      "min": 0.82,
      "max": 0.88
    }
  }
}
```

##### `severity_shifts(subject_label)`

Compares the modal severity rating for each scenario across all judges for a specific subject model. This is useful for detecting if a judge update causes a shift in how safety-critical scenarios are rated.

**Parameters:**
*   `subject_label` (`str`): The label of the subject model to analyze.

**Returns:**
A list of dictionaries, one per scenario:
```json
[
  {
    "scenario": "harmful_content",
    "modals": {
      "opus-4.7": "high",
      "opus-4.8": "critical"
    },
    "shifted": true,
    "direction": 1
  }
]
```
*   `modals`: Maps judge labels to their modal severity rating.
*   `shifted`: `True` if judges disagree on the modal severity.
*   `direction`: Only present if exactly two judges are compared and a shift occurred. Positive values indicate the second judge is stricter.

##### `to_dict()`

Serializes the entire results object into a JSON-compatible dictionary, including raw run data for each judge.

### Analyzing Results

Developers should use `severity_shifts` to identify scenarios where judge updates impact safety ratings. A `shifted` value of `True` indicates that the modal severity changed between judges, which may require manual review or prompt adjustment.

```python
# Get results from the previous example
results = asyncio.run(experiment.run_async(scenarios="nav_aap", language="Norwegian"))

# Check for severity shifts in the 'haiku-4.5' subject
shifts = results.severity_shifts("haiku-4.5")

for shift in shifts:
    if shift["shifted"]:
        print(f"Shift detected in scenario: {shift['scenario']}")
        print(f"  Modals: {shift['modals']}")
        if "direction" in shift:
            print(f"  Direction: {shift['direction']}")

# Print score summary
summary = results.score_summary()
for subject, judges in summary.items():
    print(f"\nSubject: {subject}")
    for judge, stats in judges.items():
        print(f"  Judge {judge}: Mean={stats['mean']}, Std={stats['std']}")
```

### Best Practices

1.  **Use Distinct Labels**: Ensure each judge model has a unique `label` to avoid directory collisions and confusion in results.
2.  **Monitor Severity Shifts**: Regularly check `severity_shifts` when updating judge models to ensure safety ratings remain consistent.
3.  **Save Results**: Always specify `save_dir` to allow resuming interrupted experiments and to retain raw data for further analysis.
4.  **Compare CVs**: Use the `cv` (coefficient of variation) in `score_summary` to assess the consistency of scores within a judge. High CVs may indicate unstable judging.

### Troubleshooting

*   **Duplicate Judge Labels**: If you receive a `ValueError` about duplicate labels, ensure each judge model in `judge_models` has a unique `label` key.
*   **Missing Auditor Models**: If `auditor_models` is provided, it must have the same length as `judge_models`. If omitted, the judge model is used as the auditor.
*   **Resume Runs**: To resume an interrupted experiment, instantiate a new `CrossJudgeExperiment` with the same `save_dir` and parameters. The module will skip already completed runs.

### See Also

*   [Judge Validation](judge-validation.md)
*   [Judges](judges.md)
*   [Results & Analysis](results-analysis.md)
*   [Visualization](visualization.md)
