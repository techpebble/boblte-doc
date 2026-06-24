# Scalable Notification Engine Design

**Version**: 0.1.0
**Last Updated**: 2026-06-24
**Status**: Draft
**Owner**: Backend Team
**Depends On**: `08-database-design.md`, `12-task-schema-redesign.md`, `15-hrm-schema-design.md`

---

## 1. Objective

Upgrade the notification system from an in-process MVP implementation to a reliable, multi-tenant, horizontally scalable ERP notification engine.

The target notification engine must support:

1. Durable notification creation
2. High-volume event fan-out
3. Cursor-based inbox APIs
4. Real-time delivery across multiple backend instances
5. User preferences
6. Notification templates
7. Delivery tracking
8. Retry and dead-letter handling
9. Future channels such as email, SMS, mobile push, WhatsApp, and digest emails
10. Retention, archival, cleanup, and observability

---

## 2. Current Limitations

The current implementation is good enough for MVP usage, but it is not sufficient for an ERP-scale SaaS platform.

Known limitations:

1. Notification listing is unbounded.
2. Existing indexes do not match inbox query patterns.
3. Real-time delivery uses in-memory per-instance SSE subjects.
4. Notification creation depends on synchronous in-process events.
5. There is no durable outbox.
6. There is no retry or dead-letter handling.
7. Fan-out inserts use one insert per recipient.
8. One user can only have one active SSE connection.
9. There is no notification preference model.
10. There is no delivery tracking for external channels.
11. There is no archival or retention job.

---

## 3. Target Architecture

```text
Business Module
  ↓ writes event in same DB transaction
Event Outbox
  ↓ consumed by worker
Notification Worker
  ↓ resolves recipients and renders templates
Notifications Table
  ↓
Delivery Records
  ↓
Redis Pub/Sub or Redis Streams
  ↓
API Instances
  ↓
SSE/WebSocket Clients
```

The database remains the source of truth. Real-time delivery is an enhancement, not the authoritative storage mechanism.

---

## 4. Phase 1: Fix Current Data Access

### 4.1 Cursor Pagination

Current API:

```http
GET /notifications
```

Target API:

```http
GET /notifications?limit=30&cursor=&status=unread&archived=false
```

Target response:

```json
{
  "items": [],
  "nextCursor": "..."
}
```

Recommended query parameters:

| Parameter | Notes |
|-----------|-------|
| `limit` | Max page size. Default 30, max 100 |
| `cursor` | Cursor from previous page |
| `archived` | `true` / `false` |
| `status` | `all`, `read`, `unread` |
| `type` | Optional notification type |
| `referenceType` | Optional reference type filter |

Use stable cursor ordering:

```text
ORDER BY created_at DESC, id DESC
```

### 4.2 Query-Aligned Indexes

Recommended indexes:

```sql
CREATE INDEX notifications_inbox_cursor_idx
ON notifications (tenant_id, recipient_id, is_archived, created_at DESC, id DESC);

CREATE INDEX notifications_unread_count_idx
ON notifications (tenant_id, recipient_id, is_read)
WHERE is_read = false;

CREATE INDEX notifications_reference_idx
ON notifications (tenant_id, reference_type, reference_id);

CREATE INDEX notifications_archive_cleanup_idx
ON notifications (tenant_id, archived_at);
```

### 4.3 Tenant-Safe Updates

Update notification mutations to use tenant and recipient filters directly.

Avoid:

```ts
await findById(id, tenantId, recipientId);
await updateById(id);
```

Prefer:

```ts
await updateMany({
  where: { id, tenantId, recipientId },
  data: { isRead: true, readAt: new Date() },
});
```

This removes the race between ownership check and mutation, and keeps all notification writes tenant-safe.

---

## 5. Phase 2: Durable Domain Event Outbox

### 5.1 `event_outbox`

Business modules should write domain events into an outbox table, preferably in the same transaction as the business change.

Suggested schema:

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `event_type` | text | Example: `task.assigned` |
| `aggregate_type` | text | Example: `TASK` |
| `aggregate_id` | uuid? | Business entity ID |
| `payload` | jsonb | Event payload |
| `status` | text | `PENDING`, `PROCESSING`, `PROCESSED`, `FAILED`, `DEAD_LETTER` |
| `attempts` | int | Default 0 |
| `available_at` | timestamptz | Default now |
| `locked_at` | timestamptz? | Worker lock timestamp |
| `locked_by` | text? | Worker ID |
| `processed_at` | timestamptz? | Completed timestamp |
| `failed_at` | timestamptz? | Failure timestamp |
| `error_message` | text? | Last failure |
| `created_at` | timestamptz | Default now |

Recommended indexes:

```sql
CREATE INDEX event_outbox_pending_idx
ON event_outbox (status, available_at, created_at);

CREATE INDEX event_outbox_tenant_event_idx
ON event_outbox (tenant_id, event_type, created_at DESC);
```

### 5.2 Event Writing Pattern

