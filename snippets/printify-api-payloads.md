# Printify API — Request/Response Examples

> These examples demonstrate the integration pattern for automating Print-on-Demand product creation through Printify's REST API via custom webhooks in Zapier. All IDs, tokens, and values are placeholders.

## 1. Upload Image

```http
POST https://api.printify.com/v1/uploads/images.json
Authorization: Bearer {API_KEY}
Content-Type: application/json
User-Agent: {Brand}-Integration
```

```json
{
  "file_name": "ai-generated-design-001.png",
  "contents": "{BASE64_ENCODED_IMAGE_DATA}"
}
```

**Response:**
```json
{
  "id": "64a1b2c3d4e5f6a7b8c9d0e1",
  "file_name": "ai-generated-design-001.png",
  "height": 1024,
  "width": 1024,
  "size": 245760,
  "mime_type": "image/png",
  "preview_url": "https://images.printify.com/mockup/...",
  "upload_time": "2026-03-15T08:00:00+00:00"
}
```

**Note:** The `contents` field accepts base64-encoded image data directly from the AI image generation step's output. In Zapier, this maps from the upstream step's Data URL output field.

---

## 2. Create Product

```http
POST https://api.printify.com/v1/shops/{SHOP_ID}/products.json
Authorization: Bearer {API_KEY}
Content-Type: application/json
User-Agent: {Brand}-Integration
```

```json
{
  "title": "Abstract Coastal Collection — Ocean Drift",
  "description": "AI-designed wall art print. Premium quality.",
  "blueprint_id": "{BLUEPRINT_ID}",
  "print_provider_id": "{PROVIDER_ID}",
  "variants": [
    { "id": "{VARIANT_1}", "price": "{PRICE_CENTS}", "is_enabled": true },
    { "id": "{VARIANT_2}", "price": "{PRICE_CENTS}", "is_enabled": true },
    { "id": "{VARIANT_3}", "price": "{PRICE_CENTS}", "is_enabled": true }
  ],
  "print_areas": [
    {
      "variant_ids": ["{VARIANT_1}", "{VARIANT_2}", "{VARIANT_3}"],
      "placeholders": [
        {
          "position": "front",
          "images": [
            {
              "id": "{IMAGE_ID_FROM_UPLOAD_STEP}",
              "x": 0.5,
              "y": 0.5,
              "scale": 1,
              "angle": 0
            }
          ]
        }
      ]
    }
  ]
}
```

**Key mappings:**
- `blueprint_id` / `print_provider_id` = selected product type and fulfilment partner (from Printify's catalog API)
- `price` = retail price in cents (set per variant)
- `variants` = size/format combinations enabled for this product
- `print_areas.placeholders.images.id` = the image ID returned by the Upload step

---

## 3. Publish to Shopify

```http
POST https://api.printify.com/v1/shops/{SHOP_ID}/products/{PRODUCT_ID}/publish.json
Authorization: Bearer {API_KEY}
Content-Type: application/json
User-Agent: {Brand}-Integration
```

```json
{
  "title": true,
  "description": true,
  "images": true,
  "variants": true,
  "tags": true,
  "keyFeatures": true,
  "shipping_template": true
}
```

**Important:** This endpoint returns 200 immediately, but the Shopify sync is **asynchronous**. The product's `external.handle` field will be `null` until the sync completes (~10 minutes). You must add a delay before the next step.

---

## 4. Retrieve Published Product (after delay)

```http
GET https://api.printify.com/v1/shops/{SHOP_ID}/products/{PRODUCT_ID}.json
Authorization: Bearer {API_KEY}
User-Agent: {Brand}-Integration
```

**Response (relevant fields):**
```json
{
  "id": "{PRODUCT_ID}",
  "title": "Abstract Coastal Collection — Ocean Drift",
  "external": {
    "id": "8901234567890",
    "handle": "abstract-coastal-collection-ocean-drift"
  },
  "is_locked": false
}
```

The `external.handle` is the Shopify product URL slug. The full URL is: `https://{STORE_DOMAIN}/products/{external.handle}`

This value is then written back to the pipeline data layer to complete the automation loop.

---

## Rate Limits (from Printify public docs)

| Scope | Limit |
|---|---|
| Global | 600 requests/minute |
| Catalog | 100 requests/minute |
| Publishes | 200 per 30 minutes |

---

> *All IDs, tokens, and implementation details adapted for portfolio purposes.*
