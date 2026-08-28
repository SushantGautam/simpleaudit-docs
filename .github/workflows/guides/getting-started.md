## Getting Started

SimpleAudit lightweight Python library audit code changes. Tracks file modifications, deletions, additions. Generates audit logs. Detects unauthorized changes.

### Overview

SimpleAudit monitors directory structure. Records file state changes. Useful compliance, security, debugging. No external dependencies. Pure Python.

### Installation

Install via pip.

```bash
pip install simpleaudit
```

Verify installation.

```python
import simpleaudit
print(simpleaudit.__version__)
```

### Environment Setup

Requires Python 3.8+. No system dependencies.

Create virtual environment recommended.

```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows
```

### First Audit Run

Initialize `Auditor` class. Specify target directory.

```python
from simpleaudit import Auditor

# Initialize auditor
auditor = Auditor(target_dir="/path/to/monitor")

# Run initial scan
initial_state = auditor.scan()

# Make changes to files
# ... modify files ...

# Run second scan
final_state = auditor.scan()

# Generate report
report = auditor.generate_report(initial_state, final_state)
print(report)
```

### Core Classes

#### `Auditor`

Main class. Manages audit lifecycle.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `target_dir` | `str` | Required | Path to directory to monitor |
| `ignore_patterns` | `list[str]` | `[]` | Glob patterns to ignore |
| `include_hidden` | `bool` | `False` | Include hidden files (dotfiles) |
| `hash_algorithm` | `str` | `"sha256"` | Hash algorithm for file integrity |

**Methods:**

##### `scan()`

Scans target directory. Returns dictionary of file states.

**Returns:** `dict[str, FileState]`

```python
state = auditor.scan()
# {'file1.py': FileState(hash='abc123', size=1024, mtime=1672531200), ...}
```

##### `generate_report(before, after)`

Compares two scan states. Returns audit report.

**Parameters:**

- `before`: `dict[str, FileState]` - Initial scan state
- `after`: `dict[str, FileState]` - Final scan state

**Returns:** `AuditReport`

```python
report = auditor.generate_report(initial_state, final_state)
print(report.added_files)    # List of new files
print(report.modified_files) # List of changed files
print(report.deleted_files)  # List of removed files
```

##### `save_report(report, path)`

Saves audit report to file.

**Parameters:**

- `report`: `AuditReport` - Report object
- `path`: `str` - Output file path

**Formats:** JSON, CSV, TXT based on extension.

```python
auditor.save_report(report, "audit_log.json")
```

### FileState Object

Represents single file state.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `path` | `str` | Relative file path |
| `hash` | `str` | File content hash |
| `size` | `int` | File size in bytes |
| `mtime` | `float` | Last modification timestamp |

### AuditReport Object

Contains comparison results.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `added_files` | `list[str]` | Files present in after, not before |
| `deleted_files` | `list[str]` | Files present in before, not after |
| `modified_files` | `list[str]` | Files with different hash/size |
| `unchanged_files` | `list[str]` | Files with identical state |
| `timestamp` | `str` | Report generation time (ISO 8601) |

### Configuration Options

Customize behavior via `Auditor` constructor.

**Ignore Patterns:**

```python
auditor = Auditor(
    target_dir="/project",
    ignore_patterns=["*.pyc", "__pycache__", ".git", "node_modules"]
)
```

**Hidden Files:**

```python
auditor = Auditor(
    target_dir="/project",
    include_hidden=True  # Include .env, .config, etc.
)
```

**Hash Algorithm:**

```python
auditor = Auditor(
    target_dir="/project",
    hash_algorithm="md5"  # Faster but less secure
)
```

Supported algorithms: `sha256`, `sha1`, `md5`.

### Practical Example

Monitor Python project for unauthorized changes.

```python
from simpleaudit import Auditor
import time

# Setup
target = "/home/user/myproject"
auditor = Auditor(
    target_dir=target,
    ignore_patterns=["*.pyc", "__pycache__", ".venv"]
)

# Baseline scan
baseline = auditor.scan()
print(f"Baseline: {len(baseline)} files tracked")

# Wait for changes
time.sleep(3600)  # 1 hour

# Check for changes
current = auditor.scan()
report = auditor.generate_report(baseline, current)

# Alert if changes detected
if report.added_files or report.modified_files or report.deleted_files:
    print("ALERT: Unauthorized changes detected!")
    print(f"Added: {report.added_files}")
    print(f"Modified: {report.modified_files}")
    print(f"Deleted: {report.deleted_files}")
    
    # Save report for investigation
    auditor.save_report(report, "security_alert.json")
else:
    print("No changes detected.")
```

### Error Handling

Common exceptions:

| Exception | Cause | Solution |
|-----------|-------|----------|
| `FileNotFoundError` | Target directory missing | Verify path exists |
| `PermissionError` | Insufficient read access | Run with proper permissions |
| `InvalidHashAlgorithm` | Unsupported hash method | Use sha256, sha1, or md5 |

```python
try:
    auditor = Auditor(target_dir="/nonexistent")
    state = auditor.scan()
except FileNotFoundError as e:
    print(f"Directory not found: {e}")
except PermissionError as e:
    print(f"Access denied: {e}")
```

### Performance Notes

- Large directories: Use `ignore_patterns` to exclude unnecessary files
- Hash computation: SHA256 slower than MD5 but more secure
- Memory: File states stored in memory; large projects may consume significant RAM
- Concurrent access: Not thread-safe; use locks if multi-process

### Next Steps

- [Advanced Usage](advanced-usage.md) - Scheduled audits, webhook notifications
- [API Reference](api-reference.md) - Complete method signatures
- [Troubleshooting](troubleshooting.md) - Common issues and fixes

### See Also

*   [CLI Usage](cli-usage.md)
*   [Core Architecture](core-architecture.md)
*   [Advanced Methodologies](advanced-methodologies.md)
*   [Custom Scenario Development](custom-scenario-development.md)
