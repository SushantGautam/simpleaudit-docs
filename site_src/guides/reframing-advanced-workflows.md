## Reframing & Advanced Workflows

The `simpleaudit` library provides mechanisms to validate the robustness of audit verdicts. A core concern in model auditing is distinguishing between genuine model behavior and artifacts of the evaluation apparatus. Two primary subsystems address this: **Reframing** (testing judge prompt sensitivity) and **Advanced Experiments** (managing complex, multi-model, multi-run audit configurations).

### 1. Reframing: Detecting Prompt-Induced Verdict Shifts

The `simpleaudit.reframing` module allows you to re-grade stored transcripts using different wordings of the judge prompt. If a verdict changes when the judge prompt is paraphrased (while the target model's output remains identical), the instability is likely an artifact of the judge's sensitivity to phrasing, not a flaw in the target model.

#### Key Classes

**`PromptVariant`**
Represents a single wording of the judge prompt.
*   `label` (str): Unique identifier for the variant (e.g., "baseline", "reordered").
*   `judge_prompt` (str): The full text of the judge prompt.
*   `response_schema` (Optional[Dict]): Optional JSON schema for structured output.

**`StoredRecord`**
A container for a specific audit scenario's transcript.
*   `scenario_name` (str): Name of the scenario.
*   `scenario_description` (str): Contextual description for the judge.
*   `conversation` (List[Dict]): The chat history.
*   `expected_behavior` (Optional[List[str]]): Ground truth behaviors.

**`ReframingResults`**
Aggregates verdicts across all variants.
*   `shifts()`: Returns a list of dicts identifying scenarios where verdicts differed between variants.
*   `invariant_rate()`: Returns the fraction of scenarios where the verdict was identical across all variants.

#### Functions

**`load_stored_records(source)`**
Loads transcripts from a saved `AuditResults` JSON file or a parsed dictionary.
*   `source`: Path to JSON file or the payload dict.
*   Returns: `List[StoredRecord]`.

**`reframing_check(judge_client, judge_model, records, variants, ...)`**
Executes the reframing analysis.
*   `judge_client`: An `AnyLLM` client instance.
*   `judge_model`: Model ID for the judge.
*   `records`: Sequence of `StoredRecord`.
*   `variants`: Sequence of `PromptVariant` (must contain at least 2).
*   Returns: `ReframingResults`.

#### Example: Running a Reframing Check

```python
from any_llm import AnyLLM
from simpleaudit.judges import get_judge
from simpleaudit.reframing import (
    load_stored_records, reframing_check, PromptVariant
)

# 1. Load existing transcripts
records = load_stored_records("path/to/saved_audit.json")

# 2. Define prompt variants
base_prompt = get_judge("safety")["judge_prompt"]
reordered_prompt = "..." # A semantically equivalent but reordered version

variants = [
    PromptVariant(label="baseline", judge_prompt=base_prompt),
    PromptVariant(label="reordered", judge_prompt=reordered_prompt)
]

# 3. Execute check
client = AnyLLM.create("anthropic")
results = reframing_check(
    judge_client=client,
    judge_model="claude-opus-4-7",
    records=records,
    variants=variants
)

# 4. Analyze shifts
for shift in results.shifts():
    if shift["shifted"]:
        print(f"Unstable: {shift['scenario']}")
        print(f"  Verdicts: {shift['modals']}")
        print(f"  Direction: {shift.get('direction', 'N/A')}")

print(f"Invariant Rate: {results.invariant_rate():.2%}")
```

### 2. Advanced Experiments: `AuditExperiment`

The `AuditExperiment` class in `simpleaudit.experiment` manages complex audit runs involving multiple models, repetitions, and adaptive reruns. It handles configuration merging, caching, and progress tracking.

#### Configuration & Caching

`AuditExperiment` supports **configuration fingerprinting** to prevent cache corruption. When resuming an experiment, it compares the current configuration (judge model, prompt, scenarios, etc.) against a stored `config.json` fingerprint. If they differ, cached runs are invalidated to ensure data integrity.

**Key Parameters:**
*   `models`: List of dicts, each containing a `model` key. Optional `label` key for disambiguation.
*   `n_repetitions`: Number of times to run each scenario per model.
*   `adaptive_reruns`: Optional dict for adaptive sampling.
    *   `agreement_target`: Threshold (0-1) for agreement required to stop rerunning.
    *   `max_extra`: Maximum additional runs allowed.
*   `save_dir`: Directory for caching results.

#### Label Sanitization

Model labels are sanitized for filesystem safety. Characters `/`, `:`, and ` ` are replaced with `_`. If two distinct labels sanitize to the same string (e.g., `org/model` and `org:model`), the experiment raises a `ValueError` to prevent cache collisions.

#### Example: Multi-Model Experiment with Caching

```python
from simpleaudit.experiment import AuditExperiment

experiment = AuditExperiment(
    models=[
        {"model": "claude-3-5-sonnet", "label": "Claude 3.5"},
        {"model": "gpt-4o", "label": "GPT-4o"}
    ],
    judge_model="claude-3-opus",
    n_repetitions=3,
    save_dir="./audit_cache",
    show_progress=True
)

# Run experiment
# Scenarios can be a path to a JSON file or a list of dicts
results = experiment.run_async(
    scenarios="path/to/scenarios.json",
    max_turns=10
)

# Results are saved automatically to save_dir
# Resume logic is handled internally via config fingerprinting
```

### 3. Best Practices

1.  **Validate Judge Robustness**: Always run a reframing check on a subset of scenarios before trusting audit results. A low `invariant_rate` suggests the judge is overly sensitive to prompt phrasing.
2.  **Unique Labels**: When using `AuditExperiment`, ensure model labels are unique even after sanitization. Use explicit `label` keys if model IDs contain special characters.
3.  **Cache Management**: The `save_dir` contains `config.json` and `run_N.json` files. If you change the judge prompt or scenario pack, the fingerprint mismatch will automatically trigger a re-run, ensuring you don't mix incompatible results.
4.  **Async Usage**: `reframing_check` and `AuditExperiment.run_async` are async-aware. If calling from an active event loop, use the `_async` variants directly to avoid `RuntimeError`.

### Troubleshooting

*   **`ValueError: variants must contain at least two entries`**: Reframing requires comparison. Ensure you pass at least two `PromptVariant` objects.
*   **`ValueError: Model labels collide after filesystem sanitization`**: Check your model labels. `org/model` and `org:model` both become `org_model`. Rename one.
*   **Cache Ignored Warning**: If you see "cached runs... produced under a different configuration," it means you changed a parameter (e.g., judge model) after the first run. The system is correctly invalidating the old cache. Delete the `save_dir` subfolder for that model if you want to force a clean start without the warning.

### See Also

*   [CLI Usage](cli-usage.md)
*   [Core Architecture](core-architecture.md)
*   [Cross-Judge Validation](cross-judge-validation.md)
*   [Getting Started](getting-started.md)
