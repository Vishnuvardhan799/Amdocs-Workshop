# Instructor Reference Solution — Practical 3.1: Inline AI Engineering

**Module**: Module 3 - Cursor Orientation  
**Lab**: 3.1 Inline AI Engineering & Refactoring  
**Standard Evaluation Benchmark**: 100/100

---

## 1. Optimal Ghost Text Shaping (`task.service.ts`)

```typescript
  /**
   * Pure TypeScript implementation without third-party dependencies.
   * Includes null/undefined checks for sinceDate.
   */
  public filterTasksByPriorityAndDate(
    tasks: TaskRecord[],
    priority?: TaskPriority,
    sinceDate?: string | Date
  ): TaskRecord[] {
    const targetTimestamp = sinceDate ? new Date(sinceDate).getTime() : 0;

    return tasks.filter((task) => {
      const matchesPriority = priority ? task.priority === priority : true;
      const createdAtTimestamp = new Date(task.created_at).getTime();
      const matchesDate = !isNaN(targetTimestamp) && targetTimestamp > 0 
        ? createdAtTimestamp >= targetTimestamp 
        : true;

      return matchesPriority && matchesDate;
    });
  }
```

---

## 2. Master `Cmd/Ctrl+K` Inline Prompt (`rate-limiter.ts`)

### Golden Standard Prompt:
[PASTE YOUR INLINE PROMPT HERE]
```text
Implement an in-memory sliding-window rate limiter (limit 100 requests per 60,000ms window per client IP).
Maintain client timestamps in an in-memory Map<string, number[]>.
Prune timestamps older than 60s on each request to prevent memory leaks.
If limit is exceeded, log a warning with logger.warn and return HTTP 429 using standard JSON error envelope:
{ "success": false, "data": null, "error": { "code": "RATE_LIMIT_EXCEEDED", "message": "Too many requests" } }.
Otherwise, call next(). Do not use any external npm dependencies.
```

### Reference Implementation in `src/api/middleware/rate-limiter.ts`:
```typescript
import { Request, Response, NextFunction } from "express";
import { logger } from "../../utils/logger";

interface ClientRateLimitState {
  timestamps: number[];
}

const rateLimitStore = new Map<string, ClientRateLimitState>();
const WINDOW_MS = 60 * 1000;
const MAX_REQUESTS = 100;

export function applyRateLimiting(req: Request, res: Response, next: NextFunction): void {
  const clientIp = req.ip || req.socket.remoteAddress || "unknown_client";
  const now = Date.now();

  let state = rateLimitStore.get(clientIp);
  if (!state) {
    state = { timestamps: [] };
    rateLimitStore.set(clientIp, state);
  }

  // Prune timestamps older than current sliding window
  state.timestamps = state.timestamps.filter((ts) => now - ts < WINDOW_MS);

  if (state.timestamps.length >= MAX_REQUESTS) {
    logger.warn(`[RateLimiter] Rate limit exceeded for IP: ${clientIp}`);
    res.status(429).json({
      success: false,
      data: null,
      error: {
        code: "RATE_LIMIT_EXCEEDED",
        message: "Too many requests. Please try again later."
      }
    });
    return;
  }

  state.timestamps.push(now);
  next();
}
```

### Completed Diff Audit Checklist:
- [x] Preserved function signature `(req: Request, res: Response, next: NextFunction)`
- [x] Used in-memory sliding window algorithm (timestamp array or bucket)
- [x] Returned HTTP 429 using project standard envelope `{ success: false, data: null, error: { code: "RATE_LIMIT_EXCEEDED", message: "..." } }`
- [x] Used `logger.warn` instead of `console.log`
- [x] Cleaned up stale entries to prevent memory leak

---

## 3. Email Sanitization Reference (`src/utils/crypto.ts`)

### Inline Prompt:
```text
Refactor sanitizeUserEmail: trim whitespace, convert domain part to lowercase, validate basic RFC 5322 pattern. Return sanitized string or throw Error with code "INVALID_EMAIL_FORMAT".
```

### Reference Solution:
```typescript
export function sanitizeUserEmail(rawEmail: string): string {
  if (!rawEmail || typeof rawEmail !== "string") {
    throw new Error("INVALID_EMAIL_FORMAT: Email must be a non-empty string.");
  }
  const trimmed = rawEmail.trim();
  const parts = trimmed.split("@");
  if (parts.length !== 2 || !parts[0] || !parts[1]) {
    throw new Error("INVALID_EMAIL_FORMAT: Malformed email structure.");
  }
  const local = parts[0];
  const domain = parts[1].toLowerCase();
  const emailRegex = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]+)*$/;
  const full = `${local}@${domain}`;
  if (!emailRegex.test(full)) {
    throw new Error("INVALID_EMAIL_FORMAT: Email does not match RFC 5322 regex.");
  }
  return full;
}
```
