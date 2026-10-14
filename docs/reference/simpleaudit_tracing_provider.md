## simpleaudit.tracing.provider

TraceProvider — pluggable source of traces for an audit.

### Classes

#### `TraceProvider`

Base class for a trace source used during an audit.

**Methods:**

- `start() -> 'TraceProvider'`
- `stop() -> None`
- `endpoint() -> Optional[str]` — OTLP endpoint the target should export to, or None for fetch-based.
- `fetch(trace_id: str) -> List[Dict[str, Any]]` — Return the normalized spans for `trace_id` (may be empty).

#### `BuiltinOTLP`

Run an ephemeral OTLP receiver for the audit; collect spans in-memory.

Point the target's `OTEL_EXPORTER_OTLP_ENDPOINT` at `endpoint`
(with `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`) for the duration of the
audit. Spans are discarded on `stop`.

**Methods:**

- `start() -> 'BuiltinOTLP'`
- `stop() -> None`
- `endpoint() -> Optional[str]`
- `store() -> SpanStore`
- `fetch(trace_id: str) -> List[Dict[str, Any]]`

#### `ExternalTraceProvider`

Fetch traces from an existing observability backend by trace id.

Subclass and implement `_fetch_remote` to query your backend
(Tempo, Jaeger, an OTel Collector, etc.). SimpleAudit propagates the
`traceparent` during the run; after the response, call `fetch`
with the recorded trace id to retrieve the spans.

Example (pseudo)::

    class TempoProvider(ExternalTraceProvider):
        def _fetch_remote(self, trace_id):
            # GET {base}/api/traces/{trace_id} -> OTLP JSON -> parse
            ...

**Methods:**

- `fetch(trace_id: str) -> List[Dict[str, Any]]`

### Functions

#### `audit_with_tracingasync (auditor: Any, scenarios: Any, provider: Optional[TraceProvider] = None, audit_run_id: Optional[str] = None, token_budget: Optional[int] = None, **run_kwargs: Any) -> Any`

Run an audit with a trace provider and attach per-scenario evidence.

This is the Promptfoo-style flow for a **controlled target** you can point
at an OTLP endpoint:

1. Start the provider (default: `BuiltinOTLP` ephemeral receiver).
2. **You** point the target's `OTEL_EXPORTER_OTLP_ENDPOINT` at
   `provider.endpoint` (with `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`)
   for the duration of the run. The helper exposes the endpoint via the
   returned object so you can configure the target before/around the run.
3. Run the audit. The engine propagates a W3C `traceparent` per turn and
   records `turn_id -> trace_id` in a `TraceCorrelation`.
4. After the run, collect each scenario's spans (union of its turns'
   traces) and select evidence-relevant spans.
5. Stop the provider (discarding spans for BuiltinOTLP).

The selected spans are attached to each `AuditResult` under
`result.judgment["evidence_spans"]` (with provenance), ready to feed a
trace-aware judge or a Finding's `EvidenceRef`.

**Returns**
A `AuditResults` whose per-result
`judgment["evidence_spans"]` carry the selected spans (with provenance).

**Notes**
The helper does **not** mutate the target. For a target that reads its
OTLP endpoint from the environment (e.g. Open WebUI), set
`OTEL_EXPORTER_OTLP_ENDPOINT` to `provider.endpoint` before the run.

