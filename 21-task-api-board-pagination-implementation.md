# Task API Board Pagination - Implementation Design

**Version**: 1.0.0
**Last Updated**: 2026-07-02
**Status**: Draft
**Owner**: Engineering Team
**Depends On**: `03-task-management-prd.md`, `12-task-schema-redesign.md`

---

## 1. Problem

The current task API uses one global cursor-paginated response:

```text
GET /api/v1/tasks?limit=20&cursor=...
```

The default backend limit is 20. The frontend board then splits those 20 tasks into status columns client-side:

```text
TO_DO
IN_PROGRESS
DONE
```

This causes an incomplete first view. If the first 20 tasks are mostly `TO_DO`, the board may show no `IN_PROGRESS` or `DONE` tasks even when they exist on later pages.

Increasing the limit is not a good long-term fix. It only delays the issue and makes the endpoint heavier as task volume grows.

---

## 2. Current Behavior

### Backend

- Controller: `src/modules/ops/tasks/infrastructure/http/tasks.controller.ts`
- Use case: `src/modules/ops/tasks/application/use-cases/list-tasks.usecase.ts`
- Repository: `src/modules/ops/tasks/infrastructure/persistence/tasks.repository.ts`

Current default:

```ts
limit: pagination.limit ?? 20
```

Current repository ordering:

```ts
orderBy: { createdAt: 'asc' }
```

Current response:

```ts
{
  items: TaskListItem[];
  nextCursor: string | null;
}
```

### Frontend

- Page: `boblte-webapp/app/tasks/page.tsx`
- API client: `boblte-webapp/src/core/api/tasks.ts`
- Board: `boblte-webapp/src/features/tasks/components/TaskBoard.tsx`
- List: `boblte-webapp/src/features/tasks/components/TaskList.tsx`

The frontend calls:

```ts
tasksApi.listTasks()
```

Then board/list filtering happens over only the returned page.

---

## 3. Target Design

Use two separate API shapes:

1. **Board API**: grouped, status-aware, first-view optimized
2. **List API**: server-side filtered and sorted table/list API

The board should not depend on one global first page.

---

## 4. Board API

### 4.1 Initial Board Load

Add:

```text
GET /api/v1/tasks/board?columnLimit=20
```

Response:

```ts
type TaskBoardResponse = {
  columns: {
    TO_DO: TaskBoardColumn;
    IN_PROGRESS: TaskBoardColumn;
    DONE: TaskBoardColumn;
    ON_HOLD?: TaskBoardColumn;
    CANCELLED?: TaskBoardColumn;
  };
  summary: {
    total: number;
    overdue: number;
    dueToday: number;
  };
};

type TaskBoardColumn = {
  items: TaskListItem[];
  nextCursor: string | null;
  totalCount: number;
};
```

Each column receives up to `columnLimit` tasks independently.

This means the first board view can show:

```text
20 TO_DO
20 IN_PROGRESS
20 DONE
```

instead of only 20 tasks total.

### 4.2 Load More for One Column

Add:

```text
GET /api/v1/tasks/board-column?status=TO_DO&limit=20&cursor=...
```

Response:

```ts
{
  status: "TO_DO";
  items: TaskListItem[];
  nextCursor: string | null;
  totalCount: number;
}
```

The frontend can add a **Load more** action at the bottom of each column.

---

## 5. Recommended Sorting

### Active Columns

For `TO_DO`, `IN_PROGRESS`, and `ON_HOLD`:

```text
dueDate ASC NULLS LAST
priority DESC
updatedAt DESC
id ASC
```

Rationale:

- Overdue and due-soon tasks should surface first.
- Higher-priority work should appear above lower-priority work.
- Recently touched work should not disappear deep in the list.
- `id` gives deterministic tie-breaking.

### Done / Cancelled Columns

For `DONE` and `CANCELLED`:

```text
updatedAt DESC
id ASC
```

Rationale:

- Recently completed or cancelled tasks are more useful than oldest completed tasks.

---

## 6. Cursor Strategy

The current API uses task `id` as cursor while ordering by `createdAt`. This can become unstable because the cursor field does not match the sort field.

For the board API, use opaque encoded cursors.

Example active-column cursor payload:

```json
{
  "dueDate": "2026-07-02T10:00:00.000Z",
  "priorityRank": 4,
  "updatedAt": "2026-07-01T15:30:00.000Z",
  "id": "task-id"
}
```

Encode as base64url JSON:

```text
eyJkdWVEYXRlIjoiMjAyNi0wNy0wMlQxMDowMDowMC4wMDBaIiwicHJpb3JpdHlSYW5rIjo0LCJpZCI6InRhc2staWQifQ
```

Repository code should decode the cursor and apply a seek condition matching the selected ordering.

If that is too much for the first implementation, use Prisma cursor pagination with this simpler temporary order:

```ts
orderBy: [
  { updatedAt: 'desc' },
  { id: 'asc' },
]
```

Then migrate to richer due-date ordering later.

---

## 7. Backend Implementation Plan

### 7.1 Domain Types

Update:

