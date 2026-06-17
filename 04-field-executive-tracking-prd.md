# Field Executive Tracking — Product Requirements Document (PRD)

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: In Development
**Module Code**: `FIELD`
**Owner**: Product Team
**Dependency**: Core Module + HR Module (Employee records required)

---

## 1. Overview

The Field Executive Tracking Module provides real-time visibility into the location, attendance, and daily work of employees who operate in the field (sales reps, delivery personnel, service technicians, collection agents, etc.).

This is a **mobile-first module** with a companion mobile app for field executives and a web dashboard for managers.

---

## 2. Scope

### In Scope
- GPS-based check-in / check-out
- Real-time location tracking (configurable frequency)
- Geo-fenced attendance validation
- Daily journey / visit log
- Task assignment to field executives with location context
- Work report submission by field executive
- Manager dashboard: live map, daily summary
- Route history replay

### Out of Scope
- Vehicle tracking (separate IoT integration — Phase 3)
- Navigation / route optimization
- Customer visit scheduling (→ CRM Module, Phase 6)

---

## 3. User Personas

| Persona | Description |
|---------|-------------|
| **Field Executive** | Employee who operates outside office; uses mobile app |
| **Field Manager** | Manager who supervises field executives; uses web dashboard |
| **TenantAdmin** | Configures geo-fences, tracking frequency, work areas |

---

## 4. Key Concepts

### 4.1 Check-In / Check-Out
A field executive starts their workday by **checking in** — capturing GPS location, timestamp, and optionally a selfie. At end of day they **check out**.

### 4.2 Geo-Fence
A virtual geographic boundary (circle/polygon) defined by lat/lng + radius. Attendance is validated if the executive is within the designated geo-fence at check-in time.

### 4.3 Location Ping
While the app is active, GPS coordinates are recorded at a configurable interval (default: every 5 minutes). This builds a route trace for the day.

### 4.4 Site Visit
A discrete event where the field executive logs arrival at a customer/prospect location with GPS tag, photos, and notes.

### 4.5 Daily Work Report (DWR)
An end-of-day summary submitted by the field executive: visits completed, items collected, observations.

---

## 5. Feature Requirements

### 5.1 Attendance — Check-In / Check-Out

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-A-01 | Field executive can check in from mobile app | P0 |
| FT-A-02 | Check-in captures GPS lat/lng, timestamp, device info | P0 |
| FT-A-03 | Optional selfie capture at check-in (for face-verify use case) | P1 |
| FT-A-04 | Check-in validated against assigned geo-fence if configured | P0 |
| FT-A-05 | If outside geo-fence, check-in allowed with `outsideGeoFence = true` flag | P0 |
| FT-A-06 | Manager notified if executive checks in outside geo-fence | P1 |
| FT-A-07 | Check-out records GPS location, total distance, and end timestamp | P0 |
| FT-A-08 | Only one active check-in per executive per calendar day | P0 |

### 5.2 Real-Time Location Tracking

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-L-01 | Mobile app sends location pings at configurable interval (1–30 min, default: 5 min) | P0 |
| FT-L-02 | Tracking only active while executive is checked in | P0 |
| FT-L-03 | Manager dashboard shows live location of all checked-in executives on a map | P0 |
| FT-L-04 | Location history stored for 90 days (configurable) | P1 |
| FT-L-05 | Route replay: manager can view day's GPS trail for any executive | P1 |
| FT-L-06 | Battery-aware tracking: reduce ping frequency at low battery | P2 |

### 5.3 Geo-Fence Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-G-01 | TenantAdmin can create geo-fences (name, lat/lng center, radius in meters) | P0 |
| FT-G-02 | Geo-fences can be assigned to specific Departments or individual employees | P0 |
| FT-G-03 | Geo-fence shapes: circular (Phase 1), polygon (Phase 2) | P0/P2 |
| FT-G-04 | Geo-fences support effective dates (from/to) | P1 |

### 5.4 Site Visit Logging

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-V-01 | Executive can log a site visit with: location name, lat/lng, arrival time, departure time | P0 |
| FT-V-02 | Visit supports attaching notes and photos (max 5 photos per visit) | P1 |
| FT-V-03 | Visit GPS auto-captured from device; manual override allowed with audit flag | P0 |
| FT-V-04 | Visit can be linked to a Task | P1 |

### 5.5 Daily Work Report (DWR)

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-D-01 | Executive submits DWR at or after check-out | P0 |
| FT-D-02 | DWR includes: summary text, visits count, key outcomes, next day plan | P0 |
| FT-D-03 | DWR submitted to manager for acknowledgment (not approval) | P1 |
| FT-D-04 | Manager can add comments to DWR | P1 |
| FT-D-05 | DWR missed notification sent if not submitted by EOD | P1 |

