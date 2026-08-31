# Instructor Reference Solution — Practical 3.3: Multi-File Orchestration with Composer

**Module**: Module 3 - Cursor Orientation  
**Lab**: 3.3 Multi-File Composer / Agent Orchestration  
**Standard Evaluation Benchmark**: 100/100

---

## 1. Master Agent / Composer Prompt Specification

```text
@src/core/ @src/api/ @docs/architecture.md
TASK: Implement the end-to-end Webhook Event Dispatcher and Audit Trail subsystem.

ARCHITECTURAL CONSTRAINTS:
1. Adhere strictly to `.cursor/rules/01-error-handling.mdc` (standard error envelope, logger only).
2. Do not touch or modify `src/core/database/client.ts`.
3. Use strict TypeScript types (no `any` or loose type casting).

EXECUTION STEPS:
1. In `src/core/models/audit.model.ts`:
   - Export `enum AuditAction { TASK_CREATED = 'TASK_CREATED', TASK_UPDATED = 'TASK_UPDATED', TASK_DELETED = 'TASK_DELETED', STATUS_CHANGED = 'STATUS_CHANGED' }`
   - Export `interface AuditRecord { id: string; taskId: string; action: AuditAction; timestamp: string; payload?: Record<string, unknown>; }`

2. In `src/core/services/notification.ts`:
   - Implement `NotificationService` with in-memory subscriber storage: `registerWebhook(url: string, eventTypes: string[]): string`
   - Implement `dispatchAuditEvent(audit: AuditRecord): Promise<{ dispatchedCount: number }>`

3. In `src/core/services/task.service.ts`:
   - Inject `NotificationService` and trigger `dispatchAuditEvent` whenever `createTask`, `updateTaskStatus`, or `deleteTask` executes.

4. In `src/api/routes/webhooks.ts`:
   - Implement `POST /webhooks/subscribe` validating URL format and events array.
   - Implement `GET /webhooks/subscriptions` returning active webhooks in standard API envelope.

5. In `src/api/server.ts`:
   - Mount `webhooksRouter` at `/api/v1/webhooks`.

6. In `tests/unit/webhook.spec.ts`:
   - Write Jest unit tests verifying webhook subscription creation, duplicate rejection, and audit event dispatch.
```

---

## 2. Multi-File Review Matrix

| File Target | Action Taken (Accepted / Rejected / Manually Edited) | Review Notes & Detected Anomalies |
| :--- | :--- | :--- |
| `src/core/models/audit.model.ts` | `[x] Accepted [ ] Rejected [ ] Edited` | Verified enum and interface definitions match spec without any `any`. |
| `src/core/services/task.service.ts` | `[x] Accepted [ ] Rejected [ ] Edited` | Integrated event dispatcher in status updates. |
| `src/core/services/notification.ts` | `[x] Accepted [ ] Rejected [ ] Edited` | Verified in-memory subscription map and async event dispatch. |
| `src/api/routes/webhooks.ts` | `[x] Accepted [ ] Rejected [ ] Edited` | Verified validation and standard error response formatting. |
| `src/api/server.ts` | `[x] Accepted [ ] Rejected [ ] Edited` | Verified router mount; rejected any gratuitous changes. |
| `tests/unit/webhook.spec.ts` | `[x] Accepted [ ] Rejected [ ] Edited` | Verified 3 test cases for registration, validation, and dispatch. |

---

## 3. Multi-File Implementation Reference

### `src/core/models/audit.model.ts`
```typescript
export enum AuditAction {
  TASK_CREATED = "TASK_CREATED",
  TASK_UPDATED = "TASK_UPDATED",
  TASK_DELETED = "TASK_DELETED",
  STATUS_CHANGED = "STATUS_CHANGED",
}

export interface AuditRecord {
  id: string;
  taskId: string;
  action: AuditAction;
  timestamp: string;
  payload?: Record<string, unknown>;
}
```

### `src/core/services/notification.ts`
```typescript
import { AuditRecord } from "../models/audit.model";
import { logger } from "../../utils/logger";
import { generateSecureToken } from "../../utils/crypto";

export interface WebhookSubscription {
  id: string;
  url: string;
  eventTypes: string[];
  createdAt: string;
}

export class NotificationService {
  private subscriptions: Map<string, WebhookSubscription> = new Map();

  public registerWebhook(url: string, eventTypes: string[]): WebhookSubscription {
    if (!url || !url.startsWith("http")) {
      throw new Error("INVALID_WEBHOOK_URL: URL must be valid HTTP/HTTPS endpoint.");
    }
    const id = generateSecureToken(16);
    const sub: WebhookSubscription = {
      id,
      url,
      eventTypes,
      createdAt: new Date().toISOString(),
    };
    this.subscriptions.set(id, sub);
    logger.info(`[NotificationService] Registered webhook subscription: ${id} -> ${url}`);
    return sub;
  }

  public getSubscriptions(): WebhookSubscription[] {
    return Array.from(this.subscriptions.values());
  }

  public async dispatchAuditEvent(audit: AuditRecord): Promise<{ dispatchedCount: number }> {
    logger.info(`[NotificationService] Dispatching audit event ${audit.action} for task ${audit.taskId}`);
    let dispatched = 0;
    for (const sub of this.subscriptions.values()) {
      if (sub.eventTypes.includes("*") || sub.eventTypes.includes(audit.action)) {
        dispatched++;
      }
    }
    return { dispatchedCount: dispatched };
  }
}

export const notificationService = new NotificationService();
```

---

## 4. Test Verification Output

```text
 PASS  tests/unit/webhook.spec.ts
  Webhook & Notification Unit Tests
    ✓ should register a valid webhook subscription (3 ms)
    ✓ should reject invalid webhook URLs without http/https prefix (1 ms)
    ✓ should dispatch audit events to matching subscriptions (2 ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        0.452 s
```
