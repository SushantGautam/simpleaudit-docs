## Architecture

`simpleaudit` is an LLM safety auditing framework designed to evaluate the robustness of large language models against specific safety scenarios. The architecture is built around a modular pipeline consisting of **Scenarios** (test cases), **Judges** (evaluation logic), and **Result Aggregation** (analysis and reporting).

The core execution engine is the `ModelAuditor` class, which orchestrates the interaction between the target model under test, the auditor (which generates or manages the conversation), and the judge (which evaluates the safety of the model's responses).

### Core Components

The library is structured into four primary modules:

1.  **`simpleaudit.model_auditor`**: Contains the `ModelAuditor` class, the primary interface for running audits. It handles API communication, conversation management, and retry logic.
2.  **`simpleaudit.scenarios`**: Defines `SCENARIO_PACKS`, a registry of predefined safety test cases (e.g., prompt injection, PII leakage).
3.  **`simpleaudit.judges`**: Provides `get_judge`, which retrieves judge configurations including prompts and response schemas.
4.  **`simpleaudit.results`**: Contains `AuditResult` and `AuditResults` dataclasses for storing, analyzing, and exporting audit outcomes.
5.  **`simpleaudit.experiment`**: Contains `AuditExperiment`, a higher-level orchestrator for running audits across multiple models or repetitions with caching support.

### The Audit Pipeline

The audit process follows a three-stage pipeline for each scenario:

#### 1. Scenario Execution
The `ModelAuditor` initiates a conversation with the **Target Model**.
-   **Input**: A scenario definition (from `SCENARIO_PACKS`) containing a prompt or conversation history.
-   **Process**: The auditor sends the scenario to the target model via the `AnyLLM` client. It supports multi-turn conversations (up to `max_turns`).
-   **File Handling**: If a scenario includes `file_uri` markers, the `_expand_files` helper converts these into OpenAI-compatible content blocks (e.g., image data URIs) before sending them to the provider. This allows auditing multimodal capabilities without mutating the stored conversation text.

#### 2. Judging
Once the target model responds, the **Judge Model** evaluates the safety of the interaction.
-   **Configuration**: Judges are configured via the `judge` parameter in `ModelAuditor`. This can be a named configuration (e.g., `"factuality"`) or custom prompts.
-   **Evaluation**: The judge receives the conversation transcript and a specific `judge_prompt`. It is instructed to return a structured JSON response.
-   **Response Schema**: By default, the judge returns a JSON object with:
    -   `severity`: One of `critical`, `high`, `medium`, `low`, or `pass`.
    -   `issues_found`: List of specific safety violations.
    -   `positive_behaviors`: List of safe behaviors observed.
    -   `summary`: A textual summary of the judgment.
    -   `recommendations`: Suggestions for improvement.
-   **Thinking Stripping**: The `strip_thinking` static method removes `` blocks from model outputs, ensuring that chain-of-thought reasoning does not interfere with the final judgment or stored results.

#### 3. Result Aggregation
The `ModelAuditor` parses the judge's response and constructs an `AuditResult` object.
-   **Parsing**: The `parse_json_response` utility extracts the JSON payload from the judge's text output. If parsing fails, the result is marked with `severity: "ERROR"`.
-   **Storage**: Each `AuditResult` contains the full conversation, the judge's verdict, and token usage metrics for the auditor, judge, and target models.

### Key Classes

#### `ModelAuditor`
The primary class for executing audits.

**Initialization Parameters:**
-   `model` (str): The name of the target model to audit.
-   `provider` (str): The provider for the target model (e.g., `"openai"`, `"anthropic"`, `"ollama"`).
-   `judge_model` (str): The model used for judging.
-   `judge_provider` (str): The provider for the judge model.
-   `judge` (str, optional): A named judge configuration key.
-   `probe_prompt` (str, optional): Custom prompt for the auditor.
-   `judge_prompt` (str, optional): Custom prompt for the judge.
-   `max_turns` (int): Maximum number of conversation turns (default: 5).
-   `max_retries` (int): Number of retries for API failures (default: 2).

**Key Methods:**
-   `run_async(scenarios, ...)`: Executes the audit pipeline asynchronously.
-   `get_scenarios(pack_name)`: Retrieves a list of scenario dictionaries from a named pack.

#### `AuditExperiment`
A wrapper for running complex experiments, such as comparing multiple models or repeating audits to measure variance.

**Features:**
-   **Multi-Model Support**: Accepts a list of model configurations.
-   **Caching**: Automatically caches results to disk (`save_dir`) to allow resuming interrupted experiments. It uses a configuration fingerprint to ensure cached results are only reused if the configuration (judge, scenarios, etc.) has not changed.
-   **Repetitions**: Supports `n_repetitions` to run the same audit multiple times for statistical analysis.

#### `AuditResults`
A collection of `AuditResult` objects with analysis capabilities.

**Properties:**
-   `score`: A safety score (0-100) calculated from the severity of completed scenarios. `ERROR` results are excluded from the score calculation to prevent infrastructure failures from skewing safety metrics.
-   `severity_distribution`: A dictionary mapping severity levels to their counts.
-   `token_usage`: Aggregated token counts for auditor, judge, and target models.

**Methods:**
-   `summary()`: Prints a formatted console summary of the audit results.
-   `save(filepath)`: Saves results to a JSON file atomically.
-   `load(filepath)`: Loads results from a JSON file.
-   `plot(save_path)`: Generates a visualization of the results using `matplotlib`.

### Example Usage

```python
from simpleaudit.model_auditor import ModelAuditor
from simpleaudit.results import AuditResults

# Initialize the auditor
auditor = ModelAuditor(
    model="gpt-4o",
    provider="openai",
    judge_model="gpt-4o",
    judge_provider="openai",
    judge="safety",  # Use a predefined safety judge config
    max_turns=3
)

# Get scenarios from a pack
scenarios = auditor.get_scenarios("prompt_injection")

# Run the audit
results = auditor.run_async(scenarios)

# Analyze results
results.summary()
print(f"Safety Score: {results.score}")
results.save("audit_results.json")
```

### Error Handling and Resilience

-   **Retries**: The `_call_async` method implements exponential backoff for API calls. Transient network errors or rate limits will trigger retries up to `max_retries` times.
-   **Atomic Writes**: Result files are written using a temporary file and atomic rename (`os.replace`) to prevent corruption if the process is interrupted.
-   **Cache Validation**: `AuditExperiment` validates cached results against a SHA-256 fingerprint of the configuration. If the configuration changes, stale caches are ignored, and the audit is re-run.

### Configuration

Judges can be customized via:
1.  **Named Configs**: Using the `judge` parameter to load predefined prompts and schemas from `simpleaudit.judges`.
2.  **Custom Prompts**: Passing `probe_prompt` and `judge_prompt` directly to `ModelAuditor`.
3.  **Response Schema**: The `judge_response_schema` parameter allows defining a custom JSON schema for the judge's output, enabling binary classification or other non-standard formats.

This architecture ensures that `simpleaudit` is flexible enough to test a wide range of models and safety criteria while providing robust tools for analyzing and reporting the results.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Quickstart](quickstart.md)
