---
name: hud
description: Use when building RL environments, agentic evals, or training data with HUD (hud.ai / hud-python) — covers the hud CLI, Environment/template/grader SDK, running agents (Claude/OpenAI/Gemini) via the gateway, and GRPO reward harvesting.
---

# HUD

## What it is

HUD (hud.ai, YC W25; GitHub `hud-evals/hud-python`, MIT) is a platform for building **RL environments** and **agentic evaluations** for AI agents across coding, browser, computer-use, and robotics. You define an environment once, write tasks (each is a prompt + a reward/grader), then run them as **evals** or as **training data** against any model at any scale. Every rollout is recorded as a replayable **trace** on hud.ai.

HUD is **protocol-first**: an agent and an environment exchange only a *manifest* (capabilities + tasks), `tasks.start` (returns the prompt), and `tasks.grade` (returns the reward). In between, the agent drives the environment's *capabilities* (shell, browser, GUI, robot) directly. This means any model/harness plugs into any environment, and environments outlive any single agent.

## When to use this skill

- Building an eval/benchmark for an agent (coding, browser/computer-use, deep research, ops).
- Producing post-training reward data (GRPO/PPO) — every graded rollout is training data.
- Running an existing taskset/benchmark (e.g. `SheetBench`) against Claude/GPT/Gemini and comparing.
- Wrapping a real system (a web app, a repo, a desktop, a robot sim) as an agent-testable environment.

## Setup & auth

```bash
# Install the CLI (recommended) — uv tool gives you the `hud` command
uv tool install hud-python --python 3.12
# …or as a Python library
pip install hud-python
```

Get an API key at **https://hud.ai/project/api-keys**. The single env var is **`HUD_API_KEY`**.

```bash
hud set HUD_API_KEY=your-key-here          # persists to ~/.hud/.env
# or
export HUD_API_KEY=your-key-here
```

One `HUD_API_KEY` does two things: (1) routes model calls through the **HUD gateway** (`inference.hud.ai`, an OpenAI-compatible endpoint fronting Claude/GPT/Gemini/Grok behind one key) and (2) traces every rollout to hud.ai. **Local `hud eval` runs do NOT require an API key** unless you want gateway routing or platform tracing. If a provider key (e.g. `ANTHROPIC_API_KEY`) is present, calls go straight to the provider unless you pass `--gateway`.

Other useful env vars:
- `HUD_TELEMETRY_LOCAL_DIR` — write trace spans to disk so `hud trace <id>` works offline.

Fastest start — hand the docs to your coding agent as a skill:
```bash
npx skills add https://docs.hud.ai
```

## Quickstart (verified, from docs.hud.ai/quickstart)

```bash
hud set HUD_API_KEY=your-key-here
hud init my-env        # scaffolds env.py, tasks.py, Dockerfile.hud, pyproject.toml
cd my-env
hud eval tasks.py claude    # spawn env locally, run the `claude` agent, grade, print reward
```

The scaffolded `tasks.py`:

```python
from hud import Environment

env = Environment(name="letter-count")

@env.template()
async def count_letter(word: str = "strawberry", letter: str = "r"):
    answer = yield f"How many '{letter}'s are in '{word}'? Reply with just the number."
    yield 1.0 if answer and str(word.count(letter)) in answer else 0.0

tasks = [count_letter(word=w) for w in ("strawberry", "raspberry", "blueberry")]
```

## Core SDK / API usage

### Defining an environment + tasks

A **template** is an async generator registered with `@env.template()`. It has **exactly two yields**: the 1st yields the prompt; the generator pauses while the agent works; the agent's answer comes back into the generator; the 2nd yields the reward (a float `0.0`–`1.0`). Calling a template mints a concrete `Task`; one template spans a whole *space* of tasks.

```python
from hud import Environment

env = Environment(name="letter-count")

@env.template()
async def count_letter(word: str = "strawberry", letter: str = "r"):
    answer = yield f"How many '{letter}'s are in '{word}'?"   # 1st yield: prompt
    yield 1.0 if answer == str(word.count(letter)) else 0.0   # 2nd yield: reward

task  = count_letter(word="raspberry")                         # one Task row (not yet run)
tasks = [count_letter(word=w) for w in ("strawberry", "raspberry", "blueberry")]  # a dataset
```

Build a `Task` row directly (for data pipelines / generated rows):
```python
from hud import Task
task = Task(env="letter-count", id="count_letter", args={"word": "strawberry"}, slug="count-straw")
```

