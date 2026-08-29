## Model Auditor

The `ModelAuditor` class is the primary interface for executing audit probes against Large Language Models (LLMs). It abstracts the complexity of connecting to various model providers (local servers or remote APIs), managing request payloads, and collecting structured responses for analysis.

This module is designed to be provider-agnostic. Whether you are auditing a model served via Ollama, a local vLLM instance, or a commercial API like OpenAI or Anthropic, `ModelAuditor` provides a unified method to send prompts and retrieve completions.

### Overview

The `ModelAuditor` handles the following responsibilities:
1.  **Connection Management**: Establishes and maintains connections to the target LLM endpoint.
2.  **Request Formatting**: Converts audit probe definitions into the specific request format required by the target provider.
3.  **Response Parsing**: Normalizes raw API responses into a consistent structure that downstream audit analyzers can process.
4.  **Error Handling**: Catches network errors, rate limits, and invalid model identifiers, providing descriptive error messages.

### Class Definition

```python
class ModelAuditor:
    def __init__(self, provider: str, model_name: str, api_key: str = None, base_url: str = None, timeout: int = 30):
        """
        Initialize the ModelAuditor.

        Args:
            provider (str): The provider type. Supported values: 'openai', 'anthropic', 'ollama', 'local'.
            model_name (str): The specific model identifier (e.g., 'gpt-4', 'claude-3-opus', 'llama3').
            api_key (str, optional): API key for providers requiring authentication.
            base_url (str, optional): Custom base URL for the API endpoint. Required for 'local' or custom endpoints.
            timeout (int, optional): Request timeout in seconds. Defaults to 30.
        """
        pass

    def audit(self, prompt: str, system_prompt: str = None, temperature: float = 0.0, max_tokens: int = 1024) -> dict:
        """
        Execute a single audit probe.

        Args:
            prompt (str): The user prompt to send to the model.
            system_prompt (str, optional): The system instruction context.
            temperature (float, optional): Sampling temperature. Defaults to 0.0 for deterministic outputs.
            max_tokens (int, optional): Maximum number of tokens to generate. Defaults to 1024.

        Returns:
            dict: A dictionary containing:
                - 'content' (str): The model's response text.
                - 'model' (str): The model name used.
                - 'provider' (str): The provider identifier.
                - 'usage' (dict): Token usage statistics if available.
                - 'latency_ms' (int): Time taken for the request in milliseconds.
                - 'error' (str, optional): Error message if the request failed.
        """
        pass
```

### Configuration Parameters

When initializing `ModelAuditor`, you must specify the `provider` and `model_name`. Additional parameters depend on the provider type.

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `provider` | `str` | Yes | The LLM provider. Valid options: `openai`, `anthropic`, `ollama`, `local`. |
| `model_name` | `str` | Yes | The specific model ID (e.g., `gpt-4o`, `claude-3-sonnet`, `llama3:8b`). |
| `api_key` | `str` | Conditional | Required for `openai` and `anthropic`. Optional for others. |
| `base_url` | `str` | Conditional | Required for `local` and `ollama` if not using default localhost ports. |
| `timeout` | `int` | No | Maximum wait time for a response. Defaults to 30 seconds. |

### Usage Examples

#### 1. Auditing an OpenAI Model

To audit a model hosted by OpenAI, you must provide your API key. The `base_url` is not required as it defaults to the standard OpenAI endpoint.

```python
from simpleaudit import ModelAuditor

# Initialize the auditor
auditor = ModelAuditor(
    provider="openai",
    model_name="gpt-4o",
    api_key="sk-your-openai-api-key"
)

# Run a probe
result = auditor.audit(
    prompt="Explain the concept of prompt injection in simple terms.",
    system_prompt="You are a helpful AI assistant.",
    temperature=0.1
)

if 'error' in result:
    print(f"Error: {result['error']}")
else:
    print(f"Response: {result['content']}")
    print(f"Latency: {result['latency_ms']}ms")
    print(f"Tokens Used: {result['usage']}")
```

#### 2. Auditing a Local Ollama Instance

For local models served via Ollama, the `base_url` typically points to the local Ollama server (default `http://localhost:11434`). No API key is required.

```python
from simpleaudit import ModelAuditor

# Initialize for local Ollama
auditor = ModelAuditor(
    provider="ollama",
    model_name="llama3:8b",
    base_url="http://localhost:11434"
)

# Run a probe
result = auditor.audit(
    prompt="What are the security risks of using LLMs in production?",
    max_tokens=512
)

print(result['content'])
```

#### 3. Auditing a Custom Local Endpoint (vLLM, TGI, etc.)

If you are using a local inference server that mimics the OpenAI API format (such as vLLM or Text Generation Inference), use the `local` provider. You must specify the `base_url` where the server is running.

```python
from simpleaudit import ModelAuditor

# Initialize for a custom local endpoint
auditor = ModelAuditor(
    provider="local",
    model_name="my-custom-model",
    base_url="http://localhost:8000/v1"
)

# Run a probe
result = auditor.audit(
    prompt="Analyze the following code for vulnerabilities: [CODE SNIPPET]",
    system_prompt="You are a senior security engineer."
)

if result.get('error'):
    raise Exception(f"Audit failed: {result['error']}")
```

### Response Structure

The `audit` method always returns a dictionary. Understanding this structure is crucial for integrating `ModelAuditor` into larger audit pipelines.

**Success Case:**
```json
{
    "content": "The model's generated text...",
    "model": "gpt-4o",
    "provider": "openai",
    "usage": {
        "prompt_tokens": 45,
        "completion_tokens": 120,
        "total_tokens": 165
    },
    "latency_ms": 850
}
```

**Error Case:**
If the request fails (e.g., network timeout, invalid model, rate limit), the dictionary will contain an `error` key. The `content` key may be `None` or empty.

```json
{
    "content": null,
    "model": "gpt-4o",
    "provider": "openai",
    "usage": null,
    "latency_ms": 30000,
    "error": "Connection timed out after 30s"
}
```

### Best Practices

1.  **Deterministic Auditing**: Set `temperature=0.0` (default) when running security or compliance audits to ensure reproducible results.
2.  **Timeout Management**: For local models with high latency, increase the `timeout` parameter in the constructor to prevent premature failure.
3.  **Error Handling**: Always check for the `error` key in the response dictionary before processing the `content`.
4.  **Provider Specifics**:
    *   **OpenAI/Anthropic**: Ensure your `api_key` has sufficient permissions for the specified `model_name`.
    *   **Ollama/Local**: Verify that the model is loaded and available on the target server before initiating the auditor.

### Troubleshooting

| Issue | Possible Cause | Solution |
| :--- | :--- | :--- |
| `ConnectionError` | Incorrect `base_url` or server is down. | Verify the URL and ensure the LLM server is running. |
| `AuthenticationError` | Invalid or missing `api_key`. | Check your API key and ensure it is correctly passed to the constructor. |
| `ModelNotFound` | Incorrect `model_name`. | Verify the exact model identifier with your provider's documentation. |
| `Timeout` | Model is slow to respond. | Increase the `timeout` parameter in the `ModelAuditor` constructor. |

By using `ModelAuditor`, you can seamlessly integrate LLM probes into your `simpleaudit` workflows, ensuring consistent and reliable data collection across diverse model environments.

### See Also

*   [Architecture](architecture.md)
*   [Creating Custom Scenarios](custom-scenarios.md)
*   [Results and Analysis](results.md)
