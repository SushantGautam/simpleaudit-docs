# simpleaudit

Documentation for simpleaudit

## Quick Start

```bash
pip install simpleaudit
```

## Guides

### More

- [Getting Started](guides/getting-started.md) — Installation, environment setup, and running your first audit with a local model.
- [Core Architecture](guides/core-architecture.md) — Overview of the auditing pipeline, data flow, and how components interact.
- [CLI Usage](guides/cli-usage.md) — Command-line interface options, flags, and examples for running audits.
- [Creating Custom Scenarios](guides/creating-custom-scenarios.md) — How to define new test scenarios, including system prompts and data structures.
- [Implementing Judges](guides/implementing-judges.md) — Guide to creating custom LLM judges for specific evaluation criteria.
- [Advanced Evaluation Metrics](guides/advanced-evaluation-metrics.md) — Using cross-judging, repeated results, and meta-evaluation techniques.
- [Reframing and Data Processing](guides/reframing-and-data-processing.md) — Techniques for reframing prompts and processing audit data for analysis.
- [Visualization and Reporting](guides/visualization-and-reporting.md) — Using the local server to visualize audit results and generate reports.
- [Built-in Scenarios and Judges](guides/built-in-scenarios-and-judges.md) — Reference for the included health, safety, and factuality scenarios and judges.

## API Reference

### Core Engine

- [`simpleaudit`](reference/simpleaudit.md) — 
- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — 
- [`simpleaudit.cross_judge`](reference/simpleaudit_cross_judge.md) — 
- [`simpleaudit.experiment`](reference/simpleaudit_experiment.md) — 
- [`simpleaudit.judge_the_judge`](reference/simpleaudit_judge_the_judge.md) — 
- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — 
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — 
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — 
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — 
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — 
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — 
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — 
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — 
- [`simpleaudit.judges.judge_conviction`](reference/simpleaudit_judges_judge_conviction.md) — 
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — 
- [`simpleaudit.model_auditor`](reference/simpleaudit_model_auditor.md) — 
- [`simpleaudit.reframing`](reference/simpleaudit_reframing.md) — 
- [`simpleaudit.repeated_results`](reference/simpleaudit_repeated_results.md) — 
- [`simpleaudit.results`](reference/simpleaudit_results.md) — 
- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — 
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — 
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — 
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — 
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — 
- [`simpleaudit.scenarios.helfo`](reference/simpleaudit_scenarios_helfo.md) — 
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — 
- [`simpleaudit.scenarios.judge_the_judge`](reference/simpleaudit_scenarios_judge_the_judge.md) — 
- [`simpleaudit.scenarios.lanekassen`](reference/simpleaudit_scenarios_lanekassen.md) — 
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — 
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — 
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — 
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — 
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — 
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — 
- [`simpleaudit.scenarios.vision_integrity`](reference/simpleaudit_scenarios_vision_integrity.md) — 
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — 

### Judges

- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — 
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — 
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — 
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — 
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — 
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — 
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — 
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — 
- [`simpleaudit.judges.judge_conviction`](reference/simpleaudit_judges_judge_conviction.md) — 
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — 

### Scenarios

- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — 
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — 
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — 
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — 
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — 
- [`simpleaudit.scenarios.helfo`](reference/simpleaudit_scenarios_helfo.md) — 
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — 
- [`simpleaudit.scenarios.judge_the_judge`](reference/simpleaudit_scenarios_judge_the_judge.md) — 
- [`simpleaudit.scenarios.lanekassen`](reference/simpleaudit_scenarios_lanekassen.md) — 
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — 
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — 
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — 
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — 
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — 
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — 
- [`simpleaudit.scenarios.vision_integrity`](reference/simpleaudit_scenarios_vision_integrity.md) — 

### Analysis & Utilities

- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — 
- [`simpleaudit.cross_judge`](reference/simpleaudit_cross_judge.md) — 
- [`simpleaudit.judge_the_judge`](reference/simpleaudit_judge_the_judge.md) — 
- [`simpleaudit.reframing`](reference/simpleaudit_reframing.md) — 
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — 
