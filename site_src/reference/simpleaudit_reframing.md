## simpleaudit.reframing

Reframing checks: rephrase a prompt and verify the model's answer stays consistent.

Re-grade stored transcripts under paraphrased judge prompts.

The reproducibility leg of an audit resamples: the same scenario is run
``n_repetitions`` times and the spread of the verdicts is reported. Resampling
varies the conversation as well as the grading, so a verdict that moves may be
telling you about the target, the auditor, or the judge.

This module holds the transcript fixed and varies only the wording of the judge
prompt. A verdict that survives resampling but flips between two semantically
equivalent rubrics is measuring the prompt rather than the target — an
apparatus artifact, not a finding.

Nothing here calls the target model. The transcripts are already stored, so a
check costs judge tokens only.

Usage::

    from any_llm import AnyLLM
    from simpleaudit.judges import get_judge
    from simpleaudit.reframing import (
        load_stored_records, reframing_check, PromptVariant,
    )

    base = get_judge("safety")["judge_prompt"]
    results = reframing_check(
        judge_client=AnyLLM.create("anthropic"),
        judge_model="claude-opus-4-7",
        records=load_stored_records("examples/nav_aap/nav_aap_sonnet_4_6.json"),
        variants=[
            PromptVariant("baseline", base),
            PromptVariant("reordered", reordered_rubric_text),
        ],
    )
    for entry in results.shifts():
        if entry["shifted"]:
            print(entry["scenario"], entry["modals"], entry["direction"])

### Classes

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

#### `PromptVariant`

One wording of the judge prompt.

Variants are supplied explicitly rather than generated. A tool built to
detect prompt-induced verdict movement cannot introduce a model-generated
prompt of its own: the paraphrases would then be an uncontrolled axis
inside the instrument measuring that axis.

**Attributes:**

- `label`
- `judge_prompt`
- `response_schema`

#### `StoredRecord`

A transcript to re-grade, plus the scenario context the judge needs.

**Attributes:**

- `scenario_name`
- `scenario_description`
- `conversation`
- `expected_behavior`

#### `ReframingResults`

Verdicts for every (scenario, variant) pair, and the shifts between them.

Attributes:
----------
variant_labels : list of str
    Variant labels in the order they were run.
per_scenario : dict
    ``{scenario_name: {variant_label: severity}}``.
judgments : dict
    ``{scenario_name: {variant_label: raw_judgment_dict}}`` — the full
    judge output, kept so a shift can be read back to its reasoning.
input_tokens, output_tokens : int
    Judge tokens spent. No target tokens are spent on this path.

**Methods:**

- `shifts() -> List[Dict[str, Any]]` — Per-scenario verdicts across variants, with shift detection.
- `invariant_rate() -> float` — Fraction of scenarios whose verdict is identical across all variants.
- `to_dict() -> Dict[str, Any]`

**Attributes:**

- `variant_labels`
- `per_scenario`
- `judgments`
- `input_tokens`
- `output_tokens`

### Functions

#### `normalize_severity(severity: Any) -> str`

Map a judge-emitted severity onto the framework's canonical ladder.

Case and surrounding whitespace are normalized, and known aliases are
folded in ("none" -> "pass", "error" -> "ERROR"). Values outside the
ladder are returned stripped-but-unchanged so custom judge vocabularies
survive; non-string values are coerced to str so downstream .upper()
calls cannot crash. A literal None means the judge omitted severity
entirely — that maps to "medium", NOT to the "none" alias, which is
reserved for judges that answer the string "none" meaning "no harm".

#### `severity_direction(severity_a: Any, severity_b: Any) -> Any`

Position of *severity_b* relative to *severity_a* on ``SEVERITY_ORDER``.

Returns ``idx_b - idx_a``; positive means *severity_b* is the stricter
verdict. Returns None when either value is off the canonical ladder
(e.g. "ERROR"), so callers can tell "no direction" apart from "no shift".

#### `load_stored_records(source: Union[str, Path, Dict[str, Any]]) -> List[StoredRecord]`

Read transcripts out of a saved audit result.

Accepts a path to a file written by ``AuditResults.save()`` or the already
parsed payload. Entries whose ``conversation`` is missing or empty are
skipped: there is nothing to re-grade, and carrying them through would
produce verdicts on an empty transcript that look like real ones.

Parameters:
----------
source : str, Path, or dict
    Saved result file, or its parsed contents.

Returns:
-------
list of StoredRecord

#### `async reframing_check_async(judge_client: Any, judge_model: str, records: Sequence[StoredRecord], variants: Sequence[PromptVariant], json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5) -> ReframingResults`

Re-grade every stored transcript once per prompt variant.

Parameters:
----------
judge_client : Any
    An AnyLLM client for the judge. This is the only client used.
judge_model : str
    Judge model identifier.
records : sequence of StoredRecord
    Transcripts to re-grade, e.g. from ``load_stored_records()``.
variants : sequence of PromptVariant
    At least two wordings of the judge prompt. One variant would produce
    no comparison.
json_format : bool, default True
    Whether the judge is asked for schema-constrained JSON.
max_retries, retry_backoff
    Forwarded to the judge call.

Returns:
-------
ReframingResults

#### `reframing_check(judge_client: Any, judge_model: str, records: Sequence[StoredRecord], variants: Sequence[PromptVariant], json_format: bool = True, max_retries: int = 0, retry_backoff: float = 0.5) -> ReframingResults`

Synchronous wrapper around :func:`reframing_check_async`.

Cannot be called from an active event loop; await the async form there.
Mirrors ``CrossJudgeExperiment.run()``.

