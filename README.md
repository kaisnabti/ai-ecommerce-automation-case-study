# AI-Powered E-Commerce Automation System — Case Study

> End-to-end automation pipeline for a Print-on-Demand e-commerce brand: from trend research and AI-generated product design through to Shopify publishing, social media distribution, and performance analytics.

## Overview

A solo e-commerce operator had built a functional but fragile automation stack across Zapier, Google Sheets, Shopify, Printify, and multiple social media platforms. The system had grown organically — 60+ individual automations, manual copy-paste workflows, broken integrations, and no documentation. I was brought in as an external automation consultant to audit, consolidate, and extend the system into a production-grade pipeline.

Over six delivery phases (plus an ongoing discovery engagement), I:

- **Audited and consolidated** 60+ automations into a clean 7-stage pipeline with defined status progression
- **Designed a scoring engine** that automatically evaluates product ideas before they enter the pipeline
- **Built a content engine** with multi-platform social media publishing (Facebook, Instagram, Pinterest)
- **Integrated Printify's API** via custom webhooks (no native Zapier support exists) for automated Print-on-Demand fulfilment
- **Ported AI design generation** from a deprecated image API to a current model, solving critical prompt engineering problems for print-ready artwork
- **Produced full system documentation** as a handover package
- **Conducted a data quality audit** of 1,200+ product rows, uncovering systemic data integrity issues

## Role & Positioning

This project demonstrates two complementary skill sets:

**AI/Automation Engineering** — API integration (REST, webhooks, async patterns), prompt engineering for generative AI (image generation with print-production constraints), pipeline architecture, data quality analysis, and systems thinking across a multi-platform stack.

**Solutions Consulting** — Scope definition and defense, client management (including scope-creep pattern recognition), phased delivery with clear boundaries, written audit deliverables, and translating between business intent and technical reality.

## Tech Stack

| Layer | Tools |
|---|---|
| Orchestration | Zapier (custom webhooks, paths, filters, delays) |
| Data | Google Sheets (pipeline state, scoring, content, analytics) |
| E-commerce | Shopify (storefront, product management) |
| Fulfilment | Printify API + print provider (Print-on-Demand) |
| AI/Design | OpenAI GPT Image (gpt-image-1) for product artwork generation |
| Social | Social media scheduling platform (Facebook, Instagram, Pinterest) |
| Affiliate | Affiliate network (deterministic promolink construction — no API needed) |

## Engagement Summary

| Phase | Scope | Key Outcome |
|---|---|---|
| 1 | Automation audit & consolidation | 60+ automations → 7-stage pipeline |
| 2 | Scoring engine + pipeline expansion | Automated product evaluation gate |
| 3 | Content engine + social publishing | Multi-platform content generation and distribution |
| 4 | Full system documentation | Complete handover package |
| 5 | Printify API integration | Automated POD product creation via custom webhooks |
| 6 | AI design generation port | Deprecated model → GPT Image with print-ready prompt engineering |
| Discovery | Data quality audit | 1,200+ row audit with categorised findings |

## Repository Structure

```
case-study/
├── README.md                          # This file
├── docs/
│   ├── architecture.md                # Pipeline architecture & data flow
│   ├── phases.md                      # Detailed phase-by-phase delivery
│   ├── technical-reference.md         # API patterns, prompt engineering, async handling
│   ├── data-audit.md                  # Data quality audit methodology & findings
│   └── lessons-learned.md            # Scope management, client handling, technical pitfalls
└── snippets/
    ├── printify-api-payloads.md       # Sanitized API request/response examples
    ├── gpt-image-prompt.md            # Production prompt with engineering rationale
    ├── webhook-config.md              # Custom webhook patterns for Zapier
    └── affiliate-promolinks.md        # Affiliate link construction pattern
```

## Key Technical Highlights

**Custom API Integration Without Native Support** — Printify has no native Zapier integration for product creation. Built the entire create → upload → publish → verify flow using custom webhook steps, handling async publishing (external handle is null for ~10 minutes post-publish) with timed delays and polling.

**Prompt Engineering for Physical Production** — The original AI image prompt generated photos of products (mockups). Printing a mockup image onto a physical product would produce a product-on-a-product. Rewrote the prompt to generate flat graphic artwork on a solid white background, suitable for direct-to-surface printing. Also solved concept-mashing (model combining multiple input concepts into one image) with explicit selection instructions.

**Data Quality at Scale** — Audited 1,200+ product rows and found the majority missing affiliate links, widespread duplicate product names, API error messages logged as product data, AI-hallucinated content in column headers, and placeholder URLs with unfilled template parameters.

**Scope Defense as a Deliverable** — Client exhibited a pattern of scope-creep-via-context: sending long emails with system inventories framed as "context" that actually constituted requests for free architectural review. Developed a response framework: acknowledge as context, explicitly mark out-of-scope items, and quote same-day.

---

> **Disclaimer:** This case study describes generalised patterns from automation consulting work. Specific implementation details have been adapted for portfolio purposes and do not represent any single client engagement.
