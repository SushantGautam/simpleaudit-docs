## Scenarios

The `simpleaudit.scenarios` module provides built-in test case packs for auditing AI systems. These packs define specific behaviors, safety boundaries, and domain-specific knowledge requirements. Each scenario is a dictionary containing metadata, test prompts, and expected behaviors, designed to be consumed by the `ModelAuditor` or `WiggleRunner`.

### Core Functions

The module exposes three primary functions for interacting with scenario packs:

| Function | Description |
| :--- | :--- |
| `get_scenarios(pack_name)` | Retrieves a list of scenario dictionaries for a specific pack. Returns a shallow copy to prevent mutation of the shared registry. |
| `list_scenario_packs()` | Returns a dictionary mapping pack names to their scenario counts. |
| `duplicate_scenario_names(scenarios)` | Validates a custom scenario list by identifying duplicate names. Duplicate names corrupt stability statistics in `RepeatedExperimentResults`. |

### Available Packs

Scenario packs are grouped by domain and testing focus. Use `list_scenario_packs()` to verify available packs in your environment.

| Pack Name | Description |
| :--- | :--- |
| `safety` | General AI safety: hallucination, harmful instructions, manipulation, privacy. |
| `rag` | Retrieval-Augmented Generation: source attribution, context confusion, citation fabrication. |
| `health` | Healthcare safety: diagnosis boundaries, prescription refusal, emergency response. |
| `system_prompt` | System prompt integrity: override attempts, leak prevention, context pollution. |
| `helpmed` | Help and medical guidance scenarios. |
| `ung` | Norwegian youth mental health and support scenarios. |
| `bullshitbench_v1` | 55 scenarios testing resistance to incoherent business/management premises. |
| `bullshitbench_v2` | 100 scenarios across software, finance, legal, medical, and physics. |
| `bullshitbench` | Combined v1 and v2 (155 scenarios). |
| `health_bullshit` | 15 health-specific broken premise scenarios (e.g., antibiotics for viruses). |
| `epistemic_safety` | Combined `bullshitbench` and `health_bullshit` (170 scenarios). |
| `hei_refusal` | 47 Norwegian youth Q&A edge cases testing refusal vs. guidance boundaries. |
| `nav_aap` | 15 Norwegian welfare (NAV) scenarios. |
| `skatteetaten` | Norwegian Tax Administration scenarios (in development). |
| `helfo` | 8 Norwegian health-economics (Helfo) scenarios. |
| `lanekassen` | 8 Norwegian student-finance (Lånekassen) scenarios. |
| `vision_integrity` | 8 chart-reading integrity scenarios. **Requires** vision-capable target, judge, and auditor. Excluded from `all`. |
| `judge_the_judge` | 8 judge qualification scenarios. **Requires** `WiggleRunner` with three model roles. Excluded from `all`. |
| `all` | Combined pack of all standard text-based scenarios. |

### Scenario Structure

Each scenario dictionary follows a consistent schema. While some legacy packs use simpler structures, modern packs (v2.0) include:

*   `name`: Unique identifier for the scenario.
*   `description`: Human-readable summary of the test objective.
*   `test_prompt`: The input string sent to the target model.
*   `expected_behavior`: List of criteria the judge should evaluate.
*   `category` / `subcategory`: Taxonomy for filtering and reporting.
*   `severity`: Risk level (`low`, `medium`, `high`).
*   `source`: Origin metadata (e.g., `real_case`, `synthetic`, `adapted`).
*   `metadata`: Author, creation date, rationale, and tags.

### Usage Examples

#### Retrieving a Pack

```python
from simpleaudit.scenarios import get_scenarios, list_scenario_packs

# List all available packs and their sizes
packs = list_scenario_packs()
print(packs)
# Output: {'safety': 8, 'rag': 8, 'health': 8, ...}

# Get scenarios for the 'safety' pack
safety_scenarios = get_scenarios("safety")

# Access the first scenario
first_scenario = safety_scenarios[0]
print(first_scenario["name"])
# Output: "Hallucination - Fictional Content"
print(first_scenario["test_prompt"])
```

#### Custom Scenario Validation

When creating custom scenarios, ensure names are unique to prevent data corruption in stability reports.

```python
from simpleaudit.scenarios import duplicate_scenario_names

custom_scenarios = [
    {"name": "My Test 1", "test_prompt": "Hello"},
    {"name": "My Test 2", "test_prompt": "World"},
    {"name": "My Test 1", "test_prompt": "Duplicate!"} # Duplicate name
]

duplicates = duplicate_scenario_names(custom_scenarios)
if duplicates:
    print(f"Warning: Duplicate scenario names found: {duplicates}")
    # Output: Warning: Duplicate scenario names found: {'My Test 1': 2}
```

#### Handling Special Packs

Some packs have specific runtime requirements. `vision_integrity` and `judge_the_judge` are deliberately excluded from the `all` pack to prevent failures in standard text-only setups.

```python
# Standard usage
all_scenarios = get_scenarios("all")

# Vision-specific usage (requires vision-capable models)
vision_scenarios = get_scenarios("vision_integrity")

# Judge qualification usage (requires WiggleRunner)
judge_scenarios = get_scenarios("judge_the_judge")
```

### Domain-Specific Notes

#### Healthcare and Safety
The `health` and `health_bullshit` packs focus on preventing harm. Scenarios test if the model refuses to diagnose, prescribe, or provide dangerous advice. The `health_bullshit` pack specifically tests resistance to "broken premises" (e.g., asking for antibiotic dosage for a viral infection).

#### Epistemic Safety
The `bullshitbench` packs test if the model accepts incoherent or nonsensical premises. A model that scores well demonstrates epistemic honesty, refusing to generate fluent but false answers when the question is structurally invalid.

#### Norwegian Public Sector
Packs like `hei_refusal`, `nav_aap`, `helfo`, and `lanekassen` contain scenarios in Norwegian (`language: "no"`). These test compliance with specific Norwegian administrative rules and youth advice boundaries. The `hei_refusal` pack includes both "guidance" cases (where the system should answer) and "refusal" cases (where the system should decline due to scope or safety).

### Best Practices

1.  **Do Not Mutate**: `get_scenarios()` returns a shallow copy. You can filter or append to the returned list without affecting the global registry.
2.  **Check for Duplicates**: Always run `duplicate_scenario_names()` on custom packs before auditing.
3.  **Match Judge to Pack**: Use an `abstention` judge for refusal-heavy packs like `hei_refusal`. Use a standard compliance judge for `safety` and `health`.
4.  **Verify Facts**: Rate-bearing facts in packs like `helfo` (e.g., co-payment amounts) are time-bounded. Re-verify against primary sources annually.

### See Also

*   [Judges](judges.md)
*   [Judge Validation](judge-validation.md)
*   [Visualization](visualization.md)
