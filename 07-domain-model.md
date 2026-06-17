# Domain Model Document

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Owner**: Engineering Team

---

## 1. Overview

This document describes the **domain model** — the key entities, their attributes, relationships, and business rules — for the Boblte ERP platform Core layer.

Domain entities are grouped by **bounded context** (DDD terminology). Each context maps to one or more Prisma schema files.

---

## 2. Bounded Contexts

```
┌─────────────────────────────────────────────────────────────┐
│  PLATFORM CONTEXT                                           │
│  PlatformUser · Tenant · SubscriptionPlan                   │
└─────────────────────────────────────────────────────────────┘
                          │ Tenant (root)
┌─────────────────────────▼─────────────────────────────────── ┐
│  IDENTITY & ACCESS CONTEXT                                   │
│  User · Role · Permission · PositionRole · PermissionRole    │
│  UserBusinessUnitScope · RefreshToken · BlacklistedToken     │
│  OwnerPrincipal · ExternalPrincipal · Invitation             │
└─────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼──────────────────────────────────┐
│  ORGANIZATION CONTEXT                                       │
│  BusinessUnit · Department · Designation · DepartmentUnitMap│
│  Position · PositionAssignment                              │
└──────────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┴──────────────────┐
        ▼                                    ▼
┌───────────────────┐              ┌──────────────────────┐
│  WORKFLOW CONTEXT │              │  TASK CONTEXT        │
│  Workflow         │              │  Task                │
│  WorkflowStep     │◄────────────►│  TaskComment         │
│  WorkflowInstance │              └──────────────────────┘
│  WorkflowStep-    │
│  Execution        │
└───────────────────┘
                          │
┌─────────────────────────▼──────────────────────────────────┐
│  SYSTEM CONTEXT                                             │
│  AuditLog · Notification · TenantSetting                    │
└──────────────────────────────────────────────────────────────┘

- - - MODULE BOUNDARY - - - - - - - - - - - - - - - - - - - -

┌──────────────────────────────────────────────────────────────┐
│  HR CONTEXT (Optional Module)                                │
│  Employee                                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Entity Definitions

---

### 3.1 Tenant

**Context**: Platform
**Schema File**: `platform.prisma`

> Represents a single registered company/organization using the Boblte platform.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `companyName` | String | Legal/trading name of the company |
| `domainPrefix` | String (unique) | URL subdomain (`{prefix}.boblte.com`) |
| `tenantType` | Enum | SOLE_PROPRIETORSHIP, PARTNERSHIP, PRIVATE_LIMITED, PUBLIC_LIMITED, LLP, TRUST, NGO |
| `subscriptionPlanId` | UUID? | Active subscription plan |
| `isActive` | Boolean | Soft-deactivation flag |
| `customAttributes` | JSON | Extensible key-value metadata |
| `createdAt/By` | DateTime/String | Audit trail |
| `updatedAt/By` | DateTime/String | Audit trail |

**Key Rules**:
- `domainPrefix` is globally unique across all tenants.
- All other entities carry `tenantId` for isolation.
- Deactivating a tenant (`isActive = false`) blocks all logins but retains data.

**Relationships**:
- Has many: `User`, `BusinessUnit`, `Department`, `Designation`, `Role`, `Workflow`, `Task`, `Employee`
- Belongs to: `SubscriptionPlan` (optional)

---

### 3.2 SubscriptionPlan

**Context**: Platform
**Schema File**: `platform.prisma`

> Defines the feature set and limits of a subscription tier.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `planName` | String (unique) | e.g., "Starter", "Growth", "Enterprise" |
| `maxUsersAllowed` | Int? | Null = unlimited |
| `maxUnitsAllowed` | Int? | Max business units; null = unlimited |
| `features` | JSON | Feature flags: `{ "hrModule": true, "tasksModule": true, ... }` |
| `isActive` | Boolean | Available for assignment |

---

### 3.3 PlatformUser

**Context**: Platform
**Schema File**: `platform.prisma`

> Internal Boblte staff who manage tenants on the platform admin panel.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `email` | String (unique) | Platform-level unique |
| `passwordHash` | String | bcrypt |
| `firstName` / `lastName` | String | Name |
| `isActive` | Boolean | Account status |
| `failedLoginAttempts` | Int | Brute-force counter |
| `lockedUntil` | DateTime? | Account lockout expiry |

---

### 3.4 User

**Context**: Identity & Access
**Schema File**: `auth.prisma`

> A person (or service account) who has access to a specific tenant.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `email` | String | Unique within tenant |
| `passwordHash` | String | bcrypt |
| `firstName` / `lastName` | String | Display name |
| `mobileNumber` | String? | Unique within tenant |
| `userType` | Enum | OWNER, EMPLOYEE, EXTERNAL, SYSTEM |
| `isMfaEnabled` | Boolean | TOTP MFA active |
| `mfaSecret` | String? | Encrypted TOTP seed |
| `isActive` | Boolean | Login allowed |
| `lastLoginAt` | DateTime? | Last successful login |
| `failedLoginAttempts` | Int | Brute-force counter |
| `lockedUntil` | DateTime? | Lockout expiry |
| `customAttributes` | JSON | Extensible metadata |
| `createdAt/By/updatedAt/By/deletedAt/By` | | Full audit trail |

**Key Rules**:
- `email` is unique per tenant (not globally).
- `mobileNumber` is unique per tenant when set.
- Soft delete only — `deletedAt` is set; record is never removed.
- A User can have one of: `OwnerPrincipal`, `ExternalPrincipal`, or neither (regular employee or system).

---

### 3.5 Role

**Context**: Identity & Access
**Schema File**: `auth.prisma`

> A named collection of permissions, scoped to a tenant.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `roleName` | String | Unique within tenant |
| `description` | String? | What this role does |
| `isSystemRole` | Boolean | Seeded by platform; not editable by tenant |

**Key Rules**:
- `roleName` is unique per tenant.
- System roles (`isSystemRole = true`) are seeded at tenant creation and cannot be deleted.
- Roles are assigned to **Positions** via `PositionRole` — never directly to users.

---

### 3.6 Permission

**Context**: Identity & Access
**Schema File**: `auth.prisma`

> A fine-grained capability definition, platform-level.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `moduleName` | String | e.g., `iam`, `org`, `tasks`, `workflow` |
| `actionName` | String | e.g., `manage_users`, `view_departments` |
| `description` | String? | Human description |

**Key Rules**:
- `[moduleName, actionName]` is globally unique.
- Permissions are platform-defined — tenants cannot create custom permissions (Phase 1).

---

### 3.7 Business Unit

**Context**: Organization
**Schema File**: `org.prisma`

> A geographic, functional, or legal division of the tenant's company.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `unitName` | String | Unique within tenant |
| `parentBusinessUnitId` | UUID? | Parent in hierarchy |
| `isDefault` | Boolean | Root/default unit |
| `customAttributes` | JSON | Extensible metadata |

**Key Rules**:
- Supports unlimited hierarchy depth.
- Exactly one unit per tenant has `isDefault = true`.
- Users and roles are always scoped within a business unit.

---

### 3.8 Department

**Context**: Organization
**Schema File**: `org.prisma`

> A functional team within the organization.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `departmentName` | String | Unique within tenant |

| `parentDepartmentId` | UUID? | Parent in hierarchy |
| `isDefault` | Boolean | Default department |

**Key Rules**:
- Supports unlimited hierarchy depth.

- Department can be mapped to multiple business units via `DepartmentUnitMap`.

---

### 3.9 Designation

**Context**: Organization
**Schema File**: `org.prisma`

> A job title or position label within the tenant.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `designationName` | String | e.g., "Senior Engineer", "Branch Manager" |
| `description` | String? | Optional description |

---

### 3.10 Position

**Context**: Organization
**Schema File**: `org.prisma`

> A named structural slot in the organization hierarchy. The **RBAC pivot point** — roles are assigned to positions, and users fill positions.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `positionCode` | String | Unique per tenant (e.g., `FIN-HEAD-MUM`) |
| `positionName` | String | Display name (e.g., "Head of Finance — Mumbai") |
| `businessUnitId` | UUID? | FK → BusinessUnit (where this position is located) |
| `departmentId` | UUID? | FK → Department (functional team) |
| `parentPositionId` | UUID? | FK → Position (self-referential hierarchy) |
| `positionType` | Enum | BUSINESS_UNIT_HEAD, DEPARTMENT_HEAD, MANAGER, SUPERVISOR, TEAM_LEAD, EXECUTIVE, STAFF, CUSTOM |
| `isVacant` | Boolean | true = position exists but no current holder |

**Key Rules**:
- `positionCode` is unique per tenant.
- A Position exists independently of the person filling it — if the holder leaves, `isVacant = true`.
- Supports unlimited depth via `parentPositionId` (e.g., Regional Head → Area Head → Executive).
- Deleting the person's assignment doesn't delete the Position.

**Relationships**:
- Has many: `PositionAssignment` (users holding this position), `PositionRole` (roles granted by this position)
- Belongs to: `Tenant`, `BusinessUnit?`, `Department?`, `Position?` (parent)

---

### 3.11 PositionAssignment

**Context**: Organization
**Schema File**: `org.prisma`

> Links a User to a Position for a defined time period. The mechanism by which a user gains the permissions of a position.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `positionId` | UUID | FK → Position |
| `userId` | UUID | FK → User |
| `isPrimary` | Boolean | true = this is the user's main position (for context) |
| `effectiveFrom` | DateTime | When the assignment starts |
| `effectiveTo` | DateTime? | When it ends (null = indefinite) |
| `createdAt/By` | | Audit trail |

**Key Rules**:
- A user can hold multiple positions (e.g., acting coverage, dual roles).
- Only one position per user should have `isPrimary = true` at any time.
- An expired assignment (`effectiveTo` < now) grants no permissions.
- When a position becomes vacant, all its assignments should be ended (`effectiveTo = now`).

---

### 3.10 Workflow

**Context**: Workflow
**Schema File**: `workflow.prisma`

> A named, reusable business process template.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `workflowName` | String | Unique within tenant |
| `description` | String? | Purpose of workflow |
| `status` | Enum | DRAFT, PUBLISHED, ARCHIVED |

**Key Rules**:
- Only `PUBLISHED` workflows can trigger instances.
- Archiving a workflow retains all existing instances but prevents new ones.

---

### 3.11 WorkflowStep

**Context**: Workflow
**Schema File**: `workflow.prisma`

> A single node in a workflow process definition.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `workflowId` | UUID | Parent workflow |
| `stepName` | String | e.g., "Manager Approval" |
| `stepOrder` | Int | Execution sequence |
| `stepType` | Enum | APPROVAL, REVIEW, AUTOMATED, NOTIFICATION |
| `assignedToRoleId` | UUID? | Role responsible |
| `assignedToUserId` | UUID? | Specific user responsible |
| `isRequired` | Boolean | If false, step can be skipped |

---

### 3.12 WorkflowInstance

**Context**: Workflow
**Schema File**: `workflow.prisma`

> A single execution run of a workflow, triggered for a specific business entity.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `workflowId` | UUID | Which workflow template |
| `referenceType` | String | e.g., `TASK`, `LEAVE_APPLICATION` |
| `referenceId` | String | UUID of the triggering entity |
| `status` | Enum | PENDING, IN_PROGRESS, COMPLETED, REJECTED, CANCELLED |
| `startedAt` | DateTime | When instance was created |
| `completedAt` | DateTime? | When resolved |
| `initiatedBy` | UUID? | User who triggered |

---

### 3.13 Task

**Context**: Task
**Schema File**: `tasks.prisma`

> A unit of work assigned within the organization.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `title` | String | Task title |
| `description` | String? | Markdown description |
| `taskType` | Enum | NORMAL, WORKFLOW |
| `status` | Enum | TO_DO, IN_PROGRESS, DONE, ON_HOLD, CANCELLED |
| `priority` | Enum | LOW, MEDIUM, HIGH, CRITICAL |
| `frequency` | Enum? | DAILY, WEEKLY, MONTHLY, YEARLY |
| `workflowId` | UUID? | Linked workflow (if WORKFLOW type) |
| `assigneeType` | Enum? | USER, DEPARTMENT, BUSINESS_UNIT |
| `assigneeDepartmentId` | UUID? | FK → Department |
| `assigneeBusinessUnitId` | UUID? | FK → BusinessUnit |
| `assigneeUserId` | UUID? | FK → User |
| `dueDate` | DateTime? | Due date |
| `customAttributes` | JSON | Extensible metadata |

**Key Rules**:
- Only one of `assigneeDepartmentId`, `assigneeBusinessUnitId`, `assigneeUserId` may be set at a time.
- DB check constraint enforces exclusivity (manual migration required).
- All assignee FKs reference Core entities only (User, Department, BusinessUnit).

---

### 3.14 Employee (HR Module)

**Context**: HR (Optional Module)
**Schema File**: `hr.prisma`

> Employment-specific record for a person within the tenant. Extends a `User`.

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant scope |
| `userId` | UUID? | FK → User (Core) — nullable for non-system employees |
| `employeeCode` | String? | Company-assigned employee ID |
| `firstName` / `lastName` | String | Name (used when no User record) |
| `email` / `mobileNumber` | String? | Contact (fallback when no User) |
| `designationId` | UUID? | FK → Designation |
| `departmentId` | UUID? | FK → Department |
| `businessUnitId` | UUID? | FK → BusinessUnit |
| `managerId` | UUID? | FK → Employee (self-referential) |
| `dateOfJoining` | Date? | Employment start |
| `employmentStatus` | Enum | ACTIVE, PROBATION, ON_LEAVE, SUSPENDED, RESIGNED, TERMINATED, RETIRED |
| `employmentType` | Enum? | FULL_TIME, PART_TIME, CONTRACT, TEMPORARY, INTERN, TRAINEE |
| `isActive` | Boolean | Employment active flag |
| `customAttributes` | JSON | Extensible metadata |

**Key Rules**:
- An Employee with `userId` set is a **system employee** (has login access).
- An Employee without `userId` is an **offline employee** (e.g., factory worker, no system access).
- `managerId` enables self-referential hierarchy for org chart.

---

## 4. Domain Relationships Diagram

```
PlatformUser ──creates──► Tenant ──has──► SubscriptionPlan
                │
                └──has──► User ──────────────────────────┐
                           │                              │
                      userType                     (optional)
                 OWNER/EMPLOYEE/EXTERNAL/SYSTEM          │
                           │                              │
               ┌───────────┼──────────────┐              │
               ▼           ▼              ▼               ▼
         OwnerPrincipal ExternalPrincipal Employee(HR)
                                              │
                                         managerId
                                         (self-ref)

         User ──► PositionAssignment ──► Position ──► PositionRole ──► Role ──► Permission
                                              │
                            ┌─────────────────┼────────────────┐
                            ▼                 ▼                ▼
                      BusinessUnit       Department      parentPosition
                            │
                    DepartmentUnitMap ───► Department


Task ──► User (assignee, Core)
Task ──► Department (assignee, Core)
Task ──► BusinessUnit (assignee, Core)
Task ──► Workflow

WorkflowInstance ──► Workflow ──► WorkflowStep
WorkflowInstance ──► WorkflowStepExecution ──► WorkflowStep
```

---

*Document Owner: Engineering Team | Review Cycle: On model change*
