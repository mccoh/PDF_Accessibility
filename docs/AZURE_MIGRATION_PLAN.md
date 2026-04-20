# Azure Migration Plan for PDF Accessibility Platform

## 1) Purpose and Outcomes

This document defines a full migration plan to run the current PDF Accessibility platform on Microsoft Azure.

Primary outcomes:
- Preserve business behavior of both processing paths:
  - PDF-to-HTML pipeline
  - PDF-to-PDF pipeline
- Reduce operational risk by migrating in phases
- Replace script-driven deployment with repeatable Infrastructure as Code and CI/CD
- Improve reliability, security posture, and observability

Success criteria:
- Functional parity for core workflows
- Controlled cutover with rollback capability
- Measurable performance and cost targets
- Security controls aligned to enterprise standards

## 2) In-Scope and Out-of-Scope

In-scope:
- Backend migration for both solution paths
- Event ingestion, orchestration, compute, storage, secrets, monitoring, and CI/CD
- Service adapter strategy where AWS-native APIs have no direct Azure equivalent

Out-of-scope for initial cutover:
- Major feature redesign of remediation logic
- UI redesign or rearchitecture
- Elimination of Adobe dependency in PDF-to-PDF path

## 3) Current-State Architecture Summary

### 3.1 PDF-to-HTML path

High-level flow:
1. User uploads PDF to object storage prefix uploads/
2. Event triggers containerized function
3. Function performs conversion, audit, remediation, and packaging
4. Outputs written to output/ and remediated/ prefixes

Current coupling points:
- Bedrock Data Automation is used for PDF parsing and extraction
- Bedrock model calls are used during remediation and alt text generation
- S3 event semantics and key-prefix logic are part of runtime behavior

### 3.2 PDF-to-PDF path

High-level flow:
1. User uploads PDF to object storage prefix pdf/
2. Splitter creates chunk files and starts workflow orchestration
3. Orchestration fans out chunk processing through Adobe and alt text containers
4. Chunks are merged, title is generated, pre and post checks executed
5. Final compliant artifact is written to result/

Current coupling points:
- Step Functions map and parallel semantics
- ECS task orchestration and private networking assumptions
- Adobe API credentials and runtime behavior
- Path-based contracts in temp/ and result/ conventions

### 3.3 Deployment model

Current deployment is mixed declarative and imperative:
- Infrastructure in CDK stacks
- Script-created roles and pipeline resources in deploy.sh
- Build logic in buildspec-unified.yml

Implication for migration:
- Azure deployment should be fully declarative to avoid drift

## 4) Azure Target Architecture

### 4.1 Core design principles

- Minimize business-logic changes during first cut
- Encapsulate provider-specific APIs behind adapters
- Use managed identity and least privilege by default
- Adopt resilient event processing with idempotent job tracking

### 4.2 Service mapping

| AWS Component | Azure Equivalent | Migration Notes |
|---|---|---|
| S3 | Azure Blob Storage | Preserve virtual folder conventions uploads/, output/, remediated/, pdf/, temp/, result/ |
| S3 Event Notification | Event Grid BlobCreated | Add prefix and suffix filters for route control |
| Lambda (container) | Azure Functions custom container or Azure Container Apps Jobs | Use Functions for event-driven short-to-medium tasks, Jobs for heavier long-running tasks |
| ECS Fargate | Azure Container Apps Jobs | Best fit for chunk processing containers |
| Step Functions | Azure Durable Functions orchestrator | Supports fan-out/fan-in and branch orchestration |
| ECR | Azure Container Registry | Use immutable image tags and digest promotion |
| IAM roles | Managed Identity and Azure RBAC | Replace key-based auth paths where possible |
| Secrets Manager | Azure Key Vault | Keep Adobe credentials and API keys in vault |
| CloudWatch Logs/Metrics | Azure Monitor, Log Analytics, Application Insights | Structured logs and workbook dashboards |
| Bedrock Runtime | Azure OpenAI or Azure AI Foundry endpoints | Adapter required for prompt and response contract |
| Bedrock Data Automation | Azure AI Document Intelligence plus compatibility adapter | No direct one-to-one service; highest migration risk |
| Comprehend language detection | Azure AI Language | Replace API calls in alt text/title components |

