# Testcase API Reference

Base URL is project-specific. Read it from `.testcases/info.md` as `testcase_app_url`; local development often uses `http://localhost:3001`, and deployed/internal installs may use another host or port.

All developer endpoints live under `/api/v1`.

## Auth

Use bearer auth:

```http
Authorization: Bearer <personal API key or shared token>
Content-Type: application/json
```

Supported bearer values:

- Personal API key: starts with `tcsk_`, created from Settings > API keys, shown only once, same role and project access as the owning user, revoked by deleting it from Settings.
- Shared token: `TESTCASE_API_TOKEN`, acting as `TESTCASE_API_USERNAME` or `BOOTSTRAP_ADMIN_USERNAME`.

Prefer personal API keys for user-scoped work. Store the active key/token in `.testcases/info.md` when the user provides or approves it.

Personal keys are normally created in the browser at Settings > API keys. The backing route is `/settings/api-keys` and requires a signed-in browser session; bearer auth is for `/api/v1` work after a key exists.

PowerShell discovery:

```powershell
$base = "http://localhost:3001"
$token = $env:TESTCASE_API_KEY
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod -Headers $headers "$base/api/v1"
```

For Persian JSON, ensure UTF-8. If the shell might corrupt Persian text, write the payload to a UTF-8 file and let Node read it:

```powershell
@'
{
  "releaseId": "release_id",
  "title": "تست ورود",
  "path": "/login",
  "howToTest": "با کاربر معتبر وارد شوید."
}
'@ | Set-Content -Encoding utf8 .testcases/payload.json

node -e "const fs=require('fs'); const p=JSON.parse(fs.readFileSync('.testcases/payload.json','utf8')); fetch(process.env.TESTCASE_APP_URL + '/api/v1/test-cases', {method:'POST', headers:{Authorization:'Bearer '+process.env.TESTCASE_API_KEY,'Content-Type':'application/json'}, body:JSON.stringify(p)}).then(async r=>{console.log(r.status, await r.text())})"
```

## Response Shapes

List endpoints:

```json
{ "data": [], "meta": { "total": 0, "limit": 50, "offset": 0 } }
```

Single-record and mutation endpoints:

```json
{ "data": {} }
```

Errors:

```json
{ "error": { "message": "Human-readable error", "details": {} } }
```

Common status codes: `200`, `201`, `400`, `401`, `403`, `404`, `422`, `500`.

Use `GET /api/v1` to discover available routes.

## Statuses

Test case status is one of:

```text
created | pending | passed | failed
```

New JSON test cases default to `created` when `status` is omitted. Bug reports are saved as `failed`.

## Access Model

Admins can see and manage everything. Non-admin users see only apps/projects/releases/test cases for projects they have access to. Developers are automatically granted access to projects they create. Personal API keys inherit the owning user's role and project access.

## Apps

List:

```http
GET /api/v1/apps?q=&status=&limit=&offset=
```

`status` can be `created | pending | passed | failed`. Responses include accessible projects, releases, test cases, and rollup counts:

```json
{
  "counts": {
    "projects": 1,
    "releases": 2,
    "testCases": {
      "total": 6,
      "byStatus": { "created": 2, "pending": 1, "passed": 2, "failed": 1 },
      "media": 3
    }
  }
}
```

Create/update/delete:

```http
POST /api/v1/apps
PATCH /api/v1/apps/:id
DELETE /api/v1/apps/:id
```

Create body:

```json
{ "name": "App name", "description": "Optional description" }
```

## Projects

List:

```http
GET /api/v1/projects?q=&appId=&status=&limit=&offset=
```

Project responses include app, releases, test cases, and `counts` with release totals, test-case status counts, and media totals.

Create/update/delete:

```http
POST /api/v1/projects
PATCH /api/v1/projects/:id
DELETE /api/v1/projects/:id
```

Create body:

```json
{ "appId": "app_id", "name": "Project name", "description": "Optional description" }
```

## Releases

List:

```http
GET /api/v1/releases?q=&appId=&projectId=&status=&sort=&limit=&offset=
```

