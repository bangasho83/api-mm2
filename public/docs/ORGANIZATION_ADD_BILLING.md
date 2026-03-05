# Add Billing to Existing Organization

## Overview

Use the **PATCH** endpoint to add billing (Metronome customer) to an existing organization that doesn't have billing configured yet.

## Endpoint

**Method:** `PATCH /api/organizations`

**Purpose:** Add billing to an existing organization

---

## Request

### Request Body

```json
{
  "organizationId": "0Wt0rGMr1zGgbu2NhJrK"
}
```

### Required Fields

- `organizationId` (string) - The Firebase organization ID

---

## Response Examples

### ✅ Success (200 OK)

**Billing added successfully:**

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

**Organization already has billing:**

```json
{
  "success": true,
  "message": "Organization already has billing configured",
  "organization": {
    "id": "0Wt0rGMr1zGgbu2NhJrK",
    "organizationName": "Krunch Cheese",
    "organizationUsername": "krunchcheese",
    "createdBy": "22tBJ3qzGaTVbuwks0EfMzX19wA3",
    "createdAt": "2025-10-06T10:14:32.000Z",
    "updatedAt": "2025-10-09T17:00:00.000Z",
    "billingCustomerId": "existing-customer-id"
  },
  "billing": {
    "provider": "metronome",
    "customerId": "existing-customer-id"
  }
}
```

---

### ❌ Error Responses

#### 400 - Missing organizationId

```json
{
  "error": "Missing or invalid required field: organizationId",
  "message": "organizationId is required to add billing to existing organization"
}
```

#### 404 - Organization Not Found

```json
{
  "error": "Organization not found",
  "organizationId": "invalid-id",
  "message": "The specified organization does not exist"
}
```

#### 410 - Organization Deleted

```json
{
  "error": "Organization has been deleted",
  "organizationId": "0Wt0rGMr1zGgbu2NhJrK",
  "deletedAt": "2025-10-09T15:30:00.000Z"
}
```

#### 500 - Metronome Error

```json
{
  "error": "Failed to create billing customer",
  "details": "Metronome API error message"
}
```

---

## cURL Examples

### Basic Request

```bash
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "0Wt0rGMr1zGgbu2NhJrK"
  }'
```

### With Pretty Print

```bash
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "0Wt0rGMr1zGgbu2NhJrK"
  }' | jq
```

### One-Liner

```bash
curl -X PATCH https://social-apis-two.vercel.app/api/organizations -H "Content-Type: application/json" -d '{"organizationId":"0Wt0rGMr1zGgbu2NhJrK"}'
```

---

## What Happens

### Flow

```
1. Receive PATCH request with organizationId
   ↓
2. Check if organization exists
   ↓
3. Check if organization is deleted
   ↓
4. Check if organization already has billing
   ↓
   ├─ YES → Return existing billing info (200 OK)
   │
   └─ NO → Continue to step 5
   ↓
5. Create Metronome customer
   ↓
6. Update organization with billingCustomerId
   ↓
7. Create free trial contract (500 credits, 1 year)
   ↓
8. Return success (200 OK)
```

### Database Changes

**Before:**
```javascript
{
  id: "0Wt0rGMr1zGgbu2NhJrK",
  organizationName: "Krunch Cheese",
  organizationUsername: "krunchcheese",
  createdBy: "22tBJ3qzGaTVbuwks0EfMzX19wA3",
  createdAt: Timestamp,
  updatedAt: Timestamp
  // No billingCustomerId
}
```

**After:**
```javascript
{
  id: "0Wt0rGMr1zGgbu2NhJrK",
  organizationName: "Krunch Cheese",
  organizationUsername: "krunchcheese",
  createdBy: "22tBJ3qzGaTVbuwks0EfMzX19wA3",
  createdAt: Timestamp,
  updatedAt: Timestamp,  // ← Updated
  billingCustomerId: "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5"  // ← Added
}
```

---

## Key Features

### ✅ Idempotent
- Safe to call multiple times
- If billing already exists, returns existing info
- No duplicate Metronome customers created

### ✅ Data Preservation
- Original organization data is preserved
- Only `billingCustomerId` and `updatedAt` are modified
- Organization name, username, creator remain unchanged

### ✅ Free Trial Included
- PATCH creates free trial (500 credits, 1 year)
- Same as POST endpoint

### ✅ Validation
- Checks if organization exists
- Checks if organization is deleted
- Checks if billing already configured

