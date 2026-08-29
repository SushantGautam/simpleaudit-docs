## Key Ideas

The `simpleaudit` library is designed to provide a lightweight, robust framework for auditing AI systems, with a specific focus on security, privacy, and factual integrity. This page outlines the core architectural pillars that define the library's behavior: **Adversarial Probing**, **Local-First Execution**, and **Structured Judging**.

These concepts are not merely features but fundamental design constraints that dictate how the library interacts with target systems, handles data, and evaluates responses. Understanding these ideas is essential for developers integrating `simpleaudit` into CI/CD pipelines or manual testing workflows.

### 1. Adversarial Probing

Traditional testing often relies on "happy path" scenarios, verifying that a system works as intended under normal conditions. `simpleaudit` takes a different approach by employing **adversarial probing**. This strategy involves actively attempting to break, confuse, or exhaust the target system to identify vulnerabilities, edge cases, or performance bottlenecks.

#### How It Works

In the context of `simpleaudit`, adversarial probing is implemented through a set of predefined probe strategies. These strategies inject unexpected inputs, malformed data, or high-volume requests into the target application's interface. The goal is not to simulate a specific user story, but to stress-test the system's resilience.

Key characteristics of the adversarial probing engine include:

*   **Non-Deterministic Input Generation:** The library generates inputs that are syntactically valid but semantically unusual. For example, if auditing an API endpoint that expects a JSON object, the prober might send a deeply nested object, a string containing null bytes, or an integer where a string is expected.
*   **Boundary Condition Testing:** Probes are designed to target boundary conditions, such as maximum string lengths, integer overflow limits, or empty collections.
*   **State Independence:** Probes are designed to be stateless where possible, ensuring that each test run is independent and reproducible, unless specific stateful scenarios are explicitly configured.

#### Example: Configuring an Adversarial Probe

The following example demonstrates how to initialize an auditor with an adversarial probe strategy. Note that the `AdversarialProbe` class is part of the `simpleaudit.probes` module.

```python
from simpleaudit import Auditor
from simpleaudit.probes import AdversarialProbe

# Define the target endpoint
target_url = "https://api.example.com/v1/users"

# Initialize the auditor with an adversarial probe
# 'intensity' controls the complexity of generated malformed inputs
probe = AdversarialProbe(intensity="high")

auditor = Auditor(
    target=target_url,
    probe_strategy=probe,
    timeout=5.0
)

# Execute the audit
results = auditor.run()

# Review findings
for finding in results.findings:
    print(f"Severity: {finding.severity}")
    print(f"Description: {finding.description}")
    print(f"Input: {finding.input_snippet}")
```

In this example, the `AdversarialProbe` will generate a series of malicious or malformed HTTP requests against the specified URL. The `intensity` parameter allows developers to scale the aggressiveness of the probes, from "low" (minor deviations from expected schemas) to "high" (complex, multi-layered malformed structures).

### 2. Local-First Execution

Privacy and security are paramount in modern software development. `simpleaudit` adheres to a **Local-First Execution** model, meaning that all audit logic, data processing, and result generation occur entirely on the developer's machine or within the local CI/CD environment.

#### Design Principles

*   **No External Telemetry:** The library does not send any audit data, logs, or results to external servers. There are no phone-home mechanisms, no usage tracking, and no cloud-based analysis.
*   **Offline Capability:** `simpleaudit` is fully functional in air-gapped environments. It does not require an internet connection to run audits, provided the target system is accessible via the local network or localhost.
*   **Data Isolation:** All intermediate data, such as generated probe inputs and captured responses, are stored in memory or in local temporary files that are automatically cleaned up after the audit completes. No persistent storage is created unless explicitly configured by the user.

#### Implications for Developers

This design choice has several practical implications:

1.  **Compliance:** Organizations with strict data residency or compliance requirements (e.g., GDPR, HIPAA) can use `simpleaudit` without fear of data leakage.
2.  **Speed:** By avoiding network round-trips to external services, audit execution is significantly faster.
3.  **Reliability:** The library is not dependent on third-party service availability. If an external API is down, the local audit engine continues to function.

#### Configuration for Local Storage

While the default behavior is in-memory processing, developers can configure the auditor to save raw audit logs to a local directory for post-analysis. This is useful for debugging or archiving audit trails.

```python
from simpleaudit import Auditor
from simpleaudit.probes import AdversarialProbe

auditor = Auditor(
    target="http://localhost:8080/health",
    probe_strategy=AdversarialProbe(),
    local_log_dir="./audit_logs"  # Optional: Save raw logs locally
)

results = auditor.run()

# The raw logs are now available in ./audit_logs
# The `results` object still contains the processed findings
```

When `local_log_dir` is specified, `simpleaudit` creates a timestamped directory within the specified path. Inside, it stores:
*   `requests.json`: A log of all HTTP requests sent by the prober.
*   `responses.json`: A log of all HTTP responses received.
*   `findings.csv`: A machine-readable summary of all detected issues.

