## CLI Usage

SimpleAudit provides robust command-line interface (CLI) tools. Developers execute audits directly from terminal. No Python script required. CLI wraps core library logic. Supports batch processing. Supports single file analysis.

### Overview

CLI module handles argument parsing. Executes audit tasks. Outputs results to stdout or file. Integrates with `simpleaudit.core` engine.

Key features:
- Multiple execution modes
- Flexible output formats
- Configurable logging
- Batch processing support

### Installation

Install library first.

```bash
pip install simpleaudit
```

Verify installation.

```bash
simpleaudit --version
```

### Basic Execution

Run audit on single file.

```bash
simpleaudit audit path/to/file.py
```

Run audit on directory. Recursive scan enabled by default.

```bash
simpleaudit audit path/to/project/
```

### Command Structure

General syntax:

```bash
simpleaudit <command> [options] <target>
```

#### Commands

| Command | Description |
| :--- | :--- |
| `audit` | Execute security audit |
| `report` | Generate report from previous audit |
| `config` | View or modify configuration |
| `version` | Display version information |
| `help` | Show help message |

### Audit Command

Primary command. Performs static analysis.

#### Syntax

```bash
simpleaudit audit [OPTIONS] TARGET
```

#### Options

| Flag | Short | Description | Default |
| :--- | :--- | :--- | :--- |
| `--output` | `-o` | Output file path | `stdout` |
| `--format` | `-f` | Output format: `json`, `csv`, `html`, `txt` | `txt` |
| `--severity` | `-s` | Minimum severity level: `low`, `medium`, `high`, `critical` | `low` |
| `--rules` | `-r` | Specific rule IDs to run | All rules |
| `--exclude` | `-e` | Glob patterns to exclude files | None |
| `--include` | `-i` | Glob patterns to include files | `*.py` |
| `--verbose` | `-v` | Enable debug logging | `False` |
| `--quiet` | `-q` | Suppress non-error output | `False` |
| `--timeout` | `-t` | Max execution time seconds | `300` |
| `--workers` | `-w` | Parallel worker count | `1` |

#### Examples

**Basic audit**

```bash
simpleaudit audit main.py
```

**JSON output to file**

```bash
simpleaudit audit -o results.json -f json src/
```

**High severity only**

```bash
simpleaudit audit -s high --exclude "test_*.py" src/
```

**Parallel execution**

```bash
simpleaudit audit -w 4 --verbose large_project/
```

### Report Command

Generates formatted reports. Requires prior audit data.

#### Syntax

```bash
simpleaudit report [OPTIONS] AUDIT_FILE
```

#### Options

| Flag | Description |
| :--- | :--- |
| `--template` | Custom report template path |
| `--title` | Report title string |
| `--author` | Author name |
| `--export` | Export format: `pdf`, `html`, `markdown` |

#### Example

```bash
simpleaudit report -e html --title "Q3 Security Audit" audit_data.json
```

### Configuration

CLI reads config from multiple sources. Priority order:

1. Command-line flags
2. Environment variables
3. Local `.simpleaudit.yaml`
4. Global `~/.config/simpleaudit/config.yaml`

#### Environment Variables

| Variable | Description |
| :--- | :--- |
| `SIMPLEAUDIT_OUTPUT` | Default output path |
| `SIMPLEAUDIT_FORMAT` | Default output format |
| `SIMPLEAUDIT_SEVERITY` | Default severity threshold |
| `SIMPLEAUDIT_LOG_LEVEL` | Logging level: `DEBUG`, `INFO`, `WARNING`, `ERROR` |

#### Config File Example

`.simpleaudit.yaml`

```yaml
default_severity: medium
output_format: json
exclude_patterns:
  - "*/tests/*"
  - "*/migrations/*"
rules:
  enabled:
    - "sql_injection"
    - "xss"
    - "path_traversal"
  disabled:
    - "hardcoded_passwords"
```

### Exit Codes

CLI returns specific exit codes. Useful for CI/CD pipelines.

| Code | Meaning |
| :--- | :--- |
| `0` | Audit completed. No issues found |
| `1` | Audit completed. Issues found |
| `2` | Configuration error |
| `3` | File not found |
| `4` | Timeout exceeded |
| `5` | Internal error |

#### CI/CD Integration

Fail build on high severity issues.

```bash
if simpleaudit audit -s high src/; then
  echo "Audit passed"
else
  exit 1
fi
```

### Output Formats

#### Text (Default)

Human-readable summary.

```
[CRITICAL] SQL Injection detected
  File: src/db.py
  Line: 42
  Rule: sql_injection_001

[HIGH] Hardcoded Credentials
  File: src/config.py
  Line: 15
  Rule: hardcoded_secret_002
```

#### JSON

Machine-readable. Suitable for parsing.

```json
{
  "audit_id": "abc123",
  "timestamp": "2023-10-27T10:00:00Z",
  "findings": [
    {
      "severity": "critical",
      "rule": "sql_injection_001",
      "file": "src/db.py",
      "line": 42,
      "message": "SQL Injection detected"
    }
  ],
  "summary": {
    "total": 1,
    "critical": 1,
    "high": 0,
    "medium": 0,
    "low": 0
  }
}
```

#### CSV

Spreadsheet compatible.

```csv
severity,rule,file,line,message
critical,sql_injection_001,src/db.py,42,SQL Injection detected
```

#### HTML

Styled report. Viewable in browser.

### Advanced Usage

#### Custom Rules

Load custom rule files.

```bash
simpleaudit audit -r custom_rules.yaml src/
```

Rule file structure:

```yaml
rules:
  - id: "custom_check_001"
    name: "Custom Check"
    pattern: "eval\\("
    severity: "medium"
    description: "Avoid eval usage"
```

#### Exclusion Patterns

Exclude specific files or directories.

```bash
simpleaudit audit -e "*/node_modules/*" -e "*/.git/*" src/
```

#### Timeout Handling

Long audits may timeout. Set limit.

```bash
simpleaudit audit -t 600 huge_project/
```

On timeout, CLI exits with code `4`. Partial results saved to temp file.

### Troubleshooting

#### Permission Denied

Ensure read access to target files.

```bash
chmod +r path/to/file.py
```

#### Module Not Found

Install dependencies.

```bash
pip install -r requirements.txt
```

#### No Output

Check severity filter. Lower threshold.

```bash
simpleaudit audit -s low src/
```

### Best Practices

1. **Use JSON format** for automated processing.
2. **Set severity thresholds** to focus on critical issues.
3. **Exclude test files** to reduce noise.
4. **Use parallel workers** for large codebases.
5. **Integrate with CI** using exit codes.
6. **Save reports** for historical tracking.

### API Reference

CLI wraps core functions. Direct Python access available.

```python
from simpleaudit.cli import main

# Programmatic execution
main(["audit", "-f", "json", "src/"])
```

Key classes:

- `simpleaudit.cli.AuditCommand`: Handles audit logic.
- `simpleaudit.cli.ReportCommand`: Handles report generation.
- `simpleaudit.cli.ConfigLoader`: Loads configuration files.

### Summary

CLI provides powerful audit capabilities. Simple syntax. Flexible options. Supports multiple output formats. Integrates easily with CI/CD. Developers should use JSON output for automation. Use severity filters to manage noise. Leverage parallel processing for speed.

### See Also

*   [Core Architecture](core-architecture.md)
*   [Getting Started](getting-started.md)
*   [Model Auditing](model-auditing.md)
*   [Custom Scenario Development](custom-scenario-development.md)
