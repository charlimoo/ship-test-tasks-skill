---
name: test
description: Create, manage, and report on Persian QA/test-team tasks from recent code changes. Use when the user invokes `/test`, asks for test tasks, QA handoff, tester tasks, release test cases, test status/progress, "how are tests doing", or asks to turn conversation changes, uncommitted changes, or a git time range into test cases. Default output is the Testcase app API; use `.testcases` markdown files only when the user explicitly asks for md, markdown, written files, or file output. Also use when the user invokes `/creds` to store, modify, delete, or retrieve key-value credentials through the Testcase app.
---

# Test

Use this skill to turn implemented changes into practical Persian test tasks for the QA/test team.

Arguments from direct invocation: $ARGUMENTS

## Default Output: Testcase App

Default to creating or updating test cases through the Testcase app API. Do not write markdown test-case files unless the user explicitly asks for `md`, `markdown`, `written file`, `.testcases` file output, or similar.

When using the app API, read `references/testcase-api.md` from this skill for endpoint details.

This skill must work standalone on any machine. Do not rely on a local checkout of the Testcase app source. Use only the project-local `.testcases/info.md`, environment variables, user-provided values, and this skill's bundled API reference.

## Project Files

At the project root, create `.testcases` if it does not exist.

Maintain these project-local files:

- `.testcases/info.md`: connection and routing info for the Testcase app.
- `.testcases/read-before-write-testcase.md`: user preferences for test-case process and wording.

Before creating or changing tests, always read `.testcases/read-before-write-testcase.md` if it exists and apply the stored preferences.

Always read `.testcases/info.md` before using the Testcase app, and update it whenever related durable data is discovered or changed. Examples: Testcase app URL, API username, matching app/project/release IDs, app/project/release names, current/default release, or notes needed to find this project's tests again. Keep existing user-provided values unless they are proven stale.

If the user asks to change the test-case process, preferred format, style, granularity, release naming, target app/project/release mapping, QA policy, or similar future behavior, summarize the durable preference in `.testcases/read-before-write-testcase.md`. Keep it concise and project-specific. Do not duplicate transient one-off instructions.

## `.testcases/info.md`

Before using the Testcase app for a project, ensure `.testcases/info.md` exists and has enough information to authenticate and choose the target hierarchy. Ask the user for missing required values.

Resolution order for connection values:

1. Existing `.testcases/info.md`.
2. Environment variables such as `TESTCASE_APP_URL`, `TESTCASE_API_TOKEN`, and `TESTCASE_API_USERNAME`.
3. Values already provided in the current conversation.
4. Ask the user.

Store useful project-local fields in simple `key: value` lines, for example:

```text
testcase_app_url: http://localhost:3001
api_token: <bearer token>
api_username: <api user, if relevant>
ui_username: <browser login username>
ui_password: <browser login password>
default_app_id:
default_app_name:
default_project_id:
default_project_name:
default_release_id:
default_release_name:
notes:
```

Prefer `api_token` for automation. Keep username and password only when the user provides them or asks to store them. If the user refuses to store sensitive values, leave placeholders in `info.md` and use environment variables or values from the current conversation for that run.

When creating `info.md`, include all known fields and leave unknown fields blank. If credentials come only from environment variables, do not copy secret values into `info.md` unless the user asks to persist them.

When discovering app/project/release records through the API, write any newly confirmed IDs and names back to blank or stale `default_*` fields. This lets future requests like "how are the tests doing?" automatically find the relevant tests in the app.

## Modes

- Conversation mode: if no time range is supplied, use the current conversation, recent edits, and relevant uncommitted changes to infer what changed.
- Time range mode: if a date/time range is supplied, inspect all git commits in that range plus current uncommitted changes. Do not rely on commit messages alone.
- Written file mode: only when explicitly requested, write a `.md` file under `.testcases` using the legacy output contract below.
- `/creds` mode: manage key-value secrets through the Testcase app secrets API.
- Status/report mode: when the user asks how tests are doing, use saved `info.md` mappings to fetch and summarize test status from the Testcase app.

## Test Creation Workflow

1. Find the project root. Prefer `git rev-parse --show-toplevel`; if the folder is not a git repository, use the current workspace root.
2. Determine the project name from the root folder name.
3. Create `.testcases` if needed.
4. Read `.testcases/read-before-write-testcase.md` if it exists.
5. Ensure `.testcases/info.md` has the Testcase app URL and authentication needed for API work. Ask for missing required info.
   - API mode requires `testcase_app_url` and an API token from `info.md`, environment, or the current conversation.
   - Validate the connection with `GET /api/v1` before creating records.
6. Build the change set:
   - In conversation mode, use the conversation context plus `git diff --stat`, `git diff`, and touched files when available.
   - In time range mode, inspect commits with `git log --since ... --until ... --reverse --stat --patch`, then inspect current uncommitted changes with `git diff --stat` and `git diff`.
   - Read the latest version of affected files before writing tasks. If the same feature changed multiple times, test the latest behavior only.
   - Prefer exact relative URLs from routing files, page components, links, controllers, tests, or app conventions. Use paths like `/dashboard/orders` for the API `path` field.
