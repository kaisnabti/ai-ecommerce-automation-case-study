# Technical Reference

> API integration patterns, prompt engineering, async handling, and platform-specific gotchas encountered across the build.

---

## Printify API Integration Patterns

Printify has **no native Zapier integration** for product creation. The entire image-upload → product-create → publish → verify flow was built with custom webhook steps calling the Printify REST API directly.

### Endpoints

| Action | Method | Endpoint |
|---|---|---|
| Upload image | `POST` | `/v1/uploads/images.json` |
| Create product | `POST` | `/v1/shops/{SHOP_ID}/products.json` |
| Publish to Shopify | `POST` | `/v1/shops/{SHOP_ID}/products/{id}/publish.json` |
| Retrieve product (get Shopify URL) | `GET` | `/v1/shops/{SHOP_ID}/products/{id}.json` |

### Required Headers

```
Authorization: Bearer {API_KEY}
Content-Type: application/json
User-Agent: {Brand}-Integration
```

The `User-Agent` header is a best practice for identifying integration traffic in Printify's logs and avoids being flagged by generic bot filters.

### Rate Limits

| Scope | Limit |
|---|---|
| Global | 600 requests / minute |
| Catalog | 100 requests / minute |
| Publishes | 200 per 30 minutes |

These limits were comfortably within our throughput (one product per daily run), but they become relevant when batch-processing back-catalogues or running recovery scripts.

---

## Async Publishing Pattern

Publishing a Printify product to Shopify is **asynchronous**. The `POST .../publish.json` call returns immediately, but the product does not appear on Shopify right away. The `external.handle` field on the product object — which contains the Shopify product URL — remains `null` until Printify completes the sync, typically around 10 minutes.

### The Problem

A `GET` request issued immediately after `POST publish` returns:

```json
{
  "id": "{PRODUCT_ID}",
  "external": {
    "id": null,
    "handle": null
  }
}
```

Without the external handle, the pipeline cannot write the Shopify product URL to the data layer, and downstream steps (content generation, social publishing) have no product link to reference.

### The Solution

```
POST /v1/shops/{SHOP_ID}/products/{id}/publish.json
        │
        ▼
   Delay (10 minutes)
        │
        ▼
GET /v1/shops/{SHOP_ID}/products/{id}.json
        │
        ▼
   Extract external.handle → write to Google Sheets
```

A fixed 10-minute delay was inserted between the publish call and the retrieval call. This is a pragmatic solution within Zapier's synchronous execution model — there is no webhook callback from Printify when the sync completes, so polling after a known interval is the simplest reliable approach.

### Generalised Pattern

This async-publish-then-poll pattern applies to any API integration through a synchronous orchestrator where:

1. The write operation returns before the side effect (sync to a third-party system) completes.
2. The third-party system does not offer a completion webhook.
3. The orchestrator (Zapier, Make, n8n) does not support native long-polling or await semantics.

The fixed delay is a trade-off: it adds latency to every run, but avoids the complexity of a separate polling loop or a webhook receiver.

---

## GPT Image Prompt Engineering

### Context

The original design generation step used a third-party design tool integration. That integration failed due to authentication and permission errors in its export flow, and the underlying image model was subsequently deprecated on the API. The step was rebuilt using OpenAI's GPT Image model (`gpt-image-1`).

### Model Configuration

| Parameter | Value |
|---|---|
| Model | `gpt-image-1` |
| Size | 1024 × 1024 |
| Quality | HD |
| Style | Natural |
| Output | Data URL (base64-encoded image) |

### Production Prompt

```
From the concept list below, use ONLY Concept 1. Ignore Concept 2
and Concept 3 entirely.

Concept list: {previous_step_output}

Create ONE single wall art design piece based on Concept 1.

IMPORTANT REQUIREMENTS:
- ONE design only, centered in the image
- Artwork ONLY — do NOT show a frame, room mockup, interior scene,
  or product photo
- High-resolution graphic on a clean solid white background
- Bold composition with strong visual impact
- No watermarks, no borders
- Style: modern, print-ready, suitable for large-format
  giclée or digital printing
```

### Engineering Decisions

Each line in the prompt addresses a specific failure mode discovered during testing:

| Instruction | Failure Without It | Why It Matters |
|---|---|---|
| `use ONLY Concept 1. Ignore Concept 2 and Concept 3 entirely.` | Model mashes all upstream concepts into one incoherent image | The upstream AI step generates multiple ranked concepts. Without explicit selection, the image model treats the full list as a single brief. |
| `Artwork ONLY — do NOT show a frame, room mockup, interior scene, or product photo` | Model generates a **photo of framed art on a wall** — a lifestyle mockup, not print-ready artwork | Sending this output to the print provider produces a print-of-a-photo-of-a-print. The model defaults to generating product/room mockups when "wall art" appears in the prompt. |
| `High-resolution graphic on a clean solid white background` | Design appears on textured, coloured, or gradient backgrounds | Print production transfers the entire image. A non-white background prints as a visible rectangle on the final product or requires manual masking. |
| `Bold composition with strong visual impact` | Fine details, thin lines, subtle gradients | Print resolution and viewing distance constraints mean intricate details can be lost at final output size. |
| `ONE design only, centered in the image` | Multiple design variants tiled in one image | The Printify upload expects a single artwork file mapped to a single print area. Multiple designs in one image mean partial prints on the finished product. |

