---
name: antimlabs
description: Use when generating physics-ready robotics simulation scenes/assets via Antim Labs' Gizmo REST API — covers scene/asset generation from text prompts, job polling/SSE, and export to MJCF/USD/SDF for MuJoCo, Isaac Sim, and Gazebo.
---

# Antim Labs

## What it is

**Antim Labs** builds simulation infrastructure for Physical AI / robotics. Their flagship product is **Gizmo**, an AI-powered 3D simulation authoring tool that generates physics-ready scenes and assets from text prompts or reference images. The premise: "simulation authoring is the bottleneck" for robot-learning teams, so Gizmo automates building structured 3D environments (with scale, colliders, articulation/joints, mass/inertia/friction, semantics) and exports them to the standard robotics simulator formats.

Inputs: text prompts, reference images (and per marketing: floorplans, point clouds). Outputs: **USD** (NVIDIA Isaac Sim / Omniverse), **MJCF** (MuJoCo), **SDF** (Gazebo/ROS). Ships with 3,000+ premade physics-ready assets. NVIDIA Inception partner. Product is in **beta** (browser editor is desktop-only; occasional asset-generation failures that regeneration usually fixes).

There are two surfaces:
- **Browser editor** — https://gizmo.antimlabs.com (sign-in required; desktop only)
- **REST API** — https://api.gizmo.antimlabs.com/v1 (this skill focuses here)

> Note: Do not confuse Antim Labs / Gizmo with "Antioch" (a different NY robotics-sim startup that also pitches itself as "Cursor for physical AI"). Some press search results conflate them.

## When to use this skill

Use when an engineer wants to **programmatically generate robotics simulation scenes/assets** and pull them into MuJoCo, Isaac Sim, or Gazebo — e.g. batch-generating diverse training environments, building eval suites, or scripting scene creation/editing as part of a robot-learning pipeline. Reach for the REST API for automation/batch workflows; use the browser editor for interactive authoring.

## Setup & auth

There is **no published pip/npm SDK** as of this writing — Gizmo exposes a plain **REST API**. Integrate with `requests`/`httpx` (Python), `fetch` (JS), or `curl`.

1. Get access: API access is gated in beta. Request it by emailing **viswajit@antimlabs.com** (or via the in-app "request API access"). Create keys at **Settings → API Keys**: https://gizmo.antimlabs.com/settings/api-keys
2. Keys look like `gzm_k1_...`. Each key has configurable rate limits and scopes.
3. Auth header (Bearer is the documented form; the asset docs also show a bare `authorization: YOUR_API_KEY` form):

```
Authorization: Bearer gzm_k1_YOUR_KEY
```

There is **no documented official env-var name** — use your own. Recommended convention:

```bash
export GIZMO_API_KEY="gzm_k1_..."
export GIZMO_BASE_URL="https://api.gizmo.antimlabs.com/v1"
```

Base URL: `https://api.gizmo.antimlabs.com/v1`
Full machine-readable spec: `GET https://api.gizmo.antimlabs.com/v1/openapi.json`

Verify your key:

```bash
curl "https://api.gizmo.antimlabs.com/v1/whoami" \
  -H "Authorization: Bearer $GIZMO_API_KEY"
```

## Core SDK / API usage

All generation is **asynchronous**: a `POST` returns `202 Accepted` with a `job_id` (and usually a `scene_id`/`asset_id`); you then poll `GET /v1/jobs/{job_id}` or subscribe to the SSE event stream until `succeeded`, then export.

### Generate a scene (real, from docs)

```bash
curl -X POST "https://api.gizmo.antimlabs.com/v1/scenes" \
  -H "Authorization: Bearer $GIZMO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "A modern robotics lab with two workbenches"}'
```

`POST /v1/scenes` body (`GenerateSceneBody`):
| field | type | default | notes |
|---|---|---|---|
| `prompt` | string | — | required; natural-language scene description |
| `model` | string\|null | — | override default LLM |
| `asset_pipeline` | enum `auto`\|`gizmo`\|`cad` | `auto` | asset geometry pipeline |
| `persist` | boolean | `true` | save to your library |

Response (`202`):
```json
{ "ok": true, "scene_id": "jh72k3m4n5p6q7r8", "job_id": "job_a1b2c3d4", "status": "queued" }
```
Typical scene generation: **2–5 minutes**.

### Generate a single asset (real, from docs)

```bash
curl -X POST "https://api.gizmo.antimlabs.com/v1/assets" \
  -H "Authorization: Bearer $GIZMO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A 6-DOF robotic arm with gripper end-effector",
    "asset_pipeline": "auto",
    "persist": true
  }'
```
Body fields: `prompt` (required), `scene_id` (optional, attach to a scene), `model`, `asset_pipeline`, `persist`. Returns `202` with a `job_id`. Asset gen includes geometry, joints (revolute/prismatic/spherical/fixed), materials, physics (mass/inertia/friction/collision), affordances. Typical: **30–90 seconds**.

### Poll job status (real, from docs)

```bash
curl "https://api.gizmo.antimlabs.com/v1/jobs/$JOB_ID?include_result=true" \
  -H "Authorization: Bearer $GIZMO_API_KEY"
```
Response:
```json
{
  "ok": true,
  "job": {
    "id": "job_a1b2c3d4",
    "status": "running",
    "mode": "scene",
    "scene_id": "jh72k3m4n5p6q7r8",
    "created_at": 1717800000,
    "started_at": 1717800001
  }
}
```
Status lifecycle: `queued` → `running` → `succeeded` | `failed` | `cancelled`. Add `?include_result=true` to get the full result payload once complete.

### Stream progress via SSE (real, from docs)

