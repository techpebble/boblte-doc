# Core Module — Use Cases

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Draft
**Module Code**: `CORE`
**Owner**: Product Team

---

## 1. Actors

| Actor | Description | Auth Context |
|-------|-------------|--------------|
| **PlatformAdmin** | Boblte internal staff managing the SaaS platform | `platform_users` session |
| **TenantOwner** | Business owner / director of a registered tenant | `users` session (userType = OWNER) |
| **TenantAdmin** | IT / HR admin managing tenant operations | `users` session (system role = TENANT_ADMIN) |
| **Employee** | Regular staff member | `users` session (userType = EMPLOYEE) |
| **ExternalPrincipal** | Auditor, consultant, or vendor representative | `users` session (userType = EXTERNAL) |
| **System** | Automated service account / scheduled job | API key / internal service call |

---

## 2. Platform Management

### UC-PM-01: Create Tenant

| Field | Details |
|-------|---------|
| **Actor** | PlatformAdmin |
| **Goal** | Provision a new tenant on the platform |
| **Preconditions** | PlatformAdmin is authenticated via `/platform/auth/login` |
| **Trigger** | PlatformAdmin initiates tenant creation |
| **Main Flow** | 1. PlatformAdmin provides `companyName`, `domainPrefix`, `tenantType`, and `subscriptionPlanId` <br> 2. System validates `domainPrefix` is unique across `tenants` <br> 3. System creates `tenants` record with `is_active = true` <br> 4. System seeds system roles (TENANT_OWNER, TENANT_ADMIN, VIEWER) in `roles` for the new tenant <br> 5. System creates a default `business_units` record (`is_default = true`) <br> 6. System creates a default `departments` record (`is_default = true`) <br> 7. System returns tenant details |
| **Alternate Flows** | 2a. `domainPrefix` already taken → return `409 CONFLICT` |
| **Postconditions** | Tenant exists with seeded roles, default BU, and default department |
| **Tables Involved** | `tenants`, `subscription_plans`, `roles`, `permissions`, `permission_roles`, `business_units`, `departments` |

### UC-PM-02: Deactivate Tenant

| Field | Details |
|-------|---------|
| **Actor** | PlatformAdmin |
| **Goal** | Suspend a tenant without deleting data |
| **Preconditions** | Tenant exists and `is_active = true` |
| **Main Flow** | 1. PlatformAdmin selects tenant to deactivate <br> 2. System sets `is_active = false` on the `tenants` record <br> 3. All active `refresh_tokens` for users under this tenant are revoked (`revoked_at = now()`) <br> 4. System logs action to `audit_logs` |
| **Postconditions** | Tenant users can no longer authenticate; data is preserved |
| **Tables Involved** | `tenants`, `refresh_tokens`, `audit_logs` |

### UC-PM-03: Manage Subscription Plans

| Field | Details |
|-------|---------|
| **Actor** | PlatformAdmin |
| **Goal** | Create or update subscription plans that gate module access |
| **Preconditions** | PlatformAdmin is authenticated |
| **Main Flow** | 1. PlatformAdmin creates/updates a `subscription_plans` record with `plan_name`, `max_users_allowed`, `max_units_allowed`, and `features` JSON <br> 2. System validates `plan_name` uniqueness <br> 3. System persists the plan |
| **Business Rules** | - `max_users_allowed = NULL` means unlimited <br> - `features` JSON structure controls module gating (e.g., `{ "hrModule": true }`) |
| **Tables Involved** | `subscription_plans` |

---

## 3. Authentication & Session Management

### UC-AUTH-01: Tenant User Login

