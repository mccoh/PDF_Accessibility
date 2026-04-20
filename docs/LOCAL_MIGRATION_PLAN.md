# Local Monolithic Migration Plan

## 1) Objective

This document defines a migration plan to run the current PDF Accessibility platform on a single local server instead of AWS, while preserving the existing business behavior of both processing paths:

- PDF-to-HTML remediation
- PDF-to-PDF remediation

The goal is to replace AWS infrastructure and managed services with local equivalents while keeping the core document-processing logic, Adobe integration, accessibility checks, and output structure as intact as possible.

## 2) Target Outcome

After migration, the system should run as one locally hosted application that:

- Accepts uploaded PDFs through HTTP endpoints or a local UI
- Processes both remediation pipelines on the same server
- Stores inputs, intermediates, and outputs on the local filesystem
- Uses local orchestration instead of Lambda, ECS, S3 events, and Step Functions
- Replaces Bedrock and Comprehend with local or self-hosted alternatives
- Preserves output conventions and downstream behavior as closely as practical

## 3) Current AWS-Dependent Architecture Summary

### 3.1 PDF-to-HTML path

Current flow:
1. A PDF is uploaded to S3 under `uploads/`
2. S3 event notification triggers a Lambda container
3. Lambda downloads the file to `/tmp`
4. The runtime calls Bedrock Data Automation for PDF extraction
5. HTML is audited and remediated
6. Outputs are uploaded back to S3 under `output/` and `remediated/`

Current AWS dependencies in this path:
- S3 for storage and event triggers
- Lambda for execution
- Bedrock Data Automation for extraction/parsing
- Bedrock Runtime for remediation and content generation
- CloudWatch for logs
- ECR and CDK for packaging and deployment

### 3.2 PDF-to-PDF path

Current flow:
1. A PDF is uploaded to S3 under `pdf/`
2. S3 triggers the PDF splitter Lambda
3. The splitter divides the document into chunk PDFs and stores them under `temp/`
4. Step Functions orchestrates chunk processing in parallel
5. ECS Fargate runs the Adobe autotag container and the alt-text generator container per chunk
6. A Java Lambda merges processed chunks
7. A title generator Lambda updates PDF metadata
8. Pre- and post-remediation Adobe accessibility checks generate JSON reports
9. Final outputs are uploaded to `result/`

Current AWS dependencies in this path:
- S3 for all pipeline storage
- Lambda for split, merge, title generation, and audits
- Step Functions for orchestration
- ECS Fargate for chunk processing containers
- Bedrock Runtime for title generation and alt text
- Comprehend for language detection
- Secrets Manager for Adobe credentials
- CloudWatch for logs and monitoring
- CDK for infrastructure provisioning

## 4) Proposed Local Monolithic Architecture

### 4.1 High-level design

Replace the AWS event-driven architecture with a single local application process, preferably Python-based, with background workers for document processing.

Recommended runtime model:
- FastAPI application for upload, status, and download endpoints
- Background task workers inside the same app process or a local worker pool
- Shared local storage rooted at `./data/`
- Optional Docker Compose wrapper for repeatable local deployment

### 4.2 Local directory structure

Preserve the existing storage conventions so the business logic changes as little as possible:

- `data/uploads/`
- `data/output/`
- `data/remediated/`
- `data/pdf/`
- `data/temp/`
- `data/result/`
- `data/logs/`

This allows the migrated code to keep path assumptions close to the current S3 key conventions.

### 4.3 Local orchestration model

Replace cloud orchestration with local Python orchestration:

- S3 event notifications become HTTP upload endpoints or a filesystem watcher
- Lambda handlers become normal Python function calls
- ECS tasks become local subprocess or in-process worker executions
- Step Functions becomes a local orchestrator that coordinates split, parallel chunk processing, merge, and finalization

For the PDF-to-PDF path, use `concurrent.futures.ThreadPoolExecutor` or `ProcessPoolExecutor` for fan-out/fan-in behavior.

## 5) AWS-to-Local Replacement Strategy

