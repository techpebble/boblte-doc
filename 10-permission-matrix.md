# Permission Matrix

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Owner**: Engineering Team

---

## 1. Overview

This document defines all platform permissions and maps them to system roles. Tenant admins can create **custom roles** by selecting any combination of these permissions.

### Legend
| Symbol | Meaning |
|--------|---------|
| ✅ | Granted |
| ❌ | Denied |
| ⚡ | Self only (own record) |
| 🏢 | Own department/BU only |

---

## 2. System Roles

| Role | `isSystemRole` | Description |
|------|---------------|-------------|
| `TENANT_OWNER` | true | Full access to all purchased modules |
| `TENANT_ADMIN` | true | All non-billing permissions |
| `DEPT_MANAGER` | true | Manage own department's users, tasks |
| `VIEWER` | true | Read-only across all permitted modules |
| `EMPLOYEE` | true | Self-service only (profile, my tasks, my notifications) |

> These roles are assigned to **Positions** via `PositionRole`. Users gain permissions by being assigned to a Position, never directly.

---

## 3. IAM Module Permissions

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `iam:manage_users` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `iam:view_users` | ✅ | ✅ | 🏢 | ✅ | ❌ |
| `iam:manage_roles` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `iam:view_roles` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `iam:manage_permissions` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `iam:invite_users` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `iam:manage_owner_principals` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `iam:manage_external_principals` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `iam:view_external_principals` | ✅ | ✅ | ❌ | ✅ | ❌ |

---

## 4. Organization Module Permissions

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `org:manage_business_units` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `org:view_business_units` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `org:manage_departments` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `org:view_departments` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `org:manage_designations` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `org:view_designations` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `org:manage_dept_unit_map` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `org:manage_positions` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `org:view_positions` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `org:assign_positions` | ✅ | ✅ | ❌ | ❌ | ❌ |

---

## 5. Workflow Module Permissions

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `workflow:manage_workflows` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `workflow:view_workflows` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `workflow:execute_steps` | ✅ | ✅ | ✅ | ❌ | ✅ |
| `workflow:cancel_instances` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `workflow:view_instances` | ✅ | ✅ | 🏢 | ✅ | ⚡ |

---

## 6. Task Module Permissions

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `tasks:create` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `tasks:view_all` | ✅ | ✅ | 🏢 | ✅ | ❌ |
| `tasks:view_own` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `tasks:edit` | ✅ | ✅ | ✅ | ❌ | ⚡ |
| `tasks:assign` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `tasks:change_status` | ✅ | ✅ | ✅ | ❌ | ⚡ |
| `tasks:delete` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `tasks:comment` | ✅ | ✅ | ✅ | ❌ | ✅ |

---

## 7. HR Module Permissions (Optional Module)

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `hr:manage_employees` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `hr:view_employees` | ✅ | ✅ | 🏢 | ✅ | ⚡ |
| `hr:manage_employment_status` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `hr:view_org_chart` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `hr:manage_manager_hierarchy` | ✅ | ✅ | ❌ | ❌ | ❌ |

---

## 8. System Module Permissions

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `system:view_audit_logs` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `system:manage_settings` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `system:view_settings` | ✅ | ✅ | ❌ | ✅ | ❌ |
| `system:view_notifications` | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 9. Field Executive Module Permissions (Optional Module)

| Permission | `TENANT_OWNER` | `TENANT_ADMIN` | `DEPT_MANAGER` | `VIEWER` | `EMPLOYEE` |
|------------|:--------------:|:--------------:|:--------------:|:--------:|:----------:|
| `field:check_in` | ✅ | ✅ | ✅ | ❌ | ✅ |
| `field:log_visit` | ✅ | ✅ | ✅ | ❌ | ✅ |
| `field:submit_dwr` | ✅ | ✅ | ✅ | ❌ | ✅ |
| `field:view_own` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `field:view_team` | ✅ | ✅ | 🏢 | ✅ | ❌ |
| `field:manage_geofences` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `field:view_live_map` | ✅ | ✅ | 🏢 | ✅ | ❌ |

---

## 10. Full Permission Catalogue

> Used for seeding the `permissions` table.