### Output Integration

The GPT Image step outputs a **Data URL** (base64-encoded image string). This feeds directly into the Printify image upload endpoint's `contents` field without any intermediate file storage or URL hosting — a significant simplification over the original design tool flow, which required an export-and-poll sequence to obtain a direct image URL.

---

## Webhook Configuration in Zapier

### Why Custom Webhooks

Printify's Zapier integration supports triggers (order events, product updates) but **does not expose actions for product creation, image upload, or publishing**. Every write operation required a custom "Webhooks by Zapier" step configured manually with endpoint, method, headers, and JSON body.

### Field Mapping Chain

The full data flow from AI-generated artwork to live Shopify product involves a chain of webhook steps, each consuming output from the previous:

```
[GPT Image]
  └─ Data URL ──────────────────────────────┐
                                             ▼
[Webhook: Upload Image to Printify]  ←── contents: {Data URL}
  └─ Image ID ──────────────────────────────┐
                                             ▼
[Webhook: Create Product in Printify] ←── print_areas.front: {Image ID}
  └─ Product ID ────────────────────────────┐
                                             ▼
[Webhook: Publish to Shopify]         ←── product_id: {Product ID}
  └─ (async — no immediate output)          │
                                             ▼
[Delay: 10 minutes]                         │
                                             ▼
[Webhook: GET Product]                ←── product_id: {Product ID}
  └─ external.handle ──────────────────────┐
                                            ▼
[Google Sheets: Write to Pipeline]    ←── Product URL: {external.handle}
                                            Status: "Live"
```

### Error Handling

Filter steps are placed before the webhook chain to gate execution:

- If the GPT Image step returns an empty Data URL → path is skipped entirely.
- If the Printify upload returns no image ID → downstream product creation does not fire.
- If the external handle is still null after the delay → the Sheets write contains an empty URL (flagged for manual review rather than crashing the Zap).

This "fail-safe, don't fail-loud" approach keeps the automation running for the next scheduled trigger rather than halting the entire Zap on a single bad run.

---

## Affiliate Link Pattern

### The Discovery

During the discovery phase, a key concern was whether the client's affiliate platform account tier would gate access to link-generation endpoints. If an API call were required for each link, the build would need authenticated API integration, error handling, and potentially a tier upgrade — significantly increasing the scope.

**Research revealed that this platform's affiliate links are deterministic URL construction, not API calls.**

### URL Pattern

```
https://{AFFILIATE_PLATFORM}/redirect/{PRODUCT_ID}/{AFFILIATE_ID}/{CAMPAIGN_KEY}
```

| Parameter | Source |
|---|---|
| `PRODUCT_ID` | From the affiliate product catalogue (known per product) |
| `AFFILIATE_ID` | The client's unique affiliate ID (static, stored once) |
| `CAMPAIGN_KEY` | Optional campaign tracking parameter |

### Implementation

This reduced the affiliate link automation to **string concatenation** — no API authentication, no rate limits, no tier concerns. In Zapier, this is a simple Formatter step that concatenates the base URL with the product ID and affiliate ID from the Google Sheet.

### Impact on Scope

This single discovery significantly reduced the scope of the next phase. What was initially estimated as a complex API integration became a templated string operation.

---

## Zapier Caching Gotcha

### The Problem

Products created during Zapier **test runs** (using the "Test" button on individual steps) are cached by Zapier's internal data layer. This cache is keyed by step + timestamp, and once populated, the cached output persists for **weeks** — even after the upstream data has changed.

In practice: an image generated during an earlier test run continued appearing as the "output" of the GPT Image step in later production runs, because Zapier was serving the cached test result instead of executing the step fresh.

### The Fix

Re-test every step in the Zap after making changes. Zapier's cache invalidates when a step is manually re-tested, replacing the stale output with fresh data.

### The Lesson

When debugging unexpected output in a Zapier automation:

1. **Check whether the "wrong" data matches a previous test run's output** — if it does, it's a cache issue, not a logic issue.
2. **Re-test ALL upstream steps**, not just the step you modified. The cache is per-step, so a change to one step does not invalidate earlier steps' cached output.
3. **Never trust Zapier's test data for production validation.** Always verify with a real triggered run after re-testing.

---

## Key Technical Takeaways

1. **When a native integration doesn't exist, custom webhooks are a viable production solution** — not just a prototype hack. The Printify webhook chain has been running in production without intervention.

2. **Async APIs through synchronous orchestrators require explicit delay/poll patterns.** The 10-minute delay is inelegant but reliable. A more sophisticated approach (webhook receiver, or a separate polling Zap) adds complexity without proportional benefit at this throughput level.

3. **Prompt engineering for physical production is fundamentally different from prompt engineering for digital display.** Every constraint of the manufacturing process (print method, substrate, viewing distance) must be encoded in the prompt. The model has no awareness of downstream physical constraints.

4. **Research the actual API behaviour before scoping.** The affiliate link discovery (deterministic URLs, no API needed) significantly simplified the architecture. A scope based on the assumption of API integration would have been far larger and harder to deliver.

---

*Implementation details adapted for portfolio purposes and do not represent any single client engagement.*
