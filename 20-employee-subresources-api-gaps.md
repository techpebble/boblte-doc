# Requirement: Employee Sub-Resources API Gaps

**Status**: Proposed / Pending Review
**Target Module**: `boblte-server/src/modules/hr/employees`
**Urgency**: High (Blocks full frontend HRM CRUD capabilities)

## 1. Overview
The frontend `boblte-webapp` is currently implementing the full **HRM/Employee Directory** dashboard, which manages not only basic employee records but also their sub-resources:
- Addresses
- Bank Accounts
- Emergency Contacts
- Dependents
- Identifiers

While `Dependents` and `Identifiers` have complete CRUD APIs, the resources for `Addresses`, `Bank Accounts`, and `Emergency Contacts` are missing **Listing** (`GET`) and **Deletion** (`DELETE`) routes on the backend. Additionally, there is a limitation in modifying job details via the standard `PATCH` route.

---

## 2. API Endpoints Requested

### 2.1 Employee Addresses
We require the ability to list all addresses for a specific employee and delete a specific address.

* **List Addresses**
  * **Path**: `GET /api/v1/employees/:id/addresses`
  * **Response**: `ApiResponse<EmployeeAddressResponseDto[]>`
  
* **Delete Address**
  * **Path**: `DELETE /api/v1/employees/:id/addresses/:addressId`
  * **Response**: `204 No Content` / `ApiResponse<{ success: boolean }>`

---

### 2.2 Employee Bank Accounts
We require the ability to list all bank accounts for a specific employee and delete a specific bank account.

* **List Bank Accounts**
  * **Path**: `GET /api/v1/employees/:id/bank-accounts`
  * **Response**: `ApiResponse<EmployeeBankAccountResponseDto[]>`
  
* **Delete Bank Account**
  * **Path**: `DELETE /api/v1/employees/:id/bank-accounts/:accountId`
  * **Response**: `204 No Content` / `ApiResponse<{ success: boolean }>`

---

### 2.3 Employee Emergency Contacts
We require the ability to list all emergency contacts for a specific employee and delete a specific emergency contact.

* **List Emergency Contacts**
  * **Path**: `GET /api/v1/employees/:id/emergency-contacts`
  * **Response**: `ApiResponse<EmployeeEmergencyContactResponseDto[]>`
  
* **Delete Emergency Contact**
  * **Path**: `DELETE /api/v1/employees/:id/emergency-contacts/:contactId`
  * **Response**: `204 No Content` / `ApiResponse<{ success: boolean }>`

---

### 2.4 Update Employee Core Job Properties
Currently, `PATCH /api/v1/employees/:id` validates payload against `UpdateEmployeeDto`, which only exposes personal details:
* `legalFirstName`, `legalMiddleName`, `legalLastName`, `preferredName`, `workEmail`, `personalEmail`, `mobileNumber`, `dateOfBirth`, `gender`

We require an endpoint to update job details such as department, designation, business unit, position, and reporting manager:
* **Option A**: Expand `UpdateEmployeeDto` to accept:
  * `departmentId`, `designationId`, `businessUnitId`, `positionId`, `managerEmployeeId`, `locationId`, `workerType`, `employmentStatus`, `employmentType`, `employeeCode`, `dateOfJoining`, `dateOfExit`
* **Option B**: Create a job details update route:
  * **Path**: `PATCH /api/v1/employees/:id/job-info`
  * **Payload**: `UpdateEmployeeJobInfoDto` (with the fields above)

---

### 2.5 Toggle Employee Status
The existing webapp client refers to a `/toggle-status` route:
* **Path**: `PATCH /api/v1/employees/:id/toggle-status`
* **Payload**: `{ isActive: boolean }`
* **Action**: Updates employee active state or transitions `employmentStatus` (e.g. `SUSPENDED` if active is false, `ACTIVE` if active is true).
