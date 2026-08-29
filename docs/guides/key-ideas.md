## Key Ideas

SimpleAudit is a lightweight, local-first framework for auditing and red-teaming AI systems. Its core philosophy is **adversarial probing**: instead of static test cases, the framework uses an LLM "auditor" to dynamically generate follow-up questions based on the target model's responses, simulating a realistic user trying to probe for safety issues, hallucinations, or factual errors.

The architecture separates concerns into three distinct roles:
1.  **Target Model**: The AI system being audited.
2.  **Auditor (Probe) Model**: Generates adversarial or probing questions.
3.  **Judge Model**: Evaluates the target's responses against specific criteria (safety, factuality, helpfulness, etc.) and assigns a severity or score.

### Core Components

#### 1. `AuditExperiment`
The high-level entry point for running multi-model audits. It manages the lifecycle of experiments, handles concurrency, and aggregates results.

**Constructor:**
```python
AuditExperiment(
    models: List[Dict[str, Any]],
    judge_model: Optional[str] = None,
    judge_base_url: Optional[str] = None,
    judge_api_key: Optional[str] = None,
    judge_provider: Optional[str] = None,
    auditor_model: Optional[str] = None,
    auditor_provider: Optional[str] = None,
    auditor_api_key: Optional[str] = None,
    auditor_base_url: Optional[str] = None,
    judge: Optional[str] = None,
    probe_prompt: Optional[str] = None,
    judge_prompt: Optional[str] = None,
    json_format: bool = True,
    verbose: bool = False,
    show_progress: bool = True,
    n_repetitions: int = 1,
    save_dir: Optional[str] = None,
    on_model_done: Optional[Callable[[str, 'RepeatedExperimentResults'], None]] = None
)
```

**Key Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Executes the audit synchronously.
*   `run_async(scenarios, max_turns, language, max_workers)`: Executes the audit asynchronously.

#### 2. `ModelAuditor`
The low-level engine that executes a single scenario against a specific target model. It handles the turn-by-turn conversation loop, stripping thinking tags, and parsing judge responses.

**Constructor:**
```python
ModelAuditor(
    model: str,
    provider: str,
    judge_model: str,
    judge_provider: str,
    api_key: Optional[str] = None,
    base_url: Optional[str] = None,
    system_prompt: Optional[str] = None,
    judge_api_key: Optional[str] = None,
    judge_base_url: Optional[str] = None,
    auditor_model: Optional[str] = None,
    auditor_provider: Optional[str] = None,
    auditor_api_key: Optional[str] = None,
    auditor_base_url: Optional[str] = None,
    judge: Optional[str] = None,
    probe_prompt: Optional[str] = None,
    judge_prompt: Optional[str] = None,
    judge_response_schema: Optional[Dict[str, Any]] = None,
    json_format: bool = True,
    max_turns: int = 5,
    verbose: bool = False,
    show_progress: bool = True
)
```

**Key Methods:**
*   `run_scenario(name, description, expected_behavior, test_prompt, max_turns, language, pbar_audit, pbar_judge, max_workers)`: Runs a single scenario.
*   `strip_thinking(text)`: Removes internal reasoning tags (e.g., `<thinking>`) from model outputs.
*   `parse_json_response(response, default_severity)`: Robustly parses JSON from judge outputs, handling markdown code blocks and surrounding text.

#### 3. Results and Stability
*   **`AuditResults`**: A container for individual scenario results. It exposes properties like `score`, `passed`, `failed`, `critical_count`, and `severity_distribution`. It supports serialization via `save(filepath)` and `load(filepath)`.
*   **`RepeatedExperimentResults`**: Used when `n_repetitions > 1`. It aggregates multiple runs of the same experiment to calculate stability metrics.
*   **`ModelStabilityReport`**: Generated via `RepeatedExperimentResults.stability(model_name)`. It provides `summary()` and `to_dict()` for analyzing variance in scores across repetitions.

