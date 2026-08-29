## Architecture

SimpleAudit is a lightweight, local-first framework for auditing and red-teaming Large Language Models (LLMs). The architecture is designed to decouple the **auditing logic** (probe generation and evaluation) from the **target model** being tested, allowing for flexible, multilingual, and multi-model comparisons.

The system operates on a three-role model:
1.  **Target Model**: The LLM under audit.
2.  **Auditor Model**: Generates adversarial probes (red-team inputs) to test the target.
3.  **Judge Model**: Evaluates the conversation between the Auditor and Target, assigning a severity score and generating a report.

### Core Components

The library is structured around four primary modules: `ModelAuditor`, `AuditExperiment`, `Results`, and `RepeatedResults`.

#### 1. ModelAuditor
The `ModelAuditor` class (`simpleaudit.model_auditor.ModelAuditor`) is the atomic unit of execution. It manages the lifecycle of a single audit session against a specific target model.

**Key Responsibilities:**
*   **Client Management**: Instantiates `AnyLLM` clients for the target, judge, and auditor models. It supports various providers (OpenAI, Anthropic, Ollama, vLLM) via the `provider` parameter.
*   **Probe Generation**: Uses the Auditor model to generate context-aware probes. If a `test_prompt` is provided in the scenario, it is used verbatim for the first turn; otherwise, the Auditor generates a probe based on the scenario description.
*   **Judgment**: Sends the conversation history to the Judge model. The Judge returns a structured JSON response containing severity, issues, and recommendations.
*   **Thinking Stripping**: Includes a utility `strip_thinking()` to remove `<think>` or `<thinking>` tags from model outputs, ensuring clean evaluation data.

**Constructor Parameters:**
*   `model`, `provider`: Target model configuration.
*   `judge_model`, `judge_provider`: Judge model configuration.
*   `auditor_model`, `auditor_provider`: Optional separate auditor configuration (defaults to judge config if not specified).
*   `judge`: Optional name of a built-in judge config (e.g., `"safety"`, `"factuality"`).
*   `probe_prompt`, `judge_prompt`: Custom system prompts to override default behaviors.
*   `json_format`: Whether to enforce JSON schema output from the judge (default `True`).
*   `max_turns`: Maximum number of back-and-forth turns per scenario.

```python
from simpleaudit import ModelAuditor

auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    max_turns=3,
    json_format=False  # Ollama may not support strict JSON schema
)
```

#### 2. AuditExperiment
The `AuditExperiment` class (`simpleaudit.experiment.AuditExperiment`) orchestrates audits across **multiple models** and supports **repeated runs** for statistical stability analysis.

**Key Responsibilities:**
*   **Multi-Model Orchestration**: Accepts a list of model configurations. It merges common parameters (like `judge_model`) from the experiment level into individual model configurations.
*   **Repetitions**: Supports `n_repetitions` to run the same scenario set multiple times, enabling stability metrics.
*   **Resumability**: If `save_dir` is provided, it checks for existing `run_N.json` files and skips completed runs, allowing long experiments to be resumed after interruption.
*   **Callbacks**: Supports an `on_model_done` callback for real-time processing of results as each model completes.

**Constructor Parameters:**
*   `models`: List of dictionaries, each containing at least a `"model"` key. Optional `"label"` for display.
*   `judge_model`, `judge_provider`: Default judge settings applied to all models unless overridden.
*   `n_repetitions`: Number of times to run the full scenario set per model (default `1`).
*   `save_dir`: Directory to save intermediate results for resumability.

```python
from simpleaudit.experiment import AuditExperiment

experiment = AuditExperiment(
    models=[
        {"model": "llama3.2:3b", "provider": "ollama", "label": "Llama 3.2"},
        {"model": "gemma3:latest", "provider": "ollama", "label": "Gemma 3"}
    ],
    judge_model="gemma3:latest",
    judge_provider="ollama",
    n_repetitions=3,
    save_dir="./audit_results"
)

# Run the experiment
results = experiment.run(scenarios="safety", max_turns=2)
```

#### 3. Results & Repeated Results
The library provides two levels of result aggregation:

**AuditResults** (`simpleaudit.results.AuditResults`):
*   Aggregates `AuditResult` objects from a single run.
*   Provides properties for `score`, `severity_distribution`, `token_usage`, and `passed`/`failed` counts.
*   Methods: `summary()` (prints human-readable report), `save(filepath)`, `load(filepath)`, `plot(save_path)`.

**RepeatedExperimentResults** (`simpleaudit.repeated_results.RepeatedExperimentResults`):
*   Aggregates `AuditResults` across multiple repetitions for each model.
*   Implements a dict-like interface (`__getitem__`, `keys`, `values`) for backward compatibility, returning the *first* run's results for a given model label.
*   **Stability Analysis**: The `stability(model_name)` method returns a `ModelStabilityReport` containing:
    *   `mean_score`, `std_score`, `min_score`, `max_score`.
    *   `cv` (Coefficient of Variation).
    *   `per_scenario`: A dictionary of `ScenarioStats` including `pass_rate` and `agreement_rate` (how often the most common severity occurred).

```python
# Access stability for a specific model
stability_report = results.stability("Llama 3.2")
stability_report.summary()

# Access first run results for a model (backward compatible)
first_run_results = results["Llama 3.2"]
print(first_run_results.score)
```

### Data Flow

1.  **Scenario Selection**: Scenarios are loaded from built-in packs (e.g., `"safety"`, `"bullshitbench"`) via `get_scenarios()` or provided as a custom list of dictionaries.
2.  **Probe Generation**: For each turn, the Auditor model generates a probe. If `test_prompt` is present, it is used on turn 1.
3.  **Target Interaction**: The probe is sent to the Target model. The response is appended to the conversation history.
4.  **Judgment**: After `max_turns`, the full conversation is sent to the Judge model.
5.  **Parsing**: The Judge's response is parsed into JSON. If parsing fails, a default severity (usually `"ERROR"`) is assigned.
6.  **Aggregation**: Results are stored in `AuditResult` objects, aggregated into `AuditResults`, and finally into `RepeatedExperimentResults` if repetitions were configured.

### Judge Configurations

SimpleAudit includes several built-in judge configurations accessible via the `judge` parameter in `ModelAuditor` or `AuditExperiment`. These are defined in `simpleaudit.judges`:

*   `safety`: Constitutional AI safety evaluation.
*   `abstention`: Refusal/abstention appropriateness.
*   `helpfulness`: Response quality (MT-Bench dimensions).
*   `factuality`: Hallucination and factual error detection.
*   `harm`: HELM Safety harm categorization.
*   `binary_abstention`: Binary classifier for abstention.

You can list available judges using `list_judge_configs()` from `simpleaudit.judges`.

### Error Handling & Resilience

*   **JSON Parsing**: The `parse_json_response` utility handles malformed JSON from LLMs, extracting JSON from code blocks or surrounding text. If parsing fails completely, it returns a default structure with `severity: "ERROR"`.
*   **Resumable Experiments**: By setting `save_dir` in `AuditExperiment`, each repetition's results are saved to `run_N.json`. If the experiment is interrupted, re-running it will load existing files and skip those runs, ensuring no wasted API calls.

### Local-First Design

SimpleAudit is designed to work without external APIs. By using `provider="ollama"` and local model names, the entire audit pipeline (Target, Auditor, Judge) can run locally. The `json_format` parameter should be set to `False` for providers like Ollama that may not support strict `response_format` schemas, relying instead on prompt engineering for JSON output.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Installation](installation.md)
*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
