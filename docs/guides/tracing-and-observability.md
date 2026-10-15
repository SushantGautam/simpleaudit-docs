# Tracing And Observability

SimpleAudit can capture OpenTelemetry (OTLP) traces from the system under test while an audit runs. The traces give you two things: **evidence** — retrieved documents, tool calls, and agent reasoning steps that a judge or a human can inspect per scenario — and **performance observability** — every span carries `start_time`/`end_time`, so you can see where wall-clock time goes inside each turn.

Tracing is optional: black-box auditing works with no tracing at all. This page covers how to enable OTLP tracing, how spans are stored and queried, and how to secure the receiver with credentials.

## How it fits together

The layer lives in `simpleaudit.tracing`:

| Piece | Role |
|---|---|
| `TraceProvider` / `BuiltinOTLP` / `ExternalTraceProvider` | Pluggable source of traces for an audit (`provider.py`) |
| `EphemeralOTLPReceiver` / `EphemeralOTLPGRPCReceiver` / `OTLPTraceReceiver` | OTLP ingestion servers (`otlp.py`) |
| `SpanStore` / `normalize_span` | In-memory storage + query API (`store.py`) |
| `OTLPAuth`, `AuthResult`, `Authenticator` and friends | Credential store + verification (`auth.py`) |
| `audit_with_tracing` | End-to-end helper that wires it all to an auditor run (`provider.py`) |

Two acquisition modes are supported (mirroring Promptfoo):

1. **Built-in receiver** (`BuiltinOTLP`) — SimpleAudit runs its own ephemeral OTLP receiver for the duration of the audit and you point the (controlled) target's `OTEL_EXPORTER_OTLP_ENDPOINT` at it. Best for CI, local apps, staging, and agent SDKs you control. No external collector required.
2. **External backend** (`ExternalTraceProvider`) — the target already ships traces to its own observability backend (Tempo, Jaeger, an OTel Collector, etc.). SimpleAudit propagates the W3C `traceparent` during the run and *fetches* the matching trace by id afterwards. The customer never redirects telemetry.

Both expose the same minimal contract, so the audit engine does not care which mode is in use:

```python
provider.start()
provider.endpoint          # only meaningful for BuiltinOTLP
provider.fetch(trace_id)   # -> normalized spans for that trace
provider.stop()
```

`TraceProvider` also works as a context manager (`with provider:`), calling `start()` on entry and `stop()` on exit.

## Running an audit with tracing: `audit_with_tracing`

`audit_with_tracing` is the high-level flow for a controlled target you can point at an OTLP endpoint:

```python
async def audit_with_tracing(
    auditor,
    scenarios,
    *,
    provider=None,        # defaults to BuiltinOTLP()
    audit_run_id=None,    # defaults to "audit_<12 hex chars>"
    token_budget=None,    # passed to span selection
    **run_kwargs,         # forwarded to auditor.run_async(...)
)
```

It does the following:

1. Starts the provider (default: a `BuiltinOTLP` ephemeral receiver).
2. Runs the audit via `auditor.run_async(scenarios, audit_run_id=..., trace_correlation=correlation, **run_kwargs)`. The engine propagates a W3C `traceparent` per turn and records `turn_id -> trace_id` in a `TraceCorrelation`.
3. After the run, collects each scenario's spans (union of its turns' traces), de-duplicates them, and selects the evidence-relevant kinds (optionally bounded by `token_budget`).
4. Stops the provider (discarding spans for `BuiltinOTLP`).
5. Attaches the selected spans to each result under `result.judgment["evidence_spans"]`, ready to feed a trace-aware judge.

It returns an `AuditResults` object. Note that the helper does **not** mutate the target: for a target that reads its OTLP endpoint from the environment, set `OTEL_EXPORTER_OTLP_ENDPOINT` to `provider.endpoint` before the run.

## Built-in receivers

### `BuiltinOTLP`

