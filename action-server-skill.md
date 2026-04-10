---
name: sema4ai-action-server
description: >
  Deep expertise on Sema4.ai Action Server: architecture, package creation, conda environments,
  @action decorator patterns, static file serving, --expose tunnel for rapid testing, MCP integration,
  LangChain/LangGraph connection, and production deployment. Use this skill whenever the user mentions
  Action Server, action packages, @action decorators, sema4ai-actions, action-server CLI, --expose flag,
  MultiServerMCPClient, conda.yaml for actions, package.yaml spec-version v2, or is asking how to
  expose tools to LLM agents via Sema4.ai. Also trigger when the user asks how to set up, configure,
  debug, or deploy any Sema4.ai action package — even if they just say "action server" casually.
---

# Sema4.ai Action Server — Complete Reference

## Table of Contents

1. [What is Action Server?](#1-what-is-action-server)
2. [Core Concepts](#2-core-concepts)
3. [Package Structure](#3-package-structure)
4. [Writing Actions (@action)](#4-writing-actions-action)
5. [package.yaml Reference](#5-packageyaml-reference)
6. [conda.yaml Reference](#6-condayaml-reference)
7. [CLI Reference](#7-cli-reference)
8. [--expose Flag: Rapid Testing](#8---expose-flag-rapid-testing)
9. [Static File Serving](#9-static-file-serving)
10. [MCP Integration](#10-mcp-integration)
11. [LangChain / LangGraph Connection](#11-langchain--langgraph-connection)
12. [Environment Variables](#12-environment-variables)
13. [Production Deployment](#13-production-deployment)
14. [Debugging Patterns](#14-debugging-patterns)
15. [Full Working Example](#15-full-working-example)

---

## 1. What is Action Server?

**Sema4.ai Action Server** is a lightweight Python-native server that:
- Wraps plain Python functions decorated with `@action` as HTTP endpoints
- Auto-generates an OpenAPI spec from those functions
- Exposes those endpoints as **MCP (Model Context Protocol) tools** for LLM agents
- Manages isolated Conda environments per package
- Optionally creates a public HTTPS tunnel with `--expose`

**Key insight for LLM agents**: Every `@action` function becomes both:
- A REST endpoint: `POST /api/actions/<package>/<action-name>/run`
- An MCP tool: available at `/api/mcp/`

---

## 2. Core Concepts

| Concept | Description |
|---|---|
| **Action Package** | A folder with `package.yaml` + `conda.yaml` + `actions/*.py` |
| **@action** | Decorator that registers a Python function as an agent tool |
| **package.yaml** | Package manifest (spec-version: v2): name, version, deps, action files |
| **conda.yaml** | Conda environment spec; Action Server creates an isolated env automatically |
| **MCP endpoint** | `/api/mcp/` — connects LLM agents (LangChain, Claude, etc.) |
| **--expose** | Creates a public `*.sema4ai.link` HTTPS tunnel for testing |
| **is_consequential** | Marks actions that write files or have side effects |

---

## 3. Package Structure

```
my-action-package/
├── package.yaml          ← REQUIRED: manifest
├── conda.yaml            ← REQUIRED: Python environment
├── actions/
│   ├── my_actions.py     ← @action functions
│   └── helpers.py        ← internal helpers (not exposed)
├── public/               ← optional: static assets
│   └── images/
└── README.md
```

**Rules:**
- Only files listed in `package.yaml → actions:` are scanned for `@action`
- `conda.yaml` controls the full Python environment (no system Python bleed-through)
- Action Server creates and caches the env on first `start`

---

## 4. Writing Actions (@action)

### Minimal example

```python
from sema4ai.actions import action

@action
def greet(name: str) -> str:
    """Return a greeting.

    Args:
        name: The person to greet.

    Returns:
        Greeting string.
    """
    return f"Hello, {name}!"
```

### Full decorator options

```python
from sema4ai.actions import action, get_app

@action(is_consequential=True)   # writes files, sends emails, etc.
def write_report(content: str, path: str) -> str:
    ...
```

### Type support (auto-mapped to OpenAPI / MCP schema)

| Python type | JSON schema type | Notes |
|---|---|---|
| `str` | `string` | Most common for agent I/O |
| `int`, `float` | `number` | |
| `bool` | `boolean` | |
| `list[str]` | `array` | |
| Pydantic `BaseModel` | `object` | Full schema validation |

**Best practice for LLM agents**: always return `str` (typically a JSON string). Agents handle string I/O most reliably.

### Accessing the FastAPI app

```python
from sema4ai.actions import get_app
from fastapi.staticfiles import StaticFiles

# Called at module import time — runs once when Action Server starts
app = get_app()
app.mount("/public/images", StaticFiles(directory="./public/images"), name="static")
```

### Docstring format (Google style — used to generate MCP tool descriptions)

```python
@action
def parse_pdf(source: str, output_format: str = "markdown") -> str:
    """Parse a PDF and return structured content.

    Args:
        source: HTTP/HTTPS URL or absolute local path to the PDF.
        output_format: Output format. Options: "markdown", "json".

    Returns:
        JSON string with keys: status, content, page_count.

    Examples:
        >>> result = parse_pdf("https://example.com/report.pdf")
        >>> data = json.loads(result)
    """
```

The docstring becomes the MCP tool description seen by the LLM. Write it as if you're explaining the tool to the model — be explicit about accepted values, return shape, and examples.

---

## 5. package.yaml Reference

```yaml
spec-version: v2          # REQUIRED — always v2

name: my-package           # REQUIRED — kebab-case, no spaces
description: >
  What this package does. Shown in the Action Server UI.

version: 1.0.0             # semver

documentation: README.md   # optional, path relative to package root

dependencies:
  conda: conda.yaml        # REQUIRED — path to conda env file

actions:
  - actions/my_actions.py  # REQUIRED — one or more action files
  - actions/other.py
```

**Common mistakes:**
- Using `spec-version: v1` → breaks on newer Action Server
- Spaces in `name` → kebab-case only
- Forgetting to list all action files under `actions:`

---

## 6. conda.yaml Reference

```yaml
channels:
  - conda-forge
  - defaults

dependencies:
  - python=3.11            # pin exact version
  - pip

  - pip:
    - sema4ai-actions>=0.10.0   # REQUIRED runtime
    - httpx>=0.27.0
    - fastapi>=0.111.0
    - Pillow>=10.3.0
    # ... your deps
```

**Rules:**
- Always pin `python=X.Y` — Action Server creates an isolated env
- `sema4ai-actions` is the only mandatory pip dep
- Use `conda-forge` as primary channel for best Linux compatibility
- Avoid `conda install` of packages also in `pip:` — pick one channel per package

**Rebuilding the env** (after changing conda.yaml):
```bash
action-server start --auto-reload   # detects conda.yaml changes and rebuilds
# or force:
rm -rf ~/.sema4ai/action-server/envs/<package-hash>/
action-server start ...
```

---

## 7. CLI Reference

```bash
# Start in current directory
action-server start

# Start specific package dir
action-server start --dir /path/to/my-package

# Hot-reload on file changes (dev mode)
action-server start --auto-reload

# Custom port
action-server start --port 8082

# Public tunnel (see section 8)
action-server start --expose

# All dev options together
action-server start \
  --dir ./my-package \
  --port 8082 \
  --auto-reload \
  --expose

# List discovered actions (dry run — no server)
action-server schema

# Import a package from a .zip
action-server import --dir ./my-package.zip

# Check version
action-server version
```

---

## 8. --expose Flag: Rapid Testing

This is one of the most useful features for development.

### What --expose does

When you add `--expose`, Action Server:
1. Creates a **secure reverse tunnel** to Sema4.ai's relay servers
2. Assigns a public HTTPS URL like `https://a1b2c3d4.sema4ai.link`
3. All traffic to that URL is forwarded to your local server
4. **The URL changes on every restart** (ephemeral by default)

```bash
action-server start --auto-reload --expose --dir ./my-package

# Output:
# ✓ Action Server started on http://localhost:8080
# ✓ Public URL: https://a1b2c3d4.sema4ai.link
# ✓ MCP endpoint: https://a1b2c3d4.sema4ai.link/api/mcp/
```

### Why this is powerful for testing

| Without --expose | With --expose |
|---|---|
| Only reachable on localhost | Reachable from anywhere |
| Can't test with Claude.ai or remote agents | Works immediately with any LLM platform |
| Need ngrok or similar separately | Built-in, zero config |
| No HTTPS | Full HTTPS with valid cert |
| Can't share with teammates | Share the URL and they can test instantly |

### Typical rapid-test workflow

```bash
# Terminal 1: start server
action-server start --auto-reload --expose --dir ./my-package

# Terminal 2: test with curl
export AS_URL="https://a1b2c3d4.sema4ai.link"

curl -s -X POST \
  $AS_URL/api/actions/my-package/greet/run \
  -H "Content-Type: application/json" \
  -d '{"name": "Weke"}' | python3 -m json.tool

# Test MCP discovery
curl -s $AS_URL/api/mcp/ | python3 -m json.tool
```

### Use --expose with environment variable for URL auto-injection

```bash
export ACTION_SERVER_URL="https://a1b2c3d4.sema4ai.link"
action-server start --auto-reload --expose --dir ./my-package
```

Then inside your `@action` code:
```python
import os
base_url = os.environ.get("ACTION_SERVER_URL", "http://localhost:8080")
```

### Limitations of --expose

- URL changes on every restart → update your MCP client config each time
- Requires internet access (outbound TCP to Sema4.ai relays)
- Not for production — use a proper reverse proxy (nginx/Caddy) for prod
- Rate limits may apply for the free tunnel tier

---

## 9. Static File Serving

To serve files (images, PDFs, reports) directly from Action Server:

```python
# In your actions file — runs at import time
from pathlib import Path
from sema4ai.actions import get_app
from fastapi.staticfiles import StaticFiles

PUBLIC_DIR = Path(__file__).parent.parent / "public" / "images"
PUBLIC_DIR.mkdir(parents=True, exist_ok=True)

try:
    app = get_app()
    app.mount(
        "/public/images",
        StaticFiles(directory=str(PUBLIC_DIR), html=False),
        name="static_images",
    )
except Exception as e:
    import logging
    logging.warning("Static mount skipped: %s", e)
```

Then from `@action` functions return URLs like:
```python
base = os.environ.get("ACTION_SERVER_URL", "http://localhost:8080")
url = f"{base}/public/images/{filename}"
```

With `--expose` active:
```
https://a1b2c3d4.sema4ai.link/public/images/report_page1.png
```

---

## 10. MCP Integration

Action Server implements the MCP (Model Context Protocol) spec.

### MCP endpoint

```
GET  <server>/api/mcp/              # list available tools
POST <server>/api/mcp/              # invoke a tool (MCP format)
```

### Connect with MultiServerMCPClient (LangChain)

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

async with MultiServerMCPClient({
    "my-package": {
        "url": "https://a1b2c3d4.sema4ai.link/api/mcp/",
        "transport": "streamable_http",
    }
}) as client:
    tools = client.get_tools()
    # tools is a list of LangChain BaseTool objects
```

### Multiple packages simultaneously

```python
async with MultiServerMCPClient({
    "docling-tools": {
        "url": "https://abc.sema4ai.link/api/mcp/",
        "transport": "streamable_http",
    },
    "financial-data": {
        "url": "https://xyz.sema4ai.link/api/mcp/",
        "transport": "streamable_http",
    },
    "mikrotik-agent": {
        "url": "http://192.168.1.100:8083/api/mcp/",
        "transport": "streamable_http",
    },
}) as client:
    all_tools = client.get_tools()
```

### Direct REST call (no MCP client)

```bash
curl -X POST https://a1b2c3d4.sema4ai.link/api/actions/my-package/my-action/run \
  -H "Content-Type: application/json" \
  -d '{"param1": "value1", "param2": 42}'
```

URL pattern: `/api/actions/<package-name>/<action-name-kebab>/run`
- `package-name` = `name` field in `package.yaml`
- `action-name-kebab` = Python function name with `_` replaced by `-`

---

## 11. LangChain / LangGraph Connection

### With Claude (Anthropic)

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_anthropic import ChatAnthropic

async def run_agent(query: str):
    async with MultiServerMCPClient({
        "my-package": {
            "url": "https://a1b2c3d4.sema4ai.link/api/mcp/",
            "transport": "streamable_http",
        }
    }) as client:
        tools = client.get_tools()
        llm = ChatAnthropic(model="claude-sonnet-4-20250514").bind_tools(tools)
        response = await llm.ainvoke(query)
        return response

asyncio.run(run_agent("Process this PDF: https://example.com/report.pdf"))
```

### With LangGraph ReAct agent

```python
from langgraph.prebuilt import create_react_agent

async def run_graph(query: str):
    async with MultiServerMCPClient({...}) as client:
        tools = client.get_tools()
        agent = create_react_agent(
            ChatAnthropic(model="claude-sonnet-4-20250514"),
            tools
        )
        result = await agent.ainvoke({"messages": [("human", query)]})
        return result["messages"][-1].content
```

---

## 12. Environment Variables

| Variable | Set by | Purpose |
|---|---|---|
| `ACTION_SERVER_URL` | You (or --expose output) | Base URL for building public URLs in actions |
| `SEMA4AI_HOME` | System | Cache dir for envs (~/.sema4ai/) |
| `ROBOCORP_HOME` | Legacy alias | Same as above |
| `ACTION_SERVER_DATADIR` | System | Where Action Server stores its DB |

---

## 13. Production Deployment

For production **do not use `--expose`**. Instead:

```bash
# 1. Run Action Server on a fixed port (no --expose)
action-server start \
  --port 8082 \
  --dir /opt/action-packages/my-package

# 2. Put nginx or Caddy in front for TLS + routing
# nginx snippet:
location /api/ {
    proxy_pass http://127.0.0.1:8082;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

# 3. Use systemd for auto-restart
[Service]
ExecStart=action-server start --port 8082 --dir /opt/action-packages/my-package
Restart=always
```

For multiple packages on one machine use different ports (8081, 8082, 8083…) and route via nginx path prefixes.

---

## 14. Debugging Patterns

### Action not found (404)
```bash
# Check discovered actions
action-server schema --dir ./my-package
# Verify the function name → kebab-case mapping:
# def my_cool_action → /api/actions/my-package/my-cool-action/run
```

### Conda env not rebuilding
```bash
# Force rebuild by removing cached env
find ~/.sema4ai -name "*.lock" -delete
action-server start --auto-reload --dir ./my-package
```

### get_app() returns None / mount fails
- Ensure `from sema4ai.actions import get_app` at top of file
- Call `get_app()` at module level (not inside `@action` function body)
- Check `sema4ai-actions` version: `pip show sema4ai-actions`

### Action hangs (timeout)
- Default timeout: none. Set in the HTTP client calling the action.
- For long-running actions (OCR, ML inference), return immediately and poll, or increase client timeout.

### Logs
```bash
# Verbose logging
action-server start --dir . --auto-reload --expose 2>&1 | tee action-server.log

# Inside action code:
import logging
logger = logging.getLogger(__name__)
logger.info("Processing: %s", source)
```

---

## 15. Full Working Example

For a complete production-grade example with static file serving, see the `docling-tools` reference in context. The pattern is:

```
package.yaml (spec-version: v2)
conda.yaml (python=3.11 + sema4ai-actions + your deps)
actions/
  my_actions.py:
    - Module-level: get_app() + StaticFiles mount
    - Helper functions (no @action)
    - @action functions with Google-style docstrings
    - Always return str (JSON)
    - Handle errors with try/except, return {"status":"error",...}
public/images/   ← static files served at /public/images/
```

For detailed reference on specific topics:
- `references/action-patterns.md` — advanced patterns: auth, streaming, background tasks
- `references/mcp-schemas.md` — MCP protocol payload examples

---

## Quick-start cheatsheet

```bash
# New package from scratch
mkdir my-package && cd my-package
mkdir -p actions public/images

# Create package.yaml, conda.yaml, actions/my_actions.py
# (see sections 5, 6, 4 above)

# Dev start
action-server start --auto-reload --expose --dir .

# Test one action
curl -X POST https://<tunnel>.sema4ai.link/api/actions/my-package/my-action/run \
  -H "Content-Type: application/json" \
  -d '{"arg": "value"}'

# Connect LLM agent
# url = "https://<tunnel>.sema4ai.link/api/mcp/"
# transport = "streamable_http"
```