Current pattern:

```ts
eventBus.emit('task.assigned', payload);
```

Target pattern:

```ts
await outbox.record('task.assigned', payload);
```

Critical events should be written transactionally with the business state change.

---

## 6. Phase 3: Notification Worker

Create a background worker responsible for converting outbox events into notifications.

Worker responsibilities:

1. Poll or subscribe to `event_outbox`.
2. Lock rows safely using `FOR UPDATE SKIP LOCKED`.
3. Resolve recipients.
4. Deduplicate recipients.
5. Render notification templates.
6. Insert notifications in batches.
7. Create delivery records.
8. Publish real-time delivery events.
9. Retry failed events with exponential backoff.
10. Move permanently failed events to `DEAD_LETTER`.

Recommended processing statuses:

| Status | Meaning |
|--------|---------|
| `PENDING` | Waiting to be processed |
| `PROCESSING` | Claimed by a worker |
| `PROCESSED` | Completed successfully |
| `FAILED` | Failed but retryable |
| `DEAD_LETTER` | Failed permanently or exceeded retry limit |

### 6.1 Idempotency

Add `dedupe_key` to notifications.

Recommended unique index:

```sql
CREATE UNIQUE INDEX notifications_dedupe_unique
ON notifications (tenant_id, recipient_id, dedupe_key)
WHERE dedupe_key IS NOT NULL;
```

This prevents duplicate notifications when workers retry.

---

## 7. Phase 4: Notification Schema Redesign

### 7.1 `notifications`

Recommended extended schema:

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `recipient_id` | uuid | FK to users |
| `title` | text | Required |
| `body` | text? | Optional |
| `type` | enum | `INFO`, `WARNING`, `ACTION_REQUIRED`, `SUCCESS` |
| `category` | text? | Example: `TASK`, `PAYROLL`, `LEAVE`, `HR` |
| `priority` | text? | `LOW`, `NORMAL`, `HIGH`, `CRITICAL` |
| `reference_type` | text? | Business entity type |
| `reference_id` | uuid? | Business entity ID |
| `metadata` | jsonb | Default `{}` |
| `dedupe_key` | text? | Idempotency key |
| `is_read` | boolean | Default false |
| `read_at` | timestamptz? | Optional |
| `is_archived` | boolean | Default false |
| `archived_at` | timestamptz? | Optional |
| `expires_at` | timestamptz? | Optional |
| `created_at` | timestamptz | Default now |
| `created_by` | uuid? | Actor user ID |

### 7.2 `notification_deliveries`

Track delivery attempts per channel.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `notification_id` | uuid | FK to notifications |
| `recipient_id` | uuid | FK to users |
| `channel` | text | `IN_APP`, `SSE`, `EMAIL`, `SMS`, `PUSH`, `WHATSAPP` |
| `status` | text | `PENDING`, `SENT`, `DELIVERED`, `FAILED`, `SKIPPED` |
| `provider` | text? | External provider name |
| `provider_message_id` | text? | External message ID |
| `attempts` | int | Default 0 |
| `last_attempt_at` | timestamptz? | Optional |
| `delivered_at` | timestamptz? | Optional |
| `failed_at` | timestamptz? | Optional |
| `error_message` | text? | Optional |
| `created_at` | timestamptz | Default now |

Recommended indexes:

```sql
CREATE INDEX notification_deliveries_notification_idx
ON notification_deliveries (tenant_id, notification_id);

CREATE INDEX notification_deliveries_pending_idx
ON notification_deliveries (channel, status, created_at);
```

---

## 8. Phase 5: Distributed Real-Time Delivery

Replace local-only SSE with distributed pub/sub.

Recommended options:

1. Redis Pub/Sub
2. Redis Streams
3. NATS
4. Kafka
5. Postgres `LISTEN/NOTIFY` for smaller scale only

Recommended first step: Redis Pub/Sub or Redis Streams.

Flow:

```text
Worker creates notification
        ↓
Worker publishes notification.created:{recipientId}
        ↓
All API instances subscribe
        ↓
Instance with active user connection pushes SSE/WebSocket event
```

### 8.1 Multiple Connections Per User

Current service supports one active connection per user. Replace it with:

```ts
Map<userId, Map<connectionId, Subject>>
```

This supports:

1. Multiple browser tabs
2. Multiple devices
3. Mobile and web clients
4. Reconnects
5. Heartbeat and cleanup

### 8.2 Missed Notification Recovery

Real-time delivery should not be the source of truth.

On reconnect, clients should call:

```http
GET /notifications?limit=30&cursor=
```

Optionally support:

```http
GET /notifications?after=lastSeenNotificationId
```

---

## 9. Phase 6: Templates and Preferences

### 9.1 `notification_templates`

Tenant-customizable templates.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid? | Null for platform default |
| `event_type` | text | Required |
| `channel` | text | Required |
| `title_template` | text | Required |
| `body_template` | text? | Optional |
| `locale` | text | Default `en` |
| `is_active` | boolean | Default true |
| `created_at` | timestamptz | Default now |
| `updated_at` | timestamptz | Required |

