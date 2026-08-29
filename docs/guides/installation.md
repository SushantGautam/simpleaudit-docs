## Installation

SimpleAudit is a lightweight Python library designed to streamline the process of auditing data structures, ensuring consistency, and generating detailed reports on data integrity. It is built for developers who need robust, yet simple, tools to validate and monitor data pipelines without the overhead of complex enterprise frameworks.

This page provides instructions for installing SimpleAudit, including system requirements, installation methods via PyPI and from source, and verification steps to ensure the library is correctly configured in your environment.

### System Requirements

Before installing SimpleAudit, ensure your development environment meets the following prerequisites:

*   **Python Version:** SimpleAudit requires **Python 3.11** or later. The library leverages modern Python features such as `match` statements, improved type hinting, and enhanced error messages available in Python 3.11+.
*   **Operating System:** Compatible with Linux, macOS, and Windows.
*   **Dependencies:** SimpleAudit has minimal external dependencies. It relies on standard library modules for core functionality. If you are using the optional reporting features, `jinja2` may be required for template rendering.

> **Note:** If you are using Python 3.10 or earlier, SimpleAudit will not function correctly. Please upgrade your Python environment before proceeding.

### Installing via PyPI

The recommended method for installing SimpleAudit is via the Python Package Index (PyPI). This ensures you receive the latest stable release with all dependencies resolved automatically.

#### Using `pip`

Open your terminal or command prompt and run the following command:

```bash
pip install simpleaudit
```

If you are using a virtual environment, ensure it is activated before running the command. For example, if you are using `venv`:

```bash
python -m venv myenv
source myenv/bin/activate  # On Windows: myenv\Scripts\activate
pip install simpleaudit
```

#### Using `pipx`

If you prefer to install SimpleAudit as an isolated application (useful for CLI tools), you can use `pipx`:

```bash
pipx install simpleaudit
```

### Installing from Source

Installing from source is recommended for developers who wish to contribute to the project, test the latest development version, or debug issues that may not be present in the released PyPI package.

#### Prerequisites

To install from source, you will need:

1.  **Git** installed on your system.
2.  **A Python 3.11+ environment** with `pip` and `setuptools` installed.
3.  **A C compiler** (e.g., `gcc` on Linux, `Xcode Command Line Tools` on macOS, or `Visual Studio Build Tools` on Windows) if any native extensions are being compiled.

#### Steps

1.  **Clone the repository:**

    ```bash
    git clone https://github.com/your-org/simpleaudit.git
    cd simpleaudit
    ```

2.  **Create and activate a virtual environment:**

    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows: venv\Scripts\activate
    ```

3.  **Install the package in editable mode:**

    Using editable mode (`-e`) allows you to make changes to the source code and have them reflected immediately without reinstalling the package.

    ```bash
    pip install -e .
    ```

    If you wish to install with development dependencies (such as testing frameworks and linters), use:

    ```bash
    pip install -e ".[dev]"
    ```

4.  **Verify the installation:**

    Run the following command to ensure the package is installed correctly:

    ```bash
    python -c "import simpleaudit; print(simpleaudit.__version__)"
    ```

    You should see the version number printed to the console. If no error is raised, the installation was successful.

### Verifying the Installation

After installation, it is good practice to verify that SimpleAudit is accessible and functioning as expected. You can run a simple smoke test using the Python interpreter.

```python
import simpleaudit

# Check the version
print(f"SimpleAudit Version: {simpleaudit.__version__}")

# Basic usage example
data = {"name": "Alice", "age": 30, "email": "alice@example.com"}
audit_result = simpleaudit.audit(data, schema={"name": str, "age": int, "email": str})

if audit_result.is_valid:
    print("Data passed audit.")
else:
    print("Data failed audit.")
    for error in audit_result.errors:
        print(f"  - {error.field}: {error.message}")
```

If the script runs without errors and outputs the version number and audit results, your installation is complete.

### Upgrading and Uninstalling

#### Upgrading

To upgrade to the latest version of SimpleAudit, use:

```bash
pip install --upgrade simpleaudit
```

If you are using a source installation, pull the latest changes and reinstall:

```bash
git pull
pip install -e .
```

#### Uninstalling

To remove SimpleAudit from your environment:

```bash
pip uninstall simpleaudit
```

### Troubleshooting

#### `ModuleNotFoundError: No module named 'simpleaudit'`

This error typically occurs if:

1.  You installed the package in a different Python environment than the one you are using.
2.  You did not activate your virtual environment before running the script.
3.  You are using a different Python interpreter than the one used for installation.

**Solution:** Ensure you are using the correct Python interpreter. You can verify this by running:

```bash
which python  # On Linux/macOS
where python  # On Windows
```

and comparing it with the interpreter used during installation.

#### `ImportError: cannot import name 'audit' from 'simpleaudit'`

This may occur if you are using an outdated version of the library or if the installation was incomplete.

**Solution:** Reinstall the package:

```bash
pip install --force-reinstall simpleaudit
```

#### Permission Errors

If you encounter permission errors when installing via `pip`, try using the `--user` flag to install the package in your user directory:

```bash
pip install --user simpleaudit
```

Alternatively, consider using a virtual environment to avoid permission issues.

### Next Steps

Now that you have SimpleAudit installed, you can proceed to:

*   [Getting Started](getting-started.md): Learn how to define schemas and run your first audit.
*   [API Reference](api-reference.md): Explore the full API, including classes, functions, and parameters.
*   [Configuration Guide](configuration.md): Learn how to customize audit behavior and reporting.

For more information, visit the [SimpleAudit GitHub Repository](https://github.com/your-org/simpleaudit) or join the community discussion on [GitHub Discussions](https://github.com/your-org/simpleaudit/discussions).

### See Also

*   [Quickstart](quickstart.md)
