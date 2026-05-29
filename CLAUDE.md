# CLAUDE.md

This file provides authoritative guidance to Claude Code when working in this repository. All instructions here take precedence over Claude's default behaviors. Read this file in full before taking any action.

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
├── CLAUDE.md                  # This file — AI instructions
├── .claude/
│   └── settings.local.json    # Local Claude Code permissions (do not commit)
├── modal_app.py               # Main Modal deployment file
├── .env                       # Local secrets (gitignored — never commit)
├── .env.example               # Safe template for .env
├── *.html                     # Web app pages
├── *.css                      # Stylesheets
└── *.js                       # JavaScript modules
```

Keep the structure flat. Do not introduce new folders unless explicitly requested.

---

## Git Workflow

Always commit and push after every meaningful change. Follow this exact sequence:

```bash
git add <specific-file>
git commit -m "Imperative-mood message here"
git push
```

### Commit Message Rules

- Use the **imperative mood**: `Add`, `Fix`, `Remove`, `Update`, `Refactor` — not `Added`, `Fixes`, `Removing`
- Be specific and concise: describe *what changed*, not *why*
- Keep the subject line under 72 characters
- Do **not** use generic messages like `Update files` or `Fix stuff`

**Good examples:**
```
Add keyboard shortcut for backspace
Fix division by zero display error
Deploy email reply endpoint to Modal
Add Pydantic validation to scraper endpoint
```

### Branching

- `main` is the production branch — always keep it stable and deployable
- For experimental features, create a short-lived branch: `git checkout -b feature/description`
- Merge back to `main` via a clean commit once the feature is complete

---

## Irreversible Actions

Before taking any action that cannot be undone, **stop and explicitly warn the user**. Use this format:

> **Warning — Irreversible Action**
> I am about to `[describe the action]`. This cannot be undone. Do you want to proceed?

Do not proceed until the user confirms. Examples of irreversible actions that require this warning:

- Deleting files, branches, or database records
- `git reset --hard`, `git push --force`, dropping commits
- `modal app stop <name>` or `modal secret delete <name>` (stops/removes live infrastructure)
- Overwriting a deployed endpoint with a breaking change
- Rotating or deleting a Bearer token (existing n8n workflows will immediately break)
- Removing a Modal secret that active endpoints depend on

If the action is reversible (e.g., a file edit that can be reverted, a local test run), proceed without asking.

---

## Code Style & Conventions

### General

- Prefer **readability over cleverness** — code is read far more than it is written
- Use **2-space indentation** consistently across all file types
- End every file with a **single trailing newline**
- Remove all **debug print statements and console.log calls** before committing

### HTML / CSS / JavaScript (Web App Context)

- Use semantic HTML elements: `<main>`, `<section>`, `<button>` — avoid `<div>` soup
- Every interactive element must have an accessible `aria-label` or visible label
- Use **CSS custom properties** (`--var-name`) for colors, spacing, and font sizes
- Use `const` by default; `let` only when reassignment is needed; never `var`
- Prefer `addEventListener` over inline `onclick` HTML attributes
- **No external libraries or CDN imports** unless explicitly approved

### Python (Modal Endpoints Context)

- Follow PEP 8 — 4-space indentation, snake_case for variables and functions
- Use **Pydantic models** for all request and response bodies — never use raw `dict` for input
- Always return a consistent error shape: `{"error": "<message>"}` with HTTP 200 for business logic failures; reserve 4xx/5xx for auth and infrastructure failures
- Keep endpoint functions focused — extract heavy logic into helper functions
- Use `os.environ.get("KEY")` with a fallback check rather than `os.environ["KEY"]` where a missing key should produce a clear error message

---

## Modal: Setup & Authentication

### Existing Secrets

Already created in Modal — reference with `modal.Secret.from_name()`:

| Secret Name | Environment Variable |
|---|---|
| `anthropic-api-key` | `ANTHROPIC_API_KEY` |
| `api-auth-token` | `API_AUTH_TOKEN` |

### Creating New Secrets

```bash
# Generate a secure random token
openssl rand -hex 32

# Create a Modal secret
modal secret create my-secret-name API_KEY=xxx ANOTHER_KEY=yyy

# List existing secrets
modal secret list
```

### Reconfiguring Modal Auth (if needed)

```bash
modal token set --token-id <ID> --token-secret <SECRET>
```

Token is saved to `~/.modal.toml`.

---

## Modal: Endpoint Template

All endpoints must implement Bearer token authentication. Use this as the base template:

```python
import modal
from fastapi import Header, HTTPException
from pydantic import BaseModel

