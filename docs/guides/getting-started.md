## Getting Started

Welcome to **simpleaudit**. This library provides a streamlined framework for executing automated audits of codebases, documentation, or data structures. It is designed to be lightweight, easy to integrate, and flexible enough to work with both local models and remote API endpoints.

This guide will walk you through the installation process, environment setup, and how to run your first audit using a local model.

### Prerequisites

Before installing `simpleaudit`, ensure your development environment meets the following requirements:

- **Python**: Version 3.8 or higher.
- **Operating System**: Linux, macOS, or Windows.
- **Dependencies**: `simpleaudit` relies on standard Python libraries for file handling and JSON processing. If you plan to use local LLMs, ensure your local inference engine (e.g., `llama.cpp`, `transformers`, or `ollama`) is installed and accessible via the command line or a local HTTP server.

### Installation

The recommended way to install `simpleaudit` is via `pip`.

```bash
pip install simpleaudit
```

If you are working within a virtual environment, activate it before running the installation command to keep your dependencies isolated.

For development purposes, you can clone the repository and install it in editable mode:

```bash
git clone https://github.com/your-org/simpleaudit.git
cd simpleaudit
pip install -e .
```

### Environment Setup

`simpleaudit` does not require complex configuration files for basic usage. However, it relies on environment variables to locate local model endpoints or API keys for remote services.

#### Configuring Local Models

If you are using a local model server (such as `ollama` or a local `vLLM` instance), you must ensure the server is running and accessible. By default, `simpleaudit` expects the local model endpoint to be available at `http://localhost:11434` (standard Ollama port) or `http://localhost:8000` (standard vLLM port).

You can override the default endpoint using the environment variable `SIMPLEAUDIT_MODEL_ENDPOINT`:

```bash
export SIMPLEAUDIT_MODEL_ENDPOINT="http://localhost:11434"
```

#### Configuring Remote APIs

If you are using a remote API (e.g., OpenAI, Anthropic), you must set the corresponding API key environment variable:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

### Running Your First Audit

The core of `simpleaudit` is the `AuditRunner` class. This class manages the lifecycle of an audit, including loading target files, processing them through the model, and generating a report.

#### Basic Usage

Below is a minimal example demonstrating how to audit a single Python file for potential security vulnerabilities using a local model.

```python
import simpleaudit
from simpleaudit.config import AuditConfig

# 1. Define the audit configuration
config = AuditConfig(
    model_name="llama3",  # Name of the model available on your local endpoint
    target_path="./src/main.py",  # Path to the file or directory to audit
    output_format="json",  # Output format: 'json', 'markdown', or 'text'
    verbose=True  # Enable verbose logging for debugging
)

# 2. Initialize the audit runner
runner = simpleaudit.AuditRunner(config)

# 3. Execute the audit
try:
    result = runner.run()
    
    # 4. Process the results
    if result.status == "success":
        print("Audit completed successfully.")
        print(f"Findings: {len(result.findings)}")
        
        # Print the first finding if any
        if result.findings:
            first_finding = result.findings[0]
            print(f"Line {first_finding.line_number}: {first_finding.description}")
    else:
        print(f"Audit failed: {result.error_message}")

except Exception as e:
    print(f"An unexpected error occurred: {e}")
```

#### Understanding the `AuditConfig`

The `AuditConfig` dataclass holds all parameters required to execute an audit. Below is a table of the primary parameters:

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `model_name` | `str` | `"default"` | The identifier for the model to use. Must match the model name registered on your inference endpoint. |
| `target_path` | `str` | `None` | The file path or directory path to audit. If a directory is provided, all supported files (`.py`, `.js`, `.ts`, etc.) will be scanned. |
| `output_format` | `str` | `"json"` | The format of the generated report. Supported values: `"json"`, `"markdown"`, `"text"`. |
| `max_tokens` | `int` | `1024` | The maximum number of tokens to generate per file. |
| `temperature` | `float` | `0.1` | The sampling temperature. Lower values result in more deterministic outputs. |
| `verbose` | `bool` | `False` | If `True`, logs detailed progress and model responses to `stdout`. |

#### Handling Directory Audits

You can audit an entire directory by passing the directory path to `target_path`. `simpleaudit` will recursively traverse the directory, ignoring common non-code files (e.g., `.git`, `node_modules`, `__pycache__`).

```python
config = AuditConfig(
    model_name="llama3",
    target_path="./src",  # Audit the entire 'src' directory
    output_format="markdown",
    output_file="audit_report.md"  # Save the report to a file
)

runner = simpleaudit.AuditRunner(config)
result = runner.run()

if result.status == "success":
    print(f"Report saved to {result.output_path}")
```

### Interpreting Results

The `run()` method returns an `AuditResult` object. This object contains the following attributes:

- **`status`**: A string indicating the outcome (`"success"`, `"error"`, or `"partial"`).
- **`findings`**: A list of `Finding` objects. Each `Finding` contains:
    - `file_path`: The relative path of the file where the issue was found.
    - `line_number`: The line number where the issue occurred.
    - `severity`: The severity level (`"low"`, `"medium"`, `"high"`, `"critical"`).
    - `description`: A human-readable explanation of the issue.
    - `suggestion`: A suggested fix or mitigation.
- **`error_message`**: A string describing the error if the audit failed.
- **`output_path`**: The path to the generated report file, if `output_file` was specified.

### Best Practices

1. **Start Small**: When integrating `simpleaudit` into your CI/CD pipeline, start with a small subset of files to verify that the model produces accurate results for your specific codebase.
2. **Tune Temperature**: For security audits, a lower temperature (e.g., `0.1` or `0.2`) is recommended to ensure consistent and deterministic outputs.
3. **Monitor Token Usage**: Auditing large directories can consume significant tokens. Use the `max_tokens` parameter to limit output size per file and monitor your API costs or local resource usage.
4. **Version Control**: Store your `AuditConfig` settings in a version-controlled file (e.g., `audit_config.yaml`) to ensure consistent audit parameters across different environments.

### Troubleshooting

- **Connection Error**: If you receive a connection error, verify that your local model server is running and that the `SIMPLEAUDIT_MODEL_ENDPOINT` environment variable is correctly set.
- **Model Not Found**: Ensure that the `model_name` specified in `AuditConfig` matches the exact name of the model loaded on your inference endpoint.
- **Timeouts**: For large files, the audit may take longer to complete. If you experience timeouts, consider increasing the timeout settings in your HTTP client configuration or splitting the audit into smaller chunks.

By following this guide, you should be able to successfully install and configure `simpleaudit` for your project. For advanced usage, including custom prompt templates and plugin development, please refer to the [Advanced Configuration](#) and [API Reference](#) sections of the documentation.

### See Also

*   [Advanced Evaluation Metrics](advanced-evaluation-metrics.md)
*   [Core Architecture](core-architecture.md)
*   [Creating Custom Scenarios](creating-custom-scenarios.md)
