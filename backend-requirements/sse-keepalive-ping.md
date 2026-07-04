# Requirement: Add Keep-Alive Ping to SSE Stream

## Issue Description
The frontend application continuously logs `net::ERR_INCOMPLETE_CHUNKED_ENCODING 200 (OK)` and triggers an `SSE Error, reconnecting...` in the console from the `NotificationsContext`. 

This occurs because the SSE stream established at `/api/v1/notifications/stream` sits idle when there are no new notifications. Many proxies (like Nginx, AWS ALBs) and network intermediaries have default idle timeouts (e.g., 60 seconds). When the connection sits idle without any data transfer, the proxy abruptly closes the connection, terminating the chunked transfer stream improperly.

## Required Backend Change
To prevent the connection from going idle and being killed prematurely, the backend SSE stream must emit a heartbeat or "keep-alive" ping at a regular interval (e.g., every 30 to 45 seconds).

### Suggested Implementation
In `boblte-server/src/modules/ops/notifications/notifications.controller.ts` (or the underlying `notification.service.ts`), you can merge a periodic ping event into the main RxJS stream.

```typescript
import { merge, interval } from 'rxjs';
import { map } from 'rxjs/operators';

// In NotificationService or Controller where the stream$ is built:
const ping$ = interval(30000).pipe(
  map(() => ({ data: { type: 'ping' } } as MessageEvent))
);

// Merge with the actual notification stream
return merge(this.notificationService.connect(userId), ping$);
```

*(Alternatively, emitting a simple SSE comment like `: ping\n\n` is also valid and standard, depending on how the NestJS SSE serialization is configured).*

## Impact
This change will keep the TCP connection active, preventing `ERR_INCOMPLETE_CHUNKED_ENCODING` errors and significantly reducing the frequency of frontend reconnection attempts and console spam.
