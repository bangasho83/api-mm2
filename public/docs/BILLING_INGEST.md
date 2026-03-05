# Billing Ingest API

## Overview

The Billing Ingest API allows you to track usage events and consume credits from customer accounts in Metronome. Each event is recorded with a unique transaction ID and deducts the specified number of credits from the customer's balance.

**Note:** This endpoint requires the Metronome customer ID directly for maximum efficiency. No Firebase lookups are performed.

---

## Endpoint

```
POST /api/billing/ingest
```

---

## Request

### Headers

```
Content-Type: application/json
```

### Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `customerId` | string | ✅ Yes | Metronome customer ID (from organization's billingCustomerId) |
| `eventType` | string | ✅ Yes | Event type name (e.g., "post_generation", "image-gen") |
| `timestamp` | string | ❌ No | Event timestamp (ISO 8601 format, defaults to now) |
| `properties` | object | ✅ Yes | Properties object containing event metadata |
| `properties.credits` | number | ✅ Yes | Number of credits to consume (positive integer) |
| `properties.user_id` | string | ❌ No | User ID who triggered the event |
| `properties.project_id` | string | ❌ No | Project/Brand ID associated with the event |
| `properties.*` | any | ❌ No | Any additional custom properties |

### Example Request

```bash
curl -X POST https://social-apis-two.vercel.app/api/billing/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "d2c5bf81-9a12-4cd0-af26-0e2cf5656a6e",
    "eventType": "post_generation",
    "timestamp": "2025-10-09T22:07:48.461Z",
    "properties": {
      "credits": 10,
      "user_id": "user_1",
      "project_id": "proj_demo"
    }
  }'
```

### Example with Additional Properties

```bash
curl -X POST https://social-apis-two.vercel.app/api/billing/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "d2c5bf81-9a12-4cd0-af26-0e2cf5656a6e",
    "eventType": "image-gen",
    "timestamp": "2025-10-09T22:07:48.461Z",
    "properties": {
      "credits": 25,
      "user_id": "user_1",
      "project_id": "proj_demo",
      "model": "flux-kontext",
      "resolution": "1024x1024",
      "quality": "high"
    }
  }'
```

---

## Response

### Success Response (200 OK)

```json
{
  "success": true,
  "transactionId": "d2c5bf81-9a12-4cd0-af26-0e2cf5656a6e+abc123xyz789",
  "eventType": "post_generation",
  "timestamp": "2025-10-09T22:07:48.461Z",
  "customerId": "d2c5bf81-9a12-4cd0-af26-0e2cf5656a6e",
  "properties": {
    "credits": 10,
    "user_id": "user_1",
    "project_id": "proj_demo"
  },
  "billing": {
    "provider": "metronome",
    "customerId": "d2c5bf81-9a12-4cd0-af26-0e2cf5656a6e"
  }
}
```

### Error Responses

#### 400 Bad Request - Missing Required Field

```json
{
  "error": "Missing or invalid required field: credits",
  "message": "credits must be a positive number"
}
```

#### 400 Bad Request - No Billing Configured

```json
{
  "error": "Organization does not have billing configured",
  "organizationId": "4CRMRogqT0aOBBsUs7Yq",
  "message": "Please configure billing for this organization first using PATCH /api/organizations"
}
```

#### 404 Not Found - Organization Not Found

```json
{
  "error": "Organization not found",
  "organizationId": "invalid-id",
  "message": "The specified organization does not exist"
}
```

#### 410 Gone - Organization Deleted

```json
{
  "error": "Organization has been deleted",
  "organizationId": "4CRMRogqT0aOBBsUs7Yq",
  "deletedAt": "2025-10-09T12:00:00.000Z"
}
```

#### 500 Internal Server Error - Metronome Error

```json
{
  "error": "Failed to ingest billing event",
  "details": "Insufficient credits",
  "metronomeDetails": {
    "message": "Customer has insufficient credits for this event"
  }
}
```

---

## Flow Diagram

```
POST /api/billing/ingest
  ↓
1. Validate request parameters
  ↓
2. Get organization from Firebase
  ↓
3. Check organization exists and not deleted
  ↓
4. Check organization has billingCustomerId
  ↓
5. Generate unique transaction ID
  ↓
6. Ingest event to Metronome
  ↓
7. Return success with transaction details
```

---

## Transaction ID Format

Transaction IDs are automatically generated in the format:

```
<organizationId>+<randomKey>
```

