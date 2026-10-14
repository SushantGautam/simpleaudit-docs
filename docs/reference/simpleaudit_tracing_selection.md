## simpleaudit.tracing.selection

Span selection — choose which spans to show the judge.

### Classes

#### `SelectionResult`

_No docstring._

**Methods:**

- `elided_count() -> int`

**Attributes:**

- `selected`
- `elided`
- `budget_used`
- `budget`
- `selector_version`

### Functions

#### `select_spans(spans: Sequence[Dict[str, Any]], evidence_kinds: Sequence[str] = DEFAULT_EVIDENCE_KINDS, noise_kinds: Sequence[str] = DEFAULT_NOISE_KINDS, token_budget: Optional[int] = None, prefer_failing_spans: bool = False, selector_version: str = 'v1') -> SelectionResult`

Select evidence-relevant spans within an optional size budget.

**Parameters**
spans:
    Normalized spans (see `simpleaudit.tracing.store.normalize_span`).
evidence_kinds:
    Kinds to prefer. Spans of these kinds are kept (subject to budget).
noise_kinds:
    Kinds to drop unless nothing else is selected.
token_budget:
    Max total serialized size (chars) of selected spans. `None` = no cap.
prefer_failing_spans:
    Prioritize spans with error/failure status over success spans.
selector_version:
    Version identifier for the selection policy (recorded in provenance).

#### `evidence_spans_for_turn(correlation: Any, store: Any, turn_id: str, evidence_kinds: Sequence[str] = DEFAULT_EVIDENCE_KINDS, noise_kinds: Sequence[str] = DEFAULT_NOISE_KINDS, token_budget: Optional[int] = None, prefer_failing_spans: bool = False, selector_version: str = 'v1') -> List[Dict[str, Any]]`

Build judge-ready `evidence_spans` for one audit turn.

Pulls the turn's spans from `store` via `correlation`, selects the
evidence-relevant ones, and returns them (with provenance) ready to pass
to `run_async(..., evidence_spans=...)`.

This is the glue between trace ingestion (OTLP → SpanStore) and
judge-over-spans: the engine records `turn_id → trace_id` during the
run, spans arrive in the store, and this selects what the judge sees.

#### `summarize_for_judge(result: SelectionResult, max_chars_per_span: int = 2000) -> str`

Render selected spans into a compact text block for the judge prompt.

### Constants

- `DEFAULT_EVIDENCE_KINDS`

- `DEFAULT_NOISE_KINDS`

