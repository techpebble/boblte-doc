# Core Module — Product Requirements Document (PRD)

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Module Code**: `CORE`
**Owner**: Product Team

---

## 1. Overview

The Core Module is the foundation of the Boblte ERP platform. It is **always installed** for every tenant and cannot be deactivated. All other modules depend on the Core Module.

Core provides:
- **Multi-tenant platform management** (Tenant, PlatformUser, SubscriptionPlan)
- **Identity & Access Management** (User, Role, Permission, RBAC)
- **Organization Structure** (BusinessUnit, Department, Designation)
- **Workflow Engine** (definition, execution, step management)
- **System Infrastructure** (AuditLog, Notification, TenantSetting)

---

## 2. Scope

### In Scope
- Tenant provisioning and lifecycle management
- User authentication (email/password + MFA)
- Role-based access control (RBAC) with business unit scoping
- Organization hierarchy (business units → departments → designations)
- Workflow definition and execution engine
- System-wide audit logging
- In-app notification delivery
- Tenant-specific settings management
- Invitation flow for new users

### Out of Scope
- Employee-specific HR data (→ HR Module)
- Task assignment UI (→ Task Management Module)
- GPS tracking (→ Field Executive Module)
- Financial transactions (→ Finance Module)

---

## 3. User Personas

| Persona | Description | Primary Actions |
|---------|-------------|-----------------|
| **PlatformAdmin** | Boblte internal staff | Create tenants, manage subscription plans |
| **TenantOwner** | Business owner / director | Configure org structure, manage users, set workflows |
| **TenantAdmin** | IT/HR admin of a tenant | User onboarding, role assignment, settings |
| **Employee** | Regular staff | View own profile, receive notifications |
| **ExternalPrincipal** | Auditor, consultant, vendor rep | Limited read-only access per scope |

---

## 4. Feature Requirements

### 4.1 Tenant Management

| ID | Requirement | Priority |
|----|-------------|----------|
| TM-01 | PlatformAdmin can create a new tenant with companyName, domainPrefix, tenantType, and subscriptionPlan | P0 |
| TM-02 | Each tenant has a unique `domainPrefix` used for URL routing (`{prefix}.boblte.com`) | P0 |
| TM-03 | Tenant types: SOLE_PROPRIETORSHIP, PARTNERSHIP, PRIVATE_LIMITED, PUBLIC_LIMITED, LLP, TRUST, NGO | P0 |
| TM-04 | Tenant can be deactivated (soft: `isActive = false`) without data deletion | P0 |
| TM-05 | Tenant custom attributes stored as JSON for extensibility | P1 |
| TM-06 | Tenant can be associated with a SubscriptionPlan that gates feature access | P0 |

### 4.2 User Management

| ID | Requirement | Priority |
|----|-------------|----------|
| UM-01 | Users are created within a tenant scope (tenantId is always set) | P0 |
| UM-02 | User types: OWNER, EMPLOYEE, EXTERNAL, SYSTEM | P0 |
| UM-03 | Email + password authentication with bcrypt hashing | P0 |
| UM-04 | TOTP-based MFA (Google Authenticator compatible) | P0 |
| UM-05 | Account lockout after N failed login attempts (configurable via TenantSetting) | P0 |
| UM-06 | Refresh token rotation with revocation support | P0 |
| UM-07 | JWT token blacklisting on logout | P0 |
| UM-08 | Soft delete: users are never hard-deleted | P0 |
| UM-09 | User invitation via email with tokenized invite link | P1 |
| UM-10 | User custom attributes as JSON for extensibility | P2 |
| UM-11 | Unique email per tenant; unique mobile number per tenant | P0 |

### 4.3 Role & Permission Management (RBAC)

| ID | Requirement | Priority |
|----|-------------|----------|
| RBAC-01 | Roles are tenant-scoped; each tenant manages its own role catalogue | P0 |
| RBAC-02 | System roles (isSystemRole=true) seeded at tenant creation, not editable | P0 |
| RBAC-03 | Permissions are platform-defined (moduleName + actionName) and assigned to roles | P0 |
| RBAC-04 | Users are assigned roles within a specific BusinessUnit context | P0 |
| RBAC-05 | A user can have multiple roles across multiple business units | P0 |
| RBAC-06 | UserBusinessUnitScope tracks which business units a user can switch between | P0 |
| RBAC-07 | One business unit is flagged as `isDefaultContext` per user | P0 |
| RBAC-08 | Permission check resolves: User → UserRole (in BU context) → Role → PermissionRole → Permission | P0 |

