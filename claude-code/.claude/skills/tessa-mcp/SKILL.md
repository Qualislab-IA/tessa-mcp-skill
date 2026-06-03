---
name: tessa-mcp
description: Use when the user asks to run, list, inspect, or report test cases via the TESSA MCP server (at https://agent.qualis-lab.com/mcp). Orchestrates the 8 tools (list_projects, list_test_cases, fetch_test_case, fetch_additional_cases, get_presigned_url, submit_test_result, create_analysis_draft, generate_analysis) in the correct order. Triggers on mentions of TESSA, QualisLab, "test case N", "ejecutá el caso N", "happy path", "generá casos desde este documento", "subí este PDF/Word y generá casos", or similar testing workflows.
---

# TESSA MCP Skill (Claude Code)

TESSA is a SaaS platform that generates AI-driven test cases and executes them via MCP-connected agents. This skill teaches you how to use the TESSA MCP server correctly.

## When this skill activates

Any of these user intents:

- "Run test case N" / "Ejecutá el caso de prueba N en [URL]"
- "List my TESSA tests / projects"
- "What are the steps of case N?"
- "Run the happy path of [project]"
- "Upload these screenshots to process N" / "Report the results"
- Any mention of TESSA, QualisLab, or automated test cases.

## Server

- **Production URL**: `https://agent.qualis-lab.com/mcp`
- **Auth**: Bearer token with `qai_` prefix (TESSA API token).
- Tools auto-discovered via `tools/list` on connection.
- Ensure the MCP server is configured in `~/.claude.json` or project `.mcp.json`.

## The 8 tools

### 1. `list_projects`
Paginated list of the **projects** the user can access (server-enforced: only projects of their company where they're a member).
- **Input**: `{ page: number, pageSize: number }`
- **Output**: `{ projects: [{id, name}], pagination: {...} }`
- **Use first** when no specific `caseId` is mentioned — pick a project, then list its cases.

### 2. `list_test_cases`
Paginated list of the **test cases** of a project the user can access.
- **Input**: `{ projectId: number, page: number, pageSize: number }`
- **Output**: `{ projectId, projectName, cases: [{caseId, title, status}], pagination: {...} }`
- Requires `projectId` (from `list_projects`). Returns cases in `PROCESADO` status. The `caseId` feeds the other tools.

### 3. `fetch_test_case`
Happy-path details of a test case.
- **Input**: `{ caseId: string }`
- **Output**: `{ testCaseId, title, steps: [{stepNumber, action}] }`
- Steps are **natural language descriptions**. You translate them to concrete browser actions.

### 4. `fetch_additional_cases`
Alternative / edge-case scenarios.
- **Input**: `{ caseId: string }`
- **Output**: `{ testCaseId, totalAdditionalCases, additionalCases: [...] }`
- Call after `fetch_test_case` only if user asked for "all cases" or happy path failed.

### 5. `get_presigned_url`
Generate a pre-signed S3 URL for uploading a screenshot. **Use this for every screenshot — never inline base64.**
- **Input**: `{ fileName: string, contentType: string, caseId?: string }`
- **Allowed contentType**: `image/png`, `image/jpeg`, `image/jpg`, `image/webp` only.
- **Output**: `{ uploadUrl, publicUrl }` — PUT the binary to `uploadUrl` with correct `Content-Type`, then save `publicUrl`.
- **Always pass `caseId`** when available — it organizes uploads and validates project access.

### 6. `submit_test_result`
Final execution report.
- **Input**: `{ caseId, status, executedUrl, totalDurationMs?, steps: [{stepNumber, description, status, durationMs?, screenshotUrl?, errorMessage?}] }`
- **status**: one of `PASS | FAIL | ERROR | SKIPPED`.
- **Global status rule**: if any step isn't PASS, global status can't be PASS (use worst: FAIL > ERROR > SKIPPED > PASS).
- `screenshotUrl` must come from `get_presigned_url.publicUrl`.

### 7. `create_analysis_draft`
Step 1 of generating test cases from a functional document. Creates a DRAFT process and returns a presigned S3 PUT URL to upload the document.
- **Input**: `{ projectId: number, fileName: string (≤255), contentType: string }`
- **Allowed contentType**: `application/pdf` or `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (docx) only.
- **Output**: `{ processId, uploadUrl, publicUrl }`
- After this, HTTP PUT the document binary to `uploadUrl` with the matching `Content-Type`. **Gotcha**: the URL is signed with only `host;content-type` — do NOT send any `x-amz-checksum-*` header (even though it appears in the query string) or S3 returns 403.
- Validates project access + the `CREATE_EXECUTIONS` permission server-side.

### 8. `generate_analysis`
Step 2 (after create_analysis_draft + uploading the file). Generates test cases (asynchronously) from the uploaded document.
- **Input**: `{ processId: number, fileUrl: string (≤2000), documentType: 'pdf' | 'word', originalName?: string, prompt?: string (≤5000), industry?: string, functionality?: string, platform?: string, additionals?: boolean, gherkin?: boolean, uxUi?: boolean }`
- `additionals`/`gherkin`/`uxUi` default to **false**. `industry`/`functionality`/`platform` have sensible defaults.
- **Output**: `{ processId, message }` — generation is **async**. Poll afterwards (e.g. `list_test_cases({ projectId })`) until the new case appears in `PROCESADO`.
- `fileUrl` must be the exact `publicUrl` from step 1 (it must point to this process's folder — anti-SSRF). Process must be DRAFT and owned by the caller. Uses the company's active LLM provider.

## Recommended end-to-end flow

```
1. [If no caseId given]
   → list_projects
   → list_test_cases({ projectId })   (cases of the chosen project)
   → show to user, ask which one to run

2. → fetch_test_case
3. [Optional] → fetch_additional_cases (if user asked for all cases)

4. PRE-EXECUTION CHECK
   Confirm with user: target URL, credentials if any, approval to run.
   DO NOT proceed without explicit "yes".

5. FOR EACH STEP:
   a. Translate step.action into concrete browser actions (navigate/click/fill/wait).
   b. Execute. Measure durationMs.
   c. Take screenshot.
   d. get_presigned_url → PUT binary to uploadUrl → save publicUrl.
   e. Record step status + publicUrl + error if any.

6. → submit_test_result with full steps array.
7. Summarize to user: global status, failed steps, screenshot links.
```

## Document-based generation flow

```
1. create_analysis_draft({ projectId, fileName, contentType }) → { processId, uploadUrl, publicUrl }
2. HTTP PUT the PDF/docx binary to uploadUrl (Content-Type only; no checksum header)
3. generate_analysis({ processId, fileUrl: publicUrl, documentType, ...flags }) → { processId, message }
4. Poll list_test_cases({ projectId }) until the generated case shows status PROCESADO
5. fetch_test_case({ caseId }) to read the generated steps
```

## Anti-failure patterns

### Always confirm before executing
Test cases can hit real systems (payments, account creation, etc.). **Always** ask for explicit confirmation with target URL and side-effects summary.

> "I'm going to run case 375 'Test QR Payment' on staging.example.com, which includes a simulated $500 payment. Confirm?"

No execution without an explicit "yes".

### Execution URL is mandatory
Never assume `https://example.com` or any placeholder. If the user didn't provide the target URL, **ask for it**.

### Screenshots via presigned URL, never base64
- ❌ Embed base64 inline in `submit_test_result`.
- ✅ `get_presigned_url` → PUT to S3 → pass `publicUrl` to `submit_test_result`.

### Error handling in steps
When a step fails:
1. Screenshot the **error state** (not the expected state).
2. Set `status: "FAIL"` or `"ERROR"` with a meaningful `errorMessage`.
3. Decide whether to continue or abort:
   - Assertive failure (value mismatch): continue, next steps might pass.
   - Structural failure (element not found, timeout, network): abort remaining steps as `SKIPPED`.

### Project access is enforced server-side
You only see and act on projects you're a member of. Errors like `"Project not accessible"` or `"Test case (process) not accessible"` mean the `projectId`/`caseId` isn't reachable with your token. No workarounds — ask user to verify with `list_projects` → `list_test_cases({ projectId })`.

## Common errors

| Error | Cause | Action |
|---|---|---|
| `401 Invalid API token` | Token invalid/revoked | Ask user to regenerate in TESSA → Settings → API Tokens |
| `Authentication required` | Call arrived without an authenticated user | Ensure the API token is sent as `Authorization: Bearer qai_...` |
| `Project not found` | `projectId` doesn't exist | Call `list_projects` for valid IDs |
| `Project not accessible` | Not a member of that project (or other company) | Use only `projectId`s from `list_projects` |
| `Invalid caseId` | Non-numeric or missing process | Call `list_test_cases({ projectId })` |
| `Invalid content type` | Screenshot not png/jpeg/webp | Convert to PNG |
| `Test case (process) not found` / `not accessible` | Case missing or in a project you can't see | Verify `caseId` with `list_test_cases({ projectId })` |
| `Invalid content type` (on create_analysis_draft) | contentType not pdf/docx | Use application/pdf or the docx MIME |
| `Process ... is not a draft` | processId already used/generated | Call create_analysis_draft again for a fresh draft |
| `Invalid file URL: must point to this process folder` | fileUrl doesn't match the process | Pass the exact publicUrl from create_analysis_draft |

## Example conversation

**User**: Ejecutá el caso 375 en staging.example.com

**You (internally)**:
1. Call `fetch_test_case({ caseId: "375" })` → 5 steps returned.
2. Respond: "Voy a ejecutar 'Test QR Payment' (5 pasos) en staging.example.com. Incluye pago simulado de $500. ¿Confirmás?"
3. User confirms.
4. For each step: browser action → screenshot → `get_presigned_url` → PUT.
5. `submit_test_result(...)` with all 5 steps.
6. Respond: "Ejecución OK. Status: PASS. 8.4s. Screenshots: ..."

## Boundaries

- `steps[]` max ~100 items.
- Screenshots ideally <2MB each.
- Don't run 50 executions in parallel — no rate limiting yet server-side.
- **Never log real passwords** in `errorMessage`. Redact as `pass=***`.

---

See `README.md` at the project root or in the installation package for configuration details.
