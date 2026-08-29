## Architecture

SimpleAudit is designed as a lightweight, local-first framework for validating comparative LLM safety scoring. The architecture separates concerns into three primary layers: **Scenario Management**, **Auditing Execution**, and **Result Aggregation**. This separation allows users to swap target models, judges, and scenario packs independently without modifying core logic.

### Core Components

The system relies on three main classes and supporting utility functions:

1.  **`AuditExperiment`**: The high-level orchestrator. It manages multiple models, handles repetitions for stability analysis, and coordinates the execution of audits.
2.  **`ModelAuditor`**: The execution engine. It performs the actual interaction with a specific target model and judge model for a given set of scenarios.
3.  **`AuditResults` / `RepeatedExperimentResults`**: The data structures that store, aggregate, and serialize the outcomes of the audits.

### The Auditing Pipeline

The workflow follows a strict sequence:

1.  **Scenario Selection**: Scenarios are loaded via `get_scenarios(pack_name)` or defined inline. These define the adversarial prompts and expected behaviors.
2.  **Model Interaction**: The `ModelAuditor` sends probe prompts to the target model. It supports multi-turn interactions controlled by `max_turns`.
3.  **Judgment**: A separate judge model evaluates the target's responses. The judge configuration is determined by the `judge` parameter (e.g., `"safety"`, `"abstention"`) or custom prompts.
4.  **Aggregation**: Results are compiled into `AuditResult` objects. If `n_repetitions > 1`, these are wrapped in `RepeatedExperimentResults` to calculate stability metrics.

### Key Classes

#### `AuditExperiment`

The primary entry point for running comprehensive audits across multiple models or repetitions.

**Constructor Parameters:**
*   `models` (List[Dict[str, Any]]): A list of model configurations. Each dict typically contains `model` and `provider` keys.
*   `judge_model` (Optional[str]): The model used for judging.
*   `judge_provider` (Optional[str]): The provider for the judge model.
*   `auditor_model` (Optional[str]): The model used for generating probes (if distinct from the target).
*   `judge` (Optional[str]): The name of the built-in judge strategy (e.g., `"safety"`, `"abstention"`).
*   `n_repetitions` (int): Number of times to run the experiment for each model to assess stability.
*   `save_dir` (Optional[str]): Directory to save results.
*   `verbose` (bool): Enable verbose logging.

**Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Executes the audit synchronously.
*   `run_async(scenarios, max_turns, language, max_workers)`: Executes the audit asynchronously.

#### `ModelAuditor`

Handles the low-level interaction between the target model and the judge.

**Constructor Parameters:**
*   `model` (str): The target model name.
*   `provider` (str): The provider for the target model (e.g., `"ollama"`, `"openai"`).
*   `judge_model` (str): The judge model name.
*   `judge_provider` (str): The provider for the judge model.
*   `judge` (Optional[str]): The judge strategy name.
*   `probe_prompt` (Optional[str]): Custom prompt for the auditor.
*   `judge_prompt` (Optional[str]): Custom prompt for the judge.
*   `max_turns` (int): Maximum number of back-and-forth turns per scenario.
*   `json_format` (bool): Whether to enforce JSON output from the judge.

**Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Runs the audit for this specific model.
*   `run_scenario(...)`: Executes a single scenario.
*   `get_scenarios(pack_name)`: Retrieves scenarios by pack name.

#### `AuditResults`

A container for the results of a single audit run.

**Properties:**
*   `score`: The aggregated safety score.
*   `severity_distribution`: A dictionary mapping severity levels to counts.
*   `token_usage`: Detailed token consumption statistics.
*   `passed` / `failed`: int flags indicating overall success/failure.

**Methods:**
*   `to_dict()`: Serializes results to a dictionary.
*   `save(filepath)`: Saves results to a JSON file.
*   `load(filepath)`: Class method to load results from a file.
*   `plot(save_path)`: Generates a visualization of the results.

#### `RepeatedExperimentResults`

Used when `n_repetitions > 1`. It aggregates results across multiple runs to provide stability insights.

**Methods:**
*   `stability(model_name)`: Returns a `ModelStabilityReport` for a specific model.
*   `summary()`: Provides a high-level summary of the repeated experiments.
*   `to_dict()`: Serializes the aggregated results.

### Scenario and Judge Management

SimpleAudit includes a library of predefined scenario packs and judge configurations.

**Scenario Packs:**
Available packs can be listed using `list_scenario_packs()`. Common packs include:
*   `safety`: General safety scenarios.
*   `bullshitbench`: Scenarios designed to test for hallucinations and factual errors.
*   `hei_refusal`: Scenarios focused on refusal behavior in Norwegian health contexts.

**Judge Configurations:**
Judges are retrieved using `get_judge(name)`. Built-in judges include:
*   `SAFETY_JUDGE`: Evaluates general AI safety.
*   `ABSTENTION_JUDGE`: Evaluates whether the model appropriately abstains from answering.
*   `FACTUALITY_JUDGE`: Evaluates factual accuracy.

### Code Example: Basic Audit

The following example demonstrates how to set up and run a basic audit using `AuditExperiment`.

```python
from simpleaudit import AuditExperiment, get_scenarios

# Define the models to audit
models = [
    {
        "model": "llama3.2:3b",
        "provider": "ollama"
    }
]

# Load scenarios from a built-in pack
scenarios = get_scenarios("safety")

# Initialize the experiment
experiment = AuditExperiment(
    models=models,
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    n_repetitions=1,
    verbose=True
)

# Run the audit
results = experiment.run(
    scenarios=scenarios,
    max_turns=3,
    language="en"
)

# Access the results
for model_name, model_results in results.items():
    print(f"Model: {model_name}")
    print(f"Score: {model_results.score}")
    print(f"Severity Distribution: {model_results.severity_distribution}")
```

### Visualization

SimpleAudit includes a visualization server to inspect results.

**Functions:**
*   `start_server(results_dir, host, port)`: Starts a local web server to view audit results.
*   `root()`: Handles the root endpoint of the visualization server.

To view results after an audit, you can start the server pointing to the directory where results were saved:

```python
from simpleaudit.visualization.server import start_server

start_server(results_dir="./results", host="127.0.0.1", port=8080)
```

### Design Principles

1.  **Local-First**: The framework is designed to work with local models (e.g., via Ollama) without requiring external API keys or internet connectivity for the core audit logic.
2.  **Extensibility**: Custom judges and scenarios can be injected via `probe_prompt`, `judge_prompt`, and inline scenario definitions.
3.  **Reproducibility**: The `RepeatedExperimentResults` class ensures that stability metrics are calculated consistently, allowing for rigorous comparative analysis.

By separating the execution (`ModelAuditor`) from the orchestration (`AuditExperiment`) and aggregation (`AuditResults`), SimpleAudit provides a flexible yet structured approach to LLM safety auditing.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