## 5) Migration Phases and Technical Steps

## Phase 0: Baseline, Contracts, and Governance

Objectives:
- Lock current behavior contract before changes
- Define Azure platform guardrails

Technical tasks:
1. Capture baseline artifact contracts:
   - Output zip composition and naming
   - Intermediate folder conventions
   - Remediation report schema and usage data
2. Define non-functional targets:
   - Throughput per 100 docs
   - Error rate and retry policy
   - Max end-to-end processing duration
3. Establish governance:
   - Subscription/resource-group layout for dev, test, prod
   - Tagging standard
   - RBAC role boundaries
   - Policy controls for public access, key rotation, private endpoints

Deliverables:
- Baseline behavior matrix
- Security and platform standards
- Migration acceptance criteria document

## Phase 1: Azure Foundation Infrastructure

Objectives:
- Build shared platform services needed by both paths

Technical tasks:
1. Network:
   - Create VNet, subnets for Functions and Container Apps environment
   - Configure NAT where outbound control is required
   - Create private endpoints for Blob Storage, Key Vault, and ACR
2. Data:
   - Provision Blob Storage accounts and containers
   - Configure lifecycle policies for temp and intermediate artifacts
3. Compute foundation:
   - Create Container Registry
   - Provision Function App plan and/or Container Apps environment
4. Identity and secrets:
   - Create managed identities for each workload
   - Create Key Vault, load Adobe credentials and required API secrets
5. Observability:
   - Create Log Analytics workspace
   - Configure Application Insights
   - Create workbook dashboards and baseline alerts

Deliverables:
- Azure landing zone for workload
- Shared IaC modules and environment parameter files

## Phase 1.1: PDF-to-HTML Lift-and-Adapt

Objectives:
- Migrate the simpler path first while preserving behavior

Technical tasks:
1. Event ingress:
   - Configure Event Grid BlobCreated subscription with filters:
     - startsWith uploads/
     - endsWith .pdf
2. Runtime hosting:
   - Build and publish container image to ACR
   - Deploy Azure Function custom container or Container App triggered by queue/event
3. Storage contract:
   - Preserve output locations and naming:
     - output/{filename}.html
     - output/{filename}.zip
     - remediated/final_{filename}.zip
4. Idempotency hardening:
   - Replace simple artifact existence check with durable job table:
     - Suggested store: Azure Table Storage or Cosmos DB
     - Key: deterministic document hash plus filename
     - Status states: received, processing, completed, failed, retrying
5. Intermediate cleanup:
   - Implement same cleanup behavior via policy and explicit delete routines

Deliverables:
- Azure-based PDF-to-HTML processing path in dev
- End-to-end runbook with sample files

## Phase 1.2: Document Extraction Compatibility Adapter

Objectives:
- Replace Bedrock Data Automation with Azure extraction while preserving downstream parser assumptions

Technical tasks:
1. Implement extraction provider interface:
   - Extract document structure and page-level elements
   - Return normalized intermediate JSON
2. Implement Azure adapter:
   - Use Azure AI Document Intelligence for OCR and structural extraction
   - Add post-processing layer to map to expected internal schema used by existing conversion logic
3. Validate schema parity:
   - Ensure required fields for page count, element grouping, and asset references are present
4. Add fallback and diagnostics:
   - Record extraction confidence
   - Route unsupported formats to failure queue with reason codes

Critical risk:
- This is the highest-risk workstream because Bedrock extraction format and behavior are currently assumed by downstream logic

Deliverables:
- Provider adapter package
- Schema contract tests with representative PDFs

## Phase 2: PDF-to-PDF Replatform

