# Interview Guides

Position-specific interview prep. Pick a guide based on the role you're preparing for.

---

## Available guides

| Guide | Role | Focus areas | Status |
|-------|------|-------------|--------|
| **[Sr. SWE, FinTech](sr-swe-fintech/index.md)** | Sr. Software Engineer, FinTech | Payments, reconciliation, billing, AI-driven workflows, global transaction scale | Available |

## Sr. SWE, FinTech — quick links

| Section | Link |
|---------|------|
| Overview | [Study guide](sr-swe-fintech/index.md) |
| Cheat Sheet | [cheat-sheet.md](sr-swe-fintech/cheat-sheet.md) |
| Visual Infographic | [infographic.md](sr-swe-fintech/infographic.md) |
| 48-Hour Cram Plan | [cram-plan.md](sr-swe-fintech/cram-plan.md) |
| Behavioral & Past Experience | [01-behavioral-past-experience.md](sr-swe-fintech/01-behavioral-past-experience.md) |
| System Design | [02-system-design.md](sr-swe-fintech/02-system-design.md) |
| Coding & Technical Depth | [03-coding-technical-depth.md](sr-swe-fintech/03-coding-technical-depth.md) |
| References | [references.md](sr-swe-fintech/references.md) |

### Interview weighting

| Section | Weight |
|---------|--------|
| System Design | ~50% |
| Coding & Technical Depth | ~30% |
| Behavioral & Past Experience | ~20% |

---

## Coming soon

_Additional position guides will appear here as they are created._

| Guide | Role | Status |
|-------|------|--------|
| _Example: Staff Engineer, Platform_ | _TBD_ | Planned |

---

## Guide structure template

When adding a new guide, create a folder under `docs/guides/<slug>/` with:

```
docs/guides/<slug>/
├── index.md              # Overview, format, section weighting
├── cram-plan.md          # Optional time-boxed study schedule
├── references.md         # Curated external links
├── 01-behavioral-....md  # Deep-dive sections as needed
├── 02-system-design.md
└── 03-coding-....md
```

Then register the guide in:

1. This page (`docs/guides/index.md`)
2. `mkdocs.yml` navigation

[← Home](../index.md)