### 5.6 Manager Dashboard

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-M-01 | Live map showing all checked-in executives with name, last ping time, battery | P0 |
| FT-M-02 | Filter by department, business unit | P0 |
| FT-M-03 | Today's summary: checked-in count, checked-out count, not-checked-in count | P0 |
| FT-M-04 | Per-executive day view: route trace + visits on map | P1 |
| FT-M-05 | Attendance report: last 30/90 days, sortable by executive | P1 |

### 5.7 Task Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| FT-T-01 | Field tasks visible in mobile app with location context | P0 |
| FT-T-02 | Completing a task from the app auto-logs the GPS location at completion | P1 |
| FT-T-03 | Task with location requirement triggers geo-validation before marking DONE | P2 |

---

## 6. Proposed Data Models (New Schema File: `field.prisma`)

```prisma
model GeoFence {
  id           String    @id @default(uuid())
  tenantId     String    @map("tenant_id")
  fenceName    String    @map("fence_name")
  latitude     Decimal   @db.Decimal(10, 7)
  longitude    Decimal   @db.Decimal(10, 7)
  radiusMeters Int       @map("radius_meters")
  isActive     Boolean   @default(true) @map("is_active")
  effectiveFrom DateTime? @map("effective_from")
  effectiveTo   DateTime? @map("effective_to")
  createdAt    DateTime  @default(now()) @map("created_at")
  createdBy    String?   @map("created_by")

  fieldAttendances FieldAttendance[]
  @@map("geo_fences")
}

model FieldAttendance {
  id              String    @id @default(uuid())
  tenantId        String    @map("tenant_id")
  employeeId      String    @map("employee_id")   // HR Module
  checkInAt       DateTime  @map("check_in_at")
  checkInLat      Decimal   @map("check_in_lat")  @db.Decimal(10, 7)
  checkInLng      Decimal   @map("check_in_lng")  @db.Decimal(10, 7)
  checkOutAt      DateTime? @map("check_out_at")
  checkOutLat     Decimal?  @map("check_out_lat") @db.Decimal(10, 7)
  checkOutLng     Decimal?  @map("check_out_lng") @db.Decimal(10, 7)
  outsideGeoFence Boolean   @default(false) @map("outside_geo_fence")
  geoFenceId      String?   @map("geo_fence_id")
  totalDistanceKm Decimal?  @map("total_distance_km") @db.Decimal(8, 2)

  @@map("field_attendances")
}

model LocationPing {
  id          String   @id @default(uuid())
  tenantId    String   @map("tenant_id")
  employeeId  String   @map("employee_id")
  attendanceId String  @map("attendance_id")
  latitude    Decimal  @db.Decimal(10, 7)
  longitude   Decimal  @db.Decimal(10, 7)
  accuracy    Decimal? @db.Decimal(6, 2)
  battery     Int?
  recordedAt  DateTime @map("recorded_at")

  @@index([attendanceId, recordedAt])
  @@map("location_pings")
}

model SiteVisit {
  id           String    @id @default(uuid())
  tenantId     String    @map("tenant_id")
  employeeId   String    @map("employee_id")
  attendanceId String?   @map("attendance_id")
  taskId       String?   @map("task_id")
  siteName     String    @map("site_name")
  latitude     Decimal   @db.Decimal(10, 7)
  longitude    Decimal   @db.Decimal(10, 7)
  arrivedAt    DateTime  @map("arrived_at")
  departedAt   DateTime? @map("departed_at")
  notes        String?

  @@map("site_visits")
}
```

---

## 7. Mobile App Requirements

| Platform | Framework |
|----------|-----------|
| Android | React Native / Flutter |
| iOS | React Native / Flutter |

**Key mobile features**:
- Background location permission required
- Offline support: location pings cached locally and synced when online
- Low-data mode: reduce ping frequency on 2G/poor connectivity
- Local notification for check-in reminders (8 AM)
- Biometric unlock for app

---

## 8. Privacy & Compliance

- Location data is **only collected while the executive is checked in** (not 24/7).
- Executive can view their own location history at any time.
- Location data retained for 90 days by default (configurable per tenant).
- Data collection policy disclosed at app install.
- Compliant with **DPDPA 2023** (India's Digital Personal Data Protection Act).

---

## 9. Permissions

| Permission | Description |
|------------|-------------|
| `field:check_in` | Check in/out (field executive) |
| `field:log_visit` | Log a site visit |
| `field:submit_dwr` | Submit daily work report |
| `field:view_own` | View own attendance and location history |
| `field:view_team` | View team's attendance and location (manager) |
| `field:manage_geofences` | Create/edit geo-fences (admin) |

---

*Document Owner: Product Team | Review Cycle: Per sprint*
