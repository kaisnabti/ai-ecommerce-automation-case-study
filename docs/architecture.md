# Pipeline Architecture

> How 60+ disconnected automations became a 7-stage pipeline with defined status progression, a shared data layer, and two distinct product entry paths.

## System Overview

The system orchestrates an AI-powered e-commerce pipeline that takes a product idea from initial concept through to social media publication and performance analysis — with no manual intervention once an idea enters the queue.

```
┌─────────────────────────────────────────────────────────────────┐
│                        MASTER PIPELINE                          │
│                                                                 │
│  Queued → Scoring → Approved → Publishing → Live                │
│                                               │                 │
│                                        Content ← ─ ─ ┘         │
│                                            │                    │
│                                       Distributed               │
│                                            │                    │
│                                         Tracked                 │
└─────────────────────────────────────────────────────────────────┘
```

**Seven statuses, strict linear progression.** Every product passes through each status exactly once. No status is ever skipped for affiliate products. POD (Print-on-Demand) products enter at a defined bypass point (see below).

---

## Automation Responsibilities

Each automation owns exactly one status transition. This was the core design decision from the audit phase: one automation, one job, one transition.

| Function | Transition | Description |
|----------|------------|-------------|
| **Scoring Engine** | Queued → Scoring | AI evaluates incoming product ideas against weighted criteria (market fit, margin potential, trend alignment). Outputs a numeric score. |
| **Approval Gate** | Scoring → Approved | Threshold-based auto-approval. Products scoring above the configured threshold advance automatically; below-threshold items are held for manual review. |
| **Affiliate Link Gate** | Approved → Publishing | Verifies that a valid affiliate link exists for the product before allowing it to proceed. Products without links are held — preventing broken CTAs in the storefront. |
| **Storefront Publisher** | Publishing → Live | Creates or syncs the product into the Shopify storefront. Writes the resulting product URL back to the data layer. |
| **Content Engine** | Live → Content | Generates social media copy using AI: platform-specific captions, hashtags, and descriptions based on the product data and its live Shopify URL. |
| **Content Formatter** | Content → Distributed | Formats raw AI output into platform-specific post structures and sends to a social media scheduling platform for distribution across Facebook, Instagram, and Pinterest. |
| **Analytics Collector** | Distributed → Tracked | Collects performance metrics post-publication and writes them back to the tracking sheets. |

```
  Scoring    Approval   Link Gate   Publisher   Content    Formatter   Analytics
  Engine      Gate                                Engine
    │           │           │           │          │           │           │
    ▼           ▼           ▼           ▼          ▼           ▼           ▼
 ┌──────┐  ┌───────┐  ┌────────┐  ┌────────┐  ┌──────┐  ┌────────────┐  ┌───────┐
 │Queued│─▶│Scoring│─▶│Approved│─▶│Publish │─▶│ Live │─▶│  Content   │─▶│Distri-│─▶│Tracked│
 └──────┘  └───────┘  └────────┘  │  -ing  │  │      │  │            │  │ buted │  └───────┘
                                  └────────┘  └──────┘  └────────────┘  └───────┘
```

---

## Data Layer

The data layer is built entirely on Google Sheets, functioning as both state store and inter-automation communication bus.

### Pipeline Workbook (Primary)

| Sheet | Purpose |
|-------|---------|
| **Product Pipeline** | The live pipeline worksheet. Every product has a row; the `Status` column drives which automation picks it up next. Single source of truth for pipeline state. |
| **Scoring Results** | Stores scoring criteria, weights, and per-product evaluation results. |
| **Content Output** | Holds AI-generated content before formatting. |
| **Platform Content** | Platform-specific formatted content ready for publishing. |
| **Publishing Queue** | Tracks what has been sent to the social scheduling platform. |
| **Performance Data** | Raw performance data collected by the analytics automation. |
| **Analytics Dashboard** | Aggregated metrics and trend data. |

### Affiliate Database (Separate Workbook)

A standalone Google Sheet — **not a tab inside the Pipeline workbook** — that serves as the product and affiliate link database. Contains product names, platform, affiliate links, pricing, and status flags. This sheet is the data source for the Affiliate Link Gate and the target for POD product registration.

> **Architecture note:** Keeping this as a separate workbook was a client decision for access-control reasons. It introduces a cross-workbook dependency that must be accounted for in any future migration.

---

## Two Product Entry Paths

The pipeline supports two distinct product types, each with its own entry point.

### Path 1: Affiliate Products (Standard Flow)

Affiliate products (sourced from partner networks) enter at **Queued** and pass through every status in sequence. The Affiliate Link Gate is critical for these — it prevents products from reaching the storefront without a working affiliate link.

```
  Idea sourced ──▶ Queued ──▶ Scoring ──▶ Approved ──▶ Publishing ──▶ ...full pipeline
                     │
                Enters via manual
                entry or import automation
```

