# Metronome Billing Integration

## Overview

The Metronome billing service is integrated with the organization creation flow. When a new organization is created via the Organizations API, a corresponding customer is automatically created in Metronome for billing purposes.

## Architecture

### Components

1. **Metronome Service** (`services/metronome-service.js`)
   - Handles all Metronome API interactions
   - Creates, retrieves, and lists customers

2. **Organizations API** (`api/organizations.js`)
   - POST endpoint creates organization in Firebase
   - Automatically creates Metronome customer
   - Stores Metronome Customer ID in organization document

### Data Flow

```
Client Request (POST /api/organizations)
    ↓
Create Organization in Firebase (orgs collection)
    ↓
Generate Organization ID
    ↓
Create Metronome Customer (external_id = Organization ID)
    ↓
Create Free Trial Contract (500 credits)
    ↓
Store Metronome Customer ID in Organization Document
    ↓
Return Response with Firebase ID, Metronome ID, and Free Trial Info
```

## API Endpoints

### 1. Create Organization (POST)

**Endpoint:** `POST /api/organizations`

**Request Body:**
```json
{
  "organizationName": "Sholay",
  "organizationUsername": "sholay",
  "createdBy": "user-uid-123"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "organization": {
    "id": "kq8D0URspd5I7uBck8l9",
    "organizationName": "Sholay",
    "organizationUsername": "sholay",
    "createdBy": "user-uid-123",
    "createdAt": "2025-06-09T10:00:00.000Z",
    "updatedAt": "2025-06-09T10:00:00.000Z",
    "billingCustomerId": "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5"
  },
  "billing": {
    "provider": "metronome",
    "success": true,
    "customerId": "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    "error": null,
    "freeTrial": {
      "success": true,
      "contractId": "contract-abc-123",
      "credits": 500,
      "error": null
    }
  }
}
```

**What Happens:**
1. Organization is created in Firebase `orgs` collection
2. Customer is created in Metronome with `external_id` = Firebase organization ID
3. `billingCustomerId` is stored in the organization document
4. Free trial contract is created (500 credits, 1 year duration)

**cURL Example:**
```bash
curl -X POST https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationName": "Sholay",
    "organizationUsername": "sholay",
    "createdBy": "user-uid-123"
  }'
```

### 2. Add Billing to Existing Organization (PATCH)

**Endpoint:** `PATCH /api/organizations`

**Purpose:** Add billing to an existing organization that doesn't have billing configured

**Request Body:**
```json
{
  "organizationId": "0Wt0rGMr1zGgbu2NhJrK"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Billing added to existing organization successfully",
  "organization": {
    "id": "0Wt0rGMr1zGgbu2NhJrK",
    "organizationName": "Krunch Cheese",
    "organizationUsername": "krunchcheese",
    "createdBy": "22tBJ3qzGaTVbuwks0EfMzX19wA3",
    "createdAt": "2025-10-06T10:14:32.000Z",
    "updatedAt": "2025-10-09T17:00:00.000Z",
    "billingCustomerId": "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5"
  },
  "billing": {
    "provider": "metronome",
    "success": true,
    "customerId": "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5",
    "freeTrial": {
      "success": true,
      "contractId": "contract-abc-123",
      "credits": 500,
      "error": null
    }
  }
}
```

**cURL Example:**
```bash
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "0Wt0rGMr1zGgbu2NhJrK"
  }'
```

**What Happens:**
1. Checks if organization exists
2. Checks if organization already has billing (returns existing if yes)
3. Creates Metronome customer with `external_id` = organization ID
4. Updates organization with `billingCustomerId`
5. Creates free trial contract (500 credits, 1 year duration)

**Use Cases:**
- Migration: Add billing to organizations created before billing system
- Retry: Retry billing creation if it failed during organization creation
- Manual: Admin manually adds billing to specific organization

**See:** [Add Billing Documentation](./ORGANIZATION_ADD_BILLING.md)

### 3. Get Organization by ID (GET)

