---
id: 245
title: 'The Accounts view: one add-agent form and a layout that fits'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-09-26T09:34:29Z'
updatedAt: '2026-09-26T09:34:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/245'
author: neo-opus-ada
commentsCount: 0
parentIssue: 13
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
# The Accounts view: one add-agent form and a layout that fits

## Context

The operator, 2026-09-26: *"accounts view: terrible design / styling."* A headless walk of dev (1280×800, dark) shows why:

- The view opens on a lone "bridge-pending" tab over a card that repeats the name, then a harness chip row, then three "declared" sections whose values sit at the far right edge, away from their labels.
- Under it, a second "Add an agent" form: `GitHub username` / `GitHub PAT` fields, then a harness radio row that overflows the view — "Codex Desktop" and "Claude Code" wrap onto two lines and "Kimi Code" is cut at the right edge — and the form's buttons.
- The cockpit's right rail has its own "Add agent" form with the same two fields and a chip picker, styled differently.
- The button row sits below the fold at 800 px.

## The Problem

One task, adding an agent, has two forms with two designs, and the Accounts one breaks at a common window size. The view has no layout of its own: it stacks a toolbar, the config card and the form in a `vbox`, and the harness picker is a row of `Radio` fields sized by their labels. This is the class #13 names ("Accounts card + toolbar" is its first per-surface leaf); no leaf is open for it.

## The Architectural Reality

- `apps/agentos/view/accounts/Panel.mjs`: a `DashboardPanel` with a `Toolbar` (the definition tabs), an `AgentConfigCard` (`view/fleet/detail/AgentConfigComponent.mjs`) and a `FormContainer` holding `TextField` "GitHub username", `PasswordField` "GitHub PAT", one `Radio` per `listHarnessTypes()` entry, and a button `Toolbar`.
- `apps/agentos/view/fleet/instances/AddAgentForm.mjs`: the rail's form, the same fields, the harness types as toggle `Button`s; in shell transport it drops the PAT field and sends public intent only.
- Both drive `apps/agentos/util/AddAgentFlow.mjs` (one flow, two views).
- Tokens: `apps/agentos/resources/tokens.css` (`--fm-*`); the SSOTs under `apps/agentos/design/`.

## The Fix

1. One add-agent form: the rail's `AddAgentForm` is the component, and Accounts mounts it (or the reverse; the one kept is the one that already honours shell transport). The duplicate retires.
2. The form names its forge once ("GitHub account") and the token field "Personal access token". GitLab agents are not offered yet: the Brain cannot launch one (`ai/services/fleet/managedAgentWorkspacePlan.mjs`: `gitlab-workflow` carries `unsupportedReason: 'FleetLifecycleService has no GitLab credential injection contract'`).
3. The harness picker fits any width: a wrapping chip group or a select, never a clipped radio row.
4. The Accounts layout: the definitions as a list beside the selected definition's card (master-detail), each declared value next to its label; empty states say what to do first.
5. Everything on the `--fm-*` tokens and the component skin layer, with before/after screenshots against the SSOT frame (the #13 gate).

## Acceptance Criteria

- [ ] AC-1 A single add-agent component serves Accounts and the rail; `AddAgentForm` or the Accounts form retires (grep in the PR).
- [ ] AC-2 The form names its forge once and labels the token "Personal access token"; the PAT field keeps its shell-transport behaviour (dropped in the shell, the credential window instead).
- [ ] AC-3 At 1000×640 and 1280×800 no control is clipped and nothing scrolls sideways (visual goldens at both sizes).
- [ ] AC-4 The declared rows keep label and value adjacent; the view has an empty state for "no definitions yet".
- [ ] AC-5 Before/after screenshots in the PR beside the SSOT frame; the design owner's review.

## Out of Scope

- GitLab agent definitions: they need the Brain's GitLab credential injection first (plane logins already take a GitHub or a GitLab PAT, #231).
- The provider-login config through AiConfig (`neomjs/neo#13521`).
- Benching (`neomjs/neo-agent-brain#28`).

## Related

#13 (parent: design conformance) · #10 · #9 (keeper views) · #231 (the shell's credential window)

unowned-rationale: design-led and claimable (#13's steward note names the cockpit owner as the natural seat, with Grace's design review); its author holds #241 and #242.

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:33:49Z — no equivalent. A2A in-flight sweep (all read states, last 60 min): no claim on Accounts. Memory Core sweep ("accounts view design add agent form"): none; `neomjs/neo#13521` is functional (AiConfig), not design. Own-assignment sweep: none open (#235 closed with #236 at 09:31Z).

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("Accounts view redesign one add-agent form GitHub GitLab harness picker")`

## Timeline

- 2026-09-26T09:34:31Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T09:34:31Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T09:34:31Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:34:31Z @neo-opus-ada added the `design` label
- 2026-09-26T09:35:04Z @neo-opus-ada added parent issue #13

