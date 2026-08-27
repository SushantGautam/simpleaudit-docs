# SimpleAudit Documentation

Documentation site for [SimpleAudit](https://github.com/SushantGautam/simpleaudit) — a lightweight AI safety auditing framework.

**Live site:** https://sushantgautam.github.io/simpleaudit-docs/

## What's here

| Path | Description |
|------|-------------|
| `site/` | Built static site (deployed to GitHub Pages) |
| `site_src/` | MkDocs source: guides, API reference, images, plan cache |
| `generate_docs.py` | Two-layer docs generator (Griffe + LLM) |
| `upload_to_kb.py` | Syncs simpleaudit source to the Open WebUI knowledge base |
| `mkdocs.yml` | Generated MkDocs Material config |

## How the docs are generated

The pipeline has two layers:

1. **Deterministic (no LLM):** Griffe scans the `simpleaudit/` package and renders a complete API reference (classes, functions, signatures, docstrings) into `site_src/reference/`. The nav, index page, and `mkdocs.yml` are also generated deterministically.

2. **LLM narrative:** Guide pages (getting started, architecture, CLI usage, judges, scenarios, etc.) are written by an LLM (Open WebUI backend) with source code context. Pages are only regenerated when their source files change (SHA-256 hash tracking in `site_src/.plan.json`).

## Regenerating the docs

```bash
# Prerequisites: a checkout of the simpleaudit repo
export SIMPLEAUDIT_ROOT=/path/to/simpleaudit

# Install deps
pip install mkdocs mkdocs-material griffe requests

# Incremental generate (only changed pages)
python3 generate_docs.py

# Full re-plan + regenerate everything
python3 generate_docs.py --force

# Deterministic layer only (no LLM calls)
python3 generate_docs.py --no-llm

# Generate + build the static site
python3 generate_docs.py --build
```

## CI / GitHub Pages

The `.github/workflows/docs.yml` workflow:
1. Checks out this repo and the `SushantGautam/simpleaudit` repo
2. Runs the generator (incremental, with cached plan)
3. Builds the site with `mkdocs build --strict`
4. Deploys `site/` to GitHub Pages

The workflow requires the `OWUI_API_KEY` secret (and optionally `OWUI_BASE`, `OWUI_KB_ID`, `OWUI_MODEL`) for LLM page generation. If the secret is missing, it falls back to `--no-llm` (deterministic layer only).

## Local preview

```bash
python3 -m http.server 8765 -d site/
# → http://localhost:8765
```