### Path 2: Print-on-Demand Products (Bypass Entry)

POD products (wall art and home decor prints created via the AI design pipeline) enter at **Live**, bypassing scoring, approval, and the affiliate link gate entirely. This is correct by design: POD products have no affiliate links (Printify handles fulfilment directly via Shopify), and they are already "live" the moment they sync from Printify to Shopify.

```
  AI design pipeline ──▶ Printify ──▶ Shopify ──▶ Live ──▶ ...content & social pipeline
                                                    │
                                               Enters pipeline
                                               at this status
```

---

## Trend-to-Product Automator: The POD Sub-Pipeline

The Trend-to-Product automation is a self-contained sub-pipeline that feeds into the master pipeline at the Live stage. It runs daily on a schedule.

### Step-by-Step Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│  TREND-TO-PRODUCT AUTOMATOR (Daily Scheduled Run)                    │
│                                                                      │
│  1. ⏰ Scheduled Trigger (daily)                                     │
│       │                                                              │
│  2. 🤖 AI Concept Generator                                         │
│       │  Generates 3 seasonal design concepts with descriptions      │
│       │                                                              │
│  3. 🤖 AI Quality Control                                           │
│       │  Scores each concept 1–10, gates: PROCEED / REVISE / REJECT │
│       │                                                              │
│  4. 🎨 GPT Image Generation (gpt-image-1)                           │
│       │  Generates print-ready artwork from top-scoring concept      │
│       │  (Replaced previous design tool integration that failed)     │
│       │                                                              │
│  5. 🔀 Filter + Path Split                                          │
│       │                                                              │
│  6. Path A: Printify → Shopify → Pipeline                           │
│       ├── POST: Upload image to Printify (custom webhook)            │
│       ├── POST: Create product in Printify (custom webhook)          │
│       ├── POST: Publish product to Shopify (custom webhook)          │
│       ├── ⏳ 10-minute delay (async sync wait)                       │
│       ├── GET: Retrieve product (webhook) → external handle = URL    │
│       ├── ✏️ Write to Affiliate Database                             │
│       └── ✏️ Write to Product Pipeline → Status: "Live"              │
│                                                                      │
│  ──▶ Product now enters the master pipeline at Live                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Why Custom Webhooks?

Printify has **no native Zapier integration** for product creation. The entire upload → create → publish → verify flow is built using Zapier's generic webhook steps with manually configured:

- Endpoint URLs (Printify REST API v1)
- Authorization headers (Bearer token)
- JSON payloads (image upload, product creation with variants, publish triggers)

This is a non-trivial integration — see [Technical Reference](technical-reference.md) for the full API payload patterns.

### The 10-Minute Delay

Printify's publish-to-Shopify operation is **asynchronous**. After triggering a publish, the product's `external` handle field (which contains the Shopify product URL) remains `null` until the sync completes — typically up to 10 minutes. The delay step is mandatory; without it, the GET returns a null URL that propagates through the rest of the pipeline.

---

## Cross-System Integration Map

```
                    ┌─────────────┐
                    │   Zapier    │
                    │ (7 pipeline │
                    │ automations │
                    │ + product   │
                    │ automator)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
      ┌──────────┐  ┌──────────┐  ┌──────────┐
      │  Google  │  │  OpenAI  │  │ Printify │
      │  Sheets  │  │ GPT Image│  │   API    │
      │ (data)   │  │ (design) │  │ (POD)    │
      └──────────┘  └──────────┘  └─────┬────┘
                                        │
                                        ▼
                                  ┌──────────┐
                                  │ Shopify  │
                                  │(store)   │
                                  └─────┬────┘
                                        │
                                        ▼
                                  ┌──────────┐
                                  │ Social   │
                                  │Scheduling│
                                  │Platform  │
                                  └──────────┘
                                        │
                              ┌─────────┼─────────┐
                              ▼         ▼         ▼
                          Facebook  Instagram  Pinterest
```

### Key Integration Notes

- **Zapier → Google Sheets:** Bidirectional. Automations read status to determine what to process; write status after completing their transition.
- **Zapier → OpenAI:** Unidirectional. AI steps generate content and images; results flow downstream only.
- **Zapier → Printify:** Custom webhooks (POST/GET). No native integration exists.
- **Printify → Shopify:** Managed by Printify's own sync. The 10-minute delay accounts for this.
- **Zapier → Social Platform:** Unidirectional. Content is pushed; the platform manages its own publishing schedule.
- **Social Platform → Social Channels:** Managed by the scheduling platform. The pipeline's responsibility ends at "Distributed" status.

---

> *Implementation details adapted for portfolio purposes.*

*Next: [Delivery Phases →](phases.md)*
