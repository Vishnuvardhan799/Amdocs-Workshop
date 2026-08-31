# Instructor Reference Specification: Practical 2.2

---

## 1. Ground-Truth Fallacy Breakdown

| Target Excerpt | Formal Fallacy Name | Technical Breakdown & Reasoning Defect |
| :--- | :--- | :--- |
| **Excerpt A** (*"Either 35 microservices this quarter or company fails"*) | **False Dilemma (Black-and-White Fallacy)** | Reduces architectural decision-making into an artificial binary choice. Ignores pragmatic intermediate solutions such as modularizing the existing monolith, establishing clear domain boundaries, profiling performance bottlenecks, or extracting single asynchronous background workers. |
| **Excerpt B** (*"NetStream grew 350%, so microservices will grow our revenue 300%"*) | **False Cause (Post Hoc / Correlation vs. Causation)** | Attributes multi-year subscriber growth solely to microservice topology, ignoring content catalog expansion, international marketing, and mobile device adoption. Assumes organizational patterns from large-scale enterprises apply directly to distinct business scales. |
| **Excerpt C** (*"Monolith advocates want 1990s spaghetti code"*) | **Straw Man Fallacy** | Distorts the modular monolith architectural pattern into an unmaintainable caricature. Well-architected monoliths feature strict domain boundaries, automated CI/CD pipelines, and high test coverage. |
| **Excerpt D** (*"Cloud Visionary with 150k followers settled debate"*) | **Appeal to Authority / Bandwagon Fallacy** | Bases enterprise technical architecture on social media follower counts rather than workload profiling, team competency assessments, or financial cost-benefit modeling. |

---

## 2. Unmasked Unstated Assumptions

1. **DevOps & Infrastructure Competency**: Assumes existing development squads possess production mastery in configuring Kubernetes clusters, Istio service meshes, Kafka event brokers, and OpenTelemetry distributed tracing.
2. **Network Overhead & Distributed Consistency**: Assumes replacing fast in-memory method calls with network I/O hops will not impact latency SLAs, and ignores distributed transaction (2PC/Saga) complexity.
3. **Misattribution of Release Cadence**: Assumes the 3-week release cycle is an inherent property of the codebase structure rather than inadequate automated regression testing and manual deployment processes.

---

## 3. Phased Engineering Counter-Proposal (Model Solution)

```markdown
### Strategic Counter-Proposal: Phased Modernization & Domain Encapsulation

**Executive Summary:**
Rather than executing a high-risk immediate rewrite into 35 microservices, we recommend a phased, metrics-driven modernization strategy. This approach directly addresses the 3-week release cadence while maintaining transactional reliability and managing infrastructure overhead.

#### Phase 1: Automated CI/CD & Test Automation (Weeks 1–4)
Primary deployment bottlenecks stem from manual verification rather than monolithic topology. We will:
- Implement automated unit, integration, and end-to-end regression pipelines in CI.
- Establish automated staging deployments and smoke test suites.
- Target Metric: Reduce deployment rollback rate to <2% and enable bi-weekly release cycles.

#### Phase 2: Refactoring into a Modular Monolith (Weeks 5–10)
Establish strict domain boundaries within the existing codebase:
- Enforce domain encapsulation across modules (Auth, Inventory, Billing, Notifications) with clear internal interfaces.
- Eliminate cross-domain direct database joins in application code.
- Target Metric: Zero merge conflicts across separate feature squads working in distinct domains.

#### Phase 3: Targeted Asynchronous Extraction (Weeks 11+)
Extract only high-throughput or resource-intensive sub-domains (e.g., PDF generation, bulk notification dispatch) into standalone serverless functions or containerized services where independent scaling yields measurable operational ROI.
```
