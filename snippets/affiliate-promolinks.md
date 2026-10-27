# Affiliate Promolink Construction

## The Discovery

Initial assumption: generating affiliate links for products would require API calls, and the affiliate platform's API tier might gate access to link-generation endpoints. If true, the integration would have been significantly more complex to build.

**Research finding:** The affiliate platform's promolinks are **deterministic URL construction** — no API call needed.

This single discovery fundamentally changed the architecture of the next build phase.

## Promolink Pattern

```
https://{checkout_platform}/redir/{PRODUCT_ID}/{AFFILIATE_ID}/{CAMPAIGN_KEY}
```

| Component | Source | Example |
|---|---|---|
| `PRODUCT_ID` | Platform marketplace listing | `123456` |
| `AFFILIATE_ID` | Client's affiliate account | `XXXXXX` |
| `CAMPAIGN_KEY` | Optional campaign tracking | `default` |

## Implementation in Zapier

Since the URL is deterministic, generating it requires only **string concatenation** in a Formatter step:

```
Step: Formatter by Zapier → Text → Concatenate

Input 1 (static): https://{checkout_platform}/redir/
Input 2 (mapped): {Product ID from sheet}
Input 3 (static): /
Input 4 (static): {AFFILIATE_ID}
Input 5 (static): /
Input 6 (static or mapped): {CAMPAIGN_KEY}

Output: Complete affiliate promolink
```

No authentication, no API key, no rate limits, no tier restrictions.

## Why This Matters Architecturally

**Without this discovery:**
- Would need platform API integration (custom webhooks + auth)
- Would need to research and potentially upgrade the client's API tier
- Would need error handling for API failures
- Would need to handle rate limits
- Build estimate: significantly higher complexity

**With this discovery:**
- Simple string operation in Zapier (Formatter step)
- Zero external API calls
- Zero authentication requirements
- Zero rate limit concerns
- Build estimate: minimal — just a Formatter step + write to sheet

## Data Quality Context

The audit of the product database (1,200+ rows) revealed why this automation matters:

- The majority of rows (~65%) had no affiliate link at all
- ~10% had placeholder links with unfilled template parameters
- ~3% had platform page URLs (not affiliate redirect links)
- Only ~7% had actual working affiliate redirect links

Automating promolink construction from the product ID would immediately fix the missing links and prevent future gaps — as long as the product IDs correspond to valid platform listings.

## Validation

The only check needed: does the `PRODUCT_ID` correspond to an active listing? This can be validated with a simple HTTP HEAD request to the constructed URL — a 200/302 response confirms the product exists; a 404 flags an invalid ID.

---

> *Implementation details — including specific platform names, URLs, and data figures — adapted for portfolio purposes.*
