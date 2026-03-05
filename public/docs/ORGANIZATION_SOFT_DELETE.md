# Organization Soft Delete API

## Overview

Secure soft delete endpoint for organizations with multi-layer authorization checks. This is a **critical operation** that permanently marks an organization as deleted while preserving data for audit trails.

## Security Features

### 🔒 5-Layer Security Validation

1. **Parameter Validation** - Ensures organizationId and userId are provided and valid
2. **Existence Check** - Verifies organization exists in database
3. **Deletion Status Check** - Prevents double-deletion
4. **Creator Authorization** - Verifies user is the organization creator (CRITICAL)
5. **User Validation** - Confirms user exists in the system

### 🛡️ Security Logging

All delete attempts are logged with:
- Timestamp
- User ID (who attempted)
- Organization ID
- Success/failure status
- IP address (for unauthorized attempts)
- Full audit trail

## API Endpoint

### Delete Organization (Soft Delete)

**Endpoint:** `DELETE /api/organizations`

**Method:** `DELETE`

**Request Body:**
```json
{
  "organizationId": "kq8D0URspd5I7uBck8l9",
  "userId": "user-uid-123"
}
```

**Required Fields:**
- `organizationId` (string) - Firebase organization ID
- `userId` (string) - User ID of the person requesting deletion

**Authorization:**
- Only the user who created the organization (`createdBy` field) can delete it
- No exceptions - even admins cannot delete without being the creator

## Response Examples

### ✅ Success (200)

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

### ❌ Error Responses

#### 400 - Missing organizationId
```json
{
  "error": "Missing or invalid required field: organizationId",
  "message": "organizationId must be a valid string"
}
```

#### 400 - Missing userId
```json
{
  "error": "Missing or invalid required field: userId",
  "message": "userId must be a valid string for authorization"
}
```

#### 403 - Unauthorized (Not Creator)
```json
{
  "error": "Unauthorized",
  "message": "Only the organization creator can delete this organization",
  "organizationId": "kq8D0URspd5I7uBck8l9"
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

#### 410 - Already Deleted
```json
{
  "error": "Organization already deleted",
  "organizationId": "kq8D0URspd5I7uBck8l9",
  "deletedAt": "2025-10-09T15:30:00.000Z",
  "message": "This organization has already been deleted"
}
```

## cURL Examples

### Valid Delete Request

```bash
curl -X DELETE https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "kq8D0URspd5I7uBck8l9",
    "userId": "user-uid-123"
  }'
```

### With Pretty Print

```bash
curl -X DELETE https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "organizationId": "kq8D0URspd5I7uBck8l9",
    "userId": "user-uid-123"
  }' | jq
