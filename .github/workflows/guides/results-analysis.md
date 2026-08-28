## Results & Analysis

SimpleAudit processes, stores, and interprets audit results. Module handles raw data transformation, persistence, and statistical analysis.

### Overview

`simpleaudit.results` subsystem manages lifecycle of audit findings. Captures raw events, normalizes data, persists to storage backend, computes metrics. Provides API for querying, filtering, exporting results.

### Core Classes

#### `AuditResult`

Represents single audit finding. Immutable data structure.

```python
from simpleaudit.results import AuditResult

result = AuditResult(
    id="audit-001",
    timestamp="2023-10-27T10:00:00Z",
    resource="database-01",
    action="login",
    user="admin",
    status="success",
    metadata={"ip": "192.168.1.1"}
)
```

**Parameters:**

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `id` | `str` | Unique identifier for result. |
| `timestamp` | `str` | ISO 8601 timestamp. |
| `resource` | `str` | Target resource name. |
| `action` | `str` | Action performed. |
| `user` | `str` | User identifier. |
| `status` | `str` | Outcome: `success`, `failure`, `warning`. |
| `metadata` | `dict` | Additional key-value data. |

#### `ResultStore`

Handles persistence of `AuditResult` objects. Supports multiple backends.

```python
from simpleaudit.results import ResultStore

store = ResultStore(backend="sqlite", path="audit.db")
store.save(result)
```

**Parameters:**

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `backend` | `str` | Storage type: `sqlite`, `json`, `csv`. |
| `path` | `str` | File path for local backends. |
| `options` | `dict` | Backend-specific configuration. |

**Methods:**

*   `save(result: AuditResult)`: Persist single result.
*   `save_batch(results: list[AuditResult])`: Persist multiple results.
*   `query(filter: dict) -> list[AuditResult]`: Retrieve results matching criteria.
*   `delete(id: str)`: Remove result by ID.

#### `Analyzer`

Computes statistics and patterns from stored results.

```python
from simpleaudit.results import Analyzer

analyzer = Analyzer(store)
summary = analyzer.get_summary()
```

**Methods:**

*   `get_summary() -> dict`: Aggregate counts by status, resource, user.
*   `get_timeline(interval: str) -> list[dict]`: Group results by time interval (`hour`, `day`, `week`).
*   `detect_anomalies(threshold: float) -> list[AuditResult]`: Flag unusual activity.
*   `export(format: str) -> str`: Serialize results to `json`, `csv`, `html`.

### Processing Pipeline

Raw audit events flow through normalization before storage.

1.  **Capture**: `AuditLogger` emits raw events.
2.  **Normalize**: `ResultProcessor` validates, formats timestamps, enriches metadata.
3.  **Store**: `ResultStore` persists normalized `AuditResult`.
4.  **Analyze**: `Analyzer` computes metrics on demand.

```python
from simpleaudit.results import ResultProcessor

processor = ResultProcessor()
normalized = processor.process(raw_event)
```

### Configuration

Configure storage and analysis via `SimpleAudit` initialization.

```python
from simpleaudit import SimpleAudit

audit = SimpleAudit(
    results={
        "backend": "sqlite",
        "path": "./data/audit.db",
        "anomaly_threshold": 0.95,
        "retention_days": 30
    }
)
```

**Options:**

| Key | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `backend` | `str` | `json` | Storage engine. |
| `path` | `str` | `audit.json` | Storage location. |
| `anomaly_threshold` | `float` | `0.9` | Z-score threshold for anomalies. |
| `retention_days` | `int` | `None` | Auto-delete old results. |

### Usage Examples

#### Save and Query

```python
from simpleaudit.results import AuditResult, ResultStore

store = ResultStore(backend="sqlite", path="test.db")

# Save
r1 = AuditResult(id="1", timestamp="2023-10-27T10:00:00Z", resource="db", action="query", user="alice", status="success")
store.save(r1)

# Query
failures = store.query({"status": "failure"})
print(f"Found {len(failures)} failures")
```

#### Generate Report

```python
from simpleaudit.results import Analyzer

analyzer = Analyzer(store)

# Get daily timeline
timeline = analyzer.get_timeline(interval="day")
for day in timeline:
    print(f"{day['date']}: {day['count']} events")

# Export to CSV
csv_data = analyzer.export(format="csv")
with open("report.csv", "w") as f:
    f.write(csv_data)
```

#### Anomaly Detection

```python
# Flag events with z-score > 0.95
anomalies = analyzer.detect_anomalies(threshold=0.95)
for a in anomalies:
    print(f"Anomaly: {a.id} at {a.timestamp}")
```

### File Structure

*   `simpleaudit/results/__init__.py`: Exports `AuditResult`, `ResultStore`, `Analyzer`.
*   `simpleaudit/results/store.py`: `ResultStore` implementation.
*   `simpleaudit/results/analyzer.py`: `Analyzer` implementation.
*   `simpleaudit/results/processor.py`: `ResultProcessor` normalization logic.

### Error Handling

*   `StorageError`: Raised when backend write fails.
*   `InvalidResultError`: Raised when `AuditResult` validation fails.
*   `AnalysisError`: Raised when computation fails (e.g., empty dataset).

```python
from simpleaudit.results import StorageError

try:
    store.save(result)
except StorageError as e:
    print(f"Storage failed: {e}")
```

### Performance Notes

*   `save_batch` significantly faster than repeated `save` calls.
*   `Analyzer` caches computed summaries. Call `analyzer.invalidate_cache()` after bulk updates.
*   Large datasets: Use `sqlite` backend over `json` for query performance.

### See Also

*   [Core Architecture](core-architecture.md)
*   [CLI Usage](cli-usage.md)
*   [Custom Scenario Development](custom-scenario-development.md)
*   [Getting Started](getting-started.md)