`GET /v1/jobs/{job_id}/events` — Server-Sent Events. Event types: `stage_start`, `stage_complete`, `asset_ready`, `error`, `ping`, `done`. Resume from a sequence number with `?after=N`.

### Export to MJCF / USD / SDF (real, from docs)

```bash
curl -X POST "https://api.gizmo.antimlabs.com/v1/scenes/$SCENE_ID/export" \
  -H "Authorization: Bearer $GIZMO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"format": "mjcf"}' \
  -o scene.zip
```
`format`: `mjcf` | `usd` | `sdf`. Returns a **ZIP archive**. Optionally pass a `robot_profile` to embed a robot into the scene. Single-asset export: `POST /v1/assets/{asset_id}/export` (same format options).

### End-to-end Python (assembled from documented endpoints — verify response field names against `openapi.json`)

```python
import os, time, requests

BASE = os.environ.get("GIZMO_BASE_URL", "https://api.gizmo.antimlabs.com/v1")
H = {"Authorization": f"Bearer {os.environ['GIZMO_API_KEY']}"}

# 1. kick off scene generation
r = requests.post(f"{BASE}/scenes",
                  headers={**H, "Content-Type": "application/json"},
                  json={"prompt": "A warehouse aisle with shelving and pallets"})
r.raise_for_status()
data = r.json()
job_id, scene_id = data["job_id"], data["scene_id"]

# 2. poll until done
while True:
    j = requests.get(f"{BASE}/jobs/{job_id}", headers=H).json()["job"]
    if j["status"] in ("succeeded", "failed", "cancelled"):
        break
    time.sleep(10)
assert j["status"] == "succeeded", j

# 3. export to MuJoCo MJCF as a zip
exp = requests.post(f"{BASE}/scenes/{scene_id}/export",
                    headers={**H, "Content-Type": "application/json"},
                    json={"format": "mjcf"})
exp.raise_for_status()
open("scene.zip", "wb").write(exp.content)
```

## Common use cases / patterns

- **Batch environment generation for robot learning**: loop prompts → `POST /v1/scenes` → poll → export `usd`/`mjcf` to train policies on diverse worlds.
- **Iterative scene editing in natural language**: `POST /v1/scenes/{scene_id}/edit` with body `{"prompt": "Add a conveyor belt along the north wall"}` (field is `prompt`, optional `model`). Returns a job to poll.
- **Asset library / catalog reuse** instead of generating from scratch: `GET /v1/catalog` (full-text search + category filter + pagination), `GET /v1/catalog/categories`, `GET /v1/catalog/{slug}` (returns GLB + USDZ download paths). 3,000+ premade physics-ready assets.
- **Pipeline introspection**: `GET /v1/scenes/{scene_id}/status` for per-stage pipeline timing/errors; `GET /v1/scenes/{scene_id}?include_record=true` for the full scene graph from storage.
- **Management**: `GET /v1/scenes`, `GET /v1/assets` (both newest-first; assets filterable by scene), `PATCH /v1/scenes/{scene_id}` (rename/describe without regenerating), `DELETE` on either, `POST /v1/jobs/{job_id}/cancel`.
- **Targeting a specific simulator**: MJCF → MuJoCo, USD → Isaac Sim/Omniverse, SDF → Gazebo/ROS.

## Gotchas & limits

- **Beta product.** Occasional asset-generation failures during generation — regenerating usually resolves them. Browser editor is **desktop-only**.
- **No official SDK / pip / npm package** — REST only. Treat env-var names (`GIZMO_API_KEY`) as your own convention, not vendor-blessed.
- **Async everything.** Always go POST → poll/SSE → export. Don't expect a synchronous artifact from the generate call (it returns `202` + `job_id`).
- **Rate limit:** default **100 requests/minute per key**. Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. `429 rate_limit_exceeded` on overflow.
- **Credits:** generation can return `402 insufficient_credits`. Check `GET /v1/usage` for per-key request counts, last-used, and rate config.
- **Error shape:** `{"error":{"code":"scene_not_found","message":"...","status":404}}`. Common codes: `authentication_required` (401), `insufficient_credits` (402), `scene_not_found` (404), `rate_limit_exceeded` (429), plus `422` validation errors.
- **Auth header variants:** Bearer is the documented norm, but the API also accepts `x-gizmo-service-token`, `x-gizmo-user-id`, `x-gizmo-convex-user-id` (Convex backend); the asset-gen sample shows a bare `authorization: YOUR_API_KEY`. Prefer `Authorization: Bearer`.
- **Verification flag:** the end-to-end Python script's exact result-payload field names (under `include_result=true`) and the `robot_profile` schema are not fully spelled out in the public docs — confirm against `https://api.gizmo.antimlabs.com/v1/openapi.json` before relying on them.
- **No public pricing.** Access is sales/waitlist-gated (Book Demo / Request Enterprise Demo / email).

## Links

- Homepage: https://www.antimlabs.com
- Gizmo product (browser editor, beta): https://gizmo.antimlabs.com
- API key settings: https://gizmo.antimlabs.com/settings/api-keys
- Docs portal: https://docs.gizmo.antimlabs.com
- API access guide: https://docs.gizmo.antimlabs.com/api-access
- LLM-friendly docs index: https://docs.gizmo.antimlabs.com/llms.txt  (full text: https://docs.gizmo.antimlabs.com/llms-full.txt)
- OpenAPI spec: https://api.gizmo.antimlabs.com/v1/openapi.json
- Discord: https://discord.gg/7TsNZHgtKA
- X/Twitter: https://x.com/AntimLabs
- Contact / API access: viswajit@antimlabs.com (co-founder Viswajit Vinod Nair)