### Grading (the second yield)

Three approaches, increasing power. Prefer **grading the world** (state the agent left) over grading the answer text when possible — it's the most reliable signal.

```python
# 1) Plain Python helpers from hud.graders (each returns a float)
from hud.graders import numeric_match, exact_match, contains, f1_score, normalize
yield numeric_match(answer, word.count(letter))

# 2) Async graders — score a shell command (exit code) or an LLM judgment
from hud.graders import BashGrader, LLMJudgeGrader
result = await BashGrader.grade(command="pytest tests/test_health.py -q")
yield result.value
# result = await LLMJudgeGrader.grade(answer=answer, criteria=[...]); yield result.value
```

### Capabilities (give the agent a shell / browser / GUI / robot)

| Protocol | What it exposes |
|----------|-----------------|
| `ssh`   | Shell + files in a sandboxed workspace (`env.workspace(root)`) |
| `mcp`   | Tools over the Model Context Protocol |
| `cdp`   | Browser control over Chrome DevTools Protocol |
| `rfb`   | Full computer-use over VNC (screen + keyboard/mouse) |
| `robot` *(beta)* | Schema-driven robot observation/action loop over WebSocket |

```python
from hud import Environment
from hud.capabilities import Capability

env = Environment(name="coder")
env.workspace("workspace")   # one line: brings up an ssh-backed sandbox dir + tears it down

# Or declare in the constructor / publish a daemon from a lifecycle hook:
@env.initialize
async def _up():
    browser = await launch_chromium()
    env.add_capability(Capability.cdp(name="browser", url=f"ws://127.0.0.1:{browser.port}"))

@env.shutdown
async def _down():
    ...
```

### Running an agent

`create_agent(model)` is the preferred path — selects the matching provider agent and routes through the HUD gateway by model name alone.

```python
from hud.agents import create_agent
from hud.eval import LocalRuntime
from tasks import TASKS

agent = create_agent("claude-sonnet-4-5")   # also "gpt-...", "gemini-...", "grok-..."
job = await TASKS.run(agent, runtime=LocalRuntime("env.py"))
print(job.reward)
```

Provider agents directly (full config / your own provider key):
```python
from hud.agents import ClaudeAgent
from hud.agents.types import ClaudeConfig
agent = ClaudeAgent(ClaudeConfig(model="claude-sonnet-4-5", max_steps=30))
```
Built-in agents: `ClaudeAgent`/`ClaudeConfig`, `OpenAIAgent`/`OpenAIConfig`, `GeminiAgent`/`GeminiConfig`, `OpenAIChatAgent`/`OpenAIChatConfig` (Chat Completions, point at vLLM/local via `base_url`), `ClaudeSDKAgent`/`ClaudeSDKConfig` (runs the `claude` CLI over an `ssh` capability). Configs live in `hud.agents.types`.

### Training — harvest rewards into GRPO advantages

Every rollout returns a `Run` with a `trace_id` and a `reward`, so evaluated tasks are already training data.

```python
from hud.agents import create_agent
from hud.eval import Taskset, group_relative

agent = create_agent("claude-sonnet-4-5")
job = await Taskset(count_letter(word=w) for w in words).run(agent, group=16)
for runs in job.results.values():
    advantages = group_relative([r.reward for r in runs], normalize_std=True)
    ...  # feed (run.trace_id, adv) into your own GRPO/PPO optimizer
```

## CLI reference (the `hud` command)

```bash
hud init [name] [--preset browser|cua|coding|deepresearch|ml|verilog|...]  # scaffold an env
hud serve [env.py] [-p 8765]            # serve env control channel locally (was: hud dev)
hud eval <source> <agent> [flags]       # run agent over tasks, grade, print reward
hud deploy [--all] [--env K=V]          # build + publish the env to HUD infra in one step
hud sync tasks my-taskset               # publish tasks as a named platform taskset
hud sync env                            # sync environment metadata
hud build .                             # build a portable image of the env
hud task list|start <task>|grade <task> --answer "..."   # attach to a serving/built env
hud jobs [<job-id>] [--json]            # list recent jobs / traces in a job
hud trace <trace-id> [--json]           # inspect a single rollout
hud models list                         # list gateway models
hud set KEY=VALUE                       # persist creds/vars to ~/.hud/.env
hud login / hud version / hud cancel
```

`hud eval` agent names: `claude`, `openai`, `gemini`, `openai_compatible`. Key flags:

