# API Specification

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Owner**: Engineering Team
**Base URL**: `https://api.boblte.com/v1`
**Auth**: Bearer JWT (Access Token)

---

## 1. Overview

The Boblte API is a **RESTful JSON API** following standard HTTP conventions. All endpoints require authentication unless marked as public. The API is tenant-scoped — the active tenant is resolved from the JWT claims.

### 1.1 Authentication Flow

```
POST /auth/login
  → { accessToken, refreshToken }
  → accessToken (15 min TTL) → Bearer header for all requests
  → refreshToken (7 day TTL) → POST /auth/refresh to rotate

POST /auth/logout → blacklists current accessToken
```

### 1.2 Request Format

- **Content-Type**: `application/json`
- **Accept**: `application/json`
- **Authorization**: `Bearer <access_token>`

### 1.3 Response Format

```json
// Success (200 / 201)
{
  "data": { ... },
  "meta": { "total": 100, "page": 1, "limit": 20 }  // for lists
}

// Error (4xx / 5xx)
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "details": [
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

### 1.4 Pagination

All list endpoints support:
- `?page=1` (1-indexed)
- `?limit=20` (default: 20, max: 100)
- `?sortBy=createdAt&sortOrder=desc`

### 1.5 Soft Delete Behavior

Soft-deleted records are excluded from all list responses by default.
Admin endpoints may include `?includeDeleted=true` to fetch deleted records.

---

## 2. Platform Admin API

> Accessible only to PlatformUsers. Separate auth context.

### 2.1 Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/platform/auth/login` | Platform admin login |
| POST | `/platform/auth/refresh` | Refresh platform token |
| POST | `/platform/auth/logout` | Logout |

### 2.2 Tenant Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/platform/tenants` | List all tenants |
| POST | `/platform/tenants` | Create new tenant |
| GET | `/platform/tenants/:id` | Get tenant detail |
| PATCH | `/platform/tenants/:id` | Update tenant |
| PATCH | `/platform/tenants/:id/deactivate` | Deactivate tenant |

### 2.3 Subscription Plans

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/platform/subscription-plans` | List plans |
| POST | `/platform/subscription-plans` | Create plan |
| PATCH | `/platform/subscription-plans/:id` | Update plan |

---

## 3. Tenant Auth API

### 3.1 Authentication

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/auth/login` | Login with email + password | Public |
| POST | `/auth/refresh` | Refresh access token | Public (refresh token) |
| POST | `/auth/logout` | Logout, blacklist token | ✅ Required |
| POST | `/auth/mfa/setup` | Generate TOTP QR code | ✅ Required |
| POST | `/auth/mfa/verify` | Verify TOTP + enable MFA | ✅ Required |
| POST | `/auth/mfa/disable` | Disable MFA | ✅ Required |
| POST | `/auth/mfa/validate` | Validate TOTP on login | Public |

**Login Request**:
```json
POST /auth/login
{
  "email": "user@company.com",
  "password": "SecurePass123!",
  "businessUnitId": "uuid"  // optional — defaults to user's default context
}
```

**Login Response**:
```json
{
  "data": {
    "accessToken": "eyJhbGc...",
    "refreshToken": "eyJhbGc...",
    "user": {
      "id": "uuid",
      "email": "user@company.com",
      "firstName": "John",
      "lastName": "Doe",
      "userType": "EMPLOYEE",
      "activeBusinessUnitId": "uuid",
      "mfaRequired": false
    }
  }
}
```

---

## 4. IAM API

### 4.1 Users

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/users` | `iam:view_users` | List tenant users |
| POST | `/users` | `iam:manage_users` | Create user |
| GET | `/users/me` | *(self)* | Get own profile |
| GET | `/users/:id` | `iam:view_users` | Get user detail |
| PATCH | `/users/:id` | `iam:manage_users` | Update user |
| PATCH | `/users/:id/deactivate` | `iam:manage_users` | Deactivate user |
| GET | `/users/:id/roles` | `iam:view_users` | Get user's roles |
| POST | `/users/:id/roles` | `iam:manage_users` | Assign role to user in BU |
| DELETE | `/users/:id/roles` | `iam:manage_users` | Remove role from user |

### 4.2 Invitations

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| POST | `/invitations` | `iam:manage_users` | Send invitation |
| GET | `/invitations` | `iam:view_users` | List invitations |
| DELETE | `/invitations/:id` | `iam:manage_users` | Revoke invitation |
| POST | `/invitations/accept` | Public | Accept invitation (tokenized) |

### 4.3 Roles

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/roles` | `iam:view_roles` | List roles |
| POST | `/roles` | `iam:manage_roles` | Create role |
| GET | `/roles/:id` | `iam:view_roles` | Get role detail |
| PATCH | `/roles/:id` | `iam:manage_roles` | Update role |
| DELETE | `/roles/:id` | `iam:manage_roles` | Delete role (soft) |
| GET | `/roles/:id/permissions` | `iam:view_roles` | Get role's permissions |
| POST | `/roles/:id/permissions` | `iam:manage_permissions` | Assign permission to role |
| DELETE | `/roles/:id/permissions/:permId` | `iam:manage_permissions` | Remove permission |

### 4.4 Permissions

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/permissions` | `iam:view_roles` | List all platform permissions |

---

## 5. Organization API

### 5.1 Business Units

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/business-units` | `org:view_business_units` | List all BUs |
| POST | `/business-units` | `org:manage_business_units` | Create BU |
| GET | `/business-units/:id` | `org:view_business_units` | Get BU detail |
| PATCH | `/business-units/:id` | `org:manage_business_units` | Update BU |
| DELETE | `/business-units/:id` | `org:manage_business_units` | Soft delete |
| GET | `/business-units/:id/users` | `org:view_business_units` | Users in BU |
| GET | `/business-units/tree` | `org:view_business_units` | Full hierarchy tree |

