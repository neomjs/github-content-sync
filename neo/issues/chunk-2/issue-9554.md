---
id: 9554
title: 'Enhancement: Add Data Pipeline Telemetry & Performance Metrics'
state: OPEN
labels:
  - enhancement
  - help wanted
  - no auto close
  - ai
  - performance
  - core
assignees: []
createdAt: '2026-03-25T20:12:22Z'
updatedAt: '2026-11-05T10:42:57Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9554'
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
# Enhancement: Add Data Pipeline Telemetry & Performance Metrics

### Goal
Capture and report granular performance metrics for every stage of the Data Pipeline.

### Description
To support the strategic "Benchmark Reports" and assist in debugging complex data flows, the Pipeline should capture telemetry.

**Requirements:**
1. Capture high-resolution timestamps for:
   - `connectionStart / End`
   - `parserStart / End`
   - `normalizerStart / End`
2. Track payload sizes (raw vs. parsed).
3. Return a `telemetry` object to the App Worker alongside the shaped data.
4. Integrate with `Neo.util.Logger` for optional performance profiling in the console.

## Timeline

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:42:57Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - Inspected `src/data/Pipeline.mjs:114-232`: `execute()` and `onConnectionPush()` invoke `connection`, `parser`, and `normalizer` directly without capturing `Performance` stage timestamps (`connectionStart/End`, `parserStart/End`, `normalizerStart/End`) or payload-size telemetry.
- **Verdict (`op`):** **Holds** (premise verified against `src/data/Pipeline.mjs:114-175`; unimplemented telemetry enhancement to `Neo.data.Pipeline`).

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


