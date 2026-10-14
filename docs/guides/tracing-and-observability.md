# Tracing And Observability

SimpleAudit can instrument an audit run with **OpenTelemetry (OTLP) tracing** so you can see — and later judge against — the internal execution flow of the target under test: retrieved documents, tool calls, agent reasoning, guardrail decisions, and LLM calls. The engine propagates a W3C `traceparent` header per turn; if your target is OTel-instrumented (OpenInference, OpenLLMetry, MLflow, raw OTel), its spans land in SimpleAudit's span store and are attached to each result as `evidence_spans`.

Tracing is optional: black-box auditing works with no tracing at all. This layer lives in `simpleaudit.tracing` and is fully importable from the package root:

```python
from simpleaudit.tracing import (
    BuiltinOTLP, ExternalTraceProvider, TraceProvider, audit_with_tracing,
    EphemeralOTLPReceiver, EphemeralOTLPGRPCReceiver, OTLPTraceReceiver,
    SpanStore, normalize_span, parse_otlp_json,
    TraceCorrelation, make_traceparent, new_trace_id,
    select_spans, evidence_spans_for_turn, summarize_for_judge,
)
```

## How it fits together

1. **Propagate.** For every scenario/turn, the auditor builds a trace id (`new_trace_id()`, 32 hex chars) and sends `trace_headers={"traceparent": ...}` on the `TargetContext` passed to your target. The value comes from `make_traceparent(trace_id)` and has the form `00-<32hex>-<16hex>-01`.
2. **Collect.** Spans arrive either at a built-in ephemeral OTLP receiver or are fetched afterwards from your own backend. Both paths go through `normalize_span` into a `SpanStore`.
3. **Correlate.** A `TraceCorrelation(audit_run_id=...)` maps `turn_id -> [trace_id]` via `record(turn_id, trace_id)`, and `spans_for_turn(turn_id, store)` collects every span belonging to that turn's traces (a turn may fan out to several traces).
4. **Select & attach.** Evidence-relevant spans are selected (kind filter + token budget) and attached to each `AuditResult` under `result.judgment["evidence_spans"]`, each carrying a `provenance` dict (`trace_id`, `span_id`, `kind`, `name`) so findings can reference them.

## Ingestion: parsing OTLP payloads

`parse_otlp_json(payload)` parses an OTLP/HTTP JSON `ExportTraceServiceRequest` (a dict, JSON string, or bytes) into raw span dicts. It walks `resourceSpans[].scopeSpans[].spans[]`, merges resource attributes into each span's attributes, decodes the OTLP `AnyValue` oneof (`stringValue`, `intValue`, `doubleValue`, `boolValue`, `arrayValue`, `kvlistValue`), converts nanosecond timestamps to Unix floats, and maps status codes to `"OK"` / `"ERROR"`.

`normalize_span(raw)` then flattens a raw span (OTel/OpenInference-shaped) to the store schema: `span_id`, `trace_id`, `name`, `kind`, `attributes`, `parent_span_id`, `start_time`, `end_time`, `status`. Kind handling: the string attribute `openinference.span.kind` / `span.kind` wins; an int kind (the OTLP proto enum) is mapped by name (`1 -> INTERNAL`, `2 -> SERVER`, `3 -> CLIENT`, ...); missing kinds default to `"CHAIN"`.

```python
import json
from simpleaudit.tracing import SpanStore, parse_otlp_json

payload = {
    "resourceSpans": [
        {
            "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "agent-app"}}]},
            "scopeSpans": [
                {
                    "spans": [
                        {
                            "traceId": "a" * 32,
                            "spanId": "b" * 16,
                            "name": "retriever",
                            "kind": 1,
                            "startTimeUnixNano": 1_000_000_000,
                            "endTimeUnixNano": 2_000_000_000,
                            "attributes": [
                                {"key": "openinference.span.kind", "value": {"stringValue": "RETRIEVER"}},
                                {"key": "docs", "value": {"stringValue": "doc1,doc2"}},
                            ],
                            "status": {"code": 1},
                        }
                    ]
                }
            ],
        }
    ]
}

raw_spans = parse_otlp_json(payload)          # also accepts json.dumps(payload)
store = SpanStore()
store.add_many(raw_spans)

span = store.by_trace("a" * 32)[0]
print(span["name"])                           # "retriever"
print(span["attributes"]["service.name"])     # "agent-app" (resource attr merged)
print(span["attributes"]["openinference.span.kind"])  # "RETRIEVER"
print(span["start_time"], span["end_time"])   # 1.0 2.0
```

