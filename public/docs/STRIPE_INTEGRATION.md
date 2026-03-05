# Stripe Integration

This document describes the Stripe service integration for customer management and payment processing.

## Overview

The Stripe service (`services/stripe-service.js`) provides a clean interface for interacting with the Stripe API. It follows the same pattern as the Metronome service and includes comprehensive error handling, validation, and logging.

## Configuration

### Environment Variables

Add the following to your `.env` file:

```env
STRIPE_SECRET_KEY=sk_test_your_stripe_secret_key_here
```

**Important:** 
- Use `sk_test_*` keys for testing/development
- Use `sk_live_*` keys for production
- Never commit your secret keys to version control

## Available Functions

### 1. createCustomer

Creates a new customer in Stripe.

**Parameters:**
```javascript
{
  name: string,           // Required - Customer name
  email: string,          // Required - Customer email
  description: string,    // Optional - Customer description
  metadata: object,       // Optional - Custom metadata key-value pairs
  phone: string,          // Optional - Phone number
  address: {              // Optional - Address object
    line1: string,
    line2: string,
    city: string,
    state: string,
    postal_code: string,
    country: string
  }
}
```

**Returns:**
```javascript
{
  success: true,
  data: {...},           // Full Stripe customer object
  stripe_id: "cus_...",  // Stripe customer ID
  email: "...",
  name: "..."
}
```

**Example:**
```javascript
const stripeService = require('./services/stripe-service');

const result = await stripeService.createCustomer({
  name: 'John Doe',
  email: 'john@example.com',
  description: 'Premium customer',
  metadata: {
    organization_id: 'org_123',
    plan: 'premium'
  }
});

if (result.success) {
  console.log('Customer created:', result.stripe_id);
}
```

### 2. getCustomer

Retrieves a customer by their Stripe customer ID.

**Parameters:**
```javascript
customer_id: string  // Stripe customer ID (e.g., "cus_...")
```

**Returns:**
```javascript
{
  success: true,
  data: {...}  // Full Stripe customer object
}
```

**Example:**
```javascript
const result = await stripeService.getCustomer('cus_TD89hNNQbqpAYc');

if (result.success) {
  console.log('Customer:', result.data.name);
}
```

### 3. listCustomers

Lists all customers with optional filtering and pagination.

**Parameters:**
```javascript
{
  limit: number,           // Optional - Max 100, default varies
  starting_after: string,  // Optional - Pagination cursor
  ending_before: string,   // Optional - Pagination cursor
  email: string           // Optional - Filter by email
}
```

**Returns:**
```javascript
{
  success: true,
  data: [...],      // Array of customer objects
  has_more: boolean,
  url: string
}
```

**Example:**
```javascript
const result = await stripeService.listCustomers({ 
  limit: 10,
  email: 'john@example.com'
});

if (result.success) {
  console.log('Found customers:', result.data.length);
}
```

### 4. updateCustomer

Updates an existing customer's information.

**Parameters:**
```javascript
customer_id: string,  // Required - Stripe customer ID
updates: {            // Object with fields to update
  name: string,
  email: string,
  description: string,
  metadata: object,
  phone: string,
  address: object
}
```

**Returns:**
```javascript
{
  success: true,
  data: {...}  // Updated customer object
}
```

**Example:**
```javascript
const result = await stripeService.updateCustomer('cus_TD89hNNQbqpAYc', {
  description: 'Updated description',
  metadata: {
    updated_at: new Date().toISOString()
  }
});

if (result.success) {
  console.log('Customer updated');
}
```

### 5. createCheckoutSession

Creates a Stripe checkout session for payment method setup, payments, or subscriptions.

**Parameters:**
```javascript
{
  customer_id: string,      // Required - Stripe customer ID
  success_url: string,      // Optional - Redirect URL after success
  cancel_url: string,       // Optional - Redirect URL if cancelled
  mode: string,             // Optional - 'setup', 'payment', or 'subscription' (default: 'setup')
  currency: string,         // Optional - Currency code (default: 'usd')
  metadata: object          // Optional - Custom metadata
}
```

**Returns:**
```javascript
{
  success: true,
  session_id: "cs_test_...",  // Checkout session ID
  url: "https://checkout.stripe.com/...",  // Checkout URL
  customer: "cus_...",
  mode: "setup",
  status: "open",
  data: {...}  // Full session object
}
```

**Example:**
```javascript
const result = await stripeService.createCheckoutSession({
  customer_id: 'cus_TD89hNNQbqpAYc',
  success_url: 'https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}',
  cancel_url: 'https://myapp.com/billing/cancel',
  mode: 'setup',
  currency: 'usd',
  metadata: {
    organization_id: 'org_123'
  }
});

if (result.success) {
  console.log('Checkout URL:', result.url);
  // Redirect user to result.url
}
```

