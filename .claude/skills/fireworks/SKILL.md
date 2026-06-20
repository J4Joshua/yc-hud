---
name: fireworks
description: Use when building on Fireworks AI — fast serverless inference for open models (chat, vision, embeddings, FLUX image gen) plus fine-tuning (SFT/RFT) — covers the fireworks-ai Python SDK, OpenAI-compatible API, function calling/structured outputs, and reinforcement fine-tuning with eval-protocol evaluators.
---

# Fireworks AI

## What it is

Fireworks AI (https://fireworks.ai) is a fast inference and fine-tuning platform for open-source and custom generative models. It hosts 100+ optimized models (DeepSeek, Kimi K2, Qwen, GLM, Gemma, Llama, plus vision, audio, embeddings, rerank, and FLUX image models) behind a low-latency API, and offers managed fine-tuning — supervised (SFT), LoRA, and reinforcement fine-tuning (RFT) — for models up to 1T+ params. It is positioned for building production AI agents on fine-tuned open models that can outperform closed frontier models on a specific use case.

Three inference modes: **Serverless** (pay-per-token, OpenAI/Anthropic-compatible), **On-Demand / Dedicated** (dedicated GPU deployments with autoscaling), and **Reserved** (guaranteed capacity).

## When to use this skill

- Calling fast inference on open models (chat, vision, function calling, structured JSON, embeddings, reranking, image generation).
- Migrating from OpenAI — Fireworks exposes a drop-in OpenAI-compatible endpoint.
- Building agents and then fine-tuning the underlying open model with SFT or RFT to beat a closed model on your task.
- Running reinforcement fine-tuning where you write a Python reward function instead of labeling data.

## Setup & auth

Get an API key from the Fireworks dashboard (https://fireworks.ai → sign up), then:

```bash
export FIREWORKS_API_KEY="fw_..."          # primary auth env var
# optional, used for account-scoped resources (fine-tuning, deployments):
export FIREWORKS_ACCOUNT_ID="your-account-id"
```

Install (note `--pre` — the v0.x SDK ships pre-release builds; requires Python 3.9+):

```bash
pip install --pre fireworks-ai
# extras:
pip install --pre 'fireworks-ai[training]'      # fine-tuning / training SDK
pip install --upgrade 'fireworks-ai[reward-kit]' # RFT evaluators (reward functions)
```

CLI (`firectl`) for deployments and fine-tuning jobs:

```bash
brew tap fw-ai/firectl && brew install firectl   # macOS; direct downloads exist for Linux/Windows
firectl signin                                   # add <ACCOUNT_ID> if using custom SSO
firectl whoami
```

- **Base URL (OpenAI-compatible):** `https://api.fireworks.ai/inference/v1`
- **Auth header:** `Authorization: Bearer $FIREWORKS_API_KEY`
- **Model ID format:** `accounts/fireworks/models/<model>` (e.g. `accounts/fireworks/models/deepseek-v3p1`, `accounts/fireworks/models/kimi-k2-instruct-0905`). Your own fine-tunes use `accounts/<your-account>/models/<name>`.

## Core SDK / API usage

### Chat completions (Fireworks SDK)

```python
from fireworks import Fireworks

client = Fireworks()  # reads FIREWORKS_API_KEY from env

response = client.chat.completions.create(
    model="accounts/fireworks/models/deepseek-v3p1",
    messages=[{"role": "user", "content": "Say hello in Spanish"}],
)
print(response.choices[0].message.content)
```

Streaming and async (`AsyncFireworks` + `await`, supports SSE):

```python
import asyncio
from fireworks import AsyncFireworks

async def main():
    client = AsyncFireworks()
    stream = await client.chat.completions.create(
        model="accounts/fireworks/models/kimi-k2-instruct-0905",
        messages=[{"role": "user", "content": "How do LLMs work?"}],
        stream=True,
    )
    async for chunk in stream:
        print(chunk.choices[0].delta.content or "", end="")

asyncio.run(main())
```

### OpenAI-compatible (drop-in)

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["FIREWORKS_API_KEY"],
    base_url="https://api.fireworks.ai/inference/v1",
)
resp = client.chat.completions.create(
    model="accounts/fireworks/models/deepseek-v3p1",
    messages=[{"role": "user", "content": "Say hello in Spanish"}],
)
print(resp.choices[0].message.content)
```

### Image generation (FLUX) — REST, verified from docs

The image/workflow endpoints are REST (not under `chat.completions`):

```python
import requests, os

url = ("https://api.fireworks.ai/inference/v1/workflows/"
       "accounts/fireworks/models/flux-1-schnell-fp8/text_to_image")
headers = {
    "Content-Type": "application/json",
    "Accept": "image/jpeg",
    "Authorization": f"Bearer {os.environ['FIREWORKS_API_KEY']}",
}
data = {"prompt": "A beautiful sunset over the ocean"}  # optional: aspect_ratio,
                                                        # guidance_scale (3.5),
                                                        # num_inference_steps (4), seed
r = requests.post(url, headers=headers, json=data)
if r.status_code == 200:
    open("a.jpg", "wb").write(r.content)
```

Image editing models exist too: `flux-kontext-pro`, `flux-kontext-max`.

### Embeddings

Use the OpenAI-compatible `embeddings` endpoint with an embedding model, e.g. `nomic-ai/nomic-embed-text-v1.5`:

```python
emb = client.embeddings.create(
    model="nomic-ai/nomic-embed-text-v1.5",
    input="text to embed",
)
vector = emb.data[0].embedding
```

(Embeddings work through the OpenAI-compatible client shown above; exact model availability — check the model library.)

### Reinforcement Fine-Tuning (RFT) — write a reward function, not labels

RFT's core idea: you write a Python function that scores model outputs; Fireworks handles GPUs, orchestration, and experiment management. Install the evaluator extra:

```bash
pip install --upgrade 'fireworks-ai[reward-kit]'
```

Define an evaluator (must live in its own directory — the Build SDK recursively packages sibling/child files and pulls deps from `pyproject.toml`/`requirements.txt`):

```python
from fireworks import reward_function

@reward_function(id="my-evaluator")
def evaluate(messages, **kwargs):
    """Score the model's last response (0.0–1.0)."""
    content = messages[-1]["content"]
    score = 1.0 if "expected answer" in content else 0.0
    return {"score": score}
```

Run / launch jobs. Two surfaces exist:
- **Build SDK datasets:** `Dataset.create_evaluation_job(evaluate)` runs the evaluator on Fireworks infra.
- **eval-protocol CLI** (separate `pip install eval-protocol`, the open RL-on-agents toolkit Fireworks RFT integrates with):

```bash
eval-protocol create rft \
  --base-model accounts/fireworks/models/qwen3-0p6b \
  # + dataset/evaluator args
```

RFT pathways from the docs: **Single-Turn** (~15 min, iterate on evaluator locally), **Remote Agents** (1–2 hrs, multi-turn rollouts happen in your environment over HTTP), **Secure/BYOB** (2–4 hrs, training data stays in your GCS/S3 bucket).

## Common use cases / patterns

- **OpenAI drop-in replacement** for cheaper/faster open-model inference — just change `base_url` + `api_key` + model id.
- **Function calling & structured outputs** — supported for agentic systems (reliable JSON); use the standard OpenAI `tools` / `response_format` params against the compatible endpoint.
- **Agent + RFT loop** — prototype an agent on a serverless model, then RFT the open base model on your task with a reward function to surpass a closed model.
- **Batch inference** — async large-scale processing at reduced cost.
- **Multimodal / vision / audio / image-gen** — single platform for LLM + FLUX + embeddings + rerank.
- **Dedicated deployments** via `firectl` for production latency/throughput SLAs.

## Gotchas & limits

- Install requires the `--pre` flag (`pip install --pre fireworks-ai`) — without it pip may grab an old stable build. Python 3.9+.
- Model IDs are fully qualified paths (`accounts/fireworks/models/...`), not bare names — a common source of 404s.
- Image generation is a **REST workflow endpoint**, not `client.images` / `chat.completions`. Don't try to call it through the chat client.
- Evaluators **must be in their own directory** (the SDK packages the whole directory) and return `{"score": <0.0–1.0>}`.
- Account-scoped operations (fine-tuning, deployments) may need `FIREWORKS_ACCOUNT_ID` and `firectl signin`.
- Pricing is per-token and per-model (e.g. published rates like GLM at ~$1.4/M input, $4.4/M output; FLUX/image priced per step/image). Fireworks announced a move to **prepaid billing as of July 1, 2026** — verify current billing in the dashboard.
- SDK has built-in retries (2 by default) for connection errors/timeouts/rate limits; auto-paginating iterators (`.has_next_page()`, `.get_next_page()`); raw/streaming response access via `.with_raw_response` / `.with_streaming_response`.
- `eval-protocol` is a **separate package** from `fireworks-ai`; RFT can be driven either via the Build SDK reward-kit decorator or via the eval-protocol CLI.

## Links

- Docs home / intro: https://docs.fireworks.ai/getting-started/introduction
- Serverless quickstart: https://docs.fireworks.ai/getting-started/quickstart
- API reference: https://docs.fireworks.ai/api-reference/introduction
- Image gen (FLUX) API: https://docs.fireworks.ai/api-reference/generate-a-new-image-from-a-text-prompt
- Reinforcement fine-tuning: https://docs.fireworks.ai/fine-tuning/reinforcement-fine-tuning-models
- Developing evaluators: https://docs.fireworks.ai/tools-sdks/python-client/developing-evaluators
- firectl CLI: https://docs.fireworks.ai/tools-sdks/firectl/firectl
- Python SDK (GitHub): https://github.com/fw-ai-external/python-sdk
- PyPI: https://pypi.org/project/fireworks-ai/
- Cookbook (recipes): https://github.com/fw-ai/cookbook
- eval-protocol SDK: https://github.com/eval-protocol/python-sdk · quickstart: https://github.com/eval-protocol/quickstart
- llms.txt doc index: https://docs.fireworks.ai/llms.txt