```text
src/modules/ops/tasks/domain/task.repository.interface.ts
```

Add:

```ts
export type TaskStatusKey =
  | 'TO_DO'
  | 'IN_PROGRESS'
  | 'DONE'
  | 'CANCELLED'
  | 'ON_HOLD';

export type TaskBoardColumnResult = {
  items: TaskListItem[];
  nextCursor: string | null;
  totalCount: number;
};

export type TaskBoardResult = {
  columns: Record<TaskStatusKey, TaskBoardColumnResult>;
  summary: {
    total: number;
    overdue: number;
    dueToday: number;
  };
};
```

Add repository methods:

```ts
findBoard(
  tenantId: string,
  filters: FindAllFilters,
  columnLimit: number,
): Promise<TaskBoardResult>;

findBoardColumn(
  tenantId: string,
  filters: FindAllFilters & { status: TaskStatusKey },
  pagination: PaginationInput,
): Promise<TaskBoardColumnResult>;
```

### 7.2 Use Cases

Add:

```text
src/modules/ops/tasks/application/use-cases/get-task-board.usecase.ts
src/modules/ops/tasks/application/use-cases/get-task-board-column.usecase.ts
```

Behavior:

- Clamp `columnLimit` to a safe range, e.g. `5..50`
- Default `columnLimit = 20`
- Reuse existing visibility filtering:
  - created by current user
  - directly assigned to current user
  - assigned to a position currently held by current user

### 7.3 Controller

Update:

```text
src/modules/ops/tasks/infrastructure/http/tasks.controller.ts
```

Add routes before `@Get(':id')` so static routes do not get captured as task IDs:

```ts
@Get('board')
getBoard(
  @CurrentUser() user: AuthenticatedUser,
  @Query('columnLimit') columnLimit?: string,
) {
  return this.getTaskBoardUC.execute(user.tenantId!, {
    userId: user.userId,
  }, {
    columnLimit: columnLimit ? parseInt(columnLimit, 10) : undefined,
  });
}

@Get('board-column')
getBoardColumn(
  @CurrentUser() user: AuthenticatedUser,
  @Query('status') status: string,
  @Query('limit') limit?: string,
  @Query('cursor') cursor?: string,
) {
  return this.getTaskBoardColumnUC.execute(user.tenantId!, {
    userId: user.userId,
    status,
  }, {
    limit: limit ? parseInt(limit, 10) : undefined,
    cursor: cursor || undefined,
  });
}
```

Important route order:

```text
GET /tasks/board
GET /tasks/board-column
GET /tasks/:id
```

### 7.4 Repository

Extract the current visibility filter into a helper:

```ts
private buildVisibleTaskWhere(
  tenantId: string,
  filters: FindAllFilters,
) {
  const { userId, status, priority } = filters;

  return {
    tenantId,
    deletedAt: null,
    ...(status && { status: status as any }),
    ...(priority && { priority: priority as any }),
    OR: [
      { createdBy: userId },
      {
        assignee: {
          some: {
            assigneeType: 'USER',
            assigneeUserId: userId,
          },
        },
      },
      {
        assignee: {
          some: {
            assigneeType: 'POSITION',
            assigneePosition: {
              isTaskAssignable: true,
              positionAssignments: {
                some: {
                  userId,
                  OR: [
                    { effectiveTo: null },
                    { effectiveTo: { gt: new Date() } },
                  ],
                },
              },
            },
          },
        },
      },
    ],
  };
}
```

`findBoard` should query each status independently. A simple first implementation can run these queries in parallel:

```ts
const statuses = ['TO_DO', 'IN_PROGRESS', 'DONE'] as const;

const columns = await Promise.all(
  statuses.map((status) =>
    this.findBoardColumn(
      tenantId,
      { ...filters, status },
      { limit: columnLimit },
    ),
  ),
);
```

Also run summary counts:

```ts
const total = await this.db.client.task.count({ where: baseWhere });
const overdue = await this.db.client.task.count({
  where: {
    ...baseWhere,
    status: { not: 'DONE' },
    dueDate: { lt: new Date() },
  },
});
```

### 7.5 Mapping

Reuse the current `TaskListItem` mapper. Do not include heavy details such as comments, full attachments, activity log, or all checklist details on the board endpoint unless the UI needs them.

For board cards, a lighter response is preferable:

```ts
type TaskBoardCard = {
  id: string;
  title: string;
  description: string | null;
  status: string;
  priority: string;
  dueDate: Date | null;
  version: number;
  createdBy: UserIdentity | null;
  assignees: TaskAssignee[];
  labelCount: number;
  checklistCompleted: number;
  checklistTotal: number;
  attachmentCount: number;
  commentCount: number;
};
```

The full task detail drawer should continue to use:

```text
GET /api/v1/tasks/:id
```

---

## 8. List API Improvements

Keep:

```text
GET /api/v1/tasks
```

But use it for the list/table view, not the board.

Enhance supported query params:

```text
GET /api/v1/tasks
  ?status=TO_DO
  &priority=HIGH
  &search=invoice
  &sortBy=dueDate
  &sortOrder=asc
  &limit=50
  &cursor=...
```

