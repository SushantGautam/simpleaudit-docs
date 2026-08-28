## Custom Scenario Development

SimpleAudit scenarios define specific test cases for evaluating AI model behavior. Scenarios reside in `scenarios/` directory. Each scenario file exports a list of dictionaries. Schema version `2.0` current standard.

### Scenario Structure

Scenario dictionary contains specific keys.

| Key | Type | Description |
| :--- | :--- | :--- |
| `schema_version` | string | Version identifier. Use `"2.0"`. |
| `name` | string | Unique identifier. Format: `Pack - Category: Specific Test`. |
| `description` | string | Human-readable summary. Explains test purpose. |
| `test_prompt` | string | Input text sent to model under test. |
| `language` | string | ISO 639-1 code. e.g., `"no"`, `"en"`. |
| `expected_behavior` | list[string] | Criteria for correct response. List of strings. |
| `category` | string | High-level classification. e.g., `"Harmful Content"`, `"Identity & Relationships"`. |
| `subcategory` | string | Specific issue type. e.g., `"Misinformation"`, `"Abuse"`. |
| `severity` | string | Risk level. `"low"`, `"medium"`, `"high"`. |
| `source` | dict | Provenance data. |
| `metadata` | dict | Authoring info, tags, rationale. |

### Source Object

`source` object tracks data origin.

```python
'source': {
    'type': 'synthetic', # or 'real_case'
    'inspiration': 'Description of factual basis or dataset source',
    'origin': 'Specific dataset or document name (optional)',
    'original_language': 'ISO 639-1 code (optional, for real_case)',
    'real_response': 'Reference answer (only for real_case)',
}
```

*   **synthetic**: Scenario created manually. `inspiration` cites legal basis or documentation.
*   **real_case**: Scenario derived from actual system logs. `real_response` stores original system output. `origin` identifies source dataset. `original_language` specifies prompt language.

### Metadata Object

`metadata` object provides context for maintainers.

```python
'metadata': {
    'author': 'username',
    'date_created': 'YYYY-MM-DD',
    'rationale': 'Explanation of why this scenario exists',
    'tags': ['keyword1', 'keyword2'],
    'expected_outcome': 'answer' or 'refuse', # Optional, for refusal packs
}
```

*   **rationale**: Critical for complex scenarios. Explain specific failure mode tested.
*   **tags**: Searchable keywords. Use domain-specific terms.

### Expected Behavior Patterns

`expected_behavior` list defines success criteria. Use imperative verbs.

**Positive Requirements:**
*   "State that..."
*   "Explain that..."
*   "Mention..."
*   "Acknowledge..."
*   "Provide..."
*   "Suggest..."

**Negative Requirements (Constraints):**
*   "NOT state..."
*   "NOT fabricate..."
*   "Do not provide..."
*   "Avoid..."
*   "NOT be..."

**Nuance Indicators:**
*   "Can mention (not required)..."
*   "Must not overstate..."

Example from `nav_aap.py`:

```python
'expected_behavior': [
    "State that the main rule is up to 3 years",
    "Mention that extension of up to 2 additional years is possible in particular cases",
    "NOT state categorically '3 years' without mentioning the extension possibility",
    "Recommend checking the individual decision letter (vedtak) for personal duration",
]
```

### Creating New Scenario Packs

1.  **Create File**: Name file `scenarios/<pack_name>.py`.
2.  **Define List**: Create list variable `<PACK_NAME>_SCENARIOS`.
3.  **Populate Scenarios**: Add dictionary entries.
4.  **Verify Facts**: For factual scenarios, verify against primary sources. Record verification date in docstring.
5.  **Add Docstring**: Module docstring explains pack purpose, scope, and verification status.

Example module structure:

```python
"""
<Pack Name> scenario pack.

Tests <specific domain> rules.
Facts verified against <source> on <date>.
"""

MY_SCENARIOS = [
    {
        "schema_version": "2.0",
        "name": "Test Case Name",
        "description": "Tests specific behavior.",
        "test_prompt": "User input here.",
        "language": "en",
        "expected_behavior": [
            "Model should do X",
            "Model should NOT do Y",
        ],
        "category": "Harmful Content",
        "subcategory": "Misinformation",
        "severity": "high",
        "source": {
            "type": "synthetic",
            "inspiration": "Source document or legal reference",
        },
        "metadata": {
            "author": "dev_name",
            "date_created": "2026-01-01",
            "rationale": "Why this test matters.",
            "tags": ["tag1", "tag2"],
        },
    },
]
```

### Best Practices

**Factual Accuracy**
*   Cite specific legal sections or documentation pages in `source.inspiration`.
*   Mark time-sensitive facts (rates, deadlines) as "time-bounded".
*   Note if pack requires annual re-verification.

**Refusal Scenarios**
*   Use `expected_outcome: 'refuse'` in metadata.
*   `expected_behavior` must specify refusal criteria.
*   Example: "Decline to provide substantive answer", "Suggest contacting professional".

**Guidance Scenarios**
*   Use `expected_outcome: 'answer'` in metadata.
*   `expected_behavior` must specify helpful response criteria.
*   Example: "Provide age-appropriate advice", "Acknowledge emotional context".

**Language Specificity**
*   Set `language` correctly.
*   For non-English prompts, ensure `test_prompt` matches target language.
*   `expected_behavior` can be in English for judge models, but should reflect linguistic nuances if necessary.

**Severity Assignment**
*   **high**: Direct harm, legal risk, significant financial impact.
*   **medium**: Misinformation, procedural errors, moderate confusion.
*   **low**: Minor inaccuracies, style issues, low-impact advice.

### Integration

Scenario packs load automatically if placed in `scenarios/` directory. SimpleAudit discovers files matching `*.py` pattern. Ensure variable name ends with `_SCENARIOS`.

Example usage in test runner:

```python
from simpleaudit import load_scenarios

# Load specific pack
scenarios = load_scenarios("nav_aap")

# Run audit
results = run_audit(scenarios, model="test_model")
```

### Verification Checklist

Before committing new scenarios:

1.  [ ] Schema version is `2.0`.
2.  [ ] All required keys present.
3.  [ ] `test_prompt` is realistic and representative.
4.  [ ] `expected_behavior` is unambiguous.
5.  [ ] `source` cites verifiable facts.
6.  [ ] `metadata.rationale` explains test logic.
7.  [ ] File syntax valid (run `python -m py_compile`).
8.  [ ] Docstring updated with verification date.

Maintain consistency with existing packs (`nav_aap`, `helfo`, `lanekassen`, `hei_refusal`, `helpmed`). Mirror structure for ease of maintenance.

### See Also

*   [Test Scenarios](test-scenarios.md)
*   [Advanced Methodologies](advanced-methodologies.md)
*   [CLI Usage](cli-usage.md)
*   [Core Architecture](core-architecture.md)
