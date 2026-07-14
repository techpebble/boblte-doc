# Field Executive Management Page - Implementation Instructions

**Version**: 1.0.0
**Last Updated**: 2026-07-14
**Status**: Draft
**Owner**: Product + Engineering
**Depends On**: `19-field-operations-implementation-design.md`, `15-hrm-schema-design.md`, `14-position-model-updates.md`, `10-permission-matrix.md`

---

## 1. Objective

Create a Field Executive Management page where authorized users can:

1. View all field executives.
2. Search and filter field executives.
3. Create a field executive profile for an existing employee.
4. Edit field executive settings.
5. Disable, suspend, or delete a field executive profile.
6. See linked employee, user, position, markets, current assignment, and tracking configuration.

The page should manage the field executive profile only. Employee master data, user login data, and position assignments should remain owned by their existing modules.

---

## 2. Terminology

Use **Field Executive**, not "Filed Executive", in UI labels and code names.

```text
Employee = HR person record
User = login identity
Position = org role/seat
FieldExecutiveProfile = field tracking capability/configuration
Market = sales/operation territory
```

Recommended relationship:

```text
FieldExecutiveProfile -> Employee -> User
Employee/User -> PositionAssignment -> Position -> PositionMarketAssignment -> Market
```

Do not assign permissions directly to the field executive profile.

---

## 3. Navigation

Add the page under Field Operations:

```text
/field/executives
```

Suggested sidebar label:

```text
Field Executives
```

Suggested icon:

```text
MapPinned / Navigation / UserRoundCheck from lucide-react
```

If the Field Operations section is not created yet, place it temporarily under:

```text
/hr/field-executives
```

But the preferred final route is:

```text
app/field/executives/page.tsx
```

---

## 4. Required Permissions

Add or use these permissions:

```text
field.executives.read
field.executives.create
field.executives.update
field.executives.delete
field.executives.manageTracking
field.markets.read
field.tracking.read.team
```

UI behavior:

| Permission | Capability |
|------------|------------|
| `field.executives.read` | View page and list executives |
| `field.executives.create` | Enable employee as field executive |
| `field.executives.update` | Edit profile settings |
| `field.executives.delete` | Delete or deactivate profile |
| `field.executives.manageTracking` | Edit tracking interval and tracking toggle |
| `field.markets.read` | Show assigned markets |
| `field.tracking.read.team` | Show latest location/session summary |

The page should hide actions the user cannot perform.

---

## 5. Data Source and API Contract

### 5.1 List Field Executives

```text
GET /api/v1/field/executives
```

Query params:

```text
search
status
marketId
positionId
trackingEnabled
limit
cursor
```

Response:

```ts
type FieldExecutiveListResponse = {
  items: FieldExecutiveListItem[];
  nextCursor: string | null;
  totalCount: number;
};
```

List item shape:

```ts
type FieldExecutiveListItem = {
  id: string;
  tenantId: string;
  employeeId: string;
  employeeCode: string | null;
  employeeName: string;
  userId: string | null;
  email: string | null;
  mobileNumber: string | null;
  status: 'ACTIVE' | 'INACTIVE' | 'ON_LEAVE' | 'SUSPENDED';
  trackingEnabled: boolean;
  trackingIntervalMin: number;
  activePosition: {
    id: string;
    positionCode: string;
    positionName: string;
    positionType: string;
  } | null;
  markets: {
    id: string;
    marketCode: string;
    marketName: string;
    assignmentType: 'PRIMARY' | 'SECONDARY' | 'TEMPORARY';
    isPrimary: boolean;
  }[];
  todaySession: {
    id: string;
    status: string;
    clockInAt: string | null;
    clockOutAt: string | null;
  } | null;
  lastLocation: {
    capturedAt: string;
    latitude: string;
    longitude: string;
    batteryPercent: number | null;
  } | null;
  updatedAt: string;
};
```

### 5.2 Get Field Executive Details

```text
GET /api/v1/field/executives/:id
```

