# Backend Requirements: Custom Field Definitions API

This document specifies the required REST API endpoints, DTO schemas, and permissions for managing Custom Field Definitions and Options. These endpoints will serve the HR settings page to allow tenant administrators to configure custom fields on entities (such as the Employee profile).

## Database Context
The API corresponds directly to the following Prisma models in `hr-custom.prisma`:
* `CustomFieldDefinition`
* `CustomFieldOption`

---

## 1. REST API Endpoints Specification

### 1.1 List Custom Field Definitions
Retrieve all custom field configurations for the tenant.

* **Method:** `GET`
* **Path:** `/api/v1/custom-field-definitions`
* **Query Parameters:**
  * `moduleName` (optional, string) - Filter by module (e.g. `hr`)
  * `entityType` (optional, string) - Filter by entity (e.g. `employee`)
  * `isActive` (optional, boolean) - Filter by active status
* **Required Permission:** `platform:settings:read` (or HR settings equivalent)
* **Response (Success - 200 OK):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "7ac9818b-5be7-4a4a-9bcf-10cfdcdbe123",
        "moduleName": "hr",
        "entityType": "employee",
        "fieldKey": "tshirt_size",
        "fieldLabel": "T-Shirt Size",
        "fieldType": "select",
        "isRequired": false,
        "isSearchable": false,
        "isReportable": true,
        "displayOrder": 1,
        "isActive": true,
        "options": [
          { "id": "option-1", "optionValue": "S", "optionLabel": "Small", "displayOrder": 0 },
          { "id": "option-2", "optionValue": "M", "optionLabel": "Medium", "displayOrder": 1 },
          { "id": "option-3", "optionValue": "L", "optionLabel": "Large", "displayOrder": 2 }
        ],
        "createdAt": "2026-07-03T10:00:00.000Z",
        "updatedAt": "2026-07-03T10:00:00.000Z"
      }
    ]
  }
  ```

---

### 1.2 Create Custom Field Definition
Create a new custom field configuration for the tenant, including options if `fieldType` is `select`.

* **Method:** `POST`
* **Path:** `/api/v1/custom-field-definitions`
* **Required Permission:** `platform:settings:write`
* **Request Payload (CreateCustomFieldDefinitionDto):**
  ```json
  {
    "moduleName": "hr",
    "entityType": "employee",
    "fieldKey": "tshirt_size",
    "fieldLabel": "T-Shirt Size",
    "fieldType": "select", // select, text, number, date, boolean
    "isRequired": false,
    "isSearchable": false,
    "isReportable": true,
    "displayOrder": 1,
    "options": [
      { "optionValue": "S", "optionLabel": "Small", "displayOrder": 0 },
      { "optionValue": "M", "optionLabel": "Medium", "displayOrder": 1 },
      { "optionValue": "L", "optionLabel": "Large", "displayOrder": 2 }
    ]
  }
  ```
* **Response (Success - 210 Created):** Returns the fully created `CustomFieldDefinition` with options list.

---

### 1.3 Update Custom Field Definition
Update an existing custom field definition's label, order, constraints, or status.

* **Method:** `PATCH`
* **Path:** `/api/v1/custom-field-definitions/:id`
* **Required Permission:** `platform:settings:write`
* **Request Payload (UpdateCustomFieldDefinitionDto):**
  ```json
  {
    "fieldLabel": "Staff T-Shirt Size",
    "isRequired": true,
    "displayOrder": 2,
    "isActive": true,
    "options": [
      // Updates existing options or replaces options array
      { "optionValue": "S", "optionLabel": "Small", "displayOrder": 0 },
      { "optionValue": "M", "optionLabel": "Medium", "displayOrder": 1 },
      { "optionValue": "L", "optionLabel": "Large", "displayOrder": 2 },
      { "optionValue": "XL", "optionLabel": "Extra Large", "displayOrder": 3 }
    ]
  }
  ```
* **Response (Success - 200 OK):** Returns the updated `CustomFieldDefinition` object.

---

### 1.4 Delete Custom Field Definition (Soft Delete)
Soft deletes a custom field configuration by setting `deletedAt` and `deletedBy`.

* **Method:** `DELETE`
* **Path:** `/api/v1/custom-field-definitions/:id`
* **Required Permission:** `platform:settings:write`
* **Response (Success - 200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "success": true
    }
  }
  ```

---

## 2. Validation & Rules
1. **Unique key constraint:** A partial unique index should enforce that `(tenantId, entityType, fieldKey)` is unique for all non-deleted fields.
2. **Key format:** `fieldKey` must match regex `/^[a-z0-9_]+$/` (only lowercase, digits, and underscores, e.g. `blood_group`).
3. **Type restrictions:** `fieldType` must be one of: `text`, `number`, `date`, `boolean`, `select`. If the type is `select`, the `options` array must contain at least one option.
