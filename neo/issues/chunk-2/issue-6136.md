---
id: 6136
title: form.field.Select
state: OPEN
labels:
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2024-12-05T19:02:44Z'
updatedAt: '2026-11-06T06:07:20Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6136'
author: tobiu
commentsCount: 2
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
# form.field.Select

A ComboBox supports typing into an input field, while a default `select` tag only supports selecting an `option`.

This topic gets interesting, since there is a stage 2 proposal => already implemented in Chrome, but hidden inside an experimental feature: https://developer.chrome.com/blog/rfc-customizable-select

We should create the base class and make it configurable & extensible for custom option renderers.

It should extend `form.field.Base` (and not `form.field.Text`). It should use a store, but also work with passing an options array (creating a default store as needed).

Scope: neo v9

## Timeline

- 2024-12-05T19:02:44Z @tobiu added the `enhancement` label
- 2024-12-05T19:02:44Z @tobiu added the `no auto close` label
- 2024-12-05T19:02:44Z @tobiu assigned to @tobiu
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:44Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. There is still no `form.field.Select`; `form.field.ComboBox` is the only picker field. The idea stands, but two lines in the body date from 2024: "Scope: neo v9", and the experimental status of the customizable-select proposal it cites. Re-check that status before pickup.

**Verdict: rewrite** the scope and platform lines; the ticket holds after that. The ticket is operator-authored, so the rewrite is the operator's call; it is listed on #15000.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-gpt-sophie - 2026-11-06T06:07:20Z

### Current-platform and source read · 2026-11-06

The core request remains distinct: there is still no `src/form/field/Select.mjs`; `ComboBox` is a Text-derived picker/list. The historical platform sentence is now out of date: customizable select shipped in [Chrome 135](https://developer.chrome.com/blog/a-customizable-select) and [Safari 27](https://webkit.org/blog/18325/webkit-features-for-safari-27-0/). [MDN still marks it limited availability](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Customizable_select), so a usable classic native-select fallback remains part of the shape. Rich option content must retain a text/accessibility label, as [WebKit's fallback guidance](https://webkit.org/blog/18117/the-golden-rule-of-customizable-select/) explains. The `neo v9` line is historical, not a current implementation constraint.

The original Base/Store prescription still maps to current owners:
- `form.field.Base` now owns `required_`, value-change sequencing, form naming, touched state, submission and reset. Direct-Base siblings `CheckBox` and `FileUpload` establish native listener/input/label patterns; Base itself supplies neither an input layout nor substantive validation.
- Store instances can be borrowed; options arrays can create an owned default Store. Replacement/destruction must detach listeners without destroying borrowed state. This field does not need ComboBox's filtering projection, and copying its Model instance would repeat the ownership defect tracked in #20403 / PR #20406.
- `main.DomEvents#onChange` already carries native `value` and validity, but the keyboard input classification currently names only INPUT/TEXTAREA. A native Select must be tested inside selection regions so the Engine does not consume its arrow keys.

A concrete implementation contract should settle typed/falsy key values, values supplied before data, selected-record removal/filtering, Store swaps, form set/reset/submit/required behavior, and custom option rendering with a plain-text fallback. The acceptance needs actual native keyboard interaction and an unrelated rerender preserving the current selection; a VDOM-only fixture is insufficient.

This is an intent/platform update and proposed acceptance map, not a claim or a rewrite of the operator's body. Assignment remains with @tobiu. Ada confirmed she holds no implementation claim or superseding contract. Three Memory Core framings found no specific prior Select implementation; source and the primary platform references above carry this read.

Sophie · GPT-6 Astra Ultra · Codex Desktop · session 2127005d-d2e7-46c1-a8f5-9e1575bc6fae


