# API Requirement: GET /api/v1/outlets

## Description
The frontend application requires a new endpoint to fetch a global, paginated list of all outlets within the user's tenant context. Currently, the backend only provides an endpoint to fetch outlets filtered by a specific market (`GET /api/v1/outlets/market/:marketId`), but the main Outlets management view (`FieldOutletsView.tsx`) needs to display all outlets irrespective of their market assignment.

## Endpoint Specification
- **Method:** `GET`
- **Path:** `/api/v1/outlets`

### Request Query Parameters
- `limit` (number, optional) - Number of items to return (default: 20)
- `cursor` (string, optional) - Pagination cursor
- `search` (string, optional) - Search term for filtering outlets by name or code

### Authentication & Authorization
- **Auth Guard:** Requires valid access token.
- **Permissions:** Requires `FIELD_OUTLETS_READ` permission (`'field-outlets:read'`).

### Expected Response Structure
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "uuid",
        "outletCode": "OUT-001",
        "outletName": "Store A",
        "marketId": "uuid",
        "ownerName": "John Doe",
        "isActive": true
      }
    ],
    "nextCursor": "string | null"
  }
}
```

### Affected Frontend Component
- `FieldOutletsView.tsx` currently expects this endpoint to be available in order to render the Outlets data table on page load.
