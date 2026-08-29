## Quickstart

This guide demonstrates how to run your first local audit using the `simpleaudit-docs` library. SimpleAudit is a lightweight, local-first framework for AI safety auditing. It validates comparative LLM safety scoring without requiring ground-truth labels, supporting both local models (via Ollama or Hugging Face) and API-hosted models.

The core workflow involves two main components:
1.  **`ModelAuditor`**: Executes a single audit run against a specific target model using a judge model to evaluate responses.
2.  **`AuditExperiment`**: Manages multiple models and repetitions, providing stability reports and aggregated results.

### Prerequisites

*   Python 3.11+
*   `simpleaudit` installed (`pip install simpleaudit`)
*   A local LLM runtime (e.g., Ollama) or API keys for hosted models.

For local development without API keys, ensure Ollama is running:
```bash
ollama serve
ollama pull llama3.2:3b     # Target model
ollama pull gemma3:latest   # Judge model
```

### 1. Basic Audit with `ModelAuditor`

The `ModelAuditor` class is the primary entry point for running a single audit. It requires a target model, a judge model, and their respective providers.

#### Constructor Parameters

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `model` | `str` | The name/ID of the target model to audit. |
| `provider` | `str` | The provider for the target model (e.g., `"ollama"`, `"openai"`). |
| `judge_model` | `str` | The name/ID of the model used to judge the target's responses. |
| `judge_provider` | `str` | The provider for the judge model. |
| `api_key` | `Optional[str]` | API key for the target model (if required). |
| `base_url` | `Optional[str]` | Custom base URL for the target model endpoint. |
| `judge_api_key` | `Optional[str]` | API key for the judge model. |
| `judge_base_url` | `Optional[str]` | Custom base URL for the judge model endpoint. |
| `json_format` | `bool` | Whether to enforce JSON output from the judge (default: `True`). |
| `max_turns` | `int` | Maximum number of conversational turns per scenario (default: `5`). |
| `verbose` | `bool` | Enable verbose logging (default: `False`). |
| `show_progress` | `bool` | Display progress bars (default: `True`). |

#### Example: Local Ollama Audit

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define models
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL = "gemma3:latest"

# Load a small subset of scenarios for a quick test
# 'safety' is a built-in scenario pack
scenarios = get_scenarios("safety")[:2]

# Initialize the auditor
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    json_format=False,  # Ollama may not support strict JSON mode
    max_turns=2
)

# Run the audit
results = auditor.run(scenarios)

# Inspect results
print(f"Total scenarios: {len(results)}")
print(f"Score: {results.score}")
print(f"Passed: {results.passed}")
print(f"Failed: {results.failed}")
```

### 2. Running Multiple Models with `AuditExperiment`

For comparative analysis, use `AuditExperiment`. This class handles multiple models and can perform repeated runs to generate stability reports.

#### Constructor Parameters

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `models` | `List[Dict[str, Any]]` | List of model configurations. Each dict should contain `model` and `provider` keys. |
| `judge_model` | `Optional[str]` | Name of the judge model. |
| `judge_provider` | `Optional[str]` | Provider for the judge model. |
| `n_repetitions` | `int` | Number of times to run the experiment for each model (default: `1`). |
| `save_dir` | `Optional[str]` | Directory to save results. |
| `verbose` | `bool` | Enable verbose logging. |

#### Example: Comparative Audit

```python
from simpleaudit import AuditExperiment, get_scenarios

# Define models to compare
models = [
    {"model": "llama3.2:3b", "provider": "ollama"},
    {"model": "gemma3:latest", "provider": "ollama"}
]

# Load scenarios
scenarios = get_scenarios("safety")[:3]

# Initialize experiment
experiment = AuditExperiment(
    models=models,
    judge_model="gemma3:latest",
    judge_provider="ollama",
    n_repetitions=2,  # Run twice for stability
    save_dir="./audit_results"
)

# Run the experiment
repeated_results = experiment.run(scenarios)

# Access stability report for a specific model
stability_report = repeated_results.stability("llama3.2:3b")
print(stability_report.summary())

# Save results
repeated_results.save("audit_results.json")
```

### 3. Working with Results

The `AuditResults` object returned by `ModelAuditor.run()` and `AuditExperiment.run()` provides access to detailed audit data.

#### Key Properties

*   `score`: The overall safety score.
*   `passed`: Number of scenarios that passed.
*   `failed`: Number of scenarios that failed.
*   `critical_count`: Number of critical issues found.
*   `severity_distribution`: A dictionary showing the distribution of severity levels.
*   `all_issues`: A list of all identified issues.
*   `all_recommendations`: A list of recommendations for improvement.

#### Saving and Loading Results

You can save results to a JSON file and load them later:

```python
# Save
results.save("my_audit.json")

# Load
loaded_results = AuditResults.load("my_audit.json")
print(loaded_results.summary())
```

### 4. Visualization

SimpleAudit includes a built-in visualization server to explore audit results.

```python
from simpleaudit.visualization.server import start_server

# Start the server pointing to the directory where results are saved
start_server(
    results_dir="./audit_results",
    host="localhost",
    port=8080
)
```

Navigate to `http://localhost:8080` in your browser to view the audit results, scenario details, and stability reports.

### 5. Custom Judges

You can customize the judge's behavior using the `probe_prompt` and `judge_prompt` parameters in `ModelAuditor` or `AuditExperiment`. This allows you to define specific evaluation criteria beyond the default safety schema.

```python
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    probe_prompt="You are a helpful assistant.",
    judge_prompt="Evaluate the response for factual accuracy and helpfulness."
)
```

### Next Steps

*   Explore available scenario packs using `list_scenario_packs()`.
*   Review the [Scenario Guidelines](https://github.com/kelkalot/simpleaudit/blob/main/simpleaudit/scenarios/simpleaudit_scenario_guidelines_v1.0.md) for creating custom scenarios.
*   Check the `examples/` directory in the repository for more advanced use cases, including custom judges and fake audits for CI testing.

### See Also

*   [Installation](installation.md)
*   [Key Ideas](key-ideas.md)
*   [Architecture](architecture.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
