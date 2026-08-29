## Architecture

SimpleAudit is a lightweight framework for validating LLM safety through comparative scoring. It operates without ground-truth labels by using a "judge" model to evaluate the responses of a "target" model against a set of adversarial scenarios. The architecture is divided into three core subsystems: the **Audit Pipeline** (orchestration and execution), the **Judge Registry** (evaluation criteria and prompts), and the **Scenario Structure** (test data).

### 1. The Audit Pipeline

The pipeline is responsible for managing the interaction between the target model, the auditor (prober), and the judge. It consists of two primary classes: `ModelAuditor` for single-model execution and `AuditExperiment` for multi-model, multi-run orchestration.

#### ModelAuditor
`ModelAuditor` executes a single audit session against a specific target model. It manages the conversation loop, where an "auditor" model generates probes (questions) to test the target, and a "judge" model evaluates the target's responses.

**Key Parameters:**
*   `model`: The target model name.
*   `provider`: The provider for the target model (e.g., `"ollama"`, `"openai"`).
*   `judge_model`: The model used for evaluation.
*   `judge_provider`: The provider for the judge model.
*   `judge`: Optional name of a built-in judge configuration (see Judge Registry).
*   `probe_prompt`: Custom system prompt for the auditor (overrides named judge defaults).
*   `judge_prompt`: Custom system prompt for the judge (overrides named judge defaults).
*   `json_format`: Boolean flag to enforce JSON response format (default `True`).
*   `max_turns`: Maximum number of conversation turns per scenario (default `5`).

**Execution Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Synchronous execution.
*   `run_async(scenarios, max_turns, language, max_workers)`: Asynchronous execution.

#### AuditExperiment
`AuditExperiment` orchestrates audits across multiple models and supports repeated runs for stability analysis. It merges common configuration (judge, prompts) into individual model configurations and handles caching for resumable experiments.

**Key Parameters:**
*   `models`: A list of dictionaries, each containing at least a `"model"` key. Optional keys include `"label"`, `"provider"`, and model-specific overrides.
*   `judge_model`, `judge_provider`: Default judge settings applied to all models if not overridden.
*   `n_repetitions`: Number of times to run the audit for each model (default `1`).
*   `save_dir`: Directory to save intermediate and final results for resuming.
*   `on_model_done`: Callback invoked after each model completes.

**Execution Methods:**
*   `run(scenarios, max_turns, language, max_workers)`: Synchronous wrapper.
*   `run_async(scenarios, max_turns, language, max_workers)`: Asynchronous orchestration.

### 2. Judge Registry

The judge registry provides pre-defined evaluation criteria. Judges define the "system prompt" for the judge model and the expected output schema.

**Available Judges:**
*   `safety`: Constitutional AI safety evaluation.
*   `abstention`: Refusal/abstention appropriateness.
*   `helpfulness`: Response quality (relevance, accuracy, clarity, completeness).
*   `factuality`: Hallucination and factual error detection.
*   `harm`: HELM Safety harm categorization.
*   `helsedir_sexhealth_no`: Norwegian sexual-health judge (generic).
*   `helsedir_sexhealth_no_rag`: Norwegian sexual-health judge (RAG framing).
*   `binary_abstention`: Binary classifier for abstention behavior.

**API Functions:**
*   `get_judge(name)`: Returns the configuration dictionary for a named judge.
*   `list_judge_configs()`: Returns a dictionary mapping judge names to their descriptions.

**Resolution Logic:**
When `ModelAuditor` is initialized:
1.  If `judge_prompt` is provided, it takes precedence.
2.  Else, if `judge` is provided, the corresponding `judge_prompt` and `probe_prompt` from the registry are loaded.
3.  Else, default safety prompts are used.
4.  `probe_prompt` can always override the probe prompt from a named judge.

### 3. Scenario Structure

Scenarios are defined as dictionaries within "packs." Each scenario contains:
*   `name`: Unique identifier.
*   `description`: Context for the scenario.
*   `test_prompt`: The initial user message to the target model.
*   `expected_behavior`: (Optional) List of expected behaviors for the judge to verify.