## SpanStore: querying ingested spans

`SpanStore` is an in-memory store (no storage dependency) keyed by `span_id`, with indexes for the queries the audit engine needs:

- `add(raw)` / `add_many(raws)` — normalize and store (missing `span_id`s get a fresh UUID);
- `get(span_id)` — one span or `None`;
- `by_trace(trace_id)` — all spans of a trace;
- `by_kind(kind)` — case-insensitive kind match, e.g. `by_kind("retriever")` finds `RETRIEVER` spans;
- `by_attribute(key, value)` — e.g. `by_attribute("simpleaudit.turn_id", "turn_5")`;
- `all()` and `len(store)`.

## Receiving OTLP exports

Three receivers accept spans from instrumented targets:

### OTLPTraceReceiver

A minimal async receiver you can embed in any ASGI app or drive standalone. `handle(body, authorization=None)` ingests an OTLP JSON export body, stores the spans, and returns the OTLP ack `{"partialSuccess": {"rejectedSpans": 0}}`. `trace_ids()` lists the distinct trace ids seen so far. An optional auth gate (`auth` / `authenticator`) rejects bad credentials with a 401 payload before anything is stored.

```python
import asyncio
from simpleaudit.tracing import OTLPTraceReceiver

async def main():
    receiver = OTLPTraceReceiver()
    ack = await receiver.handle(json.dumps(payload))
    assert ack["partialSuccess"]["rejectedSpans"] == 0
    print(len(receiver.store))      # 1
    print(receiver.trace_ids())     # ['aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa']

asyncio.run(main())
```

### EphemeralOTLPReceiver (OTLP/HTTP JSON)

A self-contained HTTP server (aiohttp on a background thread) that lives for one audit. It speaks the **OTLP/HTTP JSON** protocol (`POST /v1/traces`), which is what `OTEL_EXPORTER_OTLP_PROTOCOL=http/json` selects. Port `0` binds an ephemeral port; `endpoint` gives the full URL (`http://127.0.0.1:<port>/v1/traces`) and `actual_port` the bound port. It supports sync and async context managers. Spans are discarded on `stop()` — ephemeral by design.

```python
import asyncio
import httpx
from simpleaudit.tracing import EphemeralOTLPReceiver

async def main():
    rx = EphemeralOTLPReceiver().start()
    try:
        # Point your target's exporter here:
        #   OTEL_EXPORTER_OTLP_ENDPOINT=<rx.endpoint>
        #   OTEL_EXPORTER_OTLP_PROTOCOL=http/json
        async with httpx.AsyncClient(timeout=10) as client:
            r = await client.post(rx.endpoint, json=payload)
            assert r.status_code == 200
            assert r.json()["partialSuccess"] == {"rejectedSpans": 0}
        print(len(rx.store))                 # 1
        print(rx.store.by_trace("a" * 32)[0]["name"])
    finally:
        rx.stop()
    assert len(rx.store) == 0                # discarded on stop

asyncio.run(main())
```

### EphemeralOTLPGRPCReceiver (OTLP/gRPC)

The same idea over the **OTLP/gRPC** protocol (`TraceService/Export`), which is what the default `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` selects on port 4317. Useful when the target exports over gRPC and you don't want to run a full OTel Collector. The gRPC server runs on a background thread; `endpoint` is `http://<host>:<port>` (no path), and spans are likewise discarded on `stop()`.

```python
from simpleaudit.tracing import EphemeralOTLPGRPCReceiver

with EphemeralOTLPGRPCReceiver(port=0) as rx:
    # Target config: OTEL_EXPORTER_OTLP_ENDPOINT=<rx.endpoint>
    #   OTEL_EXPORTER_OTLP_PROTOCOL=grpc
    ...  # run the audit while the target exports spans
# rx.store is cleared here
```

## Trace providers

Both acquisition modes implement the small `TraceProvider` contract, so the audit engine doesn't care which mode is in use:

```
provider.start()
provider.endpoint          # only meaningful for BuiltinOTLP
provider.fetch(trace_id)   # -> normalized spans for that trace
provider.stop()
```

`TraceProvider` provides defaults (`endpoint` → `None`, `fetch` → `[]`) and context-manager support (`__enter__` starts, `__exit__` stops).

### BuiltinOTLP

