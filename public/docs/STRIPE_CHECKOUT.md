# Stripe Checkout Session API

This document describes the Stripe Checkout Session API for creating payment method setup flows.

## Overview

The Stripe Checkout Session API allows you to create a hosted checkout page where users can add payment methods to their account. This is useful for:
- Setting up payment methods for future charges
- Collecting payment information securely
- Creating subscriptions
- Processing one-time payments

## Endpoint

**POST** `/api/billing/checkout`

## Request Parameters

### Required (one of):
- `organizationId` (string) - Firebase organization ID (will fetch stripeCustomerId automatically)
- `stripeCustomerId` (string) - Stripe customer ID (if you already have it)

### Optional:
- `successUrl` (string) - URL to redirect after successful setup
  - Default: `https://yourapp.example/billing/success?session_id={CHECKOUT_SESSION_ID}`
  - Use `{CHECKOUT_SESSION_ID}` placeholder to receive the session ID
- `cancelUrl` (string) - URL to redirect if user cancels
  - Default: `https://yourapp.example/billing/cancel`
- `mode` (string) - Session mode: `'setup'`, `'payment'`, or `'subscription'`
  - Default: `'setup'`
- `currency` (string) - Currency code (e.g., 'usd', 'eur', 'gbp')
  - Default: `'usd'`
- `metadata` (object) - Custom metadata to attach to the session

## Response

### Success Response (200)

```json
{
  "success": true,
  "message": "Checkout session created successfully",
  "session_id": "cs_test_a1b2c3d4e5f6g7h8i9j0",
  "url": "https://checkout.stripe.com/c/pay/cs_test_...",
  "customer": "cus_ABC123",
  "mode": "setup",
  "status": "open",
  "data": {
    "id": "cs_test_a1b2c3d4e5f6g7h8i9j0",
    "object": "checkout.session",
    "customer": "cus_ABC123",
    "url": "https://checkout.stripe.com/c/pay/cs_test_...",
    "mode": "setup",
    "status": "open",
    "success_url": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
    "cancel_url": "https://myapp.com/billing/cancel",
    ...
  }
}
```

### Error Responses

**400 - Missing Parameters**
```json
{
  "error": "Missing required parameter",
  "message": "Either organizationId or stripeCustomerId must be provided"
}
```

**404 - Organization Not Found**
```json
{
  "error": "Organization not found",
  "message": "Organization with ID org_123 does not exist"
}
```

**400 - No Stripe Customer**
```json
{
  "error": "No Stripe customer found for this organization",
  "message": "This organization does not have a Stripe customer ID. Please create one first.",
  "organizationId": "org_123"
}
```

## Usage Examples

### Example 1: Using Organization ID

```bash
curl -X POST https://social-apis-two.vercel.app/api/billing/checkout \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "kq8D0URspd5I7uBck8l9",
    "successUrl": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
    "cancelUrl": "https://myapp.com/billing/cancel",
    "mode": "setup",
    "currency": "usd"
  }'
```

### Example 2: Using Stripe Customer ID Directly

```bash
curl -X POST https://social-apis-two.vercel.app/api/billing/checkout \
  -H "Content-Type: application/json" \
  -d '{
    "stripeCustomerId": "cus_ABC123",
    "successUrl": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
    "cancelUrl": "https://myapp.com/billing/cancel"
  }'
```

### Example 3: With Custom Metadata

```bash
curl -X POST https://social-apis-two.vercel.app/api/billing/checkout \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "org_123",
    "successUrl": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
    "cancelUrl": "https://myapp.com/billing/cancel",
    "mode": "setup",
    "metadata": {
      "user_id": "user_456",
      "plan": "premium",
      "source": "settings_page"
    }
  }'
```

### Example 4: JavaScript/Frontend Integration

```javascript
async function setupPaymentMethod(organizationId) {
  try {
    const response = await fetch('https://social-apis-two.vercel.app/api/billing/checkout', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        organizationId: organizationId,
        successUrl: `${window.location.origin}/billing/success?session_id={CHECKOUT_SESSION_ID}`,
        cancelUrl: `${window.location.origin}/billing/cancel`,
        mode: 'setup',
        currency: 'usd'
      })
    });

    const data = await response.json();

    if (data.success) {
      // Redirect user to Stripe Checkout
      window.location.href = data.url;
    } else {
      console.error('Failed to create checkout session:', data.error);
    }
  } catch (error) {
    console.error('Error:', error);
  }
}

// Usage
setupPaymentMethod('org_123');
```

## Flow Diagram

```
1. Frontend calls /api/billing/checkout with organizationId
   ↓
2. API fetches stripeCustomerId from Firebase
   ↓
3. API creates Stripe checkout session
   ↓
4. API returns checkout URL
   ↓
5. Frontend redirects user to checkout URL
   ↓
6. User completes payment method setup on Stripe
   ↓
7. Stripe redirects to success_url with session_id
   ↓
8. Frontend handles success (optional: verify session)
```

## Handling Success

When the user completes the checkout, they'll be redirected to your `success_url` with the session ID:

```
https://myapp.com/billing/success?session_id=cs_test_a1b2c3d4e5f6g7h8i9j0
```

You can then:
1. Extract the session ID from the URL
2. Optionally verify the session with Stripe
3. Update your database
4. Show a success message to the user

### Example Success Handler

```javascript
// On your success page
const urlParams = new URLSearchParams(window.location.search);
const sessionId = urlParams.get('session_id');

if (sessionId) {
  console.log('Payment method setup completed!');
  console.log('Session ID:', sessionId);
  
  // Optionally verify the session on your backend
  // and update your database
  
  // Show success message to user
  alert('Payment method added successfully!');
}
```

## Session Modes

### 1. Setup Mode (default)
- Collects payment method for future use
- Does not charge the customer
- Creates a SetupIntent in Stripe
- Best for: Adding payment methods to customer accounts

### 2. Payment Mode
- Collects one-time payment
- Charges the customer immediately
- Creates a PaymentIntent in Stripe
- Best for: One-time purchases

### 3. Subscription Mode
- Creates a subscription
- Charges the customer on a recurring basis
- Creates a Subscription in Stripe
- Best for: Recurring billing

## Security Notes

1. **Never expose Stripe secret keys** to the frontend
2. **Always validate** organizationId on the backend
3. **Use HTTPS** for all redirect URLs
4. **Verify session** on your backend after redirect
5. **Check session status** before granting access

## Testing

### Test Script

```bash
node scripts/test-stripe-checkout.js
```

### Manual Testing

1. Create a checkout session using the API
2. Open the returned URL in your browser
3. Use Stripe test card: `4242 4242 4242 4242`
4. Use any future expiry date and any CVC
5. Complete the flow
6. Verify redirect to success URL

### Test Cards

- **Success**: 4242 4242 4242 4242
- **Decline**: 4000 0000 0000 0002
- **Requires Authentication**: 4000 0025 0000 3155

## Next Steps

After setting up a payment method, you can:
- Charge the customer using the saved payment method
- Create subscriptions
- Set up automatic billing
- Generate invoices

See the [Stripe Integration Documentation](./STRIPE_INTEGRATION.md) for more details.