### 9.2 `notification_preferences`

Per-user notification preferences.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `user_id` | uuid | Required |
| `event_type` | text | Required |
| `channel` | text | Required |
| `is_enabled` | boolean | Default true |
| `created_at` | timestamptz | Default now |
| `updated_at` | timestamptz | Required |

Recommended unique index:

```sql
CREATE UNIQUE INDEX notification_preferences_unique
ON notification_preferences (tenant_id, user_id, event_type, channel);
```

### 9.3 Digest Preferences

Optional later table:

| Column | Type | Notes |
|--------|------|-------|
| `tenant_id` | uuid | Required |
| `user_id` | uuid | Required |
| `digest_frequency` | text | `NONE`, `DAILY`, `WEEKLY` |
| `quiet_hours_start` | time? | Optional |
| `quiet_hours_end` | time? | Optional |
| `timezone` | text | Required |

---

## 10. Phase 7: Bulk and Broadcast Notifications

### 10.1 `notification_batches`

Use batches for high-volume fan-out, such as announcements, payroll publishing, compliance reminders, policy updates, and workflow escalations.

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `tenant_id` | uuid | Required |
| `event_type` | text | Required |
| `status` | text | Required |
| `total_recipients` | int | Default 0 |
| `processed_recipients` | int | Default 0 |
| `failed_recipients` | int | Default 0 |
| `created_at` | timestamptz | Default now |
| `completed_at` | timestamptz? | Optional |

### 10.2 Fan-Out Rules

1. Deduplicate recipient IDs before enqueueing.
2. Process in chunks.
3. Use `createMany` for in-app notification rows.
4. Use delivery worker queues for external channels.
5. Avoid unbounded `Promise.all`.

Recommended first batch size:

```ts
const batchSize = 500;
```

---

## 11. Phase 8: Retention, Archival, and Cleanup

Recommended defaults:

| Data | Retention |
|------|-----------|
| Unread in-app notifications | Until read or expiry |
| Read notifications | Archive after 90 days |
| Archived notifications | Delete after 1 year |
| Delivery logs | 90-180 days |
| Processed outbox rows | 30-90 days |
| Dead-letter events | Retain until manually resolved |

Cleanup jobs:

1. Expire old notifications.
2. Archive old read notifications.
3. Delete old archived notifications.
4. Delete old processed outbox rows.
5. Alert when dead-letter events exceed threshold.

---

## 12. Phase 9: Observability and Operations

### 12.1 Metrics

Track:

1. Pending outbox count
2. Failed outbox count
3. Dead-letter count
4. Notification creation latency
5. Notification delivery latency
6. Active SSE users
7. Active SSE connections
8. Delivery failures by channel and provider
9. Retry count
10. Worker processing throughput

### 12.2 Internal Admin Endpoints

For internal admins only:

```http
GET /internal/notifications/outbox
GET /internal/notifications/dead-letter
POST /internal/notifications/dead-letter/:id/retry
```

### 12.3 Structured Logging

Logs should include:

1. `tenantId`
2. `eventType`
3. `notificationId`
4. `recipientId`
5. `channel`
6. `attempt`
7. `workerId`
8. `correlationId`

---

## 13. Recommended Implementation Order

1. Add notification pagination and query-aligned indexes.
2. Make notification mutations tenant-safe.
3. Add `event_outbox`.
4. Add outbox writer service.
5. Replace direct `eventBus.emit` calls in critical flows with outbox writes.
6. Add notification worker with retry and dead-letter handling.
7. Add batched notification insertion.
8. Add Redis-based distributed real-time delivery.
9. Support multiple SSE connections per user.
10. Add notification templates.
11. Add notification preferences.
12. Add delivery tracking for email, SMS, push, and WhatsApp.
13. Add retention and cleanup jobs.
14. Add metrics and internal operational tooling.

---

## 14. Initial MVP Upgrade Scope

The first scalable upgrade should include:

1. Cursor-paginated inbox API
2. Inbox/unread indexes
3. Tenant-safe mark-read/archive updates
4. Event outbox table
5. Notification worker for in-app notifications
6. Deduplication key
7. Batched inserts
8. Multiple SSE connections per user
9. Redis Pub/Sub for cross-instance push

External channels can be added after the in-app engine becomes reliable.

---

## 15. Summary

The scalable notification engine should move notification creation and delivery out of synchronous application request paths and into a durable, worker-driven architecture.

The database should store the notification inbox as the source of truth. The outbox should guarantee that business events are not lost. Workers should handle fan-out, retries, templates, and delivery records. Redis or another pub/sub layer should distribute live notifications across horizontally scaled API instances.

This architecture allows the ERP platform to support high-volume tenant activity without losing notifications, blocking business requests, or coupling delivery to a single backend process.
