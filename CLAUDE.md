# yc-hud

Working repo for the **HUD (W25) × YC RSI Hackathon** (SF, YC HQ, June 20–21 2026).

**Project:** a *verifiable navigation environment* — teach a vision-language-action (VLA)
model (SmolVLA) to drive a real rover toward a person. Trained from demonstrations in a
MuJoCo sim, deployed live on a Jetson Orin Nano, and scored on a public **HUD leaderboard**
that proves the learned policy beats its baselines (random / scripted-expert).

The submission is deliberately split so the **judged artifact needs no working rover**:
- **P0 (the floor):** MuJoCo env + verifiable reward + scripted-expert demo dataset +
  fine-tuned SmolVLA working in sim + HUD leaderboard. Complete on its own.
- **P1 (differentiator):** run the policy on the Jetson, drive the real rover to a person,
  offline (Wi-Fi killed).
- **P2 (finale):** add a thermal channel so the rover reaches a person the RGB camera can't see.

## Read these first
- **[PRD.md](PRD.md)** — product requirements: one-liner, scope boundary, goals/non-goals,
  priority rings (P0/P1/P2), success criteria, team roles, sponsor map.
- **[TRD.md](TRD.md)** — technical design: architecture (sim side vs hardware side), hardware
  BOM + status, the training/eval pipeline, hour-by-hour event budget, risks + fallback ladder.

**Honest status (June 19):** Jetson flashed & boots; IMX219 RGB camera working on the Jetson ✓;
rover not yet driving from the Jetson; thermal proven on ESP32 but not yet streamed to the Jetson;
no autonomy loop yet.

## Sponsor skills
`.claude/skills/` holds research-backed Claude skills for the hackathon sponsors, with SDK docs,
auth/env vars, real code, and gotchas — enough to start a use case once an API key is in hand:

| Skill | Role in this project | Key/auth |
|---|---|---|
| **hud** | Env wrapper + the eval leaderboard (the judged artifact) | `HUD_API_KEY` |
| **modal** | A100 GPU for SmolVLA fine-tuning + batch sim rollouts | `MODAL_TOKEN_ID`/`_SECRET` |
| **fireworks** | Optional: host/fine-tune open models, later RFT | `FIREWORKS_API_KEY` |
| **daytona** | Sandboxes (general; not on the critical path) | `DAYTONA_API_KEY` |
| **antimlabs** | Optional Gizmo scene/asset gen — off the critical path | `GIZMO_API_KEY` (own convention) |
| **protege** | Training-data marketplace — **no public SDK/API** (partnership-led) | — none |
| **hillclimb** | RL-environment training data for labs — **no public SDK/API** | — none |

Core stack also uses **MuJoCo** (simulator), **LeRobot/SmolVLA** (the VLA + `lerobot-train`),
and the **Jetson Orin Nano + WAVE ROVER (ESP32-S3)** for the hardware side.

## Conventions
- Keep the action space tiny: 2-D continuous `(linear_v, angular_w)`.
- Safety: the obstacle hard-stop sits **above** the policy and can always override it — a learned
  net can never disable the safety stop. The hand-coded controller is the swappable fallback behind
  one `Policy` interface.
- Never break an inner priority ring (P0) to chase an outer one (P1/P2).