Objectives:
- Port orchestration-heavy Adobe path to Azure

Technical tasks:
1. Splitter migration:
   - Port splitter function to Azure Function
   - Preserve chunk naming and location semantics in Blob temp path
2. Orchestration migration:
   - Implement Durable Functions orchestration:
     - Fan-out on chunks
     - Sequential chunk operations per item
     - Fan-in merge
     - Parallel branch for pre-check and post-check where required
3. Container jobs:
   - Run Adobe autotag and alt text containers in Container Apps Jobs
   - Pass job metadata via durable state payloads instead of implicit runtime path assumptions
4. Merge and title generation:
   - Port Java merger and title generator to Functions or containerized jobs
   - Validate output file naming and destination
5. Adobe integration:
   - Keep credential schema in Key Vault
   - Replace direct secret retrieval with managed identity-based vault access

Deliverables:
- Azure-based PDF-to-PDF path in dev
- Parity-tested artifacts and reports

## Phase 2.1: Reliability and Recovery Enhancements

Objectives:
- Improve resilience beyond current best-effort idempotency

Technical tasks:
1. Durable retries with limits and exponential backoff
2. Dead-letter queue for terminal failures
3. Correlation IDs across every stage
4. Structured event log schema for all components
5. Duplicate event suppression using job state locks
6. Poison message handling and operator replay tooling

Deliverables:
- Operationally safe orchestration behavior in test
- Incident response runbook

## Phase 3: CI/CD and IaC Modernization

Objectives:
- Replace script-driven deployment with declarative and auditable pipelines

Technical tasks:
1. IaC stack:
   - Bicep or Terraform modules:
     - network
     - storage
     - compute
     - identity
     - observability
2. Build pipeline:
   - Lint, unit tests, dependency scanning, container build, push to ACR
3. Release pipeline:
   - Environment promotion dev to test to prod
   - Approval gates and change control
   - Immutable image digest deployment
4. Security gates:
   - Policy checks
   - Secret scanning
   - SBOM generation and artifact retention
5. Drift detection:
   - Scheduled plan checks and alerting

Deliverables:
- Production-grade deployment pipeline
- Versioned and repeatable environment provisioning

## Phase 4: Validation, Cutover, and Decommission

Objectives:
- Cut over with confidence and rollback safety

Technical tasks:
1. Side-by-side validation:
   - Run same document sample through AWS and Azure
   - Compare outputs, reports, and timing
2. Performance and cost tuning:
   - Scale profiles
   - Concurrency limits
   - Queue and batch sizing
3. Production cutover strategy:
   - Canary by prefix or tenant
   - Progressive traffic shift
4. Rollback plan:
   - Keep AWS path hot during initial Azure rollout
   - Fast route-back mechanism by trigger reconfiguration
5. Decommission wave plan:
   - Remove unused AWS resources after stabilization window

Deliverables:
- Cutover sign-off
- Decommission checklist and closure report

## 6) Security Architecture Details

Identity:
- Use user-assigned managed identities per workload, not shared identities
- Scope RBAC to minimum resource actions

Secrets:
- Adobe and model credentials in Key Vault
- Rotation policy and access audit enabled

Network:
- Private endpoints for data-plane services
- Restrict public network access where feasible
- Enforce TLS and at-rest encryption

Data controls:
- Storage lifecycle policy for temp artifacts
- Retention controls for audit reports
- Optional customer-managed keys if required by policy

## 7) Data and Interface Contracts

### 7.1 Storage conventions to preserve

- Ingest:
  - uploads/ for PDF-to-HTML
  - pdf/ for PDF-to-PDF
- Intermediate:
  - temp/{doc}/...
- Output:
  - output/
  - remediated/
  - result/

### 7.2 Processing status model

Required status fields:
- job_id
- source_uri
- stage
- status
- retry_count
- error_code
- error_message
- started_at
- updated_at
- completed_at

### 7.3 Compatibility adapter contract

