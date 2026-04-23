# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

MkDocs Material documentation site (French) — a personal MSP knowledge base covering Intune, Entra ID, Microsoft 365, Datto RMM, and Microsoft Graph. Published automatically to GitHub Pages on every push to `main`.

## Commands

```bash
# Install dependencies (one-time)
pip install mkdocs-material pymdown-extensions

# Local dev server with live reload
mkdocs serve

# Build static site into /site (gitignored)
mkdocs build

# Deploy manually to GitHub Pages (normally done by CI)
mkdocs gh-deploy --force
```

## Architecture

- `mkdocs.yml` — site config, theme, and all enabled Markdown extensions (single source of truth)
- `docs/` — all content pages as Markdown files; no `nav:` section, so MkDocs auto-generates navigation from the folder structure
- `.github/workflows/deploy.yml` — on push to `main`: installs deps, runs `mkdocs gh-deploy --force`

## Content conventions

**Every page must have frontmatter:**
```yaml
---
title: Short title for nav and browser tab
description: One-line description for SEO and search
---
```

**Language:** All content is in French.

**Enabled extensions to use actively:**

| Extension | Syntax |
|---|---|
| Admonitions | `!!! tip/warning/danger/note/info/important "Title"` |
| Collapsible admonitions | `??? tip` (closed by default), `???+ tip` (open) |
| Tabbed content | `=== "Tab label"` blocks (requires blank line before content) |
| Mermaid diagrams | ` ```mermaid ` fenced block |
| Task lists | `- [x]` / `- [ ]` |
| Code with line numbers | ` ```powershell linenums="1" ` |
| Code annotations | `# (1)` in code + `1. Explanation` below |

**Page structure pattern** (from existing pages):
1. H1 title
2. One-paragraph context/definition
3. H2 sections with tables, code blocks, admonitions
4. "À lire ensuite" H2 at the bottom with relative links to related pages

**Linking between pages:** Use relative paths without the `.md` extension when linking within `docs/intune/` — e.g., `[Autopilot](autopilot.md)`.

**Placeholder links** for pages not yet written: keep them as `[Title](filename.md) *(à venir)*` in the "À lire ensuite" section.