**Endpoint:** `GET /api/organizations?organizationId=ORG_ID`

**Query Parameters:**
- `organizationId` (required): Firebase organization ID
- Alternative: `id` (same as organizationId)

**Response:**
```json
{
  "success": true,
  "organization": {
    "id": "kq8D0URspd5I7uBck8l9",
    "organizationName": "Sholay",
    "organizationUsername": "sholay",
    "createdBy": "user-uid-123",
    "createdAt": "June 2, 2025 at 10:54:33 PM UTC",
    "updatedAt": "June 2, 2025 at 10:54:33 PM UTC",
    "billingCustomerId": "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    "brandsCount": 5,
    "membersCount": 12
  }
}
```

**cURL Example:**
```bash
curl -X GET "https://social-apis-two.vercel.app/api/organizations?organizationId=kq8D0URspd5I7uBck8l9"
```

**Error Response (404):**
```json
{
  "error": "Organization not found",
  "organizationId": "invalid-org-id"
}
```

### 3. List Organizations (GET)

**Endpoint:** `GET /api/organizations`

**Query Parameters:**
- `limit` (optional): Number of organizations to return (default: 500, max: 500)

**Response:**
```json
{
  "count": 10,
  "organizations": [
    {
      "id": "kq8D0URspd5I7uBck8l9",
      "organizationName": "Sholay",
      "organizationUsername": "sholay",
      "createdBy": "user-uid-123",
      "createdAt": "June 2, 2025 at 10:54:33 PM UTC",
      "updatedAt": "June 2, 2025 at 10:54:33 PM UTC",
      "billingCustomerId": "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
      "brandsCount": 5,
      "membersCount": 12
    }
  ]
}
```

### 4. Delete Organization (DELETE)

**Endpoint:** `DELETE /api/organizations`

**Security:** Only the organization creator can delete. Multi-layer authorization checks.

**Request Body:**
```json
{
  "organizationId": "kq8D0URspd5I7uBck8l9",
  "userId": "user-uid-123"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Organization deleted successfully",
  "organization": {
    "id": "kq8D0URspd5I7uBck8l9",
    "organizationName": "Sholay",
    "deletedAt": "2025-10-09T15:30:00.000Z",
    "deletedBy": "user-uid-123"
  }
}
```

**cURL Example:**
```bash
curl -X DELETE https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "kq8D0URspd5I7uBck8l9",
    "userId": "user-uid-123"
  }'
```

**Security Features:**
- ✅ Parameter validation
- ✅ Organization existence check
- ✅ Deletion status check (prevents double-delete)
- ✅ Creator authorization (only creator can delete)
- ✅ User validation
- ✅ Full audit logging with IP tracking

**Error Responses:**
- `400` - Missing/invalid parameters
- `403` - Unauthorized (not the creator)
- `404` - Organization not found
- `410` - Already deleted

**See:** [Organization Soft Delete Documentation](./ORGANIZATION_SOFT_DELETE.md)

### 5. Get Billing Balance (GET)

**Endpoint:** `GET /api/billing/balance?organizationId=ORG_ID`

**Query Parameters:**
- `organizationId` (required): Firebase organization ID

**Response:**
```json
{
  "success": true,
  "organizationId": "kq8D0URspd5I7uBck8l9",
  "billing": {
    "provider": "metronome",
    "customerId": "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    "status": "active",
    "status_desc": "Active - 425 credits available",
    "balance": {
      "total": 425,
      "credits": [
        {
          "id": "ac615ee0-8480-42c9-bc25-f08e2efd9f65",
          "name": "Free Trial Credits",
          "balance": 425,
          "type": "CREDIT",
          "status": "active",
          "status_desc": "Active - Expires 10/9/2026",
          "product": {
            "id": "4983cc7c-30b6-4355-a07a-9d5afd99d1a6",
            "name": "Included Credits"
          },
          "schedule": {
            "amount": 500,
            "starting_at": "2025-10-09T15:00:00+00:00",
            "ending_before": "2026-10-09T15:00:00+00:00"
          },
          "created_at": "2025-10-09T15:09:36.301000+00:00"
        }
      ]
    }
  }
}
```

