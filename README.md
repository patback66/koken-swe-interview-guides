# Koken SWE Interview Guides

Interview prep guides for software engineering roles — organized by position.

**Live site:** [patback66.github.io/koken-swe-interview-guides](https://patback66.github.io/koken-swe-interview-guides/)  
**Repo:** [github.com/patback66/koken-swe-interview-guides](https://github.com/patback66/koken-swe-interview-guides)

## Guides

| Guide | Description |
|-------|-------------|
| [All guides](docs/guides/index.md) | Catalog of position-specific interview guides |
| [Sr. SWE, FinTech](docs/guides/sr-swe-fintech/index.md) | Payments, reconciliation, billing, AI workflows |

### Sr. SWE, FinTech — sections

| Section | Link |
|---------|------|
| Study Guide | [docs/guides/sr-swe-fintech/index.md](docs/guides/sr-swe-fintech/index.md) |
| Cheat Sheet | [docs/guides/sr-swe-fintech/cheat-sheet.md](docs/guides/sr-swe-fintech/cheat-sheet.md) |
| Visual Infographic | [docs/guides/sr-swe-fintech/infographic.md](docs/guides/sr-swe-fintech/infographic.md) |
| 48-Hour Cram Plan | [docs/guides/sr-swe-fintech/cram-plan.md](docs/guides/sr-swe-fintech/cram-plan.md) |
| Behavioral | [docs/guides/sr-swe-fintech/01-behavioral-past-experience.md](docs/guides/sr-swe-fintech/01-behavioral-past-experience.md) |
| System Design | [docs/guides/sr-swe-fintech/02-system-design.md](docs/guides/sr-swe-fintech/02-system-design.md) |
| Coding | [docs/guides/sr-swe-fintech/03-coding-technical-depth.md](docs/guides/sr-swe-fintech/03-coding-technical-depth.md) |
| References | [docs/guides/sr-swe-fintech/references.md](docs/guides/sr-swe-fintech/references.md) |

## Local development

Preview the site locally with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) after running `mkdocs serve`.

## Deployment

Pushes to `main` trigger [.github/workflows/deploy.yml](.github/workflows/deploy.yml), which builds and deploys to the `gh-pages` branch via `mkdocs gh-deploy`.

## Adding a new guide

1. Create `docs/guides/<slug>/` with `index.md` and section files
2. Register in [docs/guides/index.md](docs/guides/index.md)
3. Add to `mkdocs.yml` navigation under **Guides**

## License

Personal study material for interview preparation.
