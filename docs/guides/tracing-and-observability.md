# Tracing And Observability

SimpleAudit can act as an OpenTelemetry (OTLP) trace receiver so that an instrumented target — a RAG pipeline, an agent runtime, an app like Open WebUI — pushes its spans directly into the audit. The engine then correlates each conversation turn with a W3C `traceparent`, collects the spans the target emitted, and attaches the evidence-relevant ones to each result's judgment (`result.judgment["evidence_spans"]`) so judges and findings can cite exactly which retrieved document or tool call produced the answer.

Tracing is optional: black-box auditing works with no tracing at all. Everything in this page lives in the `simpleaudit.tracing` package.

```python
from simpleaudit.tracing import (
    BuiltinOTLP,
    EphemeralOTLPReceiver,
    EphemeralOTLPGRPCReceiver,
    ExternalTraceProvider,
    OTLPTraceReceiver,
    SpanStore,
    TraceProvider,
    TraceCorrelation,
    audit_with_tracing,
    normalize_span,
    parse_otlp_json,
)
```

## How a run is traced

The flow for a **controlled target** (one you can point at an OTLP endpoint):

1. Start a provider (default: `BuiltinOTLP`, an ephemeral OTLP receiver).
2. Point the target's `OTEL_EXPORTER_OTLP_ENDPOINT` at `provider.endpoint` (with `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`) for the duration of the run.
3. Run the audit. The engine propagates a W3C `traceparent` per turn and records `turn_id -> trace_id` in a `TraceCorrelation`.
4. After the run, collect each scenario's spans (the union of its turns' traces) and select the evidence-relevant kinds.
5. Stop the provider (spans are discarded for `BuiltinOTLP`).

## Trace providers

### `TraceProvider`

The base class defining the minimal contract every provider implements:

- `start()` / `stop()` — lifecycle; it also works as a context manager (`__enter__`/`__exit__`).
- `endpoint` property — the OTLP URL the target should export to, or `None` for fetch-based providers.
- `fetch(trace_id)` — return the normalized spans for one trace id (may be empty).

### `BuiltinOTLP`

Runs an ephemeral OTLP receiver for the audit and collects spans in memory. Best for CI, local apps, staging, and agent SDKs you control — no external collector required.

```python
def __init__(self, host="127.0.0.1", port=0, auth=None)
```

- `port=0` binds an ephemeral port; read the actual one via `actual_port` on the underlying receiver.
- `auth` optionally takes an `OTLPAuth` gate that checks each POST's `Authorization` header.
- `endpoint` — the OTLP/HTTP traces URL to configure the target's exporter to.
- `store` — the backing `SpanStore` (an empty store when not started).
- `fetch(trace_id)` — `store.by_trace(trace_id)` while running, else `[]`.
- Spans are discarded on `stop()`.

### `ExternalTraceProvider`

For targets that already ship traces to their own observability backend (Tempo, Jaeger, an OTel Collector, etc.). SimpleAudit propagates the `traceparent` during the run; after the response you call `fetch(trace_id)` to retrieve the spans from that backend. Subclass it and implement `_fetch_remote(trace_id)`:

```python
class TempoProvider(ExternalTraceProvider):
    def _fetch_remote(self, trace_id: str):
        # GET {base}/api/traces/{trace_id} -> OTLP JSON -> parse
        ...
```

The base implementation of `_fetch_remote` raises `NotImplementedError`, so a bare `ExternalTraceProvider` returns nothing until you subclass it.

## Receiving OTLP exports

### `EphemeralOTLPReceiver` (OTLP/HTTP JSON)

A self-contained receiver that lives for one audit. It speaks the **OTLP/HTTP JSON** protocol (`POST /v1/traces`), which is what `OTEL_EXPORTER_OTLP_PROTOCOL=http/json` selects. The server runs on a background thread (aiohttp), so it works from both sync and async callers.

```python
def __init__(self, host="127.0.0.1", port=0, store=None, authenticator=None, auth=None)
```

- `endpoint` — `http://{host}:{actual_port}/v1/traces`.
- `actual_port` — the bound port (useful with `port=0`).
- `store` — the in-memory `SpanStore` holding ingested spans.
- `start()` / `stop()` — start the server; `stop()` discards all spans (ephemeral by design). Both sync and async context managers are supported.

