# HRM Schema Design

**Version**: 0.1.0
**Last Updated**: 2026-06-24
**Status**: Draft
**Owner**: Backend Team
**Depends On**: `05-org-and-access-control-design.md`, `08-database-design.md`, `14-position-model-updates.md`

---

## 1. Objective

Design the HRM data model for a SaaS ERP platform that can support the full employee lifecycle:

1. Recruitment and candidate tracking
2. Onboarding
3. Employee master records
4. Employee documents
5. Attendance and shift management
6. Leave management
7. Payroll and compensation
8. Performance management
9. Training and certifications
10. Compliance
11. Offboarding
12. Tenant-specific custom fields and forms
13. Audit trails, versioning, reporting, and analytics

The `employees` table should remain the root HR entity. It must not grow into a single giant table for every HR concern.

---

## 2. Design Principles

1. **Tenant first**: Every tenant-owned HR table must include `tenant_id`.
2. **Employee as root aggregate**: `employees` stores stable employee identity and current state only.
3. **History over overwrite**: Position, manager, department, status, and compensation changes must be captured in history tables.
4. **Documents as first-class data**: Store document metadata, lifecycle, versioning, verification, and expiry in DB. Store file bytes in object storage.
5. **Customizable without migrations**: Tenant custom fields and forms should use metadata-driven tables.
6. **Reporting-ready**: Data used for reports should be typed and indexed, not hidden only in JSON.
7. **Soft delete for business records**: Use `deleted_at` and `deleted_by` where records may need recovery or audit.
8. **Immutable financial records**: Payroll run results and payslips should be immutable once finalized.
9. **Workflow integration**: Approval-heavy modules should link to workflow instances.
10. **Cursor pagination required**: All high-volume list APIs must use cursor pagination.

---

## 3. Recommended Schema Files

| File | Purpose |
|------|---------|
| `hr.prisma` | Employee root profile and employee-related master data |
| `hr-documents.prisma` | Employee document management |
| `hr-customization.prisma` | Custom fields, forms, and submissions |
| `hr-recruitment.prisma` | Recruitment and candidate tracking |
| `hr-onboarding.prisma` | Onboarding templates and instances |
| `hr-attendance.prisma` | Attendance, shifts, and regularization |
| `hr-leave.prisma` | Leave types, policies, balances, and requests |
| `hr-payroll.prisma` | Payroll, salary structures, components, and payslips |
| `hr-performance.prisma` | Goals, reviews, ratings, and feedback |
| `hr-training.prisma` | Courses, sessions, certifications |
| `hr-compliance.prisma` | Compliance requirements and employee compliance records |
| `hr-offboarding.prisma` | Exit workflows and clearance |

The team may keep fewer physical files initially, but module boundaries should remain clear.

---

## 4. Shared Enums

### 4.1 Employment and Lifecycle

| Enum | Values |
|------|--------|
| `employment_status` | `ACTIVE`, `PROBATION`, `ON_LEAVE`, `SUSPENDED`, `RESIGNED`, `TERMINATED`, `RETIRED` |
| `employment_type` | `FULL_TIME`, `PART_TIME`, `CONTRACT`, `TEMPORARY`, `INTERN`, `TRAINEE`, `CONSULTANT` |
| `worker_type` | `EMPLOYEE`, `CONTRACTOR`, `CONSULTANT`, `INTERN`, `VENDOR_STAFF` |
| `lifecycle_event_type` | `HIRED`, `ONBOARDED`, `TRANSFERRED`, `PROMOTED`, `COMPENSATION_CHANGED`, `STATUS_CHANGED`, `EXIT_INITIATED`, `EXIT_COMPLETED` |

### 4.2 Documents

| Enum | Values |
|------|--------|
| `document_status` | `DRAFT`, `SUBMITTED`, `VERIFIED`, `REJECTED`, `EXPIRED`, `ARCHIVED` |
| `document_visibility` | `EMPLOYEE`, `MANAGER`, `HR`, `PAYROLL`, `ADMIN` |
| `document_source` | `EMPLOYEE_UPLOAD`, `HR_UPLOAD`, `SYSTEM_GENERATED`, `INTEGRATION` |

### 4.3 Recruitment

