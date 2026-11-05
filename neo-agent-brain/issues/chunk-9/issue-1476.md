---
id: 1476
title: 'Both Brain lockfiles carry fixable advisories, one of them critical'
state: OPEN
labels:
  - bug
  - dependencies
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-05T10:36:15Z'
updatedAt: '2026-11-05T10:36:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1476'
author: neo-opus-vega
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
# Both Brain lockfiles carry fixable advisories, one of them critical

## Context

Found during the #300 premise read for #1438 (2026-11-05). `npm audit` on Brain `dev` 65182b08 reports advisories that a compatible lockfile refresh closes. They are not entangled with the `sharp` decision #300 owns, so they shouldn't wait for it (#300's AC-3 already says this for `qs`).

## The Problem

`npm audit --json`, run 2026-11-05 10:3xZ:

| Lockfile | total | critical | high | moderate | `fixAvailable: true` |
|---|---:|---:|---:|---:|---|
| root `package-lock.json` | 9 | 0 | 5 | 4 | `body-parser`, `fast-uri` (high); `hono`, `@hono/node-server`, `ip-address`, `qs` (moderate) |
| `cloud/package-lock.json` | 13 | **1** | 8 | 4 | `proxy-addr` (**critical**); `@modelcontextprotocol/sdk`, `zod`, `body-parser`, `fast-uri` (high); `hono`, `@hono/node-server`, `ip-address`, `qs` (moderate) |

The rest (`sharp`, `@huggingface/transformers`, `@chroma-core/default-embed`, plus the cloud package's own `neo-agent-brain` entry that inherits them) report `fixAvailable: false`. That is #300's chain.

These packages sit on the MCP servers' HTTP path (`@modelcontextprotocol/sdk`, `hono`, `body-parser`, `proxy-addr`), so they are runtime, not tooling.

## The Architectural Reality

- Two independent manifests: the root (Host profile) and `cloud/package.json` (`@neomjs/neo-agent-brain-cloud`, with its own lockfile, per #213). A fix in one is not inherited by the other.
- The root `overrides` hold only `gray-matter → js-yaml` (`package.json:127-131`); no override pins any of the packages above.
- `@modelcontextprotocol/sdk` is the MCP server SDK. Its fix may be a minor bump, so the MCP servers' unit and e2e suites are the gate, not just `npm audit`.

## The Fix

Run a compatible refresh (`npm audit fix`, never `--force`) in both the root and `cloud/`, and commit both lockfiles. If any listed advisory only resolves through a major bump, leave it, name it in the PR, and file it separately. Don't fold it in.

## Acceptance Criteria

- [ ] AC-1: `npm audit` in the root and in `cloud/` lists only the `fixAvailable: false` chain owned by #300. No `fixAvailable: true` advisory remains, and the critical one is gone.
- [ ] AC-2: No manifest range moves except where a fix needs it. The PR lists every direct dependency whose resolved version changed.
- [ ] AC-3: Brain unit CI and the MCP server suites pass on the refreshed lock.
- [ ] AC-4 *(post-merge)*: the next plane image built from `dev` boots, and a Memory Core and a Knowledge Base healthcheck answer.

## Out of Scope

The `sharp` / `transformers` / `default-embed` chain (#300). Institution's own lockfile (neomjs/neo-agent-institution#124 tracks its sharp consumer).

## Related

#300 · #213 (the two-profile manifests) · #1438

Live latest-open + closed sweep: the latest 20 open Brain issues plus closed and open searches ("npm audit advisories lockfile", "audit OR advisory OR lockfile OR dependabot") at 2026-11-05T10:3xZ. Open hits: #300 (the sharp chain, excluded above) and unrelated audits (#70, #34, #167). No equivalent. A2A (the last 30, all read-states): no claim on Brain dependencies. MC: no prior decision. Own-assignment sweep: no same-surface ticket.

Origin Session ID: 2c2f180d-6925-4573-9481-ceaf255b7ba6

Retrieval Hint: "Brain root and cloud lockfiles fixable npm audit advisories proxy-addr critical"

Authored by Vega (Opus 5.5, Claude Code)

## Timeline

- 2026-11-05T10:36:17Z @neo-opus-vega added the `bug` label
- 2026-11-05T10:36:17Z @neo-opus-vega added the `dependencies` label
- 2026-11-05T10:36:18Z @neo-opus-vega added the `ai` label
- 2026-11-05T10:36:18Z @neo-opus-vega added the `security` label
- 2026-11-05T10:36:18Z @neo-opus-vega added the `agent-os` label
- 2026-11-05T10:36:23Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-05T10:36:52Z @neo-opus-vega cross-referenced by #61

