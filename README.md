# vfa-docs

Public documentation site for **VFA — Visual & Functional Web Auditor**, an autonomous
black-box web QA agent exposed as an MCP server over stdio.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and published to
GitHub Pages.

## Local development

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve          # http://127.0.0.1:8000 with live reload
```

Build with the same strictness used in CI (warnings become errors):

```bash
mkdocs build --strict
```

## Structure

| Path | Purpose |
| --- | --- |
| `mkdocs.yml` | Site configuration: theme, palette, extensions and navigation. |
| `docs/index.md` | Project overview, hexagonal architecture diagram (Mermaid) and security model. |
| `docs/quickstart.md` | License activation, first run and verification. |
| `docs/clients.md` | MCP client wiring: Zoo/Roo Code, Claude Desktop, Cursor, troubleshooting. |
| `docs/configuration.md` | Environment variable reference and per-provider `env` presets. |
| `.github/workflows/deploy.yml` | CI: build on every push to `main` and publish via `mkdocs gh-deploy --force`. |

## Deployment

Pushes to `main` trigger `.github/workflows/deploy.yml`, which builds the site and pushes the
result to the `gh-pages` branch. For one-off manual publishing:

```bash
mkdocs gh-deploy --force
```