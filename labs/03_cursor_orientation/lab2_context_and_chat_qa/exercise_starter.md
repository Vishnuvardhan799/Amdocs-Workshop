# Instructor Reference Solution — Practical 3.2: Context Engineering & Chat Q&A

**Module**: Module 3 - Cursor Orientation  
**Lab**: 3.2 Context Engineering, Scoped Q&A & Rule Enforcement  
**Standard Evaluation Benchmark**: 100/100

---

## 1. Grounded Scoped Prompt & Diagnostic

### Grounded Prompt:
```text
@src/core/services/task.service.ts @src/api/routes/tasks.ts @src/core/models/task.model.ts
Trace the data flow in `task.service.ts:updateTaskStatus`.
Why does updating the status of an existing task occasionally overwrite or clear out `completed_at` timestamps or nested `tags`?
Point to the specific line of code causing this bug and explain the atomic fix.
```

### Ground-Truth Bug Analysis:
In `task.service.ts` around line 48:
```typescript
// DEFECTIVE IMPLEMENTATION:
const existing = this.tasks.get(id);
if (!existing) return null;

// BUG: Destructive shallow overwrite resets existing fields not provided in the patch payload:
const updated: TaskRecord = {
  ...payload, // If payload only contains { status: 'DONE' }, all other original task fields like tags, description, created_at, completed_at are lost!
  id: existing.id,
  updated_at: new Date().toISOString(),
};
```
**The Fix**:
```typescript
const updated: TaskRecord = {
  ...existing, // Preserve existing fields first
  ...payload,  // Apply changes
  completed_at: payload.status === TaskStatus.COMPLETED ? (existing.completed_at || new Date().toISOString()) : existing.completed_at,
  updated_at: new Date().toISOString(),
};
```

---

## 2. Reference `.cursor/rules/*.mdc` Specifications

### File: `.cursor/rules/01-error-handling.mdc`
```markdown
---
description: Global Error Handling, Structured Logging, and API Envelopes
globs: ["src/**/*.ts"]
alwaysApply: true
---

# Global Error Handling & Logging Standards:
1. **Never use `console.log`, `console.error`, or `console.warn`**.
   - Always import and use `{ logger }` from `../../utils/logger` (or `@utils/logger`).
2. **Standard API Response Envelope**:
   All HTTP responses must strictly conform to:
   ```typescript
   interface ApiResponse<T> {
     success: boolean;
     data: T | null;
     error: {
       code: string;
       message: string;
       details?: Record<string, unknown>;
     } | null;
   }
   ```
3. **Route Handlers**:
   - Wrap async Express route logic in `try/catch` and pass unhandled exceptions to `next(err)`.
   - Always assign explicit HTTP status codes (200, 201, 400, 404, 429, 500).
```

### File: `.cursor/rules/02-security-boundary.mdc`
```markdown
---
description: Cryptographic & Data Security Boundary Constraints
globs: ["src/**/*.ts"]
alwaysApply: true
---

# Security Boundaries:
1. **Secure Cryptographic Primitives**:
   - Never use `Math.random()` for token, session, or secret generation.
   - Always use `crypto.randomBytes(32).toString('hex')` from Node's built-in `crypto` module.
2. **Input Sanitization**:
   - Validate and sanitize all user-supplied route parameters (`id`, `email`, `query`).
   - For string searches or database queries, use parameterized criteria; never concatenate raw strings.
3. **Data Protection**:
   - Never log plain-text passwords, tokens, or PII into loggers or response errors.
```
