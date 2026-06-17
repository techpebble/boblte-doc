# Database Design Document

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Owner**: Engineering Team
**Database**: PostgreSQL (via Supabase)
**ORM**: Prisma (multi-schema folder — `prismaSchemaFolder` preview feature)

---

## 1. Database Architecture

### 1.1 Technology
- **Engine**: PostgreSQL 15+
- **Hosting**: Supabase (managed PostgreSQL)
- **ORM**: Prisma with multi-file schema (`prisma/schema/`)
- **Connection**: `DATABASE_URL` (pooled), `DIRECT_URL` (direct for migrations)

### 1.2 Design Philosophy
1. **Multi-Tenant Single Database**: All tenants share one database. Isolation enforced via `tenantId` on every tenant-scoped table.
2. **Soft Delete by Default**: All business entities use `deletedAt` / `deletedBy` instead of hard deletion.
3. **Full Audit Trail**: All mutable entities carry `createdAt`, `createdBy`, `updatedAt`, `updatedBy`.
4. **Snake_case in DB**: All table names (`@@map`) and column names (`@map`) use snake_case.
5. **UUID Primary Keys**: All primary keys use `@default(uuid())` (UUID v4).
6. **Referential Integrity**: All FKs enforce referential integrity via Prisma relations.

---

## 2. Schema File Inventory

| File | Context | Tables |
|------|---------|--------|
| `base.prisma` | Config | Generator + datasource only |
| `platform.prisma` | Platform | `tenants`, `platform_users`, `subscription_plans` |
| `auth.prisma` | IAM | `users`, `roles`, `permissions`, `permission_roles`, `position_roles`, `user_business_unit_scopes`, `refresh_tokens`, `blacklisted_tokens` |
| `identity.prisma` | Identity | `owner_principals`, `external_principals`, `invitations` |
| `org.prisma` | Organization | `business_units`, `departments`, `designations`, `department_unit_map`, `positions`, `position_assignments` |
| `workflow.prisma` | Workflow | `workflows`, `workflow_steps`, `workflow_instances`, `workflow_step_executions` |
| `tasks.prisma` | Tasks | `tasks`, `task_comments` |
| `system.prisma` | System | `audit_logs`, `notifications`, `tenant_settings` |
| `hr.prisma` | HR Module | `employees` |

---

## 3. Table Reference

### 3.1 Platform Context

#### `tenants`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| company_name | varchar | NOT NULL |
| domain_prefix | varchar | UNIQUE, NOT NULL |
| is_active | boolean | DEFAULT true |
| subscription_plan_id | uuid | FK → subscription_plans |
| tenant_type | tenant_type (enum) | DEFAULT 'PRIVATE_LIMITED' |
| custom_attributes | jsonb | DEFAULT '{}' |
| created_at | timestamptz | DEFAULT now() |
| created_by | uuid | FK → platform_users |
| updated_at | timestamptz | ON UPDATE |
| updated_by | uuid | FK → platform_users |

#### `platform_users`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| email | varchar | UNIQUE, NOT NULL |
| password_hash | varchar | NOT NULL |
| first_name | varchar | NOT NULL |
| last_name | varchar | |
| is_active | boolean | DEFAULT true |
| last_login_at | timestamptz | |
| failed_login_attempts | int | DEFAULT 0 |
| locked_until | timestamptz | |
| created_at | timestamptz | DEFAULT now() |
| updated_at | timestamptz | ON UPDATE |

#### `subscription_plans`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| plan_name | varchar | UNIQUE, NOT NULL |
| max_users_allowed | int | NULLABLE |
| max_units_allowed | int | NULLABLE |
| features | jsonb | DEFAULT '{}' |
| is_active | boolean | DEFAULT true |
| created_at | timestamptz | DEFAULT now() |
| updated_at | timestamptz | ON UPDATE |

---

### 3.2 IAM Context

