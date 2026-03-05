# Organization Stripe Integration

This document describes the Stripe integration added to the Organizations API.

## Overview

When an organization is created or updated (via POST or PATCH), the system now automatically creates **both** a Metronome customer and a Stripe customer. This dual-billing setup allows for:
- **Metronome**: Usage-based billing and credit tracking
- **Stripe**: Payment processing and subscription management

## What Changed

### 1. Organizations API (`api/organizations.js`)

**Added:**
- Import of `stripe-service`
- Automatic Stripe customer creation after Metronome customer creation
- User email retrieval from Firebase `users` collection
- Storage of `stripeCustomerId` in organization document
- Stripe customer information in API responses

### 2. Firebase Schema

**Organization Document (`orgs` collection):**

```javascript
{
  organizationName: "Acme Corp",
  organizationUsername: "acmecorp",
  createdBy: "user-uid-123",
  createdAt: Timestamp,
  updatedAt: Timestamp,
  billingCustomerId: "metronome-customer-id",  // Existing
  stripeCustomerId: "cus_ABC123"               // NEW - Stripe customer ID
}
```

### 3. API Response Format

**POST /api/organizations Response:**

```json
{
  "success": true,
  "organization": {
    "id": "org-id-123",
    "organizationName": "Acme Corp",
    "organizationUsername": "acmecorp",
    "createdBy": "user-uid-123",
    "createdAt": "2025-10-10T15:00:00.000Z",
    "updatedAt": "2025-10-10T15:00:00.000Z",
    "billingCustomerId": "metronome-customer-id",
    "stripeCustomerId": "cus_ABC123"
  },
  "billing": {
    "metronome": {
      "success": true,
      "customerId": "metronome-customer-id",
      "error": null,
      "freeTrial": {
        "success": true,
        "contractId": "contract-id",
        "credits": 500,
        "error": null
      }
    },
    "stripe": {
      "success": true,
      "customerId": "cus_ABC123",
      "email": "user@example.com",
      "error": null
    }
  }
}
```

## Flow Diagram

### POST /api/organizations

```
1. Validate request (organizationName, organizationUsername, createdBy)
   ↓
2. Create organization in Firebase
   ↓
3. Fetch user email from users collection (using createdBy)
   ↓
4. Create Metronome customer
   ├─ Store billingCustomerId in organization
   └─ Create free trial contract (500 credits)
   ↓
5. Create Stripe customer (if user email exists)
   ├─ Use organization name
   ├─ Use user email
   ├─ Add metadata (organization_id, metronome_customer_id, etc.)
   └─ Store stripeCustomerId in organization
   ↓
6. Return response with both customer IDs
```

### PATCH /api/organizations

```
1. Validate organizationId
   ↓
2. Check if organization exists
   ↓
3. Fetch user email from users collection (using createdBy from org)
   ↓
4. Create Metronome customer (if not exists)
   ├─ Store billingCustomerId
   └─ Create free trial (if new customer)
   ↓
5. Create Stripe customer (if user email exists)
   ├─ Use organization name
   ├─ Use user email
   ├─ Add metadata
   └─ Store stripeCustomerId
   ↓
6. Return response with both customer IDs
```

## Stripe Customer Metadata

Each Stripe customer is created with the following metadata:

```javascript
{
  organization_id: "firebase-org-id",
  organization_username: "acmecorp",
  created_by: "user-uid-123",
  metronome_customer_id: "metronome-customer-id"
}
```

This metadata allows you to:
- Link Stripe customers back to your Firebase organizations
- Cross-reference between Stripe and Metronome
- Track which user created the organization

## Error Handling

The integration is designed to be **fault-tolerant**:

1. **If user email is not found**: Stripe customer creation is skipped, but organization creation continues
2. **If Stripe customer creation fails**: Error is logged, but organization creation continues
3. **If Metronome customer creation fails**: Error is returned, but organization is still created in Firebase

This ensures that organization creation always succeeds, even if billing provider integrations fail.

## Testing

### Test Script

A test script is provided at `scripts/test-org-with-stripe.js`:

```bash
# Test organization creation with Stripe
node scripts/test-org-with-stripe.js

# Test adding billing to existing organization
node scripts/test-org-with-stripe.js <organizationId>
```

### Manual Testing

**1. Create Organization:**

```bash
curl -X POST https://your-api.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationName": "Test Corp",
    "organizationUsername": "testcorp",
    "createdBy": "user-uid-123"
  }'
```

**2. Verify in Stripe Dashboard:**
- Go to https://dashboard.stripe.com/test/customers
- Search for "Test Corp"
- Check metadata for organization_id

**3. Verify in Firebase:**
- Check `orgs` collection
- Verify `stripeCustomerId` field is populated

## Requirements

### Environment Variables

```env
STRIPE_SECRET_KEY=sk_test_your_stripe_secret_key
```

### User Collection

The `users` collection must have documents with the following structure:

```javascript
{
  uid: "user-uid-123",
  email: "user@example.com",  // Required for Stripe
  name: "John Doe",
  organizationId: "org-id-123",
  // ... other fields
}
```

**Important:** The user document must exist and have an `email` field for Stripe customer creation to work.

## Benefits

1. **Dual Billing Setup**: Ready for both usage-based (Metronome) and subscription-based (Stripe) billing
2. **Automatic Linking**: Stripe customers are automatically linked to organizations via metadata
3. **Email Tracking**: User email is captured in Stripe for communication and invoicing
4. **Fault Tolerant**: Organization creation succeeds even if billing providers fail
5. **Audit Trail**: Both customer IDs stored in Firebase for easy reference

## Next Steps

You can now use the Stripe customer ID to:
- Create payment methods
- Set up subscriptions
- Process one-time payments
- Generate invoices
- Create customer portal sessions

See `docs/STRIPE_INTEGRATION.md` for more details on using the Stripe service.