**cURL Example:**
```bash
curl -X GET "https://social-apis-two.vercel.app/api/billing/balance?organizationId=kq8D0URspd5I7uBck8l9"
```

**Use Cases:**
- Dashboard widget showing remaining credits
- Check balance before generating content
- Billing page showing credit breakdown
- Real-time balance updates

**Error Responses:**

**400 - Missing organizationId:**
```json
{
  "error": "Missing required parameter: organizationId",
  "message": "Please provide organizationId as a query parameter"
}
```

**404 - Organization not found:**
```json
{
  "error": "Organization not found",
  "organizationId": "invalid-org-id"
}
```

**404 - No billing customer:**
```json
{
  "error": "Billing customer not found for this organization",
  "organizationId": "kq8D0URspd5I7uBck8l9",
  "message": "This organization does not have a billing customer ID. Please contact support."
}
```

## Metronome Service Methods

### `createCustomer(params)`

Creates a new customer in Metronome.

**Parameters:**
- `external_id` (required): Firebase organization ID
- `name` (required): Organization name
- `custom_fields` (optional): Custom fields (must be pre-defined in Metronome)

**Returns:**
```javascript
{
  success: true,
  data: {
    id: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    external_id: "kq8D0URspd5I7uBck8l9",
    name: "Sholay",
    ingest_aliases: ["kq8D0URspd5I7uBck8l9"],
    custom_fields: {}
  },
  metronome_id: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
  external_id: "kq8D0URspd5I7uBck8l9"
}
```

**Example:**
```javascript
const metronomeService = require('./services/metronome-service');

const result = await metronomeService.createCustomer({
  external_id: 'kq8D0URspd5I7uBck8l9',
  name: 'Sholay'
});
```

### `getCustomer(id)`

Retrieves a customer by Metronome ID.

**Parameters:**
- `id` (required): Metronome customer ID

**Returns:**
```javascript
{
  success: true,
  data: {
    id: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    external_id: "kq8D0URspd5I7uBck8l9",
    name: "Sholay",
    ingest_aliases: ["kq8D0URspd5I7uBck8l9"],
    custom_fields: {}
  }
}
```

### `listCustomers(options)`

Lists all customers with pagination.

**Parameters:**
- `limit` (optional): Number of results
- `next_page` (optional): Pagination token

**Returns:**
```javascript
{
  success: true,
  data: [...],
  next_page: "token" or null
}
```

### `createFreeTrialContract(customer_id)`

Creates a free trial contract for a customer with 500 credits.

**Parameters:**
- `customer_id` (required): Metronome customer ID

**Returns:**
```javascript
{
  success: true,
  data: {
    id: "contract-abc-123",
    customer_id: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    starting_at: "2025-10-09T15:00:00.000Z",
    credits: [...]
  },
  contract_id: "contract-abc-123",
  credits: 500
}
```

**Example:**
```javascript
const metronomeService = require('./services/metronome-service');

const result = await metronomeService.createFreeTrialContract(
  'c6330dab-5b2a-43bc-a6c4-acc576c1fbe5'
);
```

**Configuration:**
- **Credits**: 500
- **Duration**: 1 year
- **Rate Card ID**: `cf9e5d3f-4941-4889-ad76-1c116981a086`
- **Product ID**: `4983cc7c-30b6-4355-a07a-9d5afd99d1a6`
- **Credit Type ID**: `9cb3fb43-2f8b-4cb9-b1cf-15e5ba7bb08c`
- **Priority**: 1
- **Name**: "Free Trial Credits"

### `getCustomerBalance(customer_id, covering_date)`

Gets the credit balance for a customer.

**Parameters:**
- `customer_id` (required): Metronome customer ID
- `covering_date` (optional): Date to check balance at (ISO 8601 format, defaults to now)

