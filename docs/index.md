# simpleaudit-docs

Lightweight AI safety auditing tool that validates comparative LLM safety scoring without ground-truth labels.

## Quick Start

```bash
pip install simpleaudit
```

## Guides

### Getting Started

- [Quickstart](guides/quickstart.md) — Run your first local audit using the CLI and core auditor classes.
- [Installation](guides/installation.md) — Install dependencies and configure the local-first environment.

### Core Concepts

- [Architecture](guides/architecture.md) — Understand the experiment lifecycle, model auditing, and result handling.
- [Key Ideas](guides/key-ideas.md) — Learn how judges and scenarios define adversarial probing strategies.

## API Reference

### Core

- [`simpleaudit`](reference/simpleaudit.md) — SimpleAudit - Lightweight AI Safety Auditing Framework
- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — CLI interface for SimpleAudit tools.
- [`simpleaudit.experiment`](reference/simpleaudit_experiment.md) — audit experiment
- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — Built-in judge configurations for SimpleAudit.
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — Abstention judge configuration.
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — Binary abstention judge — language-agnostic.
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — Factuality judge configuration.
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — Harm categorisation judge configuration.
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — Helpfulness judge configuration.
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — Helsedirektoratet sexual-health judge — generic variant.
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — Helsedirektoratet sexual-health judge — RAG variant.
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — Safety judge configuration.
- [`simpleaudit.model_auditor`](reference/simpleaudit_model_auditor.md) — ModelAuditor for direct API-based model auditing.
- [`simpleaudit.repeated_results`](reference/simpleaudit_repeated_results.md) — Repeated-run results for SimpleAudit.
- [`simpleaudit.results`](reference/simpleaudit_results.md) — Results handling for SimpleAudit.
- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — Built-in scenario packs for SimpleAudit.
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — Bullshitbench health scenarios for SimpleAudit
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — BullshitBench Scenario Pack for SimpleAudit
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — Healthcare domain scenarios.
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — Hei refusal scenario pack.
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — HelpMed domain scenarios.
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — NAV AAP (Arbeidsavklaringspenger) scenario pack.
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — RAG-specific scenarios.
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — General AI safety scenarios.
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — Skatteetaten (Norwegian Tax Administration) scenario pack.
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — System prompt testing scenarios.
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — ung scenarios
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — Utility functions for SimpleAudit.

### CLI

- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — CLI interface for SimpleAudit tools.

### Judges

- [`simpleaudit.judges`](reference/simpleaudit_judges.md) — Built-in judge configurations for SimpleAudit.
- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — Abstention judge configuration.
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — Binary abstention judge — language-agnostic.
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — Factuality judge configuration.
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — Harm categorisation judge configuration.
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — Helpfulness judge configuration.
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — Helsedirektoratet sexual-health judge — generic variant.
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — Helsedirektoratet sexual-health judge — RAG variant.
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — Safety judge configuration.

### Scenarios

- [`simpleaudit.scenarios`](reference/simpleaudit_scenarios.md) — Built-in scenario packs for SimpleAudit.
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — Bullshitbench health scenarios for SimpleAudit
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — BullshitBench Scenario Pack for SimpleAudit
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — Healthcare domain scenarios.
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — Hei refusal scenario pack.
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — HelpMed domain scenarios.
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — NAV AAP (Arbeidsavklaringspenger) scenario pack.
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — RAG-specific scenarios.
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — General AI safety scenarios.
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — Skatteetaten (Norwegian Tax Administration) scenario pack.
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — System prompt testing scenarios.
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — ung scenarios

### Utilities & Visualization

- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — Utility functions for SimpleAudit.
