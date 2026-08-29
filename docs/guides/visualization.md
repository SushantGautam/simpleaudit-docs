## Visualization & Reporting

The `simpleaudit.visualization` module provides a local web interface for exploring audit results and generating standalone reports. It consists of a FastAPI-based server (`server.py`) that serves an interactive visualizer, and utility functions for exporting self-contained HTML files. This subsystem allows developers to inspect detailed audit findings, compare multi-model experiment runs, and share results without requiring the recipient to have the `simpleaudit` library installed.

### Architecture Overview

The visualization subsystem is built on **FastAPI** and **Uvicorn**. The core components are:

1.  **`server.py`**: The main application logic. It defines the `FastAPI` app, handles authentication, scans the file system for result JSONs, and provides API endpoints for the frontend.
2.  **`visualizer.html`**: The frontend template. It is a single-page application that fetches data from the API endpoints and renders charts and tables.
3.  **`scenario_viewer.html`**: A standalone viewer for specific scenarios, accessible via a dedicated route.

### Configuration

The server behavior is controlled via environment variables. These must be set before starting the server.

| Variable | Description | Default |
| :--- | :--- | :--- |
| `SIMPLEAUDIT_VISUALIZER_SECRET` | A secret key required for API access. If set, all API requests must include an `X-Secret` header with this value. If empty or unset, authentication is disabled. | `""` (Disabled) |
| `SIMPLEAUDIT_VISUALIZER_EMAIL` | The contact email displayed in the frontend when authentication is enabled. | `"sushant@simula.no"` |

### Starting the Server

The server is not started automatically by the `simpleaudit` library. You must launch it manually using `uvicorn`.

```bash
# Set the results directory (path to your JSON output files)
export SIMPLEAUDIT_RESULTS_DIR="/path/to/your/results"

# Optional: Enable authentication
export SIMPLEAUDIT_VISUALIZER_SECRET="your-secret-key"

# Start the server
uvicorn simpleaudit.visualization.server:app --host 0.0.0.0 --port 8000
```

Once running, navigate to `http://localhost:8000` in your browser.

### API Endpoints

The server exposes several REST endpoints that the frontend uses to interact with the data.

#### `GET /`
Serves the main visualization interface (`visualizer.html`).

#### `GET /scenario_viewer.html`
Serves the standalone scenario viewer page.

#### `GET /favicon.png`
Serves the application favicon (`thumbnail.png`).

#### `GET /api/auth`
Checks the authentication status.
*   **Response**: `{"ok": true, "enabled": bool, "contact_email": str}`
*   If `enabled` is `true`, the frontend will prompt for the secret key.

#### `GET /api/files`
Returns a recursive tree structure of all valid JSON files in the configured results directory.
*   **Authentication**: Requires `X-Secret` header if enabled.
*   **Response**: A JSON object containing a `tree` list. Each item represents a folder or file.
    *   `type`: `"folder"`, `"file"`, or `"experiment"`.
    *   `path`: Relative path from the results root.
    *   `models`: (Only for `"experiment"` type) List of model labels found in the file.

#### `GET /api/json/{file_path}`
Retrieves the raw JSON content of a specific result file.
*   **Authentication**: Requires `X-Secret` header if enabled.
*   **Path Traversal Protection**: The server validates that the requested path stays within the results directory.
*   **Response**: The raw JSON content of the file.

### Data Format Validation

The server validates JSON files to ensure they contain valid SimpleAudit results before listing them. A file is considered valid if it matches one of the following structures:

1.  **Single Run**: A list of audit result objects.
    ```json
    [
      {"scenario_name": "test", "severity": "high", "details": "..."},
      {"scenario_name": "test2", "severity": "low", "details": "..."}
    ]
    ```
2.  **Structured Single Run**: A dictionary with a `results` key.
    ```json
    {
      "results": [
        {"scenario_name": "test", "severity": "high", "details": "..."}
      ]
    }
    ```
3.  **Multi-Model Experiment**: A dictionary with a `runs` key, where each value is a list of run objects containing `results`.
    ```json
    {
      "runs": {
        "model_a": [
          {"results": [{"scenario_name": "test", "severity": "high"}]}
        ],
        "model_b": [
          {"results": [{"scenario_name": "test", "severity": "low"}]}
        ]
      }
    }
    ```

The function `is_valid_audit_data` in `server.py` performs this validation. Files that do not match these schemas are ignored by the file tree endpoint.

### Standalone HTML Export

For sharing results with stakeholders who do not have access to the local server, you can generate a self-contained HTML file. This file embeds the JSON data directly into the HTML, allowing it to be opened in any browser without a server.

#### `export_standalone_html(json_path, output_path)`

Creates a standalone HTML file with audit results inlined.

**Parameters:**
*   `json_path` (str): Path to the audit results JSON file.
*   `output_path` (str): Where to write the standalone HTML file.

**Returns:**
*   `str`: The output path.

**Raises:**
*   `ValueError`: If the JSON is not valid audit data or if the template is missing.

**Example Usage:**

```python
from simpleaudit.visualization.server import export_standalone_html

# Export a single run
export_standalone_html(
    json_path="results/run_1.json",
    output_path="reports/run_1.html"
)

# Export a multi-model experiment
export_standalone_html(
    json_path="results/experiment_1.json",
    output_path="reports/experiment_1.html"
)
```

The generated HTML file includes the data in a `<script>` tag as `window.__inlinedData`. The visualizer detects this on load and renders the data immediately, bypassing the need for API calls.

### Security Considerations

*   **Authentication**: When `SIMPLEAUDIT_VISUALIZER_SECRET` is set, all API endpoints (except `/api/auth` for the initial check) require the `X-Secret` header. The comparison is performed in constant time to prevent timing attacks.
*   **Path Traversal**: The `/api/json/{file_path}` endpoint uses `os.path.realpath` and checks that the resolved path starts with the results directory root to prevent access to files outside the designated directory.
*   **XSS Protection**: When generating standalone HTML, the JSON payload is escaped to prevent breaking out of the `<script>` tag (e.g., `<` is replaced with `\u003c`).

### Troubleshooting

*   **"Results directory not set"**: Ensure the `SIMPLEAUDIT_RESULTS_DIR` environment variable is set and points to a valid directory before starting the server.
*   **"Unauthorized"**: If you see a 401 error, ensure you are sending the `X-Secret` header with the correct value. You can test the connection using:
    ```bash
    curl -H "X-Secret: your-secret-key" http://localhost:8000/api/files
    ```
*   **Empty File Tree**: If the file tree is empty, verify that your JSON files match the expected data formats described in the [Data Format Validation](#data-format-validation) section.

### See Also

*   [Advanced Analysis & Meta-Evaluation](advanced-analysis.md)
