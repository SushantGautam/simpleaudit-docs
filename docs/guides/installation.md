## Installation

SimpleAudit is a lightweight, local-first framework for AI safety auditing. It is designed to validate comparative LLM safety scoring without requiring ground-truth labels. The library supports open models running locally via Ollama or vLLM, as well as API-hosted models. By default, SimpleAudit does not collect or transmit user data.

### Prerequisites

*   **Python:** 3.11 or higher.
*   **Local Model Runtime (Optional):** If you intend to run models locally without API keys, you must have a local inference server running. The examples provided in the repository primarily use **Ollama**.

### Installing the Library

SimpleAudit is available on PyPI. You can install it using `pip`:

```bash
pip install simpleaudit
```

If you are working directly from the source repository, you can install it in editable mode:

```bash
git clone https://github.com/kelkalot/simpleaudit.git
cd simpleaudit
pip install -e .
```

### Configuring Local Model Dependencies

SimpleAudit is "local-first," meaning it is optimized to work with models running on your local machine. To use local models, you must ensure your inference server is running and the required models are pulled.

#### Using Ollama

The provided examples (e.g., `examples/local_ollama_audit.py`, `examples/custom_judge_ollama.py`) rely on Ollama.

1.  **Start the Ollama server:**
    ```bash
    ollama serve
    ```

2.  **Pull the required models:**
    You need at least two models: a **target model** (the model being audited) and a **judge model** (which evaluates the target's responses). The judge model should ideally support JSON output formats for structured scoring.

    For a quick start, the examples suggest:
    ```bash
    ollama pull llama3.2:3b     # Target model (small/fast)
    ollama pull gemma3:latest   # Judge model (supports json_object response format)
    ```

    *Note: You can substitute any other models available in Ollama by changing the `model` and `judge_model` parameters in the code.*

#### Using vLLM

SimpleAudit also supports vLLM for local inference. You must start your vLLM server with the desired model and ensure the `base_url` points to your local vLLM endpoint (typically `http://localhost:8000/v1`).

### Verification

To verify that the installation is successful and that your local models are accessible, you can run the provided quickstart example. This example uses Ollama and does not require API keys.

1.  Ensure Ollama is running and the models specified in the example are pulled.
2.  Run the example script:

    ```bash
    python examples/local_ollama_audit.py
    ```

    Or, if you prefer a Jupyter notebook environment:

    ```bash
    jupyter notebook examples/quickstart.ipynb
    ```

    *Note: The `quickstart.ipynb` and `quickstart_gemma_hf.ipynb` examples may require specific model configurations. The `local_ollama_audit.py` script is a good starting point for verifying local Ollama connectivity.*

### Offline Development (Fake Audit)

If you are setting up a development environment and do not yet have access to LLMs (local or API), you can use the `fake_audit.py` example. This script uses mock LLM clients to demonstrate the workflow without requiring any running model servers or API keys.

```bash
python examples/fake_audit.py
```

This is useful for:
*   Testing the installation.
*   Understanding the output structure (`AuditResults`, `AuditResult`).
*   CI/CD smoke tests.

### API Key Configuration (Optional)

If you plan to use API-hosted models (e.g., OpenAI, Anthropic) instead of local models, you will need to provide API keys. These are typically passed directly to the `ModelAuditor` or `AuditExperiment` constructors as parameters, such as `api_key`, `judge_api_key`, `auditor_api_key`, etc.

Example structure for API usage (conceptual):

```python
from simpleaudit import ModelAuditor

auditor = ModelAuditor(
    model="gpt-4o",
    provider="openai",
    api_key="YOUR_OPENAI_API_KEY",
    judge_model="gpt-4o",
    judge_provider="openai",
    judge_api_key="YOUR_OPENAI_API_KEY"
)
```

*Note: Ensure you handle your API keys securely and do not commit them to version control.*

### Next Steps

Once installed and verified, you can begin running audits by:
1.  Selecting a scenario pack using `get_scenarios(pack_name)`.
2.  Initializing a `ModelAuditor` or `AuditExperiment`.
3.  Running the audit using the `run()` or `run_async()` methods.

Refer to the **Usage** and **Scenario Packs** documentation pages for detailed instructions on configuring audits and interpreting results.

### See Also

*   [Quickstart](quickstart.md)
*   [Architecture](architecture.md)
*   [Key Ideas](key-ideas.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