### 6. hasPaymentMethod

Checks if a customer has payment methods added and if one is set as default.

**Parameters:**
```javascript
customer_id: string  // Required - Stripe customer ID
```

**Returns:**
```javascript
{
  success: true,
  has_payment_method: true,           // Whether customer has any payment methods
  payment_methods_count: 2,           // Total number of payment methods
  has_default_payment_method: true,   // Whether a default is set
  default_payment_method_id: "pm_...", // ID of default payment method
  default_payment_method: {           // Details of default payment method
    id: "pm_123",
    type: "card",
    card: {
      brand: "visa",
      last4: "4242",
      exp_month: 12,
      exp_year: 2025
    },
    created: 1234567890
  },
  payment_methods: [                  // Array of all payment methods
    {
      id: "pm_123",
      type: "card",
      card: {
        brand: "visa",
        last4: "4242",
        exp_month: 12,
        exp_year: 2025
      },
      created: 1234567890,
      is_default: true
    }
  ]
}
```

**Example:**
```javascript
const result = await stripeService.hasPaymentMethod('cus_TD89hNNQbqpAYc');

if (result.success) {
  if (result.has_payment_method) {
    console.log('Customer has payment methods!');
    console.log('Default card:', result.default_payment_method.card.brand,
                '****' + result.default_payment_method.card.last4);
  } else {
    console.log('Customer has no payment methods');
    // Redirect to checkout to add one
  }
}
```

### 7. getPaymentMethod

Retrieves detailed information about a specific payment method.

**Parameters:**
```javascript
payment_method_id: string  // Required - Stripe payment method ID (e.g., "pm_...")
```

**Returns:**
```javascript
{
  success: true,
  payment_method_id: "pm_123",
  type: "card",
  card: {
    brand: "visa",
    last4: "4242",
    exp_month: 12,
    exp_year: 2025,
    funding: "credit",
    country: "US"
  },
  customer: "cus_ABC123",
  created: 1234567890,
  billing_details: {
    name: "John Doe",
    email: "john@example.com",
    phone: "+1234567890",
    address: {...}
  },
  data: {...}  // Full Stripe payment method object
}
```

**Example:**
```javascript
const result = await stripeService.getPaymentMethod('pm_1ABC123');

if (result.success) {
  console.log('Payment Method:', result.type);
  if (result.card) {
    console.log('Card:', result.card.brand, '****' + result.card.last4);
    console.log('Expiry:', result.card.exp_month + '/' + result.card.exp_year);
    console.log('Funding:', result.card.funding);
  }
  console.log('Billing Name:', result.billing_details.name);
}
```

## Error Handling

All functions return a consistent error structure:

```javascript
{
  success: false,
  error: "Error message",
  error_type: "stripe_error_type",  // For Stripe-specific errors
  error_code: "error_code",         // For Stripe-specific errors
  details: {...},                   // Additional error details
  status: 400                       // HTTP status code
}
```

**Common Error Types:**
- `invalid_request_error` - Invalid parameters
- `api_error` - Stripe API error
- `card_error` - Card-related error
- `rate_limit_error` - Too many requests

## API Endpoints

### 1. Create Customer

**Endpoint:** `POST /api/stripe/create-customer`

**Request Body:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "description": "Premium customer",
  "metadata": {
    "organization_id": "org_123",
    "plan": "premium"
  },
  "phone": "+1234567890",
  "address": {
    "line1": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94102",
    "country": "US"
  }
}
```

**Response:**
```json
{
  "success": true,
  "message": "Customer created successfully",
  "stripe_id": "cus_TD89hNNQbqpAYc",
  "customer": {
    "id": "cus_TD89hNNQbqpAYc",
    "name": "John Doe",
    "email": "john@example.com",
    ...
  }
}
```

### 2. Create Checkout Session

**Endpoint:** `POST /api/billing/checkout`

**Request Body (using organizationId):**
```json
{
  "organizationId": "org_123",
  "successUrl": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
  "cancelUrl": "https://myapp.com/billing/cancel",
  "mode": "setup",
  "currency": "usd",
  "metadata": {
    "custom_field": "value"
  }
}
```

**Request Body (using stripeCustomerId directly):**
```json
{
  "stripeCustomerId": "cus_ABC123",
  "successUrl": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
  "cancelUrl": "https://myapp.com/billing/cancel",
  "mode": "setup"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Checkout session created successfully",
  "session_id": "cs_test_a1b2c3d4e5f6g7h8i9j0",
  "url": "https://checkout.stripe.com/c/pay/cs_test_a1b2c3d4e5f6g7h8i9j0",
  "customer": "cus_ABC123",
  "mode": "setup",
  "status": "open",
  "data": {
    "id": "cs_test_a1b2c3d4e5f6g7h8i9j0",
    "object": "checkout.session",
    "customer": "cus_ABC123",
    "url": "https://checkout.stripe.com/c/pay/cs_test_a1b2c3d4e5f6g7h8i9j0",
    ...
  }
}
```

**cURL Example:**
```bash
curl -X POST https://social-apis-two.vercel.app/api/billing/checkout \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "your-org-id",
    "successUrl": "https://myapp.com/billing/success?session_id={CHECKOUT_SESSION_ID}",
    "cancelUrl": "https://myapp.com/billing/cancel",
    "mode": "setup",
    "currency": "usd"
  }'
