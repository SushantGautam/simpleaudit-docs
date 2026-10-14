# Tracing And Observability

SimpleAudit can capture the OpenTelemetry traces your target application emits during an audit, attach the relevant spans to each result as judge evidence, and aggregate the verdicts into interpretable statistics. The tracing layer is optional — black-box auditing works with no tracing at all — but when your target is instrumented it adds richer evidence (retrieved documents, tool calls, agent reasoning) that a trace-aware judge can inspect.

Everything lives in `simpleaudit.tracing` plus two pure functions in `simpleaudit.stats`.

## How a run gets its traces

During a run, the engine propagates a W3C `traceparent` header on every turn and records a `turn_id -> trace_id` mapping in a `TraceCorrelation`. After the run, the correlation's traces are fetched from a **trace provider**, de-duplicated, filtered down to evidence-relevant span kinds, and attached to each result under `result.judgment["evidence_spans"]` (with per-span provenance: exact `trace_id`/`span_id`).

There are two acquisition modes, both exposed through the same minimal contract:

```python
provider.start()
provider.endpoint          # only meaningful for BuiltinOTLP
provider.fetch(trace_id)   # -> normalized spans for that trace
provider.stop()
```

### 1. `BuiltinOTLP` — run your own ephemeral receiver

`BuiltinOTLP` starts an in-process OTLP receiver for the duration of the audit and holds the spans in memory; they are discarded on `stop()`. Point the (controlled) target's exporter at it:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=<provider.endpoint>
OTEL_EXPORTER_OTLP_PROTOCOL=http/json
```

```python
from simpleaudit.tracing import BuiltinOTLP

provider = BuiltinOTLP()        # host="127.0.0.1", port=0 (ephemeral), auth=None
assert provider.endpoint is None    # not started yet
provider.start()
try:
    print(provider.endpoint)        # "http://127.0.0.1:<port>/v1/traces"
    spans = provider.fetch("abc...")  # [] until the target exports
finally:
    provider.stop()
assert provider.endpoint is None    # gone after stop
```

The constructor takes `host: str = "127.0.0.1"`, `port: int = 0`, and `auth: Optional[OTLPAuth] = None`. It also exposes a `store` property (a `SpanStore`) and supports the context-manager form (`with BuiltinOTLP() as provider:`).

Under the hood `BuiltinOTLP` wraps an `EphemeralOTLPReceiver`: a self-contained **OTLP/HTTP JSON** server (`POST /v1/traces`) that runs on a background thread so it works from sync and async callers alike. Port `0` binds an ephemeral port, which you read back via `endpoint` / `actual_port`. `stop()` tears down the server *and* discards the in-memory spans — this receiver is ephemeral by design. If the target speaks the default gRPC protocol instead, either put an OTel Collector in front or use the gRPC receiver below.

### 2. `ExternalTraceProvider` — fetch from your existing backend

If the target already ships traces to Tempo, Jaeger, or an OTel Collector, subclass `ExternalTraceProvider` and implement `_fetch_remote(trace_id)` against your backend's trace API. SimpleAudit still propagates the `traceparent` during the run; afterwards `fetch(trace_id)` retrieves the matching trace. The customer never redirects telemetry.

```python
from simpleaudit.tracing import ExternalTraceProvider

class TempoProvider(ExternalTraceProvider):
    def _fetch_remote(self, trace_id):
        # GET {base}/api/traces/{trace_id} -> OTLP JSON -> parse_otlp_json(...)
        ...

