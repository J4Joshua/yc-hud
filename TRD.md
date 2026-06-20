# TRD — Thermal-Seeking Rover / Verifiable VLA Navigation

**Companion to `PRD.md`.** Event: HUD × YC RSI Hackathon, June 20–21 2026, ~25h (Sat 12:30 PM → Sun 1:30 PM).
**Honest status (June 19):** Jetson flashed; **IMX219 RGB camera working on the Jetson** ✓; rover not driving from the Jetson yet; thermal proven on ESP32 but not yet streamed to the Jetson; no autonomy loop yet.

---

## 1. Architecture — two decoupled sides

The key design decision: **split the project so the judged artifact needs no working rover.**

**A) Sim side — the floor (no hardware required):**
```
MuJoCo env (camera + rover physics)
   → scripted expert drives to target, logs (image, instruction, action) → demo dataset
   → fine-tune SmolVLA on Modal
   → evaluate in sim → HUD leaderboard (random vs expert vs VLA, on unseen layouts)
```
Everything here runs in the cloud + a laptop. If the rover never moves, this is still a complete submission.

**B) Hardware side — the differentiator (at risk):**
```
Jetson runs the policy (VLA, or hand-coded controller as fallback)
   → USB-serial → ESP32 (WAVE ROVER) → motors
   → real rover drives toward a real person, offline
```
Two control loops live here, carried over from the original design:
- **Fast reflex loop (local, offline, several Hz):** `policy.act(observation) → drive`, behind one `Policy` interface so the VLA and the hand-coded controller are swappable. **Obstacle hard-stop sits ABOVE the policy and can always override it** — a learned net can never disable the safety stop.
- **Slow supervisor loop (~1 Hz, optional):** logging, dashboard, optional Claude speech on arrival. Never blocks the reflex loop; degrades gracefully with no network.

## 2. Hardware BOM + status + power
| Part | Role | Status / note |
|---|---|---|
| Jetson Orin Nano Super (8GB) | Brain: VLA inference, policy loop | **Flashed, boots.** Not yet talking to sensors/motors |
| ESP32-S3 + WAVE ROVER 4WD | Motors, motor driver, sensor co-processor | Rover **not driving from Jetson yet**; drive via serial JSON to the onboard ESP32 |
| MLX90640 thermal (I2C 0x33, 3.3V) | Heat detection (P2 input) | Works on ESP32; **not yet streamed to Jetson** |
| IMX219 CSI camera | RGB vision (primary VLA input) | **Working on the Jetson** (already tested) ✓ |
| MPU6050 IMU (I2C 0x68) | Orientation / crude collision | On ESP32 |
| **Power** | — | **18650s not procured.** Plan: **buy high-drain 18650s in SF** (Amazon same-day / battery shop) — keeps design unchanged. Wall power for all bench testing. Tethered demo acceptable as fallback. |

## 3. The pipeline in detail

