---
name: daytona
description: Use when running AI-generated or untrusted code in isolated cloud sandboxes, building code-execution agents, or spinning up ephemeral dev environments — covers the Daytona Python/TypeScript SDK, sandbox lifecycle, code/command execution, file & git ops, and preview URLs.
---

# Daytona

## What it is

Daytona (https://www.daytona.io, GitHub: daytonaio/daytona) is secure, elastic cloud infrastructure for running AI-generated / untrusted code. It gives you isolated, on-demand **sandboxes** (full Linux compute environments) that you create and drive programmatically via an SDK. Sandboxes start fast (advertised sub-90ms from a snapshot), are stateful, and support code execution, shell commands, a full filesystem, Git, LSP, snapshots/volumes, web-server preview URLs, and "computer use" (virtual desktops).

Primary use case: give an LLM/agent a safe place to execute code without touching your own infra.

## When to use this skill

- An agent needs to **execute model-generated code** (Python/JS/shell) safely and return results.
- You need **ephemeral, isolated dev environments** spun up per task/user/request.
- Running data-analysis, code-interpreter, build/test, or scraping workloads in isolation.
- You need a sandbox to **clone a repo, run it, and expose a preview URL** for the agent or a human.

## Setup & auth

Get an API key from the dashboard at https://app.daytona.io/ ($200 free compute credits at time of research).

**Python (3.9+):**
```bash
pip install daytona          # or: poetry add daytona
```

**TypeScript/JS:**
```bash
npm install @daytona/sdk
```

Other SDKs exist: Ruby, Go, Java, plus a `daytona` CLI and a REST API.

**Environment variables** (the SDK reads these automatically if no config object is passed):
- `DAYTONA_API_KEY` — your API key (required)
- `DAYTONA_API_URL` — API endpoint, default `https://app.daytona.io/api`
- `DAYTONA_TARGET` — region target, e.g. `us`, `eu` (plain string; the old `SandboxTargetRegion` enum was removed in v0.21.0)

```bash
export DAYTONA_API_KEY="your-api-key-here"
```

> Note: PyPI also hosts an older `daytona_sdk` package and `daytona-sdk` install name. Current docs use `pip install daytona` and the import `from daytona import Daytona`. Prefer the `daytona` package; treat `daytona_sdk` as legacy.

## Core SDK / API usage

### Quickstart (Python)
```python
from daytona import Daytona

daytona = Daytona()                       # reads DAYTONA_API_KEY from env
sandbox = daytona.create()                # default sandbox
response = sandbox.process.exec("echo 'Hello, World!'")
print(response.result)
sandbox.delete()
```

### Quickstart (TypeScript)
```typescript
import { Daytona } from '@daytona/sdk';

const daytona = new Daytona();            // reads DAYTONA_API_KEY from env
const sandbox = await daytona.create();
const response = await sandbox.process.executeCommand('echo "Hello, World!"');
console.log(response.result);
await sandbox.delete();
```

### Explicit config (Python)
```python
from daytona import Daytona, DaytonaConfig

config = DaytonaConfig(
    api_key="YOUR_API_KEY",
    api_url="https://app.daytona.io/api",
    target="us",
)
daytona = Daytona(config)
```

### Async client (Python)
```python
from daytona import AsyncDaytona

async with AsyncDaytona() as daytona:
    sandbox = await daytona.create()
    response = await sandbox.process.exec("echo 'Hello, World!'")
    print(response.result)
```

### Running code (stateless vs stateful)
```python
# Stateless: independent snippet in a clean interpreter
response = sandbox.process.code_run('''
x = 10
y = 20
print(f"Sum: {x + y}")
''')
print(response.result)          # -> "Sum: 30"

# Shell commands with options
response = sandbox.process.exec("ls -la")
response = sandbox.process.exec("sleep 3", cwd="workspace/src", timeout=5)
```
TypeScript equivalents: `sandbox.process.codeRun(...)`, `sandbox.process.executeCommand(...)`.

Stateful execution keeps variables across calls via contexts: `run_code()` together with `create_context()` / `delete_context()`.

### Long-running / interactive work: sessions
```python
sandbox.process.create_session("build")
sandbox.process.execute_session_command("build", "pip install -r requirements.txt")
sandbox.process.get_session("build")
sandbox.process.delete_session("build")
```
Interactive commands support async execution (`run_async: true`) and input via `send_session_command_input()`.

### Creating sandboxes with params
```python
from daytona import (
    Daytona, CreateSandboxFromImageParams, CreateSandboxFromSnapshotParams,
    Image, Resources,
)
daytona = Daytona()

# From a container image
sandbox = daytona.create(CreateSandboxFromImageParams(
    image=Image.debian_slim("3.12"),
    resources=Resources(cpu=2, memory=4, disk=8),   # memory/disk in GiB
))

# From a prebuilt snapshot (fastest start) with lifecycle + language
sandbox = daytona.create(CreateSandboxFromSnapshotParams(
    snapshot="daytona-medium",
    language="python",            # python | typescript | javascript
    auto_stop_interval=0,         # minutes idle before auto-stop; 0 = run indefinitely
    auto_archive_interval=60,     # minutes stopped before archive
    auto_delete_interval=120,     # minutes stopped before delete
))
```
v0.21.0 split the old single `CreateSandboxParams` into `CreateSandboxFromImageParams` and `CreateSandboxFromSnapshotParams`; the resource class is `Resources` (was `SandboxResources`).

### Lifecycle management
```python
for sb in daytona.list():
    print(sb.id, sb.state)        # state is flattened (was sandbox.instance.state pre-0.21)

sb = daytona.get("sandbox-id-or-name")
sb.start()
sb.stop()
sb.delete()                       # or daytona.delete(sb)
```

### Filesystem operations
```python
# Upload / download
with open("local_file.txt", "rb") as f:
    sandbox.fs.upload_file(f.read(), "remote_file.txt")
content = sandbox.fs.download_file("file1.txt")

# List / inspect
for file in sandbox.fs.list_files("workspace"):
    print(file.name, file.is_dir)
info = sandbox.fs.get_file_info("workspace/data/file.txt")  # info.size, info.mod_time

# Mutate
sandbox.fs.create_folder("workspace/new-dir", "755")
sandbox.fs.move_files("workspace/old.txt", "workspace/new.txt")
sandbox.fs.delete_file("workspace/file.txt")
sandbox.fs.set_file_permissions("workspace/file.txt", "644")
sandbox.fs.find_files(path="workspace/src", pattern="text")
sandbox.fs.replace_in_files(["workspace/file1.txt"], "old", "new")
```

### Git operations
```python
sandbox.git.clone(url="https://github.com/user/repo.git", path="workspace/repo")
status = sandbox.git.status("workspace/repo")
sandbox.git.add("workspace/repo", ["file.txt"])
sandbox.git.commit(path="workspace/repo", message="Update", author="Jane", email="jane@example.com")
sandbox.git.push(path="workspace/repo", username="user", password="github_token")
sandbox.git.pull(path="workspace/repo", username="user", password="github_token")
sandbox.git.create_branch("workspace/repo", "new-feature")
sandbox.git.checkout_branch("workspace/repo", "feature-branch")
```

### Expose a web server (preview URL)
```python
# Run a server in the sandbox first (e.g. on port 3000), then:
preview = sandbox.get_preview_link(3000)
print(preview.url, preview.token)

# Signed URL with token embedded (good for sharing / agent fetches)
signed = sandbox.create_signed_preview_url(3000, expires_in_seconds=3600)
print(signed.url)
```
TS: `sandbox.getPreviewLink(3000)` / `sandbox.getSignedPreviewUrl(3000, 3600)`. CLI: `daytona preview-url <name> --port 3000 --expires 3600`.

## Common use cases / patterns

- **Code-interpreter agent loop**: create one sandbox per session, feed model-generated Python to `process.code_run()` (or stateful `run_code()` to keep variables), return `response.result` to the model, `delete()` when done.
- **Repo run + preview**: `git.clone` → `process.exec` to install/build → start dev server → `get_preview_link(port)` and hand the URL to a human/agent.
- **Fan-out isolation**: spin many sandboxes from a shared `snapshot` for parallel, mutually isolated tasks; use Volumes to share data across them.
- **Build/test runners**: use `create_session` for long-running install/build commands and stream logs.
- **Snapshots**: prebuild an environment (`daytona.snapshot.create()` — renamed from `createImage()` in v0.21.0) so new sandboxes start instantly with deps baked in.

## Gotchas & limits

- **Pricing is pay-as-you-go, per-second.** Roughly vCPU $0.0504/hr and memory $0.0162/GiB/hr at research time (GPU options H100 / RTX PRO 6000 exist). $200 free credits to start. Always `delete()` or rely on `auto_stop_interval` / `auto_delete_interval` to avoid runaway cost.
- **Lifecycle intervals are in minutes.** `auto_stop_interval=0` means never auto-stop (runs/bills indefinitely until you stop it) — the opposite of "stop immediately".
- **v0.21.0 breaking changes** (if you find older snippets): `getCurrentSandbox` → `daytona.get`; `daytona.remove` → `sandbox.delete`/`daytona.delete`; `sandbox.info()` → `sandbox.refreshData()`; `sandbox.instance.state` → `sandbox.state`; `SandboxResources` → `Resources`; image/snapshot APIs renamed (`createImage` → `snapshot.create`); region enum → plain string.
- **Two package names exist**: prefer `daytona` (`from daytona import Daytona`); `daytona_sdk`/`daytona-sdk` is the older naming.
- Resource ranges seen in docs: cpu 1–4, memory 1–8 GiB, disk 1–10 GiB (account tier may change limits).
- Preview URLs for non-public ports require the returned **token** (or use the signed URL variant).

## Links (verified)

- Docs home: https://www.daytona.io/docs/en/
- Getting started: https://www.daytona.io/docs/en/getting-started/
- Python SDK reference: https://www.daytona.io/docs/python-sdk/sync/daytona/ and https://www.daytona.io/docs/en/python-sdk/sync/sandbox/
- Filesystem ops: https://www.daytona.io/docs/en/file-system-operations/
- Git ops: https://www.daytona.io/docs/en/git-operations/
- v0.21.0 migration guide: https://www.daytona.io/dotfiles/updates/daytona-sdk-v0-21-0-breaking-changes-migration-guide
- GitHub: https://github.com/daytonaio/daytona  •  SDK repo: https://github.com/daytonaio/sdk
- PyPI: https://pypi.org/project/daytona/  •  Dashboard/signup: https://app.daytona.io/