The extraction adapter must output a stable intermediate schema that includes:
- document metadata
- page index and count
- extracted elements with positional and semantic context
- references to extracted assets

## 8) Observability and Operations

Logging:
- Structured JSON logs with correlation_id, job_id, doc_name, stage

Metrics:
- Ingest volume
- Success and failure counts by stage
- Mean and percentile processing times
- Retry and dead-letter counts

Dashboards:
- End-to-end throughput
- Stage health
- Top error categories
- Cost and utilization trend

Alerts:
- Failure rate threshold
- Queue backlog threshold
- Timeout threshold
- Cost anomaly alerts

Runbooks:
- Replay dead-lettered jobs
- Restart failed orchestration instances
- Recover partial outputs
- Rotate credentials and validate integration

## 9) Testing and Acceptance Strategy

Test categories:
1. Unit tests for adapters and path parsers
2. Contract tests for intermediate extraction schema
3. Integration tests for event to output flow
4. Load tests with representative file distribution
5. Chaos tests for transient failures and duplicate events

Acceptance criteria by phase:
- Functional parity on representative corpus
- Reliability targets met under load
- No critical security findings
- CI/CD and rollback validated

## 10) Cost and Performance Planning

Cost drivers to monitor:
- AI extraction and model token usage
- Container/job execution duration
- Function execution and storage transactions
- Logging ingestion volume

Optimization levers:
- Batch size and orchestration fan-out controls
- Early rejection for unsupported files
- Intelligent retries and short-circuiting
- Log sampling for verbose debug paths in prod

## 11) Risks and Mitigations

Risk: No direct replacement for Bedrock Data Automation behavior
- Mitigation: Compatibility adapter and schema contract tests before cutover

Risk: Orchestration semantics drift during Step Functions to Durable migration
- Mitigation: Stage-by-stage replay tests with deterministic test data

Risk: Path-based implicit contracts break in migrated jobs
- Mitigation: Explicit payload contracts and centralized path utility library

Risk: Duplicate events and race conditions
- Mitigation: Durable idempotency store and lock semantics

Risk: Vendor API limits and transient errors
- Mitigation: Backoff, circuit-breaker behavior, dead-letter queues, operator replay tooling

## 12) Suggested Timeline

- Phase 0: 1 to 2 weeks
- Phase 1 and 1.1: 3 to 5 weeks
- Phase 1.2: 2 to 4 weeks
- Phase 2 and 2.1: 4 to 7 weeks
- Phase 3: 2 to 3 weeks
- Phase 4: 2 to 3 weeks

Total estimated program: 14 to 24 weeks depending on team size, enterprise controls, and adapter complexity.

## 13) Immediate Next Actions

1. Approve target Azure service choices for orchestration and extraction
2. Approve IaC standard (Bicep or Terraform)
3. Build Phase 0 behavior baseline and acceptance corpus
4. Stand up Phase 1 foundation in dev
5. Begin PDF-to-HTML migration slice and extraction adapter spike

## 14) Single-Developer Delivery Estimate

Based on the migration scope in this plan, one senior full stack developer working mostly solo should plan for:

- Best case: 6 to 8 months
- Most likely: 8 to 11 months
- Conservative: 11 to 14 months

Why this estimate is longer than the original program estimate:
- The phase work is largely sequential for one person rather than parallel across a team.
- The Bedrock Data Automation replacement and compatibility adapter are high-risk and discovery-heavy.
- A solo engineer also absorbs platform, security, CI/CD, testing, and cutover overhead.

Indicative single-developer breakdown:
1. Foundation and governance: 3 to 5 weeks
2. PDF-to-HTML migration plus extraction adapter spike: 8 to 14 weeks
3. PDF-to-PDF orchestration replatform: 10 to 16 weeks
4. Reliability hardening, observability, and CI/CD: 6 to 10 weeks
5. Validation, pilot cutover, and rollback readiness: 4 to 7 weeks
