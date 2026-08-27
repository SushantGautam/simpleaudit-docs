## Judges

The `simpleaudit.judges` module provides built-in evaluation configurations for assessing AI model performance. Judges define the criteria, scoring scales, and output schemas used to evaluate model responses against specific quality dimensions such as safety, helpfulness, factuality, and abstention behavior.

### Overview

SimpleAudit supports two primary judging paradigms:

1.  **Severity-Based Judges**: Emit a categorical severity level (`critical`, `high`, `medium`, `low`, `pass`). Used for safety and harm detection.
2.  **Score-Based Judges**: Emit a numerical score (1–10) with optional sub-scores. The framework automatically derives a severity level from the score for reporting consistency (9–10: pass, 7–8: low, 5–6: medium, 3–4: high, 1–2: critical).

All judges are defined as Python dictionaries containing `probe_prompt`, `judge_prompt`, `output_schema`, and optional `response_schema` (for JSON mode).

### Available Judges

| Judge Name | Description | Output Type | Reference |
| :--- | :--- | :--- | :--- |
| `safety` | Constitutional AI safety evaluation across 5 dimensions. | Severity | Bai et al. (2022) |
| `abstention` | Evaluates correct refusal on unanswerable questions. | Score (1–10) | Kirichenko et al. (2025) |
| `helpfulness` | Response quality across relevance, accuracy, clarity, completeness. | Score (1–10) | Zheng et al. (2023) |
| `factuality` | Detects hallucinations and factual errors. | Score (1–10) | Liu et al. (2023) |
| `harm` | Categorizes harm across 6 types (violence, fraud, etc.). | Severity + Category | Liang et al. (2022) |
| `binary_abstention` | Binary classifier: did the model abstain? | Boolean | Custom |
| `judge_conviction` | Meta-judge to track verdict stability under pressure. | Verdict + Boolean | Custom |
| `helsedir_sexhealth_no` | Norwegian sexual health domain judge (generic). | Severity (mapped) | Helsedirektoratet |
| `helsedir_sexhealth_no_rag` | Norwegian sexual health domain judge (RAG framing). | Severity (mapped) | Helsedirektoratet |

### API Reference

#### `get_judge(name: str) -> Dict[str, Any]`

Retrieves a built-in judge configuration by name.

*   **Args**:
    *   `name` (str): The judge configuration key (e.g., `"safety"`, `"helpfulness"`).
*   **Returns**:
    *   `Dict[str, Any]`: A shallow copy of the judge configuration dictionary.
*   **Raises**:
    *   `ValueError`: If the name is not recognized.

#### `list_judge_configs() -> Dict[str, str]`

Lists all available judge configurations and their descriptions.

*   **Returns**:
    *   `Dict[str, str]`: Mapping of judge names to one-line descriptions.

### Usage Examples

#### Using a Named Judge

Pass the judge name directly to the `ModelAuditor` constructor:

```python
from simpleaudit import ModelAuditor

# Evaluate response helpfulness
auditor = ModelAuditor(
    model="gpt-4",
    judge="helpfulness"
)

# Evaluate safety
auditor = ModelAuditor(
    model="gpt-4",
    judge="safety"
)
```

#### Customizing Probe Prompts

You can override the default probe prompt while keeping the judge's evaluation logic:

```python
auditor = ModelAuditor(
    model="gpt-4",
    judge="factuality",
    probe_prompt="Ask specifically about historical dates and verify accuracy."
)
```

#### Fully Custom Judge

Provide a custom `judge_prompt` to define entirely new evaluation criteria. This takes precedence over any named judge:

```python
auditor = ModelAuditor(
    model="gpt-4",
    judge_prompt="You are a strict grammar checker. Evaluate the response for grammatical errors. Output JSON: {\"score\": int, \"errors\": list[str]}."
)
```

### Judge Configuration Structure

Each judge configuration is a dictionary with the following keys:

*   `name` (str): Human-readable name.
*   `description` (str): Brief summary of the judge's purpose.
*   `probe_prompt` (str): System prompt for the probe model to generate user queries.
*   `judge_prompt` (str): System prompt for the judge model to evaluate the conversation.
*   `output_schema` (dict): Human-readable description of expected output fields.
*   `response_schema` (dict, optional): Machine-readable JSON Schema used when `json_format=True`.
*   `source` (dict): Academic or domain reference information.
*   `metadata` (dict): Versioning and authorship info.

### Score-to-Severity Mapping

For score-based judges (`abstention`, `helpfulness`, `factuality`), the framework maps the 1–10 score to a severity level for unified reporting:

| Score Range | Severity |
| :--- | :--- |
| 9–10 | `pass` |
| 7–8 | `low` |
| 5–6 | `medium` |
| 3–4 | `high` |
| 1–2 | `critical` |

This mapping ensures that summaries, plots, and aggregate metrics remain consistent across both severity-based and score-based judges. The raw judgment dictionary is stored unchanged in the `AuditResult`.

### Specialized Judges

#### Binary Abstention

The `binary_abstention` judge is a minimal classifier that determines if the model abstained from answering. It outputs only `abstained` (bool) and `reasoning` (str). It is designed for post-processing against ground-truth labels in scenario packs like `hei_refusal`.

```python
auditor = ModelAuditor(
    model="gpt-4",
    judge="binary_abstention"
)
```

#### Judge Conviction

The `judge_conviction` judge is a meta-judge used in the `judge_the_judge` scenario pack. It extracts the current verdict from a candidate judge's response and tracks if the verdict changed under pressure. It is typically used via `WiggleRunner` rather than directly as a `ModelAuditor` judge.

```python
from simpleaudit.judges import get_judge

config = get_judge("judge_conviction")
# Use config["judge_prompt"] as the observer system prompt
```

### Domain-Specific Judges

The `helsedir_sexhealth_no` and `helsedir_sexhealth_no_rag` judges are tailored for evaluating Norwegian sexual health bots for young users. They use a 1–5 scale from the Norwegian Directorate of Health, which is silently mapped to the standard severity enum:

*   5 (Tilfredsstillende) → `pass`
*   4 (Mindre avvik) → `low`
*   3 (Merkbart avvik) → `medium`
*   2 (Betydelig avvik) → `high`
*   1 (Uakseptabelt avvik) → `critical`

The `_rag` variant specifically references "dokumentene" (documents) in its criteria, suitable for RAG-based bots with a fixed source corpus.

### See Also

*   [Scenarios](scenarios.md)
*   [Cross-Judge Validation](cross-judge-validation.md)
*   [Judge Validation](judge-validation.md)
*   [Results & Analysis](results-analysis.md)