#### `users`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| email | varchar | NOT NULL |
| password_hash | varchar | NOT NULL |
| first_name | varchar | NOT NULL |
| last_name | varchar | |
| mobile_number | varchar | |
| is_mfa_enabled | boolean | DEFAULT false |
| mfa_secret | varchar | |
| is_active | boolean | DEFAULT true |
| last_login_at | timestamptz | |
| failed_login_attempts | int | DEFAULT 0 |
| locked_until | timestamptz | |
| user_type | user_type (enum) | DEFAULT 'EMPLOYEE' |
| custom_attributes | jsonb | DEFAULT '{}' |
| created_at | timestamptz | DEFAULT now() |
| created_by | uuid | |
| updated_at | timestamptz | ON UPDATE |
| updated_by | uuid | |
| deleted_at | timestamptz | |
| deleted_by | uuid | |

**Indexes**:
- `UNIQUE (tenant_id, email)`
- `UNIQUE (tenant_id, mobile_number)`
- `INDEX (tenant_id)`

#### `roles`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| role_name | varchar | NOT NULL |
| description | text | |
| is_system_role | boolean | DEFAULT false |
| created_at / updated_at / deleted_at | timestamptz | |
| created_by / updated_by / deleted_by | uuid | |

**Indexes**: `UNIQUE (tenant_id, role_name)`, `INDEX (tenant_id)`

#### `permissions`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| module_name | varchar | NOT NULL |
| action_name | varchar | NOT NULL |
| description | text | |

**Indexes**: `UNIQUE (module_name, action_name)`

#### `permission_roles`
| Column | Type | Constraints |
|--------|------|-------------|
| permission_id | uuid | PK (composite), FK → permissions |
| role_id | uuid | PK (composite), FK → roles |
| assigned_by | uuid | FK → users |
| assigned_at | timestamptz | DEFAULT now() |

#### `position_roles` *(sole RBAC path — roles assigned to positions)*
| Column | Type | Constraints |
|--------|------|-------------|
| position_id | uuid | PK (composite), FK → positions (CASCADE) |
| role_id | uuid | PK (composite), FK → roles (CASCADE) |
| assigned_by | uuid | FK → users |
| assigned_at | timestamptz | DEFAULT now() |

> **Permission resolution**: `User → position_assignments → positions → position_roles → roles → permission_roles → permissions`

#### `user_business_unit_scopes`
| Column | Type | Constraints |
|--------|------|-------------|
| user_id | uuid | PK (composite), FK → users |
| business_unit_id | uuid | PK (composite), FK → business_units |
| is_default_context | boolean | DEFAULT false |

#### `refresh_tokens`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| user_id | uuid? | FK → users (CASCADE) |
| platform_user_id | uuid? | FK → platform_users (CASCADE) |
| token_hash | varchar | UNIQUE |
| expires_at | timestamptz | NOT NULL |
| revoked_at | timestamptz | |
| created_at | timestamptz | DEFAULT now() |

> ⚠️ **Pending**: DB constraint needed: `CHECK (user_id IS NOT NULL OR platform_user_id IS NOT NULL)`

#### `blacklisted_tokens`
| Column | Type | Constraints |
|--------|------|-------------|
| jti | varchar | PK |
| expires_at | timestamptz | NOT NULL |

> ⚠️ **Pending**: `INDEX (expires_at)` for cleanup job performance

---

### 3.3 Identity Context

#### `owner_principals`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants |
| user_id | uuid | UNIQUE, FK → users |
| ownership_type | ownership_type (enum) | NOT NULL |
| ownership_share | decimal(5,2) | |
| effective_from | date | NOT NULL |
| effective_to | date | |
| created_at | timestamptz | |
| created_by | uuid | |

#### `external_principals`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants |
| user_id | uuid | UNIQUE, FK → users |
| organization | varchar | |
| principal_type | principal_type (enum) | NOT NULL |
| access_start_date | date | |
| access_end_date | date | |
| invited_by | uuid | FK → users |
| notes | text | |
| created_at | timestamptz | |
| created_by | uuid | |