| Field | Details |
|-------|---------|
| **Actor** | TenantOwner, TenantAdmin, Employee, ExternalPrincipal |
| **Goal** | Authenticate and obtain a session |
| **Preconditions** | User exists in `users` with `is_active = true`, tenant `is_active = true` |
| **Main Flow** | 1. User submits `email`, `password`, and optional `businessUnitId` <br> 2. System looks up user by `(tenant_id, email)` in `users` <br> 3. System verifies `password_hash` (bcrypt) <br> 4. System checks `locked_until` — if locked, reject with `401` <br> 5. System resets `failed_login_attempts` to 0 <br> 6. If `is_mfa_enabled = true`, return `MFA_REQUIRED` response <br> 7. System resolves `businessUnitId` — if not provided, look up `user_business_unit_scopes` where `is_default_context = true` <br> 8. System generates JWT access token (15 min) and refresh token (7 days) <br> 9. System stores refresh token hash in `refresh_tokens` <br> 10. System updates `last_login_at` on `users` <br> 11. Return `accessToken`, `refreshToken`, and user profile |
| **Alternate Flows** | 3a. Password mismatch → increment `failed_login_attempts`; if ≥ threshold (from `tenant_settings`), set `locked_until` <br> 2a. User not found → return generic `401 UNAUTHORIZED` |
| **Tables Involved** | `users`, `tenant_settings`, `user_business_unit_scopes`, `refresh_tokens` |

### UC-AUTH-02: MFA Setup & Verification

| Field | Details |
|-------|---------|
| **Actor** | Any authenticated tenant user |
| **Goal** | Enable TOTP-based multi-factor authentication |
| **Main Flow** | 1. User requests MFA setup → system generates TOTP secret, stores in `mfa_secret` <br> 2. System returns QR code URI (for Google Authenticator) <br> 3. User scans QR and submits verification code <br> 4. System validates TOTP code against `mfa_secret` <br> 5. System sets `is_mfa_enabled = true` on `users` record |
| **Alternate Flows** | 4a. Invalid TOTP code → return validation error, MFA not enabled |
| **Tables Involved** | `users` |

### UC-AUTH-03: Token Refresh

| Field | Details |
|-------|---------|
| **Actor** | Any authenticated user |
| **Goal** | Obtain a new access token using a valid refresh token |
| **Main Flow** | 1. Client sends refresh token <br> 2. System looks up `token_hash` in `refresh_tokens` <br> 3. System validates: not expired (`expires_at > now()`), not revoked (`revoked_at IS NULL`) <br> 4. System revokes current refresh token (`revoked_at = now()`) <br> 5. System issues new access token + new refresh token (rotation) <br> 6. System stores new refresh token hash |
| **Alternate Flows** | 3a. Token expired or revoked → return `401 UNAUTHORIZED` |
| **Tables Involved** | `refresh_tokens` |

### UC-AUTH-04: Logout & Token Blacklisting

| Field | Details |
|-------|---------|
| **Actor** | Any authenticated user |
| **Goal** | Invalidate the current session |
| **Main Flow** | 1. User calls `/auth/logout` <br> 2. System extracts `jti` from the access token <br> 3. System inserts `jti` + `expires_at` into `blacklisted_tokens` <br> 4. System revokes the associated `refresh_tokens` record |
| **Postconditions** | Access token rejected on subsequent requests; refresh token cannot be reused |
| **Tables Involved** | `blacklisted_tokens`, `refresh_tokens` |

### UC-AUTH-05: Platform Admin Login

| Field | Details |
|-------|---------|
| **Actor** | PlatformAdmin |
| **Goal** | Authenticate to the platform admin console |
| **Main Flow** | 1. PlatformAdmin submits `email` + `password` <br> 2. System looks up `platform_users` by email <br> 3. System verifies password hash, checks `locked_until` <br> 4. System issues JWT + refresh token (linked via `platform_user_id` in `refresh_tokens`) <br> 5. Updates `last_login_at` |
| **Tables Involved** | `platform_users`, `refresh_tokens` |

---

## 4. User Management

### UC-UM-01: Create Tenant User

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Add a new user to the tenant |
| **Permission** | `iam:manage_users` |
| **Main Flow** | 1. Admin provides `email`, `firstName`, `lastName`, `userType`, `password` <br> 2. System validates email uniqueness within tenant (`UNIQUE (tenant_id, email)`) <br> 3. System hashes password with bcrypt <br> 4. System creates `users` record with `tenant_id` from JWT context <br> 5. System logs action to `audit_logs` <br> 6. Return user details |
| **Alternate Flows** | 2a. Email exists for tenant → `409 CONFLICT` <br> 2b. If `userType = OWNER`, system also prompts for `OwnerPrincipal` data |
| **Postconditions** | User record created; user can log in once assigned to a position with roles |
| **Tables Involved** | `users`, `audit_logs` |

