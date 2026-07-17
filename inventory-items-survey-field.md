# Survey Form Builder: "Items" Field Requirement

## Overview
The frontend requires the ability to add an "Items" (Inventory Items) field type to the Survey Form Builder (`SurveyFormBuilder.tsx`). This allows field executives to select or interact with inventory items during a survey. 

## Backend Changes Required
To fully support this in the frontend, the backend database schema and API must be updated to accept the new field type.

### 1. Database Schema
Update the `SurveyFieldType` enum in Prisma (`prisma/schema/field/field-survey-form.prisma`) to include `INVENTORY_ITEM` (or `ITEMS`).

```prisma
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
  // NEW VALUE:
  INVENTORY_ITEM
}
```

### 2. API Validation
Ensure that any DTOs validating the `fieldType` in survey form creation/updates accept the new `INVENTORY_ITEM` enum value.

### 3. Data Integration (Optional but recommended)
If the mobile app or webapp needs to fetch the list of items for this field dynamically, ensure there is an endpoint (e.g., `GET /api/inventory/items`) that returns the list of available inventory items for the tenant.

## Frontend Impact
Once the backend supports this enum value, the frontend will be updated as follows:
1. `src/core/api/surveys.ts`: Add `INVENTORY_ITEM` to the `SurveyFieldType` TypeScript type.
2. `src/features/field/components/SurveyFormBuilder.tsx`: Add the new field to the `FIELD_TYPES` array in the palette.
3. Form rendering components will be updated to display an item selector/picker when this field type is encountered.
