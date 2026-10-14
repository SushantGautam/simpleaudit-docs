## simpleaudit.tracing.shared

Shared OTLP receiver + per-audit trace session routing.

### Classes

#### `TraceSession`

Ephemeral trace buffer for one audit run.

Spans are routed here by `trace_id`. The session expires after
`ttl` seconds (checked lazily on access) or when `close` is
called explicitly.

`max_spans` caps the number of spans retained for this session to
prevent a single audit from consuming unbounded memory under heavy
OTLP traffic. When the cap is hit, new spans are dropped (and counted
in `dropped`).

**Methods:**

- `expired() -> bool`
- `store() -> SpanStore`
- `dropped() -> int` — Number of spans dropped due to the per-session cap.
- `full() -> bool`
- `add_many(spans: List[Dict[str, Any]]) -> int` — Add spans; returns the number actually stored (vs dropped).
- `spans_for_trace(trace_id: str) -> List[Dict[str, Any]]`
- `all_spans() -> List[Dict[str, Any]]`
- `close() -> None` — Discard all spans for this session.

**Attributes:**

- `audit_id`
- `execution_id`
- `target_id`
- `created_at`
- `ttl`
- `max_spans`

#### `TraceSessionManager`

Routes incoming spans to the correct `TraceSession`.

The primary correlation mechanism is `trace_id` (from the OTLP span
itself). The manager keeps a `trace_id → session` index so that when a
span arrives, it can be routed to the right audit's buffer.

Sessions are cleaned up lazily (on access) and via `sweep`.

Heavy-traffic protections:
- **Early rejection**: if no sessions are active, `route_spans`
  returns immediately without touching the spans (zero-cost drop).
- **Per-session cap**: each `TraceSession` has a `max_spans`
  limit; excess spans are dropped and counted.
- **Global cap**: `max_total_spans` limits the total spans across all
  active sessions; when hit, new spans are dropped globally.

**Methods:**

- `dropped_no_session() -> int` — Spans dropped because no session was active (early rejection).
- `dropped_global_cap() -> int` — Spans dropped because the global span cap was hit.
- `total_spans() -> int` — Total spans across all active sessions.
- `has_active_sessions() -> bool` — Quick check: is there at least one non-expired session?
- `create(audit_id: str, execution_id: str = '', target_id: str = '', ttl: float = 300.0, max_spans: int = 10000) -> TraceSession` — Create a new trace session for an audit run.
- `register_trace(audit_id: str, trace_id: str) -> None` — Map a `trace_id` to an audit session (called by the correlation layer).
- `route_spans(spans: List[Dict[str, Any]]) -> None` — Route incoming spans to the session that owns their `trace_id`.
- `get(audit_id: str) -> Optional[TraceSession]`
- `close(audit_id: str) -> None` — Close a session and discard its spans.
- `sweep() -> int` — Close all expired sessions. Returns the number closed.
- `active_sessions() -> List[TraceSession]`

#### `SharedOTLPReceiver`

A long-lived OTLP receiver that routes spans to per-audit trace sessions.

One instance per process/deployment. Multiple targets (Open WebUI, agent
SDKs, HTTP apps) all export to the same endpoint. Spans are routed to the
correct `TraceSession` by `trace_id`.

The receiver handles both OTLP/HTTP JSON and OTLP/HTTP protobuf.

Usage::

    # Boot (once)
    shared = SharedOTLPReceiver(host="0.0.0.0", port=4317).start()

    # Per audit
    session = shared.sessions.create(audit_id="audit_101", ttl=300)
    # Engine records traceparent → call:
    shared.sessions.register_trace("audit_101", trace_id)
    # ... run audit ...
    spans = session.spans_for_trace(trace_id)
    shared.sessions.close("audit_101")

    # Shutdown
    shared.stop()

**Methods:**

- `endpoint() -> str` — The OTLP/HTTP traces URL for targets to export to.
- `actual_port() -> int`
- `stats() -> Dict[str, int]` — Drop counters and totals for observability.
- `create_session(audit_id: str, execution_id: str = '', target_id: str = '', ttl: Optional[float] = None) -> TraceSession` — Create a trace session with the receiver's configured defaults.
- `start() -> 'SharedOTLPReceiver'` — Start the shared receiver and the session sweep thread.
- `stop() -> None` — Stop the receiver and close all sessions.

#### `SharedOTLP`

A `TraceProvider` that plugs into a `SharedOTLPReceiver`.

Use this with `audit_with_tracing` when the target exports to a
shared, long-lived OTLP receiver (e.g. the Studio's global endpoint)
rather than an ephemeral per-audit receiver.

The provider creates a `TraceSession` on `start`, registers
trace ids as the engine generates them, and discards the session on
`stop`.

Usage::

    shared = SharedOTLPReceiver(port=4317).start()
    provider = SharedOTLP(shared, audit_id="audit_101")
    results = await audit_with_tracing(auditor, "safety", provider=provider)

**Methods:**

- `start() -> 'SharedOTLP'`
- `stop() -> None`
- `endpoint() -> Optional[str]`
- `audit_id() -> str`
- `register_trace(trace_id: str) -> None` — Register a trace_id with this session (called by the correlation layer).
- `fetch(trace_id: str) -> List[Dict[str, Any]]`

