# Bug Report: Response Interceptor Destroys Arrays

## Description
A recent commit (`feat(auth): change-password returns structured response with custom message`) updated the global `ResponseInterceptor` in the backend (`src/common/interceptors/response.interceptor.ts`). 

While this change successfully extracts the `message` from objects, it inadvertently breaks any endpoints that return arrays (like `/api/v1/notifications` and `/api/v1/tasks`). 

## Root Cause
In JavaScript, arrays are objects (`typeof [] === 'object'`). The new interceptor logic applies object destructuring on arrays:
```typescript
const isObject = data !== null && typeof data === 'object';
const message = isObject && 'message' in data ? (data as any).message : 'OK';
const { message: _, ...rest } = isObject ? (data as any) : {};
const payload = isObject && Object.keys(rest).length > 0 ? rest : null;
return { success: true, message, data: payload };
```
When `data` is an array (e.g., `[{ id: 1 }, { id: 2 }]`), the `{ message: _, ...rest }` destructuring transforms the array into an object with stringified indices as keys:
`rest` becomes `{ "0": { id: 1 }, "1": { id: 2 } }`.
This causes the frontend `useNotifications` and `DashboardView` hooks to crash or silently fail since they expect `data` to be an iterable array, but instead receive an object, throwing `notifications.slice is not a function`.

## Required Fix
The backend team needs to update `src/common/interceptors/response.interceptor.ts` to properly handle arrays. You can check if `data` is an array using `Array.isArray(data)` before applying the object restructuring.

### Suggested Implementation:
```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ResponseInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      map((data) => {
        // Handle Arrays specifically so they don't get destructured
        if (Array.isArray(data)) {
          return { success: true, message: 'OK', data };
        }

        const isObject = data !== null && typeof data === 'object';
        const message =
          isObject && 'message' in data ? (data as any).message : 'OK';
        const { message: _, ...rest } = isObject ? (data as any) : {};
        const payload = isObject && Object.keys(rest).length > 0 ? rest : null;
        
        return { success: true, message, data: payload };
      }),
    );
  }
}
```

## Impact
- **Severity**: High
- **Components Affected**: Any GET endpoint returning an array (Notifications, Tasks, Users list, etc.)
- **Symptoms**: Frontend crashes with `TypeError: .slice is not a function` or UI fails to render list data properly.
