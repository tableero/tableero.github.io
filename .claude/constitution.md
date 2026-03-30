# Claude Constitution — tableero.github.io

## Identity

This is a public-facing site for the Tableero organization. Content should be clear, professional, and accessible to a broad technical audience.

## Principles

1. **No direct commits to `main`** — All changes go through a feature branch and PR. Always.
2. **Squash merge** — One clean commit per PR on main. Keep history readable.
3. **Signed commits** — All commits merged to main must be signed.
4. **Language-agnostic content** — Write about concepts, architecture, and decisions. Avoid coupling explanations to specific programming languages unless the post is explicitly about one.
5. **No AI attribution in commits** — Do not add Co-Authored-By or similar tags to commit messages.
6. **Branch naming** — Follow GitOps convention: `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`. Lowercase, kebab-case.
7. **Minimal changes** — Don't refactor, add comments, or "improve" files beyond the scope of the current task.
8. **Ship small** — One post, one fix, one change per branch/PR. Keep PRs reviewable.

## Workflow

1. Create a branch from `main` using the naming convention
2. Make changes, commit with clear imperative messages
3. Push and open a PR
4. Squash merge after review

## Content Standards

- Posts are written for technical readers who may not share the same stack
- Explain the "why" and "what", not just the "how"
- Diagrams and structure over walls of text
- No emojis unless explicitly requested