| Enum | Values |
|------|--------|
| `job_requisition_status` | `DRAFT`, `PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `CLOSED`, `CANCELLED` |
| `candidate_status` | `NEW`, `SCREENING`, `INTERVIEW`, `OFFERED`, `HIRED`, `REJECTED`, `WITHDRAWN` |
| `offer_status` | `DRAFT`, `SENT`, `ACCEPTED`, `DECLINED`, `EXPIRED`, `REVOKED` |

### 4.4 Attendance and Leave

| Enum | Values |
|------|--------|
| `attendance_status` | `PRESENT`, `ABSENT`, `HALF_DAY`, `LATE`, `ON_LEAVE`, `HOLIDAY`, `WEEKLY_OFF`, `REMOTE` |
| `punch_type` | `IN`, `OUT`, `BREAK_IN`, `BREAK_OUT` |
| `leave_request_status` | `DRAFT`, `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`, `WITHDRAWN` |
| `leave_balance_unit` | `DAYS`, `HOURS` |

### 4.5 Payroll

| Enum | Values |
|------|--------|
| `salary_component_type` | `EARNING`, `DEDUCTION`, `BENEFIT`, `TAX`, `REIMBURSEMENT` |
| `salary_component_frequency` | `MONTHLY`, `ANNUAL`, `ONE_TIME`, `PER_PAY_RUN` |
| `payroll_run_status` | `DRAFT`, `PROCESSING`, `REVIEW`, `APPROVED`, `PAID`, `LOCKED`, `CANCELLED` |
| `payslip_status` | `DRAFT`, `GENERATED`, `PUBLISHED`, `REVISED`, `VOID` |

### 4.6 Performance, Training, Compliance

| Enum | Values |
|------|--------|
| `review_cycle_status` | `DRAFT`, `ACTIVE`, `CALIBRATION`, `COMPLETED`, `ARCHIVED` |
| `goal_status` | `DRAFT`, `ACTIVE`, `ACHIEVED`, `MISSED`, `CANCELLED` |
| `training_status` | `PLANNED`, `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` |
| `compliance_status` | `NOT_STARTED`, `PENDING`, `COMPLIANT`, `NON_COMPLIANT`, `EXPIRED`, `WAIVED` |

---

## 5. Core Employee Schema

### 5.1 `employees`

Root employee record. Stores stable profile and current state only.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | FK to `tenants`, required |
| `user_id` | uuid? | FK to `users`, nullable for pre-account employees |
| `employee_code` | varchar | Tenant-scoped unique for active employees |
| `worker_type` | enum | Employee, contractor, consultant, etc. |
| `employment_status` | enum | Current HR status |
| `employment_type` | enum? | Full-time, contract, etc. |
| `legal_first_name` | varchar | Required |
| `legal_middle_name` | varchar? | Optional |
| `legal_last_name` | varchar? | Optional |
| `preferred_name` | varchar? | Optional |
| `display_name` | varchar | Denormalized for search/display |
| `work_email` | varchar? | Tenant-scoped unique for active employees |
| `personal_email` | varchar? | Optional |
| `mobile_number` | varchar? | Optional |
| `date_of_birth` | date? | Sensitive |
| `gender` | varchar? | Prefer tenant-configurable options |
| `date_of_joining` | date? | Current joining date |
| `date_of_exit` | date? | Filled after exit |
| `business_unit_id` | uuid? | Current BU |
| `department_id` | uuid? | Current department |
| `designation_id` | uuid? | Current designation |
| `position_id` | uuid? | Current primary position |
| `manager_employee_id` | uuid? | Current reporting manager |
| `location_id` | uuid? | Work location |
| `custom_attributes` | jsonb | Lightweight metadata only |
| `created_at/by` | audit | Required |
| `updated_at/by` | audit | Required |
| `deleted_at/by` | audit | Soft delete |

Recommended indexes:

| Index | Purpose |
|-------|---------|
| `UNIQUE (tenant_id, employee_code) WHERE deleted_at IS NULL` | Active employee code uniqueness |
| `UNIQUE (tenant_id, work_email) WHERE work_email IS NOT NULL AND deleted_at IS NULL` | Active work email uniqueness |
| `INDEX (tenant_id, employment_status)` | HR lists |
| `INDEX (tenant_id, department_id)` | Department filters |
| `INDEX (tenant_id, business_unit_id)` | BU filters |
| `INDEX (tenant_id, manager_employee_id)` | Manager tree |
| `INDEX (tenant_id, position_id)` | Position lookup |
| `INDEX (tenant_id, deleted_at, created_at DESC, id)` | Cursor pagination |
| `GIN display/search index` | Employee search |

### 5.2 `employee_personal_details`

Sensitive personal information separated from the base employee list.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK to `employees`, unique |
| `marital_status` | varchar? | Tenant-configurable |
| `blood_group` | varchar? | Optional |
| `nationality` | varchar? | Optional |
| `tax_identifier` | varchar? | Country-specific |
| `government_identifier` | varchar? | Country-specific |
| `disability_status` | varchar? | Sensitive |
| `created_at/by`, `updated_at/by` | audit | Required |

### 5.3 `employee_addresses`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `address_type` | varchar | `PERMANENT`, `CURRENT`, `WORK`, custom |
| `line1`, `line2` | varchar | Address lines |
| `city`, `state`, `country`, `postal_code` | varchar | Address parts |
| `is_primary` | boolean | Default false |
| `effective_from`, `effective_to` | date? | History support |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id)`, `(tenant_id, address_type)`.

