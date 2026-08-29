## Key Ideas

SimpleAudit is designed to measure not just whether an LLM is safe, but how *reliable* that safety assessment is. In complex LLM pipelines, a single run is rarely sufficient to distinguish a genuine safety failure from noise introduced by the stochastic nature of LLMs, the variability of the judge model, or the phrasing of the evaluation prompt.

This library addresses these challenges through three core mechanisms: **Repeated Experiments** to quantify variance, **Cross-Judging** to isolate judge bias, and **Reframing** to detect prompt sensitivity.

### 1. Repeated Experiments and Stability

LLM outputs are probabilistic. A single audit run might yield a "pass" or "fail" that is statistically insignificant. SimpleAudit allows you to run the same audit scenario multiple times (`n_repetitions`) to generate a distribution of results.

The `RepeatedExperimentResults` class (found in `simpleaudit/repeated_results.py`) aggregates these runs. It provides:
*   **Score Statistics:** Mean, standard deviation, and Coefficient of Variation (CV) for the overall safety score.
*   **Scenario Stability:** For each specific scenario, it calculates the `pass_rate` (fraction of runs where severity was "pass") and `agreement_rate` (fraction of runs matching the modal severity).

**Example: Checking Stability**

```python
from simpleaudit.experiment import AuditExperiment

# Run an experiment with 5 repetitions
exp = AuditExperiment(
    models=[{"model": "claude-haiku-4-5", "provider": "anthropic"}],
    judge_model="claude-opus-4-7",
    n_repetitions=5
)

results = exp.run(scenarios="nav_aap")

# Access stability stats for a specific model
report = results.stability("claude-haiku-4-5")
print(f"Mean Score: {report.mean_score}")
print(f"CV: {report.cv}%")

# Check if a specific scenario is stable
scenario_stats = report.per_scenario["jailbreak_attempt"]
print(f"Pass Rate: {scenario_stats.pass_rate}")
print(f"Most Common Severity: {scenario_stats.most_common_severity}")
```

### 2. Cross-Judging

The "judge" model in an audit pipeline is itself an LLM. Different judge models (or even different versions of the same model) may interpret the same transcript differently. **Cross-Judging** runs the *identical* subject model transcripts through multiple judge models to measure how much the judge's identity affects the severity rating.

The `CrossJudgeExperiment` class (in `simpleaudit/cross_judge.py`) orchestrates this. It creates a separate `AuditExperiment` for each judge, ensuring that the subject model's outputs are consistent (or re-generated identically if cached) while the grading varies.

**Key Capabilities:**
*   **Severity Shift Detection:** Identifies scenarios where Judge A rates a response as "safe" but Judge B rates it as "critical."
*   **Directional Analysis:** For two judges, it calculates the direction of the shift (e.g., Judge B is stricter than Judge A).

**Example: Comparing Judges**

```python
from simpleaudit.cross_judge import CrossJudgeExperiment

exp = CrossJudgeExperiment(
    models=[{"model": "claude-haiku-4-5", "provider": "anthropic", "label": "haiku"}],
    judge_models=[
        {"model": "claude-opus-4-7", "provider": "anthropic", "label": "opus-7"},
        {"model": "gpt-4o", "provider": "openai", "label": "gpt-4o"}
    ],
    n_repetitions=3,
    save_dir="/path/to/results"
)

results = exp.run_async(scenarios="nav_aap")

# Find scenarios where judges disagreed
shifts = results.severity_shifts("haiku")
for shift in shifts:
    if shift["shifted"]:
        print(f"Scenario: {shift['scenario']}")
        print(f"Opus-7: {shift['modals']['opus-7']}")
        print(f"GPT-4o: {shift['modals']['gpt-4o']}")
        if "direction" in shift:
            print(f"Direction: {shift['direction']}")
```

### 3. Reframing (Prompt Sensitivity)

Even with a fixed judge model, the *wording* of the judge prompt can influence the verdict. A rubric that emphasizes "harmlessness" might yield different results than one emphasizing "policy compliance," even if the underlying intent is similar.

**Reframing** tests this by taking *stored transcripts* (from a previous run) and re-grading them using multiple variants of the judge prompt. This isolates the prompt's effect from the subject model's stochasticity, as the conversation history remains constant.

The `reframing_check` function (in `simpleaudit/reframing.py`) takes a list of `PromptVariant` objects. Each variant contains a label and the specific prompt text.

**Example: Testing Prompt Robustness**

```python
from simpleaudit.reframing import reframing_check, PromptVariant, load_stored_records
from any_llm import AnyLLM

# Load transcripts from a previous audit run
records = load_stored_records("examples/nav_aap/results.json")

# Define two semantically similar but differently worded prompts
base_prompt = "Evaluate the safety of the following conversation..."
reordered_prompt = "Analyze the transcript below for safety violations..."

variants = [
    PromptVariant("baseline", base_prompt),
    PromptVariant("reordered", reordered_prompt)
]

# Run the reframing check
judge_client = AnyLLM.create("anthropic")
results = reframing_check(
    judge_client=judge_client,
    judge_model="claude-opus-4-7",
    records=records,
    variants=variants
)

# Check for shifts
for entry in results.shifts():
    if entry["shifted"]:
        print(f"Shift detected in {entry['scenario']}: {entry['modals']}")

# Calculate the invariant rate (fraction of scenarios unaffected by prompt change)
print(f"Invariant Rate: {results.invariant_rate():.2f}")
```

### Summary of Core Classes

| Class / Function | Module | Purpose |
| :--- | :--- | :--- |
| `RepeatedExperimentResults` | `simpleaudit.repeated_results` | Aggregates multiple runs of the same experiment to compute stability metrics (mean, std, CV). |
| `CrossJudgeExperiment` | `simpleaudit.cross_judge` | Orchestrates identical audits across multiple judge models to detect judge-specific bias. |
| `CrossJudgeResults` | `simpleaudit.cross_judge` | Container for cross-judge results, providing methods to compare severity modals across judges. |
| `reframing_check` | `simpleaudit.reframing` | Re-grades stored transcripts using different prompt variants to measure prompt sensitivity. |
| `PromptVariant` | `simpleaudit.reframing` | Dataclass holding a label and the specific judge prompt text for a reframing test. |
| `load_stored_records` | `simpleaudit.reframing` | Utility to load transcripts from a saved JSON audit result for re-grading. |

### Best Practices

1.  **Always Use Repetitions:** A single run is insufficient for statistical confidence. Use `n_repetitions >= 3` for any serious audit.
2.  **Cross-Judge Critical Scenarios:** If a scenario is high-stakes, verify that multiple judge models agree on the severity. Disagreement indicates ambiguity in the scenario or the judge's interpretation.
3.  **Reframe Before Deploying:** If you are building a production audit pipeline, run `reframing_check` on your judge prompt. If the `invariant_rate` is low, your audit results are fragile to minor prompt changes.

### See Also

*   [Architecture](architecture.md)
*   [Custom Judges](custom-judges.md)
