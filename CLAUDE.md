# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Pages site using Jekyll with Minima theme, hosted at tableero.github.io.

## Development Commands

```bash
# Start dev server (Docker)
docker compose up

# Rebuild after Gemfile changes
docker compose up --build

# Local alternative
bundle install
bundle exec jekyll serve
```

Site runs at http://localhost:4000

## Git Workflow

- **Main branch:** `main` (protected — no direct pushes)
- **Merge strategy:** Squash merge only, via PR
- **Signed commits required** on `main`
- **Branch auto-delete** after merge

### Branch Naming Convention

Use GitOps-style prefixes:

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feat/` | New content or functionality | `feat/trading-bot-post` |
| `fix/` | Bug fixes, typo corrections | `fix/broken-layout` |
| `chore/` | Config, deps, CI, tooling | `chore/update-gemfile` |
| `docs/` | Documentation updates | `docs/update-about-page` |
| `refactor/` | Restructuring without behavior change | `refactor/post-categories` |

Use lowercase, kebab-case. Keep names short and descriptive.

### Commit Messages

- Use imperative mood: "Add post" not "Added post"
- No Co-Authored-By tags
- Keep subject line under 72 characters

## Content Guidelines

- Posts go in `_posts/` with format `YYYY-MM-DD-title.md`
- Use Jekyll front matter: `layout`, `title`, `date`
- Language-agnostic writing — focus on concepts and architecture, not implementation details
