## simpleaudit.tracing.store

Span store — in-memory persistence and query for ingested trace spans.

### Classes

#### `SpanStore`

In-memory span store with kind/trace/attribute queries.

**Methods:**

- `add(raw: Dict[str, Any]) -> Dict[str, Any]`
- `add_many(raws: List[Dict[str, Any]]) -> List[Dict[str, Any]]`
- `get(span_id: str) -> Optional[Dict[str, Any]]`
- `by_trace(trace_id: str) -> List[Dict[str, Any]]`
- `by_kind(kind: str) -> List[Dict[str, Any]]`
- `by_attribute(key: str, value: Any) -> List[Dict[str, Any]]`
- `all() -> List[Dict[str, Any]]`

### Functions

#### `normalize_span(raw: Dict[str, Any]) -> Dict[str, Any]`

Normalize a raw span (OTel/OpenInference-shaped) to the store schema.

Accepts the common OpenInference/OTel attribute keys and produces a flat
dict with: span_id, trace_id, name, kind, attributes, parent_span_id,
start_time, end_time, status.