### UC-UM-02: Invite User via Email

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Invite an external or new user to join the tenant |
| **Permission** | `iam:manage_users` |
| **Main Flow** | 1. Admin provides `email`, `userType`, and optional `principalType` / `ownershipType` <br> 2. System creates `invitations` record with unique `token` and `expires_at` <br> 3. System sends email with tokenized invite link <br> 4. Invitee clicks link → system validates token and expiry <br> 5. Invitee completes profile (name, password) <br> 6. System creates `users` record + appropriate identity record (`owner_principals` or `external_principals`) <br> 7. System sets `accepted_at` on the invitation |
| **Alternate Flows** | 4a. Token expired → `422 UNPROCESSABLE` <br> 4b. Token already accepted → `409 CONFLICT` |
| **Tables Involved** | `invitations`, `users`, `owner_principals`, `external_principals` |

### UC-UM-03: Deactivate User (Soft Delete)

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Disable a user's access without deleting data |
| **Permission** | `iam:manage_users` |
| **Main Flow** | 1. Admin selects user to deactivate <br> 2. System sets `deleted_at = now()` and `deleted_by = actorId` on `users` <br> 3. System revokes all active `refresh_tokens` for the user <br> 4. System logs action to `audit_logs` |
| **Business Rules** | - Cannot deactivate the last TENANT_OWNER <br> - Soft-deleted users are excluded from all list queries |
| **Tables Involved** | `users`, `refresh_tokens`, `audit_logs` |

### UC-UM-04: View Own Profile

| Field | Details |
|-------|---------|
| **Actor** | Any authenticated tenant user |
| **Goal** | View personal profile and assigned positions/roles |
| **Main Flow** | 1. User calls `/users/me` <br> 2. System returns user profile from `users` <br> 3. System loads active `position_assignments` → `positions` → `position_roles` → `roles` <br> 4. System returns merged profile with positions, roles, and accessible business units |
| **Tables Involved** | `users`, `position_assignments`, `positions`, `position_roles`, `roles`, `user_business_unit_scopes` |

### UC-UM-05: Switch Business Unit Context

| Field | Details |
|-------|---------|
| **Actor** | Any authenticated tenant user |
| **Goal** | Change the active business unit for the current session |
| **Preconditions** | User has multiple entries in `user_business_unit_scopes` |
| **Main Flow** | 1. User requests context switch to a different `businessUnitId` <br> 2. System validates user has a `user_business_unit_scopes` entry for that BU <br> 3. System issues a new access token with the updated `businessUnitId` claim <br> 4. Permissions are recalculated based on positions in the new BU context |
| **Alternate Flows** | 2a. User has no scope for requested BU → `403 FORBIDDEN` |
| **Tables Involved** | `user_business_unit_scopes`, `position_assignments`, `positions` |

---

## 5. Role & Permission Management (RBAC)

### UC-RBAC-01: Create Custom Role

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Define a new role with a specific set of permissions |
| **Permission** | `iam:manage_roles` |
| **Main Flow** | 1. Admin provides `roleName` and `description` <br> 2. System validates role name uniqueness within tenant (`UNIQUE (tenant_id, role_name)`) <br> 3. System creates `roles` record with `is_system_role = false` <br> 4. System logs to `audit_logs` |
| **Alternate Flows** | 2a. Role name exists → `409 CONFLICT` |
| **Tables Involved** | `roles`, `audit_logs` |

### UC-RBAC-02: Assign Permissions to Role

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Associate platform-defined permissions with a custom role |
| **Permission** | `iam:manage_permissions` |
| **Main Flow** | 1. Admin selects a role and one or more permissions from the `permissions` catalogue <br> 2. System creates `permission_roles` entries (composite PK: `permission_id` + `role_id`) <br> 3. System records `assigned_by` and `assigned_at` |
| **Business Rules** | - System roles cannot have permissions modified <br> - Permissions are platform-defined (`module_name:action_name`) and read-only |
| **Tables Involved** | `permissions`, `permission_roles`, `roles` |

