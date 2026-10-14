# SimpleAudit

Lightweight AI safety auditing framework that validates comparative LLM safety scoring without ground-truth labels.

## Quick Start

```bash
pip install simpleaudit
```

## Guides

### Getting Started

- Installation (not yet available) — Install SimpleAudit via pip and verify the CLI entry point.
- Quickstart (not yet available) — Run your first single-turn audit and inspect the resulting report.

### Core Concepts

- Auditing Basics (not yet available) — Understand the audit loop, experiment configuration, and perturbation strategies.
- Targets And Judges (not yet available) — Configure local or API model targets and select built-in evaluation judges.
- Scenarios (not yet available) — Explore predefined test scenarios for health, safety, and RAG contexts.

### Advanced Usage

- [Context Analysis](guides/context-analysis.md) — Analyze context attribution, derivations, and span-level findings in responses.
- Tracing And Observability (not yet available) — Enable OpenTelemetry tracing and aggregate statistical metrics from audits.
- Custom Judges (not yet available) — Build custom composite judges using checklists and cross-validation logic.

## API Reference

### Core Engine

- [`simpleaudit.auditor`](reference/simpleaudit_auditor.md) — Auditor — the primary, target-agnostic entry point.
- [`simpleaudit.checklist`](reference/simpleaudit_checklist.md) — Evidence-anchored checklist grading: verification and deterministic derivation.
- [`simpleaudit.decision`](reference/simpleaudit_decision.md) — Structured decision questions for scenarios.
- [`simpleaudit.experiment`](reference/simpleaudit_experiment.md) — experiment event
- [`simpleaudit.model_auditor`](reference/simpleaudit_model_auditor.md) — ModelAuditor for direct API-based model auditing.
- [`simpleaudit.repeated_results`](reference/simpleaudit_repeated_results.md) — Repeated-run results for SimpleAudit.
- [`simpleaudit.results`](reference/simpleaudit_results.md) — Results handling for SimpleAudit.
- [`simpleaudit.single_turn`](reference/simpleaudit_single_turn.md) — Single-turn auditing.
- [`simpleaudit.stats`](reference/simpleaudit_stats.md) — Pure statistics over audit verdicts.

### Targets

- [`simpleaudit.targets.base`](reference/simpleaudit_targets_base.md) — Core Target protocol and shared data types.
- [`simpleaudit.targets.callable`](reference/simpleaudit_targets_callable.md) — CallableTarget — an in-process Python callable as an audit target.
- [`simpleaudit.targets.decision`](reference/simpleaudit_targets_decision.md) — DecisionTarget — audit a decision model through a System One endpoint.
- [`simpleaudit.targets.http`](reference/simpleaudit_targets_http.md) — HTTPAppTarget — audit an external application over HTTP (black-box).
- [`simpleaudit.targets.model`](reference/simpleaudit_targets_model.md) — ModelTarget — an LLM endpoint target backed by AnyLLM.

### Judges

- [`simpleaudit.judges.abstention`](reference/simpleaudit_judges_abstention.md) — Abstention judge configuration.
- [`simpleaudit.judges.binary_abstention`](reference/simpleaudit_judges_binary_abstention.md) — Binary abstention judge — language-agnostic.
- [`simpleaudit.judges.checklist`](reference/simpleaudit_judges_checklist.md) — Evidence-anchored checklist judge configuration.
- [`simpleaudit.judges.choice_match`](reference/simpleaudit_judges_choice_match.md) — Choice-match judge: grade a closed question by exact option match, in code.
- [`simpleaudit.judges.compose`](reference/simpleaudit_judges_compose.md) — Compose judges from criteria and an output format.
- [`simpleaudit.judges.default`](reference/simpleaudit_judges_default.md) — SimpleAudit's default judge: what grades a run when no judge config is named.
- [`simpleaudit.judges.fact_check`](reference/simpleaudit_judges_fact_check.md) — Fact-check judge — deterministic severity for declared facts.
- [`simpleaudit.judges.factuality`](reference/simpleaudit_judges_factuality.md) — Factuality judge configuration.
- [`simpleaudit.judges.groundedness`](reference/simpleaudit_judges_groundedness.md) — Groundedness judge for marked context.
- [`simpleaudit.judges.harm`](reference/simpleaudit_judges_harm.md) — Harm categorisation judge configuration.
- [`simpleaudit.judges.helpfulness`](reference/simpleaudit_judges_helpfulness.md) — Helpfulness judge configuration.
- [`simpleaudit.judges.helsedir_sexhealth_no`](reference/simpleaudit_judges_helsedir_sexhealth_no.md) — Helsedirektoratet sexual-health judge — generic variant.
- [`simpleaudit.judges.helsedir_sexhealth_no_rag`](reference/simpleaudit_judges_helsedir_sexhealth_no_rag.md) — Helsedirektoratet sexual-health judge — RAG variant.
- [`simpleaudit.judges.safety`](reference/simpleaudit_judges_safety.md) — Safety judge configuration.