### 4.4 Organization Structure

| ID | Requirement | Priority |
|----|-------------|----------|
| ORG-01 | Business units support unlimited hierarchy depth via `parentBusinessUnitId` | P0 |
| ORG-02 | One default business unit per tenant | P0 |
| ORG-03 | Departments support unlimited hierarchy via `parentDepartmentId` | P0 |
| ORG-04 | Department head is a `User` (not Employee — Core module rule) | P0 |
| ORG-05 | DepartmentUnitMap links departments to business units (many-to-many) | P0 |
| ORG-06 | Designations are tenant-specific job titles | P0 |
| ORG-07 | All org entities support soft delete | P0 |

### 4.5 Workflow Engine

| ID | Requirement | Priority |
|----|-------------|----------|
| WF-01 | Tenant-admins can create and publish workflow definitions | P0 |
| WF-02 | Workflow statuses: DRAFT → PUBLISHED → ARCHIVED | P0 |
| WF-03 | Workflow steps support types: APPROVAL, REVIEW, AUTOMATED, NOTIFICATION | P0 |
| WF-04 | Steps assigned to a Role or a specific User | P0 |
| WF-05 | Step order defines linear execution sequence | P0 |
| WF-06 | WorkflowInstance created when a business entity triggers a workflow | P0 |
| WF-07 | WorkflowStepExecution tracks per-step actions (approve / reject / skip) with notes | P0 |
| WF-08 | Instance statuses: PENDING → IN_PROGRESS → COMPLETED / REJECTED / CANCELLED | P0 |
| WF-09 | Workflow linked to a Task via `workflowId` | P1 |
| WF-10 | Conditional branching via `stepCondition` JSON rule DSL | P2 (roadmap) |
| WF-11 | Parallel step execution flag | P2 (roadmap) |
| WF-12 | SLA-based escalation via `slaDurationMins` | P2 (roadmap) |

### 4.6 System Infrastructure

| ID | Requirement | Priority |
|----|-------------|----------|
| SYS-01 | AuditLog records every state-changing action (actor, resource, old/new values, IP) | P0 |
| SYS-02 | Notifications delivered in-app (push via websocket / polling) | P0 |
| SYS-03 | Notification types: INFO, WARNING, ACTION_REQUIRED, SUCCESS | P0 |
| SYS-04 | TenantSettings stored as key-value JSON pairs (e.g., max login attempts, timezone) | P0 |
| SYS-05 | BlacklistedToken table for JWT invalidation on logout | P0 |

---

## 5. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| API response time (p95) | < 200ms |
| Availability | 99.9% monthly uptime |
| Authentication token TTL | Access: 15 min, Refresh: 7 days |
| Audit log retention | 7 years |
| Database | PostgreSQL with row-level tenant isolation |
| Multi-region | Single region initially; multi-region in Phase 3 |

---

## 6. System Roles (Seeded at Tenant Creation)

| Role | Description |
|------|-------------|
| `TENANT_ADMIN` | Full access to all modules purchased |
| `TENANT_OWNER` | Same as TENANT_ADMIN + billing management |
| `VIEWER` | Read-only across all permitted modules |

---

## 7. Default Permissions (Seeded at Platform Level)

Permissions follow the pattern: `module_name:action_name`

### IAM Module
- `iam:manage_users`, `iam:view_users`
- `iam:manage_roles`, `iam:view_roles`
- `iam:manage_permissions`

### Org Module
- `org:manage_business_units`, `org:view_business_units`
- `org:manage_departments`, `org:view_departments`
- `org:manage_designations`

### Workflow Module
- `workflow:manage_workflows`, `workflow:view_workflows`
- `workflow:execute_steps`

### System Module
- `system:view_audit_logs`
- `system:manage_settings`

---

## 8. Open Questions / Decisions Needed

- [ ] Should `Permission` ever be tenant-customizable (i.e., tenants define their own permissions), or always platform-defined?
- [ ] What is the max hierarchy depth for BusinessUnit and Department before UI becomes impractical?
- [ ] Should workflow branching (P2) use a no-code UI builder or JSON DSL only?

---

*Document Owner: Product Team | Review Cycle: Per sprint*
