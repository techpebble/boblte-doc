# Field Operations Module - Implementation Design

**Version**: 1.0.0
**Last Updated**: 2026-06-29
**Status**: Draft
**Owner**: Engineering Team
**Depends On**: `04-field-executive-tracking-prd.md`, `05-org-and-access-control-design.md`, `14-position-model-updates.md`, `15-hrm-schema-design.md`, `16-scalable-notification-engine.md`

---

## 1. Overview

This document describes the implementation design for the Field Operations module. It expands the original Field Executive Tracking PRD into a broader Sales Force Automation module covering:

1. Market-wise field executive assignment through positions
2. Attendance and work sessions
3. Break tracking
4. GPS location tracking and route history
5. Beat planning and outlet visits
6. Sales orders
7. Expense and TA claims
8. Leave applications
9. Dynamic survey builder
10. Manager dashboards and reporting foundations

The design intentionally reuses existing ERP modules instead of duplicating core data:

| Existing Module | Reused For |
|-----------------|------------|
| Platform/Tenant | Tenant ownership |
| IAM/User | Login identity, permissions, approvers |
| HR/Employee | Employee master data |
| Org/Position | Market assignment and reporting hierarchy |
| StorageObject | Selfies, receipts, outlet photos, signatures |
| Workflow | Approvals for expenses, orders, leave, and exceptions |
| Notifications | Reminders, approvals, missed visits, late clock-in |
| Tasks | Field tasks and follow-ups |

---

## 2. Design Principles

### 2.1 Assign Markets to Positions, Not Directly to Users

Field market ownership should be stable at the position level.

```text
Market
  -> PositionMarketAssignment
    -> Position
      -> PositionAssignment
        -> User / Employee
```

If an employee leaves and another employee is assigned to the same position, the market remains assigned to the position and automatically becomes visible to the new holder.

### 2.2 One Work Session per Field Executive per Day

Each working day creates one `FieldWorkSession`.

```text
Clock In
  -> Working
  -> Break
  -> Resume
  -> Outlet Visit
  -> Break
  -> Outlet Visit
  -> Clock Out
```

All field activity for the day links back to this session where possible.

### 2.3 Keep Operational Data Modular

Sales orders, expense claims, leave applications, and survey forms are first-class submodules because they have independent lifecycles, approvals, and reports.

### 2.4 Use `StorageObject` for All Files

The field module should not store file metadata directly. Selfies, receipts, visit photos, signatures, videos, and uploaded survey files should all reference `StorageObject`.

### 2.5 Use Manual SQL for Partial Constraints

Some important constraints need PostgreSQL partial indexes or check constraints. Prisma cannot model all of them directly, so migrations should include manual SQL where needed.

---

## 3. Module Boundaries

### 3.1 In Scope

- Market hierarchy
- Market assignment to positions
- Field executive profile
- Work session attendance
- Breaks
- Location pings
- Beat plans
- Outlets
- Outlet visits
- Sales orders and items
- Expense and TA claims
- Leave applications
- Dynamic survey forms and submissions
- Daily summaries for dashboard/reporting

### 3.2 Out of Scope for First Implementation

- Full inventory valuation
- Final invoice posting
- General ledger posting
- Payroll settlement
- Route optimization engine
- AI-based beat planning
- Native mobile offline database schema

These can integrate later through IDs already included in the schema.

---

## 4. High-Level User Flows

### 4.1 Admin Setup Flow

1. Tenant admin creates markets.
2. Admin creates outlets inside markets.
3. Admin assigns markets to positions.
4. HR/admin assigns employees/users to positions.
5. Admin enables field profile for selected employees.
6. Admin configures tracking frequency, survey forms, expense rules, and approval workflows.

### 4.2 Field Executive Daily Flow

1. Field executive logs in.
2. App resolves markets from the user's active positions.
3. User starts day with GPS and optional selfie.
4. System creates `FieldWorkSession`.
5. App starts periodic `FieldLocationPing` sync.
6. User sees beat plan and outlet list.
7. User checks into outlets and completes visits.
8. User creates orders, survey submissions, collections later, or expenses.
9. User ends day.
10. System updates `FieldDailySummary`.

### 4.3 Manager Monitoring Flow

1. Manager opens dashboard.
2. Dashboard shows team attendance, live map, and market coverage.
3. Manager drills into a market.
4. System shows assigned positions, current users, route, visits, orders, expenses, and missed outlets.
5. Manager approves or rejects expense claims, leave applications, and exceptions.

---

## 5. Database Schema Placement

Create a new Prisma schema file:

```text
boblte-server/prisma/schema/field.prisma
```

Update existing schema files with relation arrays:

| File | Relation Updates |
|------|------------------|
| `platform.prisma` | Add field module arrays to `Tenant` |
| `auth.prisma` | Add relations from `User` to field sessions, approvals, profiles where needed |
| `hr.prisma` | Add relations from `Employee` to field profile, sessions, leave applications |
| `org.prisma` | Add `Position.marketAssignments`, `Position.beatPlans`, `Position.workSessions` |
| `storage.prisma` | Add relation names for selfies, receipts, photos, signatures |

---

## 6. Enums

