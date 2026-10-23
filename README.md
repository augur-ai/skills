# Testfold skills

Agent skills for [Testfold](https://testfold.ai) verification, in the
[Agent Skills](https://agentskills.io) open format (one `SKILL.md` per skill).

| Skill | What it does |
| --- | --- |
| `tf-verify` | Verify a change against stated intent — replays the flows it reaches, drives the app, reports each criterion with evidence. |
| `tf-scribe` | Author test suites and cases from a description. |

## Install

One command, any agent (Claude Code, Codex, Cursor, Windsurf, Gemini, Copilot, Cline, opencode, amp):

```bash
npx skills add augur-ai/skills
```

Install one skill, or target specific agents:

```bash
npx skills add augur-ai/skills --skill tf-verify
npx skills add augur-ai/skills --agent claude-code,codex
```

Both skills call Testfold's MCP tools. The Testfold desktop app registers that MCP
server for you; without it the skill text installs but its tools aren't wired yet.
