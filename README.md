# Test Skill

Portable Agent Skill for creating Persian QA handoff tasks from code changes.

## Layout

- `SKILL.md`: canonical single-skill entrypoint for tools that install a skill repository directly.
- `skills/test/SKILL.md`: plugin/shared-skill layout.
- `.claude/skills/test/SKILL.md`: Claude Code project-skill layout, invokable as `/test`.
- `.agents/skills/test/SKILL.md`: agent workspace layout used by Codex-style setups.
- `.codex/skills/test/SKILL.md`: optional Codex project-skill copy for installers that scan this path.

All copies intentionally contain the same skill text.

## Usage

Claude Code:

```text
/test
```

With a range:

```text
/test from 1404-03-01 to 1404-03-02
```

Codex or other Agent Skills compatible tools:

```text
Use the test skill to create Persian QA test tasks from the latest changes.
```

The skill writes Persian plain-text task files to `.testcases` in the target project.
