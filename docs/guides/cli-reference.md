## Command Line Interface

The `simpleaudit` library provides a robust Command Line Interface (CLI) for executing AI safety audits, managing audit configurations, and viewing results. The CLI is built using `argparse` and serves as the primary entry point for users who prefer terminal-based workflows over programmatic integration.

This section documents all available commands, flags, and configuration options. The CLI is designed to be modular, allowing users to run single audits, batch processes, or inspect historical data without writing Python scripts.

### Installation and Entry Point

Ensure the library is installed in your environment:

```bash
pip install simpleaudit
```

The main entry point is accessible via the `simpleaudit` command. If you are running from a source checkout, you may need to use `python -m simpleaudit`.

### Global Flags

These flags apply to all commands and control the general behavior of the CLI.

| Flag | Description | Default |
| :--- | :--- | :--- |
| `-v, --verbose` | Enable verbose logging. Outputs debug information to stderr. | `False` |
| `--config-dir` | Override the default configuration directory. | `~/.simpleaudit` |
| `--output-dir` | Override the default output directory for audit results. | `./audit_results` |
| `--version` | Display the version of the `simpleaudit` CLI and exit. | N/A |

### Commands

The CLI supports four primary commands: `run`, `list`, `show`, and `init`.

#### 1. `init`

Initializes a new audit project by creating the necessary directory structure and a default configuration file. This command is idempotent; if the configuration file already exists, it will not be overwritten unless the `--force` flag is used.

**Usage:**

```bash
simpleaudit init [project_name]
```

**Arguments:**

*   `project_name` (optional): The name of the project. If omitted, the current directory name is used.

**Flags:**

*   `--force`: Overwrite existing configuration files.
*   `--template`: Specify a configuration template (e.g., `basic`, `advanced`). Default is `basic`.

**Example:**

```bash
# Initialize a project named 'llm_safety_check'
simpleaudit init llm_safety_check

# Initialize with an advanced template and force overwrite
simpleaudit init --template advanced --force
```

#### 2. `run`

Executes an audit based on a specified configuration file. This is the core command for performing safety checks. The CLI loads the configuration, instantiates the appropriate audit modules, and writes the results to the output directory.

**Usage:**

```bash
simpleaudit run --config <config_file> [options]
```

**Required Arguments:**

*   `--config, -c`: Path to the YAML or JSON configuration file defining the audit parameters.

**Optional Flags:**

*   `--model`: Override the model name specified in the configuration. Useful for quick A/B testing without editing files.
*   `--api-key-env`: The name of the environment variable containing the API key. If not specified, the CLI looks for `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` depending on the model provider.
*   `--limit`: Limit the number of test cases to execute. Useful for debugging or quick smoke tests.
*   `--resume`: If an interrupted audit is detected, resume from the last checkpoint.
*   `--dry-run`: Parse the configuration and validate inputs but do not execute any API calls.

**Example:**

```bash
# Run an audit using a specific config file
simpleaudit run -c configs/audit.yaml

# Run a limited audit for debugging, overriding the model
simpleaudit run -c configs/audit.yaml --model gpt-4 --limit 10

# Perform a dry run to validate configuration
simpleaudit run -c configs/audit.yaml --dry-run
```

**Configuration File Structure:**

The configuration file passed via `--config` should follow this schema:

```yaml
project:
  name: "My Safety Audit"
  description: "Testing for prompt injection vulnerabilities"

model:
  provider: "openai"
  name: "gpt-4"
  temperature: 0.0

audits:
  - type: "prompt_injection"
    enabled: true
    parameters:
      max_injections: 5
  - type: "toxicity"
    enabled: true
    parameters:
      threshold: 0.8

output:
  format: "json"
  include_raw_responses: true
```

#### 3. `list`

Lists all previously executed audits stored in the output directory. This command parses the metadata from the result files to provide a summary table.

**Usage:**

```bash
simpleaudit list [options]
```

**Flags:**

*   `--limit`: Show only the last N audits. Default is `10`.
*   `--filter`: Filter audits by status (`passed`, `failed`, `error`).
*   `--json`: Output the list as a JSON array instead of a formatted table.

**Example:**

```bash
# List the last 5 audits
simpleaudit list --limit 5

# List only failed audits in JSON format
simpleaudit list --filter failed --json
```

**Output Example:**

```text
ID                 Date             Model      Status   Score
-----------------  ---------------  ---------  -------  -----
audit_20231025_01  2023-10-25 10:00 gpt-4      passed   98.5
audit_20231024_03  2023-10-24 14:30 claude-3   failed   72.1
audit_20231024_01  2023-10-24 09:15 gpt-3.5    error    N/A
```

#### 4. `show`

Displays the detailed results of a specific audit. This command reads the result file and formats the output for human readability.

**Usage:**

```bash
simpleaudit show <audit_id> [options]
```

**Arguments:**

*   `audit_id`: The unique identifier of the audit (e.g., `audit_20231025_01`).

**Flags:**

*   `--raw`: Output the raw JSON result file without formatting.
*   `--failures-only`: Show only the test cases that failed the safety checks.
*   `--verbose`: Include full model responses and reasoning traces.

**Example:**

```bash
# Show summary of a specific audit
simpleaudit show audit_20231025_01

# Show only the failed cases with verbose details
simpleaudit show audit_20231025_01 --failures-only --verbose
```

### Configuration Management

While the `init` command creates a default configuration, users can manage configurations manually. The CLI respects the following precedence order for configuration values:

1.  **CLI Flags**: Highest priority (e.g., `--model`).
2.  **Configuration File**: Values defined in the YAML/JSON file passed via `--config`.
3.  **Environment Variables**: Used for secrets (e.g., API keys) and global overrides.
4.  **Defaults**: Hardcoded defaults in the library.

#### Environment Variables

The CLI automatically checks for the following environment variables:

*   `OPENAI_API_KEY`: Required for OpenAI models.
*   `ANTHROPIC_API_KEY`: Required for Anthropic models.
*   `SIMPLEAUDIT_LOG_LEVEL`: Sets the logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`). Overrides `-v` if set.
*   `SIMPLEAUDIT_HOME`: Overrides the default `~/.simpleaudit` directory.

### Error Handling

The CLI exits with non-zero status codes to indicate failure, which is useful for CI/CD pipelines.

*   `0`: Success.
*   `1`: General error (e.g., invalid configuration, missing file).
*   `2`: Usage error (e.g., missing required arguments).
*   `3`: Audit failure (all audits completed, but safety thresholds were not met).

**Example in CI/CD:**

```bash
#!/bin/bash
simpleaudit run -c configs/ci_audit.yaml
exit_code=$?

if [ $exit_code -eq 3 ]; then
    echo "Audit failed safety checks. Blocking deployment."
    exit 1
elif [ $exit_code -ne 0 ]; then
    echo "Audit execution error."
    exit 1
else
    echo "Audit passed."
    exit 0
fi
```

### Troubleshooting

*   **"Configuration file not found"**: Ensure the path provided in `--config` is correct relative to your current working directory.
*   **"API Key not found"**: Verify that the correct environment variable is set. Use `--api-key-env` to specify a custom variable name if necessary.
*   **"Permission denied"**: Ensure you have write permissions to the `--output-dir`.

For advanced use cases, such as custom audit modules or programmatic control, refer to the [Python API Reference](../api/index.md).

### See Also

*   [Architecture](architecture.md)
*   [Cross-Judging and Validation](cross-judging.md)
*   [Creating Custom Scenarios](custom-scenarios.md)
*   [Installation](installation.md)