### UC-RBAC-03: Assign Role to Position

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Link one or more roles to an organizational position |
| **Permission** | `org:manage_positions` |
| **Main Flow** | 1. Admin selects a position and one or more roles <br> 2. System creates `position_roles` entries (composite PK: `position_id` + `role_id`) <br> 3. System records `assigned_by` and `assigned_at` <br> 4. All users currently assigned to this position automatically inherit the new role's permissions |
| **Tables Involved** | `position_roles`, `positions`, `roles` |

### UC-RBAC-04: Resolve User Permissions

| Field | Details |
|-------|---------|
| **Actor** | System (on every authenticated API request) |
| **Goal** | Determine if the requesting user has the required permission |
| **Main Flow** | 1. Extract `userId`, `tenantId`, `businessUnitId` from JWT <br> 2. Load active `position_assignments` WHERE `user_id = userId` AND (`effective_to IS NULL` OR `effective_to > now()`) <br> 3. For each assignment → load `position` → filter by `business_unit_id` matching context <br> 4. For each matched position → load `position_roles` → collect `role_id`s <br> 5. For each role → load `permission_roles` → collect permission `action_name`s <br> 6. Check if the required permission is in the aggregated set <br> 7. Allow or return `403 FORBIDDEN` |
| **Tables Involved** | `position_assignments`, `positions`, `position_roles`, `roles`, `permission_roles`, `permissions` |

---

## 6. Organization Structure

### UC-ORG-01: Create Business Unit

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Add a new business unit to the organization hierarchy |
| **Permission** | `org:manage_business_units` |
| **Main Flow** | 1. Admin provides `unitName`, optional `parentBusinessUnitId`, and `customAttributes` <br> 2. System validates name uniqueness within tenant <br> 3. If `parentBusinessUnitId` is provided, validate it exists within the same tenant <br> 4. System creates `business_units` record <br> 5. System logs to `audit_logs` |
| **Business Rules** | - Only one `is_default = true` BU per tenant <br> - Subscription plan's `max_units_allowed` enforced (if not NULL) |
| **Tables Involved** | `business_units`, `subscription_plans`, `audit_logs` |

### UC-ORG-02: Create Department

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Add a new department |
| **Permission** | `org:manage_departments` |
| **Main Flow** | 1. Admin provides `departmentName` and optional `parentDepartmentId` <br> 2. System validates name uniqueness within tenant <br> 3. System creates `departments` record <br> 4. System logs to `audit_logs` |
| **Tables Involved** | `departments`, `audit_logs` |

### UC-ORG-03: Link Department to Business Unit

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Map a department to operate within one or more business units |
| **Permission** | `org:manage_departments` |
| **Main Flow** | 1. Admin selects a department and target business unit(s) <br> 2. System validates both entities belong to the same tenant <br> 3. System creates `department_unit_map` entries (composite PK: `department_id` + `business_unit_id`) |
| **Example** | "Finance department operates in Headquarters and Mumbai Plant" |
| **Tables Involved** | `department_unit_map`, `departments`, `business_units` |


### UC-ORG-05: Create Designation

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Define a job title within the tenant |
| **Permission** | `org:manage_designations` |
| **Main Flow** | 1. Admin provides `designationName` and optional `description` <br> 2. System creates `designations` record scoped to tenant |
| **Tables Involved** | `designations` |

### UC-ORG-06: Create Position

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Define a named role-slot in the org hierarchy |
| **Permission** | `org:manage_positions` |
| **Main Flow** | 1. Admin provides `positionCode`, `positionName`, `positionType`, optional `businessUnitId`, `departmentId`, `parentPositionId` <br> 2. System validates `positionCode` uniqueness within tenant <br> 3. System creates `positions` record with `is_vacant = true` (no holder yet) <br> 4. System logs to `audit_logs` |
| **Example** | Position code: `FIN-HEAD-MUM`, name: "Head of Finance — Mumbai", type: `DEPARTMENT_HEAD` |
| **Tables Involved** | `positions`, `business_units`, `departments`, `audit_logs` |

