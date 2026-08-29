## Installation

SimpleAudit is a lightweight, local-first framework for multilingual AI safety auditing and red-teaming. It supports both local model inference (via Ollama or vLLM) and API-based providers (Anthropic, OpenAI, etc.). The package is distributed via PyPI and requires Python 3.11 or higher.

### Prerequisites

Before installing SimpleAudit, ensure your environment meets the following requirements:

1.  **Python Version**: Python 3.11+ is required.
    ```bash
    python --version
    ```
2.  **Package Manager**: `pip` (recommended) or `uv` for faster dependency resolution.

### Installation via pip

To install the core library, use the following command:

```bash
pip install simpleaudit
```

#### Optional Dependencies

Depending on your use case, you may need to install additional extras:

1.  **Plotting Support**: If you intend to use the `AuditResults.plot()` method to generate visual charts of your audit results, you must install `matplotlib`. You can install this alongside the core library using the `plot` extra:

    ```bash
    pip install "simpleaudit[plot]"
    ```

    Alternatively, if you have already installed the core library, you can add the dependency separately:
    ```bash
    pip install matplotlib
    ```

    *Note: The `local_ollama_audit.py` example in the repository demonstrates a try/except block around `results.plot()` to handle cases where `matplotlib` is not installed.*

2.  **Local Model Support (Ollama/vLLM)**:
    SimpleAudit itself does not bundle local model runtimes. To run audits against local models, you must have the corresponding server running:
    *   **Ollama**: Install from [ollama.ai](https://ollama.ai) and start the server (`ollama serve`).
    *   **vLLM**: Ensure your vLLM instance is running and accessible via an OpenAI-compatible API endpoint.

    No additional Python packages are strictly required by `simpleaudit` to connect to these services, as it communicates via standard HTTP APIs. However, ensure your network configuration allows communication with the local server (typically `http://localhost:11434` for Ollama).

### Verification

After installation, verify that the package is correctly installed by checking the version in your Python environment:

```python
import simpleaudit
print(simpleaudit.__version__)
```

You can also verify that the core classes are importable:

```python
from simpleaudit import ModelAuditor, AuditExperiment, get_scenarios, list_scenario_packs

# List available scenario packs to confirm data integrity
packs = list_scenario_packs()
print(f"Available scenario packs: {list(packs.keys())}")
```

### Local Development Installation

If you are contributing to SimpleAudit or wish to use the latest unreleased features, you can install the package from source in development mode.

1.  Clone the repository:
    ```bash
    git clone https://github.com/kelkalot/simpleaudit.git
    cd simpleaudit
    ```

2.  Install in editable mode:
    ```bash
    pip install -e .
    ```

3.  (Optional) Install development dependencies for testing:
    ```bash
    pip install -e ".[dev]"
    ```
    *Note: The specific dev extras name may vary; if `[dev]` is not defined, you may need to install test dependencies manually, such as `pytest`.*

### Environment Configuration

SimpleAudit is designed to be "local-first" and does not require API keys if you are using local models (Ollama/vLLM). However, if you plan to use API-based providers (e.g., Anthropic, OpenAI), you will need to configure your API keys.

While the `ModelAuditor` and `AuditExperiment` constructors accept `api_key` parameters directly, it is standard practice to manage these via environment variables to avoid hardcoding secrets.

**Supported Providers and Typical Environment Variables:**

| Provider | Typical Env Var | Notes |
| :--- | :--- | :--- |
| Anthropic | `ANTHROPIC_API_KEY` | Default provider for many examples. |
| OpenAI | `OPENAI_API_KEY` | For GPT-4, GPT-5, etc. |
| Grok (xAI) | `XAI_API_KEY` | For Grok models. |
| Ollama | N/A | Local server, no API key required. |
| vLLM | N/A | Local server, no API key required. |

*Note: The library documentation does not explicitly list specific environment variable names it reads internally. You should pass API keys explicitly via the `api_key` parameter in `ModelAuditor` or `AuditExperiment` constructors, or ensure your provider's SDK (if used under the hood) picks up the standard environment variables.*

### Quick Start Check

To confirm your installation is working end-to-end without requiring external API keys, you can run the provided fake audit example. This uses mock clients to simulate the audit process.

1.  Navigate to the `examples` directory (if installed from source):
    ```bash
    cd examples
    ```

2.  Run the fake audit script:
    ```bash
    python fake_audit.py
    ```

    This script imports from `tests.fakes` and `simpleaudit.experiment`. If you installed via pip, you may need to install the test utilities or replicate the logic in a local script using the `ModelAuditor` class with a local Ollama setup.

    For a pip-installed package, a minimal local check using Ollama (if available) is:

    ```python
    from simpleaudit import ModelAuditor, get_scenarios

    # Assumes Ollama is running locally
    auditor = ModelAuditor(
        model="llama3.2",
        provider="ollama",
        judge_model="llama3.2",
        judge_provider="ollama",
        max_turns=1,
        verbose=True
    )

    # Run a single scenario
    scenarios = get_scenarios("safety")[:1]
    results = auditor.run(scenarios)
    results.summary()
    ```

If this script runs without connection errors, your installation is successful and configured for local auditing.

### See Also

*   [Quickstart](quickstart.md)
*   [Architecture](architecture.md)
*   [Key Ideas](key-ideas.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
