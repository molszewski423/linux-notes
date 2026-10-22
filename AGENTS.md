# linux-notes: agent instructions

Read the global agent instructions first: `~/.claude/CLAUDE.md` (Claude Code) or `~/.codex/AGENTS.md` (Codex),
both linked to `context/global/AGENTS.md` in the private claude-memory repo. They cover conventions, approvals and
the session protocol (branches `<agent>/<topic>`, signed commits with an `Agent:` trailer, handoff notes). This file
adds what is specific to this repository; where it is stricter, it wins.

## What this is

A running log of distributions used and problems solved, one directory per distribution plus `hardware/`.

## Rules for this repository

- **Public on Gitea.** No internal addresses or credentials. Add a note when something took more than ten minutes.