```python
from simpleaudit.tracing import BuiltinOTLP

provider = BuiltinOTLP(host="127.0.0.1", port=0, auth=None)  # port 0 = ephemeral
provider.start()
print(provider.endpoint)   # e.g. http://127.0.0.1:54321/v1/traces
spans = provider.fetch(trace_id)   # normalized spans for one trace
store = provider.store             # the underlying SpanStore
provider.stop()                    # discards the in-memory spans
```

`BuiltinOTLP.__init__(host="127.0.0.1", port=0, auth=None)` takes an optional `OTLPAuth` instance. Its `endpoint` property returns the OTLP/HTTP traces URL to configure the target's exporter to (with `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`); `store` exposes the backing `SpanStore`; `fetch(trace_id)` returns `store.by_trace(trace_id)`.

### `EphemeralOTLPReceiver`

The lower-level building block behind `BuiltinOTLP`: a self-contained OTLP/HTTP JSON trace receiver that lives for one audit. It speaks the **OTLP/HTTP JSON** protocol (`POST /v1/traces`, what `OTEL_EXPORTER_OTLP_PROTOCOL=http/json` selects) and also accepts protobuf bodies. The server runs on a background thread (aiohttp), so it works from both sync and async callers; port `0` binds an ephemeral port.

```python
from simpleaudit.tracing import EphemeralOTLPReceiver

rx = EphemeralOTLPReceiver(host="127.0.0.1", port=0, store=None, authenticator=None, auth=None)
rx.start()                      # binds an ephemeral port (bounded retry, ~10 s timeout)
print(rx.endpoint)              # http://127.0.0.1:<port>/v1/traces
print(rx.actual_port)           # the real bound port
...                             # run the audit; target exports here
rx.stop()                       # stops the server AND discards the in-memory spans
```

Key details:

- `endpoint` → `http://{host}:{actual_port}/v1/traces`; `actual_port` → the real port once started.
- `start()` retries the bind up to 3 times (under load, e.g. CI, the first bind can be slow) and raises `RuntimeError("EphemeralOTLPReceiver failed to start")` if all attempts fail.
- `stop()` tears down the server and replaces the store with a fresh empty `SpanStore` — the receiver is ephemeral by design.
- It supports both sync (`with rx:`) and async (`async with rx:`) context managers.
- Parse errors never fail the export: a bad body gets a 200 ack with `rejectedSpans: 1` so the target doesn't retry-loop, and the audit continues regardless.

### `EphemeralOTLPGRPCReceiver`

For targets that export over the default **OTLP/gRPC** protocol (`TraceService/Export`, port 4317), use this instead — no full OTel Collector needed:

```python
from simpleaudit.tracing import EphemeralOTLPGRPCReceiver

rx = EphemeralOTLPGRPCReceiver(host="127.0.0.1", port=0, store=None)
rx.start()
print(rx.endpoint)              # http://127.0.0.1:<port>  (no /v1/traces path)
...                             # target's OTEL_EXPORTER_OTLP_ENDPOINT = rx.endpoint
rx.stop()
```

Same shape as the HTTP receiver: `endpoint` and `actual_port` properties, `start()`/`stop()`, sync and async context managers, and spans discarded on `stop()`. Requires `grpc` and the `opentelemetry-proto` package to serve.

### `OTLPTraceReceiver`

A minimal async OTLP/HTTP JSON receiver you can embed in any ASGI app or drive directly — useful when you don't need a standalone server:

