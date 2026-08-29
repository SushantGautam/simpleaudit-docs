## CLI Usage

The `simpleaudit` library provides a robust command-line interface (CLI) that allows developers and system administrators to execute security audits directly from the terminal. This interface wraps the core Python API, enabling quick verification of system configurations, file integrity, and compliance standards without writing custom scripts.

The CLI is designed to be non-interactive by default, making it suitable for automation pipelines, cron jobs, and CI/CD workflows. All output is structured to be easily parsed by standard Unix tools like `grep`, `awk`, or `jq` (when JSON output is selected).

### Invoking the CLI

The primary entry point for the command-line interface is the `simpleaudit` executable. It is typically installed alongside the Python package via `pip install simpleaudit`.

To view the global help message and list available subcommands, run:

```bash
simpleaudit --help
```

To view help for a specific subcommand (e.g., `scan`), use:

```bash
simpleaudit scan --help
```

### Global Options

These options can be placed before or after the subcommand and apply to the entire execution context.

| Option | Description |
| :--- | :--- |
| `-v`, `--verbose` | Increase output verbosity. Use `-vv` for debug-level logging. |
| `-q`, `--quiet` | Suppress all non-error output. Only exit codes indicate success/failure. |
| `--version` | Display the current version of the `simpleaudit` library and exit. |
| `--config FILE` | Specify an alternative configuration file path. Defaults to `~/.simpleaudit/config.yaml`. |
| `--timeout SECONDS` | Set the maximum time (in seconds) allowed for a single audit task. Defaults to `300`. |

### Subcommands

The CLI supports several subcommands tailored to different audit scenarios.

#### 1. `scan`

The `scan` command performs a comprehensive audit of the specified target path. It checks file permissions, ownership, and integrity hashes against a baseline.

**Syntax:**

```bash
simpleaudit scan [TARGET_PATH] [OPTIONS]
```

**Parameters:**

*   `TARGET_PATH`: The directory or file to audit. If omitted, the current working directory (`.`) is used.

**Options:**

| Option | Description |
| :--- | :--- |
| `--recursive` | Recursively scan all subdirectories. Disabled by default for performance. |
| `--exclude PATTERN` | Exclude files/directories matching the glob pattern. Can be specified multiple times. |
| `--baseline FILE` | Path to a baseline JSON file containing expected hashes and permissions. |
| `--output FORMAT` | Output format: `text` (default), `json`, or `csv`. |
| `--fail-on-error` | Exit with code `1` if any critical issues are found. Default exit code is `0`. |

**Example: Basic Scan**

```bash
# Scan the /etc directory recursively, outputting results as JSON
simpleaudit scan /etc --recursive --output json > audit_report.json
```

**Example: Using a Baseline**

```bash
# Compare current state against a known-good baseline
simpleaudit scan /var/www/html --baseline baseline_2023.json --fail-on-error
```

#### 2. `verify`

The `verify` command is used to validate the integrity of specific files against a provided checksum list. It is lighter than `scan` and does not check permissions.

**Syntax:**

```bash
simpleaudit verify [CHECKSUM_FILE] [OPTIONS]
```

**Parameters:**

*   `CHECKSUM_FILE`: A file containing lines in the format `<hash>  <filename>`.

**Options:**

| Option | Description |
| :--- | :--- |
| `--hash-algo ALGO` | Specify the hash algorithm: `md5`, `sha1`, `sha256` (default). |
| `--strict` | Fail if any file in the checksum list is missing on the filesystem. |

**Example:**

```bash
# Verify integrity of downloaded packages
simpleaudit verify packages.sha256 --hash-algo sha256 --strict
```

#### 3. `baseline`

The `baseline` command generates a new baseline file from the current state of the target path. This is the first step in a continuous integrity monitoring workflow.

**Syntax:**

```bash
simpleaudit baseline [TARGET_PATH] [OPTIONS]
```

**Options:**

| Option | Description |
| :--- | :--- |
| `--output FILE` | Path where the baseline JSON file will be saved. Defaults to `baseline.json`. |
| `--include-hidden` | Include hidden files (starting with `.`) in the baseline. |
| `--exclude PATTERN` | Exclude files/directories matching the glob pattern. |

**Example:**

```bash
# Create a baseline for the application directory
simpleaudit baseline /opt/myapp --output /var/lib/audit/myapp_baseline.json
```

### Exit Codes

The CLI uses standard exit codes to indicate the status of the audit, which is crucial for scripting.

| Code | Meaning |
| :--- | :--- |
| `0` | Audit completed successfully. No critical issues found (or `--fail-on-error` not set). |
| `1` | Audit completed, but critical issues were found (and `--fail-on-error` was set). |
| `2` | Invalid arguments or configuration error. |
| `3` | Target path does not exist or is not accessible. |
| `4` | Timeout exceeded. |

### Configuration File

While most options can be passed via the CLI, you can define default values in a YAML configuration file. The CLI will merge these defaults with command-line arguments, with CLI arguments taking precedence.

**Default Location:** `~/.simpleaudit/config.yaml`

**Example Configuration:**

```yaml
# ~/.simpleaudit/config.yaml
default_timeout: 600
output_format: json
exclude_patterns:
  - "*.log"
  - ".git"
  - "node_modules"
log_level: INFO
```

### Practical Examples

#### 1. CI/CD Pipeline Integration

In a Continuous Integration pipeline, you might want to ensure that no unexpected files have been added to the repository before deployment.

```bash
# 1. Generate a baseline from the main branch
simpleaudit baseline ./src --output /tmp/main_baseline.json

# 2. Check out the feature branch
git checkout feature-branch

# 3. Scan and compare against the main baseline
simpleaudit scan ./src --baseline /tmp/main_baseline.json --fail-on-error

# 4. If exit code is 0, proceed with deployment
if [ $? -eq 0 ]; then
    echo "Audit passed. Proceeding with deployment."
else
    echo "Audit failed. Aborting deployment."
    exit 1
fi
```

#### 2. Scheduled Cron Job

You can schedule regular audits to monitor for configuration drift.

```bash
# /etc/cron.d/simpleaudit
# Run audit every day at 2:00 AM
0 2 * * * root simpleaudit scan /etc --recursive --output json --quiet >> /var/log/audit.log 2>&1
```

### Troubleshooting

*   **Permission Denied:** Ensure the user running the CLI has read access to the target directories. For system directories like `/etc`, you may need to run with `sudo`.
*   **Timeout Errors:** If you are scanning a large directory tree, increase the `--timeout` value or use `--exclude` to skip large, non-critical directories like `node_modules` or `.git`.
*   **JSON Parsing Errors:** If you are piping output to `jq`, ensure that `--quiet` is not set, as it may suppress the JSON structure in some edge cases. Always use `--output json` explicitly when parsing.

### Further Reading

*   [Python API Reference](./api.md) for programmatic access to audit functions.
*   [Configuration Guide](./config.md) for advanced configuration options.
*   [Baseline Format Specification](./baseline-format.md) for understanding the JSON structure of baseline files.

### See Also

*   [Reframing and Data Processing](reframing-and-data-processing.md)
