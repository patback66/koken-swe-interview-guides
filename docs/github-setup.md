# GitHub Setup

Repo: **[patback66/koken-swe-interview-guides](https://github.com/patback66/koken-swe-interview-guides)**

| | |
|---|---|
| **Clone URL** | `https://github.com/patback66/koken-swe-interview-guides.git` |
| **GitHub Pages URL** | `https://patback66.github.io/koken-swe-interview-guides/` |

## GitHub Pages

The site is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed automatically on push to `main`.

1. Workflow: [deploy.yml](https://github.com/patback66/koken-swe-interview-guides/blob/main/.github/workflows/deploy.yml)
2. After the first successful run, go to **Settings → Pages**
3. Source: **Deploy from a branch**
4. Branch: **`gh-pages`** / **`/ (root)`**

## Local preview

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

## Repo layout

```
.
├── mkdocs.yml
├── requirements.txt
├── README.md
├── docs/
│   ├── index.md                 # Study guide (site home)
│   ├── github-setup.md          # This file
│   └── guides/
│       ├── 01-behavioral-past-experience.md
│       ├── 02-system-design.md
│       └── 03-coding-technical-depth.md
└── .github/workflows/deploy.yml
```
