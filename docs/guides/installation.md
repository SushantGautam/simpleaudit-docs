## Installation

SimpleAudit is a lightweight Python framework designed for AI safety auditing. It provides tools to systematically evaluate model behavior, detect alignment issues, and generate comprehensive audit reports. This guide covers the installation requirements, methods, and verification steps for setting up SimpleAudit in your development environment.

### Prerequisites

Before installing SimpleAudit, ensure your development environment meets the following requirements:

1.  **Python Version**: SimpleAudit requires **Python 3.11** or later. The library leverages modern Python features such as `match` statements, improved type hints, and the updated standard library modules available in Python 3.11+.
    *   *Note*: Support for Python 3.10 and earlier is not guaranteed and may result in runtime errors due to missing syntax or standard library features.
2.  **Operating System**: SimpleAudit is cross-platform and supports:
    *   Linux (Ubuntu 20.04+, CentOS 8+, etc.)
    *   macOS (11.0+)
    *   Windows 10/11
3.  **Package Manager**: `pip` version 22.0 or later is recommended for dependency resolution.

### Installation via PyPI

The recommended method for most users is to install SimpleAudit directly from the Python Package Index (PyPI). This method automatically resolves and installs all necessary dependencies.

#### Standard Installation

To install the latest stable version of SimpleAudit, open your terminal or command prompt and run:

```bash
pip install simpleaudit
```

#### Installing with Development Dependencies

If you are contributing to SimpleAudit or wish to run the test suite and build documentation locally, you can install the package with its development dependencies:

```bash
pip install simpleaudit[dev]
```

This extra dependency group includes tools such as `pytest`, `black`, `mypy`, and `sphinx`.

#### Installing a Specific Version

To pin a specific version of SimpleAudit for reproducibility in production environments:

```bash
pip install simpleaudit==1.0.0
```

Replace `1.0.0` with the desired version number found in the [Release Notes](https://pypi.org/project/simpleaudit/#history).

### Installation from Source

Installing from source is recommended for developers who wish to contribute to the project, test unreleased features, or debug the library internals.

#### 1. Clone the Repository

First, clone the SimpleAudit repository from GitHub:

```bash
git clone https://github.com/simpleaudit/simpleaudit.git
cd simpleaudit
```

#### 2. Create a Virtual Environment

It is best practice to use a virtual environment to isolate dependencies.

**For Linux/macOS:**

```bash
python3 -m venv venv
source venv/bin/activate
```

**For Windows:**

```bash
python -m venv venv
venv\Scripts\activate
```

#### 3. Install in Editable Mode

Use `pip` to install the package in editable (development) mode. This allows you to modify the source code and have changes reflected immediately without reinstalling:

```bash
pip install -e .
```

If you need development tools:

```bash
pip install -e .[dev]
```

#### 4. Build Extensions (If Applicable)

While SimpleAudit is primarily pure Python, certain performance-critical components may use Cython or C extensions. If the `setup.py` or `pyproject.toml` indicates native extensions, ensure you have a C compiler installed (e.g., `gcc` on Linux, `Xcode Command Line Tools` on macOS, or `Visual Studio Build Tools` on Windows).

The installation command above (`pip install -e .`) will automatically handle building these extensions if required.

### Verifying the Installation

After installation, verify that SimpleAudit is correctly installed and accessible by running the following command in your Python environment:

```python
import simpleaudit

print(simpleaudit.__version__)
```

If the installation was successful, this command will print the version number of the installed SimpleAudit package (e.g., `1.0.0`). If you encounter a `ModuleNotFoundError`, ensure that your virtual environment is activated and that the package was installed in the correct environment.

### Troubleshooting

#### Python Version Mismatch

If you encounter errors related to syntax or missing modules, verify your Python version:

```bash
python --version
```

If the version is below 3.11, upgrade your Python installation or use a version manager like `pyenv` to switch to a compatible version:

```bash
pyenv install 3.11.0
pyenv local 3.11.0
```

#### Dependency Conflicts

If `pip` reports dependency conflicts, try upgrading `pip` first:

```bash
pip install --upgrade pip
```

Then, attempt the installation again. If conflicts persist, consider creating a fresh virtual environment to avoid conflicts with existing packages.

#### Permission Errors

If you encounter permission errors when installing via `pip` (common on Linux systems where system-wide Python is used), avoid using `sudo`. Instead, use a virtual environment or install to the user site-packages directory:

```bash
pip install --user simpleaudit
```

### Uninstallation

To uninstall SimpleAudit from your environment:

```bash
pip uninstall simpleaudit
```

If you installed from source in editable mode, you may also need to remove the source directory if you no longer require the development setup.

### Next Steps

Once SimpleAudit is installed, you can proceed to:

1.  **Initialize an Audit Project**: Create a new audit configuration file.
2.  **Define Audit Criteria**: Specify the safety metrics and thresholds for your AI model.
3.  **Run Audits**: Execute the auditing framework against your target model.

Refer to the [Quick Start Guide](./quickstart.md) for a step-by-step tutorial on running your first audit. For detailed API reference, see the [API Documentation](./api.md).

### See Also

*   [Quickstart](quickstart.md)
*   [Local Model Setup](local-model-setup.md)
