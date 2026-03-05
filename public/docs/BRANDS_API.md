# Brands API Documentation

## Overview

The Brands API provides endpoints to manage brand documents in the system. All endpoints support CORS and return JSON responses.

**Base URL:** `https://social-apis-two.vercel.app/api/brands`

---

## Endpoints

### 1. Create Brand (POST)

**Endpoint:** `POST /api/brands`

**Description:** Creates a new brand document in the Firestore `brands` collection.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `client_name` | string | ✅ Yes | Brand/client name |
| `organizationId` | string | ✅ Yes | Organization ID this brand belongs to |
| `createdBy` | string | ✅ Yes | User ID of the creator |
| `website` | string | ❌ No | Brand website URL |
| `industry` | string | ❌ No | Industry/sector (e.g., "Technology & SaaS") |
| `country` | string | ❌ No | Country name |
| `tags` | array | ❌ No | Array of tags (e.g., ["premium", "enterprise"]) |
| `currency_symbol` | string | ❌ No | Currency symbol (e.g., "AED", "USD") |
| `brand_guidelines` | string | ❌ No | HTML or text content for brand guidelines |
| `services` | object | ❌ No | Services configuration object |

**Services Object Structure:**

```json
{
  "posts": 20,
  "blogs": 4,
  "seo": "Full SEO Audit",
  "newsletter": 2,
  "campaigns": 3,
  "fee": 5000,
  "monthly_budget": 15000
}
```

**Example Request:**

```bash
curl -X POST https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "Example Brand Inc",
    "website": "https://example.com",
    "industry": "Technology & SaaS",
    "country": "United Arab Emirates",
    "tags": ["premium", "enterprise", "tech"],
    "currency_symbol": "AED",
    "brand_guidelines": "<h2>Mission & Vision</h2><p>Our mission is...</p>",
    "services": {
      "posts": 20,
      "blogs": 4,
      "seo": "Full SEO Audit",
      "newsletter": 2,
      "campaigns": 3,
      "fee": 5000,
      "monthly_budget": 15000
    },
    "createdBy": "user_abc123",
    "organizationId": "org_xyz789"
  }'
```

**Success Response (201):**

```json
{
  "success": true,
  "message": "Brand created successfully",
  "brand": {
    "id": "brand_generated_id",
    "client_name": "Example Brand Inc",
    "organizationId": "org_xyz789",
    "createdBy": "user_abc123",
    "created_at": "2025-01-05T12:00:00.000Z",
    "website": "https://example.com",
    "industry": "Technology & SaaS",
    "country": "United Arab Emirates",
    "tags": ["premium", "enterprise", "tech"],
    "currency_symbol": "AED",
    "services": {
      "posts": 20,
      "blogs": 4,
      "seo": "Full SEO Audit",
      "newsletter": 2,
      "campaigns": 3,
      "fee": 5000,
      "monthly_budget": 15000
    }
  }
}
```

**Error Responses:**

#### 400 - Missing Required Field

```json
{
  "error": "Missing or invalid required field: client_name",
  "message": "client_name must be a valid string"
}
```

#### 400 - Invalid Field Type

```json
{
  "error": "Invalid field: tags",
  "message": "tags must be an array"
}
```

#### 500 - Server Error

```json
{
  "error": "Failed to create brand",
  "details": "Error message details"
}
```

---

### 2. Get Single Brand (GET)

**Endpoint:** `GET /api/brands?brandId=BRAND_ID`

**Description:** Retrieves a single brand by its ID. Returns 410 if the brand is archived (unless `includeArchived=true`).

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `brandId` | string | ✅ Yes | Brand ID to retrieve |
| `includeArchived` | boolean | ❌ No | Include archived brand (default: false) |

**Example Request:**

```bash
curl -X GET 'https://social-apis-two.vercel.app/api/brands?brandId=brand_123'
```

**Success Response (200):**

```json
{
  "success": true,
  "brand": {
    "id": "brand_123",
    "client_name": "Example Brand Inc",
    "organizationId": "org_xyz789",
    "website": "https://example.com",
    "industry": "Technology & SaaS",
    "country": "United Arab Emirates",
    "tags": ["premium", "enterprise", "tech"],
    "currency_symbol": "AED",
    "brand_guidelines": "<h2>Mission & Vision</h2>...",
    "services": {
      "posts": 20,
      "blogs": 4,
      "seo": "Full SEO Audit",
      "newsletter": 2,
      "campaigns": 3,
      "fee": 5000,
      "monthly_budget": 15000
    },
    "created_at": "2025-01-05T12:00:00.000Z",
    "updated_at": "2025-01-05T12:00:00.000Z",
    "createdBy": "user_abc123",
    "archived": false
  }
}
```

**Error Responses:**

#### 404 - Brand Not Found

```json
{
  "error": "Brand not found",
  "brandId": "brand_123"
}
```

