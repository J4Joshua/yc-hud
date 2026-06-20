# PRD — Thermal-Seeking Rover as a Verifiable Navigation Environment

**Event:** HUD (W25) × YC RSI Hackathon · San Francisco, YC HQ · June 20–21, 2026 (~25h hacking, Sat 12:30 PM → Sun 1:30 PM)
**Theme:** RL environments, post-training datasets, evals, RFT — *"you can improve models at anything you can verify."*
**Status (as of June 19, T-1 day):** Jetson flashed and boots; **RGB camera (IMX219) working on the Jetson** ✓. Rover does **not** drive from the Jetson yet; thermal (MLX90640) tested on an ESP32 but not yet streamed to the Jetson. No closed-loop autonomy yet. → Remaining hardware bring-up (Jetson→rover drive, thermal→Jetson) is the pre-event priority; the submission is designed so it does **not depend** on the rover driving.

---

## 1. One-liner
A verifiable navigation environment where we **teach a vision-language-action (VLA) model to drive a real rover toward a person** — trained from demonstrations in simulation, deployed live on a Jetson, and scored on a public leaderboard that proves the learned policy beats its baselines.

## 2. Core insight (what makes this not generic)
**Anything you can verify, you can teach — and "reach the target without colliding" is trivially verifiable.** Most VLA work is robot arms doing tabletop manipulation; almost nobody teaches a VLA to *navigate*, and almost nobody brings *real hardware* to a hackathon. We do both, on a task with an unambiguous reward.

The original project's "sensor-disagreement" insight survives as the **P2 finale**: feed the VLA a thermal channel alongside RGB so it learns to drive toward a person it **cannot see on camera** (obscured by a barrier) — the obscured-survivor case, now expressed as a *learned, verified* behavior rather than hand-coded logic.

## 3. Scope boundary (say this in the pitch)
We do **close-range engagement** — get to a person and confirm reach — not wide-area search. And we are building **the environment + eval that teaches the skill**, not a general-purpose robot brain. *"Drones search, ground robots engage. We don't build the foundation model — we build the verifiable task that improves one."*

## 4. Users / audience
- **Judges (primary):** HUD/YC evaluators asking *"what did you teach a model, and how do you verify it?"* — and rewarding ambition, working demos, and clear communication (grand prize is a YC interview).
- **First-responder framing (narrative):** USAR teams who need close-up confirmation a person is reachable/alive before committing rescuers.

## 5. Goals & non-goals
**Goals**
- Build a MuJoCo navigation environment with a camera the agent sees through, plus a verifiable reward (reached target, no collision).
- Generate a demonstration dataset from a scripted expert and **fine-tune a small VLA (SmolVLA)** to drive toward a person from vision.
- Produce a **HUD leaderboard** comparing random vs. scripted-expert vs. fine-tuned VLA across **unseen layouts** (the generalization claim).
- **Stretch differentiator:** deploy the policy on the Jetson and drive the **real rover** toward a real person, live.
- **P2 finale:** add thermal as a second input so the rover reaches a person hidden from the RGB camera.

**Non-goals (explicitly out of scope)**
- No general-purpose "any task / any environment" agent — generality fights verifiability; we stay narrow on purpose.
- No training a VLA from scratch — we *fine-tune* an existing small one.
- No on-device large/foundation models — SmolVLA (450M) is the ceiling that fits the Jetson.
- No SLAM / no map-building — reactive navigation only.
- No real rescue/manipulation; no flying. Ground rover only.

## 6. Priority rings (never break an inner ring for an outer one)
- **P0 — the floor, needs NO working rover:** MuJoCo env + verifiable reward + scripted-expert demo dataset + a fine-tuned VLA that works *in sim* + a HUD leaderboard beating baselines. This alone is a complete, on-theme submission.
- **P1 — the differentiator:** the fine-tuned VLA (or the hand-coded controller as fallback) running on the Jetson, driving the **real rover** toward a person, live, fully offline.
- **P2 — the finale (only if ahead):** thermal as a second VLA input → reaches a person the RGB camera can't see. Optional Claude-generated calming speech on arrival.

## 7. Demo promise (win condition)
Judges see a **leaderboard**: the fine-tuned VLA outscores a random policy and the hand-coded expert on success rate, time-to-target, and collisions — in layouts it never trained on. Then they watch the **real rover run that learned policy and drive to a person**, with Wi-Fi killed to prove it runs locally. (If hardware doesn't land: the live rover step becomes a recorded clip + the sim rollout, and the submission still stands on the leaderboard.)

## 8. Success criteria (quantified)
- **P0:** fine-tuned VLA reaches the target in **≥80% of N≥20 seeded sim episodes**, measurably beating random and matching/beating the scripted expert on at least one metric (e.g. collisions), on **held-out layouts** — all visible on a HUD leaderboard.
- **P1:** ≥3 successful live real-rover runs reaching a person from ~2–3 m, stopping before contact, with the network disabled.
- **P2:** ≥1 live run reaching a person hidden behind a visual barrier using the thermal channel.
- **Pitch:** a 3-minute story that lands the "teach a verifiable skill, prove it, run it on real hardware" arc, with the leaderboard and the rover both on screen.

## 9. Team & roles
Work divides into three tracks plus you:
- **You — hardware owner + product/vision + demo + pitch + dashboard.** You own the rover and the sim-to-real bridge.
- **Model/imitation track:** demo generation + SmolVLA fine-tuning (LeRobot / Hugging Face).
- **Infra/eval track:** Modal training jobs + the HUD leaderboard.
- **Env/dashboard support:** the MuJoCo environment and the live dashboard.

One owner per track; integrate early and often so the sim, training, and eval pieces meet well before the final hours.

## 10. Sponsor map
| Sponsor / tool | Role in the project | Tier |
|---|---|---|
| **HUD** (host) | Environment wrapper + eval **leaderboard** (the judged artifact); beta `robot` capability fits VLA policies | Core |
| **Modal** | GPU (A100) to fine-tune SmolVLA (~2–4 h) and run batch sim rollouts | Core |
| **MuJoCo** (Google DeepMind) | The actual simulator — renders the camera, simulates the rover; ties loosely to the DeepMind sponsor | Core (tool) |
| **Anthropic / Claude** | Claude Code as the team's build assistant; optional calming-speech text on arrival (P2) | Support / optional |
| **Fireworks** | Alternative for hosting/fine-tuning open models or later RFT, if Modal+LeRobot isn't enough | Optional |
| **MiniMax** | Optional text-to-speech for the calming-message audio (P2) | Optional |
| **Antim Labs** | Optional scene/asset generation (Gizmo) for richer sim rooms — **off the critical path**, alpha access is email-gated | Bonus only |

---

> **Phase 2** (post-hackathon personal learn-by-rebuilding effort) is tracked separately and deliberately kept out of this hackathon doc.