Runs an ephemeral OTLP receiver for the duration of the audit and holds spans in memory. Construct with `BuiltinOTLP(host="127.0.0.1", port=0, auth=None)`. Before `start()`, `endpoint` is `None`; after, it points at the receiver. `store` exposes the backing `SpanStore`, and `fetch(trace_id)` returns `store.by_trace(trace_id)`. Spans are discarded on `stop()`.

```python
from simpleaudit.tracing import BuiltinOTLP

provider = BuiltinOTLP()
assert provider.endpoint is None      # not started yet
provider.start()
try:
    print(provider.endpoint)          # http://127.0.0.1:<port>/v1/traces
    assert provider.fetch("nope") == []
finally:
    provider.stop()
assert provider.endpoint is None
```

### ExternalTraceProvider

For targets that already ship traces to their own observability backend (Tempo, Jaeger, an OTel Collector, etc.). SimpleAudit still propagates the `traceparent` during the run; afterwards you fetch the matching trace by id. Subclass it and implement `_fetch_remote(trace_id)` — calling `fetch` on an unimplemented subclass raises `NotImplementedError`:

```python
from simpleaudit.tracing import ExternalTraceProvider

class TempoProvider(ExternalTraceProvider):
    def _fetch_remote(self, trace_id):
        # GET {base}/api/traces/{trace_id} -> OTLP JSON -> parse
        return parse_otlp_json(response_body)

p = TempoProvider()
spans = p.fetch("a" * 32)
```

## Running an audited run: audit_with_tracing

`audit_with_tracing(auditor, scenarios, *, provider=None, audit_run_id=None, token_budget=None, **run_kwargs)` is the one-call, Promptfoo-style flow for a **controlled target** you can point at an OTLP endpoint:

1. Starts the provider (default: a `BuiltinOTLP` ephemeral receiver).
2. **You** point the target's `OTEL_EXPORTER_OTLP_ENDPOINT` at `provider.endpoint` (with `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`) for the duration of the run — the helper does not mutate the target.
3. Runs `auditor.run_async(scenarios, audit_run_id=..., trace_correlation=correlation, **run_kwargs)`, where the engine propagates a W3C `traceparent` per turn and records `turn_id -> trace_id` in a `TraceCorrelation`.
4. Collects each scenario's spans (union of its turns' traces) and selects evidence-relevant ones within `token_budget`.
5. Stops the provider (discarding spans for `BuiltinOTLP`).

It returns an `AuditResults` whose per-result `judgment["evidence_spans"]` carry the selected spans with provenance — ready to feed a trace-aware judge or a finding's evidence reference. If a provider supports `register_trace` (e.g. `SharedOTLP`), the correlation hooks into it automatically.

```python
import asyncio
from simpleaudit.tracing import BuiltinOTLP, audit_with_tracing
from tests.fakes import fixed_probe_auditor, fixed_severity_judge, fixed_target, make_auditor

auditor = make_auditor(
    target=fixed_target("ok"),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("probe"),
    max_turns=1,
    show_progress=False,
)

async def main():
    results = await audit_with_tracing(auditor, "safety", max_workers=2)
    for r in results.results:
        # Present only when the target actually emitted spans:
        spans = (r.judgment or {}).get("evidence_spans", [])
        for s in spans:
            print(s["provenance"]["trace_id"], s["provenance"]["span_id"])

asyncio.run(main())
```

If your target emits spans, they show up per result. A target that POSTs an OTLP span to `provider.endpoint` on each `send()` (reading the trace id from `context.trace_headers["traceparent"]`) will have its spans attached:

```python
import asyncio
import httpx
from simpleaudit.targets.base import TargetResponse
from simpleaudit.tracing import BuiltinOTLP, audit_with_tracing
from tests.fakes import fixed_probe_auditor, fixed_severity_judge, make_auditor

class EmittingTarget:
    """Fake target that exports one OTLP span per turn."""
    def __init__(self, endpoint: str):
        self.endpoint = endpoint

    async def send(self, *, user, history=None, context=None, **kw):
        tp = (context.trace_headers.get("traceparent") if context else "") or ""
        tid = tp.split("-")[1] if tp and len(tp.split("-")) >= 2 else "d" * 32
        payload = {
            "resourceSpans": [{
                "resource": {"attributes": []},
                "scopeSpans": [{
                    "spans": [{
                        "traceId": tid,
                        "spanId": "c" * 16,
                        "name": "LLM call",
                        "kind": 2,
                        "startTimeUnixNano": 1_000_000_000,
                        "endTimeUnixNano": 2_000_000_000,
                        "attributes": [
                            {"key": "openinference.span.kind", "value": {"stringValue": "LLM"}},
                        ],
                        "status": {"code": 1},
                    }]
                }],
            }]
        }
        async with httpx.AsyncClient(timeout=10) as client:
            await client.post(self.endpoint, json=payload)
        return TargetResponse(content="ok")

async def main():
    provider = BuiltinOTLP()
    provider.start()
    try:
        auditor = make_auditor(
            target=EmittingTarget(provider.endpoint),
            judge=fixed_severity_judge("pass"),
            auditor=fixed_probe_auditor("probe"),
            max_turns=1,
            show_progress=False,
        )
        results = await audit_with_tracing(auditor, "safety", provider=provider, max_workers=1)
        with_evidence = [r for r in results.results if (r.judgment or {}).get("evidence_spans")]
        sample = with_evidence[0].judgment["evidence_spans"][0]
        print(sample["provenance"]["trace_id"], sample["provenance"]["span_id"])
    finally:
        provider.stop()

asyncio.run(main())
```

## Selecting evidence spans

`select_spans(spans, *, evidence_kinds=DEFAULT_EVIDENCE_KINDS, noise_kinds=DEFAULT_NOISE_KINDS, token_budget=None, prefer_failing_spans=False, selector_version="v1")` implements the selection policy: keep evidence kinds (`RETRIEVER`, `TOOL`, `GUARDRAIL`, `EVALUATOR`, `AGENT`, `LLM`), drop noise (`CHAIN`, `EMBEDDING`, `PROMPT`), cap total serialized size with `token_budget`, record elisions, and stamp each selected span with provenance. It returns a `SelectionResult` with `selected`, `elided`, `budget_used`, and the `elided_count` property.

`evidence_spans_for_turn(correlation, store, turn_id, token_budget=None)` pulls the turn's spans from the store via `correlation.spans_for_turn(...)`, runs the selection, and returns the surviving spans with provenance. When evidence exists, noise kinds are dropped; when nothing is linked to the turn, it returns `[]`.

```python
from simpleaudit.tracing import SpanStore, TraceCorrelation, evidence_spans_for_turn

corr = TraceCorrelation(audit_run_id="run_1")
corr.record("turn_1", "traceA")

store = SpanStore()
store.add({"span_id": "s1", "trace_id": "traceA", "name": "retriever",
           "attributes": {"openinference.span.kind": "RETRIEVER"}})
store.add({"span_id": "s2", "trace_id": "traceA", "name": "chain",
           "attributes": {"openinference.span.kind": "CHAIN"}})

spans = evidence_spans_for_turn(corr, store, "turn_1")
print([s["span_id"] for s in spans])          # ['s1'] — CHAIN noise dropped
print(spans[0]["provenance"]["trace_id"])     # 'traceA'
```

## Offline usage (no API key)

Everything above runs without network access or API keys. The fakes in `tests/fakes.py` wire deterministic fake clients into a real `ModelAuditor`, and the ephemeral receivers bind localhost-only ports:

```python
import asyncio
from simpleaudit.tracing import BuiltinOTLP, audit_with_tracing
from tests.fakes import fixed_probe_auditor, fixed_severity_judge, fixed_target, make_auditor

auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("Tell me more about this."),
    max_turns=1,
    show_progress=False,
)

results = asyncio.run(audit_with_tracing(auditor, "safety", max_workers=2))
print(results.summary())
```

Because the fake target emits no spans, no `evidence_spans` key is attached — but the whole pipeline (provider lifecycle, correlation, selection) executes identically to a live, instrumented target.

## Where to go next

- [Auditing Basics](guides/auditing-basics.md) — how `run_async` / `run` produce the `AuditResults` that tracing decorates.
- [Targets And Judges](guides/targets-and-judges.md) — how targets receive `trace_headers` on the `TargetContext` and how judges consume the attached evidence.
- [Scenarios](guides/scenarios.md) — the scenario packs fed into `audit_with_tracing`.
- [Custom Judges](guides/custom-judges.md) — building a trace-aware judge over `evidence_spans`.
- [Quickstart](guides/quickstart.md) — running your first audit end to end.

### See Also

*   [Span verification](context-analysis.md)
*   [Customize an existing judge: `customize_judge`](custom-judges.md)
*   [Anatomy of a scenario](scenarios.md)
*   [Decision logic](auditing-basics.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `SpanStore` supports `len()`.