```python
import asyncio
from simpleaudit.tracing import OTLPTraceReceiver, SpanStore, parse_otlp_json

payload = {
    "resourceSpans": [{
        "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "agent-app"}}]},
        "scopeSpans": [{
            "spans": [{
                "traceId": "a" * 32,
                "spanId": "b" * 16,
                "name": "retrieve",
                "kind": 1,
                "startTimeUnixNano": 1_000_000_000,
                "endTimeUnixNano": 2_000_000_000,
                "attributes": [{"key": "openinference.span.kind", "value": {"stringValue": "RETRIEVER"}}],
                "status": {"code": 1},
            }],
        }],
    }]
}

receiver = OTLPTraceReceiver(store=SpanStore(), authenticator=None, auth=None)
ack = asyncio.run(receiver.handle(payload))
assert ack["partialSuccess"]["rejectedSpans"] == 0
assert len(receiver.store) == 1
assert receiver.trace_ids() == ["a" * 32]
```

- `handle(body, authorization=None)` ingests an OTLP JSON export body (dict, str, or bytes) and returns the OTLP ack payload. When an authenticator is configured, a failed check returns a 401 `{"error": ...}` payload and stores nothing.
- `trace_ids()` lists the distinct trace ids seen in the store.
- For embedding in an ASGI framework: route `POST /v1/traces` to `await receiver.handle(request_body_bytes)`.

### `parse_otlp_json`

Standalone parser for an OTLP/HTTP JSON `ExportTraceServiceRequest` into raw span dicts. Accepts a dict, JSON string, or bytes. It merges resource attributes into each span's attributes, decodes OTLP `AnyValue` values (`stringValue`, `intValue`, `doubleValue`, `boolValue`, `arrayValue`, `kvlistValue`), converts nanosecond timestamps to unix seconds, and maps status codes to `"OK"`/`"ERROR"`.

## Storing and querying spans: `SpanStore`

Ingested spans are normalized by `normalize_span(raw)` to a flat schema — `span_id`, `trace_id`, `name`, `kind`, `attributes`, `parent_span_id`, `start_time`, `end_time`, `status` — and kept in an in-memory `SpanStore`. `normalize_span` prefers the OpenInference/OTel string kind attribute (`openinference.span.kind`, e.g. `RETRIEVER`, `TOOL`, `AGENT`, `LLM`) and falls back to the OTLP proto int kind mapped to its name; missing kinds become `"CHAIN"`.

```python
from simpleaudit.tracing import SpanStore

store = SpanStore()
store.add({"span_id": "s1", "trace_id": "t1", "name": "retrieve",
           "attributes": {"openinference.span.kind": "RETRIEVER"}})
store.add_many([{"span_id": "s2", "trace_id": "t1", "name": "llm",
                 "attributes": {"openinference.span.kind": "LLM"}}])

store.get("s1")                        # one span by id (or None)
store.by_trace("t1")                   # all spans of a trace
store.by_kind("RETRIEVER")             # case-insensitive kind query
store.by_attribute("simpleaudit.target_id", "owui_17")
store.all()                            # every span
len(store)
```

Because every span keeps `start_time` and `end_time`, performance monitoring is just arithmetic over the store:

```python
for span in store.by_trace(trace_id):
    if span["start_time"] is not None and span["end_time"] is not None:
        print(span["name"], f"{span['end_time'] - span['start_time']:.3f}s")
```

## External backends: `ExternalTraceProvider`

When the target already exports to Tempo, Jaeger, or an OTel Collector, subclass `ExternalTraceProvider` and implement `_fetch_remote(trace_id)` against your backend's trace API. SimpleAudit propagates the `traceparent` during the run; after the response, call `fetch(trace_id)` to retrieve the spans. The base class raises `NotImplementedError` if `_fetch_remote` is not overridden:

```python
from simpleaudit.tracing import ExternalTraceProvider

class TempoProvider(ExternalTraceProvider):
    def __init__(self, base_url):
        self.base_url = base_url

    def _fetch_remote(self, trace_id):
        # GET {base_url}/api/traces/{trace_id} -> OTLP JSON -> parse_otlp_json(...)
        raise NotImplementedError
```

Pass the provider to `audit_with_tracing(..., provider=...)`; since `ExternalTraceProvider.endpoint` is `None`, there is nothing to point the target at — it keeps exporting to its existing backend.

