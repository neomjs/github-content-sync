---
id: 4817
title: Required should be part of the base field
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - core
assignees:
  - neo-gemini-maryam
createdAt: '2023-09-01T11:05:43Z'
updatedAt: '2026-11-03T10:30:36Z'
githubUrl: 'https://github.com/neomjs/neo/issues/4817'
author: r-l-d
commentsCount: 4
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
# Required should be part of the base field

Currently the `Required` config is part of the `Text` field, but it should be part of the `Base` field


## Timeline

- 2023-09-01T11:05:43Z @r-l-d added the `enhancement` label
### @github-actions - 2024-08-29T02:26:44Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:44Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:29:11Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:29:12Z @github-actions closed this issue
- 2026-10-09T20:38:41Z @neo-opus-grace reopened this issue
- 2026-10-09T20:38:43Z @neo-opus-grace removed the `stale` label
### @neo-opus-grace - 2026-10-09T20:38:45Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, valid but later.** Thank you @r-l-d. `required_` still lives on `form.field.Text`, not on `form.field.Base`.

- 2026-10-09T20:41:46Z @neo-opus-grace cross-referenced by #19489
- 2026-11-03T10:30:28Z @neo-gemini-maryam added the `ai` label
- 2026-11-03T10:30:28Z @neo-gemini-maryam added the `core` label
- 2026-11-03T10:30:28Z @neo-gemini-maryam added the `refactoring` label
### @neo-gemini-maryam - 2026-11-03T10:30:32Z

### Ticket Triage & Intake (`#4817`)

Triaged per `ticket-triage` skill. Applied: `ai`, `core`, `refactoring` (alongside existing `enhancement`). Stage retrospective passed. Self-assigning and proceeding to `ticket-intake`.

#### Retrospective Six-Stage Challenge & Intake Classification
1. **Premise (`Verify-Before-Assert`):** Verified in [`src/form/field/Base.mjs`](https://github.com/neomjs/neo/blob/dev/src/form/field/Base.mjs): `required_` is absent from `Neo.form.field.Base.config` and `Neo.form.field.Base#toJSON()`, and is instead duplicated across [`src/form/field/Text.mjs`](https://github.com/neomjs/neo/blob/dev/src/form/field/Text.mjs#L238-L241) and [`src/form/field/CheckBox.mjs`](https://github.com/neomjs/neo/blob/dev/src/form/field/CheckBox.mjs#L131-L134) (which also contains a shadowed duplicate `toJSON()` method at lines 717–729). Meanwhile, [`src/form/field/FileUpload.mjs`](https://github.com/neomjs/neo/blob/dev/src/form/field/FileUpload.mjs#L959) extends `Neo.form.field.Base` directly and already reads `me.required` inside `isValid()`, yet lacks a reactive `required_` config and `afterSetRequired` hook.
2. **Prescription:** `Prescription checked: src/form/field/Base.mjs — owns the concern`. Lifting `required_: false`, `afterSetRequired(value, oldValue)` (`oldValue !== undefined && this.validate(false)`), and `required: me.required` in `toJSON()` up to `Neo.form.field.Base` removes the duplicated config, hook, and serialization entries from `CheckBox.mjs` and `Text.mjs` (plus the shadowed duplicate `toJSON()` in `CheckBox.mjs`), while making `FileUpload#isValid()`'s `me.required` check reactive.
3. **Substrate:** Engine core (`src/form/field/Base.mjs`, `src/form/field/CheckBox.mjs`, `src/form/field/Text.mjs`) + unit test (`test/playwright/unit/form/field/BaseRequired.spec.mjs`).
4. **Consumer:** All `Neo.form.field.Base` subclasses (`Text`, `CheckBox`, `Radio`, `FileUpload`, `Hidden`, custom fields) and `Neo.form.Container`.
5. **Service-Boundary:** App-Worker form field hierarchy only.
6. **Decision Record impact:** None.

#### Intake Classification Ledger (`ticket-intake §1.8` & `§1.9.6`)
- **Ticket age:** `createdAt: 2023-09-01T11:05:43Z`, `updatedAt: 2026-10-09T20:38:45Z`
- **Bot stale-band:** `pre-stale` (`stale`: false, `no auto close`: false; reopened and verified valid by `@neo-opus-grace` in `#19489` set C on `2026-10-09`)
- **Currency / successor-risk evidence:** No open/merged PR or duplicate ticket covers lifting `required_` to `form.field.Base`; verified against live `dev` (`src/form/field/Base.mjs`, `src/form/field/CheckBox.mjs`, `src/form/field/Text.mjs`, `src/form/field/FileUpload.mjs`).
- **Core-Idiom Pre-Flight (`§1.9.6`):** Read `src/core/Base.mjs`, `src/Neo.mjs`, and `src/state/Provider.mjs`.
- **Verdict:** `valid-as-written` (positive ROI, net line deletion across `src/form/field/`).

Authored by Maryam (Gemini 4 Argon, Antigravity). Session `dadd700f-6b16-47b5-b22c-2f57073f86f3`. 🧭

- 2026-11-03T10:30:36Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-03T10:34:00Z @neo-gemini-maryam cross-referenced by PR #20243

