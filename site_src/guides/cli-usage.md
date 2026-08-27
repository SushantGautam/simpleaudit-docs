## CLI Usage

The `simpleaudit` command-line interface (CLI) provides tools for visualizing and exporting AI safety audit results. It enables developers to inspect JSON-based audit data through a local web server or generate standalone HTML reports for offline distribution. The CLI is implemented in `simpleaudit/cli.py` and serves as the primary entry point for post-audit analysis.

### Installation and Dependencies

The core `simpleaudit` package does not include visualization dependencies by default. To use the `serve` and `export-html` commands, optional dependencies must be installed. If these are missing, the CLI exits with an error message instructing the user to install the `visualize` extra.

```bash
pip install 'simpleaudit[visualize]'
```

### Command Overview

The CLI supports two primary subcommands:

1.  **`serve`**: Starts a local web server to dynamically visualize audit results from a directory.
2.  **`export-html`**: Generates a single, self-contained HTML file from a specific JSON result file.

If no command is provided, or if an unrecognized command is used, the CLI prints the help message and exits with status code `1`.

### Command: `serve`

The `serve` command launches a web server that allows interactive visualization of audit results. It scans a specified directory for JSON files and serves them via a browser interface.

**Syntax:**
```bash
simpleaudit serve [--results_dir DIR] [--port PORT] [--host HOST]
```

**Arguments:**

| Flag | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `--results_dir` | `str` | `.` (current directory) | Path to the directory containing JSON result files. If not specified, the current working directory is used, and a warning is printed. |
| `--port` | `int` | `8000` | The network port on which the server will listen. |
| `--host` | `str` | `127.0.0.1` | The host interface to bind the server to. Defaults to localhost for security. |

**Behavior:**
*   If `--results_dir` is omitted, the CLI defaults to `.` and prints a warning: `⚠️  Warning: --results_dir not specified, using current directory '.'`. It is recommended to explicitly set this flag to avoid confusion.
*   The command imports `start_server` from `simpleaudit.visualization.server`. If this module or its dependencies are missing, the CLI prints an error and exits.
*   Upon successful startup, the server binds to the specified host and port.

**Example Usage:**
```bash
# Serve results from a specific directory on port 5000
simpleaudit serve --results_dir ./audit_outputs --port 5000

# Serve results from the current directory on the default port
simpleaudit serve
```

### Command: `export-html`

The `export-html` command converts a single JSON audit result file into a standalone HTML file. This HTML file inlines all necessary assets, allowing it to be opened directly in a browser without a server or network connection.

**Syntax:**
```bash
simpleaudit export-html <json_path> [-o OUTPUT]
```

**Arguments:**

| Argument/Flag | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `json_path` | `str` | Yes | Path to the input JSON file containing audit results. |
| `-o`, `--output` | `str` | No | Path for the output HTML file. If omitted, the output path is derived from `json_path` by replacing the `.json` extension with `.html`. |

**Behavior:**
*   The command imports `export_standalone_html` from `simpleaudit.visualization.server`.
*   **Output Path Logic**:
    *   If `-o` is provided, it is used as the output path.
    *   If `-o` is omitted:
        *   If `json_path` contains a dot (`.`), the extension is replaced with `.html` (e.g., `results.json` becomes `results.html`).
        *   If `json_path` does not contain a dot, `.html` is appended to the filename.
*   **Error Handling**:
    *   If the input `json_path` does not exist, the CLI prints `Error: <json_path> does not exist` and exits with status code `1`.
    *   If a `ValueError` occurs during processing (e.g., invalid JSON structure), the CLI prints the error message and exits with status code `1`.
*   On success, the CLI prints the path to the generated HTML file and a note that it can be opened directly in a browser.

**Example Usage:**
```bash
# Export a specific JSON file to a default HTML name
simpleaudit export-html ./results/audit_01.json

# Export to a custom output filename
simpleaudit export-html ./results/audit_01.json -o ./reports/audit_01_report.html
```

### Error Handling and Exit Codes

The CLI uses standard exit codes to indicate success or failure:

*   **0**: Success (implicit, though not explicitly shown in the snippet, standard Python behavior).
*   **1**: Failure. This occurs in the following scenarios:
    *   No command is provided or an invalid command is used.
    *   Required optional dependencies for visualization are missing.
    *   The input JSON file for `export-html` is not found.
    *   A `ValueError` is raised during HTML export.

**Missing Dependencies Example:**
If the `visualize` extra is not installed, running either command results in:
```text
Error: the visualization server needs optional dependencies (module_name is missing).
Install them with: pip install 'simpleaudit[visualize]'
```

### Implementation Details

The CLI is defined in `simpleaudit/cli.py`. The main entry point is the `main()` function, which uses `argparse` to parse arguments.

*   **Parser Setup**: The `argparse.ArgumentParser` is initialized with `prog="simpleaudit"` and a description.
*   **Subparsers**: Two subparsers are added: `serve` and `export-html`.
*   **Lazy Imports**: Visualization modules (`simpleaudit.visualization.server`) are imported only when the corresponding command is executed. This ensures that the core CLI remains functional even if visualization dependencies are not installed, providing clear error messages instead of import failures at startup.

### Best Practices

1.  **Explicit Results Directory**: When using `serve`, always specify `--results_dir` to avoid the warning message and ensure the correct directory is scanned.
2.  **Standalone Reports**: Use `export-html` for sharing results with stakeholders who do not have Python environments or access to the original JSON files. The generated HTML is self-contained.
3.  **Dependency Management**: Ensure `simpleaudit[visualize]` is installed in environments where visualization features are required.

### Troubleshooting

**Issue: "Error: the visualization server needs optional dependencies"**
*   **Cause**: The `simpleaudit` package was installed without the `visualize` extra.
*   **Solution**: Run `pip install 'simpleaudit[visualize]'`.

**Issue: "Error: <path> does not exist"**
*   **Cause**: The `json_path` provided to `export-html` is incorrect or the file was moved/deleted.
*   **Solution**: Verify the file path and ensure the JSON file exists at the specified location.

**Issue: Server does not start**
*   **Cause**: The specified port might already be in use, or the host interface is unavailable.
*   **Solution**: Try a different port using `--port` or check firewall/network settings if binding to a non-localhost host.