```prisma
enum MarketType {
  REGION
  ZONE
  STATE
  DISTRICT
  CITY
  TOWN
  AREA
  TERRITORY
  CUSTOM

  @@map("market_type")
}

enum MarketAssignmentType {
  PRIMARY
  SECONDARY
  TEMPORARY

  @@map("market_assignment_type")
}

enum FieldExecutiveStatus {
  ACTIVE
  INACTIVE
  ON_LEAVE
  SUSPENDED

  @@map("field_executive_status")
}

enum WorkSessionStatus {
  CLOCKED_IN
  WORKING
  ON_BREAK
  CLOCKED_OUT
  AUTO_CLOSED
  APPROVAL_PENDING

  @@map("work_session_status")
}

enum BreakType {
  TEA
  LUNCH
  PERSONAL
  FUEL
  VEHICLE_ISSUE
  OFFICIAL_MEETING
  OTHER

  @@map("break_type")
}

enum BeatPlanStatus {
  DRAFT
  PUBLISHED
  IN_PROGRESS
  COMPLETED
  CANCELLED

  @@map("beat_plan_status")
}

enum VisitStatus {
  PLANNED
  CHECKED_IN
  COMPLETED
  CANCELLED
  MISSED

  @@map("visit_status")
}

enum FieldOrderStatus {
  DRAFT
  SUBMITTED
  APPROVED
  REJECTED
  CANCELLED
  CONVERTED

  @@map("field_order_status")
}

enum ExpenseClaimStatus {
  DRAFT
  SUBMITTED
  APPROVED
  REJECTED
  SETTLED
  CANCELLED

  @@map("expense_claim_status")
}

enum LeaveApplicationStatus {
  PENDING
  APPROVED
  REJECTED
  CANCELLED

  @@map("leave_application_status")
}

enum SurveyFieldType {
  TEXT
  NUMBER
  DROPDOWN
  RADIO
  CHECKBOX
  PHOTO
  VIDEO
  GPS
  SIGNATURE
  BARCODE
  QR
  DATE
  TIME

  @@map("survey_field_type")
}
```

---

## 7. Core Market and Field Tracking Schema

### 7.1 `Market`

```prisma
model Market {
  id               String     @id @default(uuid())
  tenantId         String     @map("tenant_id")
  marketCode       String     @map("market_code")
  marketName       String     @map("market_name")
  marketType       MarketType @default(AREA) @map("market_type")
  parentMarketId   String?    @map("parent_market_id")
  city             String?
  district         String?
  state            String?
  country          String?
  geoBoundary      Json?      @map("geo_boundary")
  customAttributes Json       @default("{}") @map("custom_attributes")
  isActive         Boolean    @default(true) @map("is_active")
  createdAt        DateTime   @default(now()) @map("created_at")
  createdBy        String?    @map("created_by")
  updatedAt        DateTime   @updatedAt @map("updated_at")
  updatedBy        String?    @map("updated_by")
  deletedAt        DateTime?  @map("deleted_at")
  deletedBy        String?    @map("deleted_by")

  tenant       Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  parentMarket Market?  @relation("MarketHierarchy", fields: [parentMarketId], references: [id])
  childMarkets Market[] @relation("MarketHierarchy")

  positionAssignments PositionMarketAssignment[]
  outlets             FieldOutlet[]
  beatPlans           BeatPlan[]
  visits              OutletVisit[]
  locationPings       FieldLocationPing[]
  salesOrders         FieldSalesOrder[]

  @@unique([tenantId, marketCode])
  @@unique([tenantId, marketName])
  @@index([tenantId])
  @@index([parentMarketId])
  @@map("markets")
}
```

### 7.2 `PositionMarketAssignment`

```prisma
model PositionMarketAssignment {
  id             String               @id @default(uuid())
  tenantId       String               @map("tenant_id")
  positionId     String               @map("position_id")
  marketId       String               @map("market_id")
  assignmentType MarketAssignmentType @default(PRIMARY) @map("assignment_type")
  isPrimary      Boolean              @default(false) @map("is_primary")
  effectiveFrom  DateTime             @map("effective_from")
  effectiveTo    DateTime?            @map("effective_to")
  createdAt      DateTime             @default(now()) @map("created_at")
  createdBy      String?              @map("created_by")

  tenant   Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  position Position @relation(fields: [positionId], references: [id], onDelete: Cascade)
  market   Market   @relation(fields: [marketId], references: [id], onDelete: Cascade)

  @@index([tenantId])
  @@index([positionId])
  @@index([marketId])
  @@index([positionId, effectiveTo])
  @@index([marketId, effectiveTo])
  @@map("position_market_assignments")
}
```

### 7.3 `FieldExecutiveProfile`

```prisma
model FieldExecutiveProfile {
  id                  String               @id @default(uuid())
  tenantId            String               @map("tenant_id")
  employeeId          String               @unique @map("employee_id")
  userId              String?              @map("user_id")
  status              FieldExecutiveStatus @default(ACTIVE)
  trackingEnabled     Boolean              @default(true) @map("tracking_enabled")
  trackingIntervalMin Int                  @default(5) @map("tracking_interval_min")
  allowOfflineSync    Boolean              @default(true) @map("allow_offline_sync")
  customAttributes    Json                 @default("{}") @map("custom_attributes")
  createdAt           DateTime             @default(now()) @map("created_at")
  createdBy           String?              @map("created_by")
  updatedAt           DateTime             @updatedAt @map("updated_at")
  updatedBy           String?              @map("updated_by")
  deletedAt           DateTime?            @map("deleted_at")
  deletedBy           String?              @map("deleted_by")

  tenant   Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  employee Employee @relation(fields: [employeeId], references: [id])
  user     User?    @relation(fields: [userId], references: [id])

  workSessions     FieldWorkSession[]
  locationPings    FieldLocationPing[]
  visits           OutletVisit[]
  dailySummaries   FieldDailySummary[]
  salesOrders      FieldSalesOrder[]
  expenseClaims    FieldExpenseClaim[]
  leaveApplications FieldLeaveApplication[]
  surveySubmissions SurveySubmission[]

  @@index([tenantId])
  @@index([employeeId])
  @@index([userId])
  @@index([tenantId, status])
  @@map("field_executive_profiles")
}
```

