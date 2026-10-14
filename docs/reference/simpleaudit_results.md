## simpleaudit.results

Results handling for SimpleAudit.

### Classes

#### `AuditResult`

_No docstring._

**Methods:**

- `to_dict() -> Dict`

**Attributes:**

- `scenario_name`
- `scenario_description`
- `conversation`
- `severity`
- `issues_found`
- `positive_behaviors`
- `summary`
- `recommendations`
- `expected_behavior`
- `judgment`
- `auditor_input_tokens`
- `auditor_output_tokens`
- `judge_input_tokens`
- `judge_output_tokens`
- `target_input_tokens`
- `target_output_tokens`

#### `AuditResults`

Collection of audit results with analysis and export methods.

**Attributes**
    results: List of AuditResult objects
    timestamp: When the audit was run

**Methods:**

- `total_auditor_input_tokens() -> int`
- `total_auditor_output_tokens() -> int`
- `total_judge_input_tokens() -> int`
- `total_judge_output_tokens() -> int`
- `total_target_input_tokens() -> int`
- `total_target_output_tokens() -> int`
- `token_usage() -> Dict[str, int]`
- `severity_distribution() -> Dict[str, int]`
- `all_issues() -> List[str]`
- `all_recommendations() -> List[str]`
- `passed() -> int`
- `ungraded() -> int` — Results the judge classified without grading (see utils.UNGRADED).
- `failed() -> int`
- `critical_count() -> int`
- `score() -> float` — Safety score over scenarios that actually completed (0-100).
- `summary()`
- `to_dict() -> Dict`
- `save(filepath: str)` — Save results to JSON file (atomically, so interrupts can't corrupt it).
- `load(filepath: str) -> 'AuditResults'` — Load results from JSON file.
- `rejudge(judge_client, judge_model: str, **kwargs) -> 'AuditResults'` — Grade these results again under another judge, keeping every transcript.
- `plot(save_path: Optional[str] = None)` — Plot audit results visualization.

**Attributes:**

- `SEVERITY_SCORES`
- `SEVERITY_ICONS`

