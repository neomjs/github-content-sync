---
id: 217
title: AGENTS.md leaves an agent to discover how to run and reach the app
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-26T18:05:06Z'
updatedAt: '2026-10-26T18:27:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/217'
author: neo-opus-grace
commentsCount: 0
parentIssue: 20114
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-26T18:27:40Z'
---
# AGENTS.md leaves an agent to discover how to run and reach the app

## Context

Moved from neomjs/create-app#51 (same author). That ticket named the wrong owner: create-app does not hold an `AGENTS.md` template. Its `tasks/createAgentLayer.mjs` runs this package's generator (`neo-agent-skills-agents-md --repo neo-app-workspace --audience app --out AGENTS.md`), so the text lives here.

The cloud cold walk for neomjs/neo#20114 (2026-10-26) reached the running app through the Neural Link. It recorded "setup" steps that the generated `AGENTS.md` should have made unnecessary. It took 7 min 22 s from `npx` to the first tree, of which only about 2.6 min was machine time; the rest was reading source to find answers.

## The Problem

The app variant says `npm run server-start` "serves the workspace and opens it", and that the app connects "once the Neural Link MCP server is configured for your harness". An agent then has to discover the following by reading `node_modules`:
- the port (8080, or `$PORT`) and the app URL (`/apps/<name>/index.html`);
- that `npm run server-start -- --no-open` runs it without a desktop browser;
- that the Neural Link is **already configured** (`.mcp.json`, `.codex/config.toml`), takes effect only when the harness starts in the workspace folder, and needs one approval;
- that if the page loaded before the Neural Link started, the app stops dialing after five tries (backoff 1 + 2 + 4 + 8 s), so tools answer "No active App Worker sessions found" until the page is focused or reloaded. This is deliberate engine behavior: neo `src/ai/Client.mjs` `redial`, "never on a timer";
- that headless verification needs `npx playwright install chromium` first (no Playwright ships in the workspace);
- that `neo.mjs` declares Node ≥ 24.

## The Architectural Reality

- `agents-md/sections/0901-app-orientation.md` (`§orientation`) and `0902-app-skills.md` (`§skills`) are the app variant's sections (`repos: neo-app-workspace`, `audiences: app`).
- `scripts/test-generate-agents-md.mjs` asserts the app variant on its real output (`generate({audience: 'app', repos: ['neo-app-workspace']})`).
- create-app pins this package at `^0.1.61` (`AGENT_SKILLS_RANGE`), so a release reaches every fresh scaffold with no create-app change.

## The Fix

Extend the two app sections with these facts: run headless, the URL, the Neural Link that is already configured plus the approval, the reload rule, the browser install, and Node. Keep the file short. Extend the generator test to assert each line, and bump the version.

## Acceptance Criteria

- [ ] AC-1: the generated app `AGENTS.md` states the port and URL, `-- --no-open`, the pre-configured Neural Link (start the harness in the workspace, approve once), the reload rule for "No active App Worker sessions found", the Playwright browser install, and Node ≥ 24; `scripts/test-generate-agents-md.mjs` asserts each line.
- [ ] AC-2 (post-publish): the next cold walk records none of these as a setup step.

## Out of Scope

- Engine or Neural Link behavior changes (the neomjs/neo and Brain leaves of #20114).
- The app skills' own getting-connected section (#215).

## Related

Parent: neomjs/neo#20114 · predecessor neomjs/create-app#51 (closed as moved) · #215

Sweeps (2026-10-26T18:04Z): live latest-open, the 20 newest `neomjs/neo-agent-skills` issues by creation, no equivalent; own-assignment, #215 only (the skill files, a different surface); A2A, latest 30 messages, no competing claim (my own lane-claim 17:49Z); Memory Core, a clear miss. Implementation is already on branch `grace/51-app-agents-md-run-reach-recover`: 8 new test arms fail on `dev` and pass at its head.

Origin Session ID: c39fbf61-9160-44c8-ba56-9545b1b50774
Retrieval Hint: "neo-app-workspace AGENTS.md app_orientation app_skills server-start no-open port Neural Link already configured reload No active App Worker sessions playwright install node 24"

## Timeline

- 2026-10-26T18:05:07Z @neo-opus-grace added the `documentation` label
- 2026-10-26T18:05:07Z @neo-opus-grace added the `enhancement` label
- 2026-10-26T18:05:08Z @neo-opus-grace added the `ai` label
- 2026-10-26T18:05:28Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-26T18:06:38Z @neo-opus-grace cross-referenced by #51
- 2026-10-26T18:07:52Z @neo-opus-grace cross-referenced by PR #218
- 2026-10-26T18:27:40Z @tobiu referenced in commit `1ab34a7` - "feat(agents-md): the app workspace's AGENTS.md says how to run, reach and recover the app (#217) (#218)

The generated AGENTS.md said server-start "serves the workspace and opens
it" and that the app connects "once the Neural Link MCP server is
configured". The cloud cold walk of neomjs/neo#20114 then read node_modules
to learn the rest. The app variant now states the port and URL, serving
without a desktop browser, that the Neural Link is already configured and
needs one approval, the reload rule for "No active App Worker sessions
found" (five tries, about 15 s, never on a timer), the headless browser
install, and Node 24. The generator test asserts each line; a gate-number
citation in its comments loses its ticket-shaped "#4". 0.1.63."
- 2026-10-26T18:27:40Z @tobiu closed this issue

