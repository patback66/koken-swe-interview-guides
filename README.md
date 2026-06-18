# Koken SWE Interview Guides

Structured software engineering interview prep guides for a **Sr. Software Engineer, FinTech** role (behavioral, system design, coding).

**Live site:** [patback66.github.io/koken-swe-interview-guides](https://patback66.github.io/koken-swe-interview-guides/)  
**Repo:** [github.com/patback66/koken-swe-interview-guides](https://github.com/patback66/koken-swe-interview-guides)

## Contents

| Guide | Description |
|-------|-------------|
| [Study Guide](docs/index.md) | High-level overview and quick reference |
| [Behavioral & Past Experience](docs/guides/01-behavioral-past-experience.md) | STAR stories, culture fit, common probes |
| [System Design](docs/guides/02-system-design.md) | Payments, reconciliation, AI-enhanced services |
| [Coding & Technical Depth](docs/guides/03-coding-technical-depth.md) | Idempotency, concurrency, LLM pipelines, testing |
| [References & Further Reading](docs/references.md) | Curated external resources by topic |

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

After the first deploy, enable GitHub Pages: **Settings → Pages → Source: Deploy from branch → `gh-pages` / `/ (root)`**.

## License

Personal study material for interview preparation.