### UC-ORG-07: Assign User to Position

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Place a user in an organizational position |
| **Permission** | `org:manage_positions` |
| **Main Flow** | 1. Admin selects a position and a user <br> 2. Admin provides `effectiveFrom`, optional `effectiveTo`, and `isPrimary` <br> 3. System creates `position_assignments` record <br> 4. System sets `is_vacant = false` on the position <br> 5. System updates `user_business_unit_scopes` — adds BU from the position if not already present <br> 6. If `isPrimary = true`, set `is_default_context = true` on the corresponding BU scope entry <br> 7. System logs to `audit_logs` |
| **Business Rules** | - A user can hold multiple positions simultaneously <br> - Only one position can be `is_primary = true` per user <br> - User inherits all roles assigned to the position via `position_roles` |
| **Tables Involved** | `position_assignments`, `positions`, `user_business_unit_scopes`, `audit_logs` |

### UC-ORG-08: End Position Assignment

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Remove a user from a position (e.g., transfer, resignation) |
| **Permission** | `org:manage_positions` |
| **Main Flow** | 1. Admin selects an active position assignment <br> 2. System sets `effective_to = now()` on `position_assignments` <br> 3. If no other active assignment remains for this position, set `is_vacant = true` on `positions` <br> 4. System recalculates `user_business_unit_scopes` — remove BU if no other position links to it <br> 5. System logs to `audit_logs` |
| **Tables Involved** | `position_assignments`, `positions`, `user_business_unit_scopes`, `audit_logs` |

### UC-ORG-09: View Organization Hierarchy Tree

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner, Employee (with appropriate permission) |
| **Goal** | Visualize the full org hierarchy (BUs, departments, positions) |
| **Permission** | `org:view_business_units`, `org:view_departments`, `org:view_positions` |
| **Main Flow** | 1. User requests `/business-units/tree`, `/departments/tree`, or `/positions/tree` <br> 2. System recursively loads entities following `parent_*_id` self-referential FKs <br> 3. System returns nested tree structure |
| **Tables Involved** | `business_units`, `departments`, `positions`, `department_unit_map` |

---

## 7. Identity Management

### UC-ID-01: Register Owner Principal

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Record ownership details for a user of type OWNER |
| **Main Flow** | 1. Admin provides `ownershipType` (SOLE_PROPRIETOR, PARTNER, DIRECTOR, TRUSTEE), `ownershipShare`, and `effectiveFrom` <br> 2. System validates `user_id` refers to a user with `user_type = OWNER` <br> 3. System creates `owner_principals` record <br> 4. `user_id` is UNIQUE — one ownership record per user |
| **Business Rules** | - For PARTNERSHIP tenants, sum of all `ownership_share` values should not exceed 100.00 |
| **Tables Involved** | `owner_principals`, `users` |

### UC-ID-02: Register External Principal

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Grant time-bounded access to an external party |
| **Main Flow** | 1. Admin provides `principalType` (AUDITOR, CONSULTANT, VENDOR), `organization`, `accessStartDate`, `accessEndDate` <br> 2. System validates `user_id` refers to a user with `user_type = EXTERNAL` <br> 3. System creates `external_principals` record <br> 4. System records `invited_by` |
| **Business Rules** | - When `access_end_date` passes, system should auto-deactivate the user's login (job or middleware check) |
| **Tables Involved** | `external_principals`, `users` |

---

## 8. Workflow Engine

### UC-WF-01: Define Workflow

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Create a reusable workflow definition |
| **Permission** | `workflow:manage_workflows` |
| **Main Flow** | 1. Admin provides `workflowName` and `description` <br> 2. System validates name uniqueness within tenant <br> 3. System creates `workflows` record with `status = DRAFT` <br> 4. System logs to `audit_logs` |
| **Tables Involved** | `workflows`, `audit_logs` |

### UC-WF-02: Add Steps to Workflow

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Define the sequence of steps in a workflow |
| **Permission** | `workflow:manage_workflows` |
| **Preconditions** | Workflow exists and `status = DRAFT` |
| **Main Flow** | 1. Admin adds steps with `stepName`, `stepOrder`, `stepType` (APPROVAL / REVIEW / AUTOMATED / NOTIFICATION) <br> 2. Admin assigns each step to a `role` (via `assigned_to_role_id`) or specific `user` (via `assigned_to_user_id`) <br> 3. Admin marks step as `is_required` or optional <br> 4. System validates `step_order` uniqueness within workflow <br> 5. System creates `workflow_steps` records |
| **Tables Involved** | `workflow_steps`, `workflows`, `roles`, `users` |

