---
name: modal
description: Use when running code, ML inference, training, batch jobs, or sandboxed/agent code on Modal's serverless GPU cloud from Python — covers app/function/image/GPU definitions, Sandboxes, web endpoints, secrets, volumes, and deployment.
---

# Modal

## What it is

Modal (https://modal.com) is a serverless cloud platform for AI/ML and general compute, controlled entirely from Python. You write ordinary Python functions and classes, decorate them, and Modal runs them in the cloud in containers you define in code — provisioning GPUs (H100, H200, A100, B200, A10G, L40S, T4), autoscaling from zero to 1,000+ containers, with sub-second cold starts. No Kubernetes, Dockerfiles, or YAML required.

Primary use areas: **Inference** (LLM/multimodal/online & batch serving), **Training** (fine-tuning, RL, multi-node distributed), **Sandboxes** (isolated ephemeral containers for running untrusted/agent-generated code), and **batch processing** (embeddings, evals, data generation). Billing is per-second of compute.

## When to use this skill

- You need a GPU for inference or training but don't want to manage infra.
- You want to deploy a Python function/model as an autoscaling HTTP API.
- You need to run untrusted or AI-agent-generated code in an isolated sandbox.
- You want to fan out a large batch job across many containers (`.map()`).
- You want scheduled/cron Python jobs in the cloud.

## Setup & auth

```bash
pip install modal
modal setup          # opens browser to authenticate; creates ~/.modal.toml
# if that fails: python -m modal setup
```

Import path is just `import modal`.

**Auth env vars (for CI/CD, non-interactive):** Modal reads `MODAL_TOKEN_ID` and `MODAL_TOKEN_SECRET`. These take precedence over the `~/.modal.toml` config file. Generate them via `modal token new`, or create a **Service User** in the dashboard (its `MODAL_TOKEN_ID` / `MODAL_TOKEN_SECRET` are shown only once). Set them as secrets in your CI platform — never hardcode.

```bash
export MODAL_TOKEN_ID=ak-xxxxxxxx
export MODAL_TOKEN_SECRET=as-xxxxxxxx
```

Other useful CLI: `modal run file.py` (run once), `modal serve file.py` (ephemeral hot-reloading dev deploy), `modal deploy file.py` (persistent production deploy), `modal token new`.

> Note: "API key" for Modal = the token ID + token secret pair above. Third-party credentials (OpenAI keys, HF tokens, etc.) go in **Modal Secrets** (see below), not these vars.

## Core SDK / API usage

### App, Image, Function (the basic unit)

```python
import modal

# An Image defines the container environment (built remotely, cached)
image = modal.Image.debian_slim().pip_install("torch", "numpy")
# (uv_pip_install is also available and faster: .uv_pip_install("transformers[torch]"))

app = modal.App("example", image=image)

@app.function(gpu="A100")   # gpu="H100", "H100:8" for 8 GPUs, "A10G", "T4", etc.
def run():
    import torch
    assert torch.cuda.is_available()

@app.local_entrypoint()
def main():
    run.remote()            # invoke in the cloud, block for result
```

Run it: `modal run file.py`. GPU strings: `"A100"`, `"A100-80GB"`, `"H100"`, `"H200"`, `"B200"`, `"L40S"`, `"A10G"`, `"T4"`. Append `:n` for multiple GPUs per container (e.g. `gpu="H100:8"`).

### Invocation patterns

```python
result = fn.remote(x=1)                       # one cloud call, blocking
results = list(fn.map([1, 2, 3, 4, 5]))       # fan out across containers
handle = fn.spawn(x=1); handle.get()          # async; get() retrieves result
fn.local(x=1)                                  # run locally without Modal
```

### Class with lifecycle hooks (load model once, serve many requests)

```python
@app.cls(gpu="A100", image=image)
class Model:
    @modal.enter()                 # runs once per container at startup
    def load_model(self):
        import pickle
        self.model = pickle.load(open("model.pkl", "rb"))

    @modal.method()                # callable inference method
    def predict(self, x):
        return self.model.predict(x)

@app.local_entrypoint()
def main():
    m = Model()
    print(m.predict.remote(x=123))
    print(list(m.predict.map([{"x": i} for i in range(5)])))
```

### Web endpoints (deploy a function as an HTTP API)

```python
image = modal.Image.debian_slim().pip_install("fastapi[standard]")

@app.function(image=image)
@modal.fastapi_endpoint(method="POST")   # simple JSON endpoint
def square(item: dict):
    return {"square": item["x"] ** 2}
```

Full ASGI/WSGI/raw servers:

```python
@app.function(image=image)
@modal.concurrent(max_inputs=100)        # handle 100 concurrent requests/container
@modal.asgi_app()                        # FastAPI/Starlette
def fastapi_app():
    from fastapi import FastAPI
    web_app = FastAPI()
    return web_app

# Also: @modal.wsgi_app() for Flask/Django,
#       @modal.web_server(port=8000) for raw servers (vLLM, http.server, etc.)
```

Deploy: `modal serve file.py` (dev) or `modal deploy file.py` (prod) → gives a public `*.modal.run` URL.

> Note: `@modal.web_endpoint` was renamed to `@modal.fastapi_endpoint` in recent versions; older code/snippets may still show `web_endpoint`.

### Secrets (inject third-party credentials as env vars)

```python
@app.function(secrets=[modal.Secret.from_name("my-openai-secret")])
def f():
    import os
    key = os.environ["OPENAI_API_KEY"]   # key name = whatever you named it in the Secret

# inline / dynamic:
@app.function(secrets=[modal.Secret.from_dict({"API_KEY": "..."})])
def g(): ...
# also: modal.Secret.from_dotenv()
```

Create named secrets in the dashboard or via `modal secret create my-openai-secret OPENAI_API_KEY=sk-...`.

### Volumes (persistent storage / model + dataset caching)

```python
vol = modal.Volume.from_name("hf-cache", create_if_missing=True)

@app.function(volumes={"/root/.cache/huggingface": vol}, gpu="H100")
def train(): ...
```

### Sandboxes (run arbitrary/agent code in isolation)

```python
sb_app = modal.App.lookup("my-sandbox-app", create_if_missing=True)
sb = modal.Sandbox.create(app=sb_app)          # optionally image=..., gpu=..., volumes=...

p = sb.exec("python", "-c", "print('hello')")  # run a command
print(p.stdout.read())

for line in p.stdout:                           # stream output
    print(line, end="")

sb.terminate()                                  # clean up
```

Default sandbox timeout 5 min (configurable up to 24h via `timeout=`); supports volumes, GPUs, and filesystem access. Ideal for code interpreters and AI agents.

### Scheduled / cron jobs

```python
@app.function(schedule=modal.Period(hours=5))
def interval_task(): ...

@app.function(schedule=modal.Cron("0 8 * * 1", timezone="America/New_York"))
def monday_8am(): ...
```

Deploy with `modal deploy file.py` and Modal runs them automatically.

### Real example: vLLM OpenAI-compatible LLM server

```python
vllm_image = (
    modal.Image.from_registry("nvidia/cuda:12.9.0-devel-ubuntu22.04", add_python="3.12")
    .entrypoint([])
    .uv_pip_install("vllm==0.21.0")
)
hf_cache = modal.Volume.from_name("huggingface-cache", create_if_missing=True)

@app.function(
    image=vllm_image,
    gpu="H200:1",
    scaledown_window=15 * 60,
    timeout=10 * 60,
    volumes={"/root/.cache/huggingface": hf_cache},
)
@modal.concurrent(max_inputs=100)
@modal.web_server(port=8000, startup_timeout=10 * 60)
def serve():
    import subprocess
    subprocess.Popen("vllm serve <model> --port 8000", shell=True)
```

## Common use cases / patterns

- **LLM inference API**: `@app.cls` + `@modal.enter()` to load weights once, or vLLM via `@modal.web_server`, exposing an OpenAI-compatible endpoint. Cache weights in a `modal.Volume`.
- **Batch / embeddings / evals**: `fn.map(inputs)` fans the workload across hundreds of containers automatically.
- **Fine-tuning / training**: GPU functions with volumes for datasets + checkpoints; multi-node supported.
- **Code-execution sandboxes for AI agents**: `modal.Sandbox.create()` + `exec()`.
- **Cron data pipelines**: `schedule=modal.Cron(...)`.
- **Common integration**: store HF tokens / OpenAI keys / DB creds as `modal.Secret`s; mount large models from `modal.Volume`s to avoid re-downloads on cold start.

## Gotchas & limits

- **Per-second billing** (approx, check /pricing for current): H100 ~$0.001097/s, A100-80GB ~$0.000694/s, B200 ~$0.001736/s, T4 ~$0.000164/s; CPU ~$0.0000131/physical-core/s (min 0.125 cores); memory ~$0.00000222/GiB/s; volume storage $0.09/GiB-month (1 TiB free).
- **Free / Starter tier ($0)**: $30/mo free credits, up to 3 seats, 100 containers, 10 GPU concurrency. **Team ($250/mo)**: $100/mo credits, 1,000 containers, 50 GPU concurrency, custom domains/static IP. **Enterprise**: custom, higher limits, HIPAA/SSO. Startup & academic credits available.
- Images build remotely and are cached — first run is slow, subsequent runs reuse the cache. Order layers cheap-to-expensive for cache hits.
- Imports of heavy/GPU-only libs (torch, vllm) should be **inside** the function body or guarded, since the file is also imported locally to register the app.
- `@modal.enter()` runs per container, not per request — use it for expensive one-time setup (model loading).
- `.remote()` blocks; use `.spawn()` / `.map()` for parallelism. Decorator order matters: `@app.function` (outermost) then `@modal.concurrent` then the web/asgi decorator.
- Cold starts are fast but not free for large models — cache weights in a Volume and keep containers warm via `scaledown_window`.

## Links

- Quickstart / guide: https://modal.com/docs/guide
- GPU acceleration: https://modal.com/docs/guide/gpu
- Sandboxes: https://modal.com/docs/guide/sandbox
- Web endpoints: https://modal.com/docs/guide/webhooks
- Secrets: https://modal.com/docs/guide/secrets
- Environment variables (MODAL_TOKEN_ID/SECRET): https://modal.com/docs/guide/environment_variables
- Service users (CI tokens): https://modal.com/docs/guide/service-users
- Cron/scheduling: https://modal.com/docs/guide/cron
- Lifecycle functions (@app.cls / @modal.enter): https://modal.com/docs/guide/lifecycle-functions
- modal.Image reference: https://modal.com/docs/reference/modal.Image
- vLLM inference example: https://modal.com/docs/examples/vllm_inference
- Pricing: https://modal.com/pricing
- Examples gallery: https://modal.com/docs/examples
