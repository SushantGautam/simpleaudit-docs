## Architecture

SimpleAudit is designed as a lightweight, local-first framework for validating comparative LLM safety scoring. Its architecture separates the **auditing logic** (interacting with the target model), the **evaluation logic** (judging the responses), and the **experiment orchestration** (managing repetitions and aggregation).

The core flow involves three primary layers:
1.  **Scenario Definition**: Loading predefined adversarial test cases.
2.  **Auditing**: The `ModelAuditor` interacts with the target LLM and an LLM-based judge to produce `AuditResults`.
3.  **Experimentation**: The `AuditExperiment` manages multiple models and repetitions, aggregating results into `RepeatedExperimentResults` for stability analysis.

### Core Components

#### 1. Scenario Management
Scenarios are the unit of testing. They are loaded via the `scenarios` module.

*   **`scenarios.get_scenarios(pack_name)`**: Retrieves a list of scenario dictionaries for a given pack name (e.g., `"safety"`, `"bullshitbench"`, `"health"`).
*   **`scenarios.list_scenario_packs()`**: Lists available scenario packs.

Available scenario packs include `SAFETY_SCENARIOS`, `HEALTH_SCENARIOS`, `RAG_SCENARIOS`, `BULLSHITBENCH_SCENARIOS`, and domain-specific packs like `NAV_AAP_SCENARIOS` and `SKATTEETATEN_SCENARIOS`.

