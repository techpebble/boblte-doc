# Workflow Engine Design

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Active
**Owner**: Engineering Team

---

## 1. Overview

The Boblte Workflow Engine is a **tenant-configurable business process automation** system built into the Core module. It enables tenants to define multi-step approval, review, and notification processes that can be attached to any business entity (tasks, leave applications, purchase orders, etc.).

The engine separates **definition** (the template) from **execution** (the runtime), following the BPMN-inspired pattern used in enterprise systems.

---

## 2. Key Concepts

| Concept | Model | Description |
|---------|-------|-------------|
| **Workflow** | `Workflow` | A named, reusable process template |
| **Step** | `WorkflowStep` | A single node in the process (who, what type, what order) |
| **Instance** | `WorkflowInstance` | A single run of a workflow, triggered for a specific business entity |
| **Step Execution** | `WorkflowStepExecution` | Runtime record of one step within one instance |

---

## 3. Data Model

### 3.1 Workflow (Definition)

```
Workflow
  id           — UUID
  tenantId     — Tenant scope
  workflowName — Unique per tenant
  description  — Optional description
  status       — DRAFT | PUBLISHED | ARCHIVED
  createdBy / updatedBy / deletedAt (audit fields)
```

**Status lifecycle**:
```
DRAFT → PUBLISHED → ARCHIVED
         ↑
   Only PUBLISHED workflows can trigger instances
```

### 3.2 WorkflowStep (Step Template)

```
WorkflowStep
  workflowId        — Parent workflow
  stepName          — Human label (e.g., "Manager Approval")
  stepOrder         — Integer (1, 2, 3...) defines sequence
  stepType          — APPROVAL | REVIEW | AUTOMATED | NOTIFICATION
  assignedToRoleId  — Role responsible (nullable)
  assignedToUserId  — Specific user responsible (nullable)
  isRequired        — If false, step can be skipped
```

**Step Types**:

| Type | Description | Human Action Required |
|------|-------------|----------------------|
| `APPROVAL` | Must be explicitly approved or rejected | ✅ Yes |
| `REVIEW` | Must be acknowledged/reviewed | ✅ Yes |
| `AUTOMATED` | System action (send email, update field) | ❌ No |
| `NOTIFICATION` | Send notification and auto-proceed | ❌ No |

**Assignment Priority**: `assignedToUserId` > `assignedToRoleId` > escalation rules.

### 3.3 WorkflowInstance (Execution Context)

```
WorkflowInstance
  tenantId      — Tenant scope
  workflowId    — Which workflow template
  referenceType — "TASK" | "LEAVE_APPLICATION" | "PURCHASE_ORDER" | ...
  referenceId   — UUID of the triggering entity
  status        — PENDING | IN_PROGRESS | COMPLETED | REJECTED | CANCELLED
  startedAt     — Timestamp of instance creation
  completedAt   — Timestamp of final resolution
  initiatedBy   — UserId who triggered this instance
```

### 3.4 WorkflowStepExecution (Per-Step Runtime)

```
WorkflowStepExecution
  workflowInstanceId — Parent instance
  workflowStepId     — Which step template
  status             — PENDING | IN_PROGRESS | APPROVED | REJECTED | SKIPPED
  actionedBy         — UserId who acted
  actionNote         — Optional comment/reason
  actionedAt         — Timestamp of action
```

---

## 4. Workflow Execution Flow

```
Business Entity Created (e.g., LeaveApplication)
        │
        ▼
TriggerWorkflow(workflowId, referenceType, referenceId)
        │
        ▼
WorkflowInstance created (status: PENDING)
        │
        ▼
Load WorkflowSteps (ordered by stepOrder)
        │
        ▼
For each step → create WorkflowStepExecution (status: PENDING)
        │
        ▼
Activate Step 1 → status: IN_PROGRESS
  → Notify assignedRole or assignedUser
        │
        ▼
Actioner approves/rejects
  → WorkflowStepExecution updated (status: APPROVED/REJECTED, actionedBy, actionNote)
        │
        ├── if REJECTED → WorkflowInstance status: REJECTED → Stop
        │
        └── if APPROVED
              │
              ├── if more steps → Activate next step → Loop
              │
              └── if last step → WorkflowInstance status: COMPLETED
                        → Notify initiatedBy
                        → Trigger post-completion action
```