Details should include:

- Profile fields
- Employee summary
- Linked user summary
- Active position assignments
- Assigned markets inherited from positions
- Temporary market overrides, if enabled
- Today work session
- Recent visits summary
- Recent location ping
- Tracking settings

### 5.3 Create Field Executive Profile

```text
POST /api/v1/field/executives
```

Request:

```ts
type CreateFieldExecutivePayload = {
  employeeId: string;
  status?: 'ACTIVE' | 'INACTIVE' | 'ON_LEAVE' | 'SUSPENDED';
  trackingEnabled?: boolean;
  trackingIntervalMin?: number;
  allowOfflineSync?: boolean;
};
```

Rules:

1. `employeeId` is required.
2. Employee must belong to current tenant.
3. Employee must not already have a field executive profile.
4. Employee may have `userId = null`, but show a warning because mobile login requires a linked user.
5. Creating a field profile should not create a user automatically.

### 5.4 Update Field Executive Profile

```text
PATCH /api/v1/field/executives/:id
```

Request:

```ts
type UpdateFieldExecutivePayload = {
  status?: 'ACTIVE' | 'INACTIVE' | 'ON_LEAVE' | 'SUSPENDED';
  trackingEnabled?: boolean;
  trackingIntervalMin?: number;
  allowOfflineSync?: boolean;
};
```

Rules:

1. Do not allow changing `employeeId`.
2. Clamp `trackingIntervalMin` to an allowed range, for example `1..30`.
3. If status is `SUSPENDED` or `INACTIVE`, frontend should indicate the executive cannot start new work sessions.

### 5.5 Delete or Deactivate Field Executive Profile

Preferred endpoint:

```text
DELETE /api/v1/field/executives/:id
```

Recommended behavior:

- Soft delete profile or set status to `INACTIVE`.
- Do not delete `Employee`.
- Do not delete `User`.
- Do not delete historical work sessions, visits, orders, or GPS pings.

If hard deletion is not safe, expose:

```text
POST /api/v1/field/executives/:id/deactivate
```

and reserve `DELETE` for admin/system use.

### 5.6 Employee Autocomplete

Use an employee autocomplete endpoint:

```text
GET /api/v1/employees/autocomplete?q=&excludeFieldExecutives=true
```

Each option should show:

```text
Employee name
Employee code
Work email / mobile
Current position
```

If this endpoint does not exist yet, create it before building the modal.

---

## 6. Page Layout

Follow the existing HR and org management visual pattern:

- Page header
- Search and filters
- Summary cards
- Table/list area
- Create/edit modal or drawer
- Details drawer for deeper context

### 6.1 Header

Title:

```text
Field Executives
```

Subtitle:

```text
Manage employees enabled for field tracking, visits, attendance, and market operations.
```

Primary action:

```text
Enable Field Executive
```

Use an icon button with text:

```text
Plus + Enable Field Executive
```

### 6.2 Summary Cards

Show compact summary cards:

```text
Total Field Executives
Active
On Leave
Tracking Enabled
No Linked User
```

These can be computed from the first page for MVP, but backend summary is preferred:

```text
GET /api/v1/field/executives/summary
```

### 6.3 Filters

Required filters:

```text
Search
Status
Tracking Enabled
Position
Market
```

Search should match:

- Employee name
- Employee code
- Email
- Mobile number
- Position name
- Market name

Filters should be server-side, not only client-side, once pagination is enabled.

### 6.4 Table Columns

Recommended columns:

| Column | Content |
--------|---------|
| Executive | Name, employee code, avatar/initials |
| Contact | Email, mobile |
| Status | Active, inactive, on leave, suspended |
| Position | Current active position |
| Markets | Primary market and count of additional markets |
| Tracking | Enabled/disabled, interval |
| Today | Clocked in, working, on break, not started |
| Last Location | Last ping time, battery |
| Actions | View, edit, deactivate/delete |

Use a dense operational table, not a marketing-style card grid.

---

## 7. Create/Edit Modal