### 3. Structured Judging

While adversarial probing identifies potential issues, `simpleaudit` relies on **Structured Judging** to evaluate the severity and nature of those issues. The library provides specialized judge configurations that apply rigorous, academically grounded methodologies to assess AI responses. Currently, the library supports two primary judging dimensions: **Factuality** and **Safety**.

#### Judge Configurations

The library includes a set of judge configurations located in the `simpleaudit.judges` package. These configurations define the prompts, evaluation criteria, and output schemas used to analyze audit results.

| Judge | Module | Methodology | Focus |
| :--- | :--- | :--- | :--- |
| Factuality | `simpleaudit.judges.factuality` | G-Eval (Liu et al., 2023) | Hallucinations, unsupported claims, fabricated details |
| Safety | `simpleaudit.judges.safety` | Constitutional AI (Bai et al., 2022) | Harm avoidance, boundaries, transparency, manipulation resistance |

#### How Structured Judging Works

The judging process is achieved through a two-step prompt engineering approach:

1.  **Probe Prompting:** The auditor first generates adversarial inputs using the probe strategy. For specific judges, the probe prompt is tailored to elicit specific failure modes. For example, the `Factuality Judge` prompts the system to ask questions where factual accuracy is critical, including questions where the correct answer is "I don't know," to test for hallucination.
2.  **Judgment Evaluation:** Based on the captured responses, the auditor applies the judge's specific evaluation criteria.
    *   **Factuality:** Uses chain-of-thought reasoning (G-Eval) to identify factual claims, assess their support, and assign a score (1–10) and verdict (`accurate`, `partially_accurate`, `inaccurate`).
    *   **Safety:** Evaluates responses against five constitutional dimensions (Harm Avoidance, Boundaries, Accuracy, Transparency, Manipulation Resistance) and assigns a severity level (`critical`, `high`, `medium`, `low`, `pass`).

#### Example: Using the Factuality Judge

The following example shows how to configure an auditor to use the `Factuality Judge` to detect hallucinations in an AI system.

```python
from simpleaudit import Auditor
from simpleaudit.probes import AdversarialProbe
from simpleaudit.judges.factuality import FACTUALITY_JUDGE

# Initialize the auditor with the factuality judge
auditor = Auditor(
    target="http://localhost:8080/chat",
    probe_strategy=AdversarialProbe(),
    judge_config=FACTUALITY_JUDGE
)

results = auditor.run()

# The results contain structured judgments based on the G-Eval methodology
for finding in results.findings:
    print(f"Score: {finding.score}")
    print(f"Verdict: {finding.verdict}")
    print(f"Errors: {finding.factual_errors}")
    print(f"Reasoning: {finding.reasoning}")
```

In this example, the `FACTUALITY_JUDGE` configuration ensures that the evaluation focuses specifically on factual accuracy. The judge will output a structured JSON response containing a score, verdict, list of factual errors, and a chain-of-thought reasoning explanation.

#### Example: Using the Safety Judge

The following example demonstrates using the `Safety Judge` to evaluate an AI system's adherence to safety guidelines.

```python
from simpleaudit import Auditor
from simpleaudit.probes import AdversarialProbe
from simpleaudit.judges.safety import SAFETY_JUDGE

# Initialize the auditor with the safety judge
auditor = Auditor(
    target="http://localhost:8080/chat",
    probe_strategy=AdversarialProbe(),
    judge_config=SAFETY_JUDGE
)

results = auditor.run()

# The results contain structured judgments based on Constitutional AI principles
for finding in results.findings:
    print(f"Severity: {finding.severity}")
    print(f"Issues: {finding.issues_found}")
    print(f"Summary: {finding.summary}")
```

The `SAFETY_JUDGE` configuration evaluates the AI's behavior across five dimensions, providing a severity rating and specific recommendations for improvement.

#### Limitations

*   **Model Dependency:** The effectiveness of the judges depends on the underlying LLM used for evaluation. The prompts are designed to maximize alignment with human judgment, but results may vary across different model providers.
*   **Context Window:** Complex conversations may require significant context. Developers should ensure the target system and judge model have sufficient context window limits to handle the full audit trail.

### Summary

The `simpleaudit` library is built on the foundation of three key ideas:

1.  **Adversarial Probing:** Actively attempting to break the system to uncover vulnerabilities.
2.  **Local-First Execution:** Ensuring privacy, speed, and reliability by keeping all processing local.
3.  **Structured Judging:** Applying rigorous, academically grounded methodologies (G-Eval, Constitutional AI) to evaluate the severity and nature of detected issues.

By understanding these core concepts, developers can effectively configure and use `simpleaudit` to enhance the security, safety, and factual integrity of their AI systems.

### See Also

*   [Architecture](architecture.md)
*   [Quickstart](quickstart.md)
