## Installation

SimpleAudit is a lightweight, local-first framework for AI safety auditing. It is designed to validate comparative LLM safety scoring without requiring ground-truth labels. The library supports open models running locally (via providers like Ollama or vLLM) and can optionally connect to API-hosted models. By default, SimpleAudit does not collect or transmit user data, ensuring privacy during local development and auditing workflows.

### Prerequisites

*   **Python 3.11+**: The library requires Python version 3.11 or higher.
*   **Local Model Provider (Optional but Recommended)**: For offline auditing, a local inference server such as **Ollama** or **vLLM** is required.
    *   For Ollama: Ensure `ollama serve` is running.
    *   For vLLM: Ensure your vLLM server is accessible via the configured base URL.

### Installing the Package

SimpleAudit is available on PyPI. You can install it using `pip`:

```bash
pip install simpleaudit
```

If you are working from the source code repository, you can install it in editable mode:

```bash
git clone https://github.com/kelkalot/simpleaudit.git
cd simpleaudit
pip install -e .
```

### Configuring Local Model Paths

SimpleAudit does not hardcode model paths. Instead, it relies on the **provider** and **model** parameters passed to the `ModelAuditor` or `AuditExperiment` classes. These parameters determine how the library connects to the LLMs.

#### Using Ollama

Ollama is the recommended provider for local, offline auditing. It supports JSON response formats for judge models, which is critical for structured output parsing.

To configure Ollama, you specify the model name as it appears in your local Ollama registry (e.g., `llama3.2:3b`, `gemma3:latest`).

**Example: Initializing a ModelAuditor with Ollama**

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define local Ollama model names
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL  = "gemma3:latest"

# Load a scenario pack
# 'bullshitbench' is a built-in pack; others include 'safety', 'rag', 'health'
scenarios = get_scenarios("bullshitbench")[:3]

# Initialize the auditor
# Note: json_format=False is often used for Ollama if the model does not strictly 
# enforce JSON mode, though newer versions may support it.
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    json_format=False,  # Adjust based on your Ollama model's capabilities
    verbose=True
)

# Run the audit
results = auditor.run(scenarios=scenarios, max_turns=3)
```

#### Using vLLM or OpenAI-Compatible APIs

If you are using vLLM or another OpenAI-compatible server, you can use the `openai` provider (or a custom provider string if supported by your backend) and specify the `base_url`.

**Example: Initializing with a Custom Base URL**

```python
from simpleaudit import ModelAuditor

auditor = ModelAuditor(
    model="my-local-model",
    provider="openai",  # Or the specific provider string your backend expects
    judge_model="my-judge-model",
    judge_provider="openai",
    base_url="http://localhost:8000/v1",  # Path to your vLLM or compatible server
    judge_base_url="http://localhost:8000/v1",
    # api_key="EMPTY"  # Often required for local OpenAI-compatible servers
)
```

### Verifying Installation

To verify that SimpleAudit is installed correctly and can connect to your local models, you can run a minimal smoke test using the `fake_audit.py` example provided in the repository, or a simple script that initializes the auditor.

**Option 1: Using the Repository Example (Recommended for Smoke Tests)**

The repository includes `examples/fake_audit.py`, which uses mock clients to verify the library structure without requiring actual LLM inference. This is useful for CI/CD pipelines or verifying Python environment setup.

```bash
# From the repository root
python examples/fake_audit.py
```

**Option 2: Quick Local Check**

If you have Ollama running, you can quickly check connectivity by initializing the auditor. If the connection fails, the library will raise an error during the `run` or `run_async` call.

```python
from simpleaudit import ModelAuditor, get_scenarios

try:
    auditor = ModelAuditor(
        model="llama3.2:3b",
        provider="ollama",
        judge_model="gemma3:latest",
        judge_provider="ollama"
    )
    # Fetch a single scenario to test the pipeline
    scenarios = get_scenarios("safety")[:1]
    
    # Run a single scenario with minimal turns
    result = auditor.run(scenarios=scenarios, max_turns=1)
    print("Audit successful. Score:", result.score)
except Exception as e:
    print(f"Connection or configuration error: {e}")
```

### Troubleshooting

1.  **Model Not Found**: Ensure the model name passed to `model` and `judge_model` exactly matches the name in your local provider's registry (e.g., `ollama list`).
2.  **JSON Parsing Errors**: If you encounter errors parsing judge responses, ensure your judge model supports structured output or set `json_format=False` and use a judge that outputs parseable text. The `parse_json_response` utility is used internally to handle these cases, but model compliance is required.
3.  **Base URL Issues**: When using `vLLM` or custom OpenAI-compatible servers, verify that `base_url` points to the correct API endpoint (typically `/v1`).
4.  **Provider String**: The `provider` parameter must match a supported provider string. Common values include `"ollama"` and `"openai"`. Check the `ModelAuditor` documentation for other supported providers.

### Next Steps

*   **Running Audits**: See the [Running Audits](#) documentation for details on `AuditExperiment` and `ModelAuditor` execution methods.
*   **Custom Judges**: Learn how to define custom judge prompts using `probe_prompt` and `judge_prompt` parameters in `ModelAuditor`.
*   **Scenario Packs**: Explore available scenario packs using `list_scenario_packs()` and `get_scenarios()`.

### See Also

*   [Quickstart](quickstart.md)