Use a modal for create/edit.

### 7.1 Create Mode

Fields:

```text
Employee autocomplete
Status
Tracking enabled
Tracking interval minutes
Allow offline sync
```

Read-only preview after selecting employee:

```text
Employee code
Department
Designation
Current position
Linked user
Assigned markets inherited from position
```

Warnings:

```text
No linked user: employee cannot log in to mobile app.
No active position: markets and permissions may not resolve.
No assigned market: executive can track attendance but will not see market/beat plans.
```

### 7.2 Edit Mode

Editable:

```text
Status
Tracking enabled
Tracking interval minutes
Allow offline sync
```

Read-only:

```text
Employee
Linked user
Current position
Assigned markets
```

Employee/user/position changes should happen from Employee Management or Position Management, not this modal.

---

## 8. Details Drawer

When clicking a row, open a details drawer.

Tabs:

```text
Overview
Markets
Attendance
Visits
Settings
```

### 8.1 Overview Tab

Show:

- Employee information
- Linked user
- Current position
- Status
- Today session status
- Last location ping
- Tracking settings

### 8.2 Markets Tab

Show inherited markets:

```text
Position -> PositionMarketAssignment -> Market
```

Columns:

```text
Market
Assignment type
Primary
Effective from
Effective to
Source position
```

If temporary overrides are implemented, show them separately:

```text
Temporary Overrides
```

### 8.3 Attendance Tab

Show recent work sessions:

```text
Date
Clock in
Clock out
Work minutes
Break minutes
Distance
Status
```

### 8.4 Visits Tab

Show recent outlet visits:

```text
Outlet
Market
Check in
Check out
Status
Purpose
```

### 8.5 Settings Tab

Show:

```text
Tracking enabled
Tracking interval
Offline sync
Profile status
```

Use edit actions only if user has update permission.

---

## 9. UX Rules

### 9.1 Empty States

No field executives:

```text
No field executives enabled yet.
Enable employees for field tracking to manage attendance, visits, and market activity.
```

No markets:

```text
No markets assigned through the current position.
Assign markets to the position to show beat plans and market operations.
```

No linked user:

```text
This employee is not linked to a user account.
Mobile app access and permissions will not work until a user is linked.
```

### 9.2 Destructive Actions

For delete/deactivate:

- Require confirmation modal.
- Explain that history will remain.
- Prefer label `Deactivate` over `Delete` if backend soft-deletes.

Confirmation text:

```text
Deactivate field executive profile?
Historical attendance, visits, orders, and location records will remain available.
```

### 9.3 Status Labels

Use these labels:

```text
ACTIVE -> Active
INACTIVE -> Inactive
ON_LEAVE -> On leave
SUSPENDED -> Suspended
```

Do not show raw enum values in the UI.

---

## 10. Frontend File Structure

Recommended files:

```text
boblte-webapp/app/field/executives/page.tsx
boblte-webapp/src/features/field/components/FieldExecutiveManagement.tsx
boblte-webapp/src/features/field/components/FieldExecutiveFormModal.tsx
boblte-webapp/src/features/field/components/FieldExecutiveDetailsDrawer.tsx
boblte-webapp/src/core/api/field.ts
```

If the field feature folder does not exist, create:

```text
boblte-webapp/src/features/field/
```

### 10.1 API Client Types

Add to `src/core/api/field.ts`:

```ts
export type FieldExecutiveStatus =
  | 'ACTIVE'
  | 'INACTIVE'
  | 'ON_LEAVE'
  | 'SUSPENDED';

export type FieldExecutiveListItem = {
  id: string;
  employeeId: string;
  employeeCode: string | null;
  employeeName: string;
  userId: string | null;
  email: string | null;
  mobileNumber: string | null;
  status: FieldExecutiveStatus;
  trackingEnabled: boolean;
  trackingIntervalMin: number;
  activePosition: {
    id: string;
    positionCode: string;
    positionName: string;
    positionType: string;
  } | null;
  markets: {
    id: string;
    marketCode: string;
    marketName: string;
    assignmentType: string;
    isPrimary: boolean;
  }[];
  todaySession: {
    id: string;
    status: string;
    clockInAt: string | null;
    clockOutAt: string | null;
  } | null;
  lastLocation: {
    capturedAt: string;
    latitude: string;
    longitude: string;
    batteryPercent: number | null;
  } | null;
};
```