### 5.4 `employee_emergency_contacts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `contact_name` | varchar | Required |
| `relationship` | varchar | Required |
| `mobile_number` | varchar | Required |
| `email` | varchar? | Optional |
| `address_text` | text? | Optional |
| `is_primary` | boolean | Default false |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 5.5 `employee_bank_accounts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `account_holder_name` | varchar | Required |
| `bank_name` | varchar | Required |
| `branch_name` | varchar? | Optional |
| `account_number_encrypted` | text | Required |
| `routing_code` | varchar? | IFSC/SWIFT/ABA/etc. |
| `account_type` | varchar? | Tenant/country-specific |
| `is_primary` | boolean | Default false |
| `verification_status` | varchar | Optional |
| `effective_from`, `effective_to` | date? | History support |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 5.6 `employee_dependents`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `full_name` | varchar | Required |
| `relationship` | varchar | Required |
| `date_of_birth` | date? | Optional |
| `is_nominee` | boolean | Default false |
| `metadata` | jsonb | Optional |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 5.7 `employee_identifiers`

For PAN, Aadhaar, SSN, passport, visa, insurance, tax IDs, and other country-specific identifiers.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `identifier_type` | varchar | Tenant/country-specific |
| `identifier_value_encrypted` | text | Required |
| `issuing_country` | varchar? | Optional |
| `issued_at` | date? | Optional |
| `expires_at` | date? | Optional |
| `document_id` | uuid? | FK to `employee_documents` |
| `verification_status` | varchar | Optional |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id)`, `(tenant_id, identifier_type)`, `(tenant_id, expires_at)`.

---

## 6. Employee History Tables

### 6.1 `employee_lifecycle_events`

Immutable timeline of major HR events.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `event_type` | enum | Required |
| `event_date` | date | Required |
| `summary` | text? | Optional |
| `old_values` | jsonb? | Snapshot |
| `new_values` | jsonb? | Snapshot |
| `workflow_instance_id` | uuid? | FK to workflow instance |
| `created_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id, event_date DESC)`, `(tenant_id, event_type, event_date DESC)`.

### 6.2 `employee_position_history`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `position_id` | uuid? | FK to positions |
| `designation_id` | uuid? | FK to designations |
| `department_id` | uuid? | FK to departments |
| `business_unit_id` | uuid? | FK to business units |
| `manager_employee_id` | uuid? | FK to employees |
| `effective_from` | date | Required |
| `effective_to` | date? | Null means current |
| `change_reason` | varchar? | Optional |
| `created_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id, effective_from DESC)`, `(tenant_id, department_id, effective_to)`.

### 6.3 `employee_status_history`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `status` | enum | Required |
| `effective_from` | date | Required |
| `effective_to` | date? | Null means current |
| `reason` | varchar? | Optional |
| `notes` | text? | Optional |
| `created_at/by` | audit | Required |

### 6.4 `employee_compensation_history`

Use this for high-level compensation assignment history. Payroll calculation tables are separate.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `currency_code` | varchar | Required |
| `annual_ctc` | decimal | Optional |
| `monthly_gross` | decimal | Optional |
| `salary_structure_id` | uuid? | FK |
| `effective_from` | date | Required |
| `effective_to` | date? | Null means current |
| `change_reason` | varchar? | Optional |
| `workflow_instance_id` | uuid? | Optional |
| `created_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id, effective_from DESC)`.

---

## 7. Employee Document Management

### 7.1 `document_categories`

Tenant-configurable categories such as ID Proof, Contract, Certificate, Appraisal, Compliance, Visa.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `category_name` | varchar | Required |
| `description` | text? | Optional |
| `requires_expiry` | boolean | Default false |
| `requires_verification` | boolean | Default false |
| `retention_months` | int? | Optional |
| `is_active` | boolean | Default true |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

Indexes: `UNIQUE (tenant_id, category_name) WHERE deleted_at IS NULL`.

### 7.2 `employee_documents`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `category_id` | uuid | FK to `document_categories` |
| `document_name` | varchar | Required |
| `document_number_encrypted` | text? | Optional |
| `status` | enum | Required |
| `visibility` | enum | Required |
| `source` | enum | Required |
| `issued_at` | date? | Optional |
| `expires_at` | date? | Optional |
| `verified_at` | timestamptz? | Optional |
| `verified_by` | uuid? | FK to users |
| `current_version_id` | uuid? | FK to document versions |
| `metadata` | jsonb | Optional |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id)`, `(tenant_id, category_id)`, `(tenant_id, status)`, `(tenant_id, expires_at)`.

### 7.3 `employee_document_versions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `document_id` | uuid | FK to `employee_documents` |
| `version_number` | int | Required |
| `file_key` | varchar | Object storage key |
| `file_name` | varchar | Original file name |
| `file_size` | int? | Optional |
| `mime_type` | varchar? | Optional |
| `checksum` | varchar? | Optional |
| `uploaded_by` | uuid | FK to users |
| `uploaded_at` | timestamptz | Required |
| `change_note` | text? | Optional |

