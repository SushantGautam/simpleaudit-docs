## Testing

The `simpleaudit` test suite is designed to verify the logic of the LLM safety auditing framework without requiring valid API keys or network access. By utilizing a set of configurable fake clients, developers can run the full audit pipeline in isolation, ensuring that changes to the `ModelAuditor` logic, result aggregation, or data serialization do not introduce regressions.

This page documents how to run the test suite and explains the mocking strategy used to simulate LLM interactions.

### Running the Test Suite

The project uses `pytest` as its test runner. To execute the entire test suite, run the following command from the project root:

```bash
pytest tests/
```

To run a specific test file, such as the basic integration tests:

```bash
pytest tests/test_basic.py
```

To run a specific test class or method:

```bash
pytest tests/test_audit_flow.py::TestEndToEndMockAudit::test_run_scenario_produces_valid_result
```

### Mocking Strategy

The core challenge in testing LLM applications is that the primary dependencies (the target model, the judge model, and the probe generator) are external API services. `simpleaudit` addresses this by providing a `tests/fakes.py` module that contains drop-in replacements for the `AnyLLM` client interface.

These fakes mimic the structure of the `acompletion` response expected by `ModelAuditor._call_async`. Specifically, they return an object with the following access pattern:

```python
response.choices[0].message.content
response.usage.prompt_tokens
response.usage.completion_tokens
```

#### Shared Fixtures

The `tests/conftest.py` file defines an autouse fixture, `clear_image_cache`, which clears the cache for `simpleaudit.utils.image_data_uri` before and after each test. This ensures that tests remain independent and do not interfere with one another through shared state.

### Fake Clients

The `tests/fakes.py` module provides several classes and factory functions to simulate different LLM behaviors.

#### `FakeClient`

A basic drop-in replacement for an LLM client. It accepts a `response_fn` callable that returns a string. This is useful for simple, static responses.

```python
from tests.fakes import FakeClient

client = FakeClient(lambda **kwargs: "I cannot help with that.")
```

#### `ScriptedClient`

A client that consumes a pre-defined list of responses in order. It is useful for testing multi-turn conversations where the model's response depends on the turn count. If the client is called more times than there are scripted responses, it raises a `ScriptedClientExhausted` exception, which helps catch bugs in the conversation loop.

```python
from tests.fakes import ScriptedClient

client = ScriptedClient([
    ("First turn response", 10, 5),
    ("Second turn response", 12, 6),
])
```

#### Factory Functions

For convenience, `tests/fakes.py` provides several factory functions that return pre-configured `FakeClient` or `ScriptedClient` instances:

*   **`fixed_target(text)`**: Returns a target client that always responds with the same string.
*   **`random_length_target(min_chars, max_chars)`**: Returns a target client that generates random filler text within a specified length range. Useful for testing token counting or truncation logic.
*   **`cycling_target(responses)`**: Returns a target client that cycles through a list of responses indefinitely.
*   **`fixed_severity_judge(severity)`**: Returns a judge client that always assigns the specified severity (`"critical"`, `"high"`, `"medium"`, `"low"`, or `"pass"`). The response is formatted as valid JSON matching the expected judge output schema.
*   **`cycling_severity_judge(severities)`**: Returns a judge client that cycles through a list of severities.
*   **`fixed_probe_auditor(text)`**: Returns an auditor client that always generates the same probe message.
*   **`cycling_probe_auditor(probes)`**: Returns an auditor client that cycles through a list of probe messages.

### Wiring Fakes into `ModelAuditor`

The `make_auditor` function in `tests/fakes.py` simplifies the process of creating a `ModelAuditor` instance with fake clients. It patches the `_create_anyllm_client` method during initialization to prevent actual API client creation, then injects the provided fake clients into the `target_client`, `judge_client`, and `auditor_client` attributes.

#### Example: Basic Audit Flow

The following example demonstrates how to run a single scenario using fake clients:

```python
import asyncio
from tests.fakes import make_auditor, fixed_target, fixed_severity_judge, fixed_probe_auditor

def test_basic_audit():
    # Create an auditor with fakes
    auditor = make_auditor(
        target=fixed_target("I cannot help with that."),
        judge=fixed_severity_judge("pass"),
        auditor=fixed_probe_auditor("Tell me more about this."),
        max_turns=1,
        show_progress=False,
    )

    # Run the scenario asynchronously
    result = asyncio.run(auditor.run_scenario(
        name="Test Scenario",
        description="A simple test",
        max_turns=1,
    ))

    # Assert results
    assert result.scenario_name == "Test Scenario"
    assert result.severity == "pass"
```

#### Example: Multi-Turn Conversation

To test multi-turn interactions, use `cycling_target` and `cycling_probe_auditor` to provide distinct responses for each turn:

```python
import asyncio
from tests.fakes import make_auditor, cycling_target, cycling_probe_auditor, fixed_severity_judge

def test_multi_turn():
    auditor = make_auditor(
        target=cycling_target(["First response.", "Second response."]),
        judge=fixed_severity_judge("low"),
        auditor=cycling_probe_auditor(["Question 1", "Question 2"]),
        max_turns=2,
        show_progress=False,
    )

    result = asyncio.run(auditor.run_scenario(
        name="Multi-Turn Test",
        description="Testing multiple turns",
        max_turns=2,
    ))

    # 2 turns * 2 messages (user + assistant) = 4 messages
    assert len(result.conversation) == 4
    assert result.severity == "low"
```

### Test Structure

The test suite is organized into several key files:

1.  **`tests/test_basic.py`**: Contains basic tests for package imports, scenario pack availability, and `AuditResult`/`AuditResults` data structures. It also verifies that `ModelAuditor` raises a `MissingApiKeyError` when no API keys are configured.
2.  **`tests/test_audit_flow.py`**: Contains end-to-end tests for the audit flow, including:
    *   Running scenarios with mocked clients.
    *   Testing the synchronous `run()` wrapper and its behavior within an active event loop.
    *   Verifying that system prompts are correctly forwarded to the target model.
    *   Testing save/load round-trips for `AuditResults`.
    *   Smoke tests for the `plot()` method (skipped if `matplotlib` is not installed).
    *   Tests for the `AuditExperiment` class.

### Best Practices

*   **Use `make_auditor`**: Prefer using the `make_auditor` factory function over manually instantiating `ModelAuditor` and patching attributes. This ensures that the internal client creation logic is properly bypassed.
*   **Isolate Async Tests**: Use `asyncio.run()` to execute asynchronous audit methods within synchronous test functions. Do not call the synchronous `run()` method from within an active event loop, as it will raise a `RuntimeError`.
*   **Check for Optional Dependencies**: Tests that rely on optional dependencies like `matplotlib` should use `pytest.mark.skipif` to skip gracefully if the dependency is not installed.
*   **Clean Up Temp Files**: When testing file I/O (e.g., saving/loading results), use `tempfile.NamedTemporaryFile` and ensure the file is deleted in a `finally` block to avoid leaving artifacts in the filesystem.

By leveraging these fakes and fixtures, developers can write fast, reliable, and deterministic tests for the `simpleaudit` framework without the overhead and instability of real LLM API calls.

### See Also

*   [Judges](judges.md)
*   [Quickstart](quickstart.md)
*   [Scenarios](scenarios.md)
