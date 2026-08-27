## simpleaudit.results

Result containers: per-scenario results and aggregated audit results with summaries.

Results handling for SimpleAudit.

Contains dataclasses for storing audit results and methods for
analysis, visualization, and export.

### Classes

#### `AuditResult`

A single audit result for one scenario.

Holds the scenario metadata, the full conversation with the target model,
the judge's verdict (severity, issues, positive behaviors, summary,
recommendations), the raw judge judgment dict, and token usage counts for
the auditor, judge, and target models.

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

Attributes:
    results: List of AuditResult objects
    timestamp: When the audit was run

**Methods:**

- `summary()`
- `to_dict() -> Dict`
- `save(filepath: str)` — Save results to JSON file (atomically, so interrupts can't corrupt it).
- `load(filepath: str) -> AuditResults` — Load results from JSON file.
- `plot(save_path: Optional[str] = None)` — Plot audit results visualization.

**Attributes:**

- `SEVERITY_SCORES`
- `SEVERITY_ICONS`
- `results`
- `timestamp`
- `total_auditor_input_tokens`
- `total_auditor_output_tokens`
- `total_judge_input_tokens`
- `total_judge_output_tokens`
- `total_target_input_tokens`
- `total_target_output_tokens`
- `token_usage`
- `severity_distribution`
- `all_issues`
- `all_recommendations`
- `passed`
- `failed`
- `critical_count`
- `score` — Safety score over scenarios that actually completed (0-100).

