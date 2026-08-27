## Visualization

The `simpleaudit.visualization` module provides an interactive FastAPI server for exploring, analyzing, and sharing audit results. It renders a web-based dashboard that allows users to navigate result directories, view severity distributions, and inspect individual scenario outcomes. The system supports both live server mode and static HTML export for offline sharing.

### Architecture

The visualization subsystem is centered around `simpleaudit/visualization/server.py`. It uses **FastAPI** for the backend API and **Uvicorn** as the ASGI server. The frontend consists of static HTML templates (`visualizer.html`, `scenario_viewer.html`) served directly by the application.

Key components:
*   **File Tree API**: Recursively scans a results directory to build a hierarchical view of JSON files.
*   **Data Validation**: Ensures only valid SimpleAudit result structures are served.
*   **Authentication**: Optional secret-based authentication via environment variables.
*   **Standalone Export**: Generates self-contained HTML files with inlined data for offline use.

### Configuration

Configuration is handled via environment variables:

| Variable | Description | Default |
| :--- | :--- | :--- |
| `SIMPLEAUDIT_VISUALIZER_SECRET` | If set, enables authentication. Requests must include this value in the `X-Secret` header. | `""` (disabled) |
| `SIMPLEAUDIT_VISUALIZER_EMAIL` | Contact email displayed in the frontend when authentication is enabled. | `"sushant@simula.no"` |

### Starting the Server

The server is initialized by setting the `RESULTS_DIR` global variable and running the Uvicorn server.

```python
import uvicorn
from simpleaudit.visualization.server import app, RESULTS_DIR

# Set the directory containing your audit result JSON files
RESULTS_DIR = "/path/to/your/results"

# Start the server
uvicorn.run(app, host="0.0.0.0", port=8000)
```

Once running, the dashboard is accessible at `http://localhost:8000`.

### API Endpoints

The server exposes the following REST endpoints:

#### `GET /`
Serves the main visualization dashboard HTML.

#### `GET /scenario_viewer.html`
Serves the standalone scenario viewer page.

#### `GET /favicon.png`
Serves the application favicon.

#### `GET /api/auth`
Checks authentication status.
*   **Response**: JSON object with `ok` (bool), `enabled` (bool), and `contact_email` (str).
*   **Auth**: Requires `X-Secret` header if `SIMPLEAUDIT_VISUALIZER_SECRET` is set.

#### `GET /api/files`
Returns the file tree of JSON files in the results directory.
*   **Response**: JSON object with a `tree` key containing a list of folder/file nodes.
*   **Auth**: Requires `X-Secret` header if enabled.

#### `GET /api/json/{file_path}`
Retrieves the contents of a specific JSON result file.
*   **Path Param**: `file_path` - Relative path from the results directory.
*   **Response**: JSON content of the file.
*   **Security**: Prevents directory traversal attacks by validating that the resolved path remains within `RESULTS_DIR`.
*   **Auth**: Requires `X-Secret` header if enabled.

### Data Validation

The server validates JSON structures before serving them to ensure they conform to SimpleAudit result schemas. The function `is_valid_audit_data` checks for three supported shapes:

1.  **Legacy Shape**: A list of audit result objects.
2.  **Current Shape**: A dictionary with a `"results"` key containing a list of audit result objects.
3.  **Experiment Shape**: A dictionary with a `"runs"` key, where each value is a model label mapping to a list of run objects.

An object is considered a valid audit result if it is a dictionary containing either `"scenario_name"` or `"name"`, and a `"severity"` field.

### Standalone HTML Export

For sharing results without running a server, use the `export_standalone_html` function. This creates a self-contained HTML file with the JSON data inlined into a `<script>` tag.

```python
from simpleaudit.visualization.server import export_standalone_html

# Export a single result file
output_path = export_standalone_html(
    json_path="/path/to/results/run1.json",
    output_path="/path/to/output/standalone.html"
)
print(f"Exported to: {output_path}")
```

**Features:**
*   **No Server Required**: The HTML file can be opened directly in a browser.
*   **Security**: JSON payload is escaped to prevent breaking out of the inline script tag.
*   **Mode Detection**: Automatically detects if the data is a single run or a multi-model experiment.

### Authentication

If `SIMPLEAUDIT_VISUALIZER_SECRET` is set, all API endpoints (except `/api/auth` for the initial check) require the `X-Secret` header. The frontend will prompt for the secret and include it in subsequent requests.

Example request with authentication:

```bash
curl -H "X-Secret: your-secret-here" http://localhost:8000/api/files
```

The comparison uses `secrets.compare_digest` to prevent timing attacks.

### File Tree Structure

The `get_file_tree` function recursively scans the results directory. It classifies JSON files into two types:

1.  **`file`**: Standard audit result files.
2.  **`experiment`**: Files containing multi-model run data. These include a `models` list in the tree node, indicating which models have loadable runs.

Folders are only included in the tree if they contain at least one valid JSON file.

### Troubleshooting

*   **401 Unauthorized**: Ensure the `X-Secret` header matches `SIMPLEAUDIT_VISUALIZER_SECRET`.
*   **403 Access Denied**: The requested file path attempts to escape the results directory.
*   **500 Results directory not set**: Ensure `RESULTS_DIR` is set before starting the server.
*   **Empty File Tree**: Verify that the results directory contains valid JSON files with the expected structure.