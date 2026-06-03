# TESSA MCP — Generic Agent Guide

Use this markdown as context/rules for any IDE or agent that supports MCP but isn't explicitly covered by the other packaged variants (Windsurf, Cline, Continue, Zed, Roo Code, Aider, etc.).

## Server

- **URL**: `https://agent.qualis-lab.com/mcp`
- **Auth**: Bearer token with prefix `qai_` (TESSA API token).
- Get a token: login in TESSA frontend → Settings → API Tokens → Create.

## Tools (8)

| Tool | Input | Purpose |
|---|---|---|
| `list_projects` | `{ page, pageSize }` | Paginated list of the **projects** the user can access (id + name) |
| `list_test_cases` | `{ projectId, page, pageSize }` | Test cases of an accessible project (returns project name + cases) |
| `fetch_test_case` | `{ caseId }` | Happy path details with steps |
| `fetch_additional_cases` | `{ caseId }` | Edge/error scenarios |
| `get_presigned_url` | `{ fileName, contentType, caseId? }` | Pre-signed S3 URL for screenshot upload |
| `submit_test_result` | `{ caseId, status, executedUrl, totalDurationMs?, steps }` | Final execution report |
| `create_analysis_draft` | `{ projectId, fileName, contentType }` | Step 1 of doc-based generation: creates a DRAFT process + returns a presigned PUT URL to upload the functional document |
| `generate_analysis` | `{ processId, fileUrl, documentType, ...optional }` | Step 2: generates test cases (async) from the uploaded document |

`list_projects` returns `{ projects: [{ id, name }], pagination }`. `list_test_cases` returns `{ projectId, projectName, cases: [{ caseId, title, status }], pagination }`. The `caseId` is what every other tool consumes.

## Correct workflow

1. If user didn't specify a `caseId` → call `list_projects`, pick the project, then `list_test_cases({ projectId })` and ask which case.
2. `fetch_test_case` to get steps.
3. (Optional) `fetch_additional_cases` if user asked for all cases or happy path failed.
4. **Confirm with user** — target URL, expected side-effects, explicit approval.
5. For each step:
   a. Translate `step.action` (natural language) to browser actions.
   b. Execute, measure `durationMs`.
   c. Screenshot.
   d. `get_presigned_url` → HTTP PUT binary to `uploadUrl` → save `publicUrl`.
   e. Record step result (status, duration, publicUrl, errorMessage).
6. `submit_test_result` with full steps array.
7. Summarize: global status, failed steps, screenshot URLs.

## Generating test cases from a document (create_analysis_draft → generate_analysis)

Two-step flow to turn a functional document (PDF or DOCX) into test cases.

1. **Step 1** — `create_analysis_draft({ projectId, fileName, contentType })`. `contentType` MUST be `application/pdf` or `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (docx). Returns `{ processId, uploadUrl, publicUrl }`.
2. **Upload** — HTTP PUT the document binary to `uploadUrl` with the matching `Content-Type` header. **Gotcha**: the presigned URL is signed with only `host;content-type` headers — do NOT send any `x-amz-checksum-*` header even though it appears in the URL query string, or S3 returns 403. Just PUT with `Content-Type`.
3. **Step 2** — `generate_analysis({ processId, fileUrl: <publicUrl>, documentType: 'pdf'|'word', originalName?, prompt?, industry?, functionality?, platform?, additionals?, gherkin?, uxUi? })`. `additionals`/`gherkin`/`uxUi` default to false. Returns `{ processId, message }` — generation is **ASYNCHRONOUS**; poll process status afterwards (e.g. via `list_test_cases` or the process endpoint) until cases appear in `PROCESADO`.

Input specs:

| Tool | Input | Constraints |
|---|---|---|
| `create_analysis_draft` | `projectId` | positive int |
| | `fileName` | string, max 255 |
| | `contentType` | string (`application/pdf` or docx MIME) |
| `generate_analysis` | `processId` | positive int |
| | `fileUrl` | string, max 2000 |
| | `documentType` | enum `'pdf'` \| `'word'` |
| | `originalName?` | string, max 500 |
| | `prompt?` | string, max 5000 |
| | `industry?` | string, max 200 |
| | `functionality?` | string, max 200 |
| | `platform?` | string, max 200 |
| | `additionals?` | bool, default false |
| | `gherkin?` | bool, default false |
| | `uxUi?` | bool, default false |

Constraints: `fileUrl` must point to the same process folder returned by step 1 (anti-SSRF). The process must be in `DRAFT` and owned by the caller. The provider used is the company's active LLM integration.

## Hard rules

- **Explicit user confirmation required** before running any test (they can touch real systems).
- **No assumed URLs** (`localhost`, `example.com`, etc.). Ask if not provided.
- **Screenshots via presigned URL**, never base64 inline.
- **`contentType` whitelist**: `image/png`, `image/jpeg`, `image/jpg`, `image/webp` only.
- **Global status**: worst-step wins. If any step isn't PASS, global can't be PASS (FAIL > ERROR > SKIPPED > PASS).
- **Project access server-enforced**. You only see/act on projects you're a member of. Don't try to access others' `projectId`s or `caseId`s.
- **No credentials in errorMessage** — redact as `pass=***`.
- **Max ~100 steps per execution**. No rate-limit server-side — don't parallelize wildly.

## Error handling

| Error | Fix |
|---|---|
| `401 Invalid API token` | User regenerates in TESSA → Settings → API Tokens |
| `Authentication required` | Ensure the API token is sent as `Authorization: Bearer qai_...` |
| `Project not found` / `Project not accessible` | Use only `projectId`s returned by `list_projects` |
| `Invalid caseId` | Call `list_test_cases({ projectId })` to find valid IDs |
| `Invalid content type` | Convert screenshot to PNG/JPEG/WebP |
| `Test case (process) not found` / `not accessible` | Verify `caseId` with `list_test_cases({ projectId })` |
| `Invalid content type: ...` (create_analysis_draft) | Use `application/pdf` or the docx MIME type |
| `Process ... is not a draft` / `Process not accessible` (generate_analysis) | The `processId` must come from a fresh `create_analysis_draft` call and belong to you |
| `Invalid file URL: must point to this process folder` | Pass the exact `publicUrl` returned by `create_analysis_draft` as `fileUrl` |

## Equivalent REST endpoints (fallback)

If the MCP connection isn't available, same capabilities via REST (same auth):

```
GET  /api/mcp/projects?page=N&pageSize=N
GET  /api/mcp/projects/:projectId/test-cases?page=N&pageSize=N
GET  /api/mcp/test-case/:caseId
GET  /api/mcp/test-case/:caseId/additional-cases
GET  /api/mcp/presigned-url?fileName=...&contentType=...&caseId=...
POST /api/mcp/test-result   (body: {caseId, status, executedUrl, totalDurationMs?, steps})
```

Header on all: `Authorization: Bearer qai_<token>`.

> `create_analysis_draft` and `generate_analysis` are MCP-only — no REST equivalent.
