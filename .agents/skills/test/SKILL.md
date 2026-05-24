---
name: test
description: Manage Persian QA work in the internal Testcase app. Use for `/test`, `/bugfix`, `/creds`, QA handoff, release test cases, created placeholder cases needing detail, failed test fixes, bug reports, evidence/media, test status, and questions like "how are tests doing". Default output is the Testcase API; markdown files are only for explicit md/file requests.
---

# Test

Use this skill to turn implemented changes into practical Persian test tasks in the internal Testcase app.

Arguments from direct invocation: $ARGUMENTS

## Default Output

Default to the Testcase app API. Do not write markdown test-case files unless the user explicitly asks for `md`, `markdown`, `written file`, `.testcases` file output, or similar.

Read `references/testcase-api.md` from this skill before API work. The skill must work standalone on any machine; do not rely on a local checkout of the Testcase app source.

Use UTF-8 JSON for Persian API payloads. If the shell/client might corrupt Persian text, write the Persian payload to a UTF-8 file first and have Node read that file before inserting through the API.

## Command Map

Treat command-like prompts as explicit routing hints:

- `/test`: create new test cases from code changes, or detail existing `created` placeholder cases when the user mentions created/manual/empty cases needing detail.
- `/test <range>`: inspect commits/diffs in the supplied range plus uncommitted changes before creating or detailing cases.
- `/bugfix`: fetch failed Testcase cases for this project, review comments/media/images, discuss failures, ask permission, fix approved issues, then move fixed cases back to `pending`.
- `/creds`: manage Testcase key-value secrets, API keys, app credentials, and lookup notes.
- status questions: fetch saved app/project/release tests and summarize `created`, `pending`, `passed`, and `failed`.

If a prompt names multiple commands, handle them in a safe order: read/update `.testcases/info.md`, report or discuss first when requested, then mutate tests/code only after the relevant workflow permits it.

## Project Files

At the project root, create `.testcases` if it does not exist.

Maintain:

- `.testcases/info.md`: Testcase app URL, API key/token, UI login, default app/project/release, and secrets lookup notes.
- `.testcases/read-before-write-testcase.md`: durable user preferences for this project's test process.

Always read `.testcases/info.md` before using the Testcase app. Update it whenever related durable data is discovered or changed: app URL, personal API key, shared token, API username, UI credentials, matching app/project/release IDs and names, active/default release, secret collection IDs/names, important secret keys, or notes needed to find this project's tests again.

This is an internal tool. If the user asks to store Testcase app credentials, API keys, or QA secret lookup details in `.testcases/info.md`, do it. Do not refuse solely because the values are credentials. Avoid printing secrets in the final response unless the user explicitly asks to retrieve or show them.

Before creating or changing tests, read `.testcases/read-before-write-testcase.md` if it exists. If the user changes the test-case process, preferred format, granularity, release naming, target app/project/release mapping, QA policy, credential handling, or similar future behavior, summarize the durable preference in that file.

## `.testcases/info.md`

Resolution order for connection values:

1. Existing `.testcases/info.md`.
2. Environment variables: `TESTCASE_APP_URL`, `TESTCASE_API_KEY`, `TESTCASE_API_TOKEN`, `TESTCASE_API_USERNAME`.
3. Values already provided in the current conversation.
4. Ask the user.

Use simple `key: value` lines. Include all known fields and leave unknown fields blank:

```text
testcase_app_url:
api_key:
api_key_name:
api_token:
api_username:
ui_username:
ui_password:
default_app_id:
default_app_name:
default_project_id:
default_project_name:
default_release_id:
default_release_name:
personal_secret_collection_id:
personal_secret_collection_name:
global_secret_keys:
personal_secret_keys:
secret_lookup_notes:
testcase_process_notes:
notes:
```

Prefer personal API keys (`tcsk_...`) when available. Shared `api_token` is also valid. When an API call confirms IDs or names, write them back to blank or stale `default_*` fields. Future status questions should be able to find the relevant tests automatically from this file.

Personal API keys are created from the Testcase app Settings page and are shown only once. If the user provides a newly created key, save it in `api_key` and record a useful `api_key_name`.

## Modes

- Conversation mode: use the conversation, recent edits, and relevant uncommitted changes.
- Time range mode: inspect commits in the supplied range plus current uncommitted changes. Do not rely on commit messages alone.
- Written file mode: only when explicitly requested, write a `.md` file under `.testcases`.
- `/creds` mode: manage key-value secrets through the Testcase app secrets API.
- Status/report mode: fetch and summarize test status from the Testcase app.
- Bug/evidence mode: create bug reports or attach media/evidence when requested.
- Created-detail mode: fill in `created` placeholder test cases that have only a title or weak detail.
- `/bugfix` mode: fetch failed test cases, review evidence/comments/media, discuss with the user, ask permission, fix code, and move fixed cases back to `pending`.