```bash
hud eval tasks.py claude                       # first task, one rollout
hud eval tasks.py openai -m gpt-5 --group 3    # pinned model, 3 rollouts each (reward spread)
hud eval tasks.py claude --all                 # every task in the source
hud eval tasks.py claude --full                # whole dataset: --all --auto-respond --max-steps 100
hud eval "My Taskset" claude                   # platform taskset by name → runs remotely
hud eval tasks.py claude --max-steps 30 --max-concurrent 8 --remote --gateway
```

Note: `hud eval` loads tasks from the path you pass and **spawns `env.py` automatically** — don't pass both files. In a split project, point it at `tasks.py` (or `.`).

### Local Docker iteration

```bash
hud build .
docker run -d --name run1 my-env
docker exec run1 hud task start fix_bug              # → the prompt
docker exec run1 hud task grade fix_bug --answer "…"  # → the reward
docker rm -f run1
```

### Deploy + run remotely on the platform

```bash
hud deploy                       # build & register the env (name comes from Environment(...))
hud sync tasks my-taskset
hud eval my-taskset --remote
```

## Common use cases / patterns

- **Run an existing benchmark against a model**: `hud eval "SheetBench-50" claude` (or other published tasksets) and compare via `hud jobs` / hud.ai leaderboards.
- **Coding-agent eval**: `env.workspace(...)` to give an `ssh` sandbox, grade with `BashGrader` running the repo's tests (outcome verification).
- **Browser/computer-use**: `cdp` (browser) or `rfb` (VNC desktop) capabilities; built-in agents auto-wire tools from the manifest — you never wire tools.
- **Generate post-training data**: run `--group N` per task, convert reward spreads to advantages with `group_relative()`, feed into your own RL loop.
- **Starter presets** via `hud init --preset`: `browser`, `cua`, `coding`, `deepresearch`, `ml`, `ml-triage`, `verilog`, `autonomous-businesses`, `gdpval`, `worldsim`, `videogamebench`, `blank`.

## Gotchas & limits

- **Two yields, exactly.** A template must yield the prompt, then yield a float reward in `[0.0, 1.0]`. Everything the agent does happens in the gap.
- **`env` is a name, not a live object.** A `Task` holds the environment *name* as a join key; placement/runtime brings the actual env up. Tasks are portable Pydantic rows (`task.model_dump()` / `Task.model_validate(...)`).
- **Local vs remote eval.** Local `hud eval` needs no API key, no `hud serve`, no Docker (unless the env uses `DockerRuntime`). Platform tasksets (passed by name) default to remote hosted execution.
- **Gateway vs provider keys.** Only `HUD_API_KEY` set → calls route through the gateway. A provider key present → calls hit the provider directly; pass `--gateway` to force gateway routing.
- **Default `--max-steps` is 10** for `hud eval` (raise it for real agentic tasks); `--full` bumps to 100 and auto-responds.
- **Model ids in docs may drift** (e.g. defaults shown as `claude-sonnet-4-6`, `gpt-5.5`, `gemini-3-pro-preview`). Run `hud models list` for the current gateway catalog rather than hardcoding.

### Pricing (from hud.ai, verify before relying)

- **SDK + Platform access:** Free.
- **Cloud:** ~$0.25 / environment-hour (100+ parallel instances, live telemetry, trace analysis).
- **Students/researchers:** $100 free credits with a `.edu` email.
- **Enterprise:** custom — `founders@hud.ai`, book a call at cal.com/jay-hud.

## Links

- Docs: https://docs.hud.ai
- Quickstart: https://docs.hud.ai/quickstart
- CLI reference: https://docs.hud.ai/reference/cli (v6: https://docs.hud.ai/v6/core/cli)
- Agents: https://docs.hud.ai/v6/core/agents · Environments: https://docs.hud.ai/v6/core/environment · Tasks: https://docs.hud.ai/v6/core/tasks · Graders: https://docs.hud.ai/v6/core/graders · Capabilities: https://docs.hud.ai/v6/core/capabilities · Training: https://docs.hud.ai/v6/core/training
- GitHub: https://github.com/hud-evals/hud-python · PyPI: https://pypi.org/project/hud-python/
- Leaderboards: https://hud.ai/leaderboards · Environment templates: https://hud.ai/environments · Models: https://hud.ai/models
- Discord: https://discord.gg/wkjtmHYYjm · YC: https://www.ycombinator.com/companies/hud
