## Installation

SimpleAudit is a lightweight, local-first framework for auditing AI systems. It supports both local model inference (via Ollama or vLLM) and API-hosted models (Anthropic, OpenAI, etc.). The library is designed for minimal setup and does not require external API keys if you are running models locally.

### Prerequisites

*   **Python**: Version 3.11 or higher.
*   **Python Package Manager**: `pip` or `uv`.
*   **Local Model Runtime** (Optional): If you plan to audit local models, you must have a compatible runtime installed and running:
    *   **Ollama**: For local LLM inference.
    *   **vLLM**: For high-throughput local model serving.

### Installing the Package

You can install `simpleaudit` from PyPI using `pip`.

```bash
pip install simpleaudit
```

If you are using `uv` for faster dependency resolution:

```bash
uv pip install simpleaudit
```

### Verifying the Installation

To verify that the library is installed correctly and that the core modules are importable, run the following command in your terminal:

```bash
python -c "import simpleaudit; print(simpleaudit.__version__)"
```

This should output the current version of the library (e.g., `0.1.0`).

### Configuring Local Model Runtimes

If you intend to use SimpleAudit with local models, you must configure your local inference engine before running audits. SimpleAudit communicates with these engines via their standard HTTP APIs.

#### Ollama Setup

1.  **Install Ollama**: Follow the [official Ollama installation guide](https://ollama.com/download).
2.  **Start the Service**: Ensure the Ollama server is running.
    ```bash
    ollama serve
    ```
3.  **Pull Models**: You need at least two models: one for the **target** (the model being audited) and one for the **judge** (the model evaluating the target's responses).
    ```bash
    # Example: Pull a small target model and a capable judge model
    ollama pull llama3.2:3b
    ollama pull gemma3:latest
    ```

#### vLLM Setup

If you are using vLLM, ensure your vLLM server is running and exposing an OpenAI-compatible API endpoint. You will need to specify the `base_url` and `api_key` (can be a dummy string if not required) when initializing the auditor.

### API Key Configuration

If you are using API-hosted models (Anthropic, OpenAI, Grok, etc.), you must provide the appropriate API keys. SimpleAudit supports several methods for passing these keys:

1.  **Constructor Arguments**: Pass `api_key`, `judge_api_key`, and `auditor_api_key` directly to `ModelAuditor` or `AuditExperiment`.
2.  **Environment Variables**: While the core library relies on explicit parameters for clarity, you can manage keys via your environment or a secrets manager and pass them programmatically.

**Note**: For local providers like `ollama` or `vllm`, API keys are typically not required, but you may still need to specify the `base_url` if it is not the default (e.g., `http://localhost:11434` for Ollama).

### Quick Start Example

Below is a minimal example demonstrating how to initialize a `ModelAuditor` using local Ollama models. This example does not require any API keys.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define models
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL = "gemma3:latest"

# Initialize the auditor
# Note: json_format=False is recommended for Ollama if it does not support structured output
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    json_format=False,
    verbose=True
)

# Load a scenario pack
# 'safety' is a standard pack name; use list_scenario_packs() to see available packs
scenarios = get_scenarios("safety")

# Run the audit
results = auditor.run(scenarios)

# Print summary
results.summary()
```

### Using Fake Clients for Development

If you want to test SimpleAudit's logic without incurring API costs or waiting for local inference, you can use the built-in fake clients. This is useful for CI pipelines or local development.

```python
from simpleaudit import ModelAuditor
from tests.fakes import make_auditor, fixed_probe_auditor, fixed_severity_judge

# Create a fake auditor that returns predefined responses
# This bypasses actual LLM calls
fake_auditor = make_auditor(
    probe_auditor=fixed_probe_auditor("Hello"),
    judge=fixed_severity_judge("pass")
)

# You can now run scenarios against this fake auditor
# Note: The specific setup for fake clients may require importing from tests.fakes
# and configuring the ModelAuditor to use these mocks, which is typically done
# in unit tests or specific development scripts.
```

### Troubleshooting

*   **Connection Errors**: If you see connection refused errors when using Ollama or vLLM, ensure the service is running and the `base_url` is correct.
*   **JSON Parsing Errors**: If you encounter issues with JSON parsing, ensure that your judge model supports structured output or set `json_format=False` to allow the library to use more robust text-based parsing strategies.
*   **Missing Scenarios**: If `get_scenarios` raises an error, use `list_scenario_packs()` to verify the correct pack name.

For more detailed configuration options, refer to the `ModelAuditor` and `AuditExperiment` class documentation.

### See Also

*   [Quickstart](quickstart.md)
*   [Key Ideas](key-ideas.md)