#### `invitations`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants |
| email | varchar | NOT NULL |
| user_type | user_type (enum) | NOT NULL |
| principal_type | principal_type (enum) | |
| ownership_type | ownership_type (enum) | |
| access_end_date | date | |
| invited_by | uuid | |
| token | varchar | UNIQUE |
| expires_at | timestamptz | NOT NULL |
| accepted_at | timestamptz | |
| created_at | timestamptz | DEFAULT now() |

---

### 3.4 Organization Context

#### `business_units`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| unit_name | varchar | NOT NULL |
| parent_business_unit_id | uuid? | FK → business_units (self-ref) |
| is_default | boolean | DEFAULT false |
| custom_attributes | jsonb | DEFAULT '{}' |
| audit fields | | |

**Indexes**: `UNIQUE (tenant_id, unit_name)`, `INDEX (tenant_id)`

#### `departments`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| department_name | varchar | NOT NULL |

| parent_department_id | uuid? | FK → departments (self-ref) |
| is_default | boolean | DEFAULT false |
| audit fields | | |

**Indexes**: `UNIQUE (tenant_id, department_name)`, `INDEX (tenant_id)`


#### `designations`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants |
| designation_name | varchar | NOT NULL |
| description | text | |
| audit fields | | |

#### `department_unit_map`
| Column | Type | Constraints |
|--------|------|-------------|
| department_id | uuid | PK (composite), FK → departments |
| business_unit_id | uuid | PK (composite), FK → business_units |
| tenant_id | uuid | FK → tenants |
| created_at | timestamptz | |
| created_by | uuid | |

#### `positions`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| position_code | varchar | NOT NULL |
| position_name | varchar | NOT NULL |
| business_unit_id | uuid? | FK → business_units |
| department_id | uuid? | FK → departments |
| parent_position_id | uuid? | FK → positions (self-ref) |
| position_type | position_type (enum) | NOT NULL |
| is_vacant | boolean | DEFAULT false |
| audit fields | | |

**Indexes**: `UNIQUE (tenant_id, position_code)`, `INDEX (tenant_id)`, `INDEX (business_unit_id)`, `INDEX (department_id)`

#### `position_assignments`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| position_id | uuid | FK → positions (CASCADE) |
| user_id | uuid | FK → users (CASCADE) |
| is_primary | boolean | DEFAULT true |
| effective_from | timestamptz | NOT NULL |
| effective_to | timestamptz | NULLABLE |
| created_at | timestamptz | DEFAULT now() |
| created_by | uuid | |

**Indexes**: `INDEX (position_id)`, `INDEX (user_id)`, `INDEX (tenant_id)`

---

### 3.5 Workflow Context

#### `workflows`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| workflow_name | varchar | NOT NULL |
| description | text | |
| status | workflow_status (enum) | DEFAULT 'DRAFT' |
| audit fields | | |

**Indexes**: `UNIQUE (tenant_id, workflow_name)`, `INDEX (tenant_id)`

#### `workflow_steps`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| workflow_id | uuid | FK → workflows (CASCADE) |
| step_name | varchar | NOT NULL |
| step_order | int | NOT NULL |
| step_type | workflow_step_type (enum) | NOT NULL |
| assigned_to_role_id | uuid? | FK → roles |
| assigned_to_user_id | uuid? | FK → users |
| is_required | boolean | DEFAULT true |
| audit fields | | |

**Indexes**: `UNIQUE (workflow_id, step_order)`

#### `workflow_instances`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| workflow_id | uuid | FK → workflows |
| reference_type | varchar | NOT NULL |
| reference_id | uuid | NOT NULL |
| status | workflow_instance_status (enum) | DEFAULT 'PENDING' |
| started_at | timestamptz | DEFAULT now() |
| completed_at | timestamptz | |
| initiated_by | uuid? | FK → users |

**Indexes**: `INDEX (tenant_id)`, `INDEX (reference_type, reference_id)`

#### `workflow_step_executions`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| workflow_instance_id | uuid | FK → workflow_instances (CASCADE) |
| workflow_step_id | uuid | FK → workflow_steps |
| status | workflow_step_execution_status (enum) | DEFAULT 'PENDING' |
| actioned_by | uuid? | FK → users |
| action_note | text | |
| actioned_at | timestamptz | |

