## Visualization and Reporting

The `simpleaudit` library includes a built-in FastAPI-based visualization server that allows developers to inspect audit results, compare model runs, and generate self-contained HTML reports. This subsystem is located in `visualization/server.py` and provides both an interactive web interface and programmatic utilities for exporting static reports.

### Overview

The visualization server serves two primary purposes:
1.  **Interactive Exploration**: It scans a directory of JSON audit results and provides a web-based file tree and viewer. This allows users to navigate through complex experiment structures and view detailed audit metrics without opening raw JSON files.
2.  **Standalone Reporting**: It provides a utility to embed audit results directly into an HTML file, creating a self-contained artifact that can be shared via email or archived without requiring a running server.

The server is built on **FastAPI** and **Uvicorn**. It supports optional authentication via a shared secret and includes safety checks to prevent directory traversal attacks when serving files.

### Starting the Server

To visualize results, you must first specify the directory containing your audit JSON files. The server exposes a global `RESULTS_DIR` variable that must be set before the application starts serving requests.

```python
import simpleaudit.visualization.server as server
import uvicorn

# Point the server to your results directory
server.RESULTS_DIR = "/path/to/your/audit/results"

# Start the server (usually in a separate process or thread)
uvicorn.run(server.app, host="0.0.0.0", port=8000)
```

Once running, navigate to `http://localhost:8000` in your browser. The root endpoint (`/`) serves the main visualization interface (`visualizer.html`).

### Configuration

The server behavior is controlled by environment variables. These should be set before importing or running the server module.

| Environment Variable | Default | Description |
| :--- | :--- | :--- |
| `SIMPLEAUDIT_VISUALIZER_SECRET` | `""` (Empty) | If set, all API requests (except `/api/auth`) must include an `X-Secret` header matching this value. If empty, authentication is disabled. |
| `SIMPLEAUDIT_VISUALIZER_EMAIL` | `"sushant@simula.no"` | The contact email displayed in the frontend when authentication is enabled. |

### Authentication

When `SIMPLEAUDIT_VISUALIZER_SECRET` is configured, the server enforces authentication on all data endpoints.

1.  **Check Status**: The frontend calls `GET /api/auth` to determine if authentication is required.
    *   If `enabled` is `true`, the user must provide the secret.
    *   The response includes the `contact_email` for support inquiries.
2.  **API Requests**: All subsequent requests to `/api/files` or `/api/json/{path}` must include the header:
    ```http
    X-Secret: <your_secret_value>
    ```
    The server uses `secrets.compare_digest` to perform a constant-time comparison, preventing timing attacks.

### API Endpoints

The server exposes the following REST API endpoints:

#### `GET /api/files`
Retrieves the hierarchical structure of JSON files in the `RESULTS_DIR`.

*   **Response**: A JSON object containing a `tree` array. Each item in the tree represents a folder, a single audit result file, or a multi-model experiment.
    ```json
    {
      "tree": [
        {
          "name": "experiment_1",
          "type": "experiment",
          "path": "experiment_1",
          "models": ["model_a", "model_b"]
        },
        {
          "name": "single_run.json",
          "type": "file",
          "path": "single_run.json"
        }
      ]
    }
    ```
*   **Validation**: The server parses each JSON file to determine its type. Files are classified as:
    *   `file`: A standard list of audit results or a dict with a `results` key.
    *   `experiment`: A dict with a `runs` key containing multiple model outputs.
    *   Invalid or unreadable JSON files are excluded from the tree.

#### `GET /api/json/{file_path}`
Retrieves the raw JSON content of a specific audit result file.

*   **Path Parameter**: `file_path` is the relative path from `RESULTS_DIR` as provided by `/api/files`.
*   **Security**: The server resolves the path and ensures it remains within the `RESULTS_DIR` to prevent directory traversal (e.g., `../../etc/passwd`).
*   **Response**: The raw JSON content of the file.

#### `GET /`
Serves the main HTML visualization interface.

#### `GET /scenario_viewer.html`
Serves a standalone scenario viewer page, useful for inspecting specific scenario definitions.

### Generating Standalone HTML Reports

For sharing results without running a server, use the `export_standalone_html` function. This function takes an audit results JSON file and generates a self-contained HTML file with the data inlined.

#### Function Signature

```python
def export_standalone_html(json_path: str, output_path: str) -> str
```

*   **`json_path`**: The path to the source audit results JSON file.
*   **`output_path`**: The path where the standalone HTML file will be written.
*   **Returns**: The `output_path` string.
*   **Raises**: `ValueError` if the JSON file does not contain valid SimpleAudit results or if the template is missing.

#### How It Works

1.  The function loads and validates the JSON data using `is_valid_audit_data`.
2.  It reads the `visualizer.html` template.
3.  It serializes the JSON data and escapes special characters (`<`, `\u2028`, `\u2029`) to prevent breaking the HTML structure.
4.  It injects the data into a `<script>` tag within the HTML head:
    ```html
    <script>
      window.__inlinedData = {...};
      window.__inlinedName = "filename";
      window.__standaloneMode = "single"; // or "experiment"
    </script>
    ```
5.  The visualizer JavaScript detects this inlined data on load and renders the results immediately, bypassing the need for API calls.

#### Example Usage

```python
from simpleaudit.visualization.server import export_standalone_html

# Generate a shareable report
export_standalone_html(
    json_path="results/run_20231027.json",
    output_path="reports/run_20231027.html"
)

print("Standalone report generated at reports/run_20231027.html")
```

### Data Validation

The server strictly validates the structure of audit results before displaying them. The `is_valid_audit_data` function checks for three supported formats:

1.  **Legacy List**: A JSON list where every item is a dict containing `scenario_name` (or `name`) and `severity`.
2.  **Current Dict**: A JSON dict with a `results` key containing a list of audit result dicts.
3.  **Experiment Dict**: A JSON dict with a `runs` key. Each value in `runs` must be a list (or single dict) of run objects, each containing a non-empty `results` list of valid audit result dicts.

Files that do not match these structures are ignored by the file tree and cannot be served via the JSON endpoint. This ensures the visualization interface only displays meaningful audit data.

### Best Practices

*   **Large Datasets**: The server reads and parses JSON files synchronously. For very large result directories, consider filtering the `RESULTS_DIR` to only include recent or relevant experiments to reduce initial load time.
*   **Security**: If deploying the visualizer on a network accessible by others, always set `SIMPLEAUDIT_VISUALIZER_SECRET`. Do not rely on network isolation alone.
*   **Reporting**: Use `export_standalone_html` for final reports. It ensures the visualizer version and data are locked together, preventing discrepancies if the library is updated later.

### See Also

*   [Implementing Judges](implementing-judges.md)
