# GitHub Setup

Repo: **[patback66/koken-swe-interview-guides](https://github.com/patback66/koken-swe-interview-guides)**

| | |
|---|---|
| **Clone URL** | `https://github.com/patback66/koken-swe-interview-guides.git` |
| **GitHub Pages URL** | `https://patback66.github.io/koken-swe-interview-guides/` |

## Create the remote repo

### Option A — GitHub CLI

```bash
gh repo create patback66/koken-swe-interview-guides \
  --public \
  --source=. \
  --remote=origin \
  --description "SWE interview study guides"
```

### Option B — GitHub web UI

1. Go to [github.com/new](https://github.com/new) while signed in as [patback66](https://github.com/patback66)
2. Owner: **patback66**
3. Repository name: **koken-swe-interview-guides**
4. Description: `SWE interview study guides`
5. Public (recommended for GitHub Pages)
6. Do **not** initialize with README, .gitignore, or license
7. Create repository

## Push from local

From the project root:

```bash
git init
git add .
git commit -m "Initial commit: interview study guide and deep-dive guides"
git branch -M main
git remote add origin https://github.com/patback66/koken-swe-interview-guides.git
git push -u origin main
```

If `origin` already exists with a different URL:

```bash
git remote set-url origin https://github.com/patback66/koken-swe-interview-guides.git
git push -u origin main
```

## Enable GitHub Pages (after site generator is added)

1. Repo → **Settings** → **Pages**
2. **Source:** Deploy from a branch (or GitHub Actions, depending on setup)
3. **Branch:** `main` / `/` or `/docs` — depends on the generator we choose next
4. Save; site will be live at `https://patback66.github.io/koken-swe-interview-guides/`

## Optional follow-ups

- [ ] Add site generator (MkDocs Material, Jekyll, or Docsify)
- [ ] Add GitHub Actions workflow for automatic deploy on push to `main`
- [ ] Custom domain (optional)
- [ ] Rename local folder from `swe-interview` to `koken-swe-interview-guides` for consistency

## Repo layout

```
.
├── README.md
├── study-guide.md
├── guides/
│   ├── 01-behavioral-past-experience.md
│   ├── 02-system-design.md
│   └── 03-coding-technical-depth.md
└── docs/
    └── GITHUB_SETUP.md          # this file
```
