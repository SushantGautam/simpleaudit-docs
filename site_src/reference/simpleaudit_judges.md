## simpleaudit.judges

Built-in judge configurations: safety, harm, factuality, helpfulness, abstention, and more.

Built-in judge configurations for SimpleAudit.

Available judges:
- safety:      Constitutional AI safety evaluation (Bai et al., 2022)
                Severity: critical | high | medium | low | pass
- abstention:  Refusal/abstention appropriateness, AbstentionBench taxonomy
                (Kirichenko et al., 2025). Score 1–10, with abstained/appropriate flags
- helpfulness: Response quality across four MT-Bench dimensions (Zheng et al., 2023)
                Score 1–10 with relevance, accuracy, clarity, completeness sub-scores
- factuality:  Hallucination and factual error detection (Liu et al., 2023)
                Score 1–10 with verdict and error lists

Score-based judges (abstention, helpfulness, factuality) emit a 1-10 score and
no severity; the framework derives AuditResult.severity from the score
(9-10 pass, 7-8 low, 5-6 medium, 3-4 high, 1-2 critical — see
utils.severity_from_score) so summaries and plots stay meaningful. The raw
judgment dict is stored unchanged.
- harm:        HELM Safety harm categorisation (Liang et al., 2022)
                harmful flag, category, severity across six harm types
- helsedir_sexhealth_no:
                Norwegian sexual-health judge for young users — generic variant.
                Six criteria from Helsedirektoratet domain review (2026), neutral 1–5
                scale mapped silently to {critical, high, medium, low, pass}.
- helsedir_sexhealth_no_rag:
                Same as above, RAG framing — criteria reference «dokumentene»
                verbatim per the reviewer's wording. Use for bots with a fixed
                source corpus.
- binary_abstention:
                Language-agnostic binary classifier: did the model abstain
                (decline to deliver the substantive content requested),
                yes or no? Emits {abstained, reasoning} only — no severity.
                Declares its own response_schema; works with json_format=True.

Usage:
    from simpleaudit import ModelAuditor

    # Use a named judge config
    auditor = ModelAuditor(..., judge="helpfulness")

    # Named judge + custom probe (probe_prompt overrides the config's probe_prompt)
    auditor = ModelAuditor(..., judge="factuality", probe_prompt="Ask about X...")

    # Fully custom (judge_prompt takes precedence over any named judge)
    auditor = ModelAuditor(..., judge_prompt="You are a custom judge...")

### Functions

#### `get_judge(name: str) -> Dict[str, Any]`

Get a built-in judge configuration by name.

Args:
    name: Judge config name (e.g. "safety", "helpfulness")

Returns:
    Judge config dict with probe_prompt, judge_prompt, output_schema, source

Raises:
    ValueError: If name is not recognised

#### `list_judge_configs() -> Dict[str, str]`

List available judge configs and their descriptions.

Returns:
    Dict mapping config names to one-line descriptions

### Constants

#### `JUDGE_CONFIGS`