Indexes: `UNIQUE (document_id, version_number)`, `(tenant_id, document_id, version_number DESC)`.

### 7.4 `document_access_logs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `document_id` | uuid | FK |
| `version_id` | uuid? | FK |
| `accessed_by` | uuid | FK to users |
| `access_type` | varchar | `VIEW`, `DOWNLOAD`, `VERIFY`, `DELETE` |
| `ip_address` | varchar? | Optional |
| `user_agent` | text? | Optional |
| `accessed_at` | timestamptz | Required |

Indexes: `(tenant_id, document_id, accessed_at DESC)`, `(tenant_id, accessed_by, accessed_at DESC)`.

---

## 8. Tenant Customization

### 8.1 `custom_field_definitions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `module_name` | varchar | `HR`, `PAYROLL`, `PERFORMANCE`, etc. |
| `entity_type` | varchar | `EMPLOYEE`, `CANDIDATE`, `LEAVE_REQUEST`, etc. |
| `field_key` | varchar | Stable API key |
| `field_label` | varchar | Display label |
| `field_type` | varchar | `TEXT`, `NUMBER`, `DATE`, `SELECT`, `MULTI_SELECT`, `BOOLEAN`, `FILE`, etc. |
| `is_required` | boolean | Default false |
| `is_searchable` | boolean | Default false |
| `is_reportable` | boolean | Default false |
| `validation_rules` | jsonb | Optional |
| `display_order` | int | Default 0 |
| `is_active` | boolean | Default true |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

Indexes: `UNIQUE (tenant_id, entity_type, field_key) WHERE deleted_at IS NULL`.

### 8.2 `custom_field_options`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `field_definition_id` | uuid | FK |
| `option_value` | varchar | Stable value |
| `option_label` | varchar | Display label |
| `display_order` | int | Default 0 |
| `is_active` | boolean | Default true |

### 8.3 `custom_field_values`

Typed values for custom fields. Use entity references instead of per-module JSON blobs when fields must be searchable or reportable.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `field_definition_id` | uuid | FK |
| `entity_type` | varchar | Required |
| `entity_id` | uuid | Required |
| `value_text` | text? | Optional |
| `value_number` | decimal? | Optional |
| `value_date` | date? | Optional |
| `value_boolean` | boolean? | Optional |
| `value_json` | jsonb? | Optional |
| `created_at/by`, `updated_at/by` | audit | Required |

Indexes: `UNIQUE (tenant_id, field_definition_id, entity_id)`, `(tenant_id, entity_type, entity_id)`, `(tenant_id, field_definition_id, value_text)`.

### 8.4 `custom_form_definitions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `module_name` | varchar | Required |
| `entity_type` | varchar | Required |
| `form_key` | varchar | Stable API key |
| `form_name` | varchar | Required |
| `version` | int | Required |
| `status` | varchar | `DRAFT`, `ACTIVE`, `ARCHIVED` |
| `schema_json` | jsonb | Layout and validation |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 8.5 `custom_form_submissions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `form_definition_id` | uuid | FK |
| `entity_type` | varchar | Required |
| `entity_id` | uuid | Required |
| `submitted_by` | uuid | FK to users |
| `submitted_at` | timestamptz | Required |
| `status` | varchar | `DRAFT`, `SUBMITTED`, `APPROVED`, `REJECTED` |
| `response_json` | jsonb | Snapshot of responses |
| `workflow_instance_id` | uuid? | Optional |

---

## 9. Recruitment

### 9.1 `job_requisitions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `requisition_code` | varchar | Tenant-scoped unique |
| `title` | varchar | Required |
| `department_id` | uuid? | FK |
| `business_unit_id` | uuid? | FK |
| `position_id` | uuid? | Optional planned position |
| `hiring_manager_id` | uuid? | FK to users |
| `headcount` | int | Required |
| `status` | enum | Required |
| `target_start_date` | date? | Optional |
| `job_description` | text? | Optional |
| `workflow_instance_id` | uuid? | Approval workflow |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 9.2 `candidates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `first_name` | varchar | Required |
| `last_name` | varchar? | Optional |
| `email` | varchar? | Optional |
| `mobile_number` | varchar? | Optional |
| `source` | varchar? | Referral, portal, agency, etc. |
| `resume_document_id` | uuid? | FK to candidate/employee document store |
| `status` | enum | Required |
| `converted_employee_id` | uuid? | Filled after hire |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 9.3 `candidate_applications`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `candidate_id` | uuid | FK |
| `job_requisition_id` | uuid | FK |
| `application_status` | enum | Candidate status |
| `applied_at` | timestamptz | Required |
| `current_stage` | varchar? | Optional |
| `metadata` | jsonb | Optional |

### 9.4 `interview_rounds`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `application_id` | uuid | FK |
| `round_name` | varchar | Required |
| `round_order` | int | Required |
| `scheduled_at` | timestamptz? | Optional |
| `interviewer_id` | uuid? | FK to users |
| `status` | varchar | Required |
| `feedback_summary` | text? | Optional |
| `rating` | decimal? | Optional |

### 9.5 `candidate_offers`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `application_id` | uuid | FK |
| `offer_number` | varchar | Tenant-scoped unique |
| `status` | enum | Required |
| `offered_compensation` | decimal? | Optional |
| `currency_code` | varchar? | Optional |
| `joining_date` | date? | Optional |
| `offer_document_id` | uuid? | Document link |
| `sent_at`, `accepted_at`, `declined_at` | timestamptz? | Optional |

---

## 10. Onboarding

### 10.1 `onboarding_templates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `template_name` | varchar | Required |
| `employment_type` | enum? | Optional applicability |
| `department_id` | uuid? | Optional applicability |
| `is_active` | boolean | Default true |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 10.2 `onboarding_template_tasks`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `template_id` | uuid | FK |
| `task_title` | varchar | Required |
| `task_description` | text? | Optional |
| `owner_type` | varchar | `EMPLOYEE`, `HR`, `MANAGER`, `IT`, `PAYROLL` |
| `due_offset_days` | int | Relative to joining date |
| `requires_document_category_id` | uuid? | Optional |
| `display_order` | int | Required |