## Test Creation Workflow

1. Find the project root. Prefer `git rev-parse --show-toplevel`; otherwise use the current workspace root.
2. Create `.testcases` if needed.
3. Read `.testcases/info.md` and `.testcases/read-before-write-testcase.md` if present.
4. Resolve `testcase_app_url` and a bearer credential (`api_key` preferred, then `api_token`). Validate with `GET /api/v1`.
5. Build the change set:
   - Conversation mode: use context plus `git diff --stat`, `git diff`, and touched files when available.
   - Time range mode: use `git log --since ... --until ... --reverse --stat --patch`, then inspect current uncommitted changes.
   - Read latest affected files. Test the latest behavior only.
   - Prefer exact routes from routing files, page components, links, controllers, tests, or app conventions.
6. Identify user-visible QA coverage: pages, forms, filters, tables, actions, permissions, validations, errors, empty/loading states, navigation, API-backed behavior, media/evidence flows, and visible copy.
7. Discover or create hierarchy:
   - Prefer IDs in `info.md`.
   - Otherwise list apps/projects/releases and match by saved names, project root name, or user instruction.
   - Create missing app/project/release records when the request implies a QA handoff and credentials allow it.
   - Use a Jalali date release name when no release name is supplied.
   - Update `info.md` with confirmed app/project/release IDs and names.
8. Create test cases with `POST /api/v1/test-cases`.
   - `title`: short Persian title.
   - `description`: concise Persian feature and expected behavior.
   - `path`: exact route when known.
   - `howToTest`: concise Persian end-user steps.
   - `status`: omit for default `created`, unless the user or project preference says to set `pending`.

If the API is unreachable or auth fails, report the exact blocker and ask for corrected connection details. Do not silently fall back to markdown unless file output was explicitly requested.

When API permissions fail, explain the likely role requirement and ask for a personal API key from a user with the needed role instead of abandoning the workflow.

## Created-Detail Workflow

Use this when `/test` or the user mentions `created`, empty test cases, placeholders, cases needing detail, or turning manually added titles into ready-to-test cases.

1. Read `.testcases/info.md`, resolve auth, and validate the API.
2. Resolve the target hierarchy from saved IDs or discovery.
3. Fetch created cases, usually:
   - `GET /api/v1/test-cases?releaseId=<releaseId>&status=created&sort=oldest&limit=100`
   - or use `projectId`/`appId` when no release is selected.
4. If no created cases are found, tell the user clearly and offer the current status summary.
5. Identify cases that are placeholders: title exists but `description`, `path`, or `howToTest` is missing, vague, or too short.
6. If there are many placeholders or several unrelated areas, group them by route/feature and proceed in manageable batches.
7. For each placeholder, review the codebase, routes, recent commits/diffs, and related UI/API behavior to infer the intended user-visible flow.
8. Update each case with `PATCH /api/v1/test-cases/:id`:
   - keep the original title unless it is unclear.
   - fill Persian `description`, exact `path` when known, and Persian `howToTest`.
   - change status to `pending` when the case is detailed enough for QA, unless the user prefers to keep detailed drafts as `created`.
   - if the API returns 403, ask for an admin/developer API key; tester keys cannot edit test definitions.
9. Report which created cases were detailed, which remain ambiguous, and any questions for the user.

## Status/Report Workflow

Use this when the user asks for test progress, QA state, "how are the tests doing?", or similar.

1. Read `.testcases/info.md`, resolve auth, and validate the API.
2. Prefer saved IDs:
   - `default_release_id`: list test cases for that release.
   - `default_project_id`: list releases and test cases for that project.
   - `default_app_id`: list projects/releases/test cases under that app.
   - Otherwise discover by saved names or project root name, then update `info.md`.
3. Use API rollups from app/project/release responses when available.
4. Summarize `created`, `pending`, `passed`, and `failed`, recent updates, media/evidence counts when relevant, and any failed or not-yet-run cases needing attention.
5. If a newer relevant release is clearly the active target, update `default_release_id` and `default_release_name`.
6. Report in English. Do not expose stored credentials.

## `/creds` Workflow

Use `/creds` for storing, modifying, deleting, listing, retrieving, or locating key-value credentials in the Testcase app.

