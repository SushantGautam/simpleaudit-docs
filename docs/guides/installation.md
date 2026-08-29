## Installation

SimpleAudit is a lightweight, local-first framework for AI safety auditing. It is designed to run with minimal setup, supporting both local models (via Ollama or vLLM) and API-hosted models. The library does not collect or transmit user data by default.

### System Requirements

*   **Python:** 3.11 or higher.
*   **Hardware:** A local GPU is recommended for running target models locally, though CPU-only execution is possible for smaller models or API-based workflows.

### Package Installation

You can install SimpleAudit from PyPI using `pip`:

```bash
pip install simpleaudit
```

If you are contributing to the project or need the latest development version, you can install it from source:

```bash
git clone https://github.com/kelkalot/simpleaudit.git
cd simpleaudit
pip install -e .
```

### Local Model Setup (Ollama)

SimpleAudit is optimized for local-first workflows. The most common local backend is **Ollama**. To use local models, you must first install and start the Ollama server.

1.  **Install Ollama:** Follow the official [Ollama installation guide](https://ollama.com/download).
2.  **Start the Server:** Ensure the Ollama service is running in the background:
    ```bash
    ollama serve
    ```
3.  **Pull Models:** You need at least two models: a **target model** (the one being audited) and a **judge model** (which evaluates the target's responses). The judge model should be capable of structured output (JSON) for best results.

    Example using `llama3.2:3b` as the target and `gemma3:latest` as the judge:

    ```bash
    ollama pull llama3.2:3b
    ollama pull gemma3:latest
    ```

### Local Model Setup (vLLM)

For higher throughput or specific hardware configurations, you can use **vLLM** as the provider. This requires running a vLLM server that exposes an OpenAI-compatible API endpoint.

1.  Install `vllm` via pip.
2.  Start a vLLM server pointing to your model weights:
    ```bash
    vllm serve <model_name_or_path> --port 8000
    ```
3.  When configuring SimpleAudit, set the `provider` to `"vllm"` and provide the `base_url` (e.g., `http://localhost:8000/v1`).

### Verifying the Installation

To verify that SimpleAudit is installed correctly and that your local model backend is reachable, you can run a minimal smoke test. The following example uses the `ModelAuditor` class with Ollama. It runs a single scenario from the built-in `safety` pack.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define models
TARGET_MODEL = "llama3.2:3b"
JUDGE_MODEL = "gemma3:latest"

# Get a small subset of scenarios for a quick test
scenarios = get_scenarios("safety")[:1]

# Initialize the auditor
# Note: json_format=False is often used for Ollama if it doesn't support strict JSON mode
auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    json_format=False,
    verbose=True
)

# Run the audit
results = auditor.run(scenarios, max_turns=1)

# Print a summary
print(results.summary())
```

If this script runs without connection errors and prints a summary, your installation is successful.

### Configuration Notes

SimpleAudit relies on constructor parameters for configuration rather than global environment variables for core logic. Key configuration points include:

*   **Provider:** Specifies the backend (`"ollama"`, `"vllm"`, `"openai"`, etc.).
*   **Base URL:** Required for local servers (Ollama/vLLM) or custom API endpoints.
*   **API Keys:** Only required for cloud-based providers (e.g., OpenAI, Anthropic). Local providers typically do not require API keys.

### Troubleshooting

*   **Connection Refused:** Ensure your local model server (Ollama/vLLM) is running and accessible at the specified `base_url`.
*   **JSON Parsing Errors:** If using a judge model that struggles with structured output, consider setting `json_format=False` in `ModelAuditor` or using a more capable judge model.
*   **Missing Scenarios:** If `get_scenarios(pack_name)` fails, ensure you have the latest version of the package, as scenario packs are bundled with the library. You can list available packs using `list_scenario_packs()`.

For more advanced setups, such as custom judge prompts or multi-model experiments, refer to the `examples/` directory in the repository, particularly `custom_judge_ollama.py` and `quickstart.ipynb`.

### See Also

*   [Quickstart](quickstart.md)
*   [Architecture](architecture.md)