### 10.3 `employee_onboarding_instances`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `template_id` | uuid? | FK |
| `status` | varchar | `NOT_STARTED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` |
| `started_at`, `completed_at` | timestamptz? | Optional |
| `workflow_instance_id` | uuid? | Optional |

### 10.4 `employee_onboarding_tasks`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `onboarding_instance_id` | uuid | FK |
| `task_title` | varchar | Required |
| `owner_user_id` | uuid? | FK |
| `status` | varchar | Required |
| `due_date` | date? | Optional |
| `completed_at`, `completed_by` | audit | Optional |

---

## 11. Attendance and Shift Management

### 11.1 `work_locations`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `location_name` | varchar | Required |
| `timezone` | varchar | Required |
| `address_json` | jsonb | Optional |
| `geo_lat`, `geo_lng` | decimal? | Optional |
| `is_active` | boolean | Default true |

### 11.2 `shift_definitions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `shift_name` | varchar | Required |
| `start_time` | time | Required |
| `end_time` | time | Required |
| `grace_minutes` | int | Default 0 |
| `break_minutes` | int | Default 0 |
| `timezone` | varchar | Required |
| `is_overnight` | boolean | Default false |
| `is_active` | boolean | Default true |

### 11.3 `employee_shift_assignments`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `shift_id` | uuid | FK |
| `effective_from` | date | Required |
| `effective_to` | date? | Null means current |
| `created_at/by` | audit | Required |

### 11.4 `attendance_records`

One row per employee per attendance date.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `attendance_date` | date | Required |
| `shift_id` | uuid? | FK |
| `status` | enum | Required |
| `first_in_at` | timestamptz? | Optional |
| `last_out_at` | timestamptz? | Optional |
| `worked_minutes` | int | Default 0 |
| `late_minutes` | int | Default 0 |
| `overtime_minutes` | int | Default 0 |
| `source` | varchar | Manual, biometric, mobile, import |
| `locked_at` | timestamptz? | Optional |
| `created_at/by`, `updated_at/by` | audit | Required |

Indexes: `UNIQUE (tenant_id, employee_id, attendance_date)`, `(tenant_id, attendance_date)`, `(tenant_id, status, attendance_date)`.

### 11.5 `attendance_punches`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `attendance_record_id` | uuid | FK |
| `employee_id` | uuid | FK |
| `punch_type` | enum | Required |
| `punched_at` | timestamptz | Required |
| `source` | varchar | Required |
| `geo_lat`, `geo_lng` | decimal? | Optional |
| `device_id` | varchar? | Optional |
| `metadata` | jsonb | Optional |

Indexes: `(tenant_id, employee_id, punched_at DESC)`.

### 11.6 `attendance_regularization_requests`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `attendance_record_id` | uuid? | FK |
| `requested_status` | enum | Required |
| `reason` | text | Required |
| `status` | varchar | `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED` |
| `workflow_instance_id` | uuid? | Optional |
| `created_at/by`, `updated_at/by` | audit | Required |

---

## 12. Leave Management

### 12.1 `leave_types`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `leave_code` | varchar | Tenant-scoped unique |
| `leave_name` | varchar | Required |
| `balance_unit` | enum | Days or hours |
| `is_paid` | boolean | Default true |
| `requires_approval` | boolean | Default true |
| `allow_half_day` | boolean | Default true |
| `allow_carry_forward` | boolean | Default false |
| `is_active` | boolean | Default true |

