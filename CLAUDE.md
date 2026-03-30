# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Pages personal website using Jekyll with Minima theme.

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

- Main branch: `main`
- Deploy: GitHub Pages builds automatically from `main`
