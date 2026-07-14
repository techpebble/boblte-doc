# Market Assigned Positions API Requirement

## Overview
Currently, the `MarketsController` provides an endpoint to assign a position to a market (`POST /markets/:id/assign`). However, there is no corresponding endpoint to retrieve the list of positions (or position assignments) that are assigned to a specific market.

To allow the frontend UI to display the assigned positions for a market, a new read endpoint is required.

## Requested API Endpoint

**Endpoint:** `GET /markets/:id/positions`
**Description:** Retrieves a paginated list of active positions (or position assignments) linked to the specified market.

### Request
- **Path Parameters:**
  - `id`: string (UUID of the market)
- **Query Parameters (Pagination):**
  - `limit`: number (default: 20)
  - `cursor`: string (optional)

### Expected Response
```json
{
  "items": [
    {
      "id": "assignment-uuid",
      "marketId": "market-uuid",
      "positionId": "position-uuid",
      "assignmentType": "PRIMARY | SECONDARY | TEMPORARY",
      "effectiveFrom": "2025-01-01T00:00:00Z",
      "effectiveTo": null,
      "position": {
        "id": "position-uuid",
        "positionCode": "POS-001",
        "positionName": "Regional Manager"
      }
    }
  ],
  "nextCursor": "some-cursor-string-or-null"
}
```

## Business Value
This endpoint is necessary for the frontend application to build a view/panel showing the user which positions have been assigned to a given market (under Market Management), ensuring transparency and proper access control administration.
