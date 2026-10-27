# Delivery Phases

> Six completed phases plus an ongoing discovery engagement. Each phase was scoped and agreed upfront before work began.

---

## Phase 1: Automation Audit & Consolidation

### Problem

The client had accumulated **60+ individual automations** over the course of building their e-commerce stack. Many were duplicates or near-duplicates created during experimentation. There was no documentation, no naming convention, and no clear picture of what was active, what was redundant, and what was broken. The system worked — mostly — but was fragile, and the client couldn't confidently change anything without risking a cascade of failures.

### Approach

1. **Full inventory** — catalogued every automation by trigger, action steps, and connections. Mapped which sheets, APIs, and platforms each one touched.
2. **Dependency mapping** — identified which automations fed into others (e.g., one writing a row that triggered another). Surfaced circular dependencies and race conditions.
3. **Redundancy analysis** — flagged duplicates (same trigger + same actions), near-duplicates (same intent, slightly different configuration), and orphans (automations that no longer connected to anything).
4. **Pipeline design** — proposed a 7-stage linear pipeline with defined status transitions, replacing the ad-hoc web of automations.

### Outcome

Consolidated **60+ automations into a 7-stage pipeline** with clear status progression:

```
Queued → Scoring → Approved → Publishing → Live → Content → Distributed → Tracked
```

Each stage had a single responsible automation, a defined input status, and a defined output status. The pipeline became predictable, debuggable, and documentable. This architecture became the foundation for every subsequent phase.

### What I Demonstrated

- Systems thinking: seeing the whole before optimising the parts
- Audit methodology: systematic inventory and dependency mapping at scale
- Architecture design: replacing organic growth with intentional structure
- Client communication: translating a messy technical audit into clear recommendations

---

## Phase 2: Scoring Engine & Pipeline Expansion

### Problem

Every product idea entered the pipeline with equal priority. There was no quality gate — a weak product concept consumed the same automation resources as a strong one. The client wanted an automated way to evaluate ideas before committing them to the full pipeline.

### Approach

1. **Scoring criteria design** — worked with the client to define what makes a viable product (trend relevance, market demand signals, competition level, margin potential).
2. **AI scoring step** — built an AI evaluation step that assesses each product idea against the criteria and assigns a numeric score (1–10) plus a verdict: PROCEED, REVISE, or REJECT.
3. **Threshold gate** — products scoring above the threshold auto-advance to Approved; below-threshold items are flagged for manual review or rejection.
4. **Pipeline expansion** — added the Scoring and Approved stages to the pipeline, with the scoring engine bridging Queued → Scoring and the approval gate bridging Scoring → Approved.

### Outcome

Automated go/no-go decisions on product ideas. The pipeline no longer wasted downstream resources (content generation, social publishing) on weak products. The scoring criteria were tuneable — the client could adjust thresholds without touching the automation.

### What I Demonstrated

- AI integration for business logic (not just content generation)
- Designing tuneable systems — parameters the client can adjust without developer involvement
- Pipeline architecture expansion that preserved the existing structure

---

## Phase 3: Content Engine & Social Publishing

### Problem

Social media content was entirely manual. When a product went live on Shopify, someone had to write the post copy, format it for each platform, and manually publish. This created a bottleneck — products would sit live with no social presence.

### Approach

1. **Content generation engine** — an AI step that takes product data (name, description, URL) and generates platform-appropriate social media content: captions, hashtags, and call-to-action text.
2. **Content formatter** — a second step that adapts the generated content for each target platform's requirements (character limits, hashtag strategies, link formatting).
3. **Social scheduling integration** — connected the formatted content to a social media scheduling platform for automated distribution to Facebook, Instagram, and Pinterest.
4. **Auth issue resolution** — during testing, a direct Facebook post step was causing persistent authentication errors. After investigation, the step was redundant — the scheduling platform already handles Facebook distribution. Agreed with the client to remove the step rather than debug an unnecessary integration.

### Outcome

Products automatically receive generated social content and get published across three platforms. The pipeline responsibility ends at "Distributed" status — the scheduling platform manages its own posting cadence. This boundary was explicitly documented to prevent future scope confusion about post timing.

### What I Demonstrated

- Multi-platform content automation with AI
- Pragmatic engineering: removing a broken step that duplicated existing functionality rather than spending hours debugging it
- Clear boundary definition between the pipeline's responsibility and third-party platform behaviour

---

## Phase 4: Full System Documentation