### 7.4 `FieldWorkSession`

```prisma
model FieldWorkSession {
  id                String            @id @default(uuid())
  tenantId          String            @map("tenant_id")
  fieldExecutiveId  String            @map("field_executive_id")
  userId            String            @map("user_id")
  employeeId        String            @map("employee_id")
  positionId        String?           @map("position_id")
  sessionDate       DateTime          @map("session_date") @db.Date
  status            WorkSessionStatus @default(CLOCKED_IN)
  clockInAt         DateTime          @map("clock_in_at")
  clockOutAt        DateTime?         @map("clock_out_at")
  clockInLat        Decimal?          @map("clock_in_lat") @db.Decimal(9, 6)
  clockInLng        Decimal?          @map("clock_in_lng") @db.Decimal(9, 6)
  clockOutLat       Decimal?          @map("clock_out_lat") @db.Decimal(9, 6)
  clockOutLng       Decimal?          @map("clock_out_lng") @db.Decimal(9, 6)
  clockInSelfieId   String?           @map("clock_in_selfie_id")
  clockOutSelfieId  String?           @map("clock_out_selfie_id")
  totalBreakMinutes Int               @default(0) @map("total_break_minutes")
  totalWorkMinutes  Int               @default(0) @map("total_work_minutes")
  distanceMeters    Int?              @map("distance_meters")
  createdAt         DateTime          @default(now()) @map("created_at")
  updatedAt         DateTime          @updatedAt @map("updated_at")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id], onDelete: Cascade)
  user           User                  @relation(fields: [userId], references: [id])
  employee       Employee              @relation(fields: [employeeId], references: [id])
  position       Position?             @relation(fields: [positionId], references: [id])
  clockInSelfie  StorageObject?        @relation("ClockInSelfie", fields: [clockInSelfieId], references: [id])
  clockOutSelfie StorageObject?        @relation("ClockOutSelfie", fields: [clockOutSelfieId], references: [id])

  breaks        FieldWorkBreak[]
  visits        OutletVisit[]
  expenseClaims FieldExpenseClaim[]

  @@unique([fieldExecutiveId, sessionDate])
  @@index([tenantId, sessionDate])
  @@index([userId, sessionDate])
  @@map("field_work_sessions")
}
```

### 7.5 `FieldWorkBreak`

```prisma
model FieldWorkBreak {
  id            String    @id @default(uuid())
  tenantId      String    @map("tenant_id")
  workSessionId String    @map("work_session_id")
  breakType     BreakType @map("break_type")
  startedAt     DateTime  @map("started_at")
  endedAt       DateTime? @map("ended_at")
  startLat      Decimal?  @map("start_lat") @db.Decimal(9, 6)
  startLng      Decimal?  @map("start_lng") @db.Decimal(9, 6)
  endLat        Decimal?  @map("end_lat") @db.Decimal(9, 6)
  endLng        Decimal?  @map("end_lng") @db.Decimal(9, 6)
  reason        String?
  photoId       String?   @map("photo_id")
  approvedBy    String?   @map("approved_by")
  approvedAt    DateTime? @map("approved_at")

  tenant      Tenant           @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  workSession FieldWorkSession @relation(fields: [workSessionId], references: [id], onDelete: Cascade)
  photo       StorageObject?   @relation(fields: [photoId], references: [id])

  @@index([tenantId])
  @@index([workSessionId])
  @@map("field_work_breaks")
}
```

### 7.6 `FieldLocationPing`

```prisma
model FieldLocationPing {
  id               String   @id @default(uuid())
  tenantId         String   @map("tenant_id")
  fieldExecutiveId String   @map("field_executive_id")
  marketId         String?  @map("market_id")
  capturedAt       DateTime @map("captured_at")
  latitude         Decimal  @db.Decimal(9, 6)
  longitude        Decimal  @db.Decimal(9, 6)
  accuracyMeters   Int?     @map("accuracy_meters")
  speedKmph        Decimal? @map("speed_kmph") @db.Decimal(6, 2)
  batteryPercent   Int?     @map("battery_percent")
  isMockLocation   Boolean  @default(false) @map("is_mock_location")
  isOfflineSync    Boolean  @default(false) @map("is_offline_sync")
  metadata         Json     @default("{}")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id], onDelete: Cascade)
  market         Market?               @relation(fields: [marketId], references: [id])

  @@index([tenantId])
  @@index([fieldExecutiveId, capturedAt])
  @@index([marketId, capturedAt])
  @@map("field_location_pings")
}
```

---

## 8. Beat, Outlet, and Visit Schema