7. Identify user-visible behavior that needs QA coverage:
   - New or changed pages, forms, filters, tables, actions, permissions, validations, errors, empty states, loading states, navigation, API-backed behavior, and visible copy.
   - Do not create tasks for invisible refactors unless they affect a user-visible flow or integration risk.
8. Discover or create the target Testcase hierarchy:
   - Use IDs from `info.md` when present.
   - Otherwise list apps, projects, and releases through the API and match by names from `info.md`, project name, or user instruction.
   - Create missing app/project/release records when the user request implies creating a QA handoff and credentials allow it.
   - Use a Jalali date release name when no release name is supplied. For a time range, use `{jalali start date} till {jalali end date}`.
   - After discovering or creating IDs, update blank `default_*` fields in `.testcases/info.md` so future runs do not need rediscovery.
9. Create test cases with `POST /api/v1/test-cases`.
   - `title`: short Persian title.
   - `description`: concise Persian feature/expected-behavior summary.
   - `path`: exact route when known, otherwise closest known parent route.
   - `howToTest`: concise Persian end-user steps.
   - `status`: `pending`.

If the API is unreachable or authentication fails, report the exact blocker and ask for corrected connection details. Do not silently fall back to markdown unless the user explicitly requested file output.

## Status/Report Workflow

Use this when the user asks for test progress, test status, QA state, "how are the tests doing?", or similar.

1. Find the project root and read `.testcases/info.md`.
2. Resolve `testcase_app_url` and API token using the same resolution order as test creation.
3. Read `references/testcase-api.md` and validate the API with `GET /api/v1`.
4. Prefer saved IDs from `info.md`:
   - If `default_release_id` exists, fetch test cases for that release.
   - Else if `default_project_id` exists, fetch releases and test cases for that project.
   - Else if `default_app_id` exists, fetch projects/releases/test cases under that app.
   - Else discover by saved names or project root name, then update `info.md` with confirmed IDs and names.
5. Summarize counts by `pending`, `passed`, and `failed`, recent updates if available, and any failed or pending cases that need attention.
6. If a newer relevant release is discovered for the saved project, update `default_release_id` and `default_release_name` when it is clearly the active/default target.
7. Report to the user in English. Do not expose stored credentials.

## `/creds` Workflow

Use `/creds` for storing, modifying, deleting, listing, or retrieving key-value credentials in the Testcase app.

1. Read or create `.testcases/info.md` and get `testcase_app_url` plus API token.
2. Read `references/testcase-api.md`.
3. Determine whether the user wants personal secret collections or global secrets:
   - Default to personal `secret-collections`.
   - Use `global-secrets` only if the user says global/shared or explicitly asks for it.
4. For personal secrets, list or create the named collection. If no collection name is supplied, use the project name.
5. Insert or replace values with the collection secret `POST` endpoint. To read, update, or delete by key, list values first to find the matching `secretId`.
6. For global secrets, list, create/replace, read, update, or delete global values. Remember only admins can create, update, or delete global secrets.
7. For retrieval, return only the requested key/value(s). Avoid dumping unrelated secrets.

## Writing Rules

- Write test-case content in Persian.
- Talk to the user in English.
- Avoid English words in test cases unless necessary for route names, product terms, roles, or technical labels visible in the UI.
- Test as an end user through the application UI whenever possible.
- Keep each test case manageable: not too broad, not too tiny.
- Include permission/role, validation, negative, and edge-case tests when the change implies them.
- Avoid implementation details, filenames, commits, or internal APIs unless testers need them to perform the test.
- If the exact URL cannot be proven, use the closest known parent route and make the uncertainty clear in the description.

## Legacy Markdown Output Contract

Use this section only when the user explicitly wants md, markdown, written file, `.testcases` file output, or similar.

Write a UTF-8 plain-text `.md` file under `.testcases` at the project root. The file content must be Persian, concise, direct, and professional. Do not use Markdown headings, bullets, checkboxes, tables, or code fences inside the generated task file.

Separate each task with a line containing only:

---

Each task must use exactly this shape:

```text
URL: relative-url
Feature: یک یا دو خط درباره قابلیت و تغییر اصلی و رفتار مورد انتظار
Test Case: مراحل کوتاه و روشن برای تست از دید کاربر نهایی
```

Save the task file with a Jalali date filename:

- No time range: `.testcases/{project name} - {jalali today}.md`
- Time range: `.testcases/{project name} - {jalali start date} till {jalali end date}.md`

Before finishing written file mode, confirm:

- `.testcases` exists at the project root.
- The output filename uses Jalali dates.
- Every task has `URL`, `Feature`, and `Test Case`.
- Tasks are separated by `---`.
- The written task text is Persian and concise.

## Final Response

Tell the user in English what was created or changed:

- For API mode, summarize the target app/project/release and number of test cases created or updated.
- For written file mode, provide the created file path and summarize coverage.
- For `/creds`, summarize the credential action without exposing secret values unless retrieval was explicitly requested.
