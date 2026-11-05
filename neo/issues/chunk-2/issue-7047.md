---
id: 7047
title: 'Task: Create Example for Deeply Nested Components'
state: OPEN
labels:
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2025-07-14T14:03:06Z'
updatedAt: '2026-11-05T10:53:14Z'
githubUrl: 'https://github.com/neomjs/neo/issues/7047'
author: tobiu
commentsCount: 2
parentIssue: 6992
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
# Task: Create Example for Deeply Nested Components

## Summary

Create a new example under `examples/functional/` that demonstrates deep nesting of components, mixing both classic (class-based) and new functional components.

## Rationale

To properly showcase the power and flexibility of the new functional component architecture, we need an example that goes beyond simple parent-child relationships. This example should illustrate:

1.  A functional component hosting multiple child components.
2.  A functional component hosting a classic component, which in turn hosts another functional component.
3.  A classic component hosting a functional component.
4.  State being passed down through multiple levels of nesting.
5.  The reactive updates (`component.set()`) working correctly through the entire hierarchy.

This will serve as a clear, practical guide for developers on how to structure complex UIs and manage component composition effectively.

## Acceptance Criteria

-   A new folder is created under `examples/functional/`.
-   The example contains at least three levels of component nesting.
-   The nesting hierarchy includes a mix of functional and classic components.
-   The top-level component's state drives changes in the deeply nested children.
-   The example is clean, well-documented, and easy to understand.

## Timeline

- 2025-07-14T14:03:07Z @tobiu assigned to @tobiu
- 2025-07-14T14:03:08Z @tobiu added the `enhancement` label
- 2025-07-14T14:03:08Z @tobiu added parent issue #6992
### @github-actions - 2025-10-13T02:49:42Z

This issue is stale because it has been open for 90 days with no activity.

- 2025-10-13T02:49:43Z @github-actions added the `stale` label
- 2025-10-24T10:05:50Z @tobiu removed the `stale` label
- 2025-10-24T10:05:50Z @tobiu added the `no auto close` label
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:53:13Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `examples/functional/hostComponent/` covers two levels: a functional component that hosts a classic `Button`, mounted in a classic container. Still missing are point 2's three-level mix (functional → classic → functional) and points 4–5: state passed down through several levels, and `set()` updates propagating through the whole hierarchy.

**Verdict: holds**, narrowed to points 2, 4 and 5.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


