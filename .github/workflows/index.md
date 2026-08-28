# SimpleAudit

Lightweight AI safety auditing: LLM judges score model outputs for safety, harm, factuality, and helpfulness across curated scenario packs. Results are aggregated, compared across models and judges, and visualized in the browser.

## Quick Start

```bash
pip install simpleaudit
```

```python
from simpleaudit import ModelAuditor, get_scenarios

auditor = ModelAuditor(
    model="gpt-4o-mini",
    provider="openai",
    judge_model="gpt-4o",
    judge_provider="openai",
)
results = auditor.run(get_scenarios("safety"))
results.summary()
```

```bash
# Browse results in the browser
simpleaudit serve --results_dir ./results
```

## Guides

### Start Here

- [Getting Started](guides/getting-started.md) — Installation, environment setup, and first audit run.
- [Core Architecture](guides/core-architecture.md) — High-level system design, data flow, and module interactions.
- [CLI Usage](guides/cli-usage.md) — Command-line interface options, flags, and execution modes.

### Results & Tooling

- [Results & Analysis](guides/results-analysis.md) — Processing, storing, and interpreting audit results.

### More

- [Model Auditing](guides/model-auditing.md) — Core auditing logic, model loading, and inference handling.
- [Judges System](guides/judges-system.md) — Evaluation criteria, judge implementations, and scoring logic.
- [Test Scenarios](guides/test-scenarios.md) — Defining, structuring, and running adversarial test cases.
- [Custom Scenario Development](guides/custom-scenario-development.md) — Guidelines and patterns for creating new test scenarios.
- [Advanced Methodologies](guides/advanced-methodologies.md) — Reframing techniques and judge validation methods.
- [Visualization & Reporting](guides/visualization-reporting.md) — Generating visual reports and serving results via web server.

## API Reference

### Core

- [`simpleaudit`](reference/simpleaudit.md) — Top-level package: the public API of SimpleAudit.
- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — Command-line interface (`simpleaudit` entry point).
- [`simpleaudit.cross_judge`](reference/simpleaudit_cross_judge.md) — Cross-judge experiments: score the same outputs with different judges and compare.
- [`simpleaudit.experiment`](reference/simpleaudit_experiment.md) — Batch experiments: run audits across multiple models and compare them.
- [`simpleaudit.judge_the_judge`](reference/simpleaudit_judge_the_judge.md) — Judge validation: 'wiggle' the judge's inputs to measure judge robustness.
- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — Built-in judge configurations: safety, harm, factuality, helpfulness, abstention, and more.
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — Abstention judge: detects when the model should have refused/abstained.
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — Binary abstention judge: yes/no abstention classification.
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — Factuality judge: scores factual accuracy of model outputs.
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — Harm judge: scores harmfulness of model outputs.
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — Helpfulness judge: scores how helpful the model was.
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — Helsedir sex-health (NO) judge: domain-specific Norwegian health judge.
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — Helsedir sex-health (NO) RAG judge: grounded variant with retrieval context.
- [`simpleaudit.judges.judge_conviction`](reference/simpleaudit_judges_judge_conviction.md) — Judge conviction: meta-judge that extracts the candidate judge's current verdict for cross-checking.
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — Safety judge: scores safety of model outputs.
- [`simpleaudit.model_auditor`](reference/simpleaudit_model_auditor.md) — The core audit engine: drives target model, auditor, and judge across scenarios.
- [`simpleaudit.reframing`](reference/simpleaudit_reframing.md) — Reframing checks: rephrase a prompt and verify the model's answer stays consistent.
- [`simpleaudit.repeated_results`](reference/simpleaudit_repeated_results.md) — Stability analysis: repeated runs, fragility thresholds, and model stability reports.
- [`simpleaudit.results`](reference/simpleaudit_results.md) — Result containers: per-scenario results and aggregated audit results with summaries.
- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — Scenario packs: curated test suites (health, safety, government, benchmarks).
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — BullshitBench health variant: non-informative medical answers.
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — BullshitBench v1/v2 scenarios: measuring non-informative answers.
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — Health domain scenarios: medical Q&A and advice tests.
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — HEI refusal scenarios: Norwegian youth-advice Q&A (16 refusal + 31 guidance scenarios).
- [`simpleaudit.scenarios.helfo`](reference/simpleaudit_scenarios_helfo.md) — Helfo scenarios: Norwegian health insurance authority tests.
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — HelpMed scenarios: medical help-seeking tests.
- [`simpleaudit.scenarios.judge_the_judge`](reference/simpleaudit_scenarios_judge_the_judge.md) — Judge-the-judge scenarios: probes used to validate judges.
- [`simpleaudit.scenarios.lanekassen`](reference/simpleaudit_scenarios_lanekassen.md) — Lanekassen scenarios: Norwegian pension institution tests.
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — NAV AAP scenarios: Norwegian disability benefit tests.
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — RAG scenarios: retrieval-augmented generation tests.
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — Safety scenarios: refusal and harmful-request tests.
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — Skatteetaten scenarios: Norwegian tax authority tests.
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — System-prompt scenarios: tests of system prompt adherence.
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — UNG scenarios: youth health service tests.
- [`simpleaudit.scenarios.vision_integrity`](reference/simpleaudit_scenarios_vision_integrity.md) — Vision integrity scenarios: image-based integrity tests.
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — Shared helpers: LLM client wrappers, JSON parsing, and misc utilities.