```python
import asyncio, httpx

async def main():
    rx = EphemeralOTLPReceiver().start()
    try:
        # In a real setup, set your target's OTEL_EXPORTER_OTLP_ENDPOINT here:
        #   os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"] = rx.endpoint
        #   os.environ["OTEL_EXPORTER_OTLP_PROTOCOL"] = "http/json"
        payload = {
            "resourceSpans": [
                {
                    "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "my-agent"}}]},
                    "scopeSpans": [
                        {"spans": [
                            {
                                "traceId": "a" * 32,
                                "spanId": "b" * 16,
                                "name": "POST /chat",
                                "kind": 2,
                                "startTimeUnixNano": 1_000_000_000,
                                "endTimeUnixNano": 2_000_000_000,
                                "attributes": [
                                    {"key": "openinference.span.kind", "value": {"stringValue": "RETRIEVER"}},
                                    {"key": "http.url", "value": {"stringValue": "http://x/chat"}},
                                ],
                                "status": {"code": 1},
                            }
                        ]}
                    ],
                }
            ]
        }
        async with httpx.AsyncClient(timeout=10) as client:
            r = await client.post(rx.endpoint, json=payload)
            assert r.status_code == 200
            assert r.json()["partialSuccess"] == {"rejectedSpans": 0}

        spans = rx.store.by_trace("a" * 32)
        print(spans[0]["name"], spans[0]["attributes"]["http.url"])
    finally:
        rx.stop()
    # Spans are discarded on stop (ephemeral by design).
    assert len(rx.store) == 0

asyncio.run(main())
```

Malformed bodies still get an HTTP 200, but the ack reports the rejection: `{"partialSuccess": {"rejectedSpans": 1}}`.

### `EphemeralOTLPGRPCReceiver` (OTLP/gRPC)

Same idea, but speaking the **OTLP/gRPC** protocol (`TraceService/Export`) — what `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` (the default) selects on port 4317. Useful when the target exports over gRPC and you want to capture spans without running a full OTel Collector.

```python
def __init__(self, host="127.0.0.1", port=0, store=None)
```

- `endpoint` — `http://{host}:{actual_port}` (no `/v1/traces` suffix for gRPC).
- `actual_port` — the bound port.
- `start()` / `stop()` — start the gRPC server on a background thread; `stop()` discards the in-memory spans. Sync and async context managers are supported.

```python
rx = EphemeralOTLPGRPCReceiver(port=0).start()
try:
    # Target config: OTEL_EXPORTER_OTLP_ENDPOINT = rx.endpoint
    #               OTEL_EXPORTER_OTLP_PROTOCOL  = grpc
    ...
finally:
    rx.stop()
```

Requires `grpc` and the OpenTelemetry proto packages to be installed.

### `OTLPTraceReceiver`

The minimal async building block behind the receivers above — accept a POSTed OTLP JSON body, store the spans, and return the OTLP ack. Use it standalone (e.g. to ingest a saved export file) or embed it in your own ASGI framework:

```python
def __init__(self, store=None, authenticator=None, auth=None)
```

- `handle(body, authorization=None)` — async; parses `body` (str/bytes/dict), stores the spans, and returns the ack `{"partialSuccess": {"rejectedSpans": 0}}`. When an `authenticator` was provided, a failed check returns a 401 `{"error": ...}` payload and stores nothing.
- `trace_ids()` — the distinct trace ids seen so far.
- `store` — the backing `SpanStore`.

```python
import asyncio

from simpleaudit.tracing import OTLPTraceReceiver, SpanStore

async def main():
    store = SpanStore()
    receiver = OTLPTraceReceiver(store=store)
    body = {
        "resourceSpans": [
            {
                "resource": {"attributes": []},
                "scopeSpans": [
                    {"spans": [
                        {
                            "traceId": "c" * 32,
                            "spanId": "d" * 16,
                            "name": "retriever",
                            "kind": 1,
                            "startTimeUnixNano": 1_000_000_000,
                            "endTimeUnixNano": 2_000_000_000,
                            "attributes": [
                                {"key": "openinference.span.kind", "value": {"stringValue": "RETRIEVER"}},
                            ],
                        }
                    ]}
                ],
            }
        ]
    }
    ack = await receiver.handle(body)
    print(ack)                 # {'partialSuccess': {'rejectedSpans': 0}, 'authenticated': True}
    print(receiver.trace_ids())  # ['cccccccccccccccccccccccccccccccc']
    print(len(store.by_trace("c" * 32)))  # 1

asyncio.run(main())
```

### `parse_otlp_json`

