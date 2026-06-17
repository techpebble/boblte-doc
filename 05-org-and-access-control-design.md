# Organization & Access Control Design

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Owner**: Engineering Team

---

## 1. Overview

This document describes the design of the Organization Structure and Access Control system in Boblte. It covers:

- How tenants model their real-world organization (business units, departments, designations)
- How users are identified and typed (OWNER, EMPLOYEE, EXTERNAL, SYSTEM)
- How roles and permissions are structured and enforced (RBAC with BU scoping)
- Cross-module rules for referencing organizational entities

---

## 2. Organization Hierarchy

```
Tenant
  └── BusinessUnit (e.g., "Headquarters", "Mumbai Branch")
        └── BusinessUnit (nested — "South Zone", "Retail Division")
              └── ...

Tenant
  └── Department (e.g., "Finance", "HR", "Sales")
        └── Department (nested — "Accounts Payable", "Recruitment")

Department ◄──────────► BusinessUnit  (many-to-many via DepartmentUnitMap)

Position (anchored to BusinessUnit + Department)
  └── Position (nested — "Regional Manager" → "Area Manager" → "Sales Executive")
        └── ...

User ─────► PositionAssignment ─────► Position ─────► PositionRole ─────► Role ─────► Permission
```

### 2.1 BusinessUnit
- Represents a **geographic, functional, or legal division** of the company.
- Supports unlimited depth hierarchy via `parentBusinessUnitId`.
- One `isDefault = true` business unit exists per tenant (root unit).
- All Roles, Users, and Employees are scoped to a BusinessUnit.

### 2.2 Department
- Represents a **functional team** (HR, Finance, Sales, Operations).
- Supports unlimited depth hierarchy via `parentDepartmentId`.
- Department can span multiple BusinessUnits via `DepartmentUnitMap`.
- Department head is a **User** (Core entity) — not Employee (module rule, see §6).
- One `isDefault = true` department per tenant.

### 2.3 Designation
- A **job title** within the tenant (e.g., "Senior Engineer", "Branch Manager").
- Purely a label — no hierarchy or permission implications.
- Assigned to employees (HR module); used in reporting and org charts.

### 2.4 DepartmentUnitMap
- Junction table linking `Department` ↔ `BusinessUnit`.
- A Sales department can exist across Mumbai, Delhi, and Bengaluru business units.

### 2.5 Position
- A **named role-slot** in the organizational hierarchy (e.g., "Head of Finance - Mumbai", "Regional Sales Manager").
- Bridges Org Structure and RBAC: **Roles are assigned to Positions, not directly to Users**.
- A Position belongs to one `BusinessUnit` and/or one `Department`.
- Supports unlimited hierarchy depth via `parentPositionId`.
- `isVacant = true` means the position exists but currently has no active holder.
- `positionCode` is unique per tenant (e.g., `FIN-HEAD-MUM`).

**PositionType Values**:

| Type | Description |
|------|-------------|
| `BUSINESS_UNIT_HEAD` | Head of a business unit (e.g., Branch Head) |
| `DEPARTMENT_HEAD` | Head of a department (e.g., Finance Head) |
| `MANAGER` | Generic manager role |
| `SUPERVISOR` | Supervisory role above staff |
| `TEAM_LEAD` | Technical/functional team lead |
| `EXECUTIVE` | Senior individual contributor |
| `STAFF` | General staff member |
| `CUSTOM` | Tenant-defined custom type |

### 2.6 PositionAssignment
- Links a `User` to a `Position` for a defined time period (`effectiveFrom` / `effectiveTo`).
- `isPrimary = true` marks the user's main position (for context switching).
- A user can hold multiple positions simultaneously (e.g., interim coverage).
- `effectiveTo = null` means currently active with no end date.
- When `effectiveTo` is set to a past date, the assignment is considered expired.

---

## 3. User Types

Every person in the system has a `User` record with a `userType`:

| UserType | Description | Can Log In | Has Employee Record |
|----------|-------------|------------|---------------------|
| `OWNER` | Business owner / director / trustee | ✅ | Optional (via `OwnerPrincipal`) |
| `EMPLOYEE` | Regular staff member | ✅ | Yes (via HR module `Employee`) |
| `EXTERNAL` | Auditor, consultant, vendor rep | ✅ (limited) | No (via `ExternalPrincipal`) |
| `SYSTEM` | Service accounts, automated agents | ❌ (API key only) | No |

### 3.1 OwnerPrincipal
Extends `User` with ownership context:
- `ownershipType`: SOLE_PROPRIETOR, PARTNER, DIRECTOR, TRUSTEE
- `ownershipShare`: Decimal (0–100.00) — percentage share in business
- `effectiveFrom` / `effectiveTo`: Date range of ownership

### 3.2 ExternalPrincipal
Extends `User` with external access context:
- `principalType`: AUDITOR, CONSULTANT, VENDOR
- `accessStartDate` / `accessEndDate`: Time-bounded access
- `invitedBy`: User who issued the invitation
- `organization`: The external org they represent

### 3.3 Invitation Flow

```
TenantAdmin sends invite → Invitation record created (token + expiry)
  → Email sent to invitee with link
  → Invitee clicks link → validates token + expiry
  → Invitee completes profile → User record created
  → Invitation marked acceptedAt
```

---

## 4. Role-Based Access Control (RBAC)

### 4.1 Core Concepts — Position-Based RBAC

With the Position module, the RBAC chain is:

```
User ──► PositionAssignment ──► Position ──► PositionRole ──► Role ──► PermissionRole ──► Permission
```

