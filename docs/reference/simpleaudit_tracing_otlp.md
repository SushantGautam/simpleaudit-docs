## simpleaudit.tracing.otlp

OTLP ingestion — accept OpenTelemetry trace exports and normalize to spans.

### Classes

#### `OTLPTraceReceiver`

Minimal async OTLP/HTTP JSON trace receiver.

Usage (e.g. with any ASGI framework)::

    receiver = OTLPTraceReceiver()
    # POST /v1/traces  ->  await receiver.handle(request_body_bytes)

Or standalone::

    store = SpanStore()
    receiver = OTLPTraceReceiver(store=store)
    await receiver.handle(open("export.json").read())

**Methods:**

- `async handle(body: Any, authorization: Optional[str] = None) -> Dict[str, Any]` — Ingest an OTLP JSON export body; return the OTLP ack payload.
- `trace_ids() -> List[str]`

#### `EphemeralOTLPReceiver`

A self-contained OTLP/HTTP trace receiver that lives for one audit.

Promptfoo-style: the auditor opens this receiver when an audit starts,
points the (controlled) target's `OTEL_EXPORTER_OTLP_ENDPOINT` at it,
collects the spans the target emits during the run, and closes it when the
audit ends. No external collector, no persistent store — the spans are
held in an in-memory `SpanStore` and discarded on `close`.

It speaks the **OTLP/HTTP JSON** protocol (`POST /v1/traces`), which is
what `OTEL_EXPORTER_OTLP_PROTOCOL=http/json` selects. For the default
`grpc` protocol, run an OTel Collector in front that forwards JSON, or
point the target at this receiver's HTTP endpoint.

Usage::

    async with EphemeralOTLPReceiver() as rx:
        # rx.endpoint == "http://127.0.0.1:<port>/v1/traces"
        # set the target's OTEL_EXPORTER_OTLP_ENDPOINT to rx.endpoint
        results = await auditor.run_async("safety", trace_correlation=corr)
        spans = rx.store.by_trace(trace_id)   # inspect / feed the judge

The server runs on a background thread (aiohttp) so it works from both
sync and async callers. Port 0 binds an ephemeral port.

**Methods:**

- `endpoint() -> str` — The OTLP/HTTP traces URL to configure the target's exporter to.
- `actual_port() -> int`
- `start() -> 'EphemeralOTLPReceiver'` — Start the receiver on a background thread; bind an ephemeral port.
- `stop() -> None` — Stop the server and discard the in-memory spans.

#### `EphemeralOTLPGRPCReceiver`

A self-contained OTLP/gRPC trace receiver that lives for one audit.

Speaks the **OTLP/gRPC** protocol (`TraceService/Export`), which is
what `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` (the default) selects on port
4317. Useful when the target (e.g. Open WebUI) exports over gRPC and you
want to capture spans without running a full OTel Collector.

Usage::

    rx = EphemeralOTLPGRPCReceiver(port=4317).start()
    # target's OTEL_EXPORTER_OTLP_ENDPOINT = http://127.0.0.1:4317
    # ... run audit ...
    rx.stop()

The gRPC server runs on a background thread. Port 0 binds an ephemeral
port. Spans are discarded on `stop` (ephemeral by design).

**Methods:**

- `endpoint() -> str` — The OTLP/gRPC endpoint to configure the target's exporter to.
- `actual_port() -> int`
- `start() -> 'EphemeralOTLPGRPCReceiver'` — Start the gRPC server on a background thread.
- `stop() -> None` — Stop the gRPC server and discard the in-memory spans.

### Functions

#### `parse_otlp_json(payload: Any) -> List[Dict[str, Any]]`

Parse an OTLP/HTTP JSON ExportTraceServiceRequest into raw spans.

Handles the standard shape::

    {"resourceSpans": [
        {"resource": {"attributes": [...]},
         "scopeSpans": [
            {"spans": [
                {"traceId": "...", "spanId": "...", "name": "...",
                 "kind": 1, "startTimeUnixNano": ..., "endTimeUnixNano": ...,
                 "attributes": [{"key": "...", "value": {"stringValue": "..."}}],
                 "status": {"code": 1}}
            ]}
        ]}
    ]}

`traceId` / `spanId` are hex strings in OTLP JSON. Attribute values use
the OTLP `AnyValue` oneof (stringValue, intValue, doubleValue, boolValue,
arrayValue, kvlistValue).