---

## Use Cases

### Use Case 1: Migration

Add billing to organizations created before billing system was implemented:

```bash
# Get all organizations without billing
curl -X GET https://social-apis-two.vercel.app/api/organizations | \
  jq '.organizations[] | select(.billingCustomerId == null) | .id'

# Output: List of org IDs without billing
# "0Wt0rGMr1zGgbu2NhJrK"
# "abc123xyz"
# "def456uvw"

# Add billing to each one
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "0Wt0rGMr1zGgbu2NhJrK"}'
```

### Use Case 2: Retry Failed Billing Setup

If billing creation failed during organization creation:

```bash
# Organization was created but billing failed
# Retry adding billing
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "failed-billing-org-id"}'
```

### Use Case 3: Manual Billing Addition

Admin manually adds billing to specific organization:

```bash
# Admin identifies org that needs billing
# Add billing via PATCH
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "manual-org-id"}'
```

---

## Comparison: POST vs PATCH

| Aspect | POST (Create) | PATCH (Add Billing) |
|--------|--------------|---------------------|
| **Purpose** | Create new organization | Add billing to existing org |
| **organizationId** | Auto-generated | Required in request |
| **Organization** | Creates new | Uses existing |
| **Free Trial** | ✅ Created (500 credits) | ✅ Created (500 credits) |
| **Status Code** | 201 Created | 200 OK |
| **Idempotent** | ❌ No (creates duplicate) | ✅ Yes (safe to retry) |
| **Use Case** | New organizations | Migration, retry, manual |

---

## Frontend Integration

### React Example

```javascript
async function addBillingToOrganization(organizationId) {
  try {
    const response = await fetch('/api/organizations', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ organizationId })
    });

    const data = await response.json();

    if (response.ok) {
      if (data.message.includes('already has billing')) {
        console.log('Billing already configured');
      } else {
        console.log('Billing added successfully');
      }
      return data.organization;
    } else {
      console.error('Failed to add billing:', data.error);
      throw new Error(data.error);
    }
  } catch (error) {
    console.error('Error adding billing:', error);
    throw error;
  }
}
```

### Migration Script

```javascript
async function migrateOrganizationsToBilling() {
  // Get all organizations
  const response = await fetch('/api/organizations');
  const { organizations } = await response.json();

  // Filter organizations without billing
  const orgsWithoutBilling = organizations.filter(
    org => !org.billingCustomerId
  );

  console.log(`Found ${orgsWithoutBilling.length} organizations without billing`);

  // Add billing to each
  for (const org of orgsWithoutBilling) {
    console.log(`Adding billing to: ${org.organizationName}`);
    
    try {
      await fetch('/api/organizations', {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ organizationId: org.id })
      });
      
      console.log(`✅ Billing added to: ${org.organizationName}`);
    } catch (error) {
      console.error(`❌ Failed for: ${org.organizationName}`, error);
    }
  }

  console.log('Migration complete!');
}
```

---

## Testing

### Test 1: Add Billing to Existing Org

```bash
# Step 1: Verify org exists without billing
curl -X GET "https://social-apis-two.vercel.app/api/organizations?organizationId=0Wt0rGMr1zGgbu2NhJrK"

# Step 2: Add billing
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "0Wt0rGMr1zGgbu2NhJrK"}'

# Step 3: Verify billing was added
curl -X GET "https://social-apis-two.vercel.app/api/organizations?organizationId=0Wt0rGMr1zGgbu2NhJrK"
```

### Test 2: Idempotency

```bash
# Call PATCH twice with same org ID
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "0Wt0rGMr1zGgbu2NhJrK"}'

# Second call should return "already has billing"
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "0Wt0rGMr1zGgbu2NhJrK"}'
```

### Test 3: Error Cases

```bash
# Missing organizationId
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{}'

# Invalid organizationId
curl -X PATCH https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "invalid-id"}'
```

---

## Summary

- 🎯 **Purpose**: Add billing to existing organizations
- 📝 **Method**: PATCH /api/organizations
- ✅ **Idempotent**: Safe to call multiple times
- 🔒 **Secure**: Validates organization exists and not deleted
- 💾 **Preserves Data**: Only updates billingCustomerId
- 🎁 **Free Trial**: Creates 500 credits, 1 year duration
- 🔄 **Use Cases**: Migration, retry, manual billing addition

