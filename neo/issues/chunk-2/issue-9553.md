---
id: 9553
title: 'Feature: Implement Pipeline Interceptor System (Middleware)'
state: OPEN
labels:
  - enhancement
  - help wanted
  - no auto close
  - ai
  - architecture
  - core
assignees: []
createdAt: '2026-03-25T20:10:19Z'
updatedAt: '2026-11-05T10:42:54Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9553'
author: tobiu
commentsCount: 1
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
# Feature: Implement Pipeline Interceptor System (Middleware)

### Goal
Implement a middleware/interceptor system for `Neo.data.Pipeline` to allow cross-cutting concerns to be handled declaratively.

### Description
As the Data Pipeline architecture matures, we need a way to inject logic at key stages of the data lifecycle (pre-request, post-response, post-parse).

**Requirements:**
1. Support `request` interceptors (e.g., adding Auth headers, logging params).
2. Support `response` interceptors (e.g., global error handling, refreshing tokens).
3. Interceptors must support `async` execution.
4. Allow global interceptors to be registered at the `Neo.worker.Data` level.
5. Pipelines should allow instance-specific interceptors that merge with globals.

## Timeline

- 2026-06-23T03:33:42Z @neo-gpt-euclid cross-referenced by #9554
- 2026-09-23T09:51:22Z @neo-opus-ada cross-referenced by #19105
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:42:54Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - Inspected `src/data/Pipeline.mjs:24-94, 114-232`: `Pipeline` currently orchestrates `connection_`, `parser_`, and `normalizer_` across `workerExecution: 'app' | 'data'`, with no `interceptors` config or pre-request / post-response / post-parse middleware chain on `Neo.data.Pipeline` or `Neo.worker.Data`.
- **Verdict (`op`):** **Holds** (premise verified against `src/data/Pipeline.mjs:114-175`; unimplemented enhancement to `Neo.data.Pipeline`).

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


