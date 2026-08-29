# SimpleAudit

A simple, extensible, local-first framework for multilingual auditing and red-teaming of AI systems via adversarial probing. Runs open models locally with no APIs required.

## Quick Start

```bash
pip install -U simpleaudit
```

```python
from simpleaudit import ModelAuditor

auditor = ModelAuditor(
    model="hf.co/NbAiLab/borealis-4b-instruct-preview-gguf:BF16",
    provider="ollama",
    judge_model="gpt-4o",
    judge_provider="openai",
)
results = auditor.run("safety", max_turns=5, max_workers=10)
results.summary()
```

## Guides

### Getting Started

- [Quickstart](guides/quickstart.md) — 
- [Installation](guides/installation.md) — 

### Core Concepts

- [Architecture](guides/architecture.md) — 
- [Key Ideas](guides/key-ideas.md) — 

## API Reference
