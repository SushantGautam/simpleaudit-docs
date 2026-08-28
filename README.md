# SimpleAudit Documentation

Documentation site for [SimpleAudit](https://github.com/SushantGautam/simpleaudit) — a lightweight AI safety auditing framework.

**Live site:** https://sushantgautam.github.io/simpleaudit-docs/

## What's here

| Path | Description |
|------|-------------|
| `.github/workflows/repoquill.yml` | CI workflow: generate + build + deploy |
| `.github/workflows/repoquill.config.yml` | RepoQuill config (LLM, site, nav, guides) |

That's it. Everything else is generated at build time.

## How it works

[RepoQuill](https://github.com/SushantGautam/RepoQuill) is a standalone docs generator. The CI workflow:

1. Checks out this repo, the RepoQuill repo, and the simpleaudit source
2. Installs RepoQuill from source
3. Runs `repoquill generate --config .github/workflows/repoquill.config.yml --build`
4. Deploys the built `site/` to GitHub Pages (or a branch folder)

RepoQuill produces two layers:

- **API reference (deterministic):** Griffe scans the `simpleaudit/` package and renders classes, functions, signatures, and docstrings.
- **Guide pages (LLM):** Narrative pages (getting started, architecture, judges, etc.) written by an LLM with source code context. Pages are only regenerated when their source files change.

## Published artifacts

The built site includes:

- HTML pages (guides + API reference)
- `llms.txt` — concise AI-agent-friendly index
- `llms-full.txt` — full docs concatenated for AI agents
- `SKILL.md` — agent skill for generating, maintaining, and validating the docs

## Deploy targets

The workflow supports two deploy modes (set via `workflow_dispatch` input or `DEPLOY_TARGET` repo variable):

- **`gh-pages`** (default) — publishes to a dedicated `gh-pages` branch
- **`branch-folder[:<branch>[:<folder>]]`** — commits the built site into a folder on a branch (e.g. `branch-folder:main:docs`)

## Local development

```bash
# Install RepoQuill
pip install git+https://github.com/SushantGautam/RepoQuill

# Generate + build (point SOURCE_ROOT at a simpleaudit checkout)
SOURCE_ROOT=/path/to/simpleaudit \
OPENAI_API_KEY=your-key \
repoquill generate \
  --config .github/workflows/repoquill.config.yml \
  --build

# Preview
python3 -m http.server 8765 -d site/
# → http://localhost:8765
```