```

**Usage Flow:**
1. Call the endpoint with organizationId or stripeCustomerId
2. Receive the checkout session URL
3. Redirect user to the URL
4. User completes payment method setup
5. User is redirected to your success URL with session_id
6. Verify the session and update your database

### 3. Check Payment Method Status

**Endpoint:** `GET /api/billing/has-payment`

**Query Parameters (one required):**
- `organizationId` - Firebase organization ID
- `stripeCustomerId` - Stripe customer ID

**Response (has payment methods):**
```json
{
  "success": true,
  "has_payment_method": true,
  "payment_methods_count": 2,
  "has_default_payment_method": true,
  "default_payment_method_id": "pm_1ABC123",
  "default_payment_method": {
    "id": "pm_1ABC123",
    "type": "card",
    "card": {
      "brand": "visa",
      "last4": "4242",
      "exp_month": 12,
      "exp_year": 2025
    },
    "created": 1234567890
  },
  "payment_methods": [
    {
      "id": "pm_1ABC123",
      "type": "card",
      "card": {
        "brand": "visa",
        "last4": "4242",
        "exp_month": 12,
        "exp_year": 2025
      },
      "created": 1234567890,
      "is_default": true
    },
    {
      "id": "pm_2DEF456",
      "type": "card",
      "card": {
        "brand": "mastercard",
        "last4": "5555",
        "exp_month": 6,
        "exp_year": 2026
      },
      "created": 1234567891,
      "is_default": false
    }
  ]
}
```

**Response (no payment methods):**
```json
{
  "success": true,
  "has_payment_method": false,
  "payment_methods_count": 0,
  "has_default_payment_method": false,
  "default_payment_method": null,
  "payment_methods": []
}
```

**cURL Example:**
```bash
# Using organization ID
curl -X GET "https://social-apis-two.vercel.app/api/billing/has-payment?organizationId=org_123"

# Using Stripe customer ID
curl -X GET "https://social-apis-two.vercel.app/api/billing/has-payment?stripeCustomerId=cus_ABC123"
```

**Usage Example:**
```javascript
async function checkPaymentMethod(organizationId) {
  const response = await fetch(
    `https://social-apis-two.vercel.app/api/billing/has-payment?organizationId=${organizationId}`
  );
  const data = await response.json();

  if (data.success) {
    if (data.has_payment_method) {
      console.log('Payment method on file:',
        data.default_payment_method.card.brand,
        '****' + data.default_payment_method.card.last4
      );
      // Show billing dashboard
    } else {
      console.log('No payment method on file');
      // Redirect to add payment method
    }
  }
}
```

### 4. Get Payment Method Details

**Endpoint:** `GET /api/billing/payment-info`

**Query Parameters:**
- `paymentMethodId` (required) - Stripe payment method ID

**Response:**
```json
{
  "success": true,
  "payment_method_id": "pm_1ABC123",
  "type": "card",
  "card": {
    "brand": "visa",
    "last4": "4242",
    "exp_month": 12,
    "exp_year": 2025,
    "funding": "credit",
    "country": "US"
  },
  "customer": "cus_ABC123",
  "created": 1234567890,
  "billing_details": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1234567890",
    "address": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94102",
      "country": "US"
    }
  },
  "data": {
    "id": "pm_1ABC123",
    "object": "payment_method",
    ...
  }
}
```

**cURL Example:**
```bash
curl -X GET "https://social-apis-two.vercel.app/api/billing/payment-info?paymentMethodId=pm_1ABC123"
```

**Usage Example:**
```javascript
async function displayPaymentMethod(paymentMethodId) {
  const response = await fetch(
    `https://social-apis-two.vercel.app/api/billing/payment-info?paymentMethodId=${paymentMethodId}`
  );
  const data = await response.json();

  if (data.success && data.card) {
    console.log('💳 Payment Method:');
    console.log(`   ${data.card.brand.toUpperCase()} ****${data.card.last4}`);
    console.log(`   Expires: ${data.card.exp_month}/${data.card.exp_year}`);
    console.log(`   Type: ${data.card.funding}`);

    if (data.billing_details.name) {
      console.log(`   Name: ${data.billing_details.name}`);
    }
  }
}
```

**Common Use Cases:**
- Display payment method details in billing dashboard
- Show card information before processing a charge
- Verify payment method before subscription renewal
- Display billing information to users

## Testing

A comprehensive test script is available at `scripts/test-stripe-customer.js`:

```bash
node scripts/test-stripe-customer.js
```

This script tests:
- ✅ Basic customer creation
- ✅ Customer creation with metadata
- ✅ Customer creation with full details
- ✅ Error handling with invalid data
- ✅ Customer retrieval
- ✅ Customer updates
- ✅ Customer listing

## Integration with Organizations API

The Stripe service is automatically integrated with the Organizations API. When you create or update an organization, both Metronome and Stripe customers are created automatically.

### Automatic Customer Creation

**POST /api/organizations** - Creates both Metronome and Stripe customers:

```javascript
// Request
POST /api/organizations
{
  "organizationName": "Acme Corp",
  "organizationUsername": "acmecorp",
  "createdBy": "user-uid-123"  // User ID from Firebase Auth
}

