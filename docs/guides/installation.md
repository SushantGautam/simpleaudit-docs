## Installation

SimpleAudit is a lightweight AI safety auditing framework designed for validating comparative LLM safety scoring without ground-truth labels. It supports local model serving via Ollama and vLLM, as well as API-based providers like Anthropic and OpenAI. This page covers the installation process and environment verification for Python 3.11+.

### Prerequisites

SimpleAudit requires **Python 3.11 or higher**. Verify your Python version by running:

```bash
python --version
```

If you are using a version older than 3.11, please upgrade your Python environment before proceeding.

### Installing the Package

You can install SimpleAudit from PyPI using `pip`:

```bash
pip install simpleaudit
```

If you are working with a specific project environment, it is recommended to create a virtual environment first:

```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install simpleaudit
```

### Verifying the Installation

To verify that SimpleAudit is installed correctly and that the core modules are accessible, you can run the following Python snippet. This checks the import of the primary `ModelAuditor` class and the version string.

```python
import simpleaudit

print(f"SimpleAudit version: {simpleaudit.__version__}")

# Verify core classes are importable
from simpleaudit import ModelAuditor, AuditResults, get_scenarios, list_scenario_packs

print("Core imports successful.")
```

If this script runs without errors, your installation is successful.

### Optional Dependencies for Local Models

SimpleAudit is designed to be local-first. If you plan to audit models running locally via **Ollama** or **vLLM**, you do not need to install additional Python packages for the library itself, as it communicates with these services via their HTTP APIs. However, you must ensure the local model server is running.

For **Ollama**:
1. Install Ollama from [ollama.com](https://ollama.com).
2. Start the server: `ollama serve`.
3. Pull a model: `ollama pull llama3.2:3b`.

For **vLLM**:
1. Ensure `vllm` is installed in your environment if you are running the server via Python.
2. Start the vLLM server with your chosen model.

SimpleAudit will connect to these servers using the `provider` parameter set to `"ollama"` or `"vllm"` in the `ModelAuditor` constructor.

### Quick Verification Example

To perform a minimal functional check without requiring external API keys or local models, you can inspect the available scenario packs. This confirms that the data loading mechanisms are working.

```python
from simpleaudit import list_scenario_packs

# List available scenario packs
packs = list_scenario_packs()
print(f"Available scenario packs: {packs}")
```

### Environment Configuration

SimpleAudit does not require specific environment variables for basic installation. However, if you are using API-based providers (e.g., Anthropic, OpenAI), you must configure your API keys according to the provider's standard methods (e.g., setting `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` in your environment or passing them explicitly to the `ModelAuditor` constructor via `api_key`).

For local providers (Ollama, vLLM), no API keys are required by default, as the local servers typically do not enforce authentication.

### Troubleshooting

1. **Import Errors**: If you encounter `ModuleNotFoundError: No module named 'simpleaudit'`, ensure you have activated the correct virtual environment and that `pip install simpleaudit` completed successfully.
2. **Python Version**: If you see syntax errors or compatibility issues, verify that your Python version is 3.11 or higher. SimpleAudit does not support Python 3.10 or earlier.
3. **Local Server Connectivity**: If auditing local models fails, ensure that the Ollama or vLLM server is running and accessible at the expected base URL (default is usually `http://localhost:11434` for Ollama and `http://localhost:8000` for vLLM). You can explicitly set the `base_url` in the `ModelAuditor` constructor if your server runs on a non-standard port or host.

### Next Steps

Once installed, you can begin auditing by creating a `ModelAuditor` instance. See the [Quickstart](#) guide for examples of running your first audit using `ModelAuditor.run()` or `AuditExperiment`.

### See Also

*   [Quickstart](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
