# Instructor Reference Specification: Practical 2.1

---

## 1. Ground-Truth Audit Matrix

| Claim Number | Taxonomy Category | Accuracy Assessment | Technical Diagnosis & Ground Truth |
| :--- | :--- | :--- | :--- |
| **Claim 1** (90% cloud cost reduction) | `[OPINION / MISLEADING FRAMING]` | **Misleading** | While serverless architecture provides cost advantages for intermittent or bursty traffic profiles, continuous high-throughput systems operating on Lambda can be 2x to 4x more expensive per million transactions than dedicated container instances (ECS/EKS) or reserved instances. |
| **Claim 2** (Zero cold start on Python 3.11) | `[HALLUCINATION / FACTUAL ERROR]` | **Inaccurate** | Standard AWS Lambda execution environments inherently experience cold starts (150ms to 500ms for Python depending on deployment package size and VPC configuration). Zero cold-start latency requires Provisioned Concurrency, which incurs fixed ongoing costs. |
| **Claim 3** (PostgreSQL is obsolete) | `[OPINION / BIASED FRAMING]` | **Inaccurate (Biased)** | Relational databases remain the industry standard for workloads demanding multi-table ACID transactions, financial ledgers, and complex relational joins. DynamoDB NoSQL is optimized for specific key-value and document access patterns, not a universal relational replacement. |
| **Claim 4** (`auto_sync_elastic_cluster`) | `[HALLUCINATION / FABRICATED API]` | **Inaccurate (Fabricated)** | No parameter `auto_sync_elastic_cluster` or method `update_table_settings` exists in AWS DynamoDB or Boto3 SDK. Automated ingestion into OpenSearch requires enabling DynamoDB Streams connected to an AWS Lambda consumer or OpenSearch Ingestion (OSI) pipeline. |
| **Claim 5** (60-minute Lambda timeout) | `[HALLUCINATION / LIMIT VIOLATION]` | **Inaccurate (Hard Quota)** | AWS Lambda enforces a hard, non-configurable execution timeout cap of 15 minutes (900 seconds). A 45-minute batch reconciliation job must be orchestrated via AWS Step Functions, AWS Batch, or AWS Fargate. |
| **Claim 6** (Downstream limits eliminated) | `[INFERENCE (FLAWED)]` | **Inaccurate** | Rapid horizontal scaling of Lambda functions exacerbates downstream constraints, exhausting relational database connection pools (requiring AWS RDS Proxy) and triggering HTTP 429 rate limit exceptions from third-party APIs. |

---

## 2. Primary-Source Verification Protocols

1. **SDK API Verification**: Inspect the official AWS Boto3 reference documentation ([boto3.amazonaws.com](https://boto3.amazonaws.com)) or query the local runtime using Python `dir(boto3.client('dynamodb'))`.
2. **Quota Verification**: Consult the official AWS Service Quotas documentation. Numeric service limits must never be sourced exclusively from generative AI outputs.
3. **Automation Bias Defense**: Institute automated static analysis (LSP autocomplete, MyPy type checking) and peer review gates to validate AI-generated implementation recommendations.

---

## 3. Revised Fact-Checked Executive Summary (Model Solution)

```markdown
### Executive Technical Brief: Serverless Assessment & Architectural Guardrails

Migrating our REST API to AWS Lambda provides significant operational efficiencies—including automated elastic scaling, reduced server provisioning overhead, and high cost-efficiency for bursty traffic patterns. However, deployment requires adhering to the following architectural guardrails:

1. **Execution Quotas & Batch Orchestration**: AWS Lambda enforces a strict maximum execution duration of 15 minutes. Long-running batch reconciliation workloads (45 minutes) must be executed on AWS Fargate or orchestrated as distributed workflows using AWS Step Functions.
2. **Cold-Start Latency Mitigation**: Standard Python execution environments experience 150ms-500ms initialization overhead during scale-out events. For latency-sensitive checkout paths, Provisioned Concurrency should be implemented.
3. **Data Layer Architecture**: DynamoDB is recommended for high-throughput order ingestion; however, existing PostgreSQL instances must be maintained for relational reporting and ACID transactional compliance. To protect relational databases from connection exhaustion during Lambda scale-outs, Amazon RDS Proxy must be deployed.
4. **Search Synchronization**: Streaming DynamoDB updates to OpenSearch clusters requires provisioning DynamoDB Streams with a dedicated event consumer, as direct table-level synchronization flags are not supported in the AWS SDK.
```