// Response
{
  "success": true,
  "organization": {
    "id": "org-id-123",
    "organizationName": "Acme Corp",
    "organizationUsername": "acmecorp",
    "createdBy": "user-uid-123",
    "billingCustomerId": "metronome-customer-id",  // Metronome
    "stripeCustomerId": "cus_ABC123"               // Stripe
  },
  "billing": {
    "metronome": {
      "success": true,
      "customerId": "metronome-customer-id",
      "freeTrial": {
        "success": true,
        "credits": 500
      }
    },
    "stripe": {
      "success": true,
      "customerId": "cus_ABC123",
      "email": "user@example.com"
    }
  }
}
```

**How it works:**
1. Organization is created in Firebase
2. User email is fetched from the `users` collection using `createdBy` field
3. Metronome customer is created with organization name
4. Stripe customer is created with organization name and user email
5. Both customer IDs are stored in the organization document

**PATCH /api/organizations** - Adds billing to existing organization:

```javascript
// Request
PATCH /api/organizations
{
  "organizationId": "existing-org-id"
}

// Response includes both Metronome and Stripe customer IDs
```

### Firebase Schema Update

The organization document now includes both billing customer IDs:

```javascript
{
  organizationName: "Acme Corp",
  organizationUsername: "acmecorp",
  createdBy: "user-uid-123",
  createdAt: Timestamp,
  updatedAt: Timestamp,
  billingCustomerId: "metronome-customer-id",  // Metronome
  stripeCustomerId: "cus_ABC123"               // Stripe (NEW)
}
```

## Using in Other Endpoints

You can easily integrate the Stripe service into any endpoint:

```javascript
const stripeService = require('../services/stripe-service');

module.exports = async (req, res) => {
  // Your endpoint logic
  const { name, email, organizationId } = req.body;

  // Create Stripe customer
  const stripeResult = await stripeService.createCustomer({
    name,
    email,
    metadata: {
      organization_id: organizationId
    }
  });

  if (!stripeResult.success) {
    return res.status(500).json({
      error: 'Failed to create Stripe customer',
      details: stripeResult.error
    });
  }

  // Store stripe_id in your database
  const stripeCustomerId = stripeResult.stripe_id;

  // Continue with your logic...
};
```

## Best Practices

1. **Always validate input** before calling Stripe service functions
2. **Store Stripe customer IDs** in your database for future reference
3. **Use metadata** to link Stripe customers to your internal records
4. **Handle errors gracefully** and provide meaningful error messages
5. **Log all Stripe operations** for debugging and audit purposes
6. **Use test mode** during development (keys starting with `sk_test_`)
7. **Never expose secret keys** to the frontend

## Security Notes

- The Stripe secret key should only be used on the server-side
- Never send the secret key to the frontend
- Use Stripe's publishable key (`pk_test_*` or `pk_live_*`) for frontend operations
- Validate all input before sending to Stripe
- Use HTTPS in production

## Next Steps

You can extend the Stripe service to include:
- Payment method management
- Subscription creation and management
- Invoice generation
- Payment intent creation
- Webhook handling
- Refund processing
- Customer portal sessions

## Resources

- [Stripe API Documentation](https://stripe.com/docs/api)
- [Stripe Customer Object](https://stripe.com/docs/api/customers)
- [Stripe Testing](https://stripe.com/docs/testing)
- [Stripe Webhooks](https://stripe.com/docs/webhooks)