## Securing the receiver: `OTLPAuth` and friends

By default the receivers are open (no auth check) — fine for a local, single-audit ephemeral receiver on a trusted network. When multiple targets share a receiver, use `OTLPAuth` to answer "who is allowed to submit?" and map each credential to a stable `target_id` stamped onto every ingested span.

Three modes:

- `none` — no check (the default).
- `basic` — `Authorization: Basic base64(username:password)`. Best for targets like OpenWebUI that natively support `OTEL_BASIC_AUTH_USERNAME` / `OTEL_BASIC_AUTH_PASSWORD`.
- `bearer` — `Authorization: Bearer <token>`. Best for generic OTel exporters that can set arbitrary headers (`OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <t>`).

Secrets are stored as salted SHA-256 hashes and compared in constant time (`hmac.compare_digest`) to avoid timing side-channels. Credentials are revocable and rotatable.

```python
from simpleaudit.tracing import OTLPAuth, generate_password, generate_token

auth = OTLPAuth(mode="basic")
password = generate_password()          # url-safe random secret
auth.add_basic("sa_owui_17", password, target_id="owui_17")

token = generate_token()                # "sa_otlp_" + random secret
auth.add_bearer(token, target_id="agent-2")

result = auth.verify("Basic c2Ffb3d1aV8xNzpwYXNz")   # AuthResult
if result.authenticated:                # alias for result.ok
    print(result.identity)              # "owui_17" (alias for result.target_id)

auth.rotate_basic("sa_owui_17", new_password=generate_password())
auth.rotate_bearer(old_token=token, new_token=generate_token())
auth.revoke("sa_owui_17")               # True if a credential was found and revoked
```

`verify(authorization_header)` returns an `AuthResult` with `.ok`, `.target_id`, `.reason` (plus the `.authenticated` / `.identity` aliases). Failure reasons include `missing_authorization_header`, `unknown_username`, `invalid_credentials`, `unknown_token`, and scheme mismatches such as `expected_basic_scheme`.

Hand the `OTLPAuth` instance to any receiver — `OTLPTraceReceiver(auth=...)`, `EphemeralOTLPReceiver(auth=...)`, or `BuiltinOTLP(auth=...)`. On each request the receiver calls `verify` with the raw `Authorization` header; a failed check yields a 401 and no spans are stored. A successful check tags every span in the batch with the matched identity via `tag_spans(spans, target_id)`, which stamps the attribute `simpleaudit.target_id` in place — making spans queryable later with `store.by_attribute("simpleaudit.target_id", ...)`.

### Low-level credential primitives

Everything above is built from small pure functions in `simpleaudit.tracing.auth`, which you can use directly:

```python
from simpleaudit.tracing import (
    hash_secret, verify_secret, new_salt,
    parse_basic_header, parse_bearer_header,
    token_lookup_prefix, make_basic_bearer_authenticator,
)

salt = new_salt()                       # 16 random bytes
digest = hash_secret("hunter2", salt)   # deterministic for a given salt
assert verify_secret("hunter2", salt, digest) is True
assert verify_secret("hunter3", salt, digest) is False

parse_basic_header("Basic c2E6cGFzcw==")      # ("sa", "pass") — password may contain ':'
parse_bearer_header("Bearer sa_otlp_xxx")     # "sa_otlp_xxx"
token_lookup_prefix("sa_otlp_abcdefghijklmnop")  # first 16 chars, stable lookup key

def lookup(authorization):
    parsed = parse_basic_header(authorization)
    return "owui_17" if parsed == ("sa_owui_17", "p4ss") else None

authenticator = make_basic_bearer_authenticator(lookup)   # Authenticator callable
result = authenticator("Basic c2Ffb3d1aV8xNzpwYXNz")      # AuthResult(ok=True, target_id="owui_17")
```

