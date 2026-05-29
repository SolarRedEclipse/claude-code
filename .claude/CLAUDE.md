# CLAUDE.md

This file provides authoritative guidance to Claude Code when working in this repository. All instructions here take precedence over Claude's default behaviors.

---

## Project Overview

This repository contains two distinct types of work:

1. **Client-side web app** — Built with vanilla HTML, CSS, and JavaScript. Runs directly in the browser with no build tools or server-side dependencies.
2. **Python API endpoints for n8n** — Modal-deployed Python functions that serve as HTTP endpoints consumed by n8n workflows.

Treat these as separate contexts. Rules that apply to one do not necessarily apply to the other.

---

## Repository Structure

```
/
├── .claude/
│   ├── CLAUDE.md                  # This file — AI instructions
│   ├── rules/
│   │   ├── modal.md               # Modal patterns (loads for *.py only)
│   │   └── web.md                 # Web style + ads + design recreation (loads for *.html/css/js only)
│   └── settings.local.json        # Local permissions (do not commit)
├── modal_app.py                   # Main Modal deployment file
├── .env                           # Local secrets (gitignored — never commit)
├── .env.example                   # Safe template for .env
├── *.html / *.css / *.js          # Web app files
```

Keep the structure flat. Do not introduce new folders unless explicitly requested.

---

## Git Workflow

Always commit and push after every meaningful change:

```bash
git add <specific-file>
git commit -m "Imperative-mood message here"
git push
```

- Use the **imperative mood**: `Add`, `Fix`, `Remove`, `Update`, `Refactor`
- Be specific and concise — describe *what changed*
- Keep the subject line under 72 characters
- Never use `git add .` or `git add -A` — stage specific files by name
- Never amend published commits — create a new commit instead
- Never skip pre-commit hooks (`--no-verify`)
- Never force-push to `main`
- `main` is the production branch — always keep it stable

---

## Irreversible Actions

Before taking any action that cannot be undone, stop and warn the user:

> **Warning — Irreversible Action**
> I am about to `[describe the action]`. This cannot be undone. Do you want to proceed?

Examples requiring this warning:
- Deleting files, branches, or database records
- `git reset --hard`, `git push --force`, dropping commits
- `modal app stop <name>` or `modal secret delete <name>`
- Rotating or deleting a Bearer token (existing n8n workflows will immediately break)

---

## General Code Style

- Prefer **readability over cleverness**
- 2-space indentation for HTML/CSS/JS; 4-space for Python
- End every file with a **single trailing newline**
- Remove all debug `print` / `console.log` before committing
- No comments that restate what the code does — only comment on *why* when genuinely non-obvious

---

## What Claude Should Never Do

- **Do not introduce a build system** (Webpack, Vite, Parcel, etc.) for the web app
- **Do not add npm / package.json** for the web app unless explicitly asked
- **Do not add CSS or JS frameworks** to the web app — Tailwind via CDN is permitted for design recreation only
- **Do not deploy a Modal endpoint without Bearer token auth** — no exceptions
- **Do not use raw `dict` for Modal endpoint inputs** — always use Pydantic models
- **Do not take irreversible actions without warning**

---

## Gotchas

- `.DS_Store` — never commit
- `.claude/settings.local.json` — never commit
- `.env` — never commit; use `.env.example` as the safe template
- Rotating `api-auth-token` immediately breaks all n8n workflows — treat as irreversible

---

## Tone & Collaboration

- Be concise — avoid restating what was just done
- Open-ended questions: give a recommendation + key tradeoff in 2–3 sentences, then wait for approval
- Ambiguous tasks: ask one clarifying question rather than assuming
- Prefer editing existing files over creating new ones
- Do not add features or abstractions beyond what was explicitly requested

---

## Scoped Rules (auto-loaded by file type)

@rules/modal.md
@rules/web.md