| AWS Service | Local Replacement | Notes |
|---|---|---|
| S3 | Local filesystem under `./data/` | Preserve `uploads/`, `output/`, `remediated/`, `pdf/`, `temp/`, `result/` folder conventions |
| S3 Event Notification | FastAPI upload endpoint or filesystem watcher | Triggers processing when files arrive |
| Lambda | Direct Python function execution | Lambda handlers become callable modules |
| ECS Fargate | Local subprocesses or in-process workers | Existing container logic can run without AWS |
| Step Functions | Python orchestration module | Replaces fan-out/fan-in workflow |
| Bedrock Data Automation | Docling or Marker-PDF plus compatibility adapter | Highest-risk replacement area |
| Bedrock Runtime | Ollama or another local model endpoint | Used for title generation, remediation, and alt text |
| Comprehend | `langdetect` or similar Python package | Replaces dominant language detection |
| Secrets Manager | `.env` file with `python-dotenv` | Adobe credentials loaded locally |
| CloudWatch | Local structured logs in `data/logs/` | JSON logging recommended |
| ECR | Not needed for local runtime | Optional Docker only |
| CDK | Not needed | Replaced with app config and local startup scripts |

## 6) Component Migration Plan

## Phase 0: Local Runtime Foundation

Objectives:
- Establish the local runtime shell around the existing code
- Create a single entry point for uploads, job execution, and result retrieval

Technical tasks:
1. Create a `server/` application module at repo root
2. Add a FastAPI entry point with endpoints for:
   - `POST /upload/pdf2html`
   - `POST /upload/pdf2pdf`
   - `GET /status/{job_id}`
   - `GET /result/{filename}`
3. Add a configuration loader for:
   - Adobe API credentials
   - Local model endpoint URL
   - Model names
   - Data root path
   - Cleanup flags
4. Create the `data/` folder structure used by both workflows
5. Add local logging configuration writing to `data/logs/`

Deliverables:
- Local application shell
- Config loader and environment template
- Shared data directory structure

## Phase 1: Storage Abstraction Layer

Objectives:
- Remove direct S3 dependence from application logic
- Preserve existing storage semantics as closely as possible

Technical tasks:
1. Implement a local storage adapter with methods similar to the S3 client usage in the codebase:
   - `download_file`
   - `upload_file`
   - `upload_fileobj`
   - `head_object`
   - `delete_object`
   - `delete_objects`
   - `list_objects_v2`
2. Back the adapter with local file operations under `data/`
3. Route existing code through this adapter instead of direct `boto3.client("s3")` calls
4. Preserve current key patterns such as:
   - `uploads/{filename}.pdf`
   - `pdf/{filename}.pdf`
   - `temp/{base}/...`
   - `output/{filename}.zip`
   - `remediated/final_{filename}.zip`
   - `result/COMPLIANT_{filename}.pdf`

Deliverables:
- Filesystem-backed storage adapter
- Minimal changes to existing business logic modules

## Phase 2: PDF-to-HTML Migration

Objectives:
- Rehost the PDF-to-HTML path locally with the same user-visible outputs

Technical tasks:
1. Convert `pdf2html/lambda_function.py` from an S3 event Lambda entry point into a callable local pipeline function
2. Replace its S3 download and upload logic with the storage adapter
3. Preserve current output contracts:
   - `output/{filename}.html`
   - `output/{filename}.zip`
   - `remediated/final_{filename}.zip`
4. Keep the audit and remediation stages intact where possible
5. Replace Bedrock Runtime calls in remediation services with a local LLM adapter

### 2.1 Replace Bedrock Data Automation

This is the highest-risk workstream.

Current dependency:
- `pdf2html/content_accessibility_utility_on_aws/pdf2html/services/bedrock_client.py`

Replacement plan:
1. Introduce a local extraction provider interface
2. Implement a local adapter using Docling or Marker-PDF
3. Normalize the extracted output into the schema expected by downstream HTML building and remediation logic
4. Validate parity with representative PDFs

Critical observation:
- The current downstream code assumes BDA-like structure. This means the migration cannot simply swap libraries; it must also provide a compatibility layer.

### 2.2 Replace Bedrock Runtime in HTML remediation

Current dependency:
- `pdf2html/content_accessibility_utility_on_aws/remediate/services/bedrock_client.py`

Replacement plan:
1. Create a local LLM client wrapper that mimics the current model invocation contract
2. Point it to Ollama or another local inference server
3. Preserve expected prompt and response behavior for:
   - alt text generation
   - text remediation
   - table and structure repair flows

Deliverables:
- Fully local PDF-to-HTML processing flow
- Local extraction adapter
- Local LLM adapter

## Phase 3: PDF-to-PDF Migration

Objectives:
- Replatform the orchestration-heavy PDF-to-PDF path onto a single local server