### 12.2 `leave_policies`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `policy_name` | varchar | Required |
| `leave_type_id` | uuid | FK |
| `employment_type` | enum? | Optional applicability |
| `accrual_rules` | jsonb | Required |
| `carry_forward_rules` | jsonb | Optional |
| `encashment_rules` | jsonb | Optional |
| `is_active` | boolean | Default true |
| `effective_from`, `effective_to` | date? | Versioning |

### 12.3 `employee_leave_balances`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `leave_type_id` | uuid | FK |
| `period_start`, `period_end` | date | Required |
| `opening_balance` | decimal | Default 0 |
| `accrued` | decimal | Default 0 |
| `used` | decimal | Default 0 |
| `adjusted` | decimal | Default 0 |
| `closing_balance` | decimal | Default 0 |
| `locked_at` | timestamptz? | Optional |

Indexes: `UNIQUE (tenant_id, employee_id, leave_type_id, period_start, period_end)`.

### 12.4 `leave_requests`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `leave_type_id` | uuid | FK |
| `start_date`, `end_date` | date | Required |
| `duration` | decimal | Required |
| `reason` | text? | Optional |
| `status` | enum | Required |
| `workflow_instance_id` | uuid? | Optional |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

Indexes: `(tenant_id, employee_id, start_date DESC)`, `(tenant_id, status, created_at DESC)`.

### 12.5 `leave_request_days`

Breaks a leave request into daily rows for attendance/payroll/reporting.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `leave_request_id` | uuid | FK |
| `employee_id` | uuid | FK |
| `leave_date` | date | Required |
| `duration` | decimal | Required |
| `is_half_day` | boolean | Default false |

Indexes: `UNIQUE (tenant_id, employee_id, leave_date, leave_request_id)`.

---

## 13. Payroll and Compensation

### 13.1 `salary_components`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `component_code` | varchar | Tenant-scoped unique |
| `component_name` | varchar | Required |
| `component_type` | enum | Earning, deduction, tax, etc. |
| `frequency` | enum | Required |
| `is_taxable` | boolean | Default false |
| `is_statutory` | boolean | Default false |
| `calculation_rules` | jsonb | Optional |
| `is_active` | boolean | Default true |

### 13.2 `salary_structures`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `structure_name` | varchar | Required |
| `currency_code` | varchar | Required |
| `effective_from`, `effective_to` | date? | Versioning |
| `is_active` | boolean | Default true |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 13.3 `salary_structure_components`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `salary_structure_id` | uuid | FK |
| `salary_component_id` | uuid | FK |
| `calculation_type` | varchar | `FIXED`, `PERCENTAGE`, `FORMULA` |
| `amount` | decimal? | Optional |
| `percentage` | decimal? | Optional |
| `formula` | text? | Optional |
| `display_order` | int | Required |

### 13.4 `employee_salary_assignments`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `salary_structure_id` | uuid | FK |
| `currency_code` | varchar | Required |
| `annual_ctc` | decimal? | Optional |
| `monthly_gross` | decimal? | Optional |
| `effective_from`, `effective_to` | date? | Required |
| `workflow_instance_id` | uuid? | Optional |
| `created_at/by` | audit | Required |

### 13.5 `payroll_periods`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `period_name` | varchar | Required |
| `start_date`, `end_date` | date | Required |
| `pay_date` | date? | Optional |
| `is_locked` | boolean | Default false |

### 13.6 `payroll_runs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `payroll_period_id` | uuid | FK |
| `run_code` | varchar | Tenant-scoped unique |
| `status` | enum | Required |
| `processed_at` | timestamptz? | Optional |
| `approved_at/by` | audit | Optional |
| `locked_at/by` | audit | Optional |
| `workflow_instance_id` | uuid? | Optional |

### 13.7 `payroll_run_items`

One row per employee per payroll run.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `payroll_run_id` | uuid | FK |
| `employee_id` | uuid | FK |
| `gross_earnings` | decimal | Required |
| `total_deductions` | decimal | Required |
| `net_pay` | decimal | Required |
| `currency_code` | varchar | Required |
| `calculation_snapshot` | jsonb | Required |
| `status` | varchar | Required |

Indexes: `UNIQUE (tenant_id, payroll_run_id, employee_id)`.

### 13.8 `payroll_run_item_components`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `payroll_run_item_id` | uuid | FK |
| `salary_component_id` | uuid? | FK |
| `component_code` | varchar | Snapshot |
| `component_name` | varchar | Snapshot |
| `component_type` | enum | Snapshot |
| `amount` | decimal | Required |
| `metadata` | jsonb | Optional |

### 13.9 `payslips`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `payroll_run_item_id` | uuid | FK |
| `employee_id` | uuid | FK |
| `status` | enum | Required |
| `document_id` | uuid? | FK to employee document |
| `published_at` | timestamptz? | Optional |
| `published_by` | uuid? | FK to users |

