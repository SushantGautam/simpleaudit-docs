## simpleaudit.targets.base

Core Target protocol and shared data types.

### Classes

#### `TargetResponse`

Normalized result of sending a message to a target.

`input_tokens` / `output_tokens` are `None` when the target does not
report usage (e.g. a black-box HTTP app). Callers must treat `None` as
"unknown", never as zero, so audit records do not fabricate token counts.

**Attributes:**

- `content`
- `raw`
- `input_tokens`
- `output_tokens`
- `decision`

#### `TargetContext`

Per-turn context passed to a target.

Carries the audit correlation identifiers and (optionally) the W3C trace
context so a target can propagate it to the system under test. All fields
are optional so existing call sites that do not care about tracing can
pass a bare context or `None`.

**Attributes:**

- `audit_run_id`
- `scenario_run_id`
- `turn_id`
- `trace_headers`
- `extra`

#### `Target`

Anything the auditor can send a message to and get a response from.

Implementations:
    - `ModelTarget`    (LLM endpoint via AnyLLM)
    - `HTTPAppTarget`  (external application over HTTP)
    - `CallableTarget` (in-process Python callable)
    - `DecisionTarget` (decision model over a System One endpoint)

The signature mirrors the historical `ModelAuditor._call_async` inputs so
that `ModelTarget` can delegate to the existing client path with no
behavior change. New targets may ignore the model-specific keyword
arguments (`response_format`, `file_uri`, `documents`) when they do
not apply.

**Methods:**

- `async send(system: Optional[str] = None, user: str, history: Optional[List[Dict[str, Any]]] = None, file_uri: Optional[Union[str, List[str]]] = None, documents: Optional[List[Union[str, Dict[str, Any]]]] = None, response_format: Optional[Dict[str, Any]] = None, params: Optional[Dict[str, Any]] = None, context: Optional[TargetContext] = None) -> TargetResponse` — Send one message and return the normalized response.