app = modal.App("my-app-name")
image = modal.Image.debian_slim().pip_install("fastapi")  # add deps here


class MyRequest(BaseModel):
    field_one: str
    field_two: int = 0  # optional with default


class MyResponse(BaseModel):
    result: str


@app.function(
    image=image,
    secrets=[
        modal.Secret.from_name("api-auth-token"),
    ],
    timeout=120,
)
@modal.fastapi_endpoint(method="POST")
def my_endpoint(data: MyRequest, authorization: str = Header(None)) -> dict:
    """Describe what this endpoint does."""
    import os

    expected_token = os.environ.get("API_AUTH_TOKEN")
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid authorization header")
    if authorization.replace("Bearer ", "") != expected_token:
        raise HTTPException(status_code=403, detail="Invalid authentication token")

    try:
        result = process(data)
        return {"result": result}
    except Exception as e:
        return {"error": str(e)}
```

**Security:** Every endpoint must have Bearer token authentication. Never deploy without it.

---

## Modal: Common Patterns

### AI / LLM Endpoint (Claude)

```python
import modal
from fastapi import Header, HTTPException
from pydantic import BaseModel

app = modal.App("ai-task")
image = modal.Image.debian_slim().pip_install("anthropic", "fastapi")


class PromptRequest(BaseModel):
    prompt: str
    system: str = "You are a helpful assistant."


@app.function(
    image=image,
    secrets=[
        modal.Secret.from_name("anthropic-api-key"),
        modal.Secret.from_name("api-auth-token"),
    ],
    timeout=120,
)
@modal.fastapi_endpoint(method="POST")
def process(data: PromptRequest, authorization: str = Header(None)) -> dict:
    import anthropic, os

    expected_token = os.environ.get("API_AUTH_TOKEN")
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid authorization header")
    if authorization.replace("Bearer ", "") != expected_token:
        raise HTTPException(status_code=403, detail="Invalid authentication token")

    try:
        client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
        message = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            system=data.system,
            messages=[{"role": "user", "content": data.prompt}],
        )
        return {"response": message.content[0].text}
    except Exception as e:
        return {"error": str(e)}
```

### Web Scraping Endpoint

```python
import modal
from fastapi import Header, HTTPException
from pydantic import BaseModel

app = modal.App("scraper")
image = modal.Image.debian_slim().pip_install("httpx", "beautifulsoup4", "fastapi")


class ScrapeRequest(BaseModel):
    url: str
    max_chars: int = 1000


@app.function(
    image=image,
    secrets=[modal.Secret.from_name("api-auth-token")],
    timeout=60,
)
@modal.fastapi_endpoint(method="POST")
def scrape(data: ScrapeRequest, authorization: str = Header(None)) -> dict:
    import httpx, os
    from bs4 import BeautifulSoup

    expected_token = os.environ.get("API_AUTH_TOKEN")
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid authorization header")
    if authorization.replace("Bearer ", "") != expected_token:
        raise HTTPException(status_code=403, detail="Invalid authentication token")

    try:
        response = httpx.get(data.url, timeout=30)
        soup = BeautifulSoup(response.text, "html.parser")
        return {"title": soup.title.string, "text": soup.get_text()[:data.max_chars]}
    except Exception as e:
        return {"error": str(e)}
```

### Data Processing Endpoint

```python
import modal
from fastapi import Header, HTTPException
from pydantic import BaseModel
from typing import List

app = modal.App("processor")
image = modal.Image.debian_slim().pip_install("pandas", "fastapi")


class ProcessRequest(BaseModel):
    records: List[dict]


@app.function(
    image=image,
    secrets=[modal.Secret.from_name("api-auth-token")],
    timeout=300,
)
@modal.fastapi_endpoint(method="POST")
def process_data(data: ProcessRequest, authorization: str = Header(None)) -> dict:
    import pandas as pd, os

    expected_token = os.environ.get("API_AUTH_TOKEN")
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid authorization header")
    if authorization.replace("Bearer ", "") != expected_token:
        raise HTTPException(status_code=403, detail="Invalid authentication token")

    try:
        df = pd.DataFrame(data.records)
        return {"stats": df.describe().to_dict()}
    except Exception as e:
        return {"error": str(e)}
```

---

## Modal: Deploy & Test

```bash
# Deploy to Modal
modal deploy modal_app.py

# Test a function locally without deploying
modal run modal_app.py::function_name

# List deployed apps
modal app list

