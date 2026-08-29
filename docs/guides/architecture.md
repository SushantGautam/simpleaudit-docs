## Architecture

SimpleAudit is designed as a lightweight, local-first framework for validating comparative LLM safety scoring. The architecture separates the execution of audits, the aggregation of results, and the visualization of data into distinct modules. This page outlines the core components of the experiment lifecycle, model auditing, and result handling.

### Core Components

The library is structured around three primary layers:

1.  **Experiment Layer**: Orchestrates the execution of audits across one or more models, handling concurrency and repetition.
2.  **Auditor Layer**: Executes individual scenarios against a target model and evaluates the responses using a judge model.
3.  **Results Layer**: Aggregates, stores, and analyzes the outcomes of audits, including stability metrics for repeated runs.

### Experiment Lifecycle

The `AuditExperiment` class serves as the high-level entry point for running safety audits. It manages the configuration of target models and judge models, and coordinates the execution of scenarios.

#### `AuditExperiment`

The `AuditExperiment` class allows you to define a set of models to audit and run them against specific scenario packs.

**Constructor Parameters:**
*   `models` (List[Dict[str, Any]]): A list of dictionaries defining the target models to be audited.
*   `judge_model` (Optional[str]): The name of the model used for judging.
*   `judge_provider` (Optional[str]): The provider for the judge model (e.g., "ollama", "openai").
*   `judge_api_key` (Optional[str]): API key for the judge provider.
*   `judge_base_url` (Optional[str]): Base URL for the judge provider.
*   `auditor_model` (Optional[str]): The model used for generating probe prompts (if different from the judge).
*   `auditor_provider` (Optional[str]): Provider for the auditor model.
*   `auditor_api_key` (Optional[str]): API key for the auditor provider.
*   `auditor_base_url` (Optional[str]): Base URL for the auditor provider.
*   `judge` (Optional[str]): The name of the judge configuration to use (e.g., "safety", "abstention").
*   `probe_prompt` (Optional[str]): Custom prompt for the auditor to generate probes.
*   `judge_prompt` (Optional[str]): Custom prompt for the judge to evaluate responses.
*   `json_format` (bool): Whether to enforce JSON output format for the judge.
*   `verbose` (bool): Enable verbose logging.
*   `show_progress` (bool): Display progress bars during execution.
*   `n_repetitions` (int): Number of times to repeat the experiment for stability analysis.
*   `save_dir` (Optional[str]): Directory to save results.
*   `on_model_done` (Optional[Callable]): Callback function invoked when a model's audit is complete.

**Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Executes the audit synchronously.
*   `run_async(scenarios, max_turns, language, max_workers)`: Executes the audit asynchronously.

**Example: Running an Experiment**

```python
from simpleaudit.experiment import AuditExperiment
from simpleaudit.scenarios import get_scenarios

# Define the models to audit
models = [
    {
        "name": "llama3.2:3b",
        "provider": "ollama",
        "base_url": "http://localhost:11434"
    }
]

# Initialize the experiment
experiment = AuditExperiment(
    models=models,
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    n_repetitions=2,
    save_dir="./results"
)

# Get scenarios
scenarios = get_scenarios("safety")

# Run the experiment
results = experiment.run(
    scenarios=scenarios,
    max_turns=5,
    language="en",
    max_workers=2
)
```

### Model Auditing

The `ModelAuditor` class handles the low-level execution of a single model against a set of scenarios. It manages the interaction between the target model, the auditor (probe generator), and the judge.

#### `ModelAuditor`

**Constructor Parameters:**
*   `model` (str): The name of the target model.
*   `provider` (str): The provider for the target model.
*   `judge_model` (str): The name of the judge model.
*   `judge_provider` (str): The provider for the judge model.
*   `api_key` (Optional[str]): API key for the target model.
*   `base_url` (Optional[str]): Base URL for the target model.
*   `system_prompt` (Optional[str]): System prompt for the target model.
*   `judge_api_key` (Optional[str]): API key for the judge.
*   `judge_base_url` (Optional[str]): Base URL for the judge.
*   `auditor_model` (Optional[str]): Model for generating probes.
*   `auditor_provider` (Optional[str]): Provider for the auditor.
*   `auditor_api_key` (Optional[str]): API key for the auditor.
*   `auditor_base_url` (Optional[str]): Base URL for the auditor.
*   `judge` (Optional[str]): Judge configuration name.
*   `probe_prompt` (Optional[str]): Custom probe prompt.
*   `judge_prompt` (Optional[str]): Custom judge prompt.
*   `judge_response_schema` (Optional[Dict[str, Any]]): Schema for judge response.
*   `json_format` (bool): Enforce JSON output.
*   `max_turns` (int): Maximum number of turns per scenario.
*   `verbose` (bool): Enable verbose logging.
*   `show_progress` (bool): Display progress bars.

**Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Executes the audit for this specific model.
*   `run_async(scenarios, max_turns, language, max_workers)`: Asynchronous execution.
*   `run_scenario(name, description, expected_behavior, test_prompt, max_turns, language, pbar_audit, pbar_judge, max_workers)`: Runs a single scenario.
*   `strip_thinking(text)`: Removes thinking tokens from model output.
*   `parse_json_response(response, default_severity)`: Parses JSON response from the judge.
*   `get_scenarios(pack_name)`: Retrieves scenarios by pack name.

**Example: Direct Model Auditing**

```python
from simpleaudit.model_auditor import ModelAuditor
from simpleaudit.scenarios import get_scenarios

auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    max_turns=3
)

scenarios = get_scenarios("safety")[:5]
results = auditor.run(
    scenarios=scenarios,
    max_turns=3,
    language="en",
    max_workers=1
)
```

### Result Handling

Audit results are encapsulated in the `AuditResults` and `RepeatedExperimentResults` classes. These classes provide methods for summarizing, saving, and analyzing the outcomes.

#### `AuditResults`

Represents the results of a single run of an audit.

**Properties:**
*   `total_auditor_input_tokens`: Total input tokens used by the auditor.
*   `total_auditor_output_tokens`: Total output tokens used by the auditor.
*   `total_judge_input_tokens`: Total input tokens used by the judge.
*   `total_judge_output_tokens`: Total output tokens used by the judge.
*   `total_target_input_tokens`: Total input tokens used by the target model.
*   `total_target_output_tokens`: Total output tokens used by the target model.
*   `token_usage`: Aggregated token usage statistics.
*   `severity_distribution`: Distribution of severity levels.
*   `all_issues`: List of all identified issues.
*   `all_recommendations`: List of all recommendations.
*   `passed`: Number of passed scenarios.
*   `failed`: Number of failed scenarios.
*   `critical_count`: Number of critical issues.
*   `score`: Overall safety score.

**Methods:**
*   `summary()`: Returns a summary of the results.
*   `to_dict()`: Converts results to a dictionary.
*   `save(filepath)`: Saves results to a file.
*   `load(filepath)`: Loads results from a file.
*   `plot(save_path)`: Generates a plot of the results.

#### `RepeatedExperimentResults`

Aggregates results from multiple runs of the same experiment, enabling stability analysis.

**Methods:**
*   `keys()`: Returns the model names.
*   `values()`: Returns the results for each model.
*   `items()`: Returns (model_name, results) pairs.
*   `stability(model_name)`: Returns a `ModelStabilityReport` for the specified model.
*   `summary()`: Returns a summary of the repeated results.
*   `to_dict()`: Converts results to a dictionary.
*   `save(filepath)`: Saves results to a file.
*   `load(filepath)`: Loads results from a file.

**Example: Analyzing Stability**

```python
from simpleaudit.repeated_results import RepeatedExperimentResults

# Assuming 'repeated_results' is an instance of RepeatedExperimentResults
stability_report = repeated_results.stability("llama3.2:3b")
print(stability_report.summary())

# Save the repeated results
repeated_results.save("./results/repeated_audit.json")
```

### Visualization

The `visualization.server` module provides a web interface for viewing audit results.

**Functions:**
*   `start_server(results_dir, host, port)`: Starts the visualization server.
*   `root()`: Handles the root route.
*   `get_files()`: Lists available result files.
*   `get_json_file(file_path)`: Retrieves a specific JSON file.
*   `is_valid_audit_json(file_path)`: Validates if a file is a valid audit JSON.

**Example: Starting the Server**

```python
from simpleaudit.visualization.server import start_server

start_server(
    results_dir="./results",
    host="localhost",
    port=8050
)
```

### Scenario Packs

Scenarios are organized into packs, which can be retrieved using the `get_scenarios` function.

**Available Packs:**
*   `safety`: General safety scenarios.
*   `bullshitbench`: Scenarios for detecting hallucinations and factual errors.
*   `health`: Health-related scenarios.
*   `hei_refusal`: Scenarios for evaluating refusal behavior.
*   `rag`: Retrieval-Augmented Generation scenarios.

**Example: Listing Scenario Packs**

```python
from simpleaudit.scenarios import list_scenario_packs

packs = list_scenario_packs()
print(packs)
```

### Judge Configurations

Judges are configured using predefined configurations. The `get_judge` function retrieves a specific judge configuration.

**Available Judges:**
*   `safety`: Evaluates AI safety.
*   `abstention`: Evaluates refusal/abstention behavior.
*   `factuality`: Evaluates factual accuracy.
*   `helpfulness`: Evaluates helpfulness of responses.

**Example: Listing Judge Configs**

```python
from simpleaudit.judges import list_judge_configs

configs = list_judge_configs()
print(configs)
```

This architecture allows for flexible and extensible auditing of LLMs, with clear separation of concerns between execution, evaluation, and analysis.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Installation](installation.md)
*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
