# Support Multi-User and Task Assignment Settings on Position Model

Add fields to the `Position` model to control whether a position allows multiple active user assignments concurrently, and if single-user assignable, whether it accepts task assignments.

> [!NOTE]
> This implements controls for `isMultiUserAssignable` and conditionally `isTaskAssignable`. When a position is configured as single-user assignable, we will enforce this behavior during user-position assignment.

## User Review Required

> [!IMPORTANT]
> The database migration will be required after updating the Prisma schema. Does the current `npm run dev` script or your workflow automatically apply Prisma schema changes, or should I include the `npx prisma db push` (or `migrate dev`) in the execution phase?

## Resolved Questions

Based on the feedback provided:
1. `isMultiUserAssignable` will default to `false` for existing positions in the database.
2. If `isMultiUserAssignable` is `true` (multi-user assignable), `isTaskAssignable` will be strictly enforced to `false`. Tasks can only be assigned to single-user positions.
3. The `assign-user-to-position.usecase.ts` will actively enforce the `isMultiUserAssignable` rule by throwing a `ConflictException` if we try to assign a second user to a single-user position.

## Proposed Changes

### Database Schema

#### [MODIFY] [org.prisma](file:///Users/pebble/Workspace/projects/boblte/boblte-server/prisma/schema/org.prisma)
- Add `isMultiUserAssignable Boolean @default(false) @map("is_multi_user_assignable")`
- Add `isTaskAssignable Boolean @default(false) @map("is_task_assignable")`

---

### Domain & Repository Layer

#### [MODIFY] [position.model.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/domain/position.model.ts)
- Add `isMultiUserAssignable: boolean` and `isTaskAssignable: boolean` to the `PositionModel` constructor class properties.

#### [MODIFY] [positions.repository.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/positions.repository.ts)
- Update `toModel` mapper function to map `isMultiUserAssignable` and `isTaskAssignable`.
- Pass `isMultiUserAssignable` and `isTaskAssignable` when creating a position in `create()`.
- Pass these fields when updating a position in `update()`.

---

### DTO Layer

#### [MODIFY] [create-position.dto.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/dto/create-position.dto.ts)
- Add `@ApiPropertyOptional()`, `@IsOptional()`, and `@IsBoolean()` for `isMultiUserAssignable`.
- Add `@ApiPropertyOptional()`, `@IsOptional()`, and `@IsBoolean()` for `isTaskAssignable`.
- Add validation logic (or a custom decorator/transformer) to ensure `isTaskAssignable` is `false` when `isMultiUserAssignable` is `true`.

#### [MODIFY] [update-position.dto.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/dto/update-position.dto.ts)
- Add the same properties (`isMultiUserAssignable`, `isTaskAssignable`) with validation decorators.
- Ensure the rule that `isTaskAssignable` must be `false` if `isMultiUserAssignable` is `true` applies to updates.

---

### Application Logic

#### [MODIFY] [create-position.usecase.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/application/use-cases/create-position.usecase.ts)
- Enforce business logic: if `isMultiUserAssignable` is true, force `isTaskAssignable` to false before saving to the database.

#### [MODIFY] [update-position.usecase.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/application/use-cases/update-position.usecase.ts)
- Enforce business logic during update: if `isMultiUserAssignable` is set to true, force `isTaskAssignable` to false. 

#### [MODIFY] [assign-user-to-position.usecase.ts](file:///Users/pebble/Workspace/projects/boblte/boblte-server/src/modules/org/positions/application/use-cases/assign-user-to-position.usecase.ts)
- Fetch the position using `GetPositionUseCase`.
- If `!position.isMultiUserAssignable`, use `this.repo.countActiveAssignments(positionId)` to verify if it already has an active assignment.
- Throw a `ConflictException` ("This position is configured as single-user assignable and already has an active user.") if another assignment exists.

---

### Tests

#### [MODIFY] Tests
Update mock `PositionModel` instances across multiple test files:
- `list-positions.usecase.spec.ts`
- `get-position.usecase.spec.ts`
- `create-position.usecase.spec.ts`
- `update-position.usecase.spec.ts`
- `assign-user-to-position.usecase.spec.ts`
- `delete-position.usecase.spec.ts`
- `assign-role-to-position.usecase.spec.ts`
- `remove-role-from-position.usecase.spec.ts`
- `remove-user-from-position.usecase.spec.ts`

## Verification Plan

### Automated Tests
- Run `npm test` or `npx jest` to ensure no mock compilation issues or failing tests.

### Manual Verification
- Re-generate Prisma client with `npx prisma generate` and apply the schema (e.g. `npx prisma db push`).
- Ensure the dev server restarts properly.
- Optionally manually test position creation via Postman/Swagger using single-user assignment flags.
