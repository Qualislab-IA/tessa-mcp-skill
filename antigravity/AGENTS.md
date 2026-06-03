# TESSA MCP — Antigravity Agent Rules

This workspace integrates with **TESSA** (Test Execution & Smart Synthesis Agent), a SaaS platform that generates AI test cases and executes them via MCP-connected agents.

**MCP server**: `https://agent.qualis-lab.com/mcp`
**Authentication**: Bearer token with prefix `qai_` (TESSA API token).

Configure in Antigravity Settings → MCP Servers with URL `https://agent.qualis-lab.com/mcp` and header `Authorization: Bearer qai_XXXXXXXXXXXXXXXXXXXXXXXX`.

## When to use the TESSA MCP tools

Activate this workflow when the user:

- Asks to run, execute, or reproduce a test case.
- Says "ejecutá el caso N", "corré el test N", "run the happy path".
- Asks to list TESSA test cases / projects.
- Asks for steps of a specific test case.
- Wants to report results or upload screenshots.
- Wants to generate test cases from a functional document (PDF/Word).
- Mentions TESSA, QualisLab, or automated testing.

## The 8 tools

### `list_projects({ page, pageSize })`
Paginated list of the projects the user can access (id + name).
→ Use FIRST to pick a project.
Returns `{ projects: [{id, name}], pagination: {...} }`.

### `list_test_cases({ projectId, page, pageSize })`
Paginated list of the test cases in a project. **Requires `projectId`** (from `list_projects`).
→ Use when no specific `caseId` was given, after choosing a project.
Returns `{ projectId, projectName, cases: [{caseId, title, status}], pagination: {...} }`.

### `fetch_test_case({ caseId })`
Happy path details.
Returns `{ testCaseId, title, steps: [{stepNumber, action}] }`.
**The `action` fields are natural-language descriptions** — translate them into concrete browser actions (navigate, click, fill, wait, etc.).

### `fetch_additional_cases({ caseId })`
Alternative / edge-case scenarios.
Only call when user asked for "all cases" or after the happy path failed.

### `get_presigned_url({ fileName, contentType, caseId? })`
**Use for every screenshot.** Returns `{ uploadUrl, publicUrl }`.
- Allowed `contentType`: `image/png`, `image/jpeg`, `image/jpg`, `image/webp`.
- PUT the binary image to `uploadUrl` with correct `Content-Type` header.
- Save `publicUrl` for later inclusion in `submit_test_result`.
- Always pass `caseId` when available — organizes the S3 bucket + validates ownership.

### `submit_test_result({ caseId, status, executedUrl, totalDurationMs?, steps })`
Final execution report.
- `status`: one of `PASS`, `FAIL`, `ERROR`, `SKIPPED`.
- Each step: `{ stepNumber, description, status, durationMs?, screenshotUrl?, errorMessage? }`.
- `screenshotUrl` must be a `publicUrl` from `get_presigned_url`. **No base64.**
- Global status rule: worst wins (FAIL > ERROR > SKIPPED > PASS).

### `create_analysis_draft({ projectId, fileName, contentType })`
Step 1 of generating test cases from a functional document. Creates a `DRAFT` process and returns a presigned S3 **PUT** URL to upload the document.
- `projectId` (number), `fileName` (string, ≤255), `contentType` (string).
- Allowed `contentType`: `application/pdf` or `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (docx) **only**.
- Returns `{ processId, uploadUrl, publicUrl }`.
- Then HTTP **PUT** the document binary to `uploadUrl` with the matching `Content-Type`.
- **Gotcha:** the URL is signed with only `host;content-type` headers — do NOT add any `x-amz-checksum-*` header (even though it appears in the query string) or S3 returns `403`.
- Validates project access + `CREATE_EXECUTIONS` permission server-side.

### `generate_analysis({ processId, fileUrl, documentType, originalName?, prompt?, industry?, functionality?, platform?, additionals?, gherkin?, uxUi? })`
Step 2 (after `create_analysis_draft` + uploading the file). Generates test cases **asynchronously** from the uploaded document.
- `processId` (number), `fileUrl` (string, ≤2000), `documentType`: `'pdf' | 'word'`.
- Optional: `originalName`, `prompt` (≤5000), `industry`, `functionality`, `platform`, `additionals` (default `false`), `gherkin` (default `false`), `uxUi` (default `false`).
- Returns `{ processId, message }` — **ASYNC**; poll afterwards (e.g. `list_test_cases`) until the case is `PROCESADO`.
- `fileUrl` must be the `publicUrl` from step 1 (must point to this process's folder — anti-SSRF). Process must be `DRAFT` and owned by the caller. Uses the company's active LLM provider.

## End-to-end workflow

```
1. If no caseId → list_projects → pick project → list_test_cases({ projectId }) → user chooses
2. fetch_test_case → get steps
3. (Optional) fetch_additional_cases
4. CONFIRM with user: target URL + side-effects summary
5. For each step:
   a. Translate action → browser commands
   b. Execute, measure duration
   c. Screenshot
   d. get_presigned_url → PUT binary → save publicUrl
   e. Record step status