### UC-WF-03: Publish Workflow

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Make a draft workflow available for use |
| **Permission** | `workflow:manage_workflows` |
| **Main Flow** | 1. Admin requests publish on a DRAFT workflow <br> 2. System validates at least one step exists <br> 3. System updates `status` to `PUBLISHED` <br> 4. Workflow is now available for triggering |
| **Alternate Flows** | 2a. No steps defined → `422 UNPROCESSABLE` |
| **Tables Involved** | `workflows`, `workflow_steps` |

### UC-WF-04: Trigger Workflow Instance

| Field | Details |
|-------|---------|
| **Actor** | Any user with `workflow:execute_steps`, or System |
| **Goal** | Start a workflow instance tied to a business entity |
| **Permission** | `workflow:execute_steps` |
| **Main Flow** | 1. Trigger provides `workflowId`, `referenceType` (e.g., "LEAVE_REQUEST"), and `referenceId` <br> 2. System validates workflow is `PUBLISHED` <br> 3. System creates `workflow_instances` record with `status = PENDING`, `started_at = now()` <br> 4. System creates `workflow_step_executions` for each `workflow_steps` entry, all with `status = PENDING` <br> 5. System sets instance `status = IN_PROGRESS` <br> 6. System sends notification to the first step's assignee |
| **Tables Involved** | `workflow_instances`, `workflow_step_executions`, `workflow_steps`, `notifications` |

### UC-WF-05: Act on Workflow Step

| Field | Details |
|-------|---------|
| **Actor** | User assigned to the step (by role or directly) |
| **Goal** | Approve, reject, or skip a workflow step |
| **Permission** | `workflow:execute_steps` |
| **Main Flow** | 1. Assigned user selects an action: APPROVED, REJECTED, or SKIPPED <br> 2. User optionally adds `action_note` <br> 3. System updates `workflow_step_executions`: `status`, `actioned_by`, `actioned_at` <br> 4. If APPROVED and more steps remain → advance to next step, notify next assignee <br> 5. If APPROVED and last step → set instance `status = COMPLETED`, `completed_at = now()` <br> 6. If REJECTED → set instance `status = REJECTED`, `completed_at = now()` |
| **Tables Involved** | `workflow_step_executions`, `workflow_instances`, `notifications` |

### UC-WF-06: Cancel Workflow Instance

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner, or the initiator |
| **Goal** | Cancel an in-progress workflow |
| **Permission** | `workflow:manage_workflows` |
| **Main Flow** | 1. Admin/initiator cancels the instance <br> 2. System sets instance `status = CANCELLED` <br> 3. All PENDING step executions are set to `SKIPPED` |
| **Tables Involved** | `workflow_instances`, `workflow_step_executions` |

### UC-WF-07: Archive Workflow

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Retire a published workflow so it can no longer be triggered |
| **Permission** | `workflow:manage_workflows` |
| **Main Flow** | 1. Admin archives a PUBLISHED workflow <br> 2. System sets `status = ARCHIVED` <br> 3. Existing in-progress instances continue but no new instances can be created |
| **Tables Involved** | `workflows` |

---

## 9. System Infrastructure

### UC-SYS-01: Record Audit Log

| Field | Details |
|-------|---------|
| **Actor** | System (automatic on every state-changing operation) |
| **Goal** | Maintain a complete audit trail |
| **Main Flow** | 1. On any CREATE, UPDATE, or DELETE operation → system captures: <br>&nbsp;&nbsp;- `actor_id` (from JWT) <br>&nbsp;&nbsp;- `action` ("CREATE" / "UPDATE" / "DELETE") <br>&nbsp;&nbsp;- `resource_type` (e.g., "USER", "DEPARTMENT", "ROLE") <br>&nbsp;&nbsp;- `resource_id` <br>&nbsp;&nbsp;- `old_values` (JSONB — before state, for UPDATE/DELETE) <br>&nbsp;&nbsp;- `new_values` (JSONB — after state, for CREATE/UPDATE) <br>&nbsp;&nbsp;- `ip_address`, `user_agent` from request context <br> 2. System inserts `audit_logs` record with `occurred_at = now()` |
| **Business Rules** | - Audit logs are immutable (append-only) <br> - Retained for 7 years <br> - Indexed by `(tenant_id, occurred_at DESC)` for chronological queries |
| **Tables Involved** | `audit_logs` |