**Indexes**: `INDEX (workflow_instance_id)`

---

### 3.6 Task Context

#### `tasks`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| title | varchar | NOT NULL |
| description | text | |
| task_type | task_type (enum) | DEFAULT 'NORMAL' |
| status | task_status (enum) | DEFAULT 'TO_DO' |
| priority | task_priority (enum) | DEFAULT 'MEDIUM' |
| frequency | task_frequency (enum) | |
| workflow_id | uuid? | FK → workflows |
| assignee_type | task_assignee_type (enum) | |
| assignee_department_id | uuid? | FK → departments |
| assignee_business_unit_id | uuid? | FK → business_units |
| assignee_user_id | uuid? | FK → users |
| due_date | timestamptz | |
| custom_attributes | jsonb | DEFAULT '{}' |
| audit fields | | |

**Indexes**: `INDEX (tenant_id)`, `INDEX (tenant_id, status)`, `INDEX (assignee_user_id)`, `INDEX (assignee_department_id)`, `INDEX (tenant_id, due_date)`

> ⚠️ **Pending**: DB check constraint for assignee exclusivity (see Schema Review)

#### `task_comments`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| task_id | uuid | FK → tasks (CASCADE) |
| parent_comment_id | uuid? | FK → task_comments (self-ref) |
| comment_text | text | NOT NULL |
| author_id | uuid | FK → users |
| created_at | timestamptz | DEFAULT now() |
| updated_at | timestamptz | ON UPDATE |
| updated_by / deleted_at / deleted_by | | |

---

### 3.7 System Context

#### `audit_logs`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid? | FK → tenants |
| actor_id | uuid? | User or PlatformUser who acted |
| action | varchar | e.g., "CREATE", "UPDATE", "DELETE" |
| resource_type | varchar | e.g., "USER", "DEPARTMENT" |
| resource_id | uuid? | ID of affected entity |
| old_values | jsonb | Before state |
| new_values | jsonb | After state |
| ip_address | varchar | |
| user_agent | varchar | |
| occurred_at | timestamptz | DEFAULT now() |

> ⚠️ **Pending**: `actor_type` column to distinguish User vs PlatformUser vs SYSTEM actor

**Indexes**: `INDEX (tenant_id, occurred_at DESC)`, `INDEX (resource_type, resource_id)`

#### `notifications`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| recipient_id | uuid | FK → users (CASCADE) |
| title | varchar | NOT NULL |
| body | text | |
| type | notification_type (enum) | DEFAULT 'INFO' |
| reference_type | varchar? | Entity type linked |
| reference_id | uuid? | Entity ID linked |
| is_read | boolean | DEFAULT false |
| read_at | timestamptz | |
| created_at | timestamptz | DEFAULT now() |
| created_by | uuid | |

**Indexes**: `INDEX (tenant_id)`, `INDEX (recipient_id, is_read)`

#### `tenant_settings`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| setting_key | varchar | NOT NULL |
| setting_value | jsonb | NOT NULL |
| audit fields | | |

**Indexes**: `UNIQUE (tenant_id, setting_key)`, `INDEX (tenant_id)`

---

### 3.8 HR Module

#### `employees`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants (CASCADE) |
| user_id | uuid? | UNIQUE, FK → users |
| employee_code | varchar? | |
| first_name | varchar | NOT NULL |
| last_name | varchar | |
| email | varchar? | |
| mobile_number | varchar? | |
| designation_id | uuid? | FK → designations |
| department_id | uuid? | FK → departments |
| business_unit_id | uuid? | FK → business_units |
| manager_id | uuid? | FK → employees (self-ref) |
| date_of_joining | date | |
| employment_status | employment_status (enum) | DEFAULT 'ACTIVE' |
| employment_type | employment_type (enum)? | |
| is_active | boolean | DEFAULT true |
| custom_attributes | jsonb | DEFAULT '{}' |
| audit fields | | |