**Example:**
```
4CRMRogqT0aOBBsUs7Yq+abc123xyz789def456
```

This ensures:
- ✅ Uniqueness across all events
- ✅ Traceability to organization
- ✅ Idempotency (same transaction ID won't be processed twice)

---

## Event Types

Common event types you might use:

| Event Type | Description | Typical Credits |
|------------|-------------|-----------------|
| `post_generation` | Social media post generation | 10 |
| `image_generation` | AI image generation | 25 |
| `calendar_generation` | Content calendar creation | 50 |
| `ad_generation` | Ad campaign generation | 30 |
| `analytics_query` | Analytics data query | 5 |

**Note:** Event types are flexible - you can define your own based on your use case.

---

## Usage Examples

### Example 1: Track Post Generation

```javascript
const response = await fetch('https://social-apis-two.vercel.app/api/billing/ingest', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    organizationId: '4CRMRogqT0aOBBsUs7Yq',
    eventType: 'post_generation',
    credits: 10,
    userId: 'user-123',
    brandId: 'brand-456',
    properties: {
      platform: 'instagram',
      postType: 'carousel'
    }
  })
});

const data = await response.json();
console.log('Transaction ID:', data.transactionId);
```

### Example 2: Track Image Generation

```javascript
const response = await fetch('https://social-apis-two.vercel.app/api/billing/ingest', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    organizationId: '4CRMRogqT0aOBBsUs7Yq',
    eventType: 'image_generation',
    credits: 25,
    userId: 'user-789',
    properties: {
      model: 'flux-kontext',
      resolution: '1024x1024',
      prompt: 'A beautiful sunset over mountains'
    }
  })
});

const data = await response.json();
console.log('Credits consumed:', data.credits);
```

### Example 3: Track with Custom Timestamp

```javascript
const response = await fetch('https://social-apis-two.vercel.app/api/billing/ingest', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    organizationId: '4CRMRogqT0aOBBsUs7Yq',
    eventType: 'calendar_generation',
    credits: 50,
    userId: 'user-456',
    timestamp: new Date().toISOString(),
    properties: {
      numberOfPosts: 12,
      duration: '1 month'
    }
  })
});

const data = await response.json();
```

---

## Integration Checklist

Before using the Billing Ingest API:

- [ ] Organization exists in Firebase
- [ ] Organization has billing configured (use `PATCH /api/organizations`)
- [ ] Organization has credits available (check with `GET /api/billing/balance`)
- [ ] Event type is defined and documented
- [ ] Credit cost per event is determined

---

## Best Practices

### 1. Check Balance Before Ingesting

```javascript
// Check balance first
const balanceResponse = await fetch(
  `https://social-apis-two.vercel.app/api/billing/balance?organizationId=${orgId}`
);
const balance = await balanceResponse.json();

if (balance.totalCredits >= requiredCredits) {
  // Ingest event
  await fetch('https://social-apis-two.vercel.app/api/billing/ingest', {
    method: 'POST',
    body: JSON.stringify({ ... })
  });
} else {
  console.error('Insufficient credits');
}
```

### 2. Handle Errors Gracefully

```javascript
try {
  const response = await fetch('/api/billing/ingest', {
    method: 'POST',
    body: JSON.stringify({ ... })
  });

  if (!response.ok) {
    const error = await response.json();
    
    if (error.error === 'Organization does not have billing configured') {
      // Redirect to billing setup
    } else if (error.details?.includes('Insufficient credits')) {
      // Show upgrade prompt
    }
  }
} catch (error) {
  console.error('Failed to ingest event:', error);
}
```

### 3. Include Relevant Metadata

```javascript
await fetch('/api/billing/ingest', {
  method: 'POST',
  body: JSON.stringify({
    organizationId: '...',
    eventType: 'post_generation',
    credits: 10,
    userId: currentUser.id,
    brandId: selectedBrand.id,
    properties: {
      platform: 'instagram',
      postType: 'carousel',
      aiModel: 'gpt-4',
      generationTime: 2.5 // seconds
    }
  })
});
```

---

## Related Endpoints

- **Configure Billing**: `PATCH /api/organizations` - Add billing to organization
- **Check Balance**: `GET /api/billing/balance` - Get current credit balance
- **Get Organization**: `GET /api/organizations` - Get organization details

---

## Support

For issues or questions:
- Check organization has billing configured
- Verify organization ID is correct
- Check Vercel logs for detailed error messages
- Ensure credits are available in the account

