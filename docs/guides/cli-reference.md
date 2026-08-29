## CLI Reference

The `simpleaudit` library provides a Command Line Interface (CLI) for visualizing AI safety audit results. The CLI is built using `argparse` and serves as the entry point for users who prefer terminal-based workflows to inspect audit data.

This section documents the available commands, flags, and configuration options. The CLI is designed to be modular, allowing users to serve visualizations of historical data without writing Python scripts.

### Installation and Entry Point

Ensure the library is installed in your environment:

```bash
pip install simpleaudit
```

The main entry point is accessible via the `simpleaudit` command. If you are running from a source checkout, you may need to use `python -m simpleaudit`.

### Commands

The CLI supports one primary command: `serve`.

#### 1. `serve`

Starts a web server to visualize audit results. This command loads JSON result files from a specified directory and serves them via a local web interface.

**Usage:**

```bash
simpleaudit serve [options]
```

**Flags:**

*   `--results_dir`: Directory containing JSON result files to visualize. If not specified, the current directory (`.`) is used.
*   `--port`: Port to run the server on. Default is `8000`.
*   `--host`: Host to bind the server to. Default is `127.0.0.1`.

**Example:**

```bash
# Start server on default port using current directory
simpleaudit serve

# Start server on a specific port and host, pointing to a specific results directory
simpleaudit serve --results_dir ./audit_results --port 8080 --host 0.0.0.0
```

**Behavior:**

*   If `--results_dir` is not specified, the CLI defaults to the current directory (`.`) and prints a warning recommending explicit specification to avoid confusion.
*   The server binds to the specified host and port. By default, it is accessible only via localhost (`127.0.0.1`).

### Error Handling

The CLI exits with non-zero status codes to indicate failure, which is useful for CI/CD pipelines or scripting.

*   `0`: Success.
*   `1`: Usage error (e.g., no command provided, or invalid arguments).

**Example in Scripting:**

```bash
#!/bin/bash
simpleaudit serve --results_dir ./results &
SERVER_PID=$!

# ... perform checks or wait ...

kill $SERVER_PID
```

### Troubleshooting

*   **"Port already in use"**: Ensure the port specified in `--port` is not already occupied by another process. You can change the port using `--port 8080`.
*   **"No results found"**: Ensure the `--results_dir` points to a directory containing valid JSON audit result files.
*   **"Permission denied"**: Ensure you have read permissions to the `--results_dir` and write permissions to the system's temporary directories if the server requires them.

For advanced use cases, such as custom audit modules or programmatic control, refer to the [Python API Reference](../api/index.md).

### See Also

*   [Quickstart](quickstart.md)
