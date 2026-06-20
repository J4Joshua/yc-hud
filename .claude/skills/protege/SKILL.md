---
name: protege
description: Use when sourcing real-world, licensed AI training data (healthcare, video, audio/speech, spatial) via Protege (withprotege.ai) — covers what Protege is, the contact/partnership-led engagement model, and the important fact that there is NO public self-serve SDK or API.
---

# Protege

## What it is

**Protege** (https://withprotege.ai) is a NYC-based **governed data marketplace for AI training data**. It is the "connective tissue" between organizations that hold valuable proprietary/real-world data and AI model builders who need that data to pre-train, post-train, fine-tune, and evaluate models. Protege handles custom curation, de-identification, rights/privacy protection, and quality checks, then securely delivers AI-ready datasets to model builders. Data providers monetize otherwise-underutilized data assets.

Founded 2024 by Bobby Samuels, Travis May (co-founder/former CEO of LiveRamp and Datavant), with CSO Engy Ziedan and CTO Richard Ho. Raised ~$65M total (incl. a $25M Series A in Aug 2025 and a $30M Series A extension led by a16z, Jan 2026). SOC 2 certified.

Catalog scale (per company materials): 100+ data partners; 300,000+ hours of video; 500,000+ hours of audio; billions of clinical notes; hundreds of millions of medical images. Domains span **healthcare, video, audio & speech, spatial & physical**, and other modalities (text, image, voice).

**DataLab at Protege** (https://datalab.withprotege.ai) is a separate research arm focused on data-for-AI research (e.g., clinical AI evaluation, data quality, benchmark contamination), not a developer/API product.

## IMPORTANT: There is no public SDK or API (verified June 2026)

This is the single most important thing to know before writing any code:

- **No public API, SDK, pip/npm package, API keys, or self-serve developer portal exists for Protege (withprotege.ai).** No developer docs, quickstart, or programmatic access pages were found anywhere on the site or the wider web.
- Access is **sales/partnership-led**. Engagement happens through contact forms and a manual follow-up process — datasets are scoped, curated, licensed, and delivered per-engagement, not pulled from an HTTP endpoint with a token.
- **Do NOT conflate with "Protegrity."** Search engines surface `pip install protegrity-developer-python` and `developer.docs.protegrity.com` — that is **Protegrity**, an unrelated data-protection/PII-security company. None of those packages, API keys, or docs belong to Protege/withprotege.ai. Do not use them when a user asks about Protege.
- If a user says they "have a Protege API key," treat that as unverified — confirm with them, because no public API-key program for withprotege.ai was found. It may be a private/bespoke arrangement, in which case ask them for the delivery details (endpoint, auth scheme, data format) their Protege contact provided, since those are not publicly documented.

## When to use this skill

- A user wants to **source licensed, real-world, multimodal training/eval data** (healthcare, video, audio/speech, spatial) and is asking how Protege works or how to get started.
- A user mentions "Protege" / "withprotege.ai" and you need to avoid hallucinating a fake SDK or confusing it with Protegrity.
- A user needs help drafting an access request, scoping a dataset need, or understanding the engagement model.

## Setup & auth

There is no install step and no environment variables, because there is no public SDK/API. The "setup" is a business engagement:

1. Go to https://withprotege.ai/model-builders (for AI developers) or https://withprotege.ai/data-providers (to monetize data).
2. Submit the contact form, or email the team (general contact via https://withprotege.ai/contact). Company address: 169 Madison Ave STE 38206, New York, NY 10016.
3. A Protege team member follows up to scope the dataset (modality, domain, volume, licensing, de-identification needs, delivery format).
4. Data is delivered per the negotiated agreement (format and delivery mechanism are bespoke per engagement, not publicly documented).

If/when Protege provides a private programmatic delivery method (e.g., a signed-URL bucket, SFTP, or a private API), the credentials and schema come directly from your Protege contact — capture them as project secrets at that point. There is no canonical env var name to standardize on.

## Core usage — how an engineer actually proceeds

Since there is no `import protege` to write, the practical "code" is preparing a precise data request and handling delivered data. Below are realistic, non-fabricated patterns.

### 1. Draft a scoped data request (recommended starting artifact)

Write this out before contacting Protege — it makes the sales conversation fast and concrete:

```text
Use case:        e.g., fine-tune a clinical-summarization model
Modality:        text | image | audio | video | spatial
Domain:          healthcare | media/video | audio-speech | spatial-physical
Lifecycle stage: pre-training | post-training | fine-tuning | evaluation/benchmark
Volume needed:   e.g., 50k de-identified clinical notes, or 2k hours speech
Labels/annotations required: yes/no + schema
De-identification / PHI handling: required (HIPAA), specify
Licensing constraints: commercial model training rights, redistribution, retention
Delivery format preference: JSONL / Parquet / WAV+manifest / MP4+manifest
Eval contamination concerns: needs uncontaminated/held-out test set? (DataLab)
```

### 2. After delivery — load common AI-ready formats (generic, not Protege-specific)

Protege delivers standard formats; once you have files, ingest them with normal tooling. Example for a text dataset delivered as JSONL/Parquet:

```python
# Generic ingestion of a delivered dataset — NOT a Protege SDK call.
from datasets import load_dataset  # huggingface `datasets`

ds = load_dataset(
    "parquet",
    data_files={"train": "protege_delivery/train-*.parquet"},
    split="train",
)
print(ds.features, len(ds))
```

```python
# JSONL variant
import json

with open("protege_delivery/notes.jsonl") as f:
    records = [json.loads(line) for line in f]
```

> The exact field schema, file layout, and any manifest format are defined by your specific Protege engagement and are not publicly documented — confirm with your delivery contact.

### 3. If you received a private signed-URL / bucket delivery

Patterns vary; this is the typical shape of a private object-store handoff (placeholder — use the real bucket/creds Protege gives you):

```bash
# Example only — credentials and bucket come from your Protege contact.
aws s3 sync s3://<protege-provided-bucket>/<dataset-id>/ ./protege_delivery/ \
  --endpoint-url <provided-if-any>
```

## Common use cases / patterns

- **Pre-training**: large, diverse, licensed multimodal corpora across industries.
- **Post-training**: supervised (SFT) datasets and human-feedback (RLHF-style) data.
- **Fine-tuning**: domain-specific curated datasets (e.g., clinical, video understanding, speech).
- **Evaluation & benchmarks**: uncontaminated, held-out, real-world test sets — a stated DataLab focus, useful for avoiding train/test contamination.
- **Healthcare AI**: de-identified clinical notes, medical images at scale, with HIPAA-conscious handling — a flagship vertical (https://withprotege.ai/model-builders/healthcare).
- **Data providers**: organizations licensing their proprietary data to earn revenue while keeping rights protections.

## Gotchas & limits

- **No self-serve / no instant access.** Everything goes through scoping + licensing. Plan for a sales cycle, not a `curl` call.
- **No public docs to cite.** Any "Protege SDK/endpoint/quickstart" you see referenced online is almost certainly **Protegrity** (different company) — do not use it.
- **Pricing is not public.** Negotiated per dataset/engagement; no published tiers or rate limits.
- **Licensing terms matter.** Confirm commercial model-training rights, redistribution, and data-retention obligations in writing, especially for regulated (e.g., healthcare/PHI) data.
- **Delivery formats are bespoke.** Don't hard-code assumptions about schema/format; get the manifest spec from your contact.

## Links

- Homepage: https://withprotege.ai
- For model builders (data access): https://withprotege.ai/model-builders
- Healthcare vertical: https://withprotege.ai/model-builders/healthcare
- Video: https://withprotege.ai/model-builders/video
- Audio & speech: https://withprotege.ai/model-builders/audio-speech
- Spatial & physical: https://withprotege.ai/model-builders/spatial
- For data providers: https://withprotege.ai/data-providers
- About / team: https://withprotege.ai/about and https://withprotege.ai/people
- Contact (start here): https://withprotege.ai/contact
- DataLab (research arm): https://datalab.withprotege.ai
- Substack (insights): https://withprotege.substack.com

_Verified via web research June 2026. No developer documentation, SDK, package, or public API for withprotege.ai was found; if this changes, replace the "no SDK" sections with the real install/auth details._