**Returns:**
```javascript
{
  success: true,
  customer_id: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
  total_balance: 425,
  credits: [
    {
      id: "ac615ee0-8480-42c9-bc25-f08e2efd9f65",
      name: "Free Trial Credits",
      balance: 425,
      type: "CREDIT",
      product: {
        id: "4983cc7c-30b6-4355-a07a-9d5afd99d1a6",
        name: "Included Credits"
      },
      access_schedule: {...},
      created_at: "2025-10-09T15:09:36.301000+00:00"
    }
  ],
  next_page: null
}
```

**Example:**
```javascript
const metronomeService = require('./services/metronome-service');

const result = await metronomeService.getCustomerBalance(
  'c6330dab-5b2a-43bc-a6c4-acc576c1fbe5'
);

console.log('Total Balance:', result.total_balance);
```

## Firebase Schema

### Organization Document (`orgs` collection)

```javascript
{
  organizationName: "Sholay",
  organizationUsername: "sholay",
  createdBy: "user-uid-123",
  createdAt: Timestamp,
  updatedAt: Timestamp,
  billingCustomerId: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5" // Generic billing customer ID (currently Metronome)
}
```

## Metronome Customer Schema

```javascript
{
  id: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",        // Metronome-generated ID
  external_id: "kq8D0URspd5I7uBck8l9",               // Firebase organization ID
  name: "Sholay",                                     // Organization name
  ingest_aliases: ["kq8D0URspd5I7uBck8l9"],         // Auto-generated from external_id
  custom_fields: {}                                   // Optional custom fields
}
```

## Key Features

### 1. Automatic Customer Creation with Free Trial
- When an organization is created in Firebase, a Metronome customer is automatically created
- The Firebase organization ID is used as the `external_id` in Metronome
- Metronome auto-generates `ingest_aliases` from the `external_id`
- A free trial contract with 500 credits (1-year duration) is automatically created for the new customer

### 2. Bidirectional Linking
- Organization document stores `billingCustomerId` (generic field for any billing provider)
- Metronome customer stores `external_id` (Firebase organization ID)
- Easy lookup in both directions
- Provider-agnostic design allows switching billing services in the future

### 3. Error Handling
- If Metronome customer creation fails, the organization is still created in Firebase
- Error details are logged and returned in the response
- The integration is non-blocking - organization creation succeeds even if Metronome fails

### 4. Ingest Aliases
- Metronome automatically creates ingest aliases from the `external_id`
- The Firebase organization ID becomes an ingest alias
- This allows usage tracking and billing to reference the organization ID

## Environment Variables

```bash
METRONOME_API_KEY=801659668b1acd307d90b67fdd4c7529b5107d9d5d1a6301e2259d147140a7f4
```

## Error Handling

### Metronome API Errors

**400 Bad Request:**
- Invalid parameters
- Custom fields not pre-defined in Metronome
- Invalid external_id format

**401 Unauthorized:**
- Invalid API key
- Expired API key

**422 Unprocessable Entity:**
- Duplicate external_id
- Invalid data format

**500 Internal Server Error:**
- Metronome service error

### Integration Behavior

- **Metronome fails**: Organization is still created, error logged
- **Firebase fails**: Request fails, no Metronome customer created
- **Partial failure**: Organization created, Metronome error returned in response

## Testing

The integration has been tested with:
- ✅ Customer creation
- ✅ Customer retrieval
- ✅ Customer listing
- ✅ Organization creation with Metronome integration
- ✅ Error handling and fallbacks

## Future Enhancements

1. **Usage Tracking**: Send usage events to Metronome
2. **Billing Plans**: Assign billing plans to customers
3. **Webhooks**: Handle Metronome webhooks for billing events
4. **Sync Tool**: Sync existing organizations to Metronome
5. **Custom Fields**: Add organization metadata as custom fields

## Support

For Metronome API documentation, visit: https://metronome.com/docs

