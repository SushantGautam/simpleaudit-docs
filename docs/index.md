# simpleaudit-docs

Lightweight AI safety auditing tool that validates comparative LLM safety scoring without ground-truth labels.

## Quick Start

```bash
pip install simpleaudit
```

## Guides

### Getting Started

- [Quickstart](guides/quickstart.md) — Run your first local audit using the CLI and core experiment classes.
- [Installation](guides/installation.md) — Install dependencies and configure local model paths for offline auditing.

### Core Concepts

- [Architecture](guides/architecture.md) — Understand the auditor, experiment, and result aggregation pipeline.
- [Key Ideas](guides/key-ideas.md) — Learn how custom judges and scenarios define adversarial test cases.

## API Reference

### Core

- [`simpleaudit`](reference/simpleaudit.md) — 
- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — 
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
- [`simpleaudit.repeated_results`](reference/simpleaudit_repeated_results.md) — 
- [`simpleaudit.results`](reference/simpleaudit_results.md) — 
- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — 
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — 
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — 
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — 
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — 
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — 
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — 
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — 
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — 
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — 
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — 
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — 
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — 

### CLI

- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — 

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
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — 
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — 
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — 
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — 
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — 
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — 
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — 

### Utilities & Visualization

- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — 