#### 2. The Auditor: `ModelAuditor`
The `ModelAuditor` class handles the execution of a single model against a set of scenarios. It manages the interaction between the **Target Model** (the LLM being audited) and the **Judge Model** (the LLM evaluating the target's responses).

**Constructor Parameters:**
*   `model`: The name of the target model.
*   `provider`: The provider for the target model (e.g., `"ollama"`, `"openai"`).
*   `judge_model`: The name of the judge model.
*   `judge_provider`: The provider for the judge model.
*   `api_key` / `base_url`: Credentials and endpoint for the target model.
*   `judge_api_key` / `judge_base_url`: Credentials and endpoint for the judge model.
*   `auditor_model` / `auditor_provider`: Optional separate model for the "auditor" role (probing).
*   `judge`: Name of the judge configuration (e.g., `"safety"`, `"abstention"`).
*   `probe_prompt` / `judge_prompt`: Custom prompts to override default behavior.
*   `json_format`: Boolean to enforce JSON output from the judge.
*   `max_turns`: Maximum number of interaction turns per scenario.
*   `verbose` / `show_progress`: Logging and progress bar controls.

**Key Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Executes the audit synchronously.
*   `run_async(scenarios, max_turns, language, max_workers)`: Executes the audit asynchronously.
*   `run_scenario(...)`: Executes a single scenario.
*   `get_scenarios(pack_name)`: Helper to load scenarios.
*   `parse_json_response(response, default_severity)`: Parses the judge's JSON output.

#### 3. Results Aggregation: `AuditResults`
The output of a single `ModelAuditor` run is an `AuditResults` object.

**Properties:**
*   `score`: The aggregate safety score.
*   `passed` / `failed`: int indicators for overall pass/fail status.
*   `critical_count`: Number of critical issues found.
*   `severity_distribution`: A dictionary mapping severity levels to counts.
*   `all_issues` / `all_recommendations`: Lists of specific issues and recommendations.
*   `token_usage`: Dictionary containing input/output token counts for auditor, judge, and target models.

**Methods:**
*   `summary()`: Returns a human-readable summary string.
*   `to_dict()`: Serializes results to a dictionary.
*   `save(filepath)`: Saves results to a JSON file.
*   `load(filepath)`: Loads results from a JSON file.
*   `plot(save_path)`: Generates a visualization of the results.

#### 4. Experiment Orchestration: `AuditExperiment`
For robust validation, SimpleAudit supports running multiple models and multiple repetitions. The `AuditExperiment` class orchestrates this.

**Constructor Parameters:**
*   `models`: A list of dictionaries, where each dictionary defines a model configuration (keys typically include `model`, `provider`, `api_key`, etc.).
*   `judge_model` / `judge_provider`: Global judge configuration if not specified per model.
*   `n_repetitions`: Number of times to run the experiment for each model.
*   `save_dir`: Directory to save intermediate results.
*   `on_model_done`: Callback function invoked when a model's results are complete.

**Key Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Runs the full experiment synchronously.
*   `run_async(scenarios, max_turns, language, max_workers)`: Runs the full experiment asynchronously.

The output is a `RepeatedExperimentResults` object.

#### 5. Stability Analysis: `RepeatedExperimentResults`
This container holds the results from multiple runs of the same model. It implements the mapping protocol (`__getitem__`, `keys`, `values`, etc.).

**Key Methods:**
*   `stability(model_name)`: Returns a `ModelStabilityReport` for the specified model.
*   `summary()`: Aggregated summary across all models and runs.
*   `to_dict()`: Serializes the entire experiment results.
*   `save(filepath)` / `load(filepath)`: Persistence methods.

The `ModelStabilityReport` provides metrics on the consistency of the scoring across repetitions.

### Data Flow

1.  **Initialization**: An `AuditExperiment` is created with a list of model configurations.
2.  **Scenario Loading**: Scenarios are loaded via `get_scenarios`.
3.  **Execution**:
    *   For each model in the experiment:
        *   A `ModelAuditor` is instantiated.
        *   The auditor runs the scenarios, interacting with the target and judge LLMs.
        *   An `AuditResults` object is generated.
    *   This process is repeated `n_repetitions` times.
4.  **Aggregation**:
    *   All `AuditResults` instances for a specific model are collected.
    *   `RepeatedExperimentResults` aggregates these.
    *   Stability metrics are computed via `ModelStabilityReport`.

### Code Example: Basic Audit

```python
from simpleaudit import ModelAuditor, get_scenarios

# Load scenarios
scenarios = get_scenarios("safety")

# Initialize the auditor
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    json_format=False,
    max_turns=3
)

# Run the audit
results = auditor.run(scenarios)

# Inspect results
print(results.summary())
print(f"Score: {results.score}")
print(f"Critical Issues: {results.critical_count}")

# Save results
results.save("audit_results.json")
```

### Code Example: Multi-Model Experiment

```python
from simpleaudit import AuditExperiment, get_scenarios

scenarios = get_scenarios("bullshitbench")

# Define models to audit
models = [
    {
        "model": "llama3.2:3b",
        "provider": "ollama"
    },
    {
        "model": "gemma3:latest",
        "provider": "ollama"
    }
]

# Initialize experiment with 2 repetitions
experiment = AuditExperiment(
    models=models,
    n_repetitions=2,
    save_dir="./experiment_output"
)

# Run the experiment
repeated_results = experiment.run(scenarios)

# Get stability report for a specific model
stability_report = repeated_results.stability("llama3.2:3b")
print(stability_report.summary())

# Save full experiment data
repeated_results.save("experiment_results.json")
```

### Visualization

SimpleAudit includes a visualization server for exploring results.

*   `visualization.server.start_server(results_dir, host, port)`: Starts a local web server to view audit results.
*   The server provides endpoints for viewing file trees, JSON files, and scenario details.

To start the server:

```python
from simpleaudit.visualization.server import start_server

start_server(results_dir="./experiment_output", host="localhost", port=8000)
```

### Key Design Principles

1.  **No Ground-Truth Labels**: The system relies on comparative scoring by a judge LLM rather than human-labeled ground truth.
2.  **Local-First**: Designed to work with local models (e.g., via Ollama) without requiring external API keys by default.
3.  **Extensibility**: Custom judges and prompts can be injected via `judge_prompt` and `probe_prompt` parameters.
4.  **Reproducibility**: The `AuditExperiment` class ensures that results are reproducible by managing repetitions and saving intermediate data.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Installation](installation.md)
*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
