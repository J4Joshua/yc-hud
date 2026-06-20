---
name: hillclimb
description: Use when asked about Hillclimb (hillclimb.com / hillclimb.ing, YC F25) — an early-stage company selling RL-environment training data to frontier labs. Covers what they do, founders/contact, and the verified fact that there is NO public API, SDK, or self-serve product (yet).
---

# Hillclimb

## What it is

**Hillclimb** (https://www.hillclimb.com, also reachable as hillclimb.ing which 301-redirects to hillclimb.com) is an early-stage AI training-data company. Its tagline is **"Training data for recursive self-improvement"** and its meta description is **"A virtual lab where AI can continuously experiment and learn to become research scientists."**

Hillclimb's stated mission is to improve model capabilities toward recursive self-improvement by solving two problems:
- **Aggregating all human research data**
- **Automating RL (reinforcement-learning) environment creation**

Business model: they **build RL environments and expert-derived training data and sell it to frontier AI labs** ("To sustain these ambitions, we sell the data we make to frontier labs"). They assemble elite human talent — IMO medalists, Putnam Top 50 competitors, Lean formalization experts, PhDs — to design environments and capture their problem-solving processes as training data. They position contributors as "partners, not gig workers" (offering co-authorship), and pitch tight iteration loops ("improvements in minutes, not months") versus traditional dataset revision cycles.

Track record: in a prior venture the founders used the same approach for math, building a community of IMO medalists to co-train a state-of-the-art math-competition model with **Nous Research** (announced via @NousResearch on X). Backed by Tier 1 VCs, Paul Graham, and angels.

Company facts (verified via Y Combinator):
- **Batch:** Y Combinator Fall 2025 (F25)
- **Status:** Active, ~4 people, San Francisco
- **Founders:** Jun Park (ex-DeepMind research engineer; Georgia Tech; former pro esports) and Ibrakhim Ustelbay (founder of Headstarter)
- **Contact:** founders@hillclimb.ing
- **Socials:** X @hillclimbai, LinkedIn /company/hillclimb

## When to use this skill

Use this when someone references "Hillclimb" in the context of AI training data, RL environments, or YC F25 — to correctly explain what the company does and to set accurate expectations.

**Do NOT use it to write code against a "Hillclimb SDK/API" — none exists publicly.** If a different "Hillclimb" is meant (e.g. *Hill Climb Racing* the mobile game, iRacing hillclimb, or generic hill-climbing optimization algorithms), this skill does not apply.

## Setup & auth — NONE AVAILABLE (verified 2026-06-20)

There is **no self-serve product, no public API, no SDK, no developer docs, and no signup flow**. This was verified directly:

- The homepage (https://www.hillclimb.com/) is a **single static landing page** (plain HTML/CSS, Geist + EB Garamond fonts, behind Cloudflare). It contains only positioning copy, a list of open roles, and a copy-to-clipboard contact email. No login, dashboard, or product links.
- Probed paths `/docs`, `/api`, `/developers`, `/app`, `/login`, `/signup` all return **404**.
- The GitHub user `github.com/hillclimb` has **0 public repos / 0 packages** and is not the company org; no Hillclimb org or published packages were found.
- No package exists on PyPI or npm under the Hillclimb name (searches returned unrelated AI SDKs).

So there is **no `pip install` / `npm install`, no `HILLCLIMB_API_KEY`, and no client to initialize.** Do not invent any of these.

**The only "integration" is human/sales:** email **founders@hillclimb.ing** to discuss a data partnership. Engagements are bespoke (custom RL environments / datasets delivered to a lab), not a programmatic endpoint.

## Core SDK / API usage

Not applicable — there is no SDK or API as of this writing (verified 2026-06-20). The "interface" is a sales conversation. A realistic first step for an engineer at a frontier lab:

```text
To: founders@hillclimb.ing
Subject: Data / RL-environment partnership inquiry

We're [lab/team]. We're interested in [math / domain X] RL environments and
expert-derived training data. Our use case: [post-training / evals / agentic RL].
Volume / cadence: [...]. Can we set up a call to scope a pilot?
```

If/when Hillclimb ships a programmatic product, the things to look for (and verify before coding) would be: a docs site, a published `pip`/`npm` package, an API base URL, and an API-key env var. None of these are confirmed today — treat any such snippet as unverified until you can load it from their docs.

## Common use cases / patterns

What Hillclimb is actually good for (per their own positioning):

- **Post-training data for frontier LLMs** — RL environments and expert-grade datasets, starting in **math/competition reasoning** (IMO/Putnam-level), with stated ambition to generalize to broader research domains.
- **Co-training collaborations** — direct collaboration between a lab's researchers and Hillclimb's human experts, with fast feedback loops and co-authorship for high-impact contributors (the Nous Research math model is the reference example).
- **Automated RL environment creation** — a core R&D thrust; environments are a deliverable sold to labs, not (yet) a self-serve toolkit.

Integration pattern today = **buyer is a frontier lab; delivery is bespoke data/environments via a partnership**, not an API call.

## Gotchas & limits

- **No public API/SDK/docs/pricing** (verified). Anyone claiming a Hillclimb client library or `HILLCLIMB_API_KEY` is mistaken or hallucinating — push back.
- **Name collisions are heavy.** Web/code searches for "hillclimb" surface *Hill Climb Racing* (mobile game), iRacing hillclimb, FreeClimb (unrelated comms API), and generic "hill climbing" optimization repos/notebooks. Filter these out.
- **Two domains, one site:** hillclimb.ing 301-redirects to hillclimb.com; both serve the same static landing page. The contact email uses the `.ing` domain (founders@hillclimb.ing).
- **WebFetch is blocked (403) by Cloudflare** on the homepage; use `curl` with a browser User-Agent if you need to re-verify the page contents.
- **Customer profile is narrow:** the product is aimed at frontier AI labs doing post-training, not general app developers.
- **Stage risk:** ~4-person F25 company; offering, scope, and any future product surface can change quickly. Re-verify before relying on anything here.

## Links

- Homepage: https://www.hillclimb.com/ (also https://hillclimb.ing → redirects here)
- YC company profile: https://www.ycombinator.com/companies/hillclimb
- YC launch post: https://www.ycombinator.com/launches/Oqr-hillclimb-training-data-derived-from-human-superintelligence
- Nous Research model collaboration (referenced on homepage): https://x.com/NousResearch/status/1998536543565127968
- X / Twitter: https://x.com/hillclimbai
- LinkedIn: https://www.linkedin.com/company/hillclimb
- Contact: founders@hillclimb.ing