### UC-SYS-02: View Audit Logs

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Review historical actions taken within the tenant |
| **Permission** | `system:view_audit_logs` |
| **Main Flow** | 1. Admin queries `/audit-logs` with optional filters: `resourceType`, `action`, date range <br> 2. System returns paginated results from `audit_logs` filtered by `tenant_id` <br> 3. System supports sorting by `occurred_at DESC` |
| **Tables Involved** | `audit_logs` |

### UC-SYS-03: Send In-App Notification

| Field | Details |
|-------|---------|
| **Actor** | System (triggered by business events) |
| **Goal** | Deliver actionable notifications to users |
| **Main Flow** | 1. Business event occurs (e.g., workflow step assigned, task due) <br> 2. System creates `notifications` record with `recipient_id`, `title`, `body`, `type`, and optional `reference_type` + `reference_id` <br> 3. Notification delivered via WebSocket push or polling <br> 4. Notification marked `is_read = false` by default |
| **Tables Involved** | `notifications` |

### UC-SYS-04: Mark Notifications as Read

| Field | Details |
|-------|---------|
| **Actor** | Any authenticated tenant user |
| **Goal** | Manage notification read state |
| **Main Flow** | 1. User marks a single notification as read → set `is_read = true`, `read_at = now()` <br> 2. User marks all as read → batch update all unread `notifications` for the user |
| **Tables Involved** | `notifications` |

### UC-SYS-05: Manage Tenant Settings

| Field | Details |
|-------|---------|
| **Actor** | TenantAdmin, TenantOwner |
| **Goal** | Configure tenant-specific settings (e.g., max login attempts, timezone) |
| **Permission** | `system:manage_settings` |
| **Main Flow** | 1. Admin sets or updates a `setting_key` with a `setting_value` (JSONB) <br> 2. System upserts `tenant_settings` (unique on `tenant_id + setting_key`) <br> 3. System logs change to `audit_logs` |
| **Example Settings** | `max_login_attempts: 5`, `session_timeout_mins: 30`, `timezone: "Asia/Kolkata"` |
| **Tables Involved** | `tenant_settings`, `audit_logs` |

### UC-SYS-06: Cleanup Expired Blacklisted Tokens

| Field | Details |
|-------|---------|
| **Actor** | System (scheduled job) |
| **Goal** | Prevent unbounded growth of the blacklist table |
| **Main Flow** | 1. Scheduled job runs periodically (e.g., daily) <br> 2. Delete all `blacklisted_tokens` records where `expires_at < now()` <br> 3. These tokens have already expired and are no longer valid anyway |
| **Tables Involved** | `blacklisted_tokens` |

---

## 10. Cross-Cutting Concerns

### UC-CC-01: Tenant Data Isolation

| Field | Details |
|-------|---------|
| **Actor** | System |
| **Goal** | Ensure no tenant can access another tenant's data |
| **Enforcement** | 1. Every tenant-scoped query includes `WHERE tenant_id = ?` from JWT <br> 2. Unique constraints include `tenant_id` (e.g., `UNIQUE (tenant_id, email)`) <br> 3. Cascade deletes via `onDelete: Cascade` on tenant FK <br> 4. Future: PostgreSQL Row-Level Security (RLS) |
| **Tables Involved** | All tenant-scoped tables |

### UC-CC-02: Soft Delete Operations

| Field | Details |
|-------|---------|
| **Actor** | System |
| **Goal** | Preserve data integrity while supporting logical deletion |
| **Enforcement** | 1. All delete operations set `deleted_at = now()` and `deleted_by = actorId` <br> 2. All list queries filter `WHERE deleted_at IS NULL` by default <br> 3. Admin queries can include `?includeDeleted=true` to see deleted records <br> 4. Audit log captures the delete action with `old_values` |
| **Tables Involved** | All business entity tables with `deleted_at` / `deleted_by` columns |

