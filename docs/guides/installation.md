## Installation

SimpleAudit is a lightweight, local-first framework for AI safety auditing. It is designed to minimize setup overhead, supporting both local model inference (via Ollama or vLLM) and API-based models (Anthropic, OpenAI, etc.). The library does not collect or transmit user data by default.

### System Requirements

*   **Python:** 3.11 or higher.
*   **Dependencies:** The library relies on standard Python libraries for JSON parsing, regular expressions, and asynchronous execution.

### Installing the Package

You can install SimpleAudit directly from PyPI using `pip`:

```bash
pip install simpleaudit
```

If you are working from a local repository clone, you can install it in development mode:

```bash
git clone https://github.com/kelkalot/simpleaudit.git
cd simpleaudit
pip install -e .
```

### Configuring Local Model Environments

SimpleAudit supports local model inference, which is ideal for privacy-focused auditing or environments without API access. The two primary providers for local models are **Ollama** and **vLLM**.

#### Using Ollama

Ollama is a popular tool for running LLMs locally. To use SimpleAudit with Ollama:

1.  **Install and start Ollama:**
    ```bash
    ollama serve
    ```

2.  **Pull the required models:**
    You need a target model (the one being audited) and a judge model (the one evaluating the target). The judge model should ideally support JSON response formats for reliable parsing.

    ```bash
    # Example: Pulling a small target model and a capable judge model
    ollama pull llama3.2:3b
    ollama pull gemma3:latest
    ```

3.  **Configure the Auditor:**
    When initializing `ModelAuditor`, set the `provider` to `"ollama"`. Note that Ollama may have limitations with strict JSON response formats. If you encounter parsing issues, you can set `json_format=False` to disable strict JSON enforcement, though this may reduce the reliability of the judge's output.

    ```python
    from simpleaudit import ModelAuditor, get_scenarios

    auditor = ModelAuditor(
        model="llama3.2:3b",
        provider="ollama",
        judge_model="gemma3:latest",
        judge_provider="ollama",
        json_format=False  # Disable if Ollama JSON support is unstable
    )
    ```

#### Using vLLM

vLLM is a high-performance serving engine for LLMs. To use SimpleAudit with vLLM:

1.  **Start a vLLM server:**
    Ensure your vLLM server is running and accessible via an OpenAI-compatible API endpoint.

2.  **Configure the Auditor:**
    Set the `provider` to `"vllm"` and provide the `base_url` and `api_key` if your server requires authentication.

    ```python
    from simpleaudit import ModelAuditor, get_scenarios

    auditor = ModelAuditor(
        model="your-vllm-model-name",
        provider="vllm",
        base_url="http://localhost:8000/v1",
        api_key="your-api-key", # Optional, if server requires it
        judge_model="your-judge-model-name",
        judge_provider="vllm",
        judge_base_url="http://localhost:8000/v1",
        judge_api_key="your-api-key"
    )
    ```

### Configuring API-Based Models

SimpleAudit supports several API providers out of the box, including Anthropic, OpenAI, and Grok (xAI).

#### Setting API Keys

API keys are typically passed directly to the `ModelAuditor` constructor. For providers like Anthropic and OpenAI, you can also use environment variables, but explicit passing is recommended for clarity in scripts.

*   **Anthropic:** Set `provider="anthropic"` and `api_key="your-anthropic-key"`.
*   **OpenAI:** Set `provider="openai"` and `api_key="your-openai-key"`.
*   **Grok (xAI):** Set `provider="grok"` and `api_key="your-grok-key"`.

#### Example: Anthropic Setup

```python
from simpleaudit import ModelAuditor, get_scenarios

auditor = ModelAuditor(
    model="claude-3-sonnet-20240229",
    provider="anthropic",
    api_key="your-anthropic-api-key",
    judge_model="claude-3-opus-20240229",
    judge_provider="anthropic",
    judge_api_key="your-anthropic-api-key"
)
```

### Verifying the Installation

To verify that SimpleAudit is installed correctly and that your model configuration works, you can run a quick smoke test using the `fake_audit.py` example or a minimal script.

#### Minimal Verification Script

Create a file named `verify_install.py`:

```python
import asyncio
from simpleaudit import ModelAuditor, get_scenarios

async def main():
    # Use a small set of scenarios for a quick check
    scenarios = get_scenarios("safety")[:1]
    
    # Configure for a local Ollama model
    auditor = ModelAuditor(
        model="llama3.2:3b",
        provider="ollama",
        judge_model="gemma3:latest",
        judge_provider="ollama",
        json_format=False
    )
    
    print("Starting audit...")
    results = await auditor.run_async(scenarios, max_turns=1)
    
    print("Audit complete.")
    results.summary()

if __name__ == "__main__":
    asyncio.run(main())
```

Run the script:

```bash
python verify_install.py
```

If the script completes without errors and prints a summary, your installation and model configuration are successful.

### Troubleshooting

*   **Connection Errors:** Ensure that your local model server (Ollama or vLLM) is running and accessible at the specified `base_url`.
*   **JSON Parsing Errors:** If you see warnings about invalid JSON from the judge, try setting `json_format=False` in the `ModelAuditor` constructor. This is particularly common with smaller local models that may not strictly adhere to JSON schemas.
*   **Missing Models:** Ensure that the model names specified in `model` and `judge_model` match exactly the names of the models available in your Ollama library or vLLM server.

### Next Steps

Once installed and configured, you can begin running audits using `ModelAuditor` for single-model experiments or `AuditExperiment` for comparative studies across multiple models. Refer to the **Usage** section of the documentation for detailed examples on running audits and interpreting results.

### See Also

*   [Quickstart](quickstart.md)
