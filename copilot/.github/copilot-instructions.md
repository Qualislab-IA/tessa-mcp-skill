# TESSA MCP — Copilot Instructions

This project integrates with **TESSA** (Test Execution & Smart Synthesis Agent), a SaaS for AI-generated test case execution via MCP.

**MCP server**: `https://agent.qualis-lab.com/mcp`
**Auth**: Bearer `qai_<token>` (TESSA API token).

> **Note on Copilot MCP support**: if your Copilot version supports MCP natively, the tools will appear automatically. Otherwise, use these instructions to generate correct code against the equivalent REST endpoints (`/api/mcp/*`).

## The 6 TESSA MCP tools

1. **`list_projects({ page, pageSize })`** — lists the projects the user can access (id + name). Use first. Returns `{ projects: [{id,name}], pagination }`.
2. **`list_test_cases({ projectId, page, pageSize })`** — lists the test cases in a project. Requires `projectId`. Returns `{ projectId, projectName, cases: [{caseId,title,status}], pagination }`.
3. **`fetch_test_case({ caseId })`** — returns `{ testCaseId, title, steps: [{stepNumber, action}] }`.
4. **`fetch_additional_cases({ caseId })`** — returns edge-case scenarios.
5. **`get_presigned_url({ fileName, contentType, caseId? })`** — returns `{ uploadUrl, publicUrl }` for S3 screenshot upload.
6. **`submit_test_result({ caseId, status, executedUrl, totalDurationMs?, steps })`** — reports execution outcome.

## Correct workflow

1. If no `caseId`: `list_projects` first → pick a project → `list_test_cases({ projectId })`, let user choose.
2. `fetch_test_case` to get steps.
3. Optionally `fetch_additional_cases` for edge cases.
4. **Ask user for confirmation** with target URL + side-effects before executing.
5. For each step: execute browser action → screenshot → `get_presigned_url` → PUT binary to `uploadUrl` → save `publicUrl`.
6. `submit_test_result` with all steps and their `screenshotUrl` = `publicUrl`.

## Critical rules

- **Always ask for explicit user confirmation** before running a test case (tests can hit real systems).
- **Never invent URLs**. If the target URL for execution isn't provided by the user, ask.
- **Screenshots via `get_presigned_url` + PUT to S3**. Never inline base64 in `submit_test_result`.
- **`contentType`** in `get_presigned_url` must be `image/png`, `image/jpeg`, `image/jpg`, or `image/webp` only.
- **Global `status`** in `submit_test_result` follows worst-step: if any step isn't PASS, global can't be PASS (priority: FAIL > ERROR > SKIPPED > PASS).
- **Project access is server-enforced. You only see/act on projects you're a member of.** Users can only access `caseId`s inside projects they belong to.
- **Never log real passwords** in `errorMessage`. Redact as `pass=***`.
- **Max ~100 steps per execution**. No rate limiting yet — don't parallelize excessively.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `401 Invalid API token` / `Authentication required` | Token invalid, revoked or missing | Regenerate in TESSA → Settings → API Tokens |
| `Invalid caseId` | Non-numeric or missing process | Call `list_test_cases({ projectId })` |
| `Invalid content type` | Not png/jpeg/webp | Convert screenshot to PNG |
| `Project not found` | Unknown project id | Use `list_projects` to find valid project IDs |
| `Project not accessible` | Not a member of that project | Pick a project from `list_projects` |
| `Test case (process) not found` / `not accessible` | IDOR blocked / wrong project | Verify `caseId` with `list_test_cases({ projectId })` |

## Code generation hints

When generating code that calls TESSA:

- Use `fetch` or `axios` with `Authorization: Bearer qai_...` header, never hardcode the token.
- Read token from env var `TESSA_API_TOKEN` or config file, not inline.
- For screenshot upload:
  ```ts
  const { uploadUrl, publicUrl } = await tessa.getPresignedUrl({ fileName, contentType, caseId });
  await fetch(uploadUrl, { method: 'PUT', body: screenshotBuffer, headers: { 'Content-Type': contentType } });
  // then use publicUrl in submit_test_result
  ```
- For `submit_test_result`:
  ```ts
  const overallStatus = steps.some(s => s.status === 'FAIL') ? 'FAIL'
    : steps.some(s => s.status === 'ERROR') ? 'ERROR'
    : steps.some(s => s.status === 'SKIPPED') ? 'SKIPPED'
    : 'PASS';
  ```
- Measure `durationMs` per step (`performance.now()` or `Date.now()` deltas).

## Equivalent REST endpoints (if MCP not available)

| MCP tool | REST equivalent |
|---|---|
| `list_projects` | `GET /api/mcp/projects?page=N&pageSize=N` |
| `list_test_cases` | `GET /api/mcp/projects/:projectId/test-cases?page=N&pageSize=N` |
| `fetch_test_case` | `GET /api/mcp/test-case/:caseId` |
| `fetch_additional_cases` | `GET /api/mcp/test-case/:caseId/additional-cases` |
| `get_presigned_url` | `GET /api/mcp/presigned-url?fileName=...&contentType=...&caseId=...` |
| `submit_test_result` | `POST /api/mcp/test-result` with body |

All endpoints require `Authorization: Bearer qai_<token>` header.

## Do NOT

- Don't generate code that writes screenshots as base64 into `submit_test_result`.
- Don't fabricate `caseId` values. If unknown, use `list_projects` → `list_test_cases({ projectId })`.
- Don't execute tests without user confirmation.
- Don't assume default URLs like `localhost` or `example.com`.
- Don't run tests in parallel batches larger than 5 without explicit user OK.
