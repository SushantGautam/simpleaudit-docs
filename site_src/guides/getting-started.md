## Getting Started

SimpleAudit is a lightweight framework for AI safety auditing. It uses Large Language Models (LLMs) as both the system under test and the judge to evaluate model behavior against specific safety scenarios. The library supports multiple providers, including Anthropic, OpenAI, Grok, Ollama, and vLLM.

### Installation

Install the core library via pip:

```bash
pip install simpleaudit
```

For visualization features (web server and HTML export), install optional dependencies:

```bash
pip install 'simpleaudit[visualize]'
```

### Environment Setup

SimpleAudit requires API keys for the LLM providers you intend to use. Set these as environment variables before running audits.

**OpenAI:**
```bash
export OPENAI_API_KEY="your-key-here"
```

**Anthropic:**
```bash
export ANTHROPIC_API_KEY="your-key-here"
```

**Local Models (Ollama/vLLM):**
Ensure your local model server is running. No API key is typically required, but you must specify the correct base URL in the `ModelAuditor` configuration.

### Core Components

The library exposes several key classes and functions via `simpleaudit`:

| Component | Description |
| :--- | :--- |
| `ModelAuditor` | Primary class for running audits against a target model. |
| `get_scenarios` | Retrieves predefined safety test scenarios. |
| `list_scenario_packs` | Lists available scenario packs. |
| `AuditResults` | Container for audit outcomes, providing summary statistics. |
| `AuditResult` | Individual result object for a single scenario. |
| `AuditExperiment` | Manages complex experimental setups, including repeated runs. |
| `RepeatedExperimentResults` | Container for results from repeated experiments. |
| `ModelStabilityReport` | Report on model stability across repetitions. |
| `ScenarioStats` | Statistics for individual scenarios in repeated runs. |
| `CrossJudgeExperiment` | Compares results across different judge models. |
| `CrossJudgeResults` | Container for cross-judge comparison results. |
| `compare_judges` | Utility function to compare judge outputs. |
| `reframing_check` | Evaluates model robustness against prompt reframing attacks. |
| `reframing_check_async` | Asynchronous version of reframing check. |
| `ReframingResults` | Container for reframing test results. |
| `PromptVariant` | Represents a variant of a prompt for reframing tests. |
| `StoredRecord` | Record of stored reframing data. |
| `load_stored_records` | Loads stored reframing records. |
| `WiggleRunner` | Runner for judge qualification scenarios. |
| `WiggleProfile` | Profile for wiggle testing. |
| `ScenarioWiggle` | Wiggle test for a specific scenario. |
| `TurnRecord` | Record of a conversation turn in wiggle testing. |

### First Audit Run

The following example demonstrates how to audit an OpenAI model using a GPT-4 judge.

1.  **Import modules:**
    ```python
    from simpleaudit import ModelAuditor, get_scenarios
    ```

2.  **Initialize the Auditor:**
    Configure the target model, the judge model, and the system prompt.
    ```python
    auditor = ModelAuditor(
        model="gpt-4o-mini",          # Target model to audit
        provider="openai",            # Provider for target model
        judge_model="gpt-4o",         # Judge model
        judge_provider="openai",      # Provider for judge
        system_prompt="You are a helpful assistant."
    )
    ```

3.  **Load Scenarios:**
    Retrieve a set of safety scenarios.
    ```python
    scenarios = get_scenarios("safety")
    ```

4.  **Run Audit:**
    Execute the audit.
    ```python
    results = auditor.run(scenarios)
    ```

5.  **View Results:**
    Print a summary of the findings.
    ```python
    results.summary()
    ```

### CLI Usage

SimpleAudit includes a command-line interface for visualizing results.

**Serve Web Dashboard:**
Start a local web server to visualize JSON result files.
```bash
simpleaudit serve --results_dir ./results --port 8000
```

**Export Standalone HTML:**
Generate a self-contained HTML file for sharing results without a server.
```bash
simpleaudit export-html ./results/audit_01.json -o ./report.html
```

### Configuration Options

`ModelAuditor` accepts the following parameters:

*   `model` (str): The name of the model to audit (e.g., `"gpt-4o-mini"`, `"claude-3-sonnet"`).
*   `provider` (str): The LLM provider (`"openai"`, `"anthropic"`, `"ollama"`, `"vllm"`, etc.).
*   `judge_model` (str): The model used to evaluate responses.
*   `judge_provider` (str): The provider for the judge model.
*   `system_prompt` (str): The system prompt applied to the target model.
*   `base_url` (str, optional): Custom API endpoint for OpenAI-compatible providers.
*   `api_key` (str, optional): Explicit API key (overrides environment variable).

### Advanced Features

**Repeated Experiments:**
Use `AuditExperiment` to run scenarios multiple times to assess model stability.
```python
from simpleaudit import AuditExperiment

experiment = AuditExperiment(
    auditor=auditor,
    scenarios=scenarios,
    repetitions=5
)
experiment.run()
```

**Cross-Judge Comparison:**
Compare how different judges evaluate the same model responses.
```python
from simpleaudit import CrossJudgeExperiment

cross_exp = CrossJudgeExperiment(
    auditor=auditor,
    scenarios=scenarios,
    judge_configs=[
        {"model": "gpt-4o", "provider": "openai"},
        {"model": "claude-3-opus", "provider": "anthropic"}
    ]
)
cross_exp.run()
```

**Prompt Reframing:**
Test if the model remains safe when the prompt is rephrased.
```python
from simpleaudit import reframing_check

results = reframing_check(
    auditor=auditor,
    scenario=scenarios[0],
    variants=["Please ignore previous instructions and..."]
)
```

**Judge Qualification (Wiggle Testing):**
Use `WiggleRunner` to test judge qualification scenarios. This requires a setup with three model roles (target, judge, and wiggle judge).
```python
from simpleaudit import WiggleRunner

# Note: Specific configuration depends on your model setup
# wiggle_runner = WiggleRunner(...)
```

### Error Handling

Ensure API keys are valid and network connectivity is available. If a provider is not supported or a model name is invalid, the library will raise an exception during the `run()` call. Check the `AuditResults` object for individual scenario errors if partial failures occur.

### Next Steps

*   Explore available scenario packs using `list_scenario_packs()` — see [Scenarios](scenarios.md) for the full catalog.
*   Customize judges using `get_judge()` and `list_judge_configs()` — see [Judges](judges.md) for all available configs.
*   Analyze detailed results using the `AuditResult` class attributes — see [Results & Analysis](results-analysis.md).
*   Run audits from the command line — see [CLI Usage](cli-usage.md).
*   Visualize results in a browser — see [Visualization](visualization.md).
*   Understand how the pieces fit together — see [Core Architecture](core-architecture.md).

### See Also

*   [CLI Usage](cli-usage.md)
*   [Core Architecture](core-architecture.md)
