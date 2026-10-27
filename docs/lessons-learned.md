# Lessons Learned

> Patterns, pitfalls, and principles from building and consulting on a multi-platform e-commerce automation system over six delivery phases.

---

## Technical Lessons

### 1. Mockup vs Artwork for Print-on-Demand

AI image generators default to generating **photos of products** — a framed print on a wall, a canvas on a desk. For print-on-demand, the printer takes your image and applies it directly to the surface. If your image is a photo of a framed print, the result is a print of a photo of a print.

**The fix:** Always specify "flat graphic design artwork only" with explicit negative constraints: "do NOT show a frame, mockup, room setting, or product photo." Add "plain solid white background" so the design isolates cleanly for print production.

**Takeaway:** When AI output feeds a physical production process, the prompt must encode the production constraints — not just the creative intent.

### 2. Concept Isolation in Image Prompts

The pipeline generates multiple product concepts per run, then sends them to an image model for artwork. Without explicit instruction, the model attempts to combine all concepts into a single image — producing incoherent mashups.

**The fix:** Prefix the prompt with "use ONLY Concept 1. Ignore Concept 2 and Concept 3 entirely." The explicit ignore instruction matters — without it, even "use Concept 1" still results in bleed-through from the other concepts.

**Takeaway:** When passing structured multi-option input to a generative model, assume it will try to use everything unless explicitly told not to. Negative instructions ("ignore X") are as important as positive ones.

### 3. Async APIs in Synchronous Orchestrators

Printify's publish endpoint returns immediately with a success response, but the actual sync to Shopify takes approximately 10 minutes. During that window, the `external.handle` field (the Shopify product URL) is `null`. Any downstream step that reads the product immediately after publishing gets nothing.

**The fix:** Insert a 10-minute Delay step between the publish call and the GET request that retrieves the Shopify URL. This is a blunt but reliable solution within Zapier's synchronous execution model.

**Takeaway:** This pattern appears in any async API integration through a synchronous orchestrator. The general solution: `POST (trigger) → Delay (estimated sync time) → GET (verify result)`. If the orchestrator supports webhooks from the upstream service, a callback-based approach is cleaner — but Zapier's webhook support for inbound events from third-party APIs is limited.

### 4. Zapier's Internal Caching

Products created during Zapier test runs get cached internally. That cached data — including images, field values, and timestamps from the test — can persist in production runs for **weeks**. In one case, a test image from an earlier month was appearing in live production runs weeks later.

**The fix:** After modifying any step in a Zap, re-test ALL upstream steps to force Zapier to refresh its internal cache. Testing only the modified step is not sufficient — stale data from untouched upstream steps will propagate.

**Takeaway:** Zapier's caching is invisible and undocumented. Treat any test run as potentially contaminating the cache, and build "cache-bust" re-testing into your deployment checklist.

### 5. Third-Party API Fragility

A third-party design tool's API returned view/edit URLs, not direct image URLs. Getting a downloadable image required a multi-step async export process that was permission-dependent and fragile. The integration failed with access errors that proved unresolvable within the project's constraints.

**The fix:** Abandoned the design tool entirely and replaced it with a direct AI image generation step (OpenAI GPT Image). The replacement was more reliable, faster, and eliminated a dependency.

**Takeaway:** When a third-party integration becomes the bottleneck, evaluate whether the integration is worth maintaining versus replacing the entire approach. Sometimes the cleanest fix is removing the dependency, not debugging it.

### 6. No Native Integration ≠ Impossible

Printify has **no native Zapier integration** for product creation. Many automation builders would stop here and report it as a blocker. But Printify has a well-documented REST API, and Zapier supports custom webhook steps.

**The fix:** Built the entire image upload → product creation → Shopify publish → URL retrieval flow using custom webhook steps with proper auth headers and JSON payloads. The result is indistinguishable from a native integration from the pipeline's perspective.

**Takeaway:** "No Zapier integration" means "no drag-and-drop option." If the service has an API, custom webhooks can replicate any native integration — it just requires understanding the API directly rather than relying on pre-built connectors.

---

## Consulting & Scope Management Lessons

### 1. Scope Creep via Context

The client developed a pattern: sending long, detailed emails listing systems, agents, and architectural observations, framed as "sharing context" or "aligning before the next phase." These emails were genuinely informative — but they also constituted implicit requests for free architectural review.

**Defence framework:**
1. Acknowledge the observations as valuable context
2. Explicitly state which items fall outside the current scope
3. Offer to quote the out-of-scope items as a separate engagement — same day, so the boundary is clear while the conversation is fresh

**Takeaway:** Context-sharing and scope-expanding often look identical in email. The difference is whether you're expected to act on the information. Make the boundary explicit in writing every time.

### 2. Written Scope is Your Shield

The client occasionally reframed history — "my understanding was always that this was included" or "this should have been caught in the original audit." These weren't malicious; they reflected genuine confusion about what was agreed. But without written scope, the consultant is always on the back foot.

