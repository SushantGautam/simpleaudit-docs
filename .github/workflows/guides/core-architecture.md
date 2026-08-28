## Core Architecture

SimpleAudit architecture centers on event-driven data pipeline. System captures application events, transforms data, stores records, provides query interface. Design prioritizes low overhead, extensibility, deterministic output.

### System Overview

Library operates via three core layers:

1.  **Capture Layer**: Intercepts application events. Uses decorators, middleware, or explicit API calls.
2.  **Processing Layer**: Normalizes raw event data. Applies validation, enrichment, serialization rules.
3.  **Storage Layer**: Persists audit records. Supports multiple backends (SQL, NoSQL, file-based).

Data flow follows unidirectional path:

```mermaid
graph LR
    A[Application Code] -->|Event| B(Capture Layer)
    B -->|Raw Data| C(Processing Layer)
    C -->|Validated Record| D(Storage Layer)
    D -->|Query API| E[Consumer]
```

### Module Interactions

Modules maintain strict separation of concerns. Dependencies flow inward.

| Module | Responsibility | Depends On |
| :--- | :--- | :--- |
| `simpleaudit.core` | Core data structures, event definitions | None |
| `simpleaudit.capture` | Event interception, decorators | `core` |
| `simpleaudit.process` | Data transformation, validation | `core` |
| `simpleaudit.storage` | Persistence adapters | `core`, `process` |
| `simpleaudit.query` | Search, filtering, retrieval | `storage` |

### Core Data Structures

`simpleaudit.core` defines fundamental types. All modules interact via these classes.

#### `AuditEvent`

Base class for all audit records. Immutable after creation.

```python
from simpleaudit.core import AuditEvent
import uuid
from datetime import datetime

class AuditEvent:
    def __init__(self, event_id, timestamp, actor, action, resource, metadata=None):
        self.event_id = event_id
        self.timestamp = timestamp
        self.actor = actor
        self.action = action
        self.resource = resource
        self.metadata = metadata or {}

    def to_dict(self):
        return {
            "event_id": str(self.event_id),
            "timestamp": self.timestamp.isoformat(),
            "actor": self.actor,
            "action": self.action,
            "resource": self.resource,
            "metadata": self.metadata
        }
```

**Parameters:**

*   `event_id` (UUID): Unique identifier. Auto-generated if not provided.
*   `timestamp` (datetime): Event occurrence time. Defaults to current UTC time.
*   `actor` (str): User or system identifier performing action.
*   `action` (str): Action type (e.g., "create", "update", "delete").
*   `resource` (str): Target object identifier or path.
*   `metadata` (dict): Additional context data. Optional.

#### `AuditContext`

Holds runtime state for event processing. Passed through pipeline.

```python
from simpleaudit.core import AuditContext

class AuditContext:
    def __init__(self, user_id=None, ip_address=None, session_id=None):
        self.user_id = user_id
        self.ip_address = ip_address
        self.session_id = session_id
        self.attributes = {}
```

### Capture Layer

`simpleaudit.capture` provides mechanisms to record events.

#### Decorator Approach

Most common usage pattern. Wraps functions/methods to automatically log execution.

```python
from simpleaudit.capture import audit

@audit(action="user.login", resource="auth")
def login(username, password):
    # Application logic
    return True
```

**Parameters:**

*   `action` (str): Action label.
*   `resource` (str): Resource label.
*   `capture_args` (bool): Include function arguments in metadata. Default `False`.
*   `capture_result` (bool): Include return value in metadata. Default `False`.
*   `ignore_exceptions` (bool): Log exceptions as failed events. Default `True`.

#### Explicit API

For manual control or non-function contexts.

```python
from simpleaudit import SimpleAudit

audit_client = SimpleAudit()

audit_client.record(
    action="order.created",
    resource="orders/123",
    actor="user_456",
    metadata={"amount": 99.99, "currency": "USD"}
)
```

### Processing Layer

`simpleaudit.process` handles data normalization.

#### `EventProcessor`

Main class for transforming raw events into `AuditEvent` objects.

```python
from simpleaudit.process import EventProcessor

class EventProcessor:
    def __init__(self, validators=None, enrichers=None):
        self.validators = validators or []
        self.enrichers = enrichers or []

    def process(self, raw_data, context):
        # 1. Validate raw data
        for validator in self.validators:
            validator.validate(raw_data)

        # 2. Enrich data
        for enricher in self.enrichers:
            raw_data = enricher.enrich(raw_data, context)

        # 3. Construct AuditEvent
        return AuditEvent(
            event_id=raw_data.get("id"),
            timestamp=raw_data.get("timestamp"),
            actor=context.user_id,
            action=raw_data["action"],
            resource=raw_data["resource"],
            metadata=raw_data.get("metadata", {})
        )
```

**Configuration Options:**

*   `validators`: List of `Validator` instances. Run before enrichment.
*   `enrichers`: List of `Enricher` instances. Modify raw data before event creation.

#### Custom Enricher

Example: Add IP address from context.