```typescript
// permissions.seed.ts
export const PLATFORM_PERMISSIONS = [
  // IAM
  { moduleName: 'iam', actionName: 'manage_users' },
  { moduleName: 'iam', actionName: 'view_users' },
  { moduleName: 'iam', actionName: 'manage_roles' },
  { moduleName: 'iam', actionName: 'view_roles' },
  { moduleName: 'iam', actionName: 'manage_permissions' },
  { moduleName: 'iam', actionName: 'invite_users' },
  { moduleName: 'iam', actionName: 'manage_owner_principals' },
  { moduleName: 'iam', actionName: 'manage_external_principals' },
  { moduleName: 'iam', actionName: 'view_external_principals' },

  // ORG
  { moduleName: 'org', actionName: 'manage_business_units' },
  { moduleName: 'org', actionName: 'view_business_units' },
  { moduleName: 'org', actionName: 'manage_departments' },
  { moduleName: 'org', actionName: 'view_departments' },
  { moduleName: 'org', actionName: 'manage_designations' },
  { moduleName: 'org', actionName: 'view_designations' },
  { moduleName: 'org', actionName: 'manage_dept_unit_map' },
  { moduleName: 'org', actionName: 'manage_positions' },
  { moduleName: 'org', actionName: 'view_positions' },
  { moduleName: 'org', actionName: 'assign_positions' },

  // WORKFLOW
  { moduleName: 'workflow', actionName: 'manage_workflows' },
  { moduleName: 'workflow', actionName: 'view_workflows' },
  { moduleName: 'workflow', actionName: 'execute_steps' },
  { moduleName: 'workflow', actionName: 'cancel_instances' },
  { moduleName: 'workflow', actionName: 'view_instances' },

  // TASKS
  { moduleName: 'tasks', actionName: 'create' },
  { moduleName: 'tasks', actionName: 'view_all' },
  { moduleName: 'tasks', actionName: 'view_own' },
  { moduleName: 'tasks', actionName: 'edit' },
  { moduleName: 'tasks', actionName: 'assign' },
  { moduleName: 'tasks', actionName: 'change_status' },
  { moduleName: 'tasks', actionName: 'delete' },
  { moduleName: 'tasks', actionName: 'comment' },

  // HR MODULE
  { moduleName: 'hr', actionName: 'manage_employees' },
  { moduleName: 'hr', actionName: 'view_employees' },
  { moduleName: 'hr', actionName: 'manage_employment_status' },
  { moduleName: 'hr', actionName: 'view_org_chart' },
  { moduleName: 'hr', actionName: 'manage_manager_hierarchy' },

  // SYSTEM
  { moduleName: 'system', actionName: 'view_audit_logs' },
  { moduleName: 'system', actionName: 'manage_settings' },
  { moduleName: 'system', actionName: 'view_settings' },
  { moduleName: 'system', actionName: 'view_notifications' },

  // FIELD
  { moduleName: 'field', actionName: 'check_in' },
  { moduleName: 'field', actionName: 'log_visit' },
  { moduleName: 'field', actionName: 'submit_dwr' },
  { moduleName: 'field', actionName: 'view_own' },
  { moduleName: 'field', actionName: 'view_team' },
  { moduleName: 'field', actionName: 'manage_geofences' },
  { moduleName: 'field', actionName: 'view_live_map' },
];
```

---

## 11. System Role → Permission Seed Mapping

```typescript
// roles.seed.ts
// Roles are assigned to POSITIONS via PositionRole, not directly to users.
// On tenant creation, a root 'Company Owner' position is seeded and TENANT_OWNER role is assigned to it.
export const SYSTEM_ROLE_PERMISSIONS = {
  TENANT_OWNER: '*',  // All permissions

  TENANT_ADMIN: [
    // All except manage_owner_principals
    'iam:manage_users', 'iam:view_users', 'iam:manage_roles', 'iam:view_roles',
    'iam:manage_permissions', 'iam:invite_users',
    'iam:manage_external_principals', 'iam:view_external_principals',
    'org:*', 'workflow:*', 'tasks:*', 'hr:*', 'system:*', 'field:*',
  ],

  DEPT_MANAGER: [
    'iam:view_roles', 'org:view_business_units', 'org:view_departments',
    'org:view_designations', 'org:view_positions',
    'workflow:execute_steps', 'workflow:view_workflows', 'workflow:view_instances',
    'tasks:create', 'tasks:view_own', 'tasks:view_all',
    'tasks:edit', 'tasks:assign', 'tasks:change_status', 'tasks:comment',
    'hr:view_employees', 'hr:view_org_chart', 'system:view_notifications',
    'field:view_team', 'field:view_live_map',
  ],

  VIEWER: [
    'iam:view_users', 'iam:view_roles', 'iam:view_external_principals',
    'org:view_business_units', 'org:view_departments', 'org:view_designations',
    'org:view_positions', 'workflow:view_workflows', 'workflow:view_instances',
    'tasks:view_all', 'tasks:view_own', 'hr:view_employees', 'hr:view_org_chart',
    'system:view_settings', 'system:view_notifications',
    'field:view_own', 'field:view_team',
  ],

  EMPLOYEE: [
    'org:view_business_units', 'org:view_departments', 'org:view_designations',
    'org:view_positions', 'tasks:view_own', 'tasks:change_status', 'tasks:comment',
    'workflow:execute_steps', 'workflow:view_instances',
    'hr:view_org_chart', 'system:view_notifications',
    'field:check_in', 'field:log_visit', 'field:submit_dwr', 'field:view_own',
  ],
};
```

---

## 12. User Type vs Access Capability

| User Type | Can Log In | Position Required | Module Access |
|-----------|-----------|-----------------|---------------|
| `OWNER` | ✅ | Auto-seeded ("Company Owner" position at tenant creation) | All |
| `EMPLOYEE` | ✅ | Yes — assigned by admin | Based on position's roles |
| `EXTERNAL` | ✅ | Yes — e.g., "External Auditor" position with `effectiveTo` | Based on position's roles |
| `SYSTEM` | ❌ | None — API key based, no JWT role resolution | API key-based, service calls only |

---

## 13. Context-Aware Permissions

Permissions are scoped to the Position's `businessUnitId`. A user with two positions in different BUs gets different permissions per context:

```
User: Ravi Kumar
  Position 1: "Finance Manager — Mumbai" (isPrimary: true)
    → Context: Mumbai Branch → Roles: DEPT_MANAGER → full finance + task view for Mumbai

  Position 2: "Viewer — Delhi" (isPrimary: false)
    → Context: Delhi Branch → Roles: VIEWER → read-only across Delhi
```

Context switching is driven by `PositionAssignment.position.businessUnitId`. Users call `POST /context/switch` with the target `businessUnitId`, which issues a new access token scoped to that BU's positions.

---

*Document Owner: Engineering Team | Review Cycle: On permission/role change*
