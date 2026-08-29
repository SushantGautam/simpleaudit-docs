## Key Ideas

SimpleAudit is a lightweight, local-first framework for auditing the safety and behavior of Large Language Models (LLMs). It operates by simulating adversarial interactions between a **Target Model** (the system under test) and an **Auditor** (a red-team agent), followed by an evaluation by a **Judge Model**.

The core philosophy of SimpleAudit is **extensibility** and **local-first operation**. It supports running models locally via providers like Ollama or vLLM, eliminating the need for external API keys for basic usage. The framework is designed to be minimal, requiring no data collection or transmission by default.

### Core Architecture

The audit process follows a three-stage pipeline:

1.  **Probing (Auditor):** An LLM acts as a user, generating prompts designed to elicit specific behaviors (e.g., hallucinations, harmful content, or refusal) from the Target Model.
2.  **Interaction (Target):** The Target Model responds to the probes. This is a multi-turn conversation where the Auditor may follow up based on the Target's responses.
3.  **Evaluation (Judge):** A separate LLM (the Judge) analyzes the conversation transcript. It applies a specific evaluation schema (e.g., safety, factuality, helpfulness) and outputs a structured JSON verdict.

### Key Components

#### 1. `ModelAuditor`
The primary class for executing audits on a single model. It manages the interaction loop between the Auditor and Target models.

**Key Parameters:**
*   `model`: The name of the target model to audit.
*   `provider`: The provider for the target model (e.g., `"ollama"`, `"openai"`).
*   `judge_model`: The name of the model used for evaluation.
*   `judge_provider`: The provider for the judge model.
*   `judge`: A string identifier for a predefined judge configuration (e.g., `"safety"`, `"factuality"`).
*   `probe_prompt`: Custom system prompt for the Auditor (red-team agent).
*   `judge_prompt`: Custom system prompt for the Judge.
*   `json_format`: Whether to enforce JSON output from the Judge (default `True`).
*   `max_turns`: Maximum number of conversation turns per scenario.

#### 2. `AuditExperiment`
A higher-level orchestrator that runs audits across multiple models and scenarios. It supports parallel execution and repeated runs for stability analysis.

**Key Parameters:**
*   `models`: A list of dictionaries, where each dict defines a model configuration (e.g., `{"model": "llama3", "provider": "ollama"}`).
*   `judge`: The judge configuration name to apply to all models.
*   `n_repetitions`: Number of times to run the experiment for each model (for stability reporting).
*   `save_dir`: Directory to save results. If set, enables resumable experiments.

#### 3. Judge Configurations
SimpleAudit ships with several predefined judge configurations, each tailored to a specific evaluation dimension. These are accessible via the `judge` parameter in `ModelAuditor` or `AuditExperiment`.

| Judge Name | Focus Area | Output Schema Highlights |
| :--- | :--- | :--- |
| `safety` | Constitutional AI principles (harm avoidance, boundaries) | `severity` (critical/high/medium/low/pass), `issues_found`, `summary` |
| `factuality` | G-Eval based hallucination detection | `score` (1-10), `verdict` (accurate/partially_accurate/inaccurate), `factual_errors` |
| `helpfulness` | MT-Bench based response quality | `score` (1-10), `relevance`, `accuracy`, `clarity`, `completeness` |
| `abstention` | AbstentionBench based refusal evaluation | `score` (1-10), `abstained` (bool), `appropriate` (bool), `category` |
| `binary_abstention` | Binary classification of refusal behavior | `abstained` (bool), `reasoning` |
| `harm` | HELM Safety harm categorization | `harmful` (bool), `category` (violence/fraud/etc.), `severity` |

#### 4. Scenario Packs
Scenarios define the *context* for the audit. They are not just prompts, but structured objects with a `name`, `description`, and optional `expected_behavior`.

Built-in packs include:
*   `safety`: General AI safety (hallucinations, manipulation, privacy).
*   `health`: Healthcare-specific safety (diagnosis boundaries, emergency response).
*   `rag`: Retrieval-Augmented Generation specific issues (source misattribution, context confusion).
*   `system_prompt`: Testing resistance to prompt injection and override attempts.
*   `bullshitbench`: Testing for "bullshit" (nonsensical or fabricated content).

### Usage Example

The following example demonstrates a basic audit using local Ollama models. It uses the `safety` judge and the `safety` scenario pack.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define the models
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL = "gemma3:latest"

# Load scenarios from a built-in pack
scenarios = get_scenarios("safety")[:3]  # Take first 3 for a quick test

# Initialize the auditor
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    judge="safety",  # Use the built-in safety judge
    max_turns=3,     # Limit conversation length
    verbose=True
)

# Run the audit
results = auditor.run(scenarios)

# Access results
print(f"Score: {results.score}")
print(f"Critical Issues: {results.critical_count}")
for issue in results.all_issues:
    print(f"- {issue}")
```

### Results and Analysis

The `AuditResults` object provides a comprehensive view of the audit outcome.

**Key Properties:**
*   `score`: A normalized score (0-100) reflecting overall performance.
*   `severity_distribution`: A dictionary counting issues by severity level.
*   `all_issues`: A list of all identified issues across scenarios.
*   `all_recommendations`: A list of suggested improvements from the Judge.
*   `token_usage`: A breakdown of token consumption for Auditor, Judge, and Target models.

**Serialization:**
Results can be saved to and loaded from JSON files for later analysis or visualization.

```python
# Save results
results.save("audit_results.json")

# Load results later
loaded_results = AuditResults.load("audit_results.json")
```

### Visualization

SimpleAudit includes a built-in FastAPI-based visualizer to inspect audit results.

**Starting the Server:**
```python
from simpleaudit.visualization.server import start_server

# Point to the directory containing your saved JSON results
start_server(results_dir="./results", host="127.0.0.1", port=8000)
```

**Authentication:**
The visualizer supports optional authentication via the `SIMPLEAUDIT_VISUALIZER_SECRET` environment variable. If set, all API requests must include an `X-Secret` header with the matching value.

```bash
export SIMPLEAUDIT_VISUALIZER_SECRET="your-secret-key"
python -c "from simpleaudit.visualization.server import start_server; start_server('./results')"
```

### Stability and Repeated Runs

To assess the consistency of a model's behavior, you can run experiments multiple times using `n_repetitions` in `AuditExperiment`.

```python
from simpleaudit import AuditExperiment

experiment = AuditExperiment(
    models=[{"model": "llama3.2:3b", "provider": "ollama"}],
    judge="safety",
    n_repetitions=3,  # Run 3 times
    save_dir="./stability_results"
)

results = experiment.run(get_scenarios("safety"))

# Access stability report for a specific model
stability_report = results.stability("llama3.2:3b")
print(f"Mean Score: {stability_report.mean_score}")
print(f"Pass Rate: {stability_report.per_scenario_pass_rate}")
```

The `ModelStabilityReport` provides statistics such as mean score, min/max scores, and per-scenario pass rates, helping to distinguish consistent safety failures from sporadic errors.

### See Also

*   [Architecture](architecture.md)
*   [Quickstart](quickstart.md)
*   [Installation](installation.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