`sort`: `createdDesc | updated | name`

Release responses include project/app context, test cases, and `counts.testCases` with `total`, `byStatus`, and `media`.

Create/update/delete:

```http
POST /api/v1/releases
PATCH /api/v1/releases/:id
DELETE /api/v1/releases/:id
```

Create body:

```json
{ "projectId": "project_id", "name": "Release name", "notes": "Optional notes" }
```

## Test Cases

List:

```http
GET /api/v1/test-cases?q=&appId=&projectId=&releaseId=&creatorId=&status=&updatedSince=&sort=&limit=&offset=
```

`sort`: `updated | oldest | title | status`

Detail:

```http
GET /api/v1/test-cases/:id
```

List/detail responses include comments, attachments, directAttachments, commentAttachments, media, and counts when available. For failed cases, use detail plus attachments endpoints before diagnosing the failure.

Create JSON test case:

```http
POST /api/v1/test-cases
```

```json
{
  "releaseId": "release_id",
  "title": "Short Persian test title",
  "description": "Persian feature and expected behavior",
  "path": "/settings/security",
  "howToTest": "Persian end-user test steps.",
  "status": "created"
}
```

`status` is optional and defaults to `created`.

Create multipart test case with evidence:

```bash
curl -X POST "$TESTCASE_APP_URL/api/v1/test-cases" \
  -H "Authorization: Bearer $TESTCASE_API_KEY" \
  -F "releaseId=release_id" \
  -F "title=Smoke test login" \
  -F "path=/login" \
  -F "howToTest=Login with a valid local user." \
  -F "images=@./screenshot.png"
```

Tester bug report:

```json
{
  "releaseId": "release_id",
  "reportType": "bug",
  "title": "Payment button stays disabled",
  "path": "/checkout/payment",
  "description": "Chrome on staging with a saved card",
  "steps": "Open checkout, choose saved card, try to continue.",
  "expected": "The user can continue to review the order.",
  "actual": "The button remains disabled and no validation message appears."
}
```

Tester users can only create test cases as bug reports. Bug reports are saved as failed test cases.

Update/delete:

```http
PATCH /api/v1/test-cases/:id
DELETE /api/v1/test-cases/:id
```

`PATCH` and `DELETE` test-case definition operations require an admin or developer user/API key. Testers can read accessible cases, create bug reports, upload evidence, and change status, but cannot edit test definitions.

Status change:

```http
POST /api/v1/test-cases/:id/status
```

```json
{ "status": "passed", "comment": "Verified by QA." }
```

Status changes also accept multipart `status`, optional `comment`, and evidence files.

## Created Placeholder Detailing

Manual quick-add cases may be in `created` status with only a title and little or no detail. Agents should use code, routes, commits, and project context to fill these in.

Find created cases:

```http
GET /api/v1/test-cases?releaseId=<releaseId>&status=created&sort=oldest&limit=100
GET /api/v1/test-cases?projectId=<projectId>&status=created&sort=oldest&limit=100
GET /api/v1/test-cases?appId=<appId>&status=created&sort=oldest&limit=100
```

Update a placeholder into a ready-to-test case:

```http
PATCH /api/v1/test-cases/:id
```

```json
{
  "title": "Keep or improve the existing title",
  "description": "Persian feature and expected behavior",
  "path": "/exact/path",
  "howToTest": "Persian QA steps",
  "status": "pending"
}
```

Keep status as `created` only when the case is still too ambiguous or the user prefers draft placeholders.

If no created cases are found for the selected release/project/app, report that clearly. If `PATCH` returns 403, ask for an admin/developer personal API key.

## Bugfix Flow

Use failed cases as the work queue for `/bugfix`.

Find failed cases:

```http
GET /api/v1/test-cases?releaseId=<releaseId>&status=failed&sort=updated&limit=100
GET /api/v1/test-cases?projectId=<projectId>&status=failed&sort=updated&limit=100
GET /api/v1/test-cases?appId=<appId>&status=failed&sort=updated&limit=100
```

