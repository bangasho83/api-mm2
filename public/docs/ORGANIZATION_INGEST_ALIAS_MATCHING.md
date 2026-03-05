# Organization Creation with Existing Organization Matching

## Overview

When creating an organization, you can optionally provide an `organizationId` to add billing to an existing organization. If the organization exists and doesn't have billing, it will be updated. If it doesn't exist or no ID is provided, a new organization will be created.

## Flow Diagram

```
POST /api/organizations
  ↓
1. Check if organizationId is provided
  ↓
  ├─ YES → Check if org exists in database
  │         ↓
  │         ├─ EXISTS + HAS BILLING → Return existing (200 OK)
  │         ├─ EXISTS + NO BILLING → Update with billing (200 OK)
  │         └─ DOESN'T EXIST → Create new org (201 Created)
  │
  └─ NO → Create new organization (201 Created)
  ↓
2. Create Metronome customer
  ↓
3. Add/Update billingCustomerId
  ↓
4. Create free trial (only for new orgs)
```

## Behavior

### Scenario 1: New Organization (No Match)

**Request:**
```json
POST /api/organizations
{
  "organizationName": "Disney",
  "organizationUsername": "disney",
  "createdBy": "user-123"
}
```

**Process:**
1. ✅ Create Metronome customer with temp org ID
2. ✅ Check ingest_aliases → No match found
3. ✅ Create new organization in Firebase
4. ✅ Add billingCustomerId to organization
5. ✅ Create free trial contract (500 credits)

**Response (201 Created):**
```json
{
  "success": true,
  "isExistingOrg": false,
  "message": "New organization created successfully",
  "organization": {
    "id": "xTPKl1i0bOOI3cr0KWtg",
    "organizationName": "Disney",
    "organizationUsername": "disney",
    "createdBy": "user-123",
    "createdAt": "2025-10-09T12:14:07.000Z",
    "updatedAt": "2025-10-09T12:14:07.000Z",
    "billingCustomerId": "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5"
  },
  "billing": {
    "provider": "metronome",
    "success": true,
    "customerId": "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5",
    "freeTrial": {
      "success": true,
      "contractId": "contract-abc-123",
      "credits": 500
    }
  }
}
```

---

### Scenario 2: Add Billing to Existing Organization

**Existing Organization in Database:**
```javascript
// Firebase orgs collection
{
  id: "0Wt0rGMr1zGgbu2NhJrK",
  organizationName: "Krunch Cheese",
  organizationUsername: "krunchcheese",
  createdBy: "22tBJ3qzGaTVbuwks0EfMzX19wA3",
  createdAt: "2025-10-06T10:14:32.000Z",
  // No billingCustomerId yet
}
```

**Request (WITH organizationId):**
```json
POST /api/organizations
{
  "organizationId": "0Wt0rGMr1zGgbu2NhJrK",
  "organizationName": "Krunch Cheese",
  "organizationUsername": "krunchcheese",
  "createdBy": "22tBJ3qzGaTVbuwks0EfMzX19wA3"
}
```

**Process:**
1. ✅ Check if organizationId "0Wt0rGMr1zGgbu2NhJrK" exists → **FOUND!**
2. ✅ Check if it has billing → **NO**
3. ✅ Create Metronome customer with external_id = "0Wt0rGMr1zGgbu2NhJrK"
4. ✅ Update existing organization with billingCustomerId
5. ✅ Skip free trial (org already exists)
6. ✅ Do NOT create new organization

**Response (200 OK):**
```json
{
  "success": true,
  "isExistingOrg": true,
  "message": "Existing organization updated with billing information",
  "organization": {
    "id": "abc123xyz",
    "organizationName": "Old Disney",
    "organizationUsername": "old-disney",
    "createdBy": "user-456",
    "createdAt": "2025-09-01T10:00:00.000Z",
    "updatedAt": "2025-10-09T12:14:07.000Z",
    "billingCustomerId": "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5"
  },
  "billing": {
    "provider": "metronome",
    "success": true,
    "customerId": "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5",
    "freeTrial": null
  }
}
```

**Database After Update:**
```javascript
// Firebase orgs collection - UPDATED
{
  id: "abc123xyz",
  organizationName: "Old Disney",
  organizationUsername: "old-disney",
  createdBy: "user-456",
  createdAt: "2025-09-01T10:00:00.000Z",
  updatedAt: "2025-10-09T12:14:07.000Z",
  billingCustomerId: "d2ff303d-ffb1-4266-a1fc-4d06fc4bb2c5"  // ← ADDED
}
```

---

## Key Differences

| Aspect | New Organization | Existing Organization |
|--------|-----------------|----------------------|
| **HTTP Status** | 201 Created | 200 OK |
| **isExistingOrg** | `false` | `true` |
| **Message** | "New organization created successfully" | "Existing organization updated with billing information" |
| **Organization ID** | New ID generated | Existing ID from match |
| **Organization Data** | New data from request | Existing data preserved |
| **Free Trial** | Created (500 credits) | Skipped |
| **billingCustomerId** | Added to new org | Added to existing org |

---

## Ingest Alias Matching Logic

