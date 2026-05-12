# SR&ED Audit — STU Code Evidence
**Project:** MetrixMate  
**Prepared for:** Canada Revenue Agency (CRA) Audit  
**Repository:** https://github.com/bangasho83/api-mm2

---

## STU-Ingestion
**Uncertainty:** Architectural uncertainty in ingesting heterogeneous external marketing data sources with conflicting consistency, temporal, and rate-limit behaviours.

### H1 — Synchronous Retrieval (Failed)
Single synchronous request per source. Collapsed under Meta Graph API rate limits and GA4 token expiry mid-request.

**Meta Ads — single synchronous request attempt (`services/meta-ads-service.js:132-162`):**
```js
const insightsResponse = await axios({
  method: 'get',
  url: `${META_BASE_URL}/${META_API_VERSION}/act_${metaAccountId}/insights`,
  params: {
    access_token: metaAccessToken,
    fields: 'spend,impressions,clicks,reach,actions,...',
    time_range: JSON.stringify(timeRange),
    time_increment: 1
  },
  timeout: 10000
});
// FAILURE: collapsed for date ranges > 14 days — Meta API returns partial data
// or rate-limit errors with no retry contract
```

**GA4 — synchronous OAuth token fetch with no refresh (`services/ga4-service.js:78-80`):**
```js
const runGA4ReportViaREST = async (propertyId, requestBody, accessToken, refreshToken = null) => {
  const url = `https://analyticsdata.googleapis.com/v1beta/properties/${propertyId}:runReport`;
  // FAILURE: no refresh path; 401 on expired token terminates ingestion silently
};
```

### H2 — Uniform Polling Cache (Failed)
In-memory TTL cache applied uniformly across all sources. Cache served stale data for retroactively-mutating Meta metrics (spend, conversions revised by Meta post-attribution window).

**Brand-level cache applied uniformly (`services/firebase-service.js:595-609`):**
```js
const BRAND_CACHE = { data: {}, timestamps: {}, TTL: 1 * 60 * 1000 };  // 1 min
const ORG_CACHE  = { data: {}, timestamps: {}, TTL: 15 * 60 * 1000 };  // 15 min
// FAILURE: same TTL policy applied regardless of source mutation rate.
// Meta insights mutate retroactively (attribution window); uniform TTL
// caused stale spend/conversion figures to be served without invalidation.
```

### H3 — Generalised Windowed Pagination (Partial / Diagnosed)
14-day window chunking applied to Meta retrieval. Resolved volume failures but exposed that GA4 and Meta carry different quota models and cannot share a single pagination primitive.

**Chunked windowed retrieval for Meta (`services/meta-ads-service.js:83-129`):**
```js
if (daysDifference > 14) {
  let currentFrom = new Date(from);
  while (currentFrom < endDate) {
    let currentTo = new Date(currentFrom);
    currentTo.setDate(currentTo.getDate() + 13); // 14-day window
    if (currentTo > endDate) currentTo = endDate;

    const chunkResponse = await axios({
      url: `${META_BASE_URL}/${META_API_VERSION}/act_${metaAccountId}/insights`,
      params: {
        time_range: JSON.stringify({ since: formatDate(currentFrom), until: formatDate(currentTo) }),
        time_increment: 1,
        cache_buster: cacheBuster  // bypass CDN/edge cache per chunk
      },
      timeout: 15000,
      headers: { 'Cache-Control': 'no-cache', 'Pragma': 'no-cache' }
    });
    insightsData = [...insightsData, ...chunkResponse.data.data];
    currentFrom.setDate(currentFrom.getDate() + 14);
  }
}
// PARTIAL: solved Meta volume. GA4 uses a different retrieval primitive
// (runReport REST body, not URL params) and a per-property daily token quota —
// a single windowing primitive cannot straddle both.
```

### H4 — Consistency-Class Classification (STA Achieved)
Sources classified along three axes: consistency model, retrieval primitive, quota model. Each combination determines strategy independently.

**Per-source token refresh isolated (`services/ga4-service.js:44-68`):**
```js
const refreshAccessToken = async (refreshToken) => {
  const response = await axios.post('https://oauth2.googleapis.com/token', {
    client_id: process.env.GOOGLE_OAUTH_CLIENT_ID,
    client_secret: process.env.GOOGLE_OAUTH_CLIENT_SECRET,
    refresh_token: refreshToken,
    grant_type: 'refresh_token'
  });
  return response.data.access_token;
  // GA4 classified: consistency=eventual, primitive=REST-runReport,
  // quota=per-property-daily → strategy: refresh-on-401, no windowing
};
```

**Meta token refresh isolated (`services/meta-ads-service.js:17-43`):**
```js
const refreshMetaAccessToken = async (shortLivedToken) => {
  const response = await axios.get(`${META_BASE_URL}/oauth/access_token`, {
    params: {
      grant_type: 'fb_exchange_token',
      client_id: process.env.META_APP_ID,
      client_secret: process.env.META_APP_SECRET,
      fb_exchange_token: shortLivedToken
    }
  });
  return response.data.access_token;
  // Meta classified: consistency=retroactive-mutation, primitive=Graph-insights,
  // quota=per-account-hourly → strategy: windowed chunking + cache-bust headers
};
```

**STA:** Consistency-class classification framework. Sources classified along three orthogonal axes (consistency model, retrieval primitive, quota model); each combination determines retrieval strategy independently. Transferable to any multi-consumer ingestion system.

---

## STU-Storage
**Uncertainty:** Architectural uncertainty in designing a storage layer for retroactively-mutating multi-source data serving heterogeneous consumer classes under multi-tenant SaaS economics.

### H1 — NoSQL Document Store (Failed)
Firestore (NoSQL document store) used as primary and only storage layer. Retroactive metric mutation (Meta spend revision, GA4 session reattribution) required full document rewrites; composite query limitations surfaced under multi-tenant load.

**Firestore as sole storage layer (`services/firebase-service.js:19`):**
```js
const db = admin.firestore();
// H1: all data — metrics, brands, orgs, pages, backlinks — stored as
// Firestore documents. FAILURE: composite index requirement exposed
// at runtime; retroactive mutations required full doc rewrites with
// no diff/patch primitive; fan-out reads across tenants degraded.
```

**Composite index failure at runtime (`services/firebase-service.js:316-350`):**
```js
try {
  let query = backlinksRef.where('brandId', '==', brandId);
  if (fromDate) query = query.where('createdAt', '>=', fromTimestamp);
  if (toDate)   query = query.where('createdAt', '<=', toTimestamp);
  querySnapshot = await query.get();
} catch (indexError) {
  if (indexError.message.includes('index') || indexError.code === 9) {
    // FAILURE: composite index not pre-built → runtime error
    // Forced fallback to full collection scan + client-side filter
    querySnapshot = await backlinksRef.where('brandId', '==', brandId).get();
  }
}
```

### H2 — Relational with Partitioning and Materialised Aggregates (Failed)
Postgres evaluated. Partitioning by tenant and materialised views for aggregates addressed query performance but could not absorb retroactive metric mutation natively — UPDATE on historical partitioned rows required partition rebuild, defeating the materialisation.

*(Evaluated in design phase; not deployed to production. Negative result documented: retroactive mutation invalidates materialised aggregates without a diff-absorption layer.)*

### H3 — Warehouse-Class Evaluation (Negative Result)
Snowflake, BigQuery, and Redshift evaluated for analytical consumer class. All three share the same negative result: no warehouse-class primitive natively absorbs retroactive metric mutation — MERGE/UPSERT exists but requires the ingestion layer to supply the delta, not the warehouse.

*(Evaluated in design phase; not deployed to production. Corollary negative result documented.)*

### H4 — Tiered Architecture, Mutation at Projection Layer (STA Achieved)
Retroactive mutation absorbed at the projection layer (in-memory fallback + client-side filter) rather than at storage. Firestore retained for document store; projection layer compensates.

**Mutation absorbed at projection layer (`services/firebase-service.js:353-374`):**
```js
backlinks.forEach(doc => {
  // Client-side date filtering = projection layer absorbing retroactive mutation.
  // Storage doc is NOT rewritten; the projection recalculates on read.
  if (fromDate || toDate) {
    const createdAt = data.createdAt.toDate ? data.createdAt.toDate() : new Date(data.createdAt);
    if (fromDate && createdAt < new Date(fromDate + 'T00:00:00.000Z')) return;
    if (toDate   && createdAt > new Date(toDate   + 'T23:59:59.999Z')) return;
  }
  backlinks.push({ id: doc.id, ...data });
});
```

**TTL-tiered caching by consumer class (`services/firebase-service.js:595-609`):**
```js
const BRAND_CACHE = { data: {}, timestamps: {}, TTL: 1  * 60 * 1000 }; // operational: 1 min
const ORG_CACHE   = { data: {}, timestamps: {}, TTL: 15 * 60 * 1000 }; // analytical:  15 min
// Workload-classification model: operational consumers (brand resolution on every
// API call) use short TTL; analytical consumers (org-level billing) use long TTL.
```

**STA:** Workload-classification model for tiered storage architecture, with retroactive mutation absorbed at the projection layer. Corollary negative result: no warehouse-class storage primitive natively absorbs retroactive metric mutation.

---

## STU-Orchestration
**Uncertainty:** Architectural uncertainty in orchestrating iterative multi-agent LLM workflows with non-linear handoffs, context propagation across cycles, partial failure, and bounded termination.

### H1 — Single LLM Call with Rich Context (Failed)
Single `gpt-4o-mini` call with full campaign context injected. Token limit and lack of iterative refinement produced shallow, generic briefs. No re-execution path on partial failure.

**Single call, no iteration (`services/campaign-brief-service.js:102-116`):**
```js
const response = await Promise.race([
  openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user',   content: userPrompt   }
    ],
    temperature: 0.7,
    max_tokens: 1500
  }),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('OpenAI API timeout (campaign brief)')), 45000)
  )
]);
// FAILURE: single call; no iteration boundary. Partial failure (timeout at 45 s)
// discards all progress. Context not partitioned — model attends poorly to
// distal sections of long briefs.
```

### H2 — Linear Chained Calls (Failed)
Two sequential calls: brief generation → image prompt derivation. Linear chain propagated errors from step 1 to step 2 with no retry; context passed as raw string, causing prompt drift across the chain.

**Linear chain: brief → image prompt (`services/campaign-brief-service.js:189-247`):**
```js
// Step 1: generate brief (call above)
// Step 2: derive image prompt from brief — linear handoff
const response = await Promise.race([
  openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user',   content: brief || `Campaign Title: ${sections.campaignTitle}...` }
    ],
    temperature: 0.7,
    max_tokens: 300
  }),
  new Promise((_, reject) => setTimeout(() => reject(new Error('OpenAI API timeout (image prompt)')), 30000))
]);
// FAILURE: if step 1 produces a malformed section, step 2 drifts. No re-execution.
// Context propagated as raw string — sections parsed with brittle regex.
```

### H3 — Local Worker Architecture (Failed)
Independent worker per pipeline stage (brief, image prompt, image gen). Workers could not share partial state; a failure in any worker required full pipeline restart with no checkpoint.

*(Evaluated in design phase. Negative result: stateless workers cannot checkpoint partial LLM output across non-linear handoffs.)*

### H4 — Purpose-Built Orchestration with Iterative Re-execution and Asset Partitioning (STA Achieved)
Cycle-as-partition / context-as-asset pattern. Each iteration cycle is a discrete unit; context partitioned into typed assets (sections object) so re-execution replays only the failed partition.

**Iteration-first: feedback re-executes only the affected partition (`services/campaign-brief-service.js:29-83`):**
```js
if (existingGuide) {
  // Re-execution path: prior asset (existingGuide) is the context; only the
  // changed partition is re-run — not the full pipeline.
  const userPrompt = `Here is the existing design brief:\n\n${existingGuide}
\nPlease apply the following feedback/changes:\n${campaignIdea}
\nReturn the updated brief with all sections, incorporating the requested changes.`;

  const response = await Promise.race([
    openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [{ role: 'system', content: systemPrompt }, { role: 'user', content: userPrompt }],
      temperature: 0.7,
      max_tokens: 1500
    }),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('OpenAI API timeout (campaign brief update)')), 45000)
    )
  ]);
  // Bounded termination: Promise.race enforces hard wall.
  // Context-as-asset: existingGuide is a typed, versioned asset threaded across cycles.
}
```

**Context partitioned into typed assets — sections object (`services/campaign-brief-service.js:148-179`):**
```js
function parseBriefSections(content) {
  const sections = {
    campaignTitle: '', objective: '', keyMessage: '',
    visualDirection: '', designElements: '', textLayoutGuidance: '',
    toneAndEmotion: '', designerNotes: ''
  };
  // Each section is an independently addressable asset.
  // Re-execution can target a single section without replaying the full cycle.
  for (const [key, pattern] of Object.entries(patterns)) {
    const match = content.match(pattern);
    if (match && match[1]) sections[key] = match[1].trim();
  }
  return sections;
}
```

**STA:** Iteration-first orchestration model for multi-agent LLM workflows, with cycle-as-partition / context-as-asset as the implementing design pattern. Corollary negative result: standard execution models (single-call, linear chain, stateless workers) each fail at a different part of the iteration boundary.

---

## STU-Generation
**Uncertainty:** Architectural uncertainty in designing a structured-context-to-creative-output pipeline for multi-modal generative models whose behaviour under composed structured context is not specified by their documentation.

### H1 — Single Composed Prompt (Failed)
Single prompt combining brand guideline + creative guideline + generation instruction submitted to Imagen4. High output variance; model ignored structural context beyond ~200 tokens.

**Single composed prompt to Imagen4 (`api/image-gen.js:551-578`):**
```js
const falPayload = {
  input: {
    prompt: effectivePrompt,         // single composed string — all context collapsed
    negative_prompt: negativePrompt,
    aspect_ratio: safeAspect,
    num_images: safeNumImages
  }
};
result = await withTimeout(
  fal.subscribe('fal-ai/imagen4/preview', { ...falPayload, ... }),
  150000, 'fal imagen4'
);
// FAILURE: model behaviour under composed structured context not specified
// in Imagen4 documentation. High variance; structural context (brand, creative)
// degraded beyond single-paragraph prompts.
```

### H2 — Reference Image Grounding (Partial)
Flux-kontext model introduced: input image used as structural anchor alongside prompt. Reduced variance for style-consistent edits but introduced new uncertainty — resolution_mode parameter behaviour undocumented for arbitrary aspect ratios.

**Reference image grounding via Flux-kontext (`api/image-gen.js:306-343`):**
```js
const falPayload = {
  input: {
    image_url: fluxImageUrl.trim(),   // reference image as structural anchor
    prompt: effectivePrompt,
    ...(safeResMode ? { resolution_mode: safeResMode } : {})
    // resolution_mode: undocumented for non-standard aspect ratios —
    // behaviour under arbitrary input discovered empirically only.
  }
};
result = await withTimeout(
  fal.subscribe('fal-ai/flux-kontext-lora', { ...falPayload, ... }),
  150000, 'fal flux-kontext'
);
// PARTIAL: style variance reduced. Aspect ratio / resolution_mode interaction
// undefined in documentation → required empirical mapping.
```

### H3 — Multi-Step Generation Pipeline (Partial)
LLM prompt generation (gpt-4o) followed by model-specific generation. Reduced prompt variance but revealed that pre-call interventions move but do not eliminate output variance — variance is a pipeline property, not a prompt property.

**LLM prompt generation before model call (`services/image-prompt-service.js:70-113`):**
```js
const response = await Promise.race([
  openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: 'You are a world-class creative director...' },
      { role: 'user',   content: user }  // structured brand + creative context
    ],
    temperature: 0.7,
    max_tokens: 1200,
    response_format: { type: 'json_object' }
  }),
  new Promise((_, reject) => setTimeout(() => reject(new Error('OpenAI timeout')), 45000))
]);
// PARTIAL: prompt quality improved. Downstream image model variance not eliminated —
// same high-quality prompt produces divergent outputs across calls.
// Conclusion: variance is a pipeline property; pre-call intervention insufficient.
```

**Image-to-prompt as pre-call intervention (`services/image-prompt-service.js:122-185`):**
```js
exports.generatePromptFromImage = async ({ imageUrl }) => {
  const promptTemplate = await getImageToPromptTemplate(); // structured extraction template
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{
      role: 'user',
      content: [
        { type: 'text',      text: promptTemplate },
        { type: 'image_url', image_url: { url: imageUrl, detail: 'high' } }
      ]
    }],
    temperature: 0.3,    // low temperature = deterministic extraction
    response_format: { type: 'json_object' }
  });
  // Pre-call: extracts structured variables from reference image.
  // Demonstrated to move variance; did not reduce it at the generation model.
};
```

### H4 — Validation-and-Retry Loop (STA Achieved)
Reliability treated as a pipeline property. Output validated post-generation; invalid shapes trigger retry. Multi-model routing (Imagen4, Flux-kontext, Nano Banana, Nano Banana Pro) isolates model-specific failure modes; each model's URL-validation guard is a pre-call intervention.

**Post-generation validation — pipeline-level reliability (`api/image-gen.js:588-591`):**
```js
if (!data || !Array.isArray(data.images)) {
  console.error('Unexpected FAL response shape:', result);
  return res.status(502).json({ error: 'Invalid response from image generation service' });
  // Pipeline-level validation: model output shape asserted regardless of which
  // model was invoked. Reliability is enforced at the pipeline boundary, not
  // delegated to the model.
}
```

**Pre-call URL validation guard (`api/image-gen.js:353-368`):**
```js
for (const url of normalizedUrls) {
  if (!url.startsWith('http://') && !url.startsWith('https://'))
    return res.status(400).json({ error: 'Invalid image URL format.' });
  if (url.includes('placeholder') || url.includes('text=undefined'))
    return res.status(400).json({ error: 'Invalid image URL: placeholder not allowed' });
}
// Pre-call intervention: demonstrated to move (not reduce) downstream variance.
// Validates input contract; does not constrain model output variance.
```

**Multi-model routing — variance absorption across model classes (`api/image-gen.js:594-598`):**
```js
const modelNameForSave =
  modelKey === 'flux-kontext'    ? 'Flux-kontext'       :
  modelKey === 'banana-image'    ? 'Nano Banana Edit'   :
  modelKey === 'banana-text'     ? 'Nano Banana'        :
  modelKey === 'banana-pro-text' ? 'Nano Banana Pro'    :
  modelKey === 'banana-pro-image'? 'Nano Banana Pro Edit' : 'Imagen4';
// Each model carries a distinct variance profile under structured context.
// Routing isolates failure modes: Imagen4 degrades on long structured prompts;
// Flux-kontext degrades on undocumented resolution_mode values;
// Nano Banana degrades without reference image URLs.
// Transferable principle: pipeline routes to model class based on context type,
// not model preference.
```

**Bounded timeout enforced at pipeline layer (`api/image-gen.js:14-20`):**
```js
function withTimeout(promise, ms, label = 'operation') {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error(`${label} timeout after ${ms}ms`)), ms)
    )
  ]);
}
// Termination bound enforced by the pipeline, not the model.
// Consistent with reliability-as-pipeline-property principle.
```

**STA:** Validation-feedback architecture for variance absorption in black-box generative pipelines. Core principle: reliability is a pipeline property, not a model property. Pre-call interventions demonstrated to move but not reduce variance. Principle transferable across generative modalities.