### 5.2 Departments

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/departments` | `org:view_departments` | List departments |
| POST | `/departments` | `org:manage_departments` | Create department |
| GET | `/departments/:id` | `org:view_departments` | Get detail |
| PATCH | `/departments/:id` | `org:manage_departments` | Update |
| DELETE | `/departments/:id` | `org:manage_departments` | Soft delete |
| GET | `/departments/tree` | `org:view_departments` | Full hierarchy tree |
| PATCH | `/departments/:id/head` | `org:manage_departments` | Set department head |

### 5.3 Designations

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/designations` | `org:view_departments` | List designations |
| POST | `/designations` | `org:manage_designations` | Create designation |
| PATCH | `/designations/:id` | `org:manage_designations` | Update |
| DELETE | `/designations/:id` | `org:manage_designations` | Soft delete |

---

## 6. Workflow API

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/workflows` | `workflow:view_workflows` | List workflows |
| POST | `/workflows` | `workflow:manage_workflows` | Create workflow |
| GET | `/workflows/:id` | `workflow:view_workflows` | Get detail |
| PATCH | `/workflows/:id` | `workflow:manage_workflows` | Update |
| DELETE | `/workflows/:id` | `workflow:manage_workflows` | Soft delete |
| PATCH | `/workflows/:id/publish` | `workflow:manage_workflows` | Publish |
| PATCH | `/workflows/:id/archive` | `workflow:manage_workflows` | Archive |
| GET | `/workflows/:id/steps` | `workflow:view_workflows` | List steps |
| POST | `/workflows/:id/steps` | `workflow:manage_workflows` | Add step |
| PATCH | `/workflows/:id/steps/:stepId` | `workflow:manage_workflows` | Update step |
| DELETE | `/workflows/:id/steps/:stepId` | `workflow:manage_workflows` | Delete step |
| POST | `/workflows/:id/trigger` | `workflow:execute_steps` | Trigger instance |
| GET | `/workflow-instances` | `workflow:view_workflows` | List instances |
| GET | `/workflow-instances/:id` | `workflow:view_workflows` | Get instance |
| POST | `/workflow-instances/:id/cancel` | `workflow:manage_workflows` | Cancel |
| POST | `/workflow-instances/:id/steps/:execId/action` | `workflow:execute_steps` | Act on step |

---

## 7. Tasks API

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/tasks` | `tasks:view_all` | List all tasks |
| GET | `/tasks/my` | `tasks:view_own` | My tasks |
| POST | `/tasks` | `tasks:create` | Create task |
| GET | `/tasks/:id` | `tasks:view_own` | Get task |
| PATCH | `/tasks/:id` | `tasks:edit` | Edit task |
| DELETE | `/tasks/:id` | `tasks:delete` | Soft delete |
| PATCH | `/tasks/:id/status` | `tasks:change_status` | Change status |
| PATCH | `/tasks/:id/assign` | `tasks:assign` | Re-assign |
| GET | `/tasks/:id/comments` | `tasks:view_own` | List comments |
| POST | `/tasks/:id/comments` | `tasks:comment` | Add comment |
| PATCH | `/tasks/:id/comments/:cid` | `tasks:comment` | Edit comment |
| DELETE | `/tasks/:id/comments/:cid` | `tasks:comment` | Delete comment |

---

## 8. HR API (HR Module — feature-gated)

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/employees` | `hr:view_employees` | List employees |
| POST | `/employees` | `hr:manage_employees` | Create employee |
| GET | `/employees/:id` | `hr:view_employees` | Get detail |
| PATCH | `/employees/:id` | `hr:manage_employees` | Update |
| DELETE | `/employees/:id` | `hr:manage_employees` | Soft delete |
| GET | `/employees/:id/subordinates` | `hr:view_employees` | Direct reports |
| GET | `/employees/org-chart` | `hr:view_employees` | Full org tree |

---

## 9. System API

| Method | Endpoint | Permission | Description |
|--------|----------|-----------|-------------|
| GET | `/audit-logs` | `system:view_audit_logs` | List audit logs |
| GET | `/notifications` | *(self)* | My notifications |
| PATCH | `/notifications/:id/read` | *(self)* | Mark as read |
| PATCH | `/notifications/read-all` | *(self)* | Mark all read |
| GET | `/settings` | `system:manage_settings` | Get tenant settings |
| PUT | `/settings/:key` | `system:manage_settings` | Set a setting |

---

## 10. Context Switching API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/context/business-units` | List BUs the user has access to |
| POST | `/context/switch` | Switch active business unit context |

---

## 11. Error Codes Reference

| HTTP Status | Code | Meaning |
|-------------|------|---------|
| 400 | VALIDATION_ERROR | Request body failed validation |
| 401 | UNAUTHORIZED | Missing or invalid JWT |
| 401 | MFA_REQUIRED | MFA step needed post-login |
| 403 | FORBIDDEN | Insufficient permission |
| 403 | MODULE_NOT_ACTIVATED | Feature not in subscription plan |
| 404 | NOT_FOUND | Resource doesn't exist or deleted |
| 409 | CONFLICT | Unique constraint violation |
| 422 | UNPROCESSABLE | Business rule violation |
| 429 | RATE_LIMITED | Too many requests |
| 500 | INTERNAL_ERROR | Server-side error |

---

## 12. Rate Limiting

| Endpoint Group | Limit |
|---------------|-------|
| `/auth/login` | 10 req/min per IP |
| `/auth/refresh` | 20 req/min per user |
| All other endpoints | 300 req/min per user |

---

*Document Owner: Engineering Team | Review Cycle: On API change*