```javascript
// 1. Get ingest_aliases from Metronome response
const ingestAliases = metronomeResult.data?.ingest_aliases || [];
// Example: ["abc123xyz", "org-alias-2", "org-alias-3"]

// 2. Check each alias against existing organizations
for (const alias of ingestAliases) {
  const existingOrgDoc = await db.collection('orgs').doc(alias).get();
  
  if (existingOrgDoc.exists) {
    // MATCH FOUND - Update this organization
    matchedOrgId = alias;
    break;
  }
}

// 3. Update or Create based on match
if (matchedOrgId) {
  // Update existing organization
  await db.collection('orgs').doc(matchedOrgId).update({
    billingCustomerId: metronomeResult.metronome_id,
    updatedAt: timestamp
  });
} else {
  // Create new organization
  await db.collection('orgs').doc(newOrgId).set({
    organizationName,
    organizationUsername,
    createdBy,
    billingCustomerId: metronomeResult.metronome_id,
    createdAt: timestamp,
    updatedAt: timestamp
  });
}
```

---

## Benefits

### ✅ Prevents Duplicate Organizations
- If Metronome already has a customer with matching alias, we update the existing org instead of creating a duplicate

### ✅ Preserves Existing Data
- Original organization name, username, creator, and creation date are preserved
- Only `billingCustomerId` and `updatedAt` are updated

### ✅ Proper Billing Linkage
- Existing organizations get properly linked to their Metronome billing customer
- No orphaned billing records

### ✅ Smart Free Trial Handling
- New organizations get free trial (500 credits)
- Existing organizations skip free trial (already have one or don't need it)

---

## Use Cases

### Use Case 1: Migration from Old System
**Scenario:** You have existing organizations without billing, and you're adding Metronome billing.

**Before:**
```javascript
// Existing org in Firebase
{
  id: "old-org-123",
  organizationName: "Acme Corp",
  // No billing
}
```

**Action:** Create organization via API with same details

**After:**
```javascript
// Same org, now with billing
{
  id: "old-org-123",
  organizationName: "Acme Corp",
  billingCustomerId: "metronome-customer-id"  // ← Added
}
```

---

### Use Case 2: Re-creating Organization
**Scenario:** Organization was created before, Metronome remembers it via ingest_alias.

**Process:**
1. User tries to create "Disney" organization
2. Metronome returns ingest_alias matching existing org ID
3. System updates existing org instead of creating duplicate
4. User gets existing organization with billing added

---

### Use Case 3: Fresh Organization
**Scenario:** Brand new organization, never existed before.

**Process:**
1. User creates "New Company" organization
2. Metronome creates customer, no matching ingest_alias
3. System creates new organization
4. Free trial contract created
5. User gets new organization with billing

---

## Response Fields

### New Field: `isExistingOrg`

```javascript
{
  "isExistingOrg": true | false
}
```

- `true` - Existing organization was updated with billing
- `false` - New organization was created

### New Field: `message`

```javascript
{
  "message": "New organization created successfully"
  // OR
  "message": "Existing organization updated with billing information"
}
```

---

## Testing

### Test 1: Create New Organization
```bash
curl -X POST https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationName": "Test Org",
    "organizationUsername": "test-org",
    "createdBy": "user-123"
  }'
```

**Expected:** 201 Created, `isExistingOrg: false`, free trial created

---

### Test 2: Update Existing Organization
```bash
# First, manually create an org in Firebase with ID "test-org-existing"

# Then call API
curl -X POST https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationName": "Test Org",
    "organizationUsername": "test-org",
    "createdBy": "user-123"
  }'

# If Metronome returns ingest_alias "test-org-existing"
```

**Expected:** 200 OK, `isExistingOrg: true`, no free trial

---

## Error Handling

### Metronome Fails
If Metronome customer creation fails, the system still creates the organization without billing:

```json
{
  "success": true,
  "isExistingOrg": false,
  "message": "New organization created successfully",
  "organization": {
    "id": "new-org-id",
    "organizationName": "Disney",
    "billingCustomerId": null  // ← No billing
  },
  "billing": null
}
```

---

## Logging

### New Organization
```
Creating Metronome customer: { external_id: 'temp-org-id', name: 'Disney' }
Metronome customer created: { metronome_id: 'cust-123', ingest_aliases: [] }
Checking ingest_aliases for existing organizations: []
Created new organization in Firebase: { id: 'temp-org-id', billingCustomerId: 'cust-123' }
Free trial contract created: { contract_id: 'contract-123', credits: 500 }
```

### Existing Organization
```
Creating Metronome customer: { external_id: 'temp-org-id', name: 'Disney' }
Metronome customer created: { metronome_id: 'cust-123', ingest_aliases: ['abc123xyz'] }
Checking ingest_aliases for existing organizations: ['abc123xyz']
Found existing organization matching ingest_alias: { alias: 'abc123xyz', orgId: 'abc123xyz' }
Updated existing organization with billing: { organizationId: 'abc123xyz', billingCustomerId: 'cust-123' }
Skipping free trial for existing organization
```

---

## Summary

- ✅ **Smart Matching** - Checks ingest_aliases against existing org IDs
- ✅ **Update vs Create** - Updates existing org or creates new one
- ✅ **Data Preservation** - Existing org data is preserved
- ✅ **Free Trial Logic** - Only new orgs get free trial
- ✅ **Clear Response** - `isExistingOrg` flag indicates behavior
- ✅ **Proper Status Codes** - 201 for new, 200 for update