Transport-agnostic parsing of an OTLP/HTTP JSON `ExportTraceServiceRequest` payload into raw span dicts. Accepts a dict, a JSON string, or bytes. It walks `resourceSpans[].scopeSpans[].spans`, decodes the OTLP `AnyValue` attribute oneofs (`stringValue`, `intValue`, `doubleValue`, `boolValue`, `arrayValue`, `kvlistValue`), merges resource-level attributes under span-level ones, converts nanosecond timestamps to Unix seconds, and maps status codes to `"OK"`/`"ERROR"`. Each returned span has `trace_id`, `span_id`, `parent_span_id`, `name`, `kind`, `start_time`, `end_time`, `status`, and `attributes`.

```python
from simpleaudit.tracing import parse_otlp_json

payload = {
    "resourceSpans": [
        {
            "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "open-webui"}}]},
            "scopeSpans": [
                {"spans": [
                    {
                        "traceId": "a" * 32,
                        "spanId": "b" * 16,
                        "name": "POST /chat",
                        "kind": 2,
                        "startTimeUnixNano": 1_000_000_000,
                        "endTimeUnixNano": 2_000_000_000,
                        "attributes": [{"key": "http.method", "value": {"stringValue": "POST"}}],
                        "status": {"code": 1},
                    }
                ]}
            ],
        }
    ]
}

spans = parse_otlp_json(payload)
print(spans[0]["name"])                          # POST /chat
print(spans[0]["attributes"]["service.name"])    # open-webui (merged from resource)
print(spans[0]["start_time"])                    # 1.0
```

## Storing and querying spans

### `normalize_span`

Normalizes a raw span (OTel/OpenInference-shaped) to the flat store schema: `span_id`, `trace_id`, `name`, `kind`, `attributes`, `parent_span_id`, `start_time`, `end_time`, `status`. It accepts the common OpenInference/OTel attribute keys, prefers the string kind attribute (`openinference.span.kind`, `span.kind`) over the OTLP proto int kind (which is mapped to names like `INTERNAL`/`SERVER`), and defaults the kind to `"CHAIN"` when absent.

### `SpanStore`

An in-memory, transport-agnostic store with the queries the audit engine needs:

| Method | Returns |
| --- | --- |
| `add(raw)` | the normalized span (a `uuid4` hex id is generated if the span has none) |
| `add_many(raws)` | list of normalized spans |
| `get(span_id)` | one span or `None` |
| `by_trace(trace_id)` | all spans for a trace |
| `by_kind(kind)` | spans matching a kind, case-insensitive (`RETRIEVER`, `TOOL`, `LLM`, ...) |
| `by_attribute(key, value)` | spans whose `attributes[key] == value` |
| `all()` | every stored span |
| `len(store)` | number of stored spans |

```python
from simpleaudit.tracing import SpanStore, normalize_span

store = SpanStore()
store.add({"span_id": "s1", "trace_id": "t1", "name": "retriever",
           "attributes": {"openinference.span.kind": "RETRIEVER"}})
store.add({"span_id": "s2", "trace_id": "t1", "name": "llm",
           "attributes": {"openinference.span.kind": "LLM"}})
store.add({"span_id": "s3", "trace_id": "t2", "name": "tool",
           "attributes": {"openinference.span.kind": "TOOL"}})

print([s["span_id"] for s in store.by_trace("t1")])      # ['s1', 's2']
print([s["span_id"] for s in store.by_kind("retriever")]) # ['s1']
print(len(store))                                         # 3
```

## Running an audit with tracing: `audit_with_tracing`

```python
async def audit_with_tracing(auditor, scenarios, *, provider=None, audit_run_id=None, token_budget=None, **run_kwargs)
```

The end-to-end helper. It:

