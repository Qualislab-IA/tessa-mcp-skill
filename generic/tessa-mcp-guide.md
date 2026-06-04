# TESSA MCP — Generic Agent Guide

Use this markdown as context/rules for any IDE or agent that supports MCP but isn't explicitly covered by the other packaged variants (Windsurf, Cline, Continue, Zed, Roo Code, Aider, etc.).

## Server

- **URL**: `https://agent.qualis-lab.com/mcp`
- **Auth**: Bearer token with prefix `qai_` (TESSA API token).
- Get a token: login in TESSA frontend → Settings → API Tokens → Create.

## Tools (6)

| Tool | Input | Purpose |
|---|---|---|
| `list_projects` | `{ page, pageSize }` | Paginated list of the **projects** the user can access (id + name) |
| `list_test_cases` | `{ projectId, page, pageSize }` | Test cases of an accessible project, in **all states** (returns project name + cases with their `status`) |
| `fetch_cases` | `{ processId, includeHappyPath?, includeAdditionals?, includeGherkin?, includeUxUi? }` | Generated cases of a process in **one call** (replaces the old `fetch_test_case` + `fetch_additional_cases`) |
| `get_presigned_url` | `{ fileName, contentType, caseId? }` | Pre-signed S3 URL for screenshot upload |
| `submit_test_result` | `{ caseId, status, executedUrl, totalDurationMs?, steps }` | Final execution report |
| `generate_analysis` | `{ projectId, documentText, ...optional }` | Generates test cases (async) from the **plain text** of a functional document, in a single call |

`list_projects` returns `{ projects: [{ id, name }], pagination }`. `list_test_cases` returns `{ projectId, projectName, cases: [{ caseId, title, status }], pagination }` — cases come back in **every state** (`DRAFT`, `INICIADO`, `AWAITING_APPROVAL`, `PROCESADO`, `ERROR`), each carrying its own `status`; only `PROCESADO` cases are ready to execute. The `caseId` (the `processId`) is what every other tool consumes.

### `fetch_cases` details

Brings the generated cases of a process (happy path, additional cases, Gherkin scenarios, UX/UI analysis) in a **single call**. By default it returns **happy path + additionals**; flip `includeGherkin`/`includeUxUi` to add them, or set `includeHappyPath`/`includeAdditionals` to `false` to filter them out.

- Input: only `processId` is required. `includeHappyPath`/`includeAdditionals` default `true`; `includeGherkin`/`includeUxUi` default `false`.
- Output: `{ processId, happyPath, additionals, totalAdditionalCases }`. Fields whose flag is `false` are omitted. With `includeGherkin: true` a `gherkin: [{ name, classification, steps: { given, when[], then[] } }]` array is added; with `includeUxUi: true` a `uxUi: { summary, payload }` object is added (or `null` if the process has no UX/UI analysis).
- Happy-path `steps` are **natural-language descriptions** — you (the agent) translate them into concrete browser actions.
- For happy path only: `fetch_cases({ processId, includeAdditionals: false })`. For "run all cases": keep the default (happy + additionals).

## Correct workflow

1. If user didn't specify a `caseId` → call `list_projects`, pick the project, then `list_test_cases({ projectId })` and ask which case.
2. `fetch_cases({ processId })` to get the happy path + additional cases (default brings both). For happy path only: `fetch_cases({ processId, includeAdditionals: false })`. Add `includeGherkin`/`includeUxUi` if requested.
3. **Confirm with user** — target URL, expected side-effects, explicit approval.
4. For each step:
   a. Translate `step.action` (natural language) to browser actions.
   b. Execute, measure `durationMs`.
   c. Screenshot.
   d. `get_presigned_url` → HTTP PUT binary to `uploadUrl` → save `publicUrl`.
   e. Record step result (status, duration, publicUrl, errorMessage).
5. `submit_test_result` with full steps array.
6. Summarize: global status, failed steps, screenshot URLs.

## Generating test cases from a document (generate_analysis)

Single async call to turn a functional document into test cases. The document travels as **plain text** in `documentText` — there is **no file upload** (no base64, no presigned URLs).

1. If you have a PDF/DOCX, extract its contents as plain text first.
2. `generate_analysis({ projectId, documentText, prompt?, industry?, functionality?, platform?, additionals?, gherkin?, uxUi? })`. Returns `{ processId, message }` — generation is **ASYNCHRONOUS**; the output only confirms it started.
3. Poll `list_test_cases({ projectId })` and watch the case's `status` until it turns `PROCESADO` (or `ERROR`), then call `fetch_cases({ processId })` to read the cases.

Input specs:

| Tool | Input | Constraints |
|---|---|---|
| `generate_analysis` | `projectId` | positive int, **required** |
| | `documentText` | plain text, **required**, max 2 MB |
| | `prompt?` | string, max 5000 |
| | `industry?` | string, max 200 |
| | `functionality?` | string, max 200 |
| | `platform?` | string, max 200 |
| | `additionals?` | bool, default false |
| | `gherkin?` | bool, default false |
| | `uxUi?` | bool, default false |

Constraints: validates project access + `CREATE_EXECUTIONS` permission server-side. The provider used is the company's active LLM integration. MCP-only tool (no REST equivalent).

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
| `Project not accessible` (generate_analysis) | You must be a member of the `projectId` and have `CREATE_EXECUTIONS` permission |

## Equivalent REST endpoints (fallback)

If the MCP connection isn't available, same capabilities via REST (same auth):

```
GET  /api/mcp/projects?page=N&pageSize=N
GET  /api/mcp/projects/:projectId/test-cases?page=N&pageSize=N
GET  /api/mcp/presigned-url?fileName=...&contentType=...&caseId=...
POST /api/mcp/test-result   (body: {caseId, status, executedUrl, totalDurationMs?, steps})
```

Header on all: `Authorization: Bearer qai_<token>`.

> `fetch_cases` and `generate_analysis` are MCP-only — no REST equivalent.
