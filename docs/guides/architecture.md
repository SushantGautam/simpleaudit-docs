## Architecture

SimpleAudit is a lightweight framework for validating LLM safety through comparative auditing. It operates without ground-truth labels by using a "judge" model to evaluate the outputs of a "target" model against specific safety scenarios. The architecture is modular, separating scenario definitions, judge configurations, the core audit loop, and result aggregation.

### Core Components

The system is built around four primary modules:

1.  **Scenarios (`simpleaudit.scenarios`)**: Defines the test cases. Scenarios are grouped into "packs" (e.g., `safety`, `rag`, `bullshitbench`). Each scenario is a dictionary containing a `name`, `description`, `test_prompt`, and optional `expected_behavior`.
2.  **Judges (`simpleaudit.judges`)**: Defines the evaluation criteria. Judges are configurations that specify the `probe_prompt` (how to attack the model) and `judge_prompt` (how to evaluate the response). Built-in judges include `safety`, `abstention`, `helpfulness`, `factuality`, and `harm`.
3.  **ModelAuditor (`simpleaudit.model_auditor`)**: The core engine. It manages the interaction between the target model, the auditor (probe generator), and the judge model. It handles multi-turn conversations, token counting, and JSON parsing of judge responses.
4.  **AuditExperiment (`simpleaudit.experiment`)**: The high-level orchestrator. It allows running audits across multiple models, handling repetitions for stability analysis, and managing result persistence.

### The Audit Loop

The fundamental unit of work is the **scenario audit**, executed by `ModelAuditor.run_scenario()`. The process follows a multi-turn loop:

1.  **Probe Generation**:
    *   **Turn 0**: If the scenario defines a `test_prompt`, it is used verbatim. Otherwise, the auditor model generates a probe based on the scenario description.
    *   **Subsequent Turns**: The auditor model generates the next user message based on the conversation history.
2.  **Target Response**: The target model responds to the probe.
3.  **Judgment**: After the specified number of turns (`max_turns`), the judge model evaluates the full conversation.
    *   The judge receives the conversation history and the scenario's `expected_behavior` (if defined).
    *   It outputs a JSON object containing `severity` (critical, high, medium, low, pass), `issues_found`, `positive_behaviors`, `summary`, and `recommendations`.
    *   If `json_format=True`, a strict JSON schema is enforced via the API's `response_format` parameter.

### Judge Registry

The judge registry (`simpleaudit.judges`) provides a standardized way to load evaluation criteria.

*   **`get_judge(name)`**: Returns a configuration dictionary for a named judge (e.g., `"safety"`). The config includes `probe_prompt`, `judge_prompt`, and optionally `response_schema`.
*   **`list_judge_configs()`**: Returns a dictionary mapping judge names to their descriptions.

When initializing a `ModelAuditor`, you can specify a named judge via the `judge` parameter. This automatically populates `probe_prompt` and `judge_prompt`. However, explicit `probe_prompt` or `judge_prompt` arguments always override the named judge's defaults, allowing for partial customization.

### Scenario Structure

Scenarios are accessed via `get_scenarios(pack_name)`. Available packs include:

*   `safety`: General AI safety scenarios.
*   `rag`: Retrieval-Augmented Generation specific scenarios.
*   `bullshitbench`: Combined v1 and v2 scenarios for detecting nonsensical or broken-premise responses.
*   `hei_refusal`: Norwegian youth Q&A refusal/guidance edge cases.
*   `all`: A union of all available packs.

Each scenario dictionary must contain:
*   `name` (str): Unique identifier.
*   `description` (str): Context for the auditor.
*   `test_prompt` (str): The initial user message.
*   `expected_behavior` (List[str], optional): Specific criteria the judge should verify.

### High-Level Experiments

For comparative analysis across multiple models, use `AuditExperiment`. It wraps `ModelAuditor` and adds:

*   **Multi-Model Support**: Accepts a list of model configurations.
*   **Repetitions**: Runs each model `n_repetitions` times to calculate stability metrics.
*   **Persistence**: If `save_dir` is provided, results are saved to disk after each run, enabling resumable experiments.
*   **Stability Reporting**: Generates `ModelStabilityReport` objects via `RepeatedExperimentResults.stability(model_name)`.

### Code Example: Basic Audit

```python
from simpleaudit import ModelAuditor, get_scenarios

# Load scenarios
scenarios = get_scenarios("safety")[:3]

# Initialize the auditor
# Note: 'judge' parameter loads a built-in config. 
# 'model' is the target, 'judge_model' is the evaluator.
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",  # Uses built-in safety judge config
    max_turns=3,
    verbose=True
)

# Run the audit
results = auditor.run(scenarios)

# Print summary
results.summary()
```

### Code Example: Comparative Experiment

```python
from simpleaudit.experiment import AuditExperiment
from simpleaudit import get_scenarios

# Define models to compare
models = [
    {"model": "llama3.2:3b", "provider": "ollama", "label": "Llama 3.2 3B"},
    {"model": "gemma3:latest", "provider": "ollama", "label": "Gemma 3"},
]

# Initialize experiment
experiment = AuditExperiment(
    models=models,
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    n_repetitions=2,  # Run each model twice for stability
    save_dir="./audit_results"
)

# Run experiment
# scenarios can be a pack name string or a list of dicts
results = experiment.run("safety", max_turns=2)

# Access stability report for a specific model
stability_report = results.stability("Llama 3.2 3B")
stability_report.summary()
```

### Result Aggregation

`AuditResults` provides aggregated metrics:

*   `score`: A 0-100 safety score based on severity distribution.
*   `severity_distribution`: Counts of each severity level.
*   `token_usage`: Detailed input/output token counts for auditor, judge, and target.
*   `passed` / `failed`: Counts of scenarios that passed or failed.

`RepeatedExperimentResults` extends this by storing multiple runs per model, enabling the calculation of mean scores, standard deviations, and per-scenario agreement rates.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
