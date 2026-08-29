## Testing & Development

The `simpleaudit-docs` project includes a comprehensive test suite designed to verify the integrity of the `ModelAuditor` workflow without requiring live API keys or network access. This page covers running the test suite, utilizing the provided fake LLM clients for unit testing, and guidelines for contributing to the codebase.

### Running the Test Suite

The project uses `pytest` as its testing framework. To run the full test suite, execute the following command from the project root:

```bash
pytest tests/
```

To run specific test files, such as the basic package tests:

```bash
pytest tests/test_basic.py
```

The test suite is structured into two primary files:
1.  **`tests/test_basic.py`**: Validates core data structures (`AuditResult`, `AuditResults`), scenario pack availability, and error handling for missing API keys.
2.  **`tests/test_audit_flow.py`**: Tests end-to-end audit flows, save/load round-trips, plotting capabilities, and the `AuditExperiment` class using mocked LLM interactions.

### Fake LLM Clients for Unit Testing

Testing LLM-based applications typically requires expensive API calls. The `simpleaudit-docs` library provides a robust set of fake clients in `tests/fakes.py` that mimic the `AnyLLM` client interface. These fakes allow you to test the `ModelAuditor` logic in isolation.

The `ModelAuditor` interacts with three distinct LLM roles:
*   **Target Client**: The model being audited.
*   **Judge Client**: The model that evaluates the conversation and assigns a severity.
*   **Auditor Client**: The model that generates probe messages (falls back to the Judge if not specified).

#### Core Fake Classes

| Class | Description |
| :--- | :--- |
| `FakeClient` | A drop-in replacement for an AnyLLM client. Accepts a `response_fn` callable that returns the text response. |
| `ScriptedClient` | A client that consumes a pre-defined list of `(text, input_tokens, output_tokens)` tuples in order. Raises `ScriptedClientExhausted` if called more times than scripted. |

#### Factory Functions

`tests/fakes.py` provides several factory functions to quickly generate specific types of fake clients:

**Target Factories:**
*   `fixed_target(text: str)`: Returns a `FakeClient` that always responds with the same string.
*   `random_length_target(min_chars, max_chars)`: Returns a `FakeClient` that generates plausible filler text of random length.
*   `cycling_target(responses: Sequence)`: Returns a `FakeClient` that cycles through a list of responses indefinitely.

**Judge Factories:**
*   `fixed_severity_judge(severity: str)`: Returns a `FakeClient` that always assigns a specific severity (`"critical"`, `"high"`, `"medium"`, `"low"`, `"pass"`).
*   `cycling_severity_judge(severities: Sequence)`: Returns a `FakeClient` that cycles through a list of severities.

**Auditor (Probe) Factories:**
*   `fixed_probe_auditor(text: str)`: Returns a `FakeClient` that always generates the same probe message.
*   `cycling_probe_auditor(probes: Sequence)`: Returns a `FakeClient` that cycles through a list of probe messages.

#### Wiring Fakes into ModelAuditor

Use the `make_auditor` function to create a `ModelAuditor` instance wired with fake clients. This function patches the internal client creation logic, ensuring no network calls are made.

```python
import asyncio
from tests.fakes import make_auditor, fixed_target, fixed_severity_judge, fixed_probe_auditor

# Create an auditor with fixed responses
auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("Tell me more about this."),
    max_turns=1
)

# Run a scenario
result = asyncio.run(auditor.run_scenario(
    name="Test Scenario",
    description="A basic test"
))

assert result.severity == "pass"
```

You can also swap individual clients on an existing `ModelAuditor` instance to stub specific components while keeping others real (if API keys are available):

```python
# Stub only the judge, keep real target
auditor.judge_client = fixed_severity_judge("high")
```

### Example: End-to-End Mock Test

The following example demonstrates how to test a multi-turn conversation using `cycling_target` and `cycling_probe_auditor`:

```python
import asyncio
from tests.fakes import (
    make_auditor,
    cycling_target,
    fixed_severity_judge,
    cycling_probe_auditor
)

def test_multi_turn_audit():
    auditor = make_auditor(
        target=cycling_target([
            "2+2 equals 4.",
            "Sure, it's basic arithmetic."
        ]),
        judge=fixed_severity_judge("pass"),
        auditor=cycling_probe_auditor([
            "What is 2+2?",
            "Can you explain more?"
        ]),
        max_turns=2,
        show_progress=False
    )

    result = asyncio.run(
        auditor.run_scenario(
            name="Arithmetic Test",
            description="Testing basic math",
            max_turns=2
        )
    )

    assert isinstance(result, AuditResult)
    assert result.severity == "pass"
    # 2 turns x 2 messages (user + assistant) = 4 messages
    assert len(result.conversation) == 4
```

### Contributing

When contributing to the `simpleaudit-docs` codebase, please adhere to the following guidelines:

1.  **No Live API Calls in Tests**: All new tests must use the fake clients provided in `tests/fakes.py`. Do not write tests that require valid `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or other environment variables unless explicitly testing key validation logic (as seen in `test_model_auditor_requires_provider`).
2.  **Async/Await Patterns**: The `ModelAuditor` methods (`run_scenario`, `run_async`) are asynchronous. Use `asyncio.run()` in your tests to execute these coroutines.
3.  **Data Structure Integrity**: When modifying `AuditResult` or `AuditResults`, ensure that `to_dict()` and `save()`/`load()` round-trips remain consistent. Update `tests/test_audit_flow.py` to verify these behaviors.
4.  **Scenario Packs**: If you add new scenario packs, update `test_list_scenario_packs` in `tests/test_basic.py` to assert the new pack's presence and count.

By following these practices, you ensure that the test suite remains fast, deterministic, and independent of external service availability.

### See Also

*   [Local Model Setup](local-model-setup.md)
*   [Custom Scenarios](custom-scenarios.md)
*   [Results & Visualization](results-visualization.md)


**Container capabilities:** `AuditResults` can be iterated with `for` and supports index access with `[]` and supports `len()`.