### Context Analysis

- [`simpleaudit.context_attribution`](reference/simpleaudit_context_attribution.md) — Attributing an answer's claims to the documents it was given — mechanically.
- [`simpleaudit.context_derivations`](reference/simpleaudit_context_derivations.md) — Set-level properties derived from document marks.
- [`simpleaudit.context_findings`](reference/simpleaudit_context_findings.md) — Groundedness findings, derived from what the response did with each document.
- [`simpleaudit.context_marks`](reference/simpleaudit_context_marks.md) — Document marks for context-grounding scenarios.
- [`simpleaudit.cross_judge`](reference/simpleaudit_cross_judge.md) — Cross-judge stability analysis for SimpleAudit.
- [`simpleaudit.perturbations`](reference/simpleaudit_perturbations.md) — Deterministic style perturbations for stored transcripts.
- [`simpleaudit.reframing`](reference/simpleaudit_reframing.md) — Re-grade stored transcripts with the judge only, varying one thing at a time.
- [`simpleaudit.spans`](reference/simpleaudit_spans.md) — Locate quoted spans in stored transcripts.

### Scenarios

- [`simpleaudit.scenarios.arbeidstilsynet_arbeidstid`](reference/simpleaudit_scenarios_arbeidstilsynet_arbeidstid.md) — Arbeidstilsynet arbeidstid (working hours) scenario pack.
- [`simpleaudit.scenarios.bullshitbench_health`](reference/simpleaudit_scenarios_bullshitbench_health.md) — Bullshitbench health scenarios for SimpleAudit
- [`simpleaudit.scenarios.bullshitbench_v1_v2`](reference/simpleaudit_scenarios_bullshitbench_v1_v2.md) — BullshitBench Scenario Pack for SimpleAudit
- [`simpleaudit.scenarios.context_grounding`](reference/simpleaudit_scenarios_context_grounding.md) — Context-grounding scenario pack — marked retrieval context, three conflict classes.
- [`simpleaudit.scenarios.health`](reference/simpleaudit_scenarios_health.md) — Healthcare domain scenarios.
- [`simpleaudit.scenarios.healthbench_behaviours`](reference/simpleaudit_scenarios_healthbench_behaviours.md) — HealthBench behaviours scenario pack.
- [`simpleaudit.scenarios.healthbench_loader`](reference/simpleaudit_scenarios_healthbench_loader.md) — Load OpenAI's HealthBench as SimpleAudit scenarios at run time.
- [`simpleaudit.scenarios.hei_refusal`](reference/simpleaudit_scenarios_hei_refusal.md) — Hei refusal scenario pack.
- [`simpleaudit.scenarios.helfo`](reference/simpleaudit_scenarios_helfo.md) — Helfo (Helseøkonomiforvaltningen) scenario pack.
- [`simpleaudit.scenarios.helpmed`](reference/simpleaudit_scenarios_helpmed.md) — HelpMed domain scenarios.
- [`simpleaudit.scenarios.human_rights_education`](reference/simpleaudit_scenarios_human_rights_education.md) — Human rights / right to education scenario pack (international human rights law).
- [`simpleaudit.scenarios.human_rights_fair_trial`](reference/simpleaudit_scenarios_human_rights_fair_trial.md) — Human rights / fair trial and due process scenario pack (international human rights law).
- [`simpleaudit.scenarios.human_rights_water`](reference/simpleaudit_scenarios_human_rights_water.md) — Human rights / right to water scenario pack (international human rights law).
- [`simpleaudit.scenarios.lanekassen`](reference/simpleaudit_scenarios_lanekassen.md) — Lånekassen (Statens lånekasse for utdanning) scenario pack.
- [`simpleaudit.scenarios.nav_aap`](reference/simpleaudit_scenarios_nav_aap.md) — NAV AAP (Arbeidsavklaringspenger) scenario pack.
- [`simpleaudit.scenarios.nb_kryss_ordning`](reference/simpleaudit_scenarios_nb_kryss_ordning.md) — NB kryss-ordning (Nasjonalbiblioteket cross-scheme) scenario pack.
- [`simpleaudit.scenarios.nb_rag_kontekst`](reference/simpleaudit_scenarios_nb_rag_kontekst.md) — NB RAG-kontekst (Nasjonalbiblioteket, støyende hentekontekst) — ett scenario.
- [`simpleaudit.scenarios.rag`](reference/simpleaudit_scenarios_rag.md) — RAG-specific scenarios.
- [`simpleaudit.scenarios.safety`](reference/simpleaudit_scenarios_safety.md) — General AI safety scenarios.
- [`simpleaudit.scenarios.skatteetaten`](reference/simpleaudit_scenarios_skatteetaten.md) — Skatteetaten (Norwegian Tax Administration) scenario pack.
- [`simpleaudit.scenarios.skatteetaten_legitimasjon`](reference/simpleaudit_scenarios_skatteetaten_legitimasjon.md) — Skatteetaten legitimasjon (identification at in-person attendance) scenario pack.
- [`simpleaudit.scenarios.system_prompt`](reference/simpleaudit_scenarios_system_prompt.md) — System prompt testing scenarios.
- [`simpleaudit.scenarios.toll_reisegodskvote`](reference/simpleaudit_scenarios_toll_reisegodskvote.md) — Tolletaten reisegodskvote (traveller allowance) scenario pack.
- [`simpleaudit.scenarios.ung`](reference/simpleaudit_scenarios_ung.md) — ung scenario pack.
- [`simpleaudit.scenarios.vision_integrity`](reference/simpleaudit_scenarios_vision_integrity.md) — Vision integrity scenarios — does a vision model read a chart honestly?