### Judge Configurations

SimpleAudit ships with several pre-configured judges, each tailored to a specific evaluation dimension. These are defined in `simpleaudit/judges/` and can be selected via the `judge` parameter in `AuditExperiment` or `ModelAuditor`.

| Judge Name | Focus | Key Output Fields |
| :--- | :--- | :--- |
| **Safety** | Constitutional AI principles (harm avoidance, boundaries) | `severity`, `issues_found`, `recommendations` |
| **Factuality** | Hallucination detection (G-Eval based) | `score` (1-10), `verdict`, `factual_errors` |
| **Helpfulness** | Response quality (MT-Bench based) | `score` (1-10), `relevance`, `accuracy`, `clarity` |
| **Abstention** | Correct refusal of unanswerable questions | `score` (1-10), `abstained`, `appropriate`, `category` |
| **Harm** | Categorization of harmful content (HELM Safety) | `harmful`, `category`, `severity`, `explanation` |

### Scenario Packs

Scenarios are organized into packs, each targeting specific domains or failure modes. You can load them using `get_scenarios(pack_name)`.

*   **`safety`**: General AI safety (hallucination, manipulation, privacy).
*   **`health`**: Healthcare-specific boundaries (diagnosis, prescription, emergency response).
*   **`rag`**: Retrieval-Augmented Generation issues (source misattribution, context confusion).
*   **`system_prompt`**: Testing system prompt adherence and override attempts.
*   **`bullshitbench`**: Specifically designed for detecting "bullshit" or nonsensical responses.

### Example: Running a Local Audit

The following example demonstrates how to run a basic safety audit using local Ollama models. Note that `json_format` is set to `False` for Ollama compatibility in this specific example, though it defaults to `True`.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define models
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL = "gemma3:latest"

# Load scenarios from a pack
scenarios = get_scenarios("safety")[:3]  # Take first 3 for a quick test

# Initialize the auditor
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    json_format=False,  # Ollama may not support strict JSON response format
    max_turns=3
)

# Run the scenarios
results = auditor.run(scenarios, max_turns=3, language="english")

# Inspect results
print(f"Score: {results.score}")
print(f"Critical Issues: {results.critical_count}")
for issue in results.all_issues:
    print(f"- {issue}")
```

### Custom Judges

You can override the default judge behavior by providing custom `probe_prompt` and `judge_prompt` strings. This allows you to define exactly what the auditor looks for and what schema the judge must return.

```python
from simpleaudit import AuditExperiment

custom_probe = "You are a tester. Ask a question about Python history."
custom_judge = "Evaluate the response for accuracy. Return JSON: {\"score\": 1-10, \"reason\": \"...\"}"

experiment = AuditExperiment(
    models=[{"model": "llama3.2:3b", "provider": "ollama"}],
    judge_model="gemma3:latest",
    judge_provider="ollama",
    probe_prompt=custom_probe,
    judge_prompt=custom_judge,
    json_format=False
)

results = experiment.run(scenarios=["Test Scenario"], max_turns=2)
```

### Best Practices

1.  **Local First**: SimpleAudit is designed to run locally. Use providers like `ollama` or `vllm` to avoid API costs and data transmission.
2.  **JSON Format**: If your provider supports structured output (e.g., OpenAI, Anthropic), keep `json_format=True` for more reliable parsing. For Ollama, you may need to set it to `False` and rely on the robust `parse_json_response` method.
3.  **Repetitions**: For stability analysis, set `n_repetitions > 1` in `AuditExperiment`. Use `RepeatedExperimentResults.stability()` to identify flaky scenarios.
4.  **Scenario Specificity**: Use `expected_behavior` in custom scenarios to guide the judge on what a "pass" looks like, especially for non-safety audits like helpfulness.

### See Also

*   [Architecture](architecture.md)
*   [Quickstart](quickstart.md)
*   [Installation](installation.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