Move filtering and sorting server-side. The current list view filters and sorts only the tasks already loaded on the client, which can show false empty results.

---

## 9. Frontend Implementation Plan

### 9.1 API Client

Update:

```text
boblte-webapp/src/core/api/tasks.ts
```

Add:

```ts
getTaskBoard: (params?: { columnLimit?: number }) => {
  const qs = new URLSearchParams();
  if (params?.columnLimit) qs.append('columnLimit', String(params.columnLimit));
  const queryString = qs.toString() ? `?${qs.toString()}` : '';
  return apiClient.get<{ success: boolean; data: TaskBoardResponse }>(
    `/api/v1/tasks/board${queryString}`,
  );
},

getTaskBoardColumn: (
  status: TaskStatus,
  params?: { limit?: number; cursor?: string },
) => {
  const qs = new URLSearchParams({ status });
  if (params?.limit) qs.append('limit', String(params.limit));
  if (params?.cursor) qs.append('cursor', params.cursor);
  return apiClient.get<{ success: boolean; data: TaskBoardColumnResponse }>(
    `/api/v1/tasks/board-column?${qs.toString()}`,
  );
},
```

### 9.2 Tasks Page

Update:

```text
boblte-webapp/app/tasks/page.tsx
```

Use different fetches by view mode:

- Board view calls `getTaskBoard`
- List view calls `listTasks`

Keep board state grouped:

```ts
type BoardState = {
  columns: Record<TaskStatus, {
    items: Task[];
    nextCursor?: string | null;
    totalCount: number;
  }>;
  summary: {
    total: number;
    overdue: number;
    dueToday: number;
  };
};
```

### 9.3 Task Board

Update:

```text
boblte-webapp/src/features/tasks/components/TaskBoard.tsx
```

Change props from:

```ts
tasks: Task[];
```

to:

```ts
columns: Record<TaskStatus, TaskBoardColumn>;
onLoadMoreColumn: (status: TaskStatus, cursor: string) => void;
```

Each column renders its own server-provided tasks.

Show:

```text
20 of 63
```

instead of only:

```text
20
```

Add a load-more button when `nextCursor` exists.

---

## 10. Database Indexes

Update:

```text
boblte-server/prisma/schema/tasks.prisma
```

Recommended indexes:

```prisma
model Task {
  // existing fields...

  @@index([tenantId])
  @@index([tenantId, status])
  @@index([tenantId, dueDate])
  @@index([tenantId, status, dueDate])
  @@index([tenantId, status, updatedAt])
  @@index([parentTaskId, status])
  @@map("tasks")
}

model TaskAssignment {
  // existing fields...

  @@index([taskId])
  @@index([assigneeUserId, taskId])
  @@index([assigneePositionId, taskId])
  @@map("task_assignments")
}
```

If search becomes important, add PostgreSQL full-text search or trigram indexes later. Do not solve search by loading all tasks into the browser.

---

## 11. Backward Compatibility

Keep existing endpoint:

```text
GET /api/v1/tasks
```

Do not break current consumers.

Add new endpoints:

```text
GET /api/v1/tasks/board
GET /api/v1/tasks/board-column
```

Frontend can migrate board first. List view can be migrated after.

---

## 12. Testing Plan

### Unit Tests

Add tests for:

- `GetTaskBoardUseCase`
- `GetTaskBoardColumnUseCase`
- Default `columnLimit = 20`
- Maximum limit clamping
- Invalid status rejected
- Per-column `nextCursor`
- Per-column `totalCount`

### Repository Tests

Seed:

- 30 `TO_DO`
- 25 `IN_PROGRESS`
- 10 `DONE`

Expected board response with `columnLimit = 20`:

```text
TO_DO: 20 items, nextCursor present, totalCount 30
IN_PROGRESS: 20 items, nextCursor present, totalCount 25
DONE: 10 items, nextCursor null, totalCount 10
```

### Frontend Tests / Manual QA

Verify:

1. Board initial view shows tasks in every populated status column.
2. Column counts show total count, not only loaded count.
3. Load more affects only that column.
4. Dragging a task to another column updates the card and refreshes board metadata.
5. List view does not claim "No tasks found" when matching tasks exist on later pages.

---

## 13. Rollout Plan

1. Add backend board endpoints while keeping `GET /tasks`.
2. Add repository indexes.
3. Add API client methods.
4. Update board UI to consume grouped response.
5. Add per-column load more.
6. Move list filters/sort server-side.
7. Optionally reduce payload size for board cards.

---

## 14. Acceptance Criteria

The implementation is complete when:

1. Board initial load is grouped by status.
2. Default board load returns up to 20 tasks per column.
3. Board shows total count per column.
4. User can load more tasks for one column without reloading all columns.
5. No status column appears empty just because another column filled the global first page.
6. List filters and sort operate on server-side result sets.
7. Task detail drawer still loads full task details from `GET /tasks/:id`.
8. Existing `GET /tasks` consumers continue working.