```python
from simpleaudit.process import BaseEnricher

class IPEnricher(BaseEnricher):
    def enrich(self, raw_data, context):
        if context.ip_address:
            raw_data.setdefault("metadata", {})["ip"] = context.ip_address
        return raw_data
```

### Storage Layer

`simpleaudit.storage` abstracts persistence.

#### `StorageBackend` Interface

All backends must implement this interface.

```python
from abc import ABC, abstractmethod

class StorageBackend(ABC):
    @abstractmethod
    def save(self, event: AuditEvent):
        pass

    @abstractmethod
    def query(self, filters: dict) -> list:
        pass

    @abstractmethod
    def close(self):
        pass
```

#### Built-in Backends

1.  **`SQLiteBackend`**: Default. File-based. Good for development/small scale.
    ```python
    from simpleaudit.storage import SQLiteBackend

    backend = SQLiteBackend(db_path="audit.db")
    ```

2.  **`PostgreSQLBackend`**: Production-grade. Requires `psycopg2`.
    ```python
    from simpleaudit.storage import PostgreSQLBackend

    backend = PostgreSQLBackend(
        host="localhost",
        port=5432,
        dbname="audit_db",
        user="admin",
        password="secret"
    )
    ```

3.  **`MemoryBackend`**: In-memory storage. Testing only.
    ```python
    from simpleaudit.storage import MemoryBackend

    backend = MemoryBackend()
    ```

### Query Interface

`simpleaudit.query` provides retrieval capabilities.

#### `AuditQuery`

Fluent interface for building queries.

```python
from simpleaudit.query import AuditQuery
from datetime import datetime, timedelta

query = AuditQuery(backend)

results = query \
    .filter_action("user.login") \
    .filter_time_range(
        start=datetime.utcnow() - timedelta(days=1),
        end=datetime.utcnow()
    ) \
    .filter_actor("user_456") \
    .limit(100) \
    .execute()

for event in results:
    print(event.to_dict())
```

**Methods:**

*   `filter_action(action: str)`: Filter by action type.
*   `filter_actor(actor: str)`: Filter by actor ID.
*   `filter_resource(resource: str)`: Filter by resource ID.
*   `filter_time_range(start: datetime, end: datetime)`: Filter by timestamp range.
*   `limit(n: int)`: Restrict result count.
*   `offset(n: int)`: Skip first N results.
*   `execute()`: Run query, return list of `AuditEvent`.

### Configuration

Global configuration via `SimpleAudit` initializer.

```python
from simpleaudit import SimpleAudit

audit = SimpleAudit(
    backend=SQLiteBackend("audit.db"),
    processor=EventProcessor(
        enrichers=[IPEnricher()]
    ),
    debug=False,
    max_metadata_size=1024  # Bytes
)
```

**Parameters:**

*   `backend`: Instance of `StorageBackend`.
*   `processor`: Instance of `EventProcessor`.
*   `debug`: Enable verbose logging. Default `False`.
*   `max_metadata_size`: Maximum allowed metadata size in bytes. Larger payloads truncated. Default `1024`.

### Error Handling

Library raises specific exceptions for different failure modes.

| Exception | Cause |
| :--- | :--- |
| `AuditValidationError` | Data fails validation rules. |
| `StorageError` | Backend persistence failure. |
| `QueryError` | Invalid query parameters. |

Catch exceptions at application level. Do not let audit failures crash main application logic.

```python
try:
    audit.record(action="test", resource="test")
except StorageError as e:
    logger.error(f"Audit storage failed: {e}")
```

### Performance Considerations

*   **Async Support**: `SimpleAudit` supports async operations via `asyncio`. Use `await audit.record_async(...)`.
*   **Batching**: For high-volume events, use `audit.batch_record(events_list)`. Reduces I/O overhead.
*   **Serialization**: Metadata serialized to JSON. Keep payloads small. Large objects increase latency.
*   **Indexing**: Storage backends create indexes on `action`, `actor`, `timestamp`. Ensure query filters use indexed fields.

### File Structure

```
simpleaudit/
├── __init__.py          # Main entry point, SimpleAudit class
├── core/
│   ├── __init__.py
│   ├── event.py         # AuditEvent, AuditContext
│   └── exceptions.py    # Custom exceptions
├── capture/
│   ├── __init__.py
│   └── decorators.py    # @audit decorator
├── process/
│   ├── __init__.py
│   ├── processor.py     # EventProcessor
│   └── enrichers.py     # BaseEnricher, IPEnricher
├── storage/
│   ├── __init__.py
│   ├── base.py          # StorageBackend ABC
│   ├── sqlite.py        # SQLiteBackend
│   ├── postgres.py      # PostgreSQLBackend
│   └── memory.py        # MemoryBackend
└── query/
    ├── __init__.py
    └── builder.py       # AuditQuery
```

Developers interact primarily with `simpleaudit.SimpleAudit` and `simpleaudit.capture.audit`. Internal modules (`core`, `process`, `storage`, `query`) expose public APIs for advanced customization.

### See Also

*   [CLI Usage](cli-usage.md)
*   [Getting Started](getting-started.md)
*   [Results & Analysis](results-analysis.md)
*   [Judges System](judges-system.md)
