## Architecture

SimpleAudit is a lightweight, local-first framework for auditing and red-teaming Large Language Models (LLMs). It operates by simulating adversarial user interactions (probing) and evaluating the model's responses against specific safety or quality criteria (judging). The architecture is designed to be provider-agnostic, supporting local inference engines (Ollama, vLLM) and API-based providers (OpenAI, Anthropic, etc.) through the `any_llm` abstraction layer.

The system is composed of three primary layers:
1.  **Experiment Orchestration**: `AuditExperiment` manages multi-model comparisons and repeated runs for statistical stability.
2.  **Core Auditing**: `ModelAuditor` executes the actual probing and judging logic for a single model.
3.  **Results & Analysis**: `AuditResults` and `RepeatedExperimentResults` handle data storage, serialization, and statistical reporting.

### Core Components

#### 1. `AuditExperiment`
The top-level orchestrator for running audits across multiple models. It handles the setup of judge and auditor configurations, manages concurrency, and supports resumable experiments via disk caching.

**Location:** `simpleaudit/experiment.py`

**Key Parameters:**
*   `models`: A list of dictionaries. Each dict must contain a `"model"` key. Optional keys include `"label"`, `"provider"`, `"api_key"`, `"base_url"`, and model-specific overrides for the judge/auditor.
*   `judge_model`, `judge_provider`, `judge_api_key`, `judge_base_url`: Default configuration for the judge model. These are merged into each model's configuration unless overridden at the model level.
*   `auditor_model`, `auditor_provider`, `auditor_api_key`, `auditor_base_url`: Default configuration for the auditor (probe generator). If not specified, the judge configuration is used.
*   `judge`: Optional name of a built-in judge configuration (e.g., `"safety"`, `"helpfulness"`).
*   `probe_prompt`, `judge_prompt`: Optional custom system prompts for the auditor and judge, respectively.
*   `n_repetitions`: Number of times to run the experiment for each model (default: 1).
*   `save_dir`: Optional directory path to save results incrementally. Enables resuming interrupted experiments.

**Usage:**
```python
from simpleaudit import AuditExperiment

experiment = AuditExperiment(
    models=[
        {"model": "llama3.2:3b", "provider": "ollama", "label": "Llama 3.2"},
        {"model": "gemma3:latest", "provider": "ollama", "label": "Gemma 3"}
    ],
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    n_repetitions=3,
    save_dir="./audit_results"
)

# Run with a built-in scenario pack
results = experiment.run("safety", max_turns=5, language="English")
```

#### 2. `ModelAuditor`
The engine that executes a single audit run. It manages the conversation loop, generating probes using an auditor model and evaluating responses using a judge model.

**Location:** `simpleaudit/model_auditor.py`

**Key Parameters:**
*   `model`, `provider`: The target model being audited.
*   `judge_model`, `judge_provider`: The model used to evaluate the target's responses.
*   `auditor_model`, `auditor_provider`: The model used to generate adversarial probes. Defaults to the judge model if not specified.
*   `system_prompt`: Optional system prompt applied to the target model.
*   `judge`: Name of a built-in judge config. If provided, it sets default `probe_prompt` and `judge_prompt` unless explicitly overridden.
*   `json_format`: Whether to enforce JSON schema output from the judge (default: `True`).
*   `max_turns`: Maximum number of back-and-forth turns per scenario (default: 5).

**Behavior:**
*   **Probe Generation**: On the first turn, if a `test_prompt` is defined in the scenario, it is used verbatim. Subsequent turns use the auditor model to generate the next probe based on the conversation history.
*   **Judging**: The judge model evaluates the full conversation history. It returns a structured JSON response containing `severity`, `issues_found`, `positive_behaviors`, `summary`, and `recommendations`.
*   **Thinking Tag Stripping**: The `strip_thinking` static method removes `<think>` or `<thinking>` blocks from model outputs to ensure clean evaluation.

#### 3. Results Classes

**`AuditResults`** (`simpleaudit/results.py`)
Represents the outcome of a single experiment run.
*   **Properties**: `score` (0-100), `passed`, `failed`, `critical_count`, `severity_distribution`, `token_usage`.
*   **Methods**:
    *   `summary()`: Prints a human-readable summary to the console.
    *   `save(filepath)`: Serializes results to JSON.
    *   `load(filepath)`: Class method to load results from JSON.
    *   `plot(save_path)`: Generates a matplotlib visualization (pie chart of severity distribution and bar chart of scenario scores).

**`RepeatedExperimentResults`** (`simpleaudit/repeated_results.py`)
Represents the outcome of an `AuditExperiment` with `n_repetitions > 1`.
*   **Container Protocol**: Supports `len()`, `in`, `[]` indexing (returns the first run's `AuditResults`), and iteration over model labels.
*   **Methods**:
    *   `stability(model_name)`: Returns a `ModelStabilityReport` for a specific model, calculating mean score, standard deviation, coefficient of variation, and per-scenario pass rates.
    *   `summary()`: Prints stability reports for all models.
    *   `save(filepath)` / `load(filepath)`: Serializes/deserializes the full multi-run dataset.

### Data Flow

1.  **Initialization**: `AuditExperiment` validates model configurations and merges global judge/auditor settings with model-specific overrides.
2.  **Execution**:
    *   For each model, `ModelAuditor` is instantiated.
    *   For each scenario in the pack, `run_scenario` is called.
    *   **Turn 1**: The scenario's `test_prompt` (or a generated probe) is sent to the target model.
    *   **Subsequent Turns**: The auditor model generates a new probe based on the previous conversation. The target model responds.
    *   **Judgment**: After `max_turns`, the judge model evaluates the entire conversation.
3.  **Aggregation**:
    *   Individual `AuditResult` objects are collected into an `AuditResults` instance.
    *   If `n_repetitions > 1`, multiple `AuditResults` instances are grouped by model label into `RepeatedExperimentResults`.
4.  **Persistence**: If `save_dir` is set, each run is saved to disk immediately, allowing the experiment to resume if interrupted.

### Built-in Judges and Scenarios

**Judges** (`simpleaudit/judges/__init__.py`)
Pre-defined evaluation criteria accessible via the `judge` parameter:
*   `safety`: Constitutional AI safety evaluation.
*   `abstention`: Refusal/abstention appropriateness.
*   `helpfulness`: Response quality (relevance, accuracy, clarity, completeness).
*   `factuality`: Hallucination and factual error detection.
*   `harm`: HELM Safety harm categorization.
*   `binary_abstention`: Binary classifier for abstention.

**Scenario Packs** (`simpleaudit/scenarios/__init__.py`)
Collections of test scenarios accessible via `get_scenarios(pack_name)`:
*   `safety`, `rag`, `health`, `system_prompt`, `helpmed`, `ung`.
*   `bullshitbench`, `health_bullshit`, `epistemic_safety`.
*   `hei_refusal`, `nav_aap`, `skatteetaten`.
*   `all`: Union of all available packs.

### Error Handling and Resilience

*   **JSON Parsing**: The `parse_json_response` utility handles malformed JSON from LLMs, extracting valid JSON payloads from surrounding text and defaulting to an `"ERROR"` severity if parsing fails completely.
*   **Resumability**: `AuditExperiment` checks `save_dir` for existing `run_N.json` files before starting. If a run exists, it is loaded from disk and skipped, ensuring that interrupted experiments do not waste compute resources.
*   **Concurrency**: `max_workers` controls the number of concurrent scenarios processed per model, utilizing `asyncio.Semaphore` to limit parallelism.

### See Also

*   [Key Ideas](key-ideas.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