**API Functions:**
*   `get_scenarios(pack_name)`: Returns the list of scenario dictionaries for a pack.
*   `list_scenario_packs()`: Returns a dictionary mapping pack names to the number of scenarios in each.

**Available Packs:**
`safety`, `rag`, `health`, `system_prompt`, `helpmed`, `ung`, `bullshitbench_v1`, `bullshitbench_v2`, `bullshitbench`, `health_bullshit`, `epistemic_safety`, `hei_refusal`, `nav_aap`, `skatteetaten`, `all`.

### 4. Results and Analysis

Audit results are encapsulated in `AuditResult` (single scenario) and `AuditResults` (collection) classes.

**AuditResults Properties:**
*   `score`: A float (0-100) calculated from severity scores.
*   `severity_distribution`: Dictionary of severity counts.
*   `token_usage`: Dictionary of input/output tokens for auditor, judge, and target.
*   `passed` / `failed`: Counts of scenarios with `pass` severity vs. others.

**Analysis Methods:**
*   `summary()`: Prints a formatted console summary.
*   `plot(save_path)`: Generates a matplotlib visualization (requires `matplotlib`).
*   `save(filepath)` / `load(filepath)`: Serializes/deserializes results to/from JSON.

For repeated experiments, `RepeatedExperimentResults` provides:
*   `stability(model_name)`: Returns a `ModelStabilityReport` with mean score, standard deviation, and per-scenario pass rates.
*   `save(filepath)` / `load(filepath)`: Handles serialization of multi-run data.

### Code Examples

#### Basic Single-Model Audit

```python
from simpleaudit import ModelAuditor, get_scenarios

# Load a small subset of safety scenarios
scenarios = get_scenarios("safety")[:5]

# Initialize the auditor
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",  # Use built-in safety judge
    max_turns=3
)

# Run the audit
results = auditor.run(scenarios)

# Analyze
results.summary()
print(f"Score: {results.score}")
```

#### Multi-Model Experiment with Repetitions

```python
from simpleaudit.experiment import AuditExperiment

models = [
    {"model": "llama3.2:3b", "provider": "ollama", "label": "Llama 3.2 3B"},
    {"model": "gemma3:latest", "provider": "ollama", "label": "Gemma 3"},
]

experiment = AuditExperiment(
    models=models,
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="abstention",
    n_repetitions=3,  # Run 3 times for stability
    save_dir="./audit_runs"
)

# Run asynchronously
import asyncio
results = asyncio.run(experiment.run_async("safety"))

# Check stability for a specific model
stability = results.stability("Llama 3.2 3B")
stability.summary()
```

#### Custom Judge Prompts

```python
from simpleaudit import ModelAuditor

custom_judge_prompt = """
You are a strict factuality judge. 
Evaluate the response for factual errors.
Return JSON with keys: 'severity' (pass|low|medium|high|critical), 'errors' (list).
"""

auditor = ModelAuditor(
    model="target_model",
    provider="openai",
    judge_model="gpt-4o",
    judge_provider="openai",
    judge_prompt=custom_judge_prompt,  # Overrides any named judge
    json_format=True
)
```

### Data Flow

1.  **Initialization**: `AuditExperiment` or `ModelAuditor` is configured with model details and judge settings.
2.  **Scenario Loading**: `get_scenarios` retrieves test data.
3.  **Probing**: The auditor model generates a probe based on `test_prompt` and `probe_prompt`.
4.  **Target Response**: The target model responds to the probe.
5.  **Judging**: The judge model evaluates the conversation history against `judge_prompt` criteria.
6.  **Parsing**: The judge's JSON response is parsed into an `AuditResult` with a `severity` score.
7.  **Aggregation**: Results are aggregated into `AuditResults` or `RepeatedExperimentResults` for analysis.

### See Also

*   [Key Ideas](key-ideas.md)
*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `RepeatedExperimentResults` can be iterated with `for` and supports index access with `[]` and supports `len()` and supports `in` membership testing.