**Indexes**: `UNIQUE (tenant_id, email)`, `UNIQUE (tenant_id, mobile_number)`, `INDEX (tenant_id)`, `INDEX (user_id)`, `INDEX (department_id)`, `INDEX (manager_id)`, `INDEX (tenant_id, employment_status)`

> ⚠️ **Pending**: `UNIQUE (tenant_id, employee_code)` constraint
> ⚠️ **Pending**: `@@map` on `EmploymentStatus` and `EmploymentType` enums

---

## 4. Enum Types

| Enum | Values |
|------|--------|
| `user_type` | OWNER, EMPLOYEE, EXTERNAL, SYSTEM |
| `tenant_type` | SOLE_PROPRIETORSHIP, PARTNERSHIP, PRIVATE_LIMITED, PUBLIC_LIMITED, LLP, TRUST, NGO |
| `ownership_type` | SOLE_PROPRIETOR, PARTNER, DIRECTOR, TRUSTEE |
| `principal_type` | AUDITOR, CONSULTANT, VENDOR |
| `position_type` | BUSINESS_UNIT_HEAD, DEPARTMENT_HEAD, MANAGER, SUPERVISOR, TEAM_LEAD, EXECUTIVE, STAFF, CUSTOM |
| `workflow_status` | DRAFT, PUBLISHED, ARCHIVED |
| `workflow_step_type` | APPROVAL, REVIEW, AUTOMATED, NOTIFICATION |
| `workflow_instance_status` | PENDING, IN_PROGRESS, COMPLETED, REJECTED, CANCELLED |
| `workflow_step_execution_status` | PENDING, IN_PROGRESS, APPROVED, REJECTED, SKIPPED |
| `task_type` | NORMAL, WORKFLOW |
| `task_status` | TO_DO, IN_PROGRESS, DONE, CANCELLED, ON_HOLD |
| `task_priority` | LOW, MEDIUM, HIGH, CRITICAL |
| `task_frequency` | DAILY, WEEKLY, MONTHLY, YEARLY |
| `task_assignee_type` | DEPARTMENT, BUSINESS_UNIT, USER |
| `notification_type` | INFO, WARNING, ACTION_REQUIRED, SUCCESS |
| `employment_status` | ACTIVE, PROBATION, ON_LEAVE, SUSPENDED, RESIGNED, TERMINATED, RETIRED |
| `employment_type` | FULL_TIME, PART_TIME, CONTRACT, TEMPORARY, INTERN, TRAINEE |

---

## 5. Indexing Strategy

| Pattern | Index |
|---------|-------|
| Tenant scoping | `INDEX (tenant_id)` on every tenant-scoped table |
| Soft delete filtering | `INDEX (tenant_id, deleted_at)` — consider adding |
| Status-based listing | `INDEX (tenant_id, status)` |
| Time-range queries | `INDEX (tenant_id, occurred_at DESC)` |
| Polymorphic lookup | `INDEX (resource_type, resource_id)` |
| Token expiry cleanup | `INDEX (expires_at)` on `blacklisted_tokens` |
| User-specific | `INDEX (user_id)`, `INDEX (assignee_user_id)` |

---

## 6. Pending DB Work (Priority)

| Priority | Item |
|----------|------|
| 🔴 P0 | Add `@@map` to `EmploymentStatus` and `EmploymentType` in `hr.prisma` |
| 🔴 P0 | Add `@@unique([tenantId, employeeCode])` in `hr.prisma` |
| 🔴 P0 | Add `@@index([expiresAt])` to `BlacklistedToken` in `auth.prisma` |
| 🔴 P0 | Add SQL check constraint for `tasks` assignee exclusivity |
| 🟡 P1 | Add `actor_type` to `audit_logs` |
| 🟡 P1 | Add `deleted_at` / `deleted_by` to `tenants` |
| 🟡 P1 | Add SQL check constraint on `refresh_tokens` owner |
| 🔵 P2 | Add `parent_task_id` to `tasks` for subtask support |
| 🔵 P2 | Create `tenant_subscriptions` table for billing history |

---

*Document Owner: Engineering Team | Review Cycle: On schema change*
