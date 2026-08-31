# Instructor Reference Solution — Practical 3.4: Adversarial Diff Review & Capstone

**Module**: Module 3 - Cursor Orientation  
**Lab**: 3.4 Adversarial Diff Review, Red-Teaming AI Code, & Capstone Project  
**Standard Evaluation Benchmark**: 100/100

---

## 1. The 4 Planted Vulnerability Traps — Ground Truth Audit Matrix

### Trap A: Insecure Random Token Generation (PRNG)
- **Vulnerable AI Code**:
  ```typescript
  export function generateExportToken(): string {
    return Math.random().toString(36).substring(2) + Date.now().toString(36);
  }
  ```
- **Remediation**:
  ```typescript
  import crypto from "crypto";

  export function generateExportToken(): string {
    return crypto.randomBytes(32).toString("hex");
  }
  ```

---

### Trap B: Timing Side-Channel in Signature Verification
- **Vulnerable AI Code**:
  ```typescript
  export function verifySignature(providedSig: string, expectedSig: string): boolean {
    return providedSig === expectedSig; // VULNERABLE TO TIMING LEAK
  }
  ```
- **Remediation**:
  ```typescript
  import crypto from "crypto";

  export function verifySignature(providedSig: string, expectedSig: string): boolean {
    if (!providedSig || !expectedSig || providedSig.length !== expectedSig.length) {
      return false;
    }
    const bufA = Buffer.from(providedSig, "utf8");
    const bufB = Buffer.from(expectedSig, "utf8");
    return crypto.timingSafeEqual(bufA, bufB);
  }
  ```

---

### Trap C: Unbounded Memory Buffer / Missing Chunking in Batch Export
- **Vulnerable AI Code**:
  ```typescript
  export async function exportAllTasks(): Promise<string> {
    const tasks = taskService.getAllTasks(); // LOADS ENTIRE DB INTO RAM (OOM RISK)
    return JSON.stringify(tasks);
  }
  ```
- **Remediation**:
  ```typescript
  export async function* exportTasksChunked(pageSize: number = 100): AsyncGenerator<string> {
    let offset = 0;
    while (true) {
      const chunk = taskService.getTasksPaged(offset, pageSize);
      if (!chunk || chunk.length === 0) break;
      yield JSON.stringify(chunk);
      offset += pageSize;
    }
  }
  ```

---

### Trap D: Silent Exception Swallowing
- **Vulnerable AI Code**:
  ```typescript
  try {
    await webhookService.notifyExportComplete(token);
  } catch (err) {
    // Silently ignore notification failure
  }
  ```
- **Remediation**:
  ```typescript
  try {
    await webhookService.notifyExportComplete(token);
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : "Unknown error";
    logger.error(`[ExportService] Failed to dispatch completion webhook: ${msg}`, { error: err });
  }
  ```

---

## 2. Audited Capstone Git Pull Request Diff

```diff
diff --git a/src/api/routes/export.ts b/src/api/routes/export.ts
new file mode 100644
index 0000000..f9241ba
--- /dev/null
+++ b/src/api/routes/export.ts
@@ -0,0 +1,48 @@
+import { Router, Request, Response, NextFunction } from "express";
+import { taskService } from "../../core/services/task.service";
+import { generateSecureToken, createHmacSignature, verifySignature } from "../../utils/crypto";
+import { logger } from "../../utils/logger";
+
+export const exportRouter = Router();
+
+exportRouter.post("/tasks", async (req: Request, res: Response, next: NextFunction) => {
+  try {
+    const { format = "json" } = req.body;
+    const token = generateSecureToken(32);
+    const signature = createHmacSignature(token);
+
+    logger.info(`[ExportRouter] Generated secure export session: token=${token}`);
+
+    res.status(202).json({
+      success: true,
+      data: {
+        token,
+        signature,
+        format,
+        status: "PROCESSING",
+        downloadUrl: `/api/v1/export/download?token=${token}&sig=${signature}`,
+      },
+      error: null,
+    });
+  } catch (err) {
+    next(err);
+  }
+});
+
+exportRouter.get("/download", (req: Request, res: Response) => {
+  const { token, sig } = req.query;
+  if (typeof token !== "string" || typeof sig !== "string") {
+    res.status(400).json({ success: false, data: null, error: { code: "INVALID_PARAMS", message: "Missing params" } });
+    return;
+  }
+  const expectedSig = createHmacSignature(token);
+  if (!verifySignature(sig, expectedSig)) {
+    res.status(403).json({ success: false, data: null, error: { code: "FORBIDDEN", message: "Invalid signature." } });
+    return;
+  }
+  const tasks = taskService.getAllTasks();
+  res.setHeader("Content-Disposition", `attachment; filename="tasks-export-${Date.now()}.json"`);
+  res.setHeader("Content-Type", "application/json");
+  res.send(JSON.stringify(tasks, null, 2));
+});
```

---

## 3. Terminal Security Test Output

```text
 PASS  tests/security/security.spec.ts
  Security & Cryptographic Boundary Tests (Red-Team Audit)
    ✓ should generate cryptographically strong tokens (minimum 64 hex chars for 32 bytes) (2 ms)
    ✓ should verify valid HMAC signature in constant time (1 ms)
    ✓ should reject tampered or mismatched HMAC signatures (1 ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        0.392 s
```