#### 410 - Brand Archived

```json
{
  "error": "Brand is archived",
  "brandId": "brand_123",
  "archivedAt": "2025-01-05T12:00:00.000Z",
  "message": "This brand has been archived. Use includeArchived=true to view it."
}
```

---

### 3. Get Brands List (GET)

**Endpoint:** `GET /api/brands`

**Description:** Retrieves multiple brand documents. Excludes archived brands by default.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `organizationId` | string | ❌ No | Filter by organization ID |
| `seo` | string | ❌ No | Filter by SEO service value |
| `includeArchived` | boolean | ❌ No | Include archived brands (default: false) |
| `onlyArchived` | boolean | ❌ No | Show only archived brands |

**Example Requests:**

```bash
# Get all brands for an organization
curl -X GET 'https://social-apis-two.vercel.app/api/brands?organizationId=org_xyz789'

# Get brands with specific SEO service
curl -X GET 'https://social-apis-two.vercel.app/api/brands?organizationId=org_xyz789&seo=Full%20SEO%20Audit'

# Include archived brands
curl -X GET 'https://social-apis-two.vercel.app/api/brands?organizationId=org_xyz789&includeArchived=true'
```

**Success Response (200):**

```json
{
  "count": 2,
  "organizationId": "org_xyz789",
  "brands": [
    {
      "id": "brand_123",
      "client_name": "Example Brand Inc",
      "organizationId": "org_xyz789",
      "website": "https://example.com",
      "created_at": "2025-01-05T12:00:00.000Z"
    }
  ]
}
```

---

### 4. Update Brand (PUT)

**Endpoint:** `PUT /api/brands`

**Description:** Updates one or more fields of a brand. Uses merge mode to only update provided fields.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `brandId` | string | ✅ Yes | Brand ID to update |
| `organizationId` | string | ✅ Yes | Organization ID (for verification) |
| `userId` | string | ✅ Yes | User ID performing the update |
| `data` | object | ✅ Yes | Object containing fields to update |

**Protected Fields (Cannot be updated):**
- `id`
- `createdBy`
- `created_at`
- `archived`
- `archivedAt`
- `archivedMetadata`

**Example Request - Update Single Field:**

```bash
curl -X PUT https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "brandId": "brand_123",
    "organizationId": "org_xyz789",
    "userId": "user_abc123",
    "data": {
      "website": "https://newwebsite.com"
    }
  }'
```

**Example Request - Update Multiple Fields:**

```bash
curl -X PUT https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "brandId": "brand_123",
    "organizationId": "org_xyz789",
    "userId": "user_abc123",
    "data": {
      "client_name": "Updated Brand Name",
      "website": "https://newwebsite.com",
      "industry": "Finance & Banking",
      "tags": ["finance", "premium", "enterprise"],
      "services": {
        "posts": 30,
        "blogs": 8,
        "seo": "Premium SEO Package",
        "newsletter": 4,
        "campaigns": 5,
        "fee": 8000,
        "monthly_budget": 25000
      }
    }
  }'
```

**Success Response (200):**

```json
{
  "success": true,
  "message": "Brand updated successfully",
  "brand": {
    "id": "brand_123",
    "organizationId": "org_xyz789",
    "updatedFields": ["client_name", "website", "industry", "tags", "services"],
    "updated_at": "2025-01-05T14:30:00.000Z",
    "lastUpdatedBy": "user_abc123"
  }
}
```

**Error Responses:**

#### 400 - Missing Required Field

```json
{
  "error": "Missing or invalid required field: brandId",
  "message": "brandId must be a valid string"
}
```

#### 400 - Empty Data Object

```json
{
  "error": "Invalid data field",
  "message": "data object must contain at least one field to update"
}
```

#### 400 - Protected Fields

```json
{
  "error": "Cannot update protected fields",
  "message": "The following fields cannot be updated: id, createdBy",
  "protectedFields": ["id", "createdBy"]
}
```

#### 404 - Brand Not Found

```json
{
  "error": "Brand not found",
  "brandId": "brand_123"
}
```

#### 403 - Unauthorized

```json
{
  "error": "Unauthorized",
  "message": "Brand does not belong to this organization",
  "brandId": "brand_123",
  "organizationId": "org_xyz789"
}
```

---

### 5. Delete Brand (DELETE)

**Endpoint:** `DELETE /api/brands`

**Description:** Soft deletes (archives) a brand. The brand is not permanently deleted but marked as archived.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `brandId` | string | ✅ Yes | Brand ID to archive |
| `organizationId` | string | ✅ Yes | Organization ID (for verification) |
| `userId` | string | ✅ Yes | User ID performing the deletion |

**Example Request:**

```bash
curl -X DELETE https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "brandId": "brand_123",
    "organizationId": "org_xyz789",
    "userId": "user_abc123"
  }'
```

**Success Response (200):**