---

## 14. Performance Management

### 14.1 `review_cycles`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `cycle_name` | varchar | Required |
| `period_start`, `period_end` | date | Required |
| `status` | enum | Required |
| `configuration` | jsonb | Optional |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 14.2 `performance_goals`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `review_cycle_id` | uuid? | FK |
| `goal_title` | varchar | Required |
| `goal_description` | text? | Optional |
| `weightage` | decimal? | Optional |
| `target_value` | decimal? | Optional |
| `actual_value` | decimal? | Optional |
| `status` | enum | Required |
| `created_at/by`, `updated_at/by`, `deleted_at/by` | audit | Required |

### 14.3 `employee_reviews`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `review_cycle_id` | uuid | FK |
| `employee_id` | uuid | FK |
| `manager_employee_id` | uuid? | FK |
| `status` | varchar | `DRAFT`, `SELF_REVIEW`, `MANAGER_REVIEW`, `CALIBRATION`, `COMPLETED` |
| `overall_rating` | decimal? | Optional |
| `final_comments` | text? | Optional |
| `workflow_instance_id` | uuid? | Optional |
| `completed_at` | timestamptz? | Optional |

### 14.4 `review_feedback`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_review_id` | uuid | FK |
| `reviewer_user_id` | uuid | FK |
| `feedback_type` | varchar | `SELF`, `MANAGER`, `PEER`, `HR` |
| `rating` | decimal? | Optional |
| `comments` | text? | Optional |
| `submitted_at` | timestamptz? | Optional |

---

## 15. Training and Certifications

### 15.1 `training_courses`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `course_code` | varchar | Tenant-scoped unique |
| `course_name` | varchar | Required |
| `description` | text? | Optional |
| `provider` | varchar? | Optional |
| `is_mandatory` | boolean | Default false |
| `validity_months` | int? | Optional |
| `is_active` | boolean | Default true |

### 15.2 `training_sessions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `course_id` | uuid | FK |
| `session_name` | varchar | Required |
| `start_at`, `end_at` | timestamptz? | Optional |
| `location_text` | varchar? | Optional |
| `trainer_user_id` | uuid? | FK |
| `capacity` | int? | Optional |
| `status` | enum | Required |

### 15.3 `employee_training_enrollments`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `course_id` | uuid | FK |
| `session_id` | uuid? | FK |
| `status` | varchar | `ENROLLED`, `IN_PROGRESS`, `COMPLETED`, `FAILED`, `CANCELLED` |
| `score` | decimal? | Optional |
| `completed_at` | timestamptz? | Optional |
| `certificate_document_id` | uuid? | FK to document |

### 15.4 `employee_certifications`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `certification_name` | varchar | Required |
| `issuer` | varchar? | Optional |
| `issued_at` | date? | Optional |
| `expires_at` | date? | Optional |
| `document_id` | uuid? | FK to document |
| `status` | varchar | Required |

Indexes: `(tenant_id, employee_id)`, `(tenant_id, expires_at)`.

---

## 16. Compliance

### 16.1 `compliance_requirements`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `requirement_code` | varchar | Tenant-scoped unique |
| `requirement_name` | varchar | Required |
| `description` | text? | Optional |
| `applies_to_rules` | jsonb | Department, location, worker type, etc. |
| `requires_document_category_id` | uuid? | Optional |
| `renewal_months` | int? | Optional |
| `is_active` | boolean | Default true |

### 16.2 `employee_compliance_records`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `requirement_id` | uuid | FK |
| `status` | enum | Required |
| `document_id` | uuid? | FK |
| `due_date` | date? | Optional |
| `completed_at` | timestamptz? | Optional |
| `verified_at/by` | audit | Optional |
| `notes` | text? | Optional |

Indexes: `UNIQUE (tenant_id, employee_id, requirement_id)`, `(tenant_id, status, due_date)`.

---

## 17. Offboarding

### 17.1 `offboarding_templates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `template_name` | varchar | Required |
| `employment_type` | enum? | Optional applicability |
| `department_id` | uuid? | Optional applicability |
| `is_active` | boolean | Default true |

### 17.2 `offboarding_template_tasks`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `template_id` | uuid | FK |
| `task_title` | varchar | Required |
| `owner_type` | varchar | `EMPLOYEE`, `HR`, `MANAGER`, `IT`, `PAYROLL`, `ADMIN` |
| `due_offset_days` | int | Relative to exit date |
| `display_order` | int | Required |

### 17.3 `employee_offboarding_instances`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `employee_id` | uuid | FK |
| `template_id` | uuid? | FK |
| `resignation_date` | date? | Optional |
| `last_working_date` | date | Required |
| `exit_reason` | varchar? | Optional |
| `status` | varchar | `INITIATED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` |
| `workflow_instance_id` | uuid? | Optional |
| `created_at/by`, `updated_at/by` | audit | Required |

