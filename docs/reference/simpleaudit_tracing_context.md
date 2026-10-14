## simpleaudit.tracing.context

Trace-context generation and correlation for SimpleAudit.

### Classes

#### `TurnTraceLink`

Links one audit turn to the trace(s) observed for it.

**Attributes:**

- `turn_id`
- `trace_ids`
- `traceparent`

#### `TraceCorrelation`

Correlates an audit run's turns with observed traces.

Call `record` as traces arrive; query with `trace_ids_for_turn`.

Set `on_new_trace` to a callback that is invoked the first time each
`trace_id` is recorded. Use this to register the trace with a
`SharedOTLP` session so the shared
receiver routes incoming spans to the right audit.

**Methods:**

- `link_turn(turn_id: str, traceparent: Optional[str] = None) -> TurnTraceLink`
- `record(turn_id: str, trace_id: str) -> None`
- `trace_ids_for_turn(turn_id: str) -> List[str]`
- `all_trace_ids() -> List[str]`
- `turns() -> List[TurnTraceLink]`
- `spans_for_turn(turn_id: str, store: 'Any') -> List[Dict[str, Any]]` — Return all spans in `store` belonging to the traces of `turn_id`.

**Attributes:**

- `audit_run_id`
- `on_new_trace`

### Functions

#### `new_trace_id() -> str`

Generate a 32-hex-char W3C trace id.

#### `new_span_id() -> str`

Generate a 16-hex-char W3C span id (non-zero).

#### `make_traceparent(trace_id: Optional[str] = None, span_id: Optional[str] = None) -> str`

Build a W3C `traceparent` header value.

Format: `version-traceid-parentid-traceflags` (e.g. `00-<32hex>-<16hex>-01`).