```json
{
  "success": true,
  "message": "Brand archived successfully",
  "brand": {
    "id": "brand_123",
    "name": "Example Brand Inc",
    "organizationId": "org_xyz789",
    "archivedAt": "2025-01-05T12:00:00.000Z"
  }
}
```

**Error Responses:**

#### 404 - Brand Not Found

```json
{
  "error": "Brand not found",
  "brandId": "brand_123"
}
```

#### 403 - Unauthorized

```json
{
  "error": "Unauthorized",
  "message": "Brand does not belong to this organization",
  "brandId": "brand_123",
  "organizationId": "org_xyz789"
}
```

#### 410 - Already Archived

```json
{
  "error": "Brand already archived",
  "brandId": "brand_123",
  "archivedAt": "2025-01-05T12:00:00.000Z",
  "message": "This brand has already been archived"
}
```

---

### 6. Update Brand (PATCH) - Partial Update or Restore

**Endpoint:** `PATCH /api/brands`

**Description:** Performs partial updates to brand fields OR restores an archived brand. Similar to PUT but more flexible.

---

#### Option A: Partial Update (Recommended for most updates)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `brandId` | string | ✅ Yes | Brand ID to update |
| `organizationId` | string | ✅ Yes | Organization ID (for verification) |
| `userId` | string | ✅ Yes | User ID performing the update |
| `data` | object | ✅ Yes | Object containing fields to update |

**Example Request:**

```bash
curl -X PATCH https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "brandId": "brand_123",
    "organizationId": "org_xyz789",
    "userId": "user_abc123",
    "data": {
      "website": "https://newwebsite.com",
      "tags": ["updated", "premium"]
    }
  }'
```

**Success Response (200):**

```json
{
  "success": true,
  "message": "Brand updated successfully",
  "brand": {
    "id": "brand_123",
    "organizationId": "org_xyz789",
    "updatedFields": ["website", "tags"],
    "updated_at": "2025-01-05T14:30:00.000Z",
    "lastUpdatedBy": "user_abc123"
  }
}
```

---

#### Option B: Restore Archived Brand

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `brandId` | string | ✅ Yes | Brand ID to restore |
| `organizationId` | string | ✅ Yes | Organization ID (for verification) |
| `userId` | string | ✅ Yes | User ID performing the restoration |
| `restore` | boolean | ✅ Yes | Must be `true` to restore |

**Example Request:**

```bash
curl -X PATCH https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "brandId": "brand_123",
    "organizationId": "org_xyz789",
    "userId": "user_abc123",
    "restore": true
  }'
```

**Success Response (200):**

```json
{
  "success": true,
  "message": "Brand restored successfully",
  "brand": {
    "id": "brand_123",
    "organizationId": "org_xyz789",
    "restoredAt": "2025-01-05T12:00:00.000Z"
  }
}
```

**Error Responses:**

#### 400 - Not Archived (when restore=true)

```json
{
  "error": "Brand is not archived",
  "brandId": "brand_123",
  "message": "Only archived brands can be restored"
}
```

#### 400 - Missing Data (when restore is not true)

```json
{
  "error": "Missing or invalid required field: data",
  "message": "data must be an object containing fields to update"
}
```

---

## Complete cURL Examples

### Minimal Brand Creation

```bash
curl -X POST https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "My New Brand",
    "organizationId": "org_xyz789",
    "createdBy": "user_abc123"
  }'
```

### Full Brand Creation with All Fields

```bash
curl -X POST https://social-apis-two.vercel.app/api/brands \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "Premium Tech Solutions",
    "website": "https://premiumtech.com",
    "industry": "Technology & SaaS",
    "country": "United Arab Emirates",
    "tags": ["premium", "enterprise", "tech", "b2b"],
    "currency_symbol": "AED",
    "brand_guidelines": "<h2>Brand Mission</h2><p>To deliver cutting-edge solutions...</p><h2>Core Values</h2><ul><li>Innovation</li><li>Excellence</li></ul>",
    "services": {
      "posts": 20,
      "blogs": 4,
      "seo": "Full SEO Audit",
      "newsletter": 2,
      "campaigns": 3,
      "fee": 5000,
      "monthly_budget": 15000
    },
    "createdBy": "user_abc123",
    "organizationId": "org_xyz789"
  }'
```

---

## Notes

- All timestamps are automatically generated using Firestore server timestamps
- Brands are automatically marked as `archived: false` on creation
- The `created_at` and `updated_at` fields are automatically set
- Brand IDs are auto-generated by Firestore
- All endpoints support CORS with wildcard origin (`*`)
- Maximum execution time: 30 seconds

---

## Related APIs

- **Organizations API**: `/api/organizations` - Manage organizations
- **Calendar API**: `/api/calendar` - Manage calendar posts for brands
- **Keywords API**: `/api/keywords` - Manage brand keywords

