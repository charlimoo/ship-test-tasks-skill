---
name: test
description: Create Persian QA/test-team task files from recent code changes. Use when the user invokes `/test`, asks for test tasks, QA handoff, tester tasks, release test cases, or asks to turn conversation changes, uncommitted changes, or a git time range into concise test cases stored under `.testcases`.
---

# Test

Use this skill to turn implemented changes into practical test tasks for the QA/test team.

Arguments from direct invocation: $ARGUMENTS

## Output Contract

Write a UTF-8 plain-text `.md` file under `.testcases` at the project root. The file content must be Persian, concise, direct, and professional. Do not use Markdown headings, bullets, checkboxes, tables, or code fences inside the generated task file.

Separate each task with a line containing only:

---

Each task must use exactly this shape:

URL: relative-url
Feature: یک یا دو خط درباره قابلیت و تغییر اصلی و رفتار مورد انتظار
Test Case: مراحل کوتاه و روشن برای تست از دید کاربر نهایی

## Modes

- Conversation mode: if no time range is supplied, use the current conversation, recent edits, and relevant uncommitted changes to infer what changed.
- Time range mode: if a date/time range is supplied, inspect all git commits in that range plus current uncommitted changes. Do not rely on commit messages alone.

## Workflow

1. Find the project root. Prefer `git rev-parse --show-toplevel`; if the folder is not a git repository, use the current workspace root.
2. Determine the project name from the root folder name.
3. Create `.testcases` in the project root if it does not exist.
4. Build the change set:
   - In conversation mode, use the conversation context plus `git diff --stat`, `git diff`, and touched files when available.
   - In time range mode, inspect commits with `git log --since ... --until ... --reverse --stat --patch`, then inspect current uncommitted changes with `git diff --stat` and `git diff`.
   - Read the latest version of affected files before writing tasks. If the same feature changed multiple times, test the latest behavior only.
   - Prefer exact relative URLs from routing files, page components, links, controllers, tests, or app conventions. Use URLs without a leading slash when possible, such as `dashboard/orders`.
5. Identify user-visible behavior that needs QA coverage:
   - New or changed pages, forms, filters, tables, actions, permissions, validations, errors, empty states, loading states, navigation, API-backed behavior, and visible copy.
   - Do not create tasks for invisible refactors unless they affect a user-visible flow or integration risk.
6. Save the task file with a Jalali date filename:
   - No time range: `.testcases/{project name} - {jalali today}.md`
   - Time range: `.testcases/{project name} - {jalali start date} till {jalali end date}.md`

## Writing Rules

- Write task content in Persian.
- Avoid English words unless necessary for route names, product terms, roles, or technical labels visible in the UI.
- Test as an end user through the application UI whenever possible.
- Keep each test case manageable: not too broad, not too tiny.
- Include permission/role, validation, negative, and edge-case tests when the change implies them.
- Avoid implementation details, filenames, commits, or internal APIs unless testers need them to perform the test.
- If the exact URL cannot be proven, use the closest known parent route and make the uncertainty clear in the feature text.

## Final Check

Before finishing, confirm:

- `.testcases` exists at the project root.
- The output filename uses Jalali dates.
- Every task has `URL`, `Feature`, and `Test Case`.
- Tasks are separated by `---`.
- The written task text is Persian and concise.

Then tell the user the created file path and summarize the test coverage in English.