```

## Soft Delete Behavior

### What Happens When Deleted

1. **Organization is NOT removed** from database
2. **Marked as deleted** with `deleted: true` flag
3. **Deletion metadata added**:
   - `deletedAt` - Timestamp of deletion
   - `deletedBy` - User ID who deleted it
   - `deletedMetadata` - Snapshot of organization data at deletion time
4. **Hidden from listings** - Won't appear in GET /api/organizations
5. **Returns 410 Gone** when accessed directly

### Data Preserved

```javascript
{
  // Original fields remain unchanged
  organizationName: "Sholay",
  organizationUsername: "sholay",
  createdBy: "user-uid-123",
  createdAt: Timestamp,
  billingCustomerId: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
  
  // Soft delete fields added
  deleted: true,
  deletedAt: Timestamp,
  deletedBy: "user-uid-123",
  updatedAt: Timestamp,
  
  // Audit trail
  deletedMetadata: {
    organizationName: "Sholay",
    organizationUsername: "sholay",
    createdBy: "user-uid-123",
    createdAt: "2025-06-09T10:00:00.000Z",
    billingCustomerId: "c6330dab-5b2a-43bc-a6c4-acc576c1fbe5",
    deletedTimestamp: "2025-10-09T15:30:00.000Z"
  }
}
```

## Security Audit Trail

### Successful Deletion Log

```
Organization soft deleted successfully: {
  organizationId: 'kq8D0URspd5I7uBck8l9',
  deletedBy: 'user-uid-123',
  organizationName: 'Sholay',
  timestamp: '2025-10-09T15:30:00.000Z'
}
```

### Unauthorized Attempt Log

```
SECURITY ALERT: Unauthorized delete attempt: {
  organizationId: 'kq8D0URspd5I7uBck8l9',
  attemptedBy: 'user-uid-456',
  actualCreator: 'user-uid-123',
  timestamp: '2025-10-09T15:30:00.000Z',
  ip: '192.168.1.1'
}
```

## Frontend Integration

### React Example

```javascript
async function deleteOrganization(organizationId, userId) {
  // Confirm with user
  const confirmed = window.confirm(
    'Are you sure you want to delete this organization? This action cannot be undone.'
  );
  
  if (!confirmed) return;

  try {
    const response = await fetch('/api/organizations', {
      method: 'DELETE',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        organizationId,
        userId
      })
    });

    const data = await response.json();

    if (response.ok) {
      alert('Organization deleted successfully');
      // Redirect to dashboard
      window.location.href = '/dashboard';
    } else {
      // Handle errors
      if (response.status === 403) {
        alert('You are not authorized to delete this organization');
      } else if (response.status === 410) {
        alert('This organization has already been deleted');
      } else {
        alert(`Error: ${data.message || data.error}`);
      }
    }
  } catch (error) {
    console.error('Delete failed:', error);
    alert('Failed to delete organization. Please try again.');
  }
}
```

### Delete Button Component

```jsx
function DeleteOrganizationButton({ organization, currentUserId }) {
  const [deleting, setDeleting] = useState(false);
  
  // Only show button if user is the creator
  if (organization.createdBy !== currentUserId) {
    return null;
  }

  const handleDelete = async () => {
    const confirmed = window.confirm(
      `Delete "${organization.organizationName}"? This cannot be undone.`
    );
    
    if (!confirmed) return;

    setDeleting(true);

    try {
      const response = await fetch('/api/organizations', {
        method: 'DELETE',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          organizationId: organization.id,
          userId: currentUserId
        })
      });

      if (response.ok) {
        window.location.href = '/dashboard';
      } else {
        const data = await response.json();
        alert(data.message || data.error);
      }
    } catch (error) {
      alert('Failed to delete organization');
    } finally {
      setDeleting(false);
    }
  };

  return (
    <button 
      onClick={handleDelete}
      disabled={deleting}
      className="btn-danger"
    >
      {deleting ? 'Deleting...' : 'Delete Organization'}
    </button>
  );
}
```

## Best Practices

### ✅ Do's

- Always confirm deletion with the user
- Show clear warning messages
- Verify user is the creator before showing delete option
- Handle all error responses appropriately
- Log deletion actions for audit purposes
- Redirect user after successful deletion

### ❌ Don'ts

- Don't allow deletion without confirmation
- Don't expose delete functionality to non-creators
- Don't ignore error responses
- Don't allow deletion without userId verification
- Don't hard delete - always use soft delete

## Recovery

Deleted organizations can be recovered by:

1. **Database Admin** - Manually update `deleted: false` in Firestore
2. **Support Team** - Create recovery endpoint (not implemented)

**Note:** There is currently no self-service recovery option. Once deleted, contact support.

## Related Endpoints

- `POST /api/organizations` - Create organization
- `GET /api/organizations?organizationId=ID` - Get single organization (returns 410 if deleted)
- `GET /api/organizations` - List organizations (excludes deleted)

## Testing

```bash
# Test successful deletion
curl -X DELETE https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "test-org-id", "userId": "creator-user-id"}'

# Test unauthorized deletion
curl -X DELETE https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "test-org-id", "userId": "wrong-user-id"}'

# Test missing parameters
curl -X DELETE https://social-apis-two.vercel.app/api/organizations \
  -H "Content-Type: application/json" \
  -d '{"organizationId": "test-org-id"}'
```