1. Starts the provider (defaulting to `BuiltinOTLP()` when none is given) and generates an `audit_run_id` when not supplied.
2. Creates a `TraceCorrelation` and hooks it into providers that support `register_trace` (e.g. `SharedOTLP`).
3. Calls `auditor.run_async(scenarios, audit_run_id=..., trace_correlation=..., **run_kwargs)` inside the provider's context.
4. Collects and de-duplicates the spans for every recorded trace, selects the evidence-relevant kinds (honoring `token_budget`), and stops the provider.
5. Attaches the selected spans to each result under `result.judgment["evidence_spans"]` (each span carries provenance with its exact `trace_id`/`span_id` for a finding's `EvidenceRef`).

It does **not** mutate the target: for a target that reads its OTLP endpoint from the environment, set `OTEL_EXPORTER_OTLP_ENDPOINT` to `provider.endpoint` before the run. Note that per-scenario attribution is approximate when `max_workers > 1` (the correlation is shared across scenarios); the per-span provenance remains exact.

You can also wire tracing manually by passing `audit_run_id=` and `trace_correlation=` directly to `ModelAuditor.run_async` / `run` — see the offline example below.

## Offline usage (no API key)

The whole ingestion path — receiver, parser, normalization, store, and correlation — runs locally with no network calls and no model keys. This exercises `EphemeralOTLPReceiver`, `parse_otlp_json`, `normalize_span`, `SpanStore`, and `TraceCorrelation` end to end:

```python
import asyncio
import httpx

from simpleaudit.tracing import (
    EphemeralOTLPReceiver,
    SpanStore,
    TraceCorrelation,
    normalize_span,
    parse_otlp_json,
)

TRACE_ID = "a" * 32

async def main():
    # 1. Start the built-in OTLP receiver (binds an ephemeral localhost port).
    rx = EphemeralOTLPReceiver().start()
    try:
        # 2. Simulate the target exporting one OTLP/HTTP JSON batch.
        payload = {
            "resourceSpans": [
                {
                    "resource": {"attributes": []},
                    "scopeSpans": [
                        {"spans": [
                            {
                                "traceId": TRACE_ID,
                                "spanId": "b" * 16,
                                "name": "retriever",
                                "kind": 1,
                                "startTimeUnixNano": 1_000_000_000,
                                "endTimeUnixNano": 2_000_000_000,
                                "attributes": [
                                    {"key": "openinference.span.kind", "value": {"stringValue": "RETRIEVER"}},
                                ],
                            },
                            {
                                "traceId": TRACE_ID,
                                "spanId": "c" * 16,
                                "name": "chain",
                                "kind": 1,
                                "startTimeUnixNano": 2_000_000_000,
                                "endTimeUnixNano": 3_000_000_000,
                                "attributes": [
                                    {"key": "openinference.span.kind", "value": {"stringValue": "CHAIN"}},
                                ],
                            },
                        ]}
                    ],
                }
            ]
        }
        async with httpx.AsyncClient(timeout=10) as client:
            r = await client.post(rx.endpoint, json=payload)
            assert r.status_code == 200

        # 3. Query the store the same way the audit engine does.
        spans = rx.store.by_trace(TRACE_ID)
        assert {s["span_id"] for s in spans} == {"b" * 16, "c" * 16}
        retrievers = rx.store.by_kind("RETRIEVER")
        assert len(retrievers) == 1

        # 4. Correlate a turn with the trace, as the engine does per turn.
        corr = TraceCorrelation(audit_run_id="run_demo")
        corr.record("turn_1", TRACE_ID)
        turn_spans = corr.spans_for_turn("turn_1", rx.store)
        assert {s["span_id"] for s in turn_spans} == {"b" * 16, "c" * 16}
    finally:
        rx.stop()
    assert len(rx.store) == 0  # ephemeral: spans discarded on stop

    # 5. Transport-agnostic pieces work without any server at all.
    raw = parse_otlp_json(payload)
    assert raw[0]["trace_id"] == TRACE_ID
    norm = normalize_span(raw[0])
    assert norm["kind"] == "RETRIEVER" and norm["status"] == "OK"
    store = SpanStore()
    store.add_many(raw)
    assert len(store.by_trace(TRACE_ID)) == 2

asyncio.run(main())
```

To attach these spans to real audit results without a model judge, pass a code-only judge (e.g. `judge="choice_match"`, which grades in code and makes no judge LLM calls) plus an in-process `CallableTarget` — the tracing plumbing itself never touches a model.

## Where to go next

- [Targets And Judges](guides/targets-and-judges.md) — how targets and judges fit into the audit loop, including code-only judges used in the offline example.
- [Auditing Basics](guides/auditing-basics.md) — the core `ModelAuditor.run` flow that `audit_with_tracing` wraps.
- [Context Analysis](guides/context-analysis.md) — using retrieved-context evidence (the kind of spans this layer captures) in judgments.
- [Scenarios](guides/scenarios.md) — scenario packs and custom scenarios you can feed to a traced run.
- [Quickstart](guides/quickstart.md) — get an audit running first, then add tracing.

### See Also

*   [Span verification](context-analysis.md)
*   [How an audit works](quickstart.md)
*   [How a judge config works](custom-judges.md)
*   [Decision logic](auditing-basics.md)


**Container capabilities:** `SpanStore` supports `len()`.