### Tracing & Observability

- [`simpleaudit.tracing.auth`](reference/simpleaudit_tracing_auth.md) — OTLP receiver authentication — the "WHO is allowed to submit" layer.
- [`simpleaudit.tracing.context`](reference/simpleaudit_tracing_context.md) — Trace-context generation and correlation for SimpleAudit.
- [`simpleaudit.tracing.otlp`](reference/simpleaudit_tracing_otlp.md) — OTLP ingestion — accept OpenTelemetry trace exports and normalize to spans.
- [`simpleaudit.tracing.provider`](reference/simpleaudit_tracing_provider.md) — TraceProvider — pluggable source of traces for an audit.
- [`simpleaudit.tracing.selection`](reference/simpleaudit_tracing_selection.md) — Span selection — choose which spans to show the judge.
- [`simpleaudit.tracing.shared`](reference/simpleaudit_tracing_shared.md) — Shared OTLP receiver + per-audit trace session routing.
- [`simpleaudit.tracing.store`](reference/simpleaudit_tracing_store.md) — Span store — in-memory persistence and query for ingested trace spans.

### CLI & Utilities

- [`simpleaudit._event_loop`](reference/simpleaudit__event_loop.md) — Running SimpleAudit's coroutines from synchronous code.
- [`simpleaudit.cli`](reference/simpleaudit_cli.md) — CLI interface for SimpleAudit.
- [`simpleaudit.utils`](reference/simpleaudit_utils.md) — Utility functions for SimpleAudit.