```prisma
model FieldOutlet {
  id               String    @id @default(uuid())
  tenantId         String    @map("tenant_id")
  marketId         String    @map("market_id")
  outletCode       String    @map("outlet_code")
  outletName       String    @map("outlet_name")
  ownerName        String?   @map("owner_name")
  mobileNumber     String?   @map("mobile_number")
  gstNumber        String?   @map("gst_number")
  licenseNumber    String?   @map("license_number")
  address          String?
  latitude         Decimal?  @db.Decimal(9, 6)
  longitude        Decimal?  @db.Decimal(9, 6)
  creditLimit      Decimal?  @map("credit_limit") @db.Decimal(12, 2)
  customAttributes Json      @default("{}") @map("custom_attributes")
  isActive         Boolean   @default(true) @map("is_active")
  createdAt        DateTime  @default(now()) @map("created_at")
  createdBy        String?   @map("created_by")
  updatedAt        DateTime  @updatedAt @map("updated_at")
  updatedBy        String?   @map("updated_by")
  deletedAt        DateTime? @map("deleted_at")
  deletedBy        String?   @map("deleted_by")

  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  market Market @relation(fields: [marketId], references: [id])

  beatStops   BeatPlanStop[]
  visits      OutletVisit[]
  salesOrders FieldSalesOrder[]

  @@unique([tenantId, outletCode])
  @@index([tenantId])
  @@index([marketId])
  @@map("field_outlets")
}

model BeatPlan {
  id         String         @id @default(uuid())
  tenantId   String         @map("tenant_id")
  marketId   String         @map("market_id")
  positionId String         @map("position_id")
  planDate   DateTime       @map("plan_date") @db.Date
  status     BeatPlanStatus @default(DRAFT)
  createdAt  DateTime       @default(now()) @map("created_at")
  createdBy  String?        @map("created_by")
  updatedAt  DateTime       @updatedAt @map("updated_at")
  updatedBy  String?        @map("updated_by")

  tenant   Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  market   Market   @relation(fields: [marketId], references: [id])
  position Position @relation(fields: [positionId], references: [id])

  stops BeatPlanStop[]

  @@unique([positionId, planDate])
  @@index([tenantId, planDate])
  @@index([marketId, planDate])
  @@map("beat_plans")
}

model BeatPlanStop {
  id         String    @id @default(uuid())
  beatPlanId String    @map("beat_plan_id")
  outletId   String    @map("outlet_id")
  sortOrder  Int       @default(0) @map("sort_order")
  plannedAt  DateTime? @map("planned_at")
  isRequired Boolean   @default(true) @map("is_required")

  beatPlan BeatPlan    @relation(fields: [beatPlanId], references: [id], onDelete: Cascade)
  outlet   FieldOutlet @relation(fields: [outletId], references: [id])

  visits OutletVisit[]

  @@unique([beatPlanId, outletId])
  @@index([outletId])
  @@map("beat_plan_stops")
}

model OutletVisit {
  id                  String      @id @default(uuid())
  tenantId            String      @map("tenant_id")
  workSessionId       String?     @map("work_session_id")
  beatPlanStopId      String?     @map("beat_plan_stop_id")
  fieldExecutiveId    String      @map("field_executive_id")
  marketId            String      @map("market_id")
  outletId            String      @map("outlet_id")
  status              VisitStatus @default(PLANNED)
  checkInAt           DateTime?   @map("check_in_at")
  checkOutAt          DateTime?   @map("check_out_at")
  checkInLat          Decimal?    @map("check_in_lat") @db.Decimal(9, 6)
  checkInLng          Decimal?    @map("check_in_lng") @db.Decimal(9, 6)
  checkOutLat         Decimal?    @map("check_out_lat") @db.Decimal(9, 6)
  checkOutLng         Decimal?    @map("check_out_lng") @db.Decimal(9, 6)
  purpose             String?
  remarks             String?
  nextFollowUpAt      DateTime?   @map("next_follow_up_at")
  customerSignatureId String?     @map("customer_signature_id")
  customAttributes    Json        @default("{}") @map("custom_attributes")
  createdAt           DateTime    @default(now()) @map("created_at")
  updatedAt           DateTime    @updatedAt @map("updated_at")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  workSession    FieldWorkSession?     @relation(fields: [workSessionId], references: [id])
  beatPlanStop   BeatPlanStop?         @relation(fields: [beatPlanStopId], references: [id])
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id])
  market         Market                @relation(fields: [marketId], references: [id])
  outlet         FieldOutlet           @relation(fields: [outletId], references: [id])
  signature      StorageObject?        @relation(fields: [customerSignatureId], references: [id])

  media             OutletVisitMedia[]
  salesOrders       FieldSalesOrder[]
  surveySubmissions SurveySubmission[]

  @@index([tenantId])
  @@index([fieldExecutiveId, checkInAt])
  @@index([marketId, checkInAt])
  @@index([outletId, checkInAt])
  @@map("outlet_visits")
}

model OutletVisitMedia {
  id              String   @id @default(uuid())
  tenantId        String   @map("tenant_id")
  outletVisitId   String   @map("outlet_visit_id")
  storageObjectId String   @map("storage_object_id")
  mediaType       String   @map("media_type")
  caption         String?
  createdAt       DateTime @default(now()) @map("created_at")

  tenant        Tenant        @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  outletVisit   OutletVisit   @relation(fields: [outletVisitId], references: [id], onDelete: Cascade)
  storageObject StorageObject @relation(fields: [storageObjectId], references: [id])

  @@index([outletVisitId])
  @@map("outlet_visit_media")
}
```

---

## 9. Sales Order Schema

Field sales orders are captured in the field module first. Later, approved orders can be converted into the ERP Sales module or invoice pipeline.

