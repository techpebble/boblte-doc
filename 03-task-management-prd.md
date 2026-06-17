# Task Management Module — Product Requirements Document (PRD)

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Module Code**: `TASKS`
**Owner**: Product Team
**Dependency**: Core Module (always required)

---

## 1. Overview

The Task Management Module enables tenants to create, assign, track, and comment on tasks across the organization. Tasks can be manual (standalone) or workflow-driven (auto-created by the Workflow Engine).

It is part of the **Core** layer and has **zero dependency on the HR module** — all assignees reference `User`, `Department`, or `BusinessUnit` (Core entities).

---

## 2. Scope

### In Scope
- Task creation, editing, and soft deletion
- Multi-type assignment: User, Department, Business Unit
- Task priority and status lifecycle management
- Recurring task scheduling (Daily, Weekly, Monthly, Yearly)
- Threaded task comments
- Workflow-linked tasks
- Task filtering and reporting

### Out of Scope
- Project-level task grouping with milestones (→ Project Module, Roadmap Phase 6)
- Billable hours / timesheet logging (→ Project Module)
- External task integrations (Jira, Trello) — Phase 2 consideration

---

## 3. User Personas

| Persona | Role in Tasks |
|---------|--------------|
| **TenantAdmin** | Create tasks, manage all tasks |
| **Manager / Dept Head** | Create and assign tasks to their team |
| **Employee** | View and update tasks assigned to them |
| **Workflow Engine** | Auto-creates WORKFLOW-type tasks on instance start |

---

## 4. Task Lifecycle

```
                    ┌──────────┐
                    │  TO_DO   │◄──────────────────┐
                    └────┬─────┘                   │
                         │ Start work               │ Reopen
                    ┌────▼─────┐                   │
                    │IN_PROGRESS│                  │
                    └────┬─────┘                   │
               ┌─────────┼──────────┐              │
               │         │          │              │
          ┌────▼───┐  ┌──▼───┐  ┌──▼──────┐       │
          │  DONE  │  │ON_HOLD│  │CANCELLED│───────┘
          └────────┘  └──────┘  └─────────┘
```

### Status Definitions

| Status | Description |
|--------|-------------|
| `TO_DO` | Created, not started |
| `IN_PROGRESS` | Assignee is actively working |
| `DONE` | Completed successfully |
| `ON_HOLD` | Paused, pending external input |
| `CANCELLED` | Abandoned or no longer needed |

---

## 5. Feature Requirements

### 5.1 Task Creation & Management

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-01 | Any user with `tasks:create` permission can create a task | P0 |
| TK-02 | Task title is required; description is optional (supports Markdown) | P0 |
| TK-03 | Task type: NORMAL (manual) or WORKFLOW (engine-created) | P0 |
| TK-04 | Task priority: LOW, MEDIUM (default), HIGH, CRITICAL | P0 |
| TK-05 | Task status defaults to TO_DO on creation | P0 |
| TK-06 | Task has an optional due date | P0 |
| TK-07 | Tasks support soft delete (`deletedAt`, `deletedBy`) | P0 |
| TK-08 | Tasks can store extensible metadata in `customAttributes` JSON | P1 |

### 5.2 Task Assignment

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-A-01 | Tasks can be assigned to one of: a specific User, a Department, or a Business Unit | P0 |
| TK-A-02 | Assignee type is captured via `assigneeType` enum (USER, DEPARTMENT, BUSINESS_UNIT) | P0 |
| TK-A-03 | Exactly one assignee FK must be set when assigneeType is specified (DB constraint enforced) | P0 |
| TK-A-04 | Unassigned tasks (no assigneeType) are allowed (inbox/backlog) | P0 |
| TK-A-05 | Re-assignment is allowed and logged in audit log | P1 |

### 5.3 Recurring Tasks

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-R-01 | Tasks can have a frequency: DAILY, WEEKLY, MONTHLY, YEARLY | P1 |
| TK-R-02 | Recurring task scheduler creates new task instances on schedule | P1 |
| TK-R-03 | Each recurring instance is independent (separate status, comments) | P1 |
| TK-R-04 | Recurring tasks can be paused/cancelled | P1 |

### 5.4 Workflow-Linked Tasks

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-W-01 | Tasks with `taskType = WORKFLOW` are created by the Workflow Engine | P0 |
| TK-W-02 | `workflowId` references the source Workflow definition | P0 |
| TK-W-03 | Completing a WORKFLOW task triggers the next step in the WorkflowInstance | P0 |
| TK-W-04 | WORKFLOW tasks cannot be manually deleted (only cancelled via workflow rejection) | P0 |

### 5.5 Comments & Collaboration

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-C-01 | Any user with task access can add a comment | P0 |
| TK-C-02 | Comments support threading (`parentCommentId`) for nested replies | P1 |
| TK-C-03 | Comment author is recorded; comments support soft delete | P0 |
| TK-C-04 | Comment text supports plain text; Markdown in Phase 2 | P1 |

### 5.6 Notifications

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-N-01 | Assignee receives a notification when a task is assigned to them | P0 |
| TK-N-02 | Task creator notified when assignee updates status | P1 |
| TK-N-03 | Overdue task notification (T+0, T+1 day) | P1 |

### 5.7 Filtering & Reporting

| ID | Requirement | Priority |
|----|-------------|----------|
| TK-F-01 | Filter tasks by: status, priority, assignee, department, due date range | P0 |
| TK-F-02 | My tasks view (tasks assigned to me or my department) | P0 |
| TK-F-03 | Team tasks view (manager sees tasks of their department/BU) | P1 |
| TK-F-04 | Task completion report: % done per department per period | P2 |

---

## 6. Permissions

| Permission | Description |
|------------|-------------|
| `tasks:create` | Create a new task |
| `tasks:view_all` | View all tasks in the tenant |
| `tasks:view_own` | View tasks assigned to self |
| `tasks:edit` | Edit task title, description, priority, due date |
| `tasks:assign` | Assign or reassign tasks |
| `tasks:change_status` | Move task to any status |
| `tasks:delete` | Soft-delete tasks |
| `tasks:comment` | Add comments to tasks |

---

## 7. Business Rules

1. A task without an `assigneeType` is an **unassigned backlog item** — visible to admins only.
2. When a Department is the assignee, any member of that department can claim / work on the task.
3. When a Business Unit is the assignee, any member of that business unit can work on the task.
4. WORKFLOW tasks auto-transition to DONE when the linked WorkflowInstance COMPLETED.
5. Deleting a task soft-deletes all its comments.
6. Frequency field is only meaningful when the task has a `dueDate`.

---

## 8. API Endpoints (Summary)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/tasks` | Create task |
| GET | `/tasks` | List tasks (filterable) |
| GET | `/tasks/my` | My tasks |
| GET | `/tasks/:id` | Get task detail |
| PATCH | `/tasks/:id` | Update task |
| DELETE | `/tasks/:id` | Soft-delete task |
| PATCH | `/tasks/:id/status` | Change task status |
| GET | `/tasks/:id/comments` | List comments |
| POST | `/tasks/:id/comments` | Add comment |
| PATCH | `/tasks/:id/comments/:cid` | Edit comment |
| DELETE | `/tasks/:id/comments/:cid` | Delete comment |

---

## 9. Open Questions

- [ ] Should DEPARTMENT-assigned tasks allow individual employees to "claim" them (change assigneeType to USER)?
- [ ] Should subtasks be in Phase 1 or Phase 2? (DB: `parentTaskId` on `Task`)
- [ ] What is the notification delivery mechanism: polling, SSE, or WebSocket?

---

*Document Owner: Product Team | Review Cycle: Per sprint*
