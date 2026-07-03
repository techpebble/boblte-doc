# Requirement: Employee Directory Search, Filtering, and Stats API Enhancements

**Status**: Proposed / Pending Review
**Target Module**: `boblte-server/src/modules/hr/employees`
**Urgency**: Medium-High (Required for production-scale Employee Directory performance and UX)

## 1. Overview
The current frontend `boblte-webapp` Employee Directory performs all searching and filtering client-side. The standard `GET /api/v1/employees` endpoint only supports `limit` and `cursor` query parameters for basic pagination.

While this approach works for small mock datasets, it does not scale. In a production environment with hundreds or thousands of workers:
1. **Filtering & Searching is Incomplete:** Users can only search/filter within the current pagination chunk (e.g., first 50 employees). If a matching record is on page 5, the client-side search will fail to show it unless the user repeatedly loads more pages.
2. **Heavy Client-side Overhead:** To compute summary stats (Active, Probation, Contingent count), the frontend currently must count records in the loaded list. This leads to inaccurate metrics unless the entire database is loaded.

Therefore, we request backend enhancements to support server-side search, filtering, and stats aggregation.

---

## 2. Proposed Changes

### 2.1 Enhancements to `GET /api/v1/employees`
We request adding optional query parameters to the existing list endpoint.

* **Path**: `GET /api/v1/employees`
* **Query Parameters**:
  * `search` (string): Case-insensitive fuzzy search. Should match:
    * Legal names (`legalFirstName`, `legalMiddleName`, `legalLastName`)
    * Preferred name (`preferredName`)
    * Display name (`displayName`)
    * Employee Code (`employeeCode`)
    * Work Email (`workEmail`)
  * `status` (string, enum of `EmploymentStatus` or comma-separated list): Filter by employment status (e.g., `ACTIVE,PROBATION`).
  * `workerType` (string, enum of `WorkerType` or comma-separated list): Filter by worker type (e.g., `EMPLOYEE,CONTRACTOR`).
  * `businessUnitId` (string, UUID): Filter by business unit.
  * `departmentId` (string, UUID): Filter by department.
  * `designationId` (string, UUID): Filter by designation.
  * `limit` (number, optional, default: 20): Standard pagination size.
  * `cursor` (string, optional): Pagination cursor.
* **Pagination under filters:**
  The server-side cursor pagination should apply *after* the filtering query is resolved, returning `nextCursor` for the matching filtered subset.

#### Example Request:
```http
GET /api/v1/employees?search=john&status=ACTIVE&departmentId=d852a466-9ab8-466d-88b1-a67e74852c00&limit=10
```

---

### 2.2 New Endpoint: Get Employee Stats Metrics
To render the top-level stats dashboard card metrics without fetching the entire directory, we require a lightweight metrics endpoint.

* **Path**: `GET /api/v1/employees/stats`
* **HTTP Method**: `GET`
* **Required Permission**: `employees:read`
* **Response Model**: `ApiResponse<EmployeeStatsResponseDto>`

#### Response Body Format:
```json
{
  "success": true,
  "data": {
    "activeCount": 142,
    "probationCount": 12,
    "contingentCount": 24,
    "totalCount": 178
  }
}
```

---

## 3. Database Impact
The corresponding Prisma queries should leverage indexes on the filtered fields:
- `tenantId` (existing)
- `deletedAt` (existing)
- `employmentStatus`
- `workerType`
- `departmentId`
- `designationId`
- `businessUnitId`

For fuzzy text searches, standard database operators (such as Prisma `contains` or database-specific full-text indexes) are recommended on columns `displayName`, `employeeCode`, and `workEmail`.
