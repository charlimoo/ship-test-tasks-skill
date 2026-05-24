# Testcase API Reference

Base URL is project-specific. Read it from `.testcases/info.md` as `testcase_app_url`; local development usually uses `http://localhost:3001`.

All endpoints live under `/api/v1`.

## Auth

Use bearer auth for agents:

```http
Authorization: Bearer <api_token>
Content-Type: application/json
```

The app maps the bearer token to `TESTCASE_API_USERNAME`; if unset, it falls back to `BOOTSTRAP_ADMIN_USERNAME`.

Agents may call the API with any available HTTP client. In PowerShell:

```powershell
$base = "http://localhost:3001"
$token = $env:TESTCASE_API_TOKEN
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod -Headers $headers "$base/api/v1"
```

For JSON mutations:

```powershell
$headers = @{ Authorization = "Bearer $token"; "Content-Type" = "application/json" }
$body = @{ name = "Release name"; projectId = "project_id" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Headers $headers -Body $body "$base/api/v1/releases"
```

Never hard-code machine-specific paths. The API URL must come from `.testcases/info.md`, environment variables, or the user.

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

## Access Model

Admins can see and manage everything. Non-admin users only see apps, projects, releases, and test cases for projects they have been granted. Developers are automatically granted access to projects they create.

## Apps

List:

```http
GET /api/v1/apps?q=&status=&limit=&offset=
```

Create:

```http
POST /api/v1/apps
```

```json
{ "name": "App name", "description": "Optional description" }
```

Update/delete:

```http
PATCH /api/v1/apps/:id
DELETE /api/v1/apps/:id
```

## Projects

List:

```http
GET /api/v1/projects?q=&appId=&status=&limit=&offset=
```

Create:

```http
POST /api/v1/projects
```

```json
{ "appId": "app_id", "name": "Project name", "description": "Optional description" }
```

Update/delete:

```http
PATCH /api/v1/projects/:id
DELETE /api/v1/projects/:id
```

## Releases

List:

```http
GET /api/v1/releases?q=&appId=&projectId=&status=&sort=&limit=&offset=
```

`status`: `pending | passed | failed`

`sort`: `createdDesc | updated | name`

Create:

```http
POST /api/v1/releases
```

```json
{ "projectId": "project_id", "name": "Release name", "notes": "Optional notes" }
```

Update/delete:

```http
PATCH /api/v1/releases/:id
DELETE /api/v1/releases/:id
```

## Test Cases

List:

```http
GET /api/v1/test-cases?q=&appId=&projectId=&releaseId=&creatorId=&status=&updatedSince=&sort=&limit=&offset=
```

`status`: `pending | passed | failed`

`sort`: `updated | oldest | title | status`

Create:

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
  "status": "pending"
}
```

Update/delete:

```http
PATCH /api/v1/test-cases/:id
DELETE /api/v1/test-cases/:id
```

Change status and preserve an audit note:

```http
POST /api/v1/test-cases/:id/status
```

```json
{ "status": "passed", "comment": "Verified by QA." }
```

Use `PATCH /api/v1/test-cases/:id` when changing the test definition. Use the status endpoint when recording a pass/fail result.

## Personal Secret Collections

Secret collections are personal to the authenticated user. List responses include decrypted values for that same user.

List collections:

```http
GET /api/v1/secret-collections?q=&limit=&offset=
```

Create collection:

```http
POST /api/v1/secret-collections
```

```json
{ "name": "Staging credentials", "description": "Personal staging logins" }
```

Update/delete collection:

```http
PATCH /api/v1/secret-collections/:id
DELETE /api/v1/secret-collections/:id
```

List values in a collection:

```http
GET /api/v1/secret-collections/:id/secrets?q=&limit=&offset=
```

Insert or replace a key value:

```http
POST /api/v1/secret-collections/:id/secrets
```

```json
{ "key": "STAGING_USERNAME", "value": "qa-user" }
```

Read/update/delete one value:

```http
GET /api/v1/secret-collections/:id/secrets/:secretId
PATCH /api/v1/secret-collections/:id/secrets/:secretId
DELETE /api/v1/secret-collections/:id/secrets/:secretId
```

## Global Secrets

All signed-in users can read global secrets. Only admins can create, update, or delete them.

List:

```http
GET /api/v1/global-secrets?q=&limit=&offset=
```

Create or replace:

```http
POST /api/v1/global-secrets
```

```json
{ "key": "GLOBAL_API_BASE_URL", "value": "https://staging.example.com", "description": "Shared staging API base URL" }
```

Read/update/delete:

```http
GET /api/v1/global-secrets/:id
PATCH /api/v1/global-secrets/:id
DELETE /api/v1/global-secrets/:id
```

## Recommended Agent Flow

1. Validate auth with `GET /api/v1`.
2. Discover hierarchy:

```http
GET /api/v1/apps?limit=100
GET /api/v1/projects?appId=<appId>&limit=100
GET /api/v1/releases?projectId=<projectId>&limit=100
```

3. Create missing hierarchy records if needed.
4. Create cases with `POST /api/v1/test-cases`.
5. Audit recent changes with `GET /api/v1/test-cases?updatedSince=<iso-date>&sort=updated`.

Prefer IDs over names once discovered. Use `limit` and `offset` for pagination.

## Upsert and Lookup Notes

- For hierarchy records, list first and match by name before creating to avoid duplicates.
- For test cases, create new cases for a new QA handoff unless the user asks to modify existing cases.
- For personal secrets, `POST /secret-collections/:id/secrets` inserts or replaces by key.
- To update, read, or delete a personal secret by key, list collection values first and use the returned `secretId`.
- For global secrets, `POST /global-secrets` creates or replaces by key.
