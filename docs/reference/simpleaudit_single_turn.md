## simpleaudit.single_turn

Single-turn auditing.

### Classes

#### `SingleTurnAuditor`

ModelAuditor that runs each scenario as one exchange, not a conversation.

Differences from the base class:

- the scenario's `test_prompt` is sent verbatim; nothing is generated
- exactly one target call, so `max_turns` is never read
- the auditor model is never called, so `auditor_*` token counts stay 0
- the scenario's `documents` ride to the target as text-only blocks while
  their marks and the derived set properties go to the judge alone

Everything else — client construction, judge config resolution, retries,
JSON schema handling, result shape — is inherited unchanged.

Correctness next to provenance. The groundedness judge is blind to the
scenario's `expected_behavior` by design, so its severity measures what
the answer relied on, not whether the answer was right. When a marked
scenario also carries `expected_behavior`, the runner makes a second
judge call with the `correctness_judge` config (`"checklist"` by
default) on the same single exchange and combines the two with
`combine_judgments`: the stricter severity wins, both halves are kept
under `provenance` and `correctness`. Pass `correctness_judge=None`
to get the provenance judgment alone.

**Methods:**

- `async run_async(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1, params: Optional[Dict[str, Any]] = None, target_params: Optional[Dict[str, Any]] = None, judge_params: Optional[Dict[str, Any]] = None, auditor_params: Optional[Dict[str, Any]] = None, on_turn: Optional[OnTurn] = None, evidence_spans: Optional[List[Dict[str, Any]]] = None, audit_run_id: Optional[str] = None, trace_correlation: Optional[Any] = None, evidence_resolver: Optional[Callable[[ScenarioExecution], Union[List[Dict[str, Any]], None]]] = None) -> AuditResults` — Run every scenario single-turn and collect the results.

### Functions

#### `combine_judgments(provenance: Dict[str, Any], correctness: Dict[str, Any]) -> Dict[str, Any]`

One judgment from a provenance half and a correctness half.

The groundedness judge says what the answer relied on; the checklist judge
says whether the answer met the scenario's expectations. Neither alone is
the verdict: a wrong answer in the model's own words has clean provenance,
and a right answer can still have leaned on a superseded chunk. The
combined severity is therefore the stricter of the two. An off-ladder half
(an ERROR from a parse failure, or an ungraded result with no ground truth
to score) makes the verdict *incomplete*: it is ERROR, or ungraded when no
half errored — never a pass. A passing result must rest on two halves that
were both actually graded.

The five default fields come from the correctness half, which is the one
that quotes the transcript, with each fired provenance finding appended to
`issues_found`. Both halves are kept whole under `provenance` and
`correctness` so nothing is lost by combining.

### Constants

- `PROVENANCE_FINDINGS`

