## Local Model Setup

Running AI safety audits typically requires access to large language models (LLMs) for both generating responses (the target) and evaluating them (the judge). While cloud APIs offer high performance, they incur costs and require external network access. **simpleaudit-docs** supports fully local inference via **Ollama**, allowing you to run complete audits on your local machine without API keys or cloud dependencies.

This guide covers how to configure Ollama as the inference backend, select appropriate models, and utilize named judge configurations for local evaluations.

### Prerequisites

Before running local audits, ensure the following are installed and running on your system:

1.  **Ollama**: Download and install from [ollama.ai](https://ollama.ai).
2.  **Ollama Service**: Start the background service:
    ```bash
    ollama serve
    ```
3.  **Models**: Pull the specific models you intend to use. For a basic setup, `llama3.2` or `gemma3` are recommended.
    ```bash
    ollama pull llama3.2
    ollama pull gemma3
    ```

### Configuring `ModelAuditor` for Ollama

The `ModelAuditor` class accepts a `provider` parameter to specify the inference backend. Set this to `"ollama"` for both the target model and the judge model to ensure a fully local workflow.

#### Basic Configuration

You can specify the model names directly as strings. The `model` parameter defines the system under test, while `judge_model` defines the evaluator.

```python
from simpleaudit import ModelAuditor, get_scenarios

# Define local models
TARGET_MODEL = "gemma3"      # The model being audited
JUDGE_MODEL = "gemma3"       # The model performing the audit (can be same or different)

auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    system_prompt="You are a helpful AI assistant. Be concise and accurate.",
    max_turns=3,  # Keep turns low for faster local testing
    verbose=True,
)

# Run a subset of scenarios
scenarios = get_scenarios("safety")[:3]
results = auditor.run(scenarios)
results.summary()
```

#### Key Parameters

| Parameter | Description |
| :--- | :--- |
| `model` | The name of the Ollama model to test (e.g., `"llama3.2"`, `"gemma3"`). |
| `provider` | Must be set to `"ollama"` for local inference. |
| `judge_model` | The name of the Ollama model used for evaluation. |
| `judge_provider` | Must be set to `"ollama"` if using a local judge. |
| `json_format` | **Important**: Set to `False` for Ollama. Ollama does not support OpenAI-style `json_object` response formatting. |
| `max_turns` | Number of conversation turns per scenario. Lower values (e.g., 2-3) are recommended for local models to reduce latency. |

### Named Judge Configurations

Instead of writing custom judge prompts, `simpleaudit` provides built-in named judge configurations grounded in published evaluation research. These are particularly useful for local setups where you want consistent, high-quality evaluations without manual prompt engineering.

You can list available configurations using `list_judge_configs()`:

```python
from simpleaudit import list_judge_configs

for name, desc in list_judge_configs().items():
    print(f"{name}: {desc}")
```

Common configurations include:

1.  **`safety`**: Based on Constitutional AI (Bai et al., 2022). Evaluates harm, manipulation, and boundary violations.
2.  **`helpfulness`**: Based on MT-Bench (Zheng et al., 2023). Evaluates relevance, accuracy, clarity, and completeness.
3.  **`factuality`**: Based on G-Eval (Liu et al., 2023). Detects hallucinations and factual errors.

#### Example: Running a Safety Audit with Local Models

The following example demonstrates a complete local audit using the `safety` judge configuration.

```python
from simpleaudit import ModelAuditor, get_scenarios

TARGET_MODEL = "llama3.2:3b"  # Small/fast target model
JUDGE_MODEL = "gemma3:latest" # Larger judge model for better accuracy

auditor = ModelAuditor(
    model=TARGET_MODEL,
    provider="ollama",
    judge_model=JUDGE_MODEL,
    judge_provider="ollama",
    judge="safety",       # Use the named 'safety' configuration
    json_format=False,    # Required for Ollama
    max_turns=2,
    verbose=False,
    show_progress=True,
)

scenarios = get_scenarios("safety")[:5]
results = auditor.run(scenarios)

# Display per-scenario severity
for r in results:
    print(f"{r.scenario_name}: {r.severity.upper()}")
    if r.summary:
        print(f"  -> {r.summary[:100]}")

results.save("local_safety_audit.json")
```

### Performance Considerations

Local inference is significantly slower than cloud APIs. To optimize your workflow:

1.  **Model Selection**: Use smaller models (e.g., `llama3.2:3b`) for the target if speed is critical. Use larger models (e.g., `gemma3`, `llama3:8b`) for the judge to ensure evaluation quality.
2.  **Scenario Count**: Start with a small subset of scenarios (e.g., 3-5) to verify your setup before running the full suite.
3.  **Max Turns**: Reduce `max_turns` to 2 or 3. Multi-turn conversations increase latency exponentially on local hardware.
4.  **JSON Format**: Always set `json_format=False` when using Ollama to avoid parsing errors.

### Testing with a Mock Server

If you do not have Ollama installed or wish to test the `simpleaudit` framework without running heavy local models, you can use the provided mock server. This server simulates an OpenAI-compatible API with intentional safety issues for testing purposes.

1.  **Install dependencies**:
    ```bash
    pip install fastapi uvicorn
    ```

2.  **Start the mock server**:
    ```bash
    python examples/mock_server.py
    ```

3.  **Run audit against mock server**:
    ```python
    from simpleaudit import ModelAuditor

    auditor = ModelAuditor(
        model="mock",
        provider="openai",
        base_url="http://localhost:8000/v1",
        judge_model="judge",
        judge_provider="openai",
        judge_base_url="http://localhost:8000/v1",
    )

    results = auditor.run("safety")
    results.summary()
    ```

This approach is useful for CI/CD pipelines or development environments where local GPU resources are unavailable.

### Saving and Visualizing Results

Local audits produce the same result objects as cloud-based audits. You can save results to JSON and generate charts if `matplotlib` is installed.

```python
# Save results
results.save("local_audit_results.json")

# Generate chart (optional)
try:
    results.plot(save_path="local_audit_chart.png")
    print("Chart saved to local_audit_chart.png")
except ImportError:
    print("Install matplotlib for charts: pip install simpleaudit[plot]")
```

By following this guide, you can perform rigorous AI safety audits entirely on your local infrastructure, ensuring data privacy and eliminating API costs.

### See Also

*   [Custom Judges](custom-judges.md)
*   [Results & Visualization](results-visualization.md)
*   [Testing & Development](testing-development.md)
*   [Installation](installation.md)
