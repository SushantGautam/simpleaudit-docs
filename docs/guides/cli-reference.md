## Command Line Interface

The `simpleaudit` library provides a robust Command Line Interface (CLI) that allows developers and system administrators to initiate audit trails, manage audit configurations, and query historical data without writing custom Python scripts. The CLI is built using `argparse` and is accessible via the `simpleaudit` executable installed with the package.

This module serves as the primary entry point for non-programmatic interaction with the audit engine. It handles argument parsing, configuration loading, and dispatches commands to the underlying audit services.

### Installation and Invocation

Ensure the `simpleaudit` package is installed in your environment. The CLI is invoked using the `simpleaudit` command followed by a subcommand.

```bash
simpleaudit --help
```

### Global Options

The following flags are available for all subcommands. They must be placed before the subcommand name.

| Flag | Description | Default |
| :--- | :--- | :--- |
| `-v`, `--verbose` | Enable verbose logging output. | `False` |
| `-q`, `--quiet` | Suppress all non-error output. | `False` |
| `--config PATH` | Path to a custom YAML/JSON configuration file. | `~/.simpleaudit/config.yaml` |
| `--version` | Display the library version and exit. | N/A |

### Subcommands

The CLI supports the following core subcommands:

#### 1. `init`

Initializes a new audit project or configures the audit engine for a specific application. This command generates the necessary directory structure and default configuration files.

**Usage:**
```bash
simpleaudit init [PROJECT_NAME]
```

**Arguments:**
*   `PROJECT_NAME`: (Optional) The name of the project. If omitted, the current directory name is used.

**Example:**
```bash
simpleaudit init my_application
```

This command creates a `simpleaudit_config.yaml` file in the current directory with default settings for log rotation, storage backend, and audit levels.

#### 2. `start`

Starts the audit daemon or initiates a one-shot audit session. This is the primary command for capturing events.

**Usage:**
```bash
simpleaudit start [--mode MODE] [--source SOURCE]
```

**Options:**
*   `--mode MODE`: Specifies the audit mode.
    *   `daemon`: Runs as a background service.
    *   `foreground`: Runs in the current terminal.
    *   `once`: Captures a single event and exits.
*   `--source SOURCE`: Specifies the data source to audit (e.g., `database`, `filesystem`, `api`). If not specified, it defaults to the source defined in the configuration file.

**Example:**
```bash
# Start auditing the database in daemon mode
simpleaudit start --mode daemon --source database

# Capture a single filesystem change
simpleaudit start --mode once --source filesystem
```

#### 3. `query`

Retrieves and filters audit logs from the storage backend. This command supports complex filtering using JSON-like syntax.

**Usage:**
```bash
simpleaudit query [--filter FILTER_JSON] [--limit LIMIT] [--output FORMAT]
```

**Options:**
*   `--filter FILTER_JSON`: A JSON string specifying query filters.
    *   Example: `{"user": "admin", "action": "login", "timestamp": ">2023-10-01"}`
*   `--limit LIMIT`: Maximum number of records to return. Default is `100`.
*   `--output FORMAT`: Output format for the results.
    *   `table`: Human-readable table (default).
    *   `json`: Raw JSON output.
    *   `csv`: Comma-separated values.

**Example:**
```bash
# Query the last 10 login attempts by user 'admin'
simpleaudit query --filter '{"user": "admin", "action": "login"}' --limit 10

# Export all failed logins to a CSV file
simpleaudit query --filter '{"status": "failed"}' --output csv > failed_logins.csv
```

#### 4. `status`

Displays the current status of the audit engine, including active sessions, storage health, and recent error counts.

**Usage:**
```bash
simpleaudit status
```

**Output Example:**
```text
Audit Engine Status: ACTIVE
Mode: Daemon
Source: database
Storage: PostgreSQL (Connected)
Last Event: 2023-10-27 10:45:22 UTC
Errors in last 24h: 0
```

#### 5. `config`

Manages the configuration file.

**Usage:**
```bash
simpleaudit config [show | set | validate]
```

**Sub-commands:**
*   `show`: Prints the current effective configuration.
*   `set KEY VALUE`: Sets a specific configuration key.
*   `validate`: Checks the configuration file for syntax errors and missing required fields.

**Example:**
```bash
# Show current configuration
simpleaudit config show

# Set the log level to DEBUG
simpleaudit config set logging.level DEBUG

# Validate the configuration
simpleaudit config validate
```

### Configuration File Reference

The CLI relies on a configuration file (typically `simpleaudit_config.yaml`) for persistent settings. Below are the key parameters supported by the CLI:

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `storage.backend` | `string` | Storage backend type (`sqlite`, `postgresql`, `elasticsearch`). |
| `storage.connection` | `string` | Connection string for the storage backend. |
| `logging.level` | `string` | Logging verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`). |
| `audit.sources` | `list` | List of enabled audit sources. |
| `rotation.max_size` | `int` | Maximum size of log files before rotation (in MB). |
| `rotation.backup_count` | `int` | Number of backup log files to keep. |

### Exit Codes

The CLI returns specific exit codes to indicate success or failure, which can be used in scripting:

*   `0`: Success.
*   `1`: General error (e.g., invalid arguments, configuration error).
*   `2`: Storage connection failure.
*   `3`: Permission denied.
*   `4`: Audit engine not running (when required).

### Troubleshooting

If the CLI fails to connect to the storage backend, ensure that:
1.  The `storage.connection` string in your configuration is correct.
2.  The database service is running and accessible.
3.  You have the necessary permissions to write to the storage location.

Use the `--verbose` flag to get detailed error messages:

```bash
simpleaudit start --verbose
```

This will output stack traces and connection details to stderr, which can help diagnose network or authentication issues.

### See Also

*   [Advanced Analysis & Meta-Evaluation](advanced-analysis.md)
*   [Architecture](architecture.md)
*   [Creating Custom Scenarios](custom-scenarios.md)
*   [Installation](installation.md)
