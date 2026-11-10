---
id: 1540
title: CONTRIBUTING setup does not work as described on Windows x64
state: OPEN
labels: []
assignees: []
createdAt: '2026-11-09T19:42:53Z'
updatedAt: '2026-11-09T19:42:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1540'
author: anasdahani220
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# CONTRIBUTING setup does not work as described on Windows x64

## Context
Following CONTRIBUTING.md ("Set up the Brain from a clone") on Windows x64. The guide says to report any command that does not behave as described, so here are the four I hit while working on #1495 (PR #1517).

Environment: Windows 11 x64, Node.js 24.22.2, npm 11, PowerShell.

## What happened

1. **`npm install` fails building `better-sqlite3`**
   `gyp ERR! find VS - missing any VC++ toolset` / `Could not find any Visual Studio installation to use`.
   Visual Studio 2026 Community is installed but without the "Desktop development with C++" workload. Workaround: `npm install --ignore-scripts`.

2. **`npm run test-unit` runs no tests: the Chroma setup project fails**
   `Error: Unsupported Windows architecture: x64. Only ARM64 is supported.` (from `node_modules/chromadb/dist/cli.mjs`).
   1 failed, 103 did not run. Workaround: `--no-deps`, which only works for specs that do not need Chroma.

3. **Unit specs fail on Windows independent of any change**
   With `--no-deps`, specs under `test/playwright/unit/ai/mcp` fail with `EPERM: operation not permitted, symlink`, `spawn /bin/sh ENOENT`, `EPERM ... fsync`, and path assertions (`D:\custom\corpus` vs `/custom/corpus`, `D:\D:\projects\...`). They reproduce identically on a clean `dev`.

4. **`.agents/skills` is not materialized after `--ignore-scripts`**
   `agent-preflight` points to `.agents/skills/pull-request/references/pull-request-workflow.md`, which does not exist then. The file is reachable at `node_modules/neo-agent-skills/.agents/skills/...`.

## Suggestion
A short "Windows" note in CONTRIBUTING.md: Developer Mode for symlinks, the VC++ workload for `better-sqlite3`, the Chroma x64 limitation (WSL2 as the supported path?), and where the PR template lives when skills are not linked. Happy to open the docs PR if that direction works for you.

