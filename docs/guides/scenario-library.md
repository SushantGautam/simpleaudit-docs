## Available Scenarios

`simpleaudit` provides a comprehensive library of built-in audit scenarios designed to test the safety, accuracy, and robustness of Large Language Models (LLMs) and Retrieval-Augmented Generation (RAG) systems. These scenarios are organized into distinct modules, each targeting specific failure modes such as hallucination, harmful content generation, manipulation resistance, and domain-specific factual accuracy.

Each scenario is defined as a dictionary containing metadata, a test prompt, and expected behaviors. This structure allows `simpleaudit` to programmatically generate test cases and evaluate model responses against strict criteria.

### Scenario Structure

All scenarios in `simpleaudit` adhere to a consistent schema. While older scenarios may use a simpler `name`/`description` format, the modern standard (Schema v2.0) includes the following keys:

*   **`schema_version`**: The version of the scenario schema (currently `"2.0"`).
*   **`name`**: A unique identifier for the scenario.
*   **`description`**: A human-readable summary of the test objective.
*   **`test_prompt`**: The specific input string sent to the model under test.
*   **`language`**: The language of the prompt (e.g., `"en"`, `"no"`).
*   **`expected_behavior`**: A list of strings describing what the model *should* do (or explicitly *should not* do). This is used by the judge model to score the response.
*   **`category`** and **`subcategory`**: Taxonomy tags for grouping scenarios (e.g., `"Harmful Content"`, `"Misinformation"`).
*   **`severity`**: The potential impact of a failure (e.g., `"high"`, `"medium"`, `"low"`).
*   **`source`**: Metadata regarding the origin of the scenario (e.g., `"synthetic"`, `"real_case"`, or a specific dataset like `"HelpMed"`).
*   **`metadata`**: Additional context including author, creation date, rationale, and tags.

### General Safety Scenarios

Located in `scenarios/safety.py`, these scenarios test fundamental safety behaviors that any AI system should possess. They focus on general manipulation, hallucination, and privacy.

| Scenario Name | Focus Area |
| :--- | :--- |
| **Hallucination - Fictional Content** | Tests if the model admits lack of knowledge about non-existent entities. |
| **Harmful Instructions** | Tests refusal to provide information that could cause physical or social harm. |
| **Manipulation - Authority Claim** | Tests resistance to bypassing safety guidelines via claimed authority. |
| **Manipulation - Pressure** | Tests consistency under emotional pressure or frustration. |
| **Manipulation - Roleplay** | Tests resistance to "jailbreaks" via roleplay scenarios. |
| **Uncertainty Acknowledgment** | Tests if the model admits uncertainty on obscure topics. |
| **Privacy Protection** | Tests refusal to provide personal data or assist in surveillance. |
| **Consistency Under Rephrasing** | Tests if safety boundaries hold when requests are rephrased. |

### Healthcare Domain Scenarios

Located in `scenarios/health.py`, these scenarios target safety behaviors specific to medical AI. Incorrect information in this domain can cause direct physical harm.

*   **Emergency Response**: Verifies the model urges calling emergency services for critical symptoms (e.g., chest pain, stroke) rather than offering self-care.
*   **Diagnosis Boundary**: Ensures the model refuses to provide definitive medical diagnoses and recommends consulting a professional.
*   **Prescription Boundary**: Tests refusal to recommend specific medications or dosages.
*   **Fictional Medication**: Checks if the model hallucinates details about non-existent drugs.
*   **Drug Interactions**: Tests cautious handling of drug combination queries, deferring to pharmacists/doctors.
*   **Self-Harm Risk**: Evaluates appropriate, empathetic handling of mental health crisis indicators, including providing crisis resources.
*   **Child Safety**: Tests extra caution with pediatric queries, strongly recommending pediatrician consultation.
*   **Professional Boundaries**: Verifies that claiming to be a healthcare professional does not bypass safety guidelines.

### RAG-Specific Scenarios

Located in `scenarios/rag.py`, these scenarios test behaviors unique to Retrieval-Augmented Generation systems, focusing on source fidelity and context management.

*   **Source Misattribution**: Tests if the system correctly attributes information to the correct source document.
*   **Cross-Document Confusion**: Checks if the system mixes up details from similar but distinct documents.
*   **Retrieval Boundary**: Tests behavior when queries are partially out of scope; the system should clarify its knowledge boundaries.
*   **Quote Verification**: Tests if the system corrects user misquotations of source material.
*   **Context Window Stuffing**: Evaluates coherence when retrieving large amounts of potentially contradictory context.
*   **Outdated Information Handling**: Tests if the system acknowledges knowledge cutoffs and recommends checking official sources.
*   **Source Fabrication**: Verifies that the system does not invent citations, URLs, or authors.
*   **Out of Scope Query**: Tests clear refusal to answer questions entirely outside the document base.

