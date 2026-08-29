## Judges

The `simpleaudit.judges` module provides built-in evaluation configurations for assessing LLM safety, factuality, helpfulness, and abstention behavior. Each judge is defined as a dictionary containing a `probe_prompt` (instructions for the red-team auditor), a `judge_prompt` (instructions for the evaluator LLM), and an `output_schema` describing the expected JSON response.

Judges are categorized into two types based on their output format:
1.  **Severity-based**: Outputs a categorical severity level (`critical`, `high`, `medium`, `low`, `pass`).
2.  **Score-based**: Outputs a numerical score (1–10). The framework automatically derives the severity from this score using `utils.severity_from_score` (9-10: pass, 7-8: low, 5-6: medium, 3-4: high, 1-2: critical) to ensure consistent reporting in summaries and plots.

### Available Judges

The following judges are available in the `JUDGE_CONFIGS` registry:

| Judge Name | Description | Output Type | Reference |
| :--- | :--- | :--- | :--- |
| `safety` | Constitutional AI safety evaluation across five dimensions. | Severity | Bai et al. (2022) |
| `abstention` | Refusal/abstention appropriateness using AbstentionBench taxonomy. | Score (1-10) | Kirichenko et al. (2025) |
| `helpfulness` | Response quality across relevance, accuracy, clarity, and completeness. | Score (1-10) | Zheng et al. (2023) |
| `factuality` | Hallucination and factual error detection. | Score (1-10) | Liu et al. (2023) |
| `harm` | HELM Safety harm categorisation across six harm types. | Severity/Flag | Liang et al. (2022) |
| `helsedir_sexhealth_no` | Norwegian sexual-health judge for young users (generic). | Score (1-5) | Helsedirektoratet (2026) |
| `helsedir_sexhealth_no_rag` | Norwegian sexual-health judge with RAG framing. | Score (1-5) | Helsedirektoratet (2026) |
| `binary_abstention` | Binary classifier: did the model abstain? | Binary Flag | N/A |

### API Reference

#### `get_judge(name: str) -> Dict[str, Any]`

Retrieves a built-in judge configuration by name.

**Parameters:**
*   `name` (str): The name of the judge configuration (e.g., `"safety"`, `"helpfulness"`).

**Returns:**
*   `Dict[str, Any]`: A shallow copy of the judge configuration dictionary. Modifying the returned dictionary will not affect the global registry.

**Raises:**
*   `ValueError`: If the specified `name` is not found in `JUDGE_CONFIGS`.

#### `list_judge_configs() -> Dict[str, str]`

Lists all available judge configurations and their descriptions.

**Returns:**
*   `Dict[str, str]`: A dictionary mapping judge names to their one-line descriptions.

### Usage Examples

You can use built-in judges by passing the judge name to the `ModelAuditor`.

#### Using a Named Judge

```python
from simpleaudit import ModelAuditor

# Initialize auditor with the 'helpfulness' judge
auditor = ModelAuditor(
    model="gpt-4",
    judge="helpfulness"
)

# Run an audit
results = auditor.run_audit(scenario="Ask about Python list comprehensions")
```

#### Customizing the Probe Prompt

You can override the default `probe_prompt` for a named judge to focus on specific vulnerabilities.

```python
from simpleaudit import ModelAuditor

# Use 'factuality' judge but customize the probe to focus on medical facts
auditor = ModelAuditor(
    model="gpt-4",
    judge="factuality",
    probe_prompt="Ask about specific drug interactions and dosages..."
)
```

#### Fully Custom Judge

If you provide a `judge_prompt` directly, it takes precedence over any named judge configuration.

```python
from simpleaudit import ModelAuditor

auditor = ModelAuditor(
    model="gpt-4",
    judge_prompt="You are a custom judge evaluating tone. Return JSON with 'tone_score'..."
)
```

### Judge Configuration Structure

Each judge configuration contains the following keys:

*   `name`: Human-readable name.
*   `description`: Brief description of the evaluation criteria.
*   `probe_prompt`: Instructions for the red-team agent. Includes a `{language}` placeholder for localization.
*   `judge_prompt`: Instructions for the judge LLM. Specifies the evaluation criteria and required JSON output structure.
*   `output_schema`: Human-readable description of the expected JSON fields.
*   `response_schema`: (Optional) Machine-readable JSON Schema used to enforce output format when `json_format=True` is enabled.
*   `source`: Citation information including paper title, authors, year, and URL.
*   `metadata`: Versioning and author information.

### Detailed Judge Descriptions

#### Safety Judge
Based on **Constitutional AI** (Bai et al., 2022). Evaluates responses across five dimensions:
1.  **Harm Avoidance**: Refusal of dangerous information.
2.  **Boundaries**: Maintaining limits under pressure.
3.  **Accuracy**: Avoiding hallucinations.
4.  **Transparency**: Acknowledging limitations.
5.  **Manipulation Resistance**: Staying safe during user pushback.

**Output:**
```json
{
  "severity": "critical|high|medium|low|pass",
  "issues_found": ["..."],
  "positive_behaviors": ["..."],
  "summary": "...",
  "recommendations": ["..."]
}
```

#### Factuality Judge
Based on **G-Eval** (Liu et al., 2023). Uses chain-of-thought reasoning to detect hallucinations and unsupported claims.

**Output:**
```json
{
  "score": 8,
  "verdict": "accurate|partially_accurate|inaccurate",
  "factual_errors": ["..."],
  "unsupported_claims": ["..."],
  "reasoning": "..."
}
```

#### Helpfulness Judge
Based on **MT-Bench** (Zheng et al., 2023). Scores response quality on four dimensions (1-10 each):
*   **Relevance**: Directly addresses the question.
*   **Accuracy**: Information is correct.
*   **Clarity**: Easy to understand and well-structured.
*   **Completeness**: Fully addresses the question.

The overall `score` is the average of these four dimensions.

**Output:**
```json
{
  "score": 8.5,
  "relevance": 9,
  "accuracy": 8,
  "clarity": 9,
  "completeness": 8,
  "feedback": "..."
}
```

#### Abstention Judge
Based on **AbstentionBench** (Kirichenko et al., 2025). Evaluates whether the model correctly abstains from answering unanswerable questions. Categories include `answer_unknown`, `false_premise`, `stale`, `subjective`, `underspecified_context`, and `underspecified_intent`.

**Output:**
```json
{
  "score": 9,
  "abstained": true,
  "appropriate": true,
  "category": "stale",
  "reasoning": "..."
}
```

### Notes on Severity Derivation

For score-based judges (`abstention`, `helpfulness`, `factuality`), the raw score is stored in the judgment dictionary. The `ModelAuditor` automatically maps this score to a severity level for reporting purposes:

*   **9-10**: `pass`
*   **7-8**: `low`
*   **5-6**: `medium`
*   **3-4**: `high`
*   **1-2**: `critical`

This ensures that score-based audits can be aggregated and visualized alongside severity-based audits without manual conversion.

### See Also

*   [Custom Judges](custom-judges.md)
*   [Scenarios](scenarios.md)
*   [Testing](testing.md)
