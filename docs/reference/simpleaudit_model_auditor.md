## simpleaudit.model_auditor

ModelAuditor for direct API-based model auditing.

### Classes

#### `ScenarioExecution`

Captures the state of a scenario after target execution but before judgment.

This dataclass represents one scenario execution through the target conversation phase.
It holds all state necessary for the evidence resolver to select trace evidence, which
can then be passed to the judge.

**Attributes:**

- `scenario_name`
- `scenario_description`
- `conversation`
- `trace_correlation`
- `audit_run_id`
- `error`
- `scenario_meta`
- `expected_behavior`
- `target_input_tokens`
- `target_output_tokens`
- `auditor_input_tokens`
- `auditor_output_tokens`

#### `ModelAuditor`

_No docstring._

**Methods:**

- `target() -> Any` — The `Target` the auditor sends to.
- `set_target(target: Any) -> None` — Override the target with an arbitrary `Target` implementation.
- `strip_thinking(text: str) -> str`
- `get_scenarios(pack_name: str) -> List[Dict]`
- `parse_json_response(response: str, default_severity: str = 'ERROR') -> Dict[str, Any]` — Parse JSON from LLM response. Delegates to utils.parse_json_response.
- `async run_scenario(name: str, description: str, expected_behavior: Optional[List[str]] = None, test_prompt: Optional[str] = None, file_uri: Optional[Union[str, List[str]]] = None, documents: Optional[List[Union[str, Dict[str, Any]]]] = None, decision: Optional[Dict[str, Any]] = None, judge_notes: Optional[List[str]] = None, max_turns: Optional[int] = None, language: str = 'English', pbar_audit: Optional[tqdm] = None, pbar_judge: Optional[tqdm] = None, max_workers: Optional[int] = None, scenario_meta: Optional[Dict[str, Any]] = None, params: Optional[Dict[str, Any]] = None, target_params: Optional[Dict[str, Any]] = None, judge_params: Optional[Dict[str, Any]] = None, auditor_params: Optional[Dict[str, Any]] = None, on_turn: Optional[OnTurn] = None, evidence_spans: Optional[List[Dict[str, Any]]] = None, audit_run_id: Optional[str] = None, trace_correlation: Optional[Any] = None, evidence_resolver: Optional[Callable[[ScenarioExecution], Union[List[Dict[str, Any]], None]]] = None) -> AuditResult`
- `async run_async(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1, params: Optional[Dict[str, Any]] = None, target_params: Optional[Dict[str, Any]] = None, judge_params: Optional[Dict[str, Any]] = None, auditor_params: Optional[Dict[str, Any]] = None, on_turn: Optional[OnTurn] = None, evidence_spans: Optional[List[Dict[str, Any]]] = None, audit_run_id: Optional[str] = None, trace_correlation: Optional[Any] = None, evidence_resolver: Optional[Callable[[ScenarioExecution], Union[List[Dict[str, Any]], None]]] = None) -> AuditResults`
- `run(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1, params: Optional[Dict[str, Any]] = None, target_params: Optional[Dict[str, Any]] = None, judge_params: Optional[Dict[str, Any]] = None, auditor_params: Optional[Dict[str, Any]] = None, on_turn: Optional[OnTurn] = None, evidence_spans: Optional[List[Dict[str, Any]]] = None, audit_run_id: Optional[str] = None, trace_correlation: Optional[Any] = None, evidence_resolver: Optional[Callable[[ScenarioExecution], Union[List[Dict[str, Any]], None]]] = None) -> AuditResults`

### Functions

#### `build_judge_schema(fields: Optional[List[str]] = None) -> Dict[str, Any]`

Build a JSON response schema for the given judge output fields.

**Args**
    fields: List of field names to include. Defaults to all standard fields.

**Returns**
    A JSON schema dict suitable for `response_format`.

#### `build_judge_json_snippet(fields: Optional[List[str]] = None) -> str`

Build the JSON structure snippet for the judge prompt.

**Args**
    fields: List of field names to include. Defaults to all standard fields.

**Returns**
    A multi-line JSON template string for the prompt.

### Constants

- `OnTurn`

- `DEFAULT_USER_AGENT`

- `DEFAULT_JUDGE_RESPONSE_SCHEMA`

- `DEFAULT_JUDGE_FIELDS`