---

## 5. Step Execution State Machine

```
           ┌─────────┐
           │ PENDING  │
           └────┬─────┘
                │ Step activated
           ┌────▼──────┐
           │IN_PROGRESS│
           └────┬───── ┘
      ┌─────────┼──────────┬────────────┐
      │         │          │            │
  ┌───▼───┐ ┌──▼────┐ ┌───▼────┐ ┌────▼───┐
  │APPROVED│ │REJECTED│ │SKIPPED │ │(timeout)│
  └────────┘ └───────┘ └────────┘ └─────────┘
                                        │
                                    escalate
```

---

## 6. Instance Status State Machine

```
PENDING → IN_PROGRESS → COMPLETED
                     ↘ REJECTED
                     ↘ CANCELLED
```

- `PENDING`: Instance created, not yet started (deferred start use case)
- `IN_PROGRESS`: At least one step is being actioned
- `COMPLETED`: All required steps APPROVED
- `REJECTED`: Any required step REJECTED
- `CANCELLED`: Manually cancelled (by initiator or admin)

---

## 7. Workflow Trigger API

```http
POST /workflows/:workflowId/trigger
Content-Type: application/json

{
  "referenceType": "LEAVE_APPLICATION",
  "referenceId": "uuid-of-leave-application"
}
```

Response:
```json
{
  "workflowInstanceId": "uuid",
  "status": "IN_PROGRESS",
  "currentStep": {
    "stepId": "uuid",
    "stepName": "Manager Approval",
    "assignedTo": { "type": "role", "roleId": "uuid", "roleName": "DEPT_MANAGER" }
  }
}
```

---

## 8. Action API (Approve / Reject / Skip)

```http
POST /workflow-instances/:instanceId/steps/:executionId/action
Content-Type: application/json

{
  "action": "APPROVED",   // APPROVED | REJECTED | SKIPPED
  "note": "Approved as per policy."
}
```

---

## 9. Workflow Notification Events

On each state transition, the engine dispatches a notification:

| Event | Recipient | Notification Type |
|-------|-----------|------------------|
| Step assigned | Assignee (user or role members) | `ACTION_REQUIRED` |
| Step approved | Instance initiator | `INFO` |
| Step rejected | Instance initiator + previous actors | `WARNING` |
| Instance completed | Instance initiator | `SUCCESS` |
| Instance rejected | Instance initiator | `WARNING` |

---

## 10. Linking Workflows to Tasks

A `Task` with `taskType = WORKFLOW` is auto-created by the engine:
- One task per workflow instance
- Task title = `[Workflow Name] — [referenceType] [referenceId]`
- Task status mirrors instance status
- Completing the task resolves the current pending step

---

## 11. Planned Enhancements (Roadmap)

### 11.1 Conditional Branching (P2)

```prisma
model WorkflowStep {
  ...
  stepCondition    Json?    // rule DSL
  onApproveStepId  String?  // jump to specific step on approval
  onRejectStepId   String?  // jump to specific step on rejection
  ...
}
```

**Rule DSL Example**:
```json
{
  "field": "amount",
  "operator": "gt",
  "value": 100000,
  "then": "step-cfo-approval",
  "else": "step-manager-approval"
}
```

### 11.2 Parallel Steps (P2)

```prisma
isParallel Boolean @default(false) @map("is_parallel")
```

Steps with the same `stepOrder` and `isParallel = true` execute simultaneously. Instance proceeds only when all parallel steps are resolved.

### 11.3 SLA Escalation (P2)

```prisma
slaDurationMins Int? @map("sla_duration_mins")
```

If a step is not actioned within `slaDurationMins`, the engine auto-escalates to the next role in the hierarchy or sends a high-priority alert.

---

## 12. Module Integration Points

| Module | Integration |
|--------|-------------|
| **Tasks** | WorkflowInstance creates a WORKFLOW Task; task completion triggers step action |
| **HR (Leave)** | LeaveApplication triggers HR approval workflow |
| **Procurement** | PurchaseRequisition triggers PO approval workflow |
| **Notifications** | Engine dispatches Notification records on every step transition |
| **AuditLog** | Every step action is audit-logged |

---

*Document Owner: Engineering Team | Review Cycle: On engine architecture change*
