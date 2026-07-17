# Field Force App: Work Session Backend Requirements

This document outlines the required backend changes to support the Work Session feature in the Boblte Field app.

## 1. Schema & API Signature Updates

**Goal**: Simplify the `clock-in` process by removing redundant ID requirements from the `WorkSession` API and database schema.

**Changes Required**:
- **Database Schema**: Remove the `employeeId` and `positionId` fields from the `FieldWorkSession` model (and underlying database tables).
- **DTOs**: Remove `employeeId` and `positionId` from `ClockInDto` (used in `POST /work-sessions/clock-in`). The only executive identifier required should be `fieldExecutiveId`.

---

## 2. New Endpoint: Field Executive Profile

**Goal**: Provide a single endpoint for the mobile app to fetch the logged-in user's profile details and their associated `fieldExecutiveId`.

**Changes Required**:
- **Endpoint**: Implement `GET /api/v1/field-executives/me`.
- **Response Structure**: The response should aggregate the standard `User` data (email, name, avatar, permissions, etc.) with the user's active `FieldExecutive` data.
- **Critical Fields**: The response MUST include `fieldExecutiveId` at the top level or within a nested `executiveProfile` object.

### Example Expected Response structure:
```json
{
  "success": true,
  "data": {
    "id": "usr_123",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "avatarUrl": "https://...",
    "fieldExecutiveId": "exec_abc123",  // <-- CRITICAL FOR APP
    "permissions": ["FIELD_WORK_SESSIONS_CREATE", "FIELD_EXECUTIVES_READ"]
  }
}
```

## Note for Mobile Developers
Until this endpoint is fully implemented and deployed, the mobile app will mock the `fieldExecutiveId` during development to allow for UI/UX testing of the Clock In/Out flow.