```prisma
model FieldSalesOrder {
  id                  String           @id @default(uuid())
  tenantId            String           @map("tenant_id")
  orderNumber         String           @map("order_number")
  fieldExecutiveId    String           @map("field_executive_id")
  marketId            String           @map("market_id")
  outletId            String           @map("outlet_id")
  outletVisitId       String?          @map("outlet_visit_id")
  orderDate           DateTime         @default(now()) @map("order_date")
  status              FieldOrderStatus @default(DRAFT)
  subtotalAmount      Decimal          @default(0) @map("subtotal_amount") @db.Decimal(12, 2)
  discountAmount      Decimal          @default(0) @map("discount_amount") @db.Decimal(12, 2)
  taxAmount           Decimal          @default(0) @map("tax_amount") @db.Decimal(12, 2)
  totalAmount         Decimal          @default(0) @map("total_amount") @db.Decimal(12, 2)
  remarks             String?
  customerSignatureId String?          @map("customer_signature_id")
  isOfflineSync       Boolean          @default(false) @map("is_offline_sync")
  createdAt           DateTime         @default(now()) @map("created_at")
  submittedAt         DateTime?        @map("submitted_at")
  approvedAt          DateTime?        @map("approved_at")
  approvedBy          String?          @map("approved_by")
  rejectedReason      String?          @map("rejected_reason")

  tenant            Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fieldExecutive    FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id])
  market            Market                @relation(fields: [marketId], references: [id])
  outlet            FieldOutlet           @relation(fields: [outletId], references: [id])
  outletVisit       OutletVisit?          @relation(fields: [outletVisitId], references: [id])
  customerSignature StorageObject?        @relation(fields: [customerSignatureId], references: [id])
  approvedByUser    User?                 @relation("FieldSalesOrderApprovedBy", fields: [approvedBy], references: [id])

  items FieldSalesOrderItem[]

  @@unique([tenantId, orderNumber])
  @@index([tenantId, orderDate])
  @@index([fieldExecutiveId, orderDate])
  @@index([outletId, orderDate])
  @@map("field_sales_orders")
}

model FieldSalesOrderItem {
  id             String  @id @default(uuid())
  salesOrderId   String  @map("sales_order_id")
  productId      String? @map("product_id")
  skuCode        String? @map("sku_code")
  productName    String  @map("product_name")
  quantity       Decimal @db.Decimal(12, 3)
  unitPrice      Decimal @map("unit_price") @db.Decimal(12, 2)
  discountAmount Decimal @default(0) @map("discount_amount") @db.Decimal(12, 2)
  taxAmount      Decimal @default(0) @map("tax_amount") @db.Decimal(12, 2)
  lineTotal      Decimal @map("line_total") @db.Decimal(12, 2)

  salesOrder FieldSalesOrder @relation(fields: [salesOrderId], references: [id], onDelete: Cascade)

  @@index([salesOrderId])
  @@map("field_sales_order_items")
}
```

Sales order flow:

```text
Outlet visit
  -> Create draft order
  -> Add products/items
  -> Calculate totals
  -> Capture optional signature
  -> Submit
  -> Approval workflow
  -> Convert to ERP sales order/invoice later
```

---

## 10. Expense and TA Claim Schema

Expense claims can be manually entered or generated from GPS-derived distance and configured TA rules.

```prisma
model FieldExpenseClaim {
  id               String             @id @default(uuid())
  tenantId         String             @map("tenant_id")
  claimNumber      String             @map("claim_number")
  fieldExecutiveId String             @map("field_executive_id")
  workSessionId    String?            @map("work_session_id")
  claimDate        DateTime           @map("claim_date") @db.Date
  distanceMeters   Int?               @map("distance_meters")
  totalAmount      Decimal            @default(0) @map("total_amount") @db.Decimal(12, 2)
  status           ExpenseClaimStatus @default(DRAFT)
  submittedAt      DateTime?          @map("submitted_at")
  approvedAt       DateTime?          @map("approved_at")
  approvedBy       String?            @map("approved_by")
  rejectedReason   String?            @map("rejected_reason")
  settledAt        DateTime?          @map("settled_at")
  createdAt        DateTime           @default(now()) @map("created_at")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id])
  workSession    FieldWorkSession?     @relation(fields: [workSessionId], references: [id])
  approvedByUser User?                 @relation("FieldExpenseApprovedBy", fields: [approvedBy], references: [id])

  items FieldExpenseClaimItem[]

  @@unique([tenantId, claimNumber])
  @@index([tenantId, claimDate])
  @@index([fieldExecutiveId, claimDate])
  @@map("field_expense_claims")
}

model FieldExpenseClaimItem {
  id             String  @id @default(uuid())
  expenseClaimId String  @map("expense_claim_id")
  expenseType    String  @map("expense_type")
  amount         Decimal @db.Decimal(12, 2)
  receiptId      String? @map("receipt_id")
  remarks        String?

  expenseClaim FieldExpenseClaim @relation(fields: [expenseClaimId], references: [id], onDelete: Cascade)
  receipt      StorageObject?    @relation(fields: [receiptId], references: [id])

  @@index([expenseClaimId])
  @@map("field_expense_claim_items")
}

model FieldTaRule {
  id               String    @id @default(uuid())
  tenantId         String    @map("tenant_id")
  ruleName         String    @map("rule_name")
  vehicleType      String?   @map("vehicle_type")
  ratePerKm        Decimal?  @map("rate_per_km") @db.Decimal(10, 2)
  dailyAllowance   Decimal?  @map("daily_allowance") @db.Decimal(10, 2)
  effectiveFrom    DateTime  @map("effective_from") @db.Date
  effectiveTo      DateTime? @map("effective_to") @db.Date
  isActive         Boolean   @default(true) @map("is_active")
  customAttributes Json      @default("{}") @map("custom_attributes")

  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@index([tenantId])
  @@index([tenantId, isActive])
  @@map("field_ta_rules")
}
```

