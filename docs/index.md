# simpleaudit

Documentation for simpleaudit

## Quick Start

```bash
pip install simpleaudit
```

## Guides

### Getting Started

- [Quickstart](guides/quickstart.md) — Run your first audit in under 5 minutes using the CLI and a pre-built scenario.
- [Installation](guides/installation.md) — Install SimpleAudit via pip or from source, including Python version requirements and optional dependencies for local model inference.

### Core Concepts

- [Architecture](guides/architecture.md) — High-level overview of the SimpleAudit pipeline: how experiments, scenarios, judges, and results interact.
- [Key Ideas](guides/key-ideas.md) — Core concepts including adversarial probing, local-first design, multilingual support, and the methodology for comparative safety scoring.

### More

- [Command Line Interface](guides/cli.md) — Reference for all CLI commands, flags, and options for running audits, managing models, and visualizing results.
- [Model Auditor](guides/model-auditor.md) — How to configure and use the ModelAuditor class to run probes against local or API-hosted LLMs.
- [Creating Custom Scenarios](guides/custom-scenarios.md) — Guide for building new test scenarios, including structure, system prompts, and best practices for adversarial probing.
- [Available Scenarios](guides/scenarios.md) — Catalog of built-in scenarios covering health, safety, factuality, and specific Norwegian public sector use cases.
- [Judges and Evaluation](guides/judges.md) — How to use and extend the judge system for evaluating model outputs, including abstention, harm, and helpfulness metrics.
- [Results and Analysis](guides/results.md) — Understanding result structures, aggregating repeated runs, and interpreting audit outputs.
- [Visualization Server](guides/visualization.md) — Launch the local web server to visualize audit results, compare models, and explore individual probe responses.
- [Image Generation Utilities](guides/image-utils.md) — Tools for generating and processing images used in vision-based integrity tests and scenario assets.

## API Reference

### Core Engine

- [`simpleaudit`](reference/simpleaudit.md) — 
- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — 
- [`simpleaudit.cross_judge`](reference/simpleaudit_cross_judge.md) — 
- [`simpleaudit.experiment`](reference/simpleaudit_experiment.md) — 
- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — 
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — 
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — 
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — 
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — 
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — 
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — 
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — 
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
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — 

### Scenarios

- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — 
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — 
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — 
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — 
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — 
- [`simpleaudit.scenarios.helfo`](reference/simpleaudit_scenarios_helfo.md) — 
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — 
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
- [`simpleaudit.reframing`](reference/simpleaudit_reframing.md) — 
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — 
