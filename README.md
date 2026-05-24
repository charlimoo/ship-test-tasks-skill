# Test Skill

Portable Agent Skill for creating and managing Persian QA work in the internal Testcase app.

## Layout

- `.agents/skills/test/SKILL.md`: canonical skill entrypoint for agent workspaces.
- `.agents/skills/test/references/testcase-api.md`: standalone Testcase API reference for agents.
- `.agents/skills/test/agents/openai.yaml`: OpenAI-facing display metadata and default command prompt.

The repository intentionally keeps only the `.agents` skill layout.

## Usage

```text
Use the test skill to create Persian QA test tasks from the latest changes.
```

```text
/test from 1404-03-01 to 1404-03-02
```

By default, the skill writes test cases into the Testcase app through its API. It only writes `.testcases/*.md` files when the user explicitly asks for markdown or file output.
