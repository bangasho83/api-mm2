# 🔍 Billing Integration Audit

## Rate Card vs Metronome Event Types

| API Endpoint | Rate Card Name | Credits | Metronome Event Type | Status |
|-------------|----------------|---------|---------------------|--------|
| api/image-gen.js | image-gen | 25 | `image-gen` | ✅ INTEGRATED |
| api/image-gen.js (pro) | image-gen-pro | 50 | `image-gen-pro` | ✅ INTEGRATED |
| api/html-scraper.js | html-scraper | 10 | `html-scrapper` ⚠️ | ❌ NOT INTEGRATED |
| api/site-audit.js | site-audit | 300 | `site-audit` | ❌ NOT INTEGRATED |
| api/keyword-sim.js | keyword-sim | 100 | `keyword-sim` | ❌ NOT INTEGRATED |
| api/meta-ad-sim.js | meta-ad-sim | 50 | `meta-ad-sim` | ❌ NOT INTEGRATED |
| api/image-prompt.js | image-prompt | 10 | `image-prompt` | ❌ NOT INTEGRATED |
| api/calendar-sim.js | calendar-sim | 200 | `calendar-sim` | ❌ NOT INTEGRATED |
| api/keyword-research.js | keyword-research | 5 | `keyword-research` | ❌ NOT INTEGRATED |
| api/mates-take/overview.js | mates-take/overview | 5 | `mates-take-overview` | ❌ NOT INTEGRATED |

⚠️ **Note**: Metronome uses `html-scrapper` (double 'p'), not `html-scraper`

---

## ✅ APIs WITH Billing Integration

### 1. **image-gen** (api/image-gen.js)
- ✅ **Status**: INTEGRATED
- ✅ **Credits**: 25 per image (standard), 50 per image (pro models)
- ✅ **Event Types**: `image-gen`, `image-gen-pro`
- ✅ **Implementation**: Lines 615-671
- ✅ **Logging**: Comprehensive logging added
- **Models**:
  - Standard (25 credits): `imagen4`, `flux-kontext`, `banana-text`, `banana-image`
  - Pro (50 credits): `banana-pro-text`, `banana-pro-image`

---

## ❌ APIs MISSING Billing Integration

### 2. **html-scraper** (api/html-scraper.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 10 (per rate card)
- ❌ **Event Type**: `html-scraper`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 3. **site-audit** (api/site-audit.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 300 (per rate card)
- ❌ **Event Type**: `site-audit`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 4. **keyword-sim** (api/keyword-sim.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 100 (per rate card)
- ❌ **Event Type**: `keyword-sim`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 5. **meta-ad-sim** (api/meta-ad-sim.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 50 (per rate card)
- ❌ **Event Type**: `meta-ad-sim`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 6. **image-prompt** (api/image-prompt.js or api/gen-image-prompt.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 10 (per rate card)
- ❌ **Event Type**: `image-prompt`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 7. **calendar-sim** (api/calendar-sim.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 200 (per rate card)
- ❌ **Event Type**: `calendar-sim`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 8. **keyword-research** (api/keyword-research.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 5 (per rate card)
- ❌ **Event Type**: `keyword-research`
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

### 9. **mates-take/overview** (api/mates-take/overview.js)
- ❌ **Status**: NOT INTEGRATED
- ❌ **Credits**: 5 (per rate card)
- ❌ **Event Type**: `mates-take-overview` (normalized from `mates-take/overview`)
- ❌ **Missing**: No metronomeService.ingestEvent call
- **Action Required**: Add billing integration

---

## 📊 Summary

| Status | Count | APIs |
|--------|-------|------|
| ✅ Integrated | 1 | image-gen |
| ❌ Missing | 8 | html-scraper, site-audit, keyword-sim, meta-ad-sim, image-prompt, calendar-sim, keyword-research, mates-take/overview |

**Total APIs**: 9  
**Integration Coverage**: 11% (1/9)

---

## 🔧 Required Actions

All missing APIs need to add billing integration following this pattern:

```javascript
const metronomeService = require('../services/metronome-service');

// After successful API operation, before returning response:
if (organizationId && billingCustomerId) {
  try {
    await metronomeService.ingestEvent({
      organization_id: organizationId,
      customer_id: billingCustomerId,
      event_type: 'EVENT_TYPE_HERE', // e.g., 'html-scraper'
      timestamp: null,
      properties: {
        credits: CREDITS_AMOUNT, // e.g., 10 for html-scraper
        project_id: 'metrixmate',
        organization_id: organizationId
        // Add any additional metadata
      }
    });
  } catch (billingError) {
    console.error('[API_NAME] Failed to ingest billing event:', billingError?.message);
    // Don't fail the request if billing fails
  }
}
```