### 17.4 `employee_offboarding_tasks`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `offboarding_instance_id` | uuid | FK |
| `task_title` | varchar | Required |
| `owner_user_id` | uuid? | FK |
| `status` | varchar | Required |
| `due_date` | date? | Optional |
| `completed_at/by` | audit | Optional |

### 17.5 `exit_interviews`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `offboarding_instance_id` | uuid | FK |
| `employee_id` | uuid | FK |
| `interviewer_user_id` | uuid? | FK |
| `scheduled_at` | timestamptz? | Optional |
| `completed_at` | timestamptz? | Optional |
| `feedback_json` | jsonb | Optional |
| `document_id` | uuid? | Optional |

---

## 18. Audit, Versioning, and Reporting

### 18.1 Audit

Use the existing `audit_logs` table for generic audit records. Add module-specific audit events for:

1. Employee personal detail changes
2. Compensation changes
3. Bank account changes
4. Document upload, verification, download, archive
5. Leave approval and balance adjustment
6. Attendance regularization
7. Payroll run approval, lock, and publish
8. Performance review completion
9. Compliance verification
10. Offboarding completion

### 18.2 Versioned Entities

The following entities should be versioned or immutable:

| Entity | Strategy |
|--------|----------|
| Employee documents | `employee_document_versions` |
| Custom forms | version number on form definitions |
| Salary structures | effective dating |
| Employee salary assignments | effective dating |
| Payroll run results | immutable after lock |
| Leave policies | effective dating |
| Attendance records | corrections through regularization, not silent overwrite |
| Compliance requirements | effective dating if regulatory rules change |

### 18.3 Reporting Read Models

For analytics at scale, add read models or materialized views later:

1. `employee_headcount_summary`
2. `employee_attrition_summary`
3. `attendance_daily_summary`
4. `leave_balance_summary`
5. `payroll_monthly_summary`
6. `compliance_expiry_summary`
7. `training_completion_summary`

---

## 19. Cross-Cutting Index Rules

Every high-volume HR table should have:

1. `INDEX (tenant_id, deleted_at, created_at DESC, id)` where soft delete applies
2. `INDEX (tenant_id, employee_id)` for employee child tables
3. Cursor-compatible indexes for list APIs
4. Date indexes for reporting and dashboards
5. Partial unique indexes for active records where soft delete applies

Examples:

```sql
CREATE UNIQUE INDEX employees_active_code_unique
ON employees (tenant_id, employee_code)
WHERE deleted_at IS NULL;

CREATE INDEX employees_tenant_created_cursor_idx
ON employees (tenant_id, deleted_at, created_at DESC, id);

CREATE INDEX employee_documents_expiry_idx
ON employee_documents (tenant_id, expires_at)
WHERE deleted_at IS NULL;

CREATE INDEX attendance_records_date_idx
ON attendance_records (tenant_id, attendance_date, employee_id);
```

---

## 20. Implementation Phasing

### Phase 1: HR Foundation

1. Refactor `employees`
2. Add personal details, addresses, emergency contacts, bank accounts, identifiers
3. Add position/status/compensation history
4. Add cursor pagination and search APIs

### Phase 2: Documents and Customization

1. Add document categories, documents, versions, access logs
2. Add custom fields and forms
3. Add audit hooks for sensitive HR changes

### Phase 3: Lifecycle Modules

1. Recruitment
2. Onboarding
3. Offboarding

### Phase 4: Core HR Operations

1. Attendance and shifts
2. Leave management
3. Payroll foundation

### Phase 5: Talent and Compliance

1. Performance reviews
2. Training and certifications
3. Compliance requirements and employee compliance records

### Phase 6: Reporting and Optimization

1. Add materialized views/read models
2. Add archival policies for high-volume records
3. Add partitioning review for attendance, audit logs, payroll details, and document access logs

---

## 21. Open Decisions

1. Should employee identifiers be encrypted at application level, database level, or both?
2. Should documents support candidate documents separately, or reuse a generic document entity with `owner_type` and `owner_id`?
3. Should payroll be implemented in-house or integrated with an external payroll provider first?
4. Should attendance punch ingestion be real-time, batch, or both?
5. Should custom fields be available in reporting from day one, or only after read-model generation?
6. Which HR modules are required for the first commercial release?

---

## 22. Recommendation

Start with a strong HR foundation before building the full module set. The minimum scalable foundation is:

1. Refactored employee root table
2. Employee profile child tables
3. Employee history tables
4. Document management
5. Custom fields and forms
6. Tenant-safe indexes and partial unique constraints
7. Audit coverage for sensitive HR operations

Once this foundation is in place, attendance, leave, payroll, performance, training, compliance, and offboarding can be added without forcing repeated redesign of the employee master schema.