Expense and TA flow:

```text
End day
  -> Calculate distance from pings
  -> Apply TA rule
  -> User adds receipts/extra expenses
  -> Submit claim
  -> Manager approval
  -> Finance settlement later
```

---

## 11. Leave Application Schema

This is a field-aware leave model. If a general HR leave module is implemented later, this can be migrated or integrated as a specialized source.

```prisma
model FieldLeaveApplication {
  id               String                 @id @default(uuid())
  tenantId         String                 @map("tenant_id")
  fieldExecutiveId String                 @map("field_executive_id")
  employeeId       String                 @map("employee_id")
  leaveType        String                 @map("leave_type")
  fromDate         DateTime               @map("from_date") @db.Date
  toDate           DateTime               @map("to_date") @db.Date
  isHalfDay        Boolean                @default(false) @map("is_half_day")
  reason           String?
  status           LeaveApplicationStatus @default(PENDING)
  appliedAt        DateTime               @default(now()) @map("applied_at")
  approvedAt       DateTime?              @map("approved_at")
  approvedBy       String?                @map("approved_by")
  rejectedReason   String?                @map("rejected_reason")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id])
  employee       Employee              @relation(fields: [employeeId], references: [id])
  approvedByUser User?                 @relation("FieldLeaveApprovedBy", fields: [approvedBy], references: [id])

  @@index([tenantId, fromDate])
  @@index([fieldExecutiveId, fromDate])
  @@map("field_leave_applications")
}
```

Leave flow:

```text
Apply leave
  -> Manager approval
  -> Approved leave blocks normal attendance expectations
  -> Dashboard shows ON_LEAVE
```

---

## 12. Dynamic Survey Builder Schema

The survey builder supports reusable forms for outlet audits, market surveys, competitor surveys, display audits, complaint forms, expense forms, collection forms, and daily reports.

```prisma
model SurveyForm {
  id          String   @id @default(uuid())
  tenantId    String   @map("tenant_id")
  formCode    String   @map("form_code")
  formName    String   @map("form_name")
  formType    String   @map("form_type")
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  createdBy   String?  @map("created_by")
  updatedAt   DateTime @updatedAt @map("updated_at")
  updatedBy   String?  @map("updated_by")

  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fields SurveyFormField[]
  submissions SurveySubmission[]

  @@unique([tenantId, formCode])
  @@index([tenantId])
  @@map("survey_forms")
}

model SurveyFormField {
  id         String          @id @default(uuid())
  formId     String          @map("form_id")
  fieldKey   String          @map("field_key")
  label      String
  fieldType  SurveyFieldType @map("field_type")
  isRequired Boolean         @default(false) @map("is_required")
  sortOrder  Int             @default(0) @map("sort_order")
  options    Json?
  validation Json?

  form SurveyForm @relation(fields: [formId], references: [id], onDelete: Cascade)
  answers SurveySubmissionAnswer[]

  @@unique([formId, fieldKey])
  @@index([formId])
  @@map("survey_form_fields")
}

model SurveySubmission {
  id               String   @id @default(uuid())
  tenantId         String   @map("tenant_id")
  formId           String   @map("form_id")
  fieldExecutiveId String   @map("field_executive_id")
  outletVisitId    String?  @map("outlet_visit_id")
  outletId         String?  @map("outlet_id")
  marketId         String?  @map("market_id")
  submittedAt      DateTime @default(now()) @map("submitted_at")
  latitude         Decimal? @db.Decimal(9, 6)
  longitude        Decimal? @db.Decimal(9, 6)
  isOfflineSync    Boolean  @default(false) @map("is_offline_sync")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  form           SurveyForm            @relation(fields: [formId], references: [id])
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id])
  outletVisit    OutletVisit?          @relation(fields: [outletVisitId], references: [id])
  outlet         FieldOutlet?          @relation(fields: [outletId], references: [id])
  market         Market?               @relation(fields: [marketId], references: [id])

  answers SurveySubmissionAnswer[]

  @@index([tenantId, submittedAt])
  @@index([formId, submittedAt])
  @@index([fieldExecutiveId, submittedAt])
  @@map("survey_submissions")
}

model SurveySubmissionAnswer {
  id              String @id @default(uuid())
  submissionId    String @map("submission_id")
  fieldId         String @map("field_id")
  answerValue     Json?  @map("answer_value")
  storageObjectId String? @map("storage_object_id")

  submission    SurveySubmission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  field         SurveyFormField  @relation(fields: [fieldId], references: [id])
  storageObject StorageObject?   @relation(fields: [storageObjectId], references: [id])

  @@index([submissionId])
  @@map("survey_submission_answers")
}
```

Survey flow:

```text
Admin creates form
  -> Admin adds fields
  -> Form is published
  -> Executive opens form during visit or as standalone survey
  -> Answers are submitted with GPS and optional media
  -> Manager reviews submissions
```

---

## 13. Daily Summary Schema

Daily summaries are derived and can be recalculated. They exist to make dashboards fast.