### UC-CC-03: Subscription Plan Enforcement

| Field | Details |
|-------|---------|
| **Actor** | System (middleware check) |
| **Goal** | Gate features and resource limits based on the tenant's subscription |
| **Main Flow** | 1. On module-gated API calls, check `subscription_plans.features` JSON for the required module flag <br> 2. On user creation, check `max_users_allowed` (if not NULL) <br> 3. On business unit creation, check `max_units_allowed` (if not NULL) <br> 4. If limit exceeded → return `403 MODULE_NOT_ACTIVATED` or `422 UNPROCESSABLE` |
| **Tables Involved** | `subscription_plans`, `tenants` |

---

## Appendix: Use Case to Table Mapping

| Use Case ID | Primary Tables |
|-------------|---------------|
| UC-PM-01 | `tenants`, `subscription_plans`, `roles`, `business_units`, `departments` |
| UC-PM-02 | `tenants`, `refresh_tokens`, `audit_logs` |
| UC-PM-03 | `subscription_plans` |
| UC-AUTH-01 | `users`, `tenant_settings`, `user_business_unit_scopes`, `refresh_tokens` |
| UC-AUTH-02 | `users` |
| UC-AUTH-03 | `refresh_tokens` |
| UC-AUTH-04 | `blacklisted_tokens`, `refresh_tokens` |
| UC-AUTH-05 | `platform_users`, `refresh_tokens` |
| UC-UM-01 | `users`, `audit_logs` |
| UC-UM-02 | `invitations`, `users`, `owner_principals`, `external_principals` |
| UC-UM-03 | `users`, `refresh_tokens`, `audit_logs` |
| UC-UM-04 | `users`, `position_assignments`, `positions`, `position_roles`, `roles` |
| UC-UM-05 | `user_business_unit_scopes`, `position_assignments` |
| UC-RBAC-01 | `roles`, `audit_logs` |
| UC-RBAC-02 | `permissions`, `permission_roles` |
| UC-RBAC-03 | `position_roles`, `positions`, `roles` |
| UC-RBAC-04 | `position_assignments`, `positions`, `position_roles`, `permission_roles`, `permissions` |
| UC-ORG-01 | `business_units`, `subscription_plans`, `audit_logs` |
| UC-ORG-02 | `departments`, `users`, `audit_logs` |
| UC-ORG-03 | `department_unit_map`, `departments`, `business_units` |
| UC-ORG-05 | `designations` |
| UC-ORG-06 | `positions`, `business_units`, `departments`, `audit_logs` |
| UC-ORG-07 | `position_assignments`, `positions`, `user_business_unit_scopes`, `audit_logs` |
| UC-ORG-08 | `position_assignments`, `positions`, `user_business_unit_scopes`, `audit_logs` |
| UC-ORG-09 | `business_units`, `departments`, `positions`, `department_unit_map` |
| UC-ID-01 | `owner_principals`, `users` |
| UC-ID-02 | `external_principals`, `users` |
| UC-WF-01 | `workflows`, `audit_logs` |
| UC-WF-02 | `workflow_steps`, `workflows`, `roles`, `users` |
| UC-WF-03 | `workflows`, `workflow_steps` |
| UC-WF-04 | `workflow_instances`, `workflow_step_executions`, `notifications` |
| UC-WF-05 | `workflow_step_executions`, `workflow_instances`, `notifications` |
| UC-WF-06 | `workflow_instances`, `workflow_step_executions` |
| UC-WF-07 | `workflows` |
| UC-SYS-01 | `audit_logs` |
| UC-SYS-02 | `audit_logs` |
| UC-SYS-03 | `notifications` |
| UC-SYS-04 | `notifications` |
| UC-SYS-05 | `tenant_settings`, `audit_logs` |
| UC-SYS-06 | `blacklisted_tokens` |
| UC-CC-01 | All tenant-scoped tables |
| UC-CC-02 | All soft-deletable tables |
| UC-CC-03 | `subscription_plans`, `tenants` |

---

*Document Owner: Product Team | Review Cycle: Per sprint*