# Stop a deployed app
modal app stop app-name        # ⚠️ IRREVERSIBLE — warn user before running

# List secrets
modal secret list
```

Deployed endpoint URL format:
```
https://<modal-profile>--<app-name>-<function-name>.modal.run
```

---

## Modal: Returning Results to the User

After every successful deployment, give the user all three of these:

**1. Endpoint URL**
```
https://your-profile--app-name-function-name.modal.run
```

**2. cURL command** (ready to paste into terminal or n8n HTTP Request node)
```bash
curl -X POST "https://your-profile--app-name-function-name.modal.run" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{"your": "payload"}'
```

**3. n8n HTTP Request node config**
- Method: `POST`
- URL: the endpoint URL
- Authentication: Header Auth
  - Name: `Authorization`
  - Value: `Bearer YOUR_TOKEN_HERE`
- Body: JSON with the required fields

---

## Modal: Checklist for Each New Endpoint

- [ ] Define Pydantic request and response models
- [ ] Add required `pip_install` packages to the image
- [ ] Add required secrets to `@app.function`
- [ ] Implement Bearer token authentication block
- [ ] Wrap logic in `try/except` returning `{"error": str(e)}` on failure
- [ ] Test locally: `modal run modal_app.py::function_name`
- [ ] Deploy: `modal deploy modal_app.py`
- [ ] Return to user: endpoint URL + cURL + n8n config

---

## Website Design Recreation

When the user provides a reference image (screenshot) and asks to replicate it:

### Workflow

1. **Generate** a single HTML file using Tailwind CSS via CDN. Include all content inline — no external files unless requested.
2. **Screenshot** the rendered page using Puppeteer or `open` + browser screenshot.
3. **Compare** the screenshot against the reference image. Check for mismatches in:
   - Spacing and padding
   - Font sizes, weights, and line heights
   - Colors (exact hex values)
   - Alignment and positioning
   - Border radii, shadows, and effects
   - Image/icon sizing and placement
4. **Fix** every mismatch found and re-screenshot.
5. **Repeat** until no visible differences remain — always do at least 2 comparison rounds.

### Technical Defaults

- Use Tailwind CSS via CDN: `<script src="https://cdn.tailwindcss.com"></script>`
- Use placeholder images from `https://placehold.co/` when source images are not provided
- Single `index.html` file unless the user requests otherwise

### Rules

- Do not add features, sections, or content not in the reference image
- Match the reference exactly — do not "improve" or "enhance" the design
- If the user provides CSS classes or style tokens, use them verbatim
- When reporting mismatches be specific: "heading is 32px but reference shows ~24px"
- Tailwind via CDN is allowed for recreation tasks — the "no frameworks" rule applies to the web app project only

---

## What Claude Should Never Do

- **Do not introduce a build system** (Webpack, Vite, Parcel, etc.) for the web app unless explicitly asked
- **Do not add npm / package.json** for the web app unless explicitly asked
- **Do not add CSS or JS frameworks** (Bootstrap, Tailwind, React, Vue, etc.) to the web app project unless explicitly asked — Tailwind via CDN is permitted for design recreation tasks only
- **Do not deploy a Modal endpoint without Bearer token auth** — no exceptions
- **Do not use raw `dict` for Modal endpoint inputs** — always use Pydantic models
- **Do not add comments** that merely restate what the code does — only comment on *why* when genuinely non-obvious
- **Do not use `git add .` or `git add -A`** — always stage specific files by name
- **Do not amend published commits** — create a new commit instead
- **Do not skip pre-commit hooks** (`--no-verify`)
- **Do not push to `main` with `--force`**
- **Do not take irreversible actions without warning** — see the Irreversible Actions section

---

## Gotchas & Project-Specific Notes

- `calculator.html` was the original web app entry point — check git log if referencing prior state
- `.DS_Store` files are untracked macOS metadata — never commit them
- `.claude/settings.local.json` contains local permissions — never commit it
- `.env` contains local secrets — never commit it; use `.env.example` as the safe template
- Rotating `api-auth-token` will immediately break all n8n workflows using that token — treat as an irreversible action and warn the user

---

## Tone & Collaboration

- Be **concise** in responses — avoid restating what was just done
- When asked an open-ended question, give a **recommendation with the key tradeoff** in 2–3 sentences, then wait for approval before implementing
- When a task is ambiguous, **ask one clarifying question** rather than making assumptions
- Prefer **editing existing files** over creating new ones
- Do not add features, abstractions, or refactors beyond what was explicitly requested
