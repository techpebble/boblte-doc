# Bug Report: Business Unit `parentBusinessUnitId` and `isActive` Update Failure

## Issue Description
When a client application attempts to update the `parentBusinessUnitId` (or `isActive` status) of an existing `BusinessUnit` using the `PATCH /org/business-units/:id` endpoint, the update silently fails. The backend ignores the fields, and the changes are not persisted to the database.

## Root Cause Analysis
The issue resides in two places within the backend `boblte-server`:

1. **DTO Definition (`UpdateBusinessUnitDto`)**:
   The Data Transfer Object `UpdateBusinessUnitDto` (located in `src/modules/org/business-units/dto/request/update-business-unit.dto.ts`) does not include the `parentBusinessUnitId` or `isActive` properties.
   ```typescript
   export class UpdateBusinessUnitDto {
     @ApiPropertyOptional({ example: 'Asia Pacific Region' })
     @IsOptional()
     @IsString()
     @IsNotEmpty()
     unitName?: string;

     @ApiPropertyOptional({ type: 'object', additionalProperties: true })
     @IsOptional()
     @IsObject()
     customAttributes?: Record<string, unknown>;
   }
   ```
   Because these fields are missing, the ValidationPipe likely strips them out of the request payload before they even reach the controller (if `whitelist: true` is enabled).

2. **Use Case Execution (`UpdateBusinessUnitUseCase`)**:
   Even if the fields were present in the DTO, the `UpdateBusinessUnitUseCase` (located in `src/modules/org/business-units/application/use-cases/update-business-unit.usecase.ts`) explicitly ignores them when passing data to the repository:
   ```typescript
   // src/modules/org/business-units/application/use-cases/update-business-unit.usecase.ts
   return this.repo.update(id, {
     unitName: dto.unitName,
     customAttributes: dto.customAttributes,
     // BUG: parentBusinessUnitId and isActive are completely omitted here!
   });
   ```

## Required Backend Fixes
To resolve this issue, the backend development team needs to:

1. **Update `UpdateBusinessUnitDto`:**
   Add the missing optional fields to allow clients to send them.
   ```typescript
   export class UpdateBusinessUnitDto {
     // ... existing fields ...

     @ApiPropertyOptional({ example: '123e4567-e89b-12d3-a456-426614174000' })
     @IsOptional()
     @IsString()
     parentBusinessUnitId?: string | null;

     @ApiPropertyOptional({ example: true })
     @IsOptional()
     @IsBoolean()
     isActive?: boolean;
   }
   ```

2. **Update `UpdateBusinessUnitUseCase`:**
   Pass the fields down to the repository interface. (Note: the `IBusinessUnitRepository` already supports `parentBusinessUnitId` in its `UpdateBusinessUnitInput` type).
   ```typescript
   return this.repo.update(id, {
     unitName: dto.unitName,
     parentBusinessUnitId: dto.parentBusinessUnitId, // ADDED
     isActive: dto.isActive,                         // ADDED
     customAttributes: dto.customAttributes,
   });
   ```

## Frontend Status
The frontend (`boblte-webapp`) has been updated to explicitly send `parentBusinessUnitId: null` in the payload when a user selects "-- No Parent (Root Level) --". Once the backend is patched, the UI will correctly detach Business Units from their parents.