### Problem

Zero documentation existed. The entire system — automation configurations, sheet structures, API connections, status flows, platform credentials, and troubleshooting knowledge — lived in the client's head and in scattered configurations. No one else could maintain, debug, or extend the system.

### Approach

1. **Automation-by-automation documentation** — every workflow documented with its trigger, each action step, field mappings, filter conditions, and error handling.
2. **Sheet architecture** — documented every sheet in the master workbook: column definitions, status values, formulas, and cross-sheet references.
3. **Integration reference** — API endpoints, authentication methods, rate limits, and known quirks for each connected platform.
4. **Status flow diagrams** — visual representation of product lifecycle through the pipeline.
5. **Troubleshooting guide** — common failure modes, how to diagnose them, and how to fix them.

### Outcome

A complete handover package that made the system maintainable by someone other than the original builder. This proved its value immediately — the documentation became the reference for all subsequent phases and the client's own monitoring work.

### What I Demonstrated

- Technical writing: translating complex multi-platform automation into clear documentation
- System-level thinking: documenting not just individual components but their interactions
- Professional delivery: producing a maintainable handover, not just working code

---

## Phase 5: Print-on-Demand API Integration

### Problem

The client's Print-on-Demand wall art products needed to flow from AI-generated designs into their Shopify storefront via Printify. The critical constraint: **Printify has no native Zapier integration for product creation**. Every other integration in the pipeline used built-in app connectors, but Printify required custom API work.

### Approach

Built a custom webhook chain that replicates the full Printify product creation workflow:

1. **Image upload** — `POST /v1/uploads/images.json` with the AI-generated artwork (base64 data URL from the image generation step)
2. **Product creation** — `POST /v1/shops/{shop_id}/products.json` with the uploaded image mapped into print areas, blueprint, print provider, and size/colour variants
3. **Shopify publish** — `POST /v1/shops/{shop_id}/products/{id}/publish.json`
4. **Timed delay** — critical: Printify's publish operation is **asynchronous**. The external handle field (Shopify product URL) is `null` until Printify completes its sync to Shopify, which takes up to ~10 minutes
5. **Product retrieval** — `GET /v1/shops/{shop_id}/products/{id}.json` to fetch the now-populated external handle (the real Shopify URL)
6. **Pipeline insertion** — writes the product data and Shopify URL to both the affiliate database and the pipeline sheet with status "Live"

### Technical Challenges Solved

**Async publishing pattern** — the most subtle problem. Without the delay, the GET request returns a null external handle and the pipeline writes an empty URL. This was not documented in any obvious way — discovered through testing.

**Platform caching** — products created during test runs on old dates get cached by Zapier's internal system. This stale data persisted for weeks, causing confusion during debugging. Fix: re-test individual steps to force cache refresh.

**No native integration** — every Printify call required custom webhook configuration: URL construction, authorization headers, JSON body mapping, and response field extraction. This is significantly more complex than using built-in app steps.

### Outcome

Fully automated Print-on-Demand pipeline: an AI-generated design goes from image data to a live Shopify product listing with correct variants and a real product URL — all without manual intervention. POD products enter the main pipeline at "Live" status, skipping the scoring and affiliate gates (since POD products have no affiliate links and are fulfilled directly by the print provider).

### What I Demonstrated

