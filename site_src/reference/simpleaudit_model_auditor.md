## simpleaudit.model_auditor

The core audit engine: drives target model, auditor, and judge across scenarios.

ModelAuditor for direct API-based model auditing.

This module provides the ModelAuditor class that audits LLM models directly
via their APIs (OpenAI, Anthropic Claude, Grok, Ollama, vLLM, and any OpenAI-compatible endpoint) 
rather than through an HTTP endpoint.

Key features:
- Direct API auditing without external server
- Optional system prompt configuration
- Separate provider selection for judge vs target model

### Classes

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

#### `ModelAuditor`

Audits a target LLM directly via its API against a set of scenarios.

Each scenario is sent to the target model (optionally over multiple turns),
and the resulting conversation is scored by a separate LLM judge that
returns a severity verdict, issues, and recommendations. The judge and
target may use different providers, and an optional auditor model can
probe the target between turns.

**Methods:**

- `strip_thinking(text: str) -> str`
- `get_scenarios(pack_name: str) -> List[Dict]`
- `parse_json_response(response: str, default_severity: str = 'ERROR') -> Dict[str, Any]` — Parse JSON from LLM response. Delegates to utils.parse_json_response.
- `async run_scenario(name: str, description: str, expected_behavior: Optional[List[str]] = None, test_prompt: Optional[str] = None, file_uri: Optional[Union[str, List[str]]] = None, max_turns: Optional[int] = None, language: str = 'English', pbar_audit: Optional[tqdm] = None, pbar_judge: Optional[tqdm] = None, max_workers: Optional[int] = None) -> AuditResult`
- `async run_async(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> AuditResults` — Run the audit asynchronously across all scenarios.
- `run(scenarios: Union[str, List[Dict]], max_turns: Optional[int] = None, language: str = 'English', max_workers: int = 1) -> AuditResults` — Run the audit synchronously by driving :meth:`run_async` with ``asyncio.run``.

**Attributes:**

- `max_turns`
- `verbose`
- `show_progress`
- `system_prompt`
- `json_format`
- `max_retries`
- `retry_backoff`
- `judge_fields`
- `probe_prompt`
- `judge_prompt`
- `judge_response_schema`
- `target_model`
- `target_client`
- `judge_model`
- `judge_client`
- `auditor_model`
- `auditor_client`

### Functions

#### `get_judge(name: str) -> Dict[str, Any]`

Get a built-in judge configuration by name.

Args:
    name: Judge config name (e.g. "safety", "helpfulness")

Returns:
    Judge config dict with probe_prompt, judge_prompt, output_schema, source

Raises:
    ValueError: If name is not recognised

#### `image_data_uri(file_uri: str) -> str`

Read an image and inline it as a base64 data URI.

fsspec resolves local paths, http(s), s3, gs and friends through one call,
so scenario authors can point at wherever the image actually lives.

Cached because the first user message is re-sent on every turn of a
multi-turn audit — without this the same file is re-read and re-encoded
once per turn per scenario. Entries are keyed on the URI alone, so
run_async clears the cache to pick up files changed between runs.

#### `image_content_block(file_uri: str) -> Dict[str, Any]`

Build an OpenAI-style image content block from a URI.

#### `normalize_severity(severity: Any) -> str`

Map a judge-emitted severity onto the framework's canonical ladder.

Case and surrounding whitespace are normalized, and known aliases are
folded in ("none" -> "pass", "error" -> "ERROR"). Values outside the
ladder are returned stripped-but-unchanged so custom judge vocabularies
survive; non-string values are coerced to str so downstream .upper()
calls cannot crash. A literal None means the judge omitted severity
entirely — that maps to "medium", NOT to the "none" alias, which is
reserved for judges that answer the string "none" meaning "no harm".

#### `severity_from_score(score: Any, max_score: float = 10.0) -> Any`

Map a numeric judge score onto the canonical severity ladder.

Score-based judges (helpfulness, factuality, abstention) emit a 1-10
score and no severity; without a mapping every such scenario shows up
as the "medium" default in summaries and plots. Bands (for a 1-10
scale):  9-10 -> pass, 7-8 -> low, 5-6 -> medium, 3-4 -> high,
1-2 -> critical. Returns None when score is not numeric so the caller
keeps its default behaviour.

#### `build_judge_schema(fields: Optional[List[str]] = None) -> Dict[str, Any]`

Build a JSON response schema for the given judge output fields.

Args:
    fields: List of field names to include. Defaults to all standard fields.

Returns:
    A JSON schema dict suitable for ``response_format``.

#### `build_judge_json_snippet(fields: Optional[List[str]] = None) -> str`

Build the JSON structure snippet for the judge prompt.

Args:
    fields: List of field names to include. Defaults to all standard fields.

Returns:
    A multi-line JSON template string for the prompt.

### Constants

#### `SCENARIO_PACKS`
#### `DEFAULT_JUDGE_RESPONSE_SCHEMA`
#### `DEFAULT_JUDGE_FIELDS`
