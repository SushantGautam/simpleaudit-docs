## Installation

SimpleAudit is a lightweight Python framework for red-teaming AI systems. It supports multiple LLM providers, including Anthropic (Claude), OpenAI (GPT-4, GPT-5, etc.), xAI (Grok), Ollama (local models), vLLM, and any OpenAI-compatible API.

This page covers how to install the library and configure the necessary environment variables for API keys.

### Prerequisites

*   **Python:** Version 3.9 or higher.
*   **Operating System:** Linux, macOS, or Windows.

### Installing the Package

SimpleAudit is available on PyPI. You can install it using `pip`:

```bash
pip install simpleaudit
```

If you are working in a virtual environment, ensure it is activated before running the installation command.

#### Verifying the Installation

To verify that the installation was successful, you can check the version of the installed package:

```python
import simpleaudit

print(simpleaudit.__version__)
```

Alternatively, you can check the version directly from the command line:

```bash
pip show simpleaudit
```

### Configuring API Keys

SimpleAudit requires API keys to interact with external LLM providers. These keys must be set as environment variables before running any audits. The specific variables required depend on the provider you intend to use.

#### Supported Providers and Environment Variables

| Provider | Environment Variable | Notes |
| :--- | :--- | :--- |
| **Anthropic** | `ANTHROPIC_API_KEY` | Default provider for SimpleAudit. |
| **OpenAI** | `OPENAI_API_KEY` | Supports GPT-4, GPT-5, and other OpenAI models. |
| **xAI (Grok)** | `XAI_API_KEY` | For accessing Grok models via xAI API. |
| **Ollama** | *(None)* | Local models do not require an API key. Ensure Ollama is running locally. |
| **vLLM** | *(None)* | Local model serving. Configure the base URL in the `ModelAuditor` constructor. |
| **OpenAI-Compatible** | `OPENAI_API_KEY` | For any API that mimics the OpenAI interface. You may need to specify a custom `base_url`. |

#### Setting Environment Variables

You can set environment variables in several ways:

1.  **Shell Session (Temporary):**
    ```bash
    export ANTHROPIC_API_KEY="your-anthropic-key"
    export OPENAI_API_KEY="your-openai-key"
    ```

2.  **`.env` File (Recommended for Development):**
    Create a `.env` file in your project root directory:
    ```env
    ANTHROPIC_API_KEY=your-anthropic-key
    OPENAI_API_KEY=your-openai-key
    XAI_API_KEY=your-xai-key
    ```
    *Note: SimpleAudit does not automatically load `.env` files. You must use a library like `python-dotenv` to load them into your environment before importing `simpleaudit`.*

    ```python
    from dotenv import load_dotenv
    load_dotenv()

    import simpleaudit
    ```

3.  **System Environment (Permanent):**
    Add the variables to your shell profile (`~/.bashrc`, `~/.zshrc`) or system environment settings.

### Quick Start Example

Once installed and configured, you can run a basic audit using the `ModelAuditor` class.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Initialize the auditor
# Note: 'provider' determines which API key is used.
auditor = ModelAuditor(
    model="claude-3-sonnet-20240229",
    provider="anthropic",
    judge_model="claude-3-opus-20240229",
    judge_provider="anthropic",
    system_prompt="You are a helpful assistant."
)

# Load a set of safety scenarios
scenarios = get_scenarios("safety")

# Run the audit
results = auditor.run(scenarios)

# Print a summary of the results
results.summary()
```

### Local Model Configuration (Ollama/vLLM)

If you are using local models via Ollama or vLLM, no API key is required. However, you must ensure the local server is running and accessible.

For **Ollama**:
1.  Install and start Ollama.
2.  Pull a model (e.g., `ollama pull llama3`).
3.  Use `provider="ollama"` in `ModelAuditor`.

For **vLLM** or other OpenAI-compatible local servers:
1.  Start your local server.
2.  Use `provider="openai"` and specify the `base_url` in the `ModelAuditor` constructor if it is not the default `http://localhost:8000/v1`.

```python
auditor = ModelAuditor(
    model="my-local-model",
    provider="openai",
    base_url="http://localhost:8000/v1",
    judge_model="my-local-judge",
    judge_provider="openai",
    judge_base_url="http://localhost:8000/v1"
)
```

### Troubleshooting

*   **`PackageNotFoundError`:** If you see `simpleaudit.__version__` returning `0.1.9` or a similar hardcoded value, it may indicate you are running from a source checkout without installing the package. Ensure you have run `pip install -e .` in the source directory or `pip install simpleaudit` in your target environment.
*   **Authentication Errors:** If you receive 401 or 403 errors, verify that the correct environment variable is set and that the API key is valid.
*   **Connection Errors:** For local providers (Ollama/vLLM), ensure the service is running and the port is accessible.

For more advanced configuration options, refer to the `ModelAuditor` documentation.

### See Also

*   [Quickstart](quickstart.md)