For each candidate, fetch full context:

```http
GET /api/v1/test-cases/:id
GET /api/v1/test-cases/:id/attachments
```

Review comments, status-change comments, directAttachments, commentAttachments, media, image URLs, title, description, path, and howToTest before proposing code fixes.

If many failed cases are returned, group by route/feature and ask which group to fix first unless the user asked to fix all. If no failed cases are returned, report that there is nothing to fix for the selected scope.

After a fix is implemented and verified, move the case back to pending for QA retry:

```http
POST /api/v1/test-cases/:id/status
```

```json
{
  "status": "pending",
  "comment": "Fixed the issue and moved this case back to pending for QA retry."
}
```

Do not mark a case `passed` after code changes unless the user explicitly asks the agent to perform QA verification as the tester.

## Media And Evidence

Test case responses include:

```json
{
  "attachments": [],
  "comments": [],
  "directAttachments": [],
  "commentAttachments": [],
  "media": [],
  "counts": {
    "comments": 0,
    "directAttachments": 0,
    "commentAttachments": 0,
    "media": 0
  }
}
```

Use `directAttachments`, `commentAttachments`, or `media` for UI-like evidence handling without double counting.

List/upload/delete evidence:

```http
GET /api/v1/test-cases/:id/attachments
POST /api/v1/test-cases/:id/attachments
DELETE /api/v1/test-cases/:id/attachments/:attachmentId
```

Upload uses multipart form data. Accepted file fields: `images`, `media`, `files`, `evidence`.

## Personal Secret Collections

Secret collections are personal to the authenticated user. List responses include decrypted values for that same user.

List/create/update/delete collections:

```http
GET /api/v1/secret-collections?q=&limit=&offset=
POST /api/v1/secret-collections
PATCH /api/v1/secret-collections/:id
DELETE /api/v1/secret-collections/:id
```

Create body:

```json
{ "name": "Staging credentials", "description": "Personal staging logins" }
```

List/upsert/read/update/delete values:

```http
GET /api/v1/secret-collections/:id/secrets?q=&limit=&offset=
POST /api/v1/secret-collections/:id/secrets
GET /api/v1/secret-collections/:id/secrets/:secretId
PATCH /api/v1/secret-collections/:id/secrets/:secretId
DELETE /api/v1/secret-collections/:id/secrets/:secretId
```

Upsert value:

```json
{ "key": "STAGING_USERNAME", "value": "qa-user" }
```

To update, read, or delete by key, list collection values first and use the returned `secretId`.

## Global Secrets

All signed-in users can read global secrets. Only admins can create, update, or delete them.

```http
GET /api/v1/global-secrets?q=&limit=&offset=
POST /api/v1/global-secrets
GET /api/v1/global-secrets/:id
PATCH /api/v1/global-secrets/:id
DELETE /api/v1/global-secrets/:id
```

Create or replace:

```json
{ "key": "GLOBAL_API_BASE_URL", "value": "https://staging.example.com", "description": "Shared staging API base URL" }
```

## Recommended Agent Flow

1. Validate auth with `GET /api/v1`.
2. Discover hierarchy:

```http
GET /api/v1/apps?limit=100
GET /api/v1/projects?appId=<appId>&limit=100
GET /api/v1/releases?projectId=<projectId>&limit=100
```

3. Update `.testcases/info.md` with confirmed IDs and names.
4. Create missing hierarchy records if needed.
5. Create cases with `POST /api/v1/test-cases`.
6. Find not-yet-run cases:

```http
GET /api/v1/test-cases?releaseId=<releaseId>&status=created&sort=oldest&limit=50
```

7. Mark result with `POST /api/v1/test-cases/:id/status`.
8. Audit recent changes with `GET /api/v1/test-cases?updatedSince=<iso-date>&sort=updated`.
9. For `/bugfix`, fetch failed cases and attachments, discuss with the user, ask permission, fix code, then move fixed cases to `pending`.

Prefer IDs over names once discovered. Use `limit` and `offset` for pagination. List before creating apps/projects/releases to avoid duplicates.
