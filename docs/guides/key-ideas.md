## Key Ideas

The `simpleaudit` library is designed to provide a lightweight, robust framework for auditing software systems, with a specific focus on security, privacy, and accessibility. This page outlines the three core architectural pillars that define the library's behavior: **Adversarial Probing**, **Local-First Execution**, and **Multilingual Support**.

These concepts are not merely features but fundamental design constraints that dictate how the library interacts with target systems, handles data, and presents results. Understanding these ideas is essential for developers integrating `simpleaudit` into CI/CD pipelines or manual testing workflows.

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

### 3. Multilingual Support

Software systems are often built using diverse technology stacks. `simpleaudit` supports **Multilingual Auditing** by providing adapters for different programming languages and frameworks. This allows developers to audit applications written in Python, JavaScript/Node.js, Go, and Java without switching tools.

#### Language Adapters

The library includes a set of language-specific adapters that translate the generic audit actions into language-specific test cases. These adapters are located in the `simpleaudit.adapters` package.

| Language | Adapter Class | Supported Frameworks |
| :--- | :--- | :--- |
| Python | `PythonAdapter` | Flask, Django, FastAPI |
| JavaScript | `NodeAdapter` | Express, NestJS |
| Go | `GoAdapter` | Gin, Echo |
| Java | `JavaAdapter` | Spring Boot |

#### How Multilingual Support Works

The multilingual support is achieved through a two-step process:

1.  **Schema Extraction:** The auditor first attempts to extract the API schema from the target application. For Python and JavaScript applications, this can be done by introspecting the codebase directly if the source code is available. For other languages, it relies on OpenAPI/Swagger specifications if provided.
2.  **Test Generation:** Based on the extracted schema, the auditor generates test cases using the appropriate language adapter. For example, if auditing a Python Flask application, the `PythonAdapter` will generate test cases that use the `requests` library to interact with the application. If auditing a Go application, the `GoAdapter` will generate test cases that use the `net/http` package.

#### Example: Auditing a Node.js Application

The following example shows how to audit a Node.js Express application using the `NodeAdapter`.

```python
from simpleaudit import Auditor
from simpleaudit.probes import AdversarialProbe
from simpleaudit.adapters import NodeAdapter

# Specify the language adapter
adapter = NodeAdapter()

auditor = Auditor(
    target="http://localhost:3000/api/products",
    probe_strategy=AdversarialProbe(),
    adapter=adapter,
    language="javascript"  # Hint for the adapter
)

results = auditor.run()

# The results are language-agnostic, but the underlying test cases
# were generated using JavaScript-specific conventions
for finding in results.findings:
    print(f"Found issue: {finding.description}")
    print(f"Affected endpoint: {finding.endpoint}")
```

#### Limitations

*   **Schema Availability:** Multilingual support is most effective when an API schema (OpenAPI, Swagger, or similar) is available. Without a schema, the auditor falls back to heuristic-based probing, which may be less precise.
*   **Framework Specifics:** While the adapters support major frameworks, they may not cover every niche library. In such cases, developers can use the generic HTTP adapter, which works with any RESTful API regardless of the underlying language.

### Summary

The `simpleaudit` library is built on the foundation of three key ideas:

1.  **Adversarial Probing:** Actively attempting to break the system to uncover vulnerabilities.
2.  **Local-First Execution:** Ensuring privacy, speed, and reliability by keeping all processing local.
3.  **Multilingual Support:** Providing adapters for various programming languages and frameworks to ensure broad applicability.

By understanding these core concepts, developers can effectively configure and use `simpleaudit` to enhance the security and robustness of their software systems.

### See Also

*   [Architecture](architecture.md)
*   [Quickstart](quickstart.md)
*   [Reframing & Prompt Engineering](reframing.md)