Technical tasks:
1. Convert the splitter Lambda into a normal function that:
   - reads from local `pdf/`
   - writes chunks to local `temp/`
   - directly invokes the local orchestrator instead of Step Functions
2. Replace Step Functions with a local orchestration module that:
   - processes chunks in parallel
   - waits for completion
   - merges outputs
   - triggers final title generation and audits
3. Replace ECS task launches with direct script execution or in-process function calls
4. Keep Adobe API usage unchanged except for local secret loading

### 3.1 Adobe autotag processor

Current file:
- `adobe-autotag-container/adobe_autotag_processor.py`

Changes required:
1. Replace S3 reads and writes with local storage adapter usage
2. Replace Secrets Manager lookup with environment-based credential loading
3. Replace Comprehend language detection with a local package such as `langdetect`
4. Keep Adobe API calls intact

### 3.2 Alt text generator

Current file:
- `alt-text-generator-container/alt_text_generator.js`

Changes required:
1. Replace S3 SDK access with local filesystem reads and writes
2. Replace Bedrock Runtime calls with a local vision-capable model endpoint such as Ollama
3. Preserve current output contract:
   - `temp/{base}/FINAL_{chunk}.pdf`

Recommended approach:
- Keep the Node.js implementation initially to reduce change scope
- Port to Python later only if maintaining a mixed-runtime local server becomes a burden

### 3.3 PDF merger

Current component:
- Java merger Lambda at `lambda/pdf-merger-lambda/`

Options:
1. Keep the Java implementation and run it locally through `subprocess`
2. Replace it with Python `pypdf.PdfMerger` or PyMuPDF-based merge logic

Recommended approach:
- Start by keeping the Java merger if Java is already available on the target server
- Replace with Python later if runtime simplification becomes necessary

### 3.4 Title generator

Current file:
- `lambda/title-generator-lambda/title_generator.py`

Changes required:
1. Replace Bedrock Runtime calls with the local LLM adapter
2. Replace S3 access with local file access through the storage layer
3. Preserve metadata-writing behavior and output path under `result/`

### 3.5 Pre- and post-remediation audits

Current files:
- `lambda/pre-remediation-accessibility-checker/main.py`
- `lambda/post-remediation-accessibility-checker/main.py`

Changes required:
1. Replace Secrets Manager access with local environment config
2. Replace S3 access with local file access
3. Keep Adobe accessibility check workflow unchanged

Deliverables:
- Fully local PDF-to-PDF pipeline
- Local orchestration replacing Step Functions and ECS

## Phase 4: Local Model Hosting

Objectives:
- Replace Bedrock model usage with locally hosted or self-managed inference

Recommended baseline:
- Ollama for local hosting

Suggested model roles:
- Vision-capable model for image alt text generation
- Text model for title generation and remediation prompts

Technical tasks:
1. Install and validate Ollama on the target server
2. Pull required local models
3. Build an adapter that mimics the current Bedrock client interface used by the application
4. Add timeout, retry, and response validation behavior
5. Validate prompt outputs against current JSON or text expectations

Deliverables:
- Local LLM inference layer
- Prompt compatibility wrapper

## Phase 5: Server Packaging and Operations

Objectives:
- Make the local application easy to run, monitor, and operate

Technical tasks:
1. Add a local startup command using Uvicorn
2. Optionally add Docker Compose with:
   - app service
   - local model service
3. Add structured logging and log rotation
4. Add a simple local job registry using SQLite or an in-memory dictionary with persistence if needed
5. Add health endpoints for operations

Deliverables:
- Runnable local server package
- Optional containerized local deployment
- Basic operational tooling

## 7) Files and Modules Impacted

Primary files to modify:
- `pdf2html/lambda_function.py`
- `pdf2html/content_accessibility_utility_on_aws/pdf2html/services/bedrock_client.py`
- `pdf2html/content_accessibility_utility_on_aws/remediate/services/bedrock_client.py`
- `lambda/pdf-splitter-lambda/main.py`
- `lambda/pre-remediation-accessibility-checker/main.py`
- `lambda/post-remediation-accessibility-checker/main.py`
- `lambda/title-generator-lambda/title_generator.py`
- `adobe-autotag-container/adobe_autotag_processor.py`
- `alt-text-generator-container/alt_text_generator.js`

New modules recommended:
- `server/main.py`
- `server/config.py`
- `server/storage.py`
- `server/llm_client.py`
- `server/orchestrator_pdf2html.py`
- `server/orchestrator_pdf2pdf.py`