1. Read or create `.testcases/info.md`; resolve app URL and auth.
2. Default to personal `secret-collections`. Use `global-secrets` only if the user says global/shared.
3. For personal secrets, list or create the named collection. If no collection name is supplied, use the project name.
4. Insert or replace values with `POST /api/v1/secret-collections/:id/secrets`.
5. To read, update, or delete by key, list values first to find `secretId`.
6. Update `.testcases/info.md` with collection ID/name, useful key names, and lookup notes so future agents can find the values.
7. For retrieval, return only requested values. Avoid dumping unrelated secrets.

## Bug And Evidence Workflow

- Tester users can create test cases only as bug reports. Send `reportType: "bug"`; bug reports are saved as failed test cases.
- For bug reports, use Persian `title`, `path`, `description` for context, and structured `steps`, `expected`, and `actual` when available.
- Upload evidence with multipart requests using field names `images`, `media`, `files`, or `evidence`.
- Attach evidence to an existing case with `POST /api/v1/test-cases/:id/attachments`.
- Attach evidence to a pass/fail decision with multipart `POST /api/v1/test-cases/:id/status`.

## `/bugfix` Workflow

Use this when the user invokes `/bugfix` or asks to fix failed tests from the Testcase app.

1. Read `.testcases/info.md`, resolve auth, validate the API, and resolve the target app/project/release.
2. Fetch failed cases, usually:
   - `GET /api/v1/test-cases?releaseId=<releaseId>&status=failed&sort=updated&limit=100`
   - or use `projectId`/`appId` when no release is selected.
3. If no failed cases are found, tell the user there is nothing to fix for the selected scope and summarize the current status.
4. If there are many failed cases, group them by likely feature/route and ask the user which group to tackle first unless the user asked to fix all.
5. For each failed case that may be in scope, fetch detail with `GET /api/v1/test-cases/:id` and evidence with `GET /api/v1/test-cases/:id/attachments`.
6. Review all relevant context before proposing fixes:
   - title, description, path, howToTest, status, comments, status-change comments, direct attachments, comment attachments, media URLs/images, and timestamps.
   - open or download accessible media/image URLs when they are relevant to understanding the failure.
   - related source files, routes, tests, logs, recent commits, and current diffs.
7. Talk with the user in English about the failed cases. Summarize the likely issues, affected routes/features, and proposed fix plan.
8. Ask for explicit permission before editing code. Do not start code changes for `/bugfix` until the user agrees.
9. After permission, fix the issues in the codebase using normal engineering workflow and verify with relevant tests or checks when possible.
10. For each fixed failed case, log the fix in the app and move it back to `pending` with:
   - `POST /api/v1/test-cases/:id/status`
   - body: `{ "status": "pending", "comment": "..." }`
   - include a concise English or Persian comment that says what was fixed and that the case is ready for QA retry.
11. If a failure cannot be fixed, leave it `failed`, add a status/comment only if useful, and explain the blocker.
12. Final response: summarize fixed cases, cases moved to `pending`, cases still failed, code files changed, and verification results.

## Writing Rules

- Write test-case content in Persian.
- Talk to the user in English.
- Avoid English words in test cases unless necessary for route names, product terms, roles, or technical UI labels.
- Test as an end user through the application UI whenever possible.
- Keep each test case manageable: not too broad, not too tiny.
- Include role, permission, validation, negative, media, and edge-case tests when implied.
- Avoid implementation details, filenames, commits, or internal APIs unless testers need them.
- If the exact URL cannot be proven, use the closest known parent route and make the uncertainty clear.

## Legacy Markdown Output Contract

Use only when explicitly requested.

Write a UTF-8 plain-text `.md` file under `.testcases` at the project root. The content must be Persian, concise, direct, and professional. Do not use Markdown headings, bullets, checkboxes, tables, or code fences inside the generated task file.

Separate each task with a line containing only:

---

Each task must use exactly this shape:

```text
URL: relative-url
Feature: یک یا دو خط درباره قابلیت و تغییر اصلی و رفتار مورد انتظار
Test Case: مراحل کوتاه و روشن برای تست از دید کاربر نهایی
```

Save with a Jalali date filename:

- No time range: `.testcases/{project name} - {jalali today}.md`
- Time range: `.testcases/{project name} - {jalali start date} till {jalali end date}.md`

## Final Response

Tell the user in English what changed:

- API mode: target app/project/release and number of test cases created/updated.
- Created-detail mode: number of created placeholders detailed, moved to `pending`, or left ambiguous.
- `/bugfix` mode: failed cases reviewed, user-approved fixes applied, cases moved to `pending`, remaining blockers, files changed, and verification.
- Status mode: concise status summary.
- Written file mode: created file path and coverage summary.
- `/creds`: credential action and lookup location, without exposing secret values unless requested.