p = TempoProvider()
spans = p.fetch("abc...")
```

A bare `ExternalTraceProvider().fetch(...)` raises `NotImplementedError` — the base class is a template, not a working source. Its `endpoint` property returns `None` (fetch-based providers have nothing to point at).

## The one-call flow: `audit_with_tracing`

`audit_with_tracing` is the Promptfoo-style helper that wires all of this together:

```python
async def audit_with_tracing(
    auditor,
    scenarios,
    *,
    provider=None,          # defaults to BuiltinOTLP()
    audit_run_id=None,      # defaults to "audit_<12 hex chars>"
    token_budget=None,      # passed to span selection
    **run_kwargs,           # forwarded to auditor.run_async(...)
)
```

It:

1. Starts the provider (default: `BuiltinOTLP`).
2. Runs `auditor.run_async(scenarios, audit_run_id=..., trace_correlation=..., **run_kwargs)` while the provider is alive.
3. Collects each scenario's spans (the union of its turns' traces) and selects the evidence-relevant ones (honoring `token_budget`).
4. Stops the provider.
5. Attaches the selected spans to each result's `judgment["evidence_spans"]` (only if any were found) and returns the `AuditResults`.

Note that the helper does **not** mutate the target. For a target that reads its OTLP endpoint from the environment, set `OTEL_EXPORTER_OTLP_ENDPOINT` to `provider.endpoint` before the run:

```python
import asyncio
import os
from simpleaudit.tracing import BuiltinOTLP, audit_with_tracing
from tests.fakes import fixed_probe_auditor, fixed_severity_judge, fixed_target, make_auditor

auditor = make_auditor(
    target=fixed_target("ok"),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("probe"),
    max_turns=1,
    show_progress=False,
)

async def _run():
    provider = BuiltinOTLP()
    provider.start()
    try:
        os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"] = provider.endpoint
        results = await audit_with_tracing(auditor, "safety", provider=provider, max_workers=1)
        for r in results.results:
            spans = (r.judgment or {}).get("evidence_spans", [])
            # each span carries span["provenance"]["trace_id"] / ["span_id"]
    finally:
        provider.stop()
        del os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"]

asyncio.run(_run())
```

If the target emits no spans, `evidence_spans` is simply not attached to the results.

## Receiving OTLP directly

### `OTLPTraceReceiver`

`OTLPTraceReceiver` is a minimal async OTLP/HTTP JSON receiver you can embed in any ASGI app or drive standalone:

```python
from simpleaudit.tracing import OTLPTraceReceiver, SpanStore

store = SpanStore()
receiver = OTLPTraceReceiver(store=store)

ack = await receiver.handle(open("export.json").read())
# ack == {"partialSuccess": {"rejectedSpans": 0}, "authenticated": True}

print(receiver.trace_ids())   # distinct trace ids seen so far
```

`handle(body, authorization=None)` ingests an OTLP JSON export body and returns the OTLP ack payload (`{"partialSuccess": {"rejectedSpans": 0}}`). When constructed with `authenticator` or `auth` (an `OTLPAuth`), the request's `Authorization` header is checked first and a failed check returns a 401 `{"error": ...}` payload without storing anything. Without auth the receiver is open — the historical behaviour for the local single-audit case.

### `EphemeralOTLPReceiver` and `EphemeralOTLPGRPCReceiver`

These are the self-contained servers described above, usable directly:

```python
from simpleaudit.tracing import EphemeralOTLPReceiver

with EphemeralOTLPReceiver() as rx:
    # rx.endpoint == "http://127.0.0.1:<port>/v1/traces"
    # point the target's OTEL_EXPORTER_OTLP_ENDPOINT at rx.endpoint here
    ...
    spans = rx.store.by_trace(trace_id)   # inspect / feed the judge
# rx.stop() ran: server stopped, spans discarded
```

Both accept `host: str = "127.0.0.1"`, `port: int = 0`, and `store: Optional[SpanStore] = None`; the HTTP variant additionally accepts `authenticator` and `auth`. Both expose `endpoint`, `actual_port`, `start()`, `stop()`, and sync + async context managers. A malformed export body never fails the export — the receiver acks with a rejection count so the target doesn't retry-loop, and the audit continues regardless.

`EphemeralOTLPGRPCReceiver` speaks **OTLP/gRPC** (`TraceService/Export`), what `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` (the default) selects on port 4317. Use it when the target (e.g. Open WebUI) exports over gRPC and you want to capture spans without running a full OTel Collector:

```python
from simpleaudit.tracing import EphemeralOTLPGRPCReceiver

