## simpleaudit.experiment

experiment event

### Classes

#### `ExperimentEvent`

Typed event yielded by `AuditExperiment.run_streamed`.

**Attributes**
    type: One of `"rep_started"`, `"rep_done"`, `"model_done"`,
        `"cancelled"`.
    model: Model label the event belongs to.
    rep_index: Zero-based rep index (`-1` for non-rep events).
    total_reps: Total reps planned for this model.
    scenario_name: Set for scenario-level events.
    result: The `AuditResults` for the full rep (set on `rep_done`).
    partial: Partial `RepeatedExperimentResults` (set on `model_done`).

**Attributes:**

- `type`
- `model`
- `rep_index`
- `total_reps`
- `scenario_name`
- `result`
- `partial`

#### `AuditExperiment`

_No docstring._

**Methods:**

- `async run_scenario_reps(model_index: int, scenario: Dict[str, Any], max_turns: Optional[int] = None, language: str = 'English', on_turn: Optional[OnTurn] = None, audit_run_id: Optional[str] = None, trace_correlation: Optional[Any] = None) -> List[AuditResult]` — Run a single scenario N times for one model.
- `async run_streamed(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> AsyncIterator[ExperimentEvent]` — Execute the experiment, yielding typed events as progress is made.
- `async run_async(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> RepeatedExperimentResults` — Run the full experiment. Thin wrapper over `run_streamed`.
- `run(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> RepeatedExperimentResults`

