---
id: 235
title: 'The credential window cuts off its buttons: a fixed height shorter than its content'
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T09:03:17Z'
updatedAt: '2026-09-26T09:31:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/235'
author: neo-opus-ada
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T09:31:07Z'
---
# The credential window cuts off its buttons: a fixed height shorter than its content

## Context

@tobiu, 2026-09-26, on the credential window that #227 / PR #231 shipped: *"too little height => scrollable vertically, while there would be enough space => height increase recommended."* In their screenshot the window opens as a macOS sheet over the cockpit, and Cancel / Connect sit cut off at its bottom edge.

## The Problem

`createCredentialPrompt` (`harness/credentialPrompt.mjs`) opens the window at a fixed `height: 260` × `width: 540`. What the document needs is:
- the heading
- a two-line lede (the final wording wraps at 540 px)
- the label, the field row, and the footer buttons
- 28 px of padding top and bottom

That is more than 260 px, so the page scrolls inside a window that has room to grow. The Electron probe in #231 caught the pixels of an earlier, one-line lede. The final copy came after it, and I never re-measured the layout. That is the whole cause.

## The Architectural Reality

- The window is modal to the cockpit window, so on macOS it is a sheet: no title bar of its own, and the sheet's frame is its content.
- The page is script-free by design (ADR 0034 §2.3 item 9). The window's own preload runs in an isolated world and already wires the form, so it can measure the document without adding page script.
- Any fixed number stays wrong for other wording, fonts or a longer label. The height has to come from the content.

## The Fix

1. The preload reports the document's natural height (`document.documentElement.scrollHeight`) to main once, on `DOMContentLoaded`, over the window's own `webContents.ipc`.
2. Main sizes the content area to that height with `setContentSize`, clamped to a sane band. The window is created with `useContentSize: true`, so width and height mean the page, with or without a title bar.
3. The initial height becomes a floor that fits the known content, so a window that shows before the report arrives still isn't cut.

## Acceptance Criteria

- [ ] AC-1 With the shipped wording, the window shows every control, including the Cancel / Connect row, with no vertical scroll. An Electron receipt opens the window as a modal sheet over a parent (the operator's case) and reads `scrollHeight <= innerHeight`, with a screenshot.
- [ ] AC-2 Unit arms: a height report sizes the content area (clamped), a non-finite or foreign report is ignored, and the preload reports once.
- [ ] AC-3 No page script is added. The measurement lives in the window's own preload.

## Out of Scope

- The window's width.
- Its wording.
- The cockpit's own PasswordFields.

## Related

- #227 / PR #231 shipped the window.
- #7 is the Electron shell epic.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-26T09:03Z; no equivalent found.

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a

## Timeline

- 2026-09-26T09:03:18Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T09:03:19Z @neo-opus-ada added the `bug` label
- 2026-09-26T09:03:19Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:03:19Z @neo-opus-ada added the `design` label
- 2026-09-26T09:03:24Z @neo-opus-ada added parent issue #7
- 2026-09-26T09:05:56Z @neo-opus-ada cross-referenced by PR #236
- 2026-09-26T09:31:07Z @tobiu referenced in commit `df48b5b` - "Merge pull request #236 from neomjs/ada/235-credential-window-height

fix(shell): the credential window fits its content, so its buttons are never cut off (#235)"
- 2026-09-26T09:31:08Z @tobiu closed this issue
- 2026-09-26T09:34:23Z @neo-opus-ada cross-referenced by #241
- 2026-09-26T09:34:26Z @neo-opus-ada cross-referenced by #242
- 2026-09-26T09:34:27Z @neo-opus-ada cross-referenced by #243
- 2026-09-26T09:34:29Z @neo-opus-ada cross-referenced by #244
- 2026-09-26T09:34:30Z @neo-opus-ada cross-referenced by #245
- 2026-09-26T09:34:32Z @neo-opus-ada cross-referenced by #246
- 2026-09-26T09:34:33Z @neo-opus-ada cross-referenced by #247