- A **Permission** is a platform-defined capability: `module_name:action_name`
- A **Role** is a tenant-defined collection of permissions (e.g., "Finance Manager")
- A **PositionRole** assigns one or more Roles to a Position (e.g., Finance Head position → Finance Manager role + Audit Viewer role)
- A **Position** is a slot in the org structure (e.g., "Head of Finance — Mumbai")
- A **PositionAssignment** assigns a User to a Position for a time period

All permissions flow exclusively through the Position chain. There is no direct user→role assignment.

### 4.2 Business Unit Scoping via Position

A Position is anchored to a `BusinessUnit` and/or `Department`, so permissions are implicitly scoped:

- "Head of Finance — Mumbai Branch" position → Finance Manager role → `Finance Manager` permissions, scoped to Mumbai Branch
- Assigning a user to that position grants those permissions in the Mumbai context automatically
- `UserBusinessUnitScope` still tracks which contexts a user can switch between (driven by their active positions)
- `isDefaultContext` marks the primary business unit for login (derived from `PositionAssignment.isPrimary`)

### 4.3 Dual RBAC Path

| Path | When Used | Models |
|------|-----------|--------|
| **Position-based** (only path) | All users — OWNER, EMPLOYEE, EXTERNAL | `PositionAssignment → Position → PositionRole → Role` |

### 4.4 Permission Resolution

```
Request arrives with JWT (userId, tenantId, businessUnitId)
  ↓
Load active PositionAssignments
  WHERE userId = ? AND (effectiveTo IS NULL OR effectiveTo > now())
  → For each Position → Load PositionRoles → collect Role IDs
  → For each Role → Load PermissionRoles → collect Permission actionNames
  ↓
Check if required permission is in the set → Allow / Deny
```

### 4.5 System Roles (Seeded)

| Role | isSystemRole | Permissions |
|------|-------------|-------------|
| `TENANT_OWNER` | true | All permissions |
| `TENANT_ADMIN` | true | All non-billing permissions |
| `VIEWER` | true | All `view_*` permissions |

Tenant admins can create **custom roles** by combining any set of permissions.

### 4.5 Permission Naming Convention

```
{module}:{action}_{resource}

Examples:
  iam:manage_users
  iam:view_users
  org:manage_departments
  tasks:create
  tasks:view_all
  tasks:view_own
  workflow:execute_steps
  system:view_audit_logs
```

---

## 5. Access Control in API Layer (NestJS)

```typescript
// Applied via guard + decorator pattern
@Get('departments')
@RequirePermission('org:view_departments')
findAll() { ... }

// Guard resolution:
// 1. Extract userId, tenantId, businessUnitId from JWT
// 2. Load user's permissions for that BU context
// 3. Check if 'org:view_departments' is in set
// 4. Allow or throw ForbiddenException
```

### Module Feature Guard (Optional Module Gating)

```typescript
@Get('employees')
@RequireModule('HR')          // check SubscriptionPlan.features.hrModule
@RequirePermission('hr:view_employees')
findAll() { ... }
```

---

## 6. Cross-Module Reference Rules

> **Golden Rule**: A Core entity's FK must only point to other Core entities. A Module entity can point to Core entities. No Core entity may point to a Module entity.

| FK Source | FK Target | Layer | Allowed? |
|-----------|-----------|-------|----------|
| `Department.departmentHeadId` | `User` | Core → Core | ✅ |
| `Department.departmentHeadId` | `Employee` | Core → HR Module | ❌ |
| `Task.assigneeUserId` | `User` | Core → Core | ✅ |
| `Task.assigneeDepartmentId` | `Department` | Core → Core | ✅ |
| `Employee.userId` | `User` | HR → Core | ✅ |
| `Employee.managerId` | `Employee` | HR → HR | ✅ |
| `FieldAttendance.employeeId` | `Employee` | Field → HR | ✅ (Field depends on HR) |

### Resolving the Department Head Dilemma

```
Business scenario: "Show me the department head's employment details"

Step 1: Load Department.departmentHeadId → User (Core query, always works)
Step 2: If HR module is active → Load User.employee for employment context
Step 3: Application layer merges: { user, employee? }

This way, if HR module is not purchased:
- Step 1 still works ✅
- Step 2 is skipped ✅
- Display falls back to user's name from User model ✅
```

---

## 7. Tenant Isolation Strategy

All tenant-scoped tables carry `tenantId` and enforce:

1. **Application Layer**: Every query includes `WHERE tenant_id = ?` from JWT claim.
2. **Unique Constraints**: `@@unique([tenantId, email])`, `@@unique([tenantId, roleName])`, etc.
3. **Cascade Deletes**: `onDelete: Cascade` ensures deactivating a tenant orphans nothing.
4. **Index on tenantId**: Every tenant-scoped table has `@@index([tenantId])` for query performance.

> Future: PostgreSQL Row-Level Security (RLS) via Supabase for an additional DB-layer enforcement.

---

## 8. Org Structure Diagram (Example Tenant)

```
Tenant: Acme Manufacturing Pvt Ltd
│
├── BusinessUnit: Headquarters (default)
│     ├── BusinessUnit: Mumbai Plant
│     └── BusinessUnit: Delhi Distribution
│
├── Department: Human Resources (head: Priya Sharma — User)
│     └── Department: Recruitment
│
├── Department: Finance (head: Raj Mehta — User)
│     ├── Department: Accounts Payable
│     └── Department: Accounts Receivable
│
└── Department: Operations (head: Vikram Singh — User)
      └── Department: Dispatch

DepartmentUnitMap:
  Finance ↔ Headquarters
  Finance ↔ Mumbai Plant (shared finance team)
  Operations ↔ Mumbai Plant
  Operations ↔ Delhi Distribution
```

---

*Document Owner: Engineering Team | Review Cycle: On architecture change*
