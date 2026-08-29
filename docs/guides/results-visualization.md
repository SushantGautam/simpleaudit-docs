## Results and Visualization

The `simpleaudit` library provides a lightweight FastAPI-based visualization server to interpret audit results. This subsystem allows developers to inspect generated JSON reports, compare model performance across experiments, and export self-contained HTML files for sharing. The server is designed to be secure, supporting optional authentication, and efficient, handling large result sets without blocking the event loop.

### Core Components

The visualization logic resides in `visualization/server.py`. The primary entry point is the `app` object, a `FastAPI` instance titled "SimpleAudit Visualizer".

Key features include:
1.  **File Tree Discovery**: Recursively scans a configured results directory to identify valid audit JSON files.
2.  **Data Validation**: Ensures only files matching the expected audit schema are served.
3.  **Authentication**: Optional secret-based access control via environment variables.
4.  **Standalone Export**: Generates self-contained HTML files with inlined data for offline viewing or sharing.

### Configuration

The server behavior is controlled via environment variables. These should be set before starting the server.

| Variable | Description | Default |
| :--- | :--- | :--- |
| `SIMPLEAUDIT_VISUALIZER_SECRET` | A string secret required in the `X-Secret` header for API access. If empty, authentication is disabled. | `""` (disabled) |
| `SIMPLEAUDIT_VISUALIZER_EMAIL` | The contact email displayed in the frontend when authentication is enabled. | `"sushant@simula.no"` |

### Data Schema Validation

The server strictly validates JSON structures before serving them. A file is considered valid if it matches one of the following shapes:

1.  **Legacy List**: A JSON array where every item is a dictionary containing `scenario_name` (or `name`) and `severity`.
2.  **Current Single Run**: A JSON object with a `results` key containing a non-empty list of audit result dictionaries.
3.  **Multi-Model Experiment**: A JSON object with a `runs` key, which maps model labels to lists of run objects. Each run object must contain a `results` list of valid audit results.

The helper function `_looks_like_audit_result` checks for the presence of `scenario_name`/`name` and `severity` keys. The function `_experiment_models` extracts valid model labels from experiment files.

### API Endpoints

The server exposes the following HTTP endpoints. All `/api/*` endpoints (except `/api/auth`) require authentication if a secret is configured.

#### `GET /`
Serves the main visualization HTML page (`visualizer.html`).

#### `GET /scenario_viewer.html`
Serves the standalone scenario viewer page.

#### `GET /api/auth`
Checks the authentication status.
*   **Response**: JSON object with `ok` (bool), `enabled` (bool), and `contact_email` (string).
*   **Behavior**: If a secret is configured, the request must include a valid `X-Secret` header. If no secret is configured, it returns `enabled: false`.

#### `GET /api/files`
Returns the file tree of valid JSON files in the results directory.
*   **Response**: JSON object with a `tree` key containing a nested list of folders and files.
*   **Note**: This endpoint is synchronous (`def`) to allow FastAPI to run it in a threadpool, preventing blocking of the event loop during file I/O.

#### `GET /api/json/{file_path:path}`
Retrieves the contents of a specific JSON file.
*   **Path Parameter**: `file_path` – The relative path to the JSON file within the results directory.
*   **Security**: Performs path traversal protection by resolving the real path and ensuring it starts with the results directory root.
*   **Response**: The raw JSON content of the file.
*   **Errors**:
    *   `400`: Invalid path or not a JSON file.
    *   `403`: Access denied (path traversal attempt).
    *   `404`: File not found.

### Running the Server

To start the visualization server, you must specify the results directory. The `RESULTS_DIR` global variable in `server.py` must be set before the server starts. Typically, this is done via a command-line argument or environment variable in the startup script.

Example startup command (assuming a standard `uvicorn` invocation):

```bash
# Set the results directory and secret
export SIMPLEAUDIT_RESULTS_DIR="/path/to/your/results"
export SIMPLEAUDIT_VISUALIZER_SECRET="your-secret-key"

# Start the server
uvicorn simpleaudit.visualization.server:app --host 0.0.0.0 --port 8000
```

*Note: The provided source code snippet shows `RESULTS_DIR` as a global variable initialized to `None`. In a full implementation, this is likely populated from an environment variable or CLI argument during the `main()` or startup event. Ensure your deployment script sets this correctly.*

### Standalone HTML Export

For sharing results without running a server, you can generate a self-contained HTML file. This file includes the visualizer template and the audit data inlined as a JavaScript variable.

#### Function: `export_standalone_html`

```python
def export_standalone_html(json_path: str, output_path: str) -> str
```

*   **`json_path`**: Path to the valid audit results JSON file.
*   **`output_path`**: Where to write the standalone HTML file.
*   **Returns**: The output path.
*   **Raises**: `ValueError` if the JSON is not valid audit data or if the template is malformed.

**Usage Example:**

```python
from simpleaudit.visualization.server import export_standalone_html

# Export a single run result
export_standalone_html(
    json_path="results/run_1.json",
    output_path="shared_results.html"
)

# The resulting HTML file can be opened directly in a browser
# or emailed to colleagues. No server is required.
```

The exported HTML detects the inlined data on load and renders it immediately. It supports both single-run and multi-model experiment data.

### Security Considerations

1.  **Authentication**: If `SIMPLEAUDIT_VISUALIZER_SECRET` is set, all API requests must include the header `X-Secret: <secret>`. The comparison uses `secrets.compare_digest` to prevent timing attacks.
2.  **Path Traversal**: The `/api/json/{file_path:path}` endpoint strictly validates that the requested file is within the `RESULTS_DIR`. Attempts to access files outside this directory result in a `403 Forbidden` error.
3.  **XSS Prevention**: When inlining data into standalone HTML, the JSON payload is escaped to prevent breaking out of the `<script>` tag. Specifically, `<`, `\u2028`, and `\u2029` characters are escaped.

### Troubleshooting

*   **"Results directory not set"**: Ensure `RESULTS_DIR` is correctly initialized in the server context.
*   **"Unauthorized" (401)**: Verify that the `X-Secret` header matches the `SIMPLEAUDIT_VISUALIZER_SECRET` environment variable.
*   **Empty File Tree**: Ensure the results directory contains valid JSON files matching the audit schema. Invalid files are silently ignored.

### See Also

*   [Cross-Judging and Validation](cross-judging.md)
*   [Architecture](architecture.md)
*   [Vision and Image Integrity](vision-integrity.md)