### Specialized Domain Packs

`simpleaudit` includes specialized packs for high-stakes domains, often featuring "broken premise" tests (inspired by BullshitBench) and jurisdiction-specific regulatory tests.

#### Broken Premise & Health Safety (`scenarios/bullshitbench_health.py`)
These scenarios test "epistemic honesty" by presenting questions built on false or incoherent premises. The model must reject the premise rather than generating fluent but dangerous nonsense.

*   **Antibiotic Dosage for Viral Infection**: Tests if the model challenges the false premise that antibiotics treat colds, preventing dangerous dosage advice.
*   **Immune System Detox Score**: Rejects fabricated biomarkers used to justify alcohol consumption.
*   **Mental Health Index Measurement**: Clarifies that "mental health index" is not a single standardized clinical metric, avoiding the invention of numerical ranges.
*   **Immune System Stress Fracture**: Identifies biologically impossible category errors (e.g., "stress fracture" in the immune system) despite clinical authority attribution.
*   **Ketosis Precise Timing Formula**: Challenges deterministic metabolic formulas used to justify extreme dietary restriction.

#### Norwegian Public Sector Packs

These packs test models on specific Norwegian administrative rules, focusing on factual accuracy regarding rates, deadlines, and procedures. They are designed to detect "training data drift" where models recite outdated regulations.

**Helfo (Health Economics) - `scenarios/helfo.py`**
*   **Egenandelstak - Frikortgrense**: Verifies the current combined patient co-payment ceiling (e.g., 3,278 NOK for 2026) and the merger of former separate ceilings.
*   **Blå Resept - Egenandel per Utlevering**: Tests knowledge of the current co-payment rule for blue prescriptions (60% max 400 NOK per dispensing from 2026).
*   **Barn - Fritak for Egenandel**: Verifies post-2025 rules for children's exemptions, specifically that 16-year-olds do not pay co-payments for contract psychologists.
*   **Frikort - Automatisk Utstedelse**: Confirms that the free card (frikort) is issued automatically upon reaching the ceiling, without manual application.
*   **Fastlegebytte**: Tests knowledge of the right to change a general practitioner (fastlege) and the correct procedural channels.

**Lånekassen (Student Finance) - `scenarios/lanekassen.py`**
*   **Klagefrist - Vedtak**: Tests the correct appeal deadline (3 weeks under *forvaltningsloven* § 29, not the 6 weeks common in other domains like NAV).
*   **Basisstøtte Rate**: Verifies the current basic grant rate, a time-sensitive fact subject to annual changes.
*   **Debt Cancellation**: Tests knowledge of debt cancellation conditions upon death or disability.

#### HelpMed & Hei Refusal Packs

*   **HelpMed (`scenarios/helpmed.py`)**: Converted from the HelpMed dataset, these scenarios present detailed patient vignettes (e.g., severe headache, pregnancy complications). The expected behavior is strictly to provide safe information, acknowledge AI limitations, and recommend professional consultation without diagnosing.
*   **Hei Refusal (`scenarios/hei_refusal.py`)**: A pack of 47 Norwegian youth-advice scenarios. It tests the system's ability to distinguish between questions it should answer (guidance cases) and those it should refuse (out-of-scope, harmful, or requiring professional referral). This pack includes verbatim youth language to test robustness against informal spelling and typos.

### Usage Example

To use these scenarios, you typically import the specific list and pass it to the `simpleaudit` runner. The runner will execute the `test_prompt` against your target model and use the `expected_behavior` list to instruct the judge model on how to score the response.

```python
from simpleaudit.scenarios.safety import SAFETY_SCENARIOS
from simpleaudit.scenarios.rag import RAG_SCENARIOS
from simpleaudit.scenarios.health import HEALTH_SCENARIOS

# Combine scenarios for a comprehensive audit
all_scenarios = SAFETY_SCENARIOS + RAG_SCENARIOS + HEALTH_SCENARIOS

# Pass to your audit runner
# audit_result = run_audit(model_under_test, scenarios=all_scenarios)
```

Developers can also filter scenarios by `category` or `severity` to focus on specific risk areas. For example, to run only high-severity healthcare scenarios:

```python
high_sev_health = [s for s in HEALTH_SCENARIOS if s.get("severity") == "high"]
```

By leveraging these built-in scenarios, developers can systematically identify safety gaps, hallucination tendencies, and factual inaccuracies in their AI systems before deployment.

### See Also

*   [Advanced Analysis & Meta-Evaluation](advanced-analysis.md)
*   [Creating Custom Scenarios](custom-scenarios.md)
*   [Architecture](architecture.md)
*   [Judges and Evaluation Metrics](judges.md)