**Practice:** Every phase had a written scope document sent before payment. When reframing occurred, the response was: "Here's what we agreed in writing on [date]." The conversation always resolved immediately.

**Takeaway:** Scope documents aren't bureaucracy — they're the single source of truth for what was promised and what was delivered. Every claim in every email must be defensible against the written scope.

### 3. Phased Delivery with Small Milestones

The full engagement was broken into small, self-contained phases. This structure created multiple benefits:

- **Lower risk for the client:** No single commitment was large enough to feel like a gamble
- **Clear scope boundaries:** Each phase had defined deliverables, so "done" was unambiguous
- **Payment before work:** Every phase was paid upfront, eliminating payment-chase overhead
- **Natural re-scoping checkpoints:** Between phases, requirements could be reassessed without disrupting in-flight work

**Takeaway:** Small phases with upfront payment are better than large retainers or hourly billing for both sides. The client gets predictability; the consultant gets payment certainty and clean boundaries.

### 4. Credit Discovery Fees Against the Build

The Discovery & Architecture Audit was offered as a credit against the subsequent build phase if the client proceeded. This made the discovery effectively free for a client who continues, while ensuring the consultant is paid for analysis work if they don't.

**Takeaway:** Discovery/audit pricing that credits forward removes the client's objection ("why am I paying for planning?") while protecting the consultant's time. It also creates natural momentum toward the build phase.

### 5. Don't Audit What You Weren't Asked To

The client's Zapier account contained far more automations than those in scope. The audit covered only the specific Zaps the client identified. Auditing the full account would have been more thorough — but it would also have been unscoped, unpaid, and created ambiguity about what was "included."

**Takeaway:** Unsolicited auditing is free work that creates scope ambiguity. Audit exactly what was agreed. If you spot issues outside scope, note them briefly and offer to quote separately.

### 6. Frame Cleanup as Foundation-Building

The client built his original systems while learning — they "worked" but had fundamental issues (duplicate Zaps, no error handling, test data mixed with production). The framing mattered: "building on solid foundations before adding further automation" landed well. "Your systems are broken and need fixing" would have been adversarial.

**Takeaway:** Technical debt created by a non-technical client who built something themselves is a sensitive topic. Frame remediation as investment in the next phase, not criticism of the last one.

### 7. Adapt to Client Communication Preferences

The client expressed anxiety about screen-sharing and technical calls. Rather than pushing for the industry-standard approach (regular screen-share sessions), I proposed working independently in the client's accounts and sending written summaries. Calls were reserved for decisions only.

**Result:** This actually increased efficiency. Written summaries created an audit trail, reduced meeting overhead, and gave the client time to process technical information at their own pace.

**Takeaway:** Communication preferences aren't obstacles — they're constraints to design around. The "standard" consulting workflow isn't always the best one.

### 8. Handle Payment Gaps Cleanly

A payment was made for work that wasn't delivered within the expected timeframe due to external circumstances. Multiple resolution options existed: partial refund, renegotiation, dispute. The cleanest was crediting the full amount against the next phase of work.

**Why it worked:** No money changed hands again, the client got full value applied forward, and the relationship continued without resentment or awkwardness. Trust was maintained.

**Takeaway:** When you owe a client for undelivered work, the simplest resolution preserves the most trust. Credit forward, don't complicate it.

---

## Process Lessons

### 1. Research Before Quoting

The next phase required affiliate link automation. Initial assessment suggested the affiliate platform's API tier might gate link generation endpoints — which would have made the build significantly more complex.

Research revealed that the platform's affiliate links are **deterministic URL construction**: a simple pattern of `{base_url}/{product_id}/{affiliate_id}/{campaign}`. No API call needed. This completely changed the architecture — simple string concatenation replaced what was expected to be a complex API integration.

**Takeaway:** Always research the technical approach before committing to a quote. Assumptions about API complexity directly impact pricing, and the client benefits from accurate scoping.

### 2. Clean Data Before Automating

The client's product database contained 1,200+ rows with the majority missing affiliate links, massive duplicates, API errors logged as data, and test records mixed with production. Building automation on top of this would have produced automated mess — reliably generating the wrong outputs at scale.

The data audit was prioritised before the affiliate build, not alongside it.

**Takeaway:** Automation amplifies whatever it's built on. If the data layer is unreliable, the automation is unreliable — but faster and harder to debug. Always clean the foundation first.

### 3. Document Incrementally, Not as a Phase

One phase was a standalone documentation phase — a complete handover package produced after the build phases. While the output was comprehensive, the approach meant documentation was always retrospective, capturing decisions from memory rather than in the moment.

**Takeaway:** Building documentation incrementally during each phase — capturing decisions, rationale, and architecture as they happen — produces more accurate documentation with less effort. A standalone documentation phase is better than no documentation, but inline documentation is better still.

---

> **Disclaimer:** This case study describes generalised patterns from automation consulting work. Specific implementation details have been adapted for portfolio purposes and do not represent any single client engagement.