rx = EphemeralOTLPGRPCReceiver(port=4317).start()
# target's OTEL_EXPORTER_OTLP_ENDPOINT = http://127.0.0.1:4317
...
rx.stop()
```

Its `endpoint` is `http://<host>:<port>` (no `/v1/traces` path — that's an HTTP thing). Like the HTTP receiver, `stop()` discards the in-memory spans.

### `parse_otlp_json`

`parse_otlp_json(payload)` converts an OTLP/HTTP JSON export body (a dict or a JSON string) into a list of raw span dicts ready for `SpanStore.add_many`. It walks `resourceSpans[*].scopeSpans[*].spans[*]` and maps each span's `traceId`/`spanId`/`name`/`kind`/`startTimeUnixNano`/`endTimeUnixNano` onto the store schema:

```python
from simpleaudit.tracing import SpanStore, parse_otlp_json

payload = {
    "resourceSpans": [
        {
            "scopeSpans": [
                {
                    "spans": [
                        {
                            "traceId": "a" * 32,
                            "spanId": "b" * 16,
                            "name": "LLM call",
                            "kind": 1,
                            "attributes": {"openinference.span.kind": "LLM"},
                        }
                    ]
                }
            ]
        }
    ]
}

raw_spans = parse_otlp_json(payload)
store = SpanStore()
store.add_many(raw_spans)
assert len(store) == 1
```

## Querying spans: `SpanStore` and `normalize_span`

All received spans land in a `SpanStore` — an in-memory, transport-agnostic store supporting the queries the audit engine needs: by `trace_id`, by span kind, and by attribute.

```python
from simpleaudit.tracing import SpanStore

store = SpanStore()
store.add({"span_id": "s1", "trace_id": "t1", "name": "retriever",
           "attributes": {"openinference.span.kind": "RETRIEVER"}})
store.add({"span_id": "s2", "trace_id": "t1", "name": "llm",
           "attributes": {"openinference.span.kind": "LLM"}})

store.by_trace("t1")       # both spans, in insertion order
store.by_kind("retriever") # kind match is case-insensitive
store.by_attribute("openinference.span.kind", "LLM")
store.get("s1")            # single span or None
store.all()                # everything
len(store)                 # 2
```

Every `add` passes the raw dict through `normalize_span(raw)`, which flattens OTel/OpenInference-shaped input into the canonical schema: `span_id`, `trace_id`, `name`, `kind`, `parent_span_id`, `attributes`, `start_time`, `end_time`, `status`. Details worth knowing:

- The span kind prefers the string attributes `openinference.span.kind` / `span.kind`; an OTLP proto-int kind (1=INTERNAL, 2=SERVER, ...) is mapped to its name; missing kinds become `"CHAIN"`.
- Missing `span_id`s get a fresh UUID hex id assigned.
- Timestamps arrive as unix seconds (converted from nanos by the parser); `status` defaults to `"OK"`.

You can also normalize spans yourself and query them without ever running a receiver:

```python
from simpleaudit.tracing import SpanStore, normalize_span

span = normalize_span({
    "span_id": "s1",
    "trace_id": "t1",
    "name": "retriever",
    "attributes": {"openinference.span.kind": "RETRIEVER"},
})
assert span["kind"] == "RETRIEVER"
assert span["status"] == "OK"
```

## Aggregating statistical metrics from audits

Once you have verdicts, `simpleaudit.stats` provides two dependency-free building blocks that turn a count of passing trials into something interpretable. They operate on plain counts (`k` successes out of `n` trials) — no runs, results, or storage — so the same formulas serve the in-process engine, a CLI, or an out-of-process caller tracking a recurring audit over time. Deciding which trials count as pass/fail/error is your domain logic and stays out of these functions.