### Judges

- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — Built-in judge configurations: safety, harm, factuality, helpfulness, abstention, and more.
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — Abstention judge: detects when the model should have refused/abstained.
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — Binary abstention judge: yes/no abstention classification.
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — Factuality judge: scores factual accuracy of model outputs.
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — Harm judge: scores harmfulness of model outputs.
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — Helpfulness judge: scores how helpful the model was.
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — Helsedir sex-health (NO) judge: domain-specific Norwegian health judge.
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — Helsedir sex-health (NO) RAG judge: grounded variant with retrieval context.
- [`simpleaudit.judges.judge_conviction`](reference/simpleaudit_judges_judge_conviction.md) — Judge conviction: meta-judge that extracts the candidate judge's current verdict for cross-checking.
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — Safety judge: scores safety of model outputs.

### Scenarios

- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — Scenario packs: curated test suites (health, safety, government, benchmarks).
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — BullshitBench health variant: non-informative medical answers.
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — BullshitBench v1/v2 scenarios: measuring non-informative answers.
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — Health domain scenarios: medical Q&A and advice tests.
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — HEI refusal scenarios: Norwegian youth-advice Q&A (16 refusal + 31 guidance scenarios).
- [`simpleaudit.scenarios.helfo`](reference/simpleaudit_scenarios_helfo.md) — Helfo scenarios: Norwegian health insurance authority tests.
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — HelpMed scenarios: medical help-seeking tests.
- [`simpleaudit.scenarios.judge_the_judge`](reference/simpleaudit_scenarios_judge_the_judge.md) — Judge-the-judge scenarios: probes used to validate judges.
- [`simpleaudit.scenarios.lanekassen`](reference/simpleaudit_scenarios_lanekassen.md) — Lanekassen scenarios: Norwegian pension institution tests.
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — NAV AAP scenarios: Norwegian disability benefit tests.
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — RAG scenarios: retrieval-augmented generation tests.
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — Safety scenarios: refusal and harmful-request tests.
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — Skatteetaten scenarios: Norwegian tax authority tests.
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — System-prompt scenarios: tests of system prompt adherence.
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — UNG scenarios: youth health service tests.
- [`simpleaudit.scenarios.vision_integrity`](reference/simpleaudit_scenarios_vision_integrity.md) — Vision integrity scenarios: image-based integrity tests.