6. submit_test_result
7. Summary to user: global status + failing steps + screenshot URLs
```

## Doc-based generation workflow (PDF/Word → test cases)

```
1. create_analysis_draft({ projectId, fileName, contentType }) → { processId, uploadUrl, publicUrl }
2. HTTP PUT the PDF/docx binary to uploadUrl (set Content-Type only — NO x-amz-checksum-* header)
3. generate_analysis({ processId, fileUrl: publicUrl, documentType, ...flags }) → { processId, message }
4. Poll list_test_cases until the case is PROCESADO, then fetch_test_case for the steps
```

## Hard rules

1. **Confirm before executing.** Test cases can touch real systems (payments, signups, external APIs). Before running: state target URL + known side-effects + ask for explicit confirmation. Wait for a "yes".

2. **Target URL is mandatory.** Never assume `localhost`, `example.com`, or staging without being told. If the user didn't provide one, ask.

3. **Screenshots via presigned URLs, never base64.** Base64 inline in `submit_test_result` bloats the DB and breaks access control. Always: `get_presigned_url` → PUT to S3 → pass `publicUrl`.

4. **Project access is server-enforced. You only see/act on projects you're a member of.** Errors like `"Project not accessible"` or `"Test case (process) not found"` mean the project or `caseId` isn't reachable by your token. Don't try to work around it — verify with `list_projects` → `list_test_cases({ projectId })`.

5. **Redact credentials.** Real passwords or API keys must NEVER appear in `errorMessage` or step descriptions. Use `pass=***` or similar.

6. **Don't spam executions.** Max reasonable size for `steps[]` is ~100. Don't run 50 cases in parallel — no server-side rate limiting yet.

## Error handling

| Error | Response |
|---|---|
| `401 Invalid API token` / `Authentication required` | Tell user to regenerate in TESSA → Settings → API Tokens |
| `Invalid caseId` | Call `list_test_cases({ projectId })` to find valid IDs |
| `Invalid content type` | Convert screenshot to PNG before retry |
| `Project not found` | Use `list_projects` to find valid project IDs |
| `Project not accessible` | You're not a member of that project — pick one from `list_projects` |
| `Test case (process) not found` / `not accessible` | Verify `caseId` with `list_test_cases({ projectId })` |

## Step failures during execution

- **Assertive failure** (expected ≠ actual): continue remaining steps, mark this step `FAIL`.
- **Structural failure** (element not found, timeout, network): abort remaining, mark as `SKIPPED`.
- Always screenshot the actual error state (not the expected state).
- Populate `errorMessage` with a clear, concise diagnosis.

## Example interaction

> **User**: Corré el caso 375 en https://staging.acme.com
>
> **You**:
> 1. `fetch_test_case({ caseId: "375" })` → 5 steps retrieved.
> 2. "Voy a ejecutar 'Test QR Payment' (5 pasos) en https://staging.acme.com. Incluye un pago simulado de $500. ¿Confirmás?"
> 3. (User: sí)
> 4. Execute each step, screenshot, upload via `get_presigned_url`, collect results.
> 5. `submit_test_result({ caseId: "375", status: "PASS", executedUrl: "https://staging.acme.com", totalDurationMs: 8420, steps: [...] })`.
> 6. "Ejecución OK. Status: PASS. Duración 8.4s. Screenshots:
>    - Paso 1: https://bucket.s3.amazonaws.com/executions/375/...
>    - Paso 2: ..."

## Equivalent REST endpoints (fallback if MCP unavailable)

| Tool | REST |
|---|---|
| `list_projects` | `GET /api/mcp/projects?page=N&pageSize=N` |
| `list_test_cases` | `GET /api/mcp/projects/:projectId/test-cases?page=N&pageSize=N` |
| `fetch_test_case` | `GET /api/mcp/test-case/:caseId` |
| `fetch_additional_cases` | `GET /api/mcp/test-case/:caseId/additional-cases` |
| `get_presigned_url` | `GET /api/mcp/presigned-url?fileName=...&contentType=...&caseId=...` |
| `submit_test_result` | `POST /api/mcp/test-result` |

`create_analysis_draft` and `generate_analysis` are **MCP-only** — they have no REST equivalent.

Always with `Authorization: Bearer qai_<token>`.
