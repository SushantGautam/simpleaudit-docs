## Key Ideas

SimpleAudit operates on a comparative evaluation model: it does not require ground-truth labels to determine if a model is "safe." Instead, it uses a **Judge** model to evaluate the responses of a **Target** model against a specific **Scenario**. The system validates the consistency and comparative safety of LLMs by analyzing how the target model behaves under adversarial or domain-specific probes.

The three core components—Judges, Scenarios, and Visualization—integrate as follows:
1.  **Scenarios** define the test cases (prompts and expected behaviors).
2.  **Judges** define the criteria and output schema for evaluating the target model's responses.
3.  **Visualization** provides a local web interface to inspect the resulting JSON data.

### Judges

Judges are configurations that dictate how the evaluation is performed. They consist of a `probe_prompt` (instructions for the auditor to interact with the target) and a `judge_prompt` (instructions for the judge model to score the target's response).

SimpleAudit provides several built-in judges, accessible via `simpleaudit.judges`.

#### Built-in Judges

| Judge Name | Description |
| :--- | :--- |
| `safety` | Constitutional AI safety evaluation. Severity: critical, high, medium, low, pass. |
| `abstention` | Refusal/abstention appropriateness. Score 1–10. |
| `helpfulness` | Response quality across MT-Bench dimensions. Score 1–10. |
| `factuality` | Hallucination and factual error detection. Score 1–10. |
| `harm` | HELM Safety harm categorisation. |
| `helsedir_sexhealth_no` | Norwegian sexual-health judge for young users. |
| `helsedir_sexhealth_no_rag` | Norwegian sexual-health judge with RAG framing. |
| `binary_abstention` | Language-agnostic binary classifier for abstention. |

#### Using Judges

You can select a judge by name when initializing `ModelAuditor`. You can also override the default prompts using `probe_prompt` and `judge_prompt` parameters.

```python
from simpleaudit import ModelAuditor

# Use a named judge config
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety"
)

# Named judge + custom probe
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="factuality",
    probe_prompt="Ask about specific historical dates to test recall."
)

# Fully custom judge (judge_prompt takes precedence over named judge)
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge_prompt="You are a custom judge. Evaluate if the response is polite."
)
```

To list available judges programmatically:

```python
from simpleaudit.judges import list_judge_configs

configs = list_judge_configs()
for name, desc in configs.items():
    print(f"{name}: {desc}")
```

### Scenarios

Scenarios are lists of dictionaries containing test cases. Each scenario typically includes a `name`, `description`, `test_prompt`, and `expected_behavior`.

SimpleAudit includes several built-in scenario packs, accessible via `simpleaudit.scenarios`.

#### Built-in Scenario Packs

| Pack Name | Description |
| :--- | :--- |
| `safety` | General AI safety scenarios |
| `rag` | RAG-specific scenarios |
| `health` | Healthcare domain scenarios |
| `system_prompt` | System prompt adherence/bypass testing |
| `helpmed` | Help and medical scenarios |
| `ung` | UNG scenarios |
| `bullshitbench` | BullshitBench v1+v2 combined (155 scenarios) |
| `hei_refusal` | Norwegian youth Q&A refusal/guidance edge cases |
| `nav_aap` | Norwegian welfare scenarios |
| `all` | All scenarios combined |

#### Using Scenarios

You can retrieve scenarios by pack name. These can be passed directly to `ModelAuditor.run()` or `AuditExperiment.run()`.

```python
from simpleaudit.scenarios import get_scenarios, list_scenario_packs

# List available packs and their sizes
packs = list_scenario_packs()
print(packs)

# Get scenarios for a specific pack
scenarios = get_scenarios("safety")
print(f"Loaded {len(scenarios)} safety scenarios")

# Run audit with specific scenarios
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety"
)

results = auditor.run(scenarios, max_turns=5)
```

### Visualization

SimpleAudit includes a local FastAPI server for visualizing audit results. This server reads JSON files from a specified directory and provides a web interface to explore the data.

#### Starting the Server

The `start_server` function initializes the visualization server. It requires a `results_dir` containing the JSON output from your audits.

**Environment Variables:**
*   `SIMPLEAUDIT_VISUALIZER_SECRET`: If set, the server requires an `X-Secret` header for API requests.
*   `SIMPLEAUDIT_VISUALIZER_EMAIL`: Contact email displayed in the authentication overlay.

```python
import os
from simpleaudit.visualization.server import start_server

# Set optional secret for authentication
os.environ["SIMPLEAUDIT_VISUALIZER_SECRET"] = "my-secret-key"

# Start the server on port 8000
start_server(
    results_dir="./audit_results",
    host="127.0.0.1",
    port=8000
)
```

#### Server Endpoints

*   `/`: Serves the main visualization HTML page.
*   `/scenario_viewer.html`: Serves the standalone scenario viewer.
*   `/api/auth`: Checks authentication status and returns the contact email.
*   `/api/files`: Returns a tree structure of valid JSON audit files in the results directory.
*   `/api/json/{file_path}`: Returns the contents of a specific JSON file.

The server validates JSON files before including them in the file tree. A file is considered valid if it is a non-empty array or an object with a non-empty `results` array.

### Integration Example

A typical workflow involves running an experiment and then visualizing the results.

```python
import os
from simpleaudit import ModelAuditor
from simpleaudit.scenarios import get_scenarios
from simpleaudit.visualization.server import start_server

# 1. Define Auditor
auditor = ModelAuditor(
    model="llama3.2:3b",
    provider="ollama",
    judge_model="gemma3:latest",
    judge_provider="ollama",
    judge="safety",
    save_dir="./my_audit_results"
)

# 2. Run Audit
scenarios = get_scenarios("safety")[:5] # Run first 5 scenarios for demo
results = auditor.run(scenarios, max_turns=3)

# 3. Save Results (handled by save_dir in ModelAuditor if configured, 
#    or manually via results.save())
# Note: ModelAuditor with save_dir typically handles saving internally 
#    or results can be saved explicitly:
# results.save("./my_audit_results/run1.json")

# 4. Visualize
# Ensure the directory exists and contains the JSON files
start_server(results_dir="./my_audit_results", port=8000)
```

This workflow demonstrates how the components integrate: the `ModelAuditor` executes the scenarios using the specified judge, produces structured results, and the `start_server` function provides a local interface to inspect those results.

### See Also

*   [Architecture](architecture.md)
*   [Quickstart](quickstart.md)
*   [Installation](installation.md)