```prisma
model FieldDailySummary {
  id               String   @id @default(uuid())
  tenantId         String   @map("tenant_id")
  fieldExecutiveId String   @map("field_executive_id")
  summaryDate      DateTime @map("summary_date") @db.Date
  firstPingAt      DateTime? @map("first_ping_at")
  lastPingAt       DateTime? @map("last_ping_at")
  clockInAt        DateTime? @map("clock_in_at")
  clockOutAt       DateTime? @map("clock_out_at")
  workMinutes      Int      @default(0) @map("work_minutes")
  breakMinutes     Int      @default(0) @map("break_minutes")
  idleMinutes      Int      @default(0) @map("idle_minutes")
  distanceMeters   Int      @default(0) @map("distance_meters")
  plannedVisits    Int      @default(0) @map("planned_visits")
  completedVisits  Int      @default(0) @map("completed_visits")
  missedVisits     Int      @default(0) @map("missed_visits")
  orderCount       Int      @default(0) @map("order_count")
  orderAmount      Decimal  @default(0) @map("order_amount") @db.Decimal(12, 2)
  expenseAmount    Decimal  @default(0) @map("expense_amount") @db.Decimal(12, 2)
  createdAt        DateTime @default(now()) @map("created_at")
  updatedAt        DateTime @updatedAt @map("updated_at")

  tenant         Tenant                @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  fieldExecutive FieldExecutiveProfile @relation(fields: [fieldExecutiveId], references: [id], onDelete: Cascade)

  @@unique([fieldExecutiveId, summaryDate])
  @@index([tenantId, summaryDate])
  @@map("field_daily_summaries")
}
```

---

## 14. Manual SQL Constraints

Add these in the migration generated for `field.prisma`.

```sql
-- A position should not have duplicate active assignments to the same market.
CREATE UNIQUE INDEX position_active_market_unique
ON position_market_assignments (position_id, market_id)
WHERE effective_to IS NULL;

-- A position should only have one active primary market.
CREATE UNIQUE INDEX position_one_primary_market
ON position_market_assignments (position_id)
WHERE is_primary = true AND effective_to IS NULL;

-- A field executive should only have one open session.
CREATE UNIQUE INDEX field_exec_one_open_session
ON field_work_sessions (field_executive_id)
WHERE clock_out_at IS NULL;

-- A work session should only have one open break.
CREATE UNIQUE INDEX field_exec_one_open_break
ON field_work_breaks (work_session_id)
WHERE ended_at IS NULL;

-- Order totals cannot be negative.
ALTER TABLE field_sales_orders
ADD CONSTRAINT field_sales_orders_non_negative_totals
CHECK (
  subtotal_amount >= 0
  AND discount_amount >= 0
  AND tax_amount >= 0
  AND total_amount >= 0
);

-- Expense claim item amount cannot be negative.
ALTER TABLE field_expense_claim_items
ADD CONSTRAINT field_expense_claim_items_non_negative_amount
CHECK (amount >= 0);

-- Leave date range must be valid.
ALTER TABLE field_leave_applications
ADD CONSTRAINT field_leave_valid_date_range
CHECK (to_date >= from_date);
```

---

## 15. Backend Module Structure

Recommended server structure:

```text
src/modules/field/
  field.module.ts

  markets/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    markets.controller.ts

  executives/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    field-executives.controller.ts

  work-sessions/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    work-sessions.controller.ts

  beats/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    beat-plans.controller.ts

  outlets/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    outlets.controller.ts

  visits/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    outlet-visits.controller.ts

  orders/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    field-orders.controller.ts

  expenses/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    field-expenses.controller.ts

  leave/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    field-leave.controller.ts

  surveys/
    domain/
    application/use-cases/
    infrastructure/
    dto/
    surveys.controller.ts
```

This follows the existing module style: domain model, repository interface, use cases, repository implementation, and controller.

---

## 16. Key Use Cases

### Markets

- Create market
- Update market
- List markets
- Get market
- Delete/soft-delete market
- Assign market to position
- Unassign market from position
- List current users for market

### Field Executives

- Enable field executive profile
- Disable/suspend field executive profile
- Get current assigned markets
- Get current beat plan
- Get dashboard summary

### Work Sessions

- Start day
- End day
- Start break
- End break
- Add location ping
- Get current session
- Get route history
- Recalculate daily summary

### Beat and Visits

- Create beat plan
- Publish beat plan
- Add beat stops
- Check in to outlet
- Check out from outlet
- Mark missed visit
- Record extra visit
- Add visit photo/signature/media

### Sales Orders

- Create draft order
- Add/update/remove order item
- Submit order
- Approve order
- Reject order
- Convert approved field order to ERP sales order later

### Expenses and TA

- Generate daily TA draft from work session
- Add expense item
- Upload receipt
- Submit claim
- Approve/reject claim
- Mark settled

### Leave

- Apply leave
- Approve/reject leave
- Cancel leave
- Block or warn on attendance for approved leave dates

### Surveys

- Create form
- Add/update form fields
- Publish form
- Submit survey
- Review submissions

---

## 17. API Surface Draft

