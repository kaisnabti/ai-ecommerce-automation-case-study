# Custom Webhook Patterns for Zapier

> When a platform has no native Zapier integration, custom webhook steps fill the gap. This documents the pattern used for Printify API integration.

## Why Custom Webhooks?

Printify has **no native Zapier integration** for product creation, publishing, or image upload. The only way to automate the full product lifecycle from within Zapier is through custom HTTP webhook steps (Webhooks by Zapier → Custom Request).

## Webhook Step Configuration

### Standard Headers (all requests)

```
Authorization: Bearer {API_KEY}
Content-Type: application/json
User-Agent: {Brand}-Integration
```

The `User-Agent` header is recommended by Printify's API documentation for identifying integration traffic.

### Step Chain Pattern

```
[Upload Image]  →  [Create Product]  →  [Publish]  →  [Delay]  →  [GET Product]
   POST                POST               POST         10 min        GET
   returns             uses               returns      (async        returns
   image_id            image_id           200 OK        sync)        external.handle
```

### Field Mapping Between Steps

Each webhook step references output fields from previous steps using Zapier's field mapper:

| Step | Input Mapped From | Output Used By |
|---|---|---|
| Upload Image | GPT Image → "Data Url" | Create Product → `print_areas.images.id` |
| Create Product | Upload → `id` | Publish → URL path `/{product_id}/publish` |
| Publish | Create → `id` | Delay (no mapping, just waits) |
| GET Product | Create → `id` | Google Sheets → `external.handle` |

### Error Handling via Filters

A **Filter** step sits between the GPT Image output and the webhook chain. It checks:
- Image data is not empty/null
- QC score from the scoring step meets threshold

If the filter fails, the entire downstream path is skipped — no broken products are created in Printify.

### The Async Pattern

```
POST /publish          →  Returns 200 immediately
                           (Printify queues the Shopify sync)

[10-minute Delay]      →  Zapier pauses execution

GET /products/{id}     →  external.handle is now populated
                           (Shopify URL slug available)
```

**Why 10 minutes?** Printify's sync to Shopify takes up to 10 minutes. A GET request before sync completes returns `external: { handle: null }`. The delay is a pragmatic solution within Zapier's synchronous execution model — there's no native webhook callback or polling mechanism available.

### Dead Steps (Intentional)

The Zap contains several steps from a previous design tool integration that are **still present but disconnected**. No downstream step references their output. They were left in place deliberately:

- Removing them was out of scope for the design generation port
- They serve as documentation of the previous approach
- Deletion risks breaking Zapier's internal step numbering references
- If the client ever wants to revert, the skeleton is there

## Zapier-Specific Gotchas

1. **Caching:** Test runs cache their outputs. A product created during testing weeks ago will show up in field mappers and test previews until you re-test the step.

2. **Data URL size:** GPT Image outputs base64 Data URLs which can be large (200KB+). Zapier handles these in webhook bodies, but they count against the task's data transfer limits.

3. **Step references:** Zapier references steps by internal ID, not position. Deleting a step in the middle doesn't break downstream references to steps after it — but it can break references TO that step.

4. **Path execution:** In a Paths step, only the matching path executes. This means the webhook chain only fires when the filter conditions are met, saving API calls and preventing partial product creation.

---

> *Implementation details adapted for portfolio purposes.*
