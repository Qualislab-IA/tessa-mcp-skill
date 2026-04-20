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
- Mentions TESSA, QualisLab, or automated testing.

## The 5 tools

### `list_test_cases({ page, pageSize })`
Paginated list of the user's test cases.
→ Use FIRST when no specific `caseId` was given.
Returns `{ projects: [{id, name}], pagination: {...} }`.

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

## End-to-end workflow

```
1. If no caseId → list_test_cases → user chooses
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

## Hard rules

1. **Confirm before executing.** Test cases can touch real systems (payments, signups, external APIs). Before running: state target URL + known side-effects + ask for explicit confirmation. Wait for a "yes".

2. **Target URL is mandatory.** Never assume `localhost`, `example.com`, or staging without being told. If the user didn't provide one, ask.

3. **Screenshots via presigned URLs, never base64.** Base64 inline in `submit_test_result` bloats the DB and breaks access control. Always: `get_presigned_url` → PUT to S3 → pass `publicUrl`.

4. **Ownership is server-enforced.** Errors like `"Process not found or not owned by caller"` mean the `caseId` doesn't belong to your token. Don't try to work around it — verify with `list_test_cases`.

5. **Redact credentials.** Real passwords or API keys must NEVER appear in `errorMessage` or step descriptions. Use `pass=***` or similar.

6. **Don't spam executions.** Max reasonable size for `steps[]` is ~100. Don't run 50 cases in parallel — no server-side rate limiting yet.

## Error handling

| Error | Response |
|---|---|
| `401 Invalid API token` | Tell user to regenerate in TESSA → Settings → API Tokens |
| `Invalid caseId` | Call `list_test_cases` to find valid IDs |
| `Invalid content type` | Convert screenshot to PNG before retry |
| `Process not found or not owned by caller` | Verify `caseId` with `list_test_cases` |

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
| `list_test_cases` | `GET /api/mcp/test-cases?page=N&pageSize=N` |
| `fetch_test_case` | `GET /api/mcp/test-case/:caseId` |
| `fetch_additional_cases` | `GET /api/mcp/test-case/:caseId/additional-cases` |
| `get_presigned_url` | `GET /api/mcp/presigned-url?fileName=...&contentType=...&caseId=...` |
| `submit_test_result` | `POST /api/mcp/test-result` |

Always with `Authorization: Bearer qai_<token>`.