### `wilson_interval(k, n, z=1.96)`

Wilson score interval for a binomial proportion — preferred over the naive Wald interval for small `n` or proportions near 0 or 1. Returns `(lower, upper)` clamped to `[0, 1]`; when `n == 0` there is no evidence, so it returns the full range `(0.0, 1.0)`.

### `two_proportion_z(k1, n1, k2, n2)`

Pooled two-proportion z statistic for `p2 - p1`: tests whether the second group's pass rate differs from the first (baseline), pooling the samples under the null hypothesis. A large magnitude (`|z| >= 1.96` for 95%) indicates a real difference rather than sampling noise. Returns `None` when undefined — either group has no trials, or the pooled standard error is zero (both groups agree perfectly, so there is no variance to test against).

```python
from simpleaudit.stats import wilson_interval, two_proportion_z

# 18 of 20 trials passed
lo, hi = wilson_interval(18, 20)
print(f"pass rate 95% CI: [{lo:.3f}, {hi:.3f}]")

# Baseline: 18/20 vs new build: 15/20
z = two_proportion_z(18, 20, 15, 20)
print(f"z = {z}")                      # negative: new build lower
print("significant at 95%" if z is not None and abs(z) >= 1.96 else "not significant")

# Edge cases
print(wilson_interval(0, 0))           # (0.0, 1.0)
print(two_proportion_z(5, 5, 5, 5))    # None (zero pooled SE)
print(two_proportion_z(0, 0, 3, 10))   # None (no baseline trials)
```

A typical pattern for a recurring audit: compute `k = results.passed` and `n = len(results)` from an `AuditResults` (see [Auditing Basics](guides/auditing-basics.md)), report `wilson_interval(k, n)` for the current run, and compare against the previous run's counts with `two_proportion_z` to flag regressions.

## Offline usage (no API key)

The whole tracing pipeline works offline with no model calls and no network beyond the loopback receiver. You can ingest a recorded OTLP export file straight into a store and query it:

```python
import json
from simpleaudit.tracing import SpanStore, parse_otlp_json

with open("export.json") as f:
    body = f.read()

store = SpanStore()
store.add_many(parse_otlp_json(body))
for tid in sorted({s["trace_id"] for s in store.all()}):
    print(tid, "->", [s["name"] for s in store.by_trace(tid)])
```

And the statistics need nothing but counts:

```python
from simpleaudit.stats import wilson_interval
print(wilson_interval(9, 10))   # (0.625, 0.982) roughly
```

For end-to-end offline audits (fake target/judge/auditor clients, no model keys), see the quickstart's fake-client helpers referenced throughout `tests/fakes.py`.

## Where to go next

- [Installation](guides/installation.md) — install SimpleAudit and its optional tracing dependencies (aiohttp for the HTTP receiver, grpc + opentelemetry-proto for the gRPC receiver).
- [Quickstart](guides/quickstart.md) — run your first audit and see where `audit_with_tracing` fits.
- [Auditing Basics](guides/auditing-basics.md) — the `AuditResults` object whose verdicts you feed into `wilson_interval` and `two_proportion_z`.
- [Targets And Judges](guides/targets-and-judges.md) — how targets emit spans and how judges consume `evidence_spans`.
- [Scenarios](guides/scenarios.md) — the scenario packs you pass to `audit_with_tracing`.
- [Context Analysis](guides/context-analysis.md) — deriving findings from the retrieved-context spans captured here.
- [Custom Judges](guides/custom-judges.md) — writing a trace-aware judge over the attached evidence spans.

### See Also

*   [The audit loop](auditing-basics.md)
*   [Span verification: `spans`](context-analysis.md)
*   [Building a judge from scratch: `build_judge`](custom-judges.md)
*   [What you'll build](quickstart.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `SpanStore` supports `len()`.