```text
GET    /field/markets
POST   /field/markets
GET    /field/markets/:id
PATCH  /field/markets/:id
DELETE /field/markets/:id

POST   /field/markets/:id/positions
DELETE /field/markets/:id/positions/:assignmentId

GET    /field/executives/me
GET    /field/executives/me/markets
GET    /field/executives/me/beat-plan/today

POST   /field/work-sessions/start
POST   /field/work-sessions/:id/end
POST   /field/work-sessions/:id/breaks/start
POST   /field/work-sessions/:id/breaks/:breakId/end
POST   /field/location-pings

POST   /field/beat-plans
POST   /field/beat-plans/:id/publish
POST   /field/beat-plans/:id/stops

POST   /field/outlet-visits/check-in
POST   /field/outlet-visits/:id/check-out
POST   /field/outlet-visits/:id/media

POST   /field/orders
PATCH  /field/orders/:id
POST   /field/orders/:id/items
POST   /field/orders/:id/submit
POST   /field/orders/:id/approve
POST   /field/orders/:id/reject

POST   /field/expense-claims
POST   /field/expense-claims/:id/items
POST   /field/expense-claims/:id/submit
POST   /field/expense-claims/:id/approve
POST   /field/expense-claims/:id/reject

POST   /field/leave-applications
POST   /field/leave-applications/:id/approve
POST   /field/leave-applications/:id/reject

POST   /field/survey-forms
POST   /field/survey-forms/:id/fields
POST   /field/survey-forms/:id/publish
POST   /field/survey-submissions
```

---

## 18. Permissions Draft

Add permission constants for:

```text
field.markets.create
field.markets.read
field.markets.update
field.markets.delete
field.marketAssignments.manage

field.executives.read
field.executives.manage
field.liveTracking.read

field.workSessions.start
field.workSessions.end
field.workSessions.read
field.workSessions.manage

field.beatPlans.create
field.beatPlans.read
field.beatPlans.update
field.beatPlans.publish

field.outlets.create
field.outlets.read
field.outlets.update
field.outlets.delete

field.visits.create
field.visits.read
field.visits.update

field.orders.create
field.orders.read
field.orders.update
field.orders.approve

field.expenses.create
field.expenses.read
field.expenses.update
field.expenses.approve
field.expenses.settle

field.leave.create
field.leave.read
field.leave.approve

field.surveys.create
field.surveys.read
field.surveys.update
field.surveys.submit
```

---

## 19. Implementation Phases

### Phase 1 - Market and Field Tracking Foundation

- Add `field.prisma`
- Add market CRUD
- Add position market assignment
- Add field executive profile
- Add start/end day
- Add location pings
- Add manager live dashboard APIs

### Phase 2 - Beat and Outlet Visits

- Add outlet CRUD
- Add beat plan CRUD
- Add beat stops
- Add outlet check-in/check-out
- Add visit media
- Add missed and extra visit reporting

### Phase 3 - Sales Orders

- Add field sales order draft
- Add order items
- Add submit/approve/reject lifecycle
- Add sales order dashboard metrics
- Prepare conversion hook for future Sales module

### Phase 4 - Expense and TA

- Add TA rules
- Add expense claim lifecycle
- Add receipt upload through `StorageObject`
- Add approval flow
- Add settlement marker

### Phase 5 - Leave Applications

- Add field leave applications
- Add approval flow
- Integrate leave status into attendance expectations

### Phase 6 - Dynamic Survey Builder

- Add forms
- Add fields
- Add survey submissions
- Add file/signature answers via `StorageObject`
- Add outlet visit survey flow

---

## 20. Reporting Queries Enabled

### Market-Wise Report

```text
Market
  -> PositionMarketAssignment
  -> Position
  -> active PositionAssignment
  -> User/Employee
  -> WorkSession, Visit, Order, Expense
```

Metrics:

- Active executives
- Present executives
- Current locations
- Planned visits
- Completed visits
- Missed visits
- Order amount
- Expense amount
- Distance travelled

### Executive-Wise Report

Metrics:

- Clock-in and clock-out time
- Work minutes
- Break minutes
- Idle minutes
- GPS route
- Beat completion
- Order count and value
- Expense claim status
- Leave history
- Survey submission count

### Outlet-Wise Report

Metrics:

- Last visit date
- Visit frequency
- Orders
- Surveys
- Photos/signatures
- Complaints or feedback through survey forms

---

## 21. Open Design Questions

1. Should outlets eventually become CRM customers, or remain field-specific until conversion?
2. Should `FieldLeaveApplication` be temporary until a full HR leave module exists?
3. Should sales order products reference an inventory/product table immediately, or keep `productId` nullable for MVP?
4. Should TA rules be global per tenant, per role/position, or per employee?
5. Should location pings have a retention policy table, or should retention be tenant setting based?
6. Do market hierarchies need arbitrary depth, or should the UI enforce fixed levels?

---

## 22. Verification Plan

After implementation:

1. Run Prisma validation.
2. Generate Prisma client.
3. Run unit tests for domain/use cases.
4. Run integration tests for tenant isolation and position-based market resolution.
5. Verify manual SQL partial indexes exist in the test database.
6. Seed a sample flow:
   - Tenant
   - Employee/user
   - Position
   - Market
   - Position market assignment
   - Field executive profile
   - Outlet
   - Beat plan
   - Work session
   - Visit
   - Order
   - Expense claim
   - Survey submission

---

## 23. MVP Acceptance Criteria

The first release is acceptable when:

1. Admin can create markets and outlets.
2. Admin can assign markets to positions.
3. A user assigned to the position can see assigned markets.
4. Field executive can start and end the day.
5. Field executive can send location pings.
6. Manager can see current field status market-wise.
7. Manager can create a beat plan.
8. Field executive can check in and out of outlets.
9. Field executive can create a sales order.
10. Field executive can submit an expense claim.
11. Field executive can apply for leave.
12. Admin can create a survey form.
13. Field executive can submit a survey during an outlet visit.

