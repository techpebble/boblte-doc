# Field Work Sessions - Multiple Shifts Per Day / Error Handling

## Problem Context
Currently, when a field executive completes a shift (clocks in, then clocks out) and attempts to clock in again on the same calendar day, the backend throws a raw 500 error or a poorly handled DB constraint error: `A record with that value already exists`. 

This is triggered by the Prisma database constraint in `schema.prisma`:
```prisma
@@unique([fieldExecutiveId, sessionDate])
```
Because the `sessionDate` is mapped to `@db.Date` and the time is truncated, any secondary attempt to clock in on the same date violates this unique constraint.

## Issue 1: Raw Database Error Leakage
The backend is leaking raw Prisma constraints to the frontend (e.g. `PrismaClientKnownRequestError: P2002`).

**Requested Fix:** The `ClockInUseCase` should catch this specific `P2002` error and throw a clean, business-logic `400 Bad Request` or `409 Conflict` (e.g., `Shift already completed for today`).

## Issue 2: Business Logic Clarification (Multiple Shifts)
If the business logic *is supposed* to allow a field executive to have multiple distinct work sessions in a single day (for example, clocking out for an extended period and clocking back in), the current database schema prohibits this.

**Requested Fix:** If multiple shifts per day are required, the backend team must alter the `@@unique` constraint. If only one shift per day is allowed, please implement **Issue 1** to provide clean error handling to the mobile application.
