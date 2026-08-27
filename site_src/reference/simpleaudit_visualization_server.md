## simpleaudit.visualization.server

FastAPI visualization server: browse audit results in a browser.

FastAPI server for visualizing SimpleAudit results.

### Functions

#### is_valid_audit_data

```python
def is_valid_audit_data(data)
```

Check whether parsed JSON has the shape of audit results.

#### is_valid_audit_json

```python
def is_valid_audit_json(file_path)
```

Check if a JSON file contains valid audit results.

Args:
    file_path: Full path to the JSON file

Returns:
    True if the file contains valid audit results, False otherwise

#### get_file_tree

```python
def get_file_tree(directory, base_path)
```

Recursively get the file tree structure for JSON files.

Args:
    directory: Full path to scan
    base_path: Relative path from the root results directory

Returns:
    List of dicts representing folders and JSON files

#### root

```python
async def root()
```

Serve the main visualization page.

#### scenario_viewer

```python
async def scenario_viewer()
```

Serve the standalone scenario viewer page.

#### favicon

```python
async def favicon()
```

Serve the favicon.

#### export_standalone_html

```python
def export_standalone_html(json_path, output_path)
```

Create a self-contained HTML file with the audit results inlined.

The output can be opened directly in a browser (or sent to someone) —
no server and no JSON upload step required. The visualizer detects the
inlined data on load and renders it as a custom upload.

Args:
    json_path: Path to the audit results JSON file.
    output_path: Where to write the standalone HTML file.

Returns:
    The output path.

Raises:
    ValueError: If the JSON is not valid audit data.

#### check_secret

```python
def check_secret(request)
```

Raise HTTP 401 if a secret is configured and the request does not
provide the correct value in an X-Secret header.  When no secret is
configured the check is a no-op.

#### auth_check

```python
async def auth_check(request)
```

Endpoint used by the frontend to verify a key and learn if auth
is enabled.  When the server has no secret configured it still
returns 200 but sets ``enabled`` to False.

The response also includes ``contact_email`` which is read from the
``SIMPLEAUDIT_VISUALIZER_EMAIL`` environment variable and defaults
to the original maintainer address.  The frontend uses this to
populate the authentication overlay message.

#### get_files

```python
def get_files()
```

Get the file tree of JSON files in the results directory.

#### get_json_file

```python
def get_json_file(file_path)
```

Get the contents of a specific JSON file.

#### get_image

```python
def get_image(uri)
```

_No docstring._

#### start_server

```python
def start_server(results_dir, host, port)
```

Start the FastAPI server.

Args:
    results_dir: Directory containing JSON result files
    host: Host to bind to
    port: Port to run on

### Constants

- `app`

- `SECRET`

- `CONTACT_EMAIL`

- `RESULTS_DIR`