## 8) Data and Path Contracts to Preserve

The following path conventions should remain stable to reduce business logic churn:

### PDF-to-HTML
- `uploads/{filename}.pdf`
- `output/{filename}.html`
- `output/{filename}.zip`
- `remediated/final_{filename}.zip`

### PDF-to-PDF
- `pdf/{filename}.pdf`
- `temp/{base}/{base}_chunk_{n}.pdf`
- `temp/{base}/output_autotag/COMPLIANT_{chunk}.pdf`
- `temp/{base}/FINAL_{chunk}.pdf`
- `temp/{base}/merged_{base}.pdf`
- `result/COMPLIANT_{filename}.pdf`

Maintaining these conventions will reduce the amount of refactoring required across the codebase.

## 9) Risks and Constraints

### Highest-risk items

1. Bedrock Data Automation replacement
   - No direct local equivalent will perfectly match current output structure.
   - A compatibility adapter is required.

2. Vision-model quality for alt text generation
   - Local models may not match Bedrock Nova output quality.
   - Prompt tuning and validation will be required.

3. Mixed runtime complexity
   - The local server may need Python, Node.js, and Java unless components are consolidated.

4. Performance and concurrency on one machine
   - Large documents and concurrent uploads may stress CPU, RAM, and disk I/O.

5. Adobe dependency remains external
   - This migration removes AWS, not Adobe API reliance.

## 10) Recommended Implementation Order

1. Create the local server shell and storage adapter
2. Migrate PDF-to-HTML first because it has simpler orchestration
3. Build and validate the local extraction compatibility adapter
4. Replace Bedrock Runtime usage with a local LLM adapter
5. Migrate PDF-to-PDF orchestration and chunk processing
6. Decide whether to keep or replace the Java merger
7. Harden local operations, logging, and job tracking

## 11) Validation Strategy

Validation should be done incrementally with the same sample PDFs used in AWS.

Required checks:
1. PDF-to-HTML output zip contents match current expectations
2. PDF-to-PDF output files and reports are written to the expected locations
3. Temporary files are cleaned after processing
4. Logs clearly identify each job and stage
5. Parallel chunk processing works without corrupting outputs
6. Local LLM outputs conform to expected JSON or text formats

## 12) Recommendation

The local monolithic migration is feasible, but it should be treated as an infrastructure replacement plus a provider-adapter project rather than a simple lift-and-shift.

Best technical strategy:
- Preserve the current business logic and path contracts
- Introduce local adapters for storage, orchestration, extraction, and model inference
- Migrate the PDF-to-HTML path first
- Keep Adobe integration unchanged
- Replace AWS-specific behavior one dependency layer at a time

The primary engineering challenge is not HTTP hosting or local execution. It is reproducing the behavior currently provided by Bedrock Data Automation and Bedrock Runtime closely enough that the downstream remediation logic continues to work with minimal redesign.

## 13) Single-Developer Delivery Estimate

Based on the migration scope in this document, one senior full stack developer working mostly solo should plan for:

- Best case: 4 to 6 months
- Most likely: 6 to 9 months
- Conservative: 9 to 12 months

This estimate assumes the goal is a working local monolithic server that preserves the current business behavior of both pipelines, not just a prototype.

### Why this timeline is substantial

- The work is not only a hosting change. It also replaces multiple AWS-managed platform capabilities with local equivalents.
- The Bedrock Data Automation replacement requires both product evaluation and a compatibility adapter for downstream code.
- The PDF-to-PDF path has orchestration complexity, mixed runtimes, and chunk-parallel processing that must be re-created locally.
- A solo developer must also absorb testing, environment setup, troubleshooting, logging, and operational hardening.

### Indicative solo-developer phase breakdown

1. Local server foundation, config, and storage adapter: 2 to 4 weeks
2. PDF-to-HTML migration and local LLM integration: 4 to 8 weeks
3. Local extraction compatibility adapter for BDA replacement: 4 to 8 weeks
4. PDF-to-PDF orchestration migration and chunk processing rework: 6 to 10 weeks
5. Reliability hardening, validation, cleanup, and packaging: 3 to 6 weeks

### Planning recommendation

For planning purposes, use 7 to 8 months as the most realistic target for one strong senior full stack developer, with the understanding that the BDA replacement is the largest schedule risk and could move the effort toward the upper end of the range.