**Step 1 — MuJoCo environment.** A room with obstacles, a wheeled (differential-drive) rover, and a person-target. Observation = RGB camera render (+ thermal channel in P2). Action = 2-D continuous `(linear_v, angular_w)` (SmolVLA's continuous head supports this directly). Reward = +reach target, −collision, −time. **Domain randomization** (layout, target position, lighting, obstacle count, sensor noise) is the sim-to-real lever and the source of the "unseen layout" generalization claim.

**Step 2 — Expert + demo dataset.** A scripted expert with *privileged* access to the target position in sim (your hand-coded controller logic) drives to the target repeatedly. Log ~**50 episodes** of `(camera image, instruction="drive to the person", action)` in **LeRobotDataset** format. This is the **post-training dataset** (on-theme).

**Step 3 — Fine-tune SmolVLA.** `lerobot-train --policy.path=lerobot/smolvla_base ...` on a **Modal A100** (~2–4 h for 20k steps; ~50 episodes; keep the action space tiny). Output = a navigation VLA.

**Step 4 — Deploy on Jetson (P1).** Run SmolVLA inference on the Orin Nano — **measured ~7 Hz, ~2 GB RAM** in BF16, fits the ~5GB budget. Wrap it behind the `Policy` interface; the hand-coded controller is the swappable fallback. Send actions over USB-serial to the rover.

**Step 5 — Eval on HUD.** Wrap the env via HUD's `robot` capability (or a thin MCP capability + a two-yield Python grader for reward = task success). Run a **leaderboard** comparing random vs. scripted-expert vs. fine-tuned VLA on success rate / time-to-target / collisions across held-out seeds. **This is the judged artifact.**

**Step 6 — Thermal upgrade (P2).** Add the 32×24 thermal as a second camera input (render it as an image); retrain; show the rover reaching a person hidden behind a visual barrier the RGB camera can't see.

## 4. Sponsor / tool stack
- **HUD** — env wrapper + eval leaderboard. *Caveat:* the `robot` capability is **beta and thinly documented** (wire protocol `openpi/0`); budget time to read source/cookbook, and verify how to register a non-LLM (random/scripted) policy as a baseline. Minimal qualifying use = 1 env + a taskset + a published 3-policy leaderboard.
- **Modal** — fine-tuning + batch rollouts on A100. Use event-provided credits.
- **MuJoCo** — simulator; free, fast, installs in minutes, renders RGB.
- **Anthropic/Claude** — Claude Code as build assistant; optional arrival-speech text.
- **Antim Labs** — optional scene assets only; alpha access is email-gated (email *today* if you want it), never a dependency.

## 5. Event hour budget (~25h, parallel tracks)
Tracks map to the team's split. If a track is short-staffed, collapse to the **HW + Sim** essentials and cut P2.

| Window (Sat→Sun) | You (HW + demo) | Model track | Infra/eval track | Checkpoint |
|---|---|---|---|---|
| 12:30–16:00 | Jetson↔ESP32 drive + thermal→Jetson | MuJoCo env + expert + start logging demos | Modal + HUD accounts, eval skeleton | Env smoke-tested by 16:00 |
| 16:00–19:00 | Teleop rover from Jetson | Demo dataset done; first fine-tune submitted | HUD `robot`/MCP wrapper working | First VLA checkpoint training by 19:00 |
| 19:00–23:00 | Policy interface + safety stop on Jetson | Fine-tune iterate; sim eval | Leaderboard: random vs expert vs VLA | **P0 DONE: leaderboard live** |
| 23:00–03:00 | Deploy VLA on Jetson; first real runs | Support sim-to-real; domain-gap fixes | Held-out-layout eval | First real-rover run (P1) |
| 03:00–06:00 | Harden real runs OR thermal P2 | Thermal-input retrain (P2) | Eval the thermal policy | P2 attempt or P1 hardening |
| 06:00–10:00 | Repeat live runs; record backup video | Final checkpoint pinned | Final leaderboard frozen | Demo de-risked + recorded |
| 10:00–13:30 | **Frozen** — dashboard + pitch + rehearsal | Pitch support | Pitch support | No new code after ~11:00 |

## 6. Pre-event critical path (TODAY, June 19)
Highest-leverage, highest-risk. In order:
1. **Jetson drives the rover** via serial JSON to the WAVE ROVER ESP32 (teleop) — the top remaining hardware risk. 2. **Thermal streaming** ESP32→Jetson over serial. 3. **Buy 18650s in SF.** 4. Smoke-test a trivial MuJoCo render so the env track can start instantly at the event. *(RGB camera already works on the Jetson — done.)*
**Gate:** if (1) doesn't land tonight, walk in committed to the **sim-only submission** and treat the rover as a teleoperated showpiece + recorded clip. Decide this tonight, not on the demo floor.

## 7. Risks + fallback ladder
1. **Hardware integration doesn't finish (NEW #1).** Rover/sensors/Jetson never fully bridge. → *Mitigation:* the entire P0 needs no rover; sim+leaderboard stands alone. Live rover degrades to teleop + recorded clip.
2. **VLA nav transfer fails.** SmolVLA is pretrained on manipulation arms, not driving; differential-drive nav is novel/unproven. → *Mitigation:* keep action space 2-D, unfreeze the VLM, in-domain demos; **hand-coded controller is the swappable fallback** for the live run, and the leaderboard still shows the VLA's sim performance.
3. **Brownout** — Jetson resets under motor-current spikes. → Correct buck converter to exactly 5.0V (multimeter), bulk capacitance; fix power, don't suppress the symptom.
4. **8GB memory pressure** — VLA + camera contend. → SmolVLA measured ~2GB; keep RGB-only first; measure before adding thermal.
5. **HUD robot API is beta** — sparse docs. → Fallback to the simpler MCP-capability + Python grader path; the leaderboard is what matters, not the transport.
6. **Scope creep / coordination** — a team pulling in different directions. → Freeze scope at P0; one owner per track; integrate early so the pieces meet before the final hours.

**Fallback ladder (always have the next rung):** VLA on real rover → hand-coded controller on real rover → teleop rover + sim rollout on screen → recorded video + live leaderboard.

## 8. Tradeoffs to defend (interview/judge prep)
- **Fine-tune SmolVLA over training a VLA from scratch** — scratch needs millions of trajectories; we post-train a 450M model on ~50 demos.
- **Narrow + verifiable over general** — generality fights the reward; "reach the person, don't collide" is unambiguously gradeable. On-theme by construction.
- **MuJoCo over Antim Labs/Gizmo** — Gizmo is a scene-asset generator, not a sensor-sim; the real sim work lives in MuJoCo anyway.
- **Imitation/post-training over RL-from-scratch** — behavior-cloning from a scripted expert is achievable in 25h; RFT (HUD supports GRPO/PPO) is a later upgrade, not the backbone.
- **Jetson + small VLA over cloud VLA** — keeps the reflex loop offline (Wi-Fi-kill demo); cloud latency would break real-time control.
- **RGB-first, thermal second** — leverages SmolVLA's visual pretraining to de-risk, then thermal revives the obscured-survivor insight as a learned behavior.
- **Ground rover vs drone** — proximity, endurance, can-enter-voids, steady enough for the camera; concedes wide-area search to drones.
