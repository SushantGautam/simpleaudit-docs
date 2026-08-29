## Command Line Interface

The `simpleaudit` library provides a robust Command Line Interface (CLI) that allows developers to execute audits, manage data models, and visualize results without writing Python scripts. The CLI is built using `argparse` and is accessible via the `simpleaudit` executable or the `python -m simpleaudit` module invocation.

This reference covers all available subcommands, flags, and options.

### Getting Started

To view the general help message and list of available subcommands, run:

```bash
simpleaudit --help
```

Or equivalently:

```bash
python -m simpleaudit --help
```

The primary entry point is located in `simpleaudit/cli.py`. The CLI is designed to be stateless by default, meaning each command executes independently unless explicitly configured to persist state via configuration files.

### Global Options

These options apply to all subcommands.

| Option | Description |
| :--- | :--- |
| `-h`, `--help` | Show the help message and exit. |
| `-v`, `--verbose` | Enable verbose logging output. |
| `--version` | Print the version of the `simpleaudit` library. |

### Subcommands

The CLI supports three main categories of operations: **Auditing**, **Model Management**, and **Visualization**.

#### 1. Running Audits

The `audit` subcommand is used to execute audit rules against a dataset.

**Usage:**

```bash
simpleaudit audit [OPTIONS] INPUT_FILE
```

**Arguments:**

*   `INPUT_FILE`: Path to the input data file (CSV, JSON, or Parquet).

**Options:**

| Option | Description |
| :--- | :--- |
| `--rules` | Path to the YAML or JSON file containing audit rules. |
| `--output` | Path to save the audit results. Defaults to `stdout` if not specified. |
| `--format` | Output format for results (`json`, `csv`, `table`). Default is `table`. |
| `--fail-on-error` | Exit with a non-zero status code if any audit rule fails. Useful for CI/CD pipelines. |
| `--limit` | Limit the number of rows processed. Useful for testing. |

**Example:**

```bash
# Run audits defined in rules.yaml against data.csv and save results to results.json
simpleaudit audit --rules rules.yaml --output results.json --format json data.csv

# Run audits and fail the script if any errors are found
simpleaudit audit --rules rules.yaml --fail-on-error data.csv
```

#### 2. Managing Models

The `model` subcommand allows you to inspect, validate, and generate schemas for data models.

**Usage:**

```bash
simpleaudit model [COMMAND] [OPTIONS]
```

**Sub-subcommands:**

##### `model validate`

Validates a model definition file against the schema.

```bash
simpleaudit model validate MODEL_FILE
```

*   `MODEL_FILE`: Path to the model definition file (YAML).

**Example:**

```bash
simpleaudit model validate models/user_schema.yaml
```

##### `model generate`

Generates a sample model file based on an input dataset. This helps in bootstrapping new model definitions.

```bash
simpleaudit model generate INPUT_FILE -o OUTPUT_FILE
```

*   `INPUT_FILE`: Path to the input data file.
*   `-o`, `--output`: Path to save the generated model file.

**Example:**

```bash
simpleaudit model generate data/sales.csv -o models/sales_model.yaml
```

##### `model list`

Lists all models found in the specified directory.

```bash
simpleaudit model list --dir MODELS_DIR
```

*   `--dir`: Directory path to scan for model files. Defaults to `./models`.

**Example:**

```bash
simpleaudit model list --dir ./config/models
```

#### 3. Visualizing Results

The `visualize` subcommand generates charts and reports from previously saved audit results.

**Usage:**

```bash
simpleaudit visualize [OPTIONS] RESULTS_FILE
```

**Arguments:**

*   `RESULTS_FILE`: Path to the audit results file (JSON).

**Options:**

| Option | Description |
| :--- | :--- |
| `--chart-type` | Type of chart to generate (`bar`, `pie`, `line`). Default is `bar`. |
| `--metric` | Specific metric to visualize (e.g., `error_count`, `pass_rate`). |
| `--output` | Path to save the generated image (PNG or SVG). |
| `--width` | Width of the chart in pixels. Default is `800`. |
| `--height` | Height of the chart in pixels. Default is `600`. |

**Example:**

```bash
# Generate a bar chart of error counts from results.json
simpleaudit visualize --chart-type bar --metric error_count --output errors.png results.json

# Generate a pie chart of pass rates
simpleaudit visualize --chart-type pie --metric pass_rate --output pass_rate.png results.json
```

### Configuration File

The CLI can be configured via a `simpleaudit.yaml` file in the current working directory or specified via the `--config` global option. This allows you to set default values for common options.

**Example `simpleaudit.yaml`:**

```yaml
default_output_format: json
log_level: INFO
model_dir: ./models
```

If a value is provided in both the configuration file and the command line, the command-line argument takes precedence.

### Exit Codes

The CLI uses standard exit codes to indicate success or failure, which is critical for automation.

| Code | Description |
| :--- | :--- |
| `0` | Success. All operations completed without errors. |
| `1` | General error. An unexpected exception occurred. |
| `2` | Usage error. Invalid arguments or missing files. |
| `3` | Audit Failure. Used only when `--fail-on-error` is set and an audit rule fails. |

### Troubleshooting

**"File not found" errors:**
Ensure that the paths provided for `INPUT_FILE`, `--rules`, and `--output` are correct relative to your current working directory. You can use absolute paths to avoid ambiguity.

**"Invalid Rule Format" errors:**
If the CLI reports a rule format error, use `simpleaudit model validate` on your rule file if it is structured as a model, or check the YAML syntax manually. The `simpleaudit` library expects strict adherence to the defined schema in `simpleaudit/schemas/rules.py`.

**Permission denied:**
Ensure you have read permissions for input files and write permissions for output directories.

### Programmatic Access

While the CLI is designed for shell usage, the underlying logic is exposed via the `simpleaudit.cli` module. You can import the main parser for integration into other tools:

```python
from simpleaudit.cli import build_parser

parser = build_parser()
args = parser.parse_args(['audit', '--rules', 'rules.yaml', 'data.csv'])
# Process args...
```

This allows you to embed `simpleaudit` functionality into larger Python applications while maintaining consistency with the CLI behavior.

### See Also

*   [Results and Analysis](results.md)
