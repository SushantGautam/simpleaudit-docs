## Quickstart

This guide demonstrates how to run your first local audit using the `simpleaudit-docs` library. You will learn how to initialize the `ModelAuditor` class, select scenario packs, and interpret the resulting `AuditResults`.

### Prerequisites

SimpleAudit requires Python 3.11 or higher. Ensure you have the library installed:

```bash
pip install simpleaudit
```

For local model testing without API keys, you can use Ollama. Ensure `ollama serve` is running and that you have pulled the necessary models (e.g., `llama3.2:3b` for the target and `gemma3:latest` for the judge).

### 1. Selecting Scenarios

Audits are driven by scenario packs. You can list available packs and retrieve specific scenarios using the `scenarios` module.

```python
from simpleaudit import get_scenarios, list_scenario_packs

# List all available scenario packs
print(list_scenario_packs())

# Retrieve a specific pack (e.g., 'safety' or 'bullshitbench')
# get_scenarios returns a list of scenario dictionaries
scenarios = get_scenarios("safety")
print(f"Loaded {len(scenarios)} scenarios from 'safety' pack")
```

### 2. Initializing the ModelAuditor

The core class for running audits is `ModelAuditor`. It requires the target model configuration and the judge model configuration.

**Constructor Parameters:**
*   `model` (str): The name of the target model to audit.
*   `provider` (str): The provider for the target model (e.g., `"ollama"`, `"openai"`).
*   `judge_model` (str): The name of the model used to evaluate the target's responses.
*   `judge_provider` (str): The provider for the judge model.
*   `api_key` (Optional[str]): API key for the target model (if required).
*   `base_url` (Optional[str]): Base URL for the target model API.
*   `judge_api_key` (Optional[str]): API key for the judge model.
*   `judge_base_url` (Optional[str]): Base URL for the judge model API.
*   `json_format` (bool): Whether to enforce JSON output format for the judge (default `True`).
*   `max_turns` (int): Maximum number of conversational turns per scenario (default `5`).
*   `verbose` (bool): Enable verbose logging (default `False`).
*   `show_progress` (bool): Show progress bars (default `True`).

#### Example: Local Ollama Audit

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define models
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL = "gemma3:latest"

# Load scenarios
scenarios = get_scenarios("safety")[:5]  # Limit to 5 scenarios for quick start

# Initialize the auditor
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    json_format=False,  # Ollama may not support strict JSON mode in all versions
    max_turns=3,
    verbose=True
)
```

### 3. Running the Audit

Once initialized, you can run the audit synchronously using the `run` method.

**Method Signature:**
`run(scenarios, max_turns, language, max_workers)`

*   `scenarios`: List of scenario dictionaries.
*   `max_turns`: Overrides the constructor's `max_turns` if provided.
*   `language`: Language code for the audit (e.g., `"en"`, `"no"`).
*   `max_workers`: Number of concurrent workers for async execution.

```python
# Run the audit
results = auditor.run(
    scenarios=scenarios,
    max_turns=3,
    language="en",
    max_workers=2
)
```

### 4. Interpreting Results

The `run` method returns an `AuditResults` object. This object provides access to individual results, summaries, and token usage statistics.

**Key Properties of `AuditResults`:**
*   `score`: The overall safety score.
*   `passed`: Number of scenarios that passed.
*   `failed`: Number of scenarios that failed.
*   `critical_count`: Number of critical issues found.
*   `severity_distribution`: A dictionary mapping severity levels to counts.
*   `token_usage`: A dictionary containing input/output token counts for auditor, judge, and target models.

**Key Methods:**
*   `summary()`: Returns a human-readable summary of the audit.
*   `to_dict()`: Converts the results to a dictionary.
*   `save(filepath)`: Saves the results to a JSON file.
*   `plot(save_path)`: Generates a plot of the results.

```python
# Print a summary
print(results.summary())

# Access specific metrics
print(f"Score: {results.score}")
print(f"Passed: {results.passed}, Failed: {results.failed}")
print(f"Critical Issues: {results.critical_count}")

# Save results to a file
results.save("audit_results.json")

# Generate a plot
results.plot("audit_plot.png")
```

### 5. Using AuditExperiment for Multi-Model Audits

If you need to audit multiple models simultaneously or run repeated experiments for stability analysis, use the `AuditExperiment` class.

**Constructor Parameters:**
*   `models` (List[Dict[str, Any]]): A list of dictionaries, each defining a model configuration (keys: `model`, `provider`, `api_key`, `base_url`).
*   `judge_model` (Optional[str]): The judge model name.
*   `judge_provider` (Optional[str]): The judge provider.
*   `n_repetitions` (int): Number of times to repeat the experiment (default `1`).
*   `save_dir` (Optional[str]): Directory to save results.

```python
from simpleaudit import AuditExperiment

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

experiment = AuditExperiment(
    models=models,
    judge_model="gemma3:latest",
    judge_provider="ollama",
    n_repetitions=2,
    save_dir="./experiment_results"
)

# Run the experiment
repeated_results = experiment.run(
    scenarios=get_scenarios("safety"),
    max_turns=3,
    language="en",
    max_workers=2
)

# Access stability report
stability_report = repeated_results.stability("llama3.2:3b")
print(stability_report.summary())
```

### CLI Usage

SimpleAudit also provides a CLI entry point via `cli.main()`. While the specific CLI flags are not detailed in the API surface provided, you can invoke the library's command-line interface if installed as a package. For programmatic control, the Python API described above is recommended.

### Best Practices

1.  **Start Small**: Use `[:5]` slicing on your scenario list to quickly verify your setup before running full packs.
2.  **Judge Selection**: Choose a judge model that is capable of understanding the context of your scenarios. For safety audits, models with strong instruction-following capabilities are preferred.
3.  **Token Monitoring**: Monitor `results.token_usage` to estimate costs if using API-hosted models.
4.  **Reproducibility**: Use `save_dir` in `AuditExperiment` to automatically archive results for each run.

By following these steps, you can perform comprehensive safety audits on local or API-hosted LLMs using SimpleAudit.

### See Also

*   [Installation](installation.md)
*   [Key Ideas](key-ideas.md)
*   [Architecture](architecture.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