- `new_salt()` → 16 random bytes; `hash_secret(secret, salt)` → salted SHA-256 digest; `verify_secret(secret, salt, expected_hash)` → constant-time comparison (rejects empty salt/hash).
- `parse_basic_header` / `parse_bearer_header` return `None` for missing or malformed headers.
- `token_lookup_prefix(token)` returns the first 16 characters of a token.
- `make_basic_bearer_authenticator(verify)` wraps a `verify(authorization) -> Optional[target_id]` callback into an `Authenticator` — a callable returning an `AuthResult` with `ok` set when the lookup succeeded. Pass the resulting `Authenticator` as `authenticator=` to any receiver (an alternative to passing a full `OTLPAuth` as `auth=`).

## Offline usage (no API key)

The entire tracing layer is offline: no model, no LLM API key, and (for `OTLPTraceReceiver`) no network server. You can ingest a recorded OTLP export file, authenticate it, and query the store locally:

```python
import asyncio, base64, json

from simpleaudit.tracing import (
    OTLPAuth, OTLPTraceReceiver, SpanStore, parse_otlp_json,
    generate_password,
)

# A captured OTLP/HTTP JSON export (e.g. saved from a previous run).
payload = {
    "resourceSpans": [{
        "resource": {"attributes": []},
        "scopeSpans": [{
            "spans": [{
                "traceId": "c" * 32, "spanId": "d" * 16, "name": "tool_call",
                "kind": 1, "startTimeUnixNano": 1_000_000_000,
                "endTimeUnixNano": 1_500_000_000,
                "attributes": [{"key": "openinference.span.kind",
                                "value": {"stringValue": "TOOL"}}],
            }],
        }],
    }]
}

# 1. Pure parsing, no server at all:
spans = parse_otlp_json(json.dumps(payload))
print(spans[0]["kind"], spans[0]["end_time"] - spans[0]["start_time"])

# 2. Authenticated ingestion through the async receiver, still fully local:
auth = OTLPAuth(mode="basic")
password = generate_password()
auth.add_basic("local_target", password, target_id="local_target")
header = "Basic " + base64.b64encode(f"local_target:{password}".encode()).decode()

receiver = OTLPTraceReceiver(store=SpanStore(), auth=auth)
ack = asyncio.run(receiver.handle(payload, authorization=header))
assert ack["partialSuccess"]["rejectedSpans"] == 0
assert len(receiver.store) == 1
assert receiver.store.by_attribute("simpleaudit.target_id", "local_target")

bad_ack = asyncio.run(receiver.handle(payload, authorization="Basic bm9uZTpub25l"))
assert bad_ack["status"] == 401
```

To exercise the threaded `EphemeralOTLPReceiver` end-to-end without any external service, POST the payload to its `endpoint` with a plain HTTP client — everything stays on `127.0.0.1`.

## Where to go next

- [Auditing Basics](auditing-basics.md) — how audits, scenarios, and results work, including where `judgment["evidence_spans"]` lands.
- [Targets And Judges](targets-and-judges.md) — the targets whose telemetry you capture, and the judges that consume the selected spans.
- [Scenarios](scenarios.md) — the scenario packs you pass to `audit_with_tracing`.
- [Context Analysis](context-analysis.md) — turning captured evidence (retrieved docs, tool calls) into findings.
- [Custom Judges](custom-judges.md) — writing a trace-aware judge over the collected spans.
- [Quickstart](quickstart.md) — get an audit running first, then add tracing.
- [Installation](installation.md) — install the package plus the optional extras (`aiohttp` for the HTTP receiver, `grpc`/`opentelemetry-proto` for the gRPC receiver).

### See Also

*   [Document marks](context-analysis.md)
*   [Anatomy of a judge config](custom-judges.md)
*   [1. Run a single-model audit](quickstart.md)
*   [Scenario shape](scenarios.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.


**Container capabilities:** `SpanStore` supports `len()`.
