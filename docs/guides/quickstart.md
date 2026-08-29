## Quickstart

SimpleAudit is a lightweight, local-first framework for AI safety auditing. It uses adversarial probing to test LLMs against specific safety scenarios and evaluates the responses using a judge model. The framework supports both local models (via Ollama or Hugging Face) and API-hosted models (OpenAI, Anthropic, etc.).

This guide demonstrates how to run your first audit using `ModelAuditor` with a local Ollama setup, and how to use `AuditExperiment` for multi-model comparisons.

### Prerequisites

1.  **Install SimpleAudit**:
    ```bash
    pip install simpleaudit
    ```
2.  **Install Ollama** (for local model testing):
    *   Download from [ollama.ai](https://ollama.ai).
    *   Start the server: `ollama serve`
    *   Pull a target model (e.g., `gemma3` or `llama3.2`): `ollama pull gemma3`
    *   Pull a judge model (can be the same as target): `ollama pull gemma3`

### Basic Audit with ModelAuditor

The `ModelAuditor` class is the primary entry point for auditing a single model. It manages the conversation loop between the target model, the auditor (prober), and the judge.

#### 1. Initialize the Auditor

Create a `ModelAuditor` instance by specifying the target model and the judge model. Both can run locally via Ollama.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Configuration
TARGET_MODEL = "gemma3"      # Model to test
JUDGE_MODEL  = "gemma3"      # Model to judge responses

# Initialize the auditor
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    system_prompt="You are a helpful AI assistant. Be concise and accurate.",
    max_turns=3,             # Number of back-and-forth turns per scenario
    verbose=True,             # Print progress to console
)
```

#### 2. Select Scenarios

SimpleAudit includes built-in scenario packs. You can list available packs and retrieve specific scenarios.

```python
# List available scenario packs
from simpleaudit import list_scenario_packs
print(list_scenario_packs())

# Get the first 3 scenarios from the "safety" pack
scenarios = get_scenarios("safety")[:3]
```

#### 3. Run the Audit

Execute the audit synchronously using the `run` method. This returns an `AuditResults` object.

```python
# Run the audit
results = auditor.run(scenarios, max_turns=3)

# Display a summary of the results
results.summary()

# Save results to a JSON file
results.save("local_audit_results.json")
```

#### 4. Visualize Results (Optional)

If `matplotlib` is installed, you can generate a chart of the severity distribution.

```python
try:
    results.plot(save_path="local_audit_chart.png")
    print("Chart saved to: local_audit_chart.png")
except ImportError:
    print("Install matplotlib for charts: pip install simpleaudit[plot]")
```

### Multi-Model Comparison with AuditExperiment

For comparing multiple models or running repeated experiments to assess stability, use `AuditExperiment`. This class manages a list of models and runs the audit for each, optionally repeating the experiment `n_repetitions` times.

#### 1. Define Models

Define a list of model configurations. Each model is a dictionary with `model`, `provider`, and optionally `label`.

```python
from simpleaudit.experiment import AuditExperiment

models = [
    {
        "model": "gemma3",
        "provider": "ollama",
        "label": "gemma3-local"
    },
    {
        "model": "llama3.2",
        "provider": "ollama",
        "label": "llama3.2-local"
    }
]
```

#### 2. Initialize the Experiment

Create an `AuditExperiment` instance. Specify the judge model configuration separately.

```python
experiment = AuditExperiment(
    models=models,
    judge_model="gemma3",
    judge_provider="ollama",
    n_repetitions=1,       # Run each model once
    verbose=True,
    show_progress=True
)
```

#### 3. Run the Experiment

Use `run_async` for asynchronous execution (recommended for multiple models) or `run` for synchronous execution.

```python
import asyncio

async def run_experiment():
    # Get scenarios
    scenarios = get_scenarios("safety")[:5]
    
    # Run the experiment
    results = await experiment.run_async(
        scenarios=scenarios,
        max_turns=3
    )
    
    # Access results for a specific model by label
    gemma_results = results["gemma3-local"]
    gemma_results.summary()
    
    # Get stability report if n_repetitions > 1
    if experiment.n_repetitions > 1:
        stability = results.stability("gemma3-local")
        stability.summary()

# Execute
asyncio.run(run_experiment())
```

### Key Concepts

*   **Target Model**: The LLM being audited.
*   **Judge Model**: The LLM that evaluates the target's responses for safety issues. It outputs a severity level (`critical`, `high`, `medium`, `low`, `pass`) and a summary.
*   **Scenarios**: Predefined test cases (prompts) that probe specific safety vulnerabilities (e.g., harmful instructions, PII extraction).
*   **Max Turns**: The maximum number of conversation turns allowed per scenario. The auditor may ask follow-up questions to probe deeper.

### Saving and Loading Results

`AuditResults` objects can be saved to and loaded from JSON files, allowing you to analyze results later without re-running the audit.

```python
# Save
results.save("audit_output.json")

# Load
from simpleaudit.results import AuditResults
loaded_results = AuditResults.load("audit_output.json")
loaded_results.summary()
```

### Troubleshooting

*   **Ollama not found**: Ensure `ollama serve` is running and the model is pulled (`ollama list`).
*   **Timeouts**: Local models can be slow. Increase `max_turns` cautiously or use smaller models for initial testing.
*   **JSON Format**: Some providers (like Ollama) may not support strict JSON response formats. If you encounter issues, set `json_format=False` in the `ModelAuditor` or `AuditExperiment` constructor.

### See Also

*   [Installation](installation.md)
*   [Key Ideas](key-ideas.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
