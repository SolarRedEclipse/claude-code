---
paths:
  - "*.py"
  - "modal_app.py"
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
openssl rand -hex 32
modal secret create my-secret-name API_KEY=xxx ANOTHER_KEY=yyy
modal secret list
```

### Reconfiguring Modal Auth

```bash
modal token set --token-id <ID> --token-secret <SECRET>
```

---

## Modal: Endpoint Template

All endpoints must implement Bearer token authentication:

```python
import modal
from fastapi import Header, HTTPException
from pydantic import BaseModel

app = modal.App("my-app-name")
image = modal.Image.debian_slim().pip_install("fastapi")


class MyRequest(BaseModel):
    field_one: str
    field_two: int = 0


class MyResponse(BaseModel):
    result: str


@app.function(
    image=image,
    secrets=[modal.Secret.from_name("api-auth-token")],
    timeout=120,
)
@modal.fastapi_endpoint(method="POST")
def my_endpoint(data: MyRequest, authorization: str = Header(None)) -> dict:
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
modal deploy modal_app.py
modal run modal_app.py::function_name
modal app list
modal app stop app-name        # ⚠️ IRREVERSIBLE — warn user before running
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

**2. cURL command**
```bash
curl -X POST "https://your-profile--app-name-function-name.modal.run" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{"your": "payload"}'
```

**3. n8n HTTP Request node config**
- Method: `POST`
- URL: the endpoint URL
- Authentication: Header Auth → Name: `Authorization`, Value: `Bearer YOUR_TOKEN_HERE`
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
