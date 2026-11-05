---
id: 9555
title: 'Feature: Implementation of Data-Worker Side Caching'
state: OPEN
labels:
  - enhancement
  - help wanted
  - no auto close
  - ai
  - performance
  - core
assignees: []
createdAt: '2026-03-25T20:28:46Z'
updatedAt: '2026-11-05T10:43:01Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9555'
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
# Feature: Implementation of Data-Worker Side Caching

### Goal
Implement an intelligent caching layer within the Data Worker to reduce redundant network requests.

### Description
By caching shaped data results inside the Data Worker, we can significantly improve TTI for repeated views.

**Requirements:**
1. Add a `cache: boolean` configuration to `Neo.data.Pipeline`.
2. Implement a key-value cache in the Data Worker using a hash of the `read()` parameters.
3. Support `TTL` (Time-To-Live) configurations.
4. Provide an API to explicitly clear/invalidate the cache (`pipeline.clearCache()`).

## Timeline

- 2026-06-23T03:33:42Z @neo-gpt-euclid cross-referenced by #9554
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:43:01Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - Inspected `src/data/Pipeline.mjs:24-94, 114-175`: `Pipeline` has no `cache` / `ttl` config or `clearCache()` method for caching shaped results inside the Data Worker.
- **Verdict (`op`):** **Holds** (premise verified against `src/data/Pipeline.mjs:24-94`; unimplemented caching enhancement to `Neo.data.Pipeline`).

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