- REST API integration without native tooling support
- Async pattern handling (understanding that "published" doesn't mean "synced")
- Debugging platform-specific quirks (caching behaviour)

---

## Phase 6: AI Design Generation Port

### Problem

The design generation step was completely dead. Two separate failures had converged:

1. **Previous design integration broken** — the export flow from the design tool was failing with authentication and permission errors. The tool's API returned edit URLs but not direct image URLs, and the export pipeline required to get direct URLs had stopped working.
2. **Image model deprecated** — the DALL-E model was deprecated on the OpenAI API. Only GPT Image models (gpt-image-1) were available on the client's account.

The net result: no automated design generation was happening. The entire product pipeline was stalled.

### Approach

**Decision: abandon the previous design tool entirely.** The export issue was in the tool's API permissions, not in the automation configuration. Debugging it would mean working through their support process with no guaranteed timeline. Meanwhile, GPT Image could generate artwork directly — no intermediate design platform needed.

**Ported to OpenAI GPT Image (gpt-image-1):** Replaced the broken steps with a single GPT Image step that generates artwork directly from concept text.

### Critical Prompt Engineering Problems

**Problem 1 — Mockup vs. Artwork:**
The original prompt generated **photographs of products** — lifestyle mockups showing a framed print on a wall or in a room setting. This looks great in a marketing image. But the image was being sent to Printify as the **print artwork**. The result would be a wall art print of a photo of wall art hanging on a wall — a print-within-a-print.

The fix required understanding the production pipeline: the image isn't a marketing photo, it's the actual graphic that gets printed. The prompt had to specify:
- "Design artwork ONLY — do NOT show a frame, mockup, room setting, or product photo"
- "Flat artwork on a plain solid white background"
- "Suitable for high-resolution print production"

**Problem 2 — Concept Mashing:**
The upstream AI step generates multiple design concepts as options. Without explicit instruction, GPT Image attempted to combine all concepts into a single image — producing incoherent, cluttered designs. The fix was explicit selection:

> "Use ONLY the first concept. Disregard all other concepts entirely."

This instruction had to come **before** the concept list in the prompt. Placing it after (as a closing instruction) was inconsistently followed by the model.

### Production Prompt Structure (Adapted)

```
Use ONLY the first concept listed below.
Disregard all other concepts entirely.

Concept list: {upstream_concept_output}

Create ONE single wall art design based on the first concept.

REQUIREMENTS:
- ONE design only, centered in the image
- Design artwork ONLY — no frame, no mockup, no room scene, no product photo
- Flat artwork on a plain solid white background
- High contrast and visually striking at print scale
- No watermarks or text overlays
- Style: modern, clean, suitable for high-resolution print production
```

### Handling Legacy Steps

The dead design-tool steps were **left in the automation deliberately**. Nothing downstream consumed their output — the new GPT Image step fed directly into the Printify upload. Removing them was out of scope for this phase (design port only, not full cleanup), and deleting steps from a working automation carries risk. They were documented as dormant for a future cleanup phase.

### Outcome

A fully working daily automation: scheduled trigger fires concept generation → AI scores concepts → GPT Image generates print-ready artwork → Printify creates the product → Shopify lists it → pipeline tracks it through to social distribution. Verified live with a real product run that produced a fresh, unique design, published to Shopify, and processed through the pipeline to "Distributed" status.

### What I Demonstrated

- Prompt engineering for physical production constraints (not just text/content use cases)
- Rapid platform migration with zero downtime to the client
- Understanding the full production chain: knowing *where* in the physical process an image lands changes what the image must be
- Pragmatic scope management: fixing the dead steps, leaving cleanup for a future scoped phase

---

## Discovery Phase: Data Quality Audit (Partial — Ongoing)

### Context

After Phase 6, the client flagged issues with the affiliate product database — duplicates, placeholder links, and test data mixed with production data. I proposed a **Discovery & Architecture Audit** as a formal scoped engagement: audit the data, review the client's existing monitoring tools, and produce an architecture recommendation for the next build phase (affiliate link automation).

### Data Audit

Analysed **1,200+ data rows** in the affiliate product database. Findings were severe (see [data-audit.md](data-audit.md) for full methodology and results):

- **Over 65% of rows** had no affiliate link at all
- **Dozens of product names** appeared more than once (worst offender: 30+ duplicates of a single product)
- **~100 rows** had empty product names
- **Header text** duplicated as data rows
- **Placeholder affiliate IDs** present in URLs (literal template strings, not real IDs)
- **Numerous rows** contained API error messages logged as product data
- **One column header** contained hallucinated AI content (an encyclopaedia-style paragraph instead of a column name)
- **Under 10%** of all rows were actual, valid affiliate redirect links

### Affiliate Link Research

A key concern was whether the affiliate network's API tier would gate link generation (which would substantially increase the scope of the next build). Research revealed that **affiliate promolinks are deterministic URL construction**:

```
https://{affiliate_platform}/redir/{PRODUCT_ID}/{AFFILIATE_ID}/{CAMPAIGN_KEY}
```

No API call is needed to generate an affiliate link — the URL pattern is predictable from the product ID and affiliate ID. This significantly simplified the architecture for the next build phase.

### What I Demonstrated

- Data quality audit at scale (1,200+ rows, multiple dimensions of analysis)
- Research that reduces project scope (deterministic link discovery eliminated API-tier dependency)
- Structured discovery process: audit → findings → architecture recommendation → scoped quote

---

> *Implementation details in this case study have been adapted for portfolio purposes and do not represent any single client engagement.*