Methods:

```ts
fieldApi.listFieldExecutives(params)
fieldApi.getFieldExecutive(id)
fieldApi.createFieldExecutive(payload)
fieldApi.updateFieldExecutive(id, payload)
fieldApi.deleteFieldExecutive(id)
fieldApi.getFieldExecutiveSummary()
```

---

## 11. Backend Requirements

The page requires these backend endpoints:

```text
GET    /api/v1/field/executives
GET    /api/v1/field/executives/summary
GET    /api/v1/field/executives/:id
POST   /api/v1/field/executives
PATCH  /api/v1/field/executives/:id
DELETE /api/v1/field/executives/:id
```

Optional but recommended:

```text
GET /api/v1/employees/autocomplete?excludeFieldExecutives=true
GET /api/v1/field/executives/:id/work-sessions
GET /api/v1/field/executives/:id/visits
GET /api/v1/field/executives/:id/markets
```

### 11.1 Backend Validation

Create:

```text
CreateFieldExecutiveDto
UpdateFieldExecutiveDto
ListFieldExecutivesQueryDto
```

Validation:

- `employeeId` must be UUID.
- `trackingIntervalMin` must be integer between 1 and 30.
- `status` must be enum.
- `trackingEnabled` must be boolean.
- Employee must belong to authenticated tenant.
- Profile must be unique per employee.

---

## 12. Data Resolution Rules

### 12.1 Current Position

Resolve active position from:

```text
PositionAssignment
where userId = employee.userId
and effectiveTo is null or effectiveTo > now
```

If multiple positions exist:

1. Prefer `isPrimary = true`.
2. Else choose the latest active assignment.
3. Show additional positions in details if needed.

### 12.2 Assigned Markets

Resolve markets through:

```text
Employee.userId
  -> active PositionAssignment
  -> Position
  -> active PositionMarketAssignment
  -> Market
```

If employee has no linked user, markets cannot be resolved through position assignment unless employee-position assignment is added separately.

### 12.3 Permissions

Resolve permissions through:

```text
User
  -> PositionAssignment
  -> PositionRole
  -> Role
  -> PermissionRole
  -> Permission
```

Field executive profile must not become a separate RBAC source.

---

## 13. Suggested Page States

### Loading

Use spinner with subtle overlay, matching existing app style.

### Error

Show inline error banner:

```text
Unable to load field executives. Please try again.
```

### Saving

Disable modal submit button and show:

```text
Saving...
```

### Success

After create/update/delete:

- Close modal.
- Refresh list.
- Keep filters.
- Show short success message if toast system exists.

---

## 14. MVP Acceptance Criteria

The page is ready when:

1. User with read permission can open the page.
2. User can search and filter field executives.
3. User can create a field executive profile for an existing employee.
4. Duplicate field executive profiles are blocked.
5. User can edit status and tracking settings.
6. User can deactivate/delete a profile without deleting employee/user data.
7. Page shows current position and inherited markets.
8. Page warns when employee has no linked user.
9. Page warns when employee has no market through position.
10. Action buttons obey permissions.
11. List is paginated server-side.
12. Details drawer shows overview, markets, attendance, visits, and settings.

---

## 15. Out of Scope for This Page

Do not implement these inside Field Executive Management:

- Creating employees
- Creating users
- Assigning users to positions
- Assigning roles to positions
- Creating markets
- Assigning markets to positions
- Viewing full live map
- Editing attendance records
- Approving expenses or leave

Instead, link to the relevant modules when needed:

```text
Employee Management
User Management
Position Management
Market Management
Live Tracking Dashboard
```

