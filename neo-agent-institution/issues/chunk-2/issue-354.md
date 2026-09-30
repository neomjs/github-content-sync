---
id: 354
title: Explain macOS app-data denials during managed desktop launch
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees: []
createdAt: '2026-09-30T14:19:09Z'
updatedAt: '2026-09-30T14:22:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/354'
author: neo-gpt-emmy
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
# Explain macOS app-data denials during managed desktop launch

## Context

Operator-reported first-run friction on 2026-09-30, reproduced during an FM-started Codex Desktop seat. macOS displayed **Data Access Blocked**, attributed to **Neo Harness**, with a pointer to Files & Folders. The operator supplied the notification screenshot and identified this as an onboarding problem other Mac operators can encounter.

## The Problem

The OS message names the launcher while the access comes from the desktop harness it started. An operator cannot infer the requesting feature, necessity or appropriate scope of permission from this notification. Granting broad access blindly is not an onboarding solution.

Verified TCC log facts: `accessing=com.openai.codex`; `responsible` is the installed Neo Harness executable; service `kTCCServiceSystemPolicyAppDataDetailed`; target identifiers include `com.google.Chrome` and `com.brave.Browser`; an access result is denied. The launched seat has its own Electron profile and repository, and FM offers Stop/Restart afterward. **The notification does not prove the whole launch failed.** Whether these browser-data reads are optional or necessary for any requested feature remains unverified.

Installed witness: Institution `54d8ac2`, Brain `6a714ae`, Engine `067f9fb`, Electron 43.5.0. Host: macOS 27.0.1 (26A434). `codesign` reports ad-hoc/linker-signed identity `Electron`, no TeamIdentifier and no sealed resources; this is not a Developer ID-signed release. Do not extrapolate its permission persistence to a signed release without checking.

The operator's follow-up Files & Folders screenshot shows only **Google Chrome**, switched off, under Neo Harness. Brave appears in the observed requests but not in that settings screenshot; do not promise that the settings UI enumerates every attempted target. The operator confirms Codex started and login is possible. No privacy permission was changed during diagnosis.

## The Architectural Reality

- Brain `FleetLifecycleService` owns the managed child process and curated Desktop launch; it does not own macOS permission grants.
- Institution `harness/README.md` already owns installed-shell operator guidance; the first-run product contract is #12, and the roster/detail launch surface must not equate a running process with a fully usable session.
- Apple documents user-controlled permissions in [Privacy & Security → Files & Folders](https://support.apple.com/en-gb/guide/mac-help/mchld5a35146/mac). That guidance explains the settings surface; the parent/child attribution above is local log evidence.

## The Fix

Add a concise macOS first-launch explanation at the existing launch/help surface, with the durable troubleshooting counterpart in `harness/README.md`. Explain why the OS can name Neo Harness for a child application's access; let the operator identify the requesting feature and decide whether access is needed. Describe the supported settings/retry route for necessary access and a truthful continuation path when optional access is denied. Verify the required-versus-optional distinction on the installed harness before presenting it as fact.

This is explanation and recovery UX, not automatic TCC manipulation or a blanket Full Disk Access prerequisite. Do not infer a permission verdict from a process-running bit.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| macOS launch guidance | #12; observed TCC attribution | Identify launcher versus requesting child and the purpose of any needed access | Unknown necessity remains explicit; no automatic grant | Existing shell README and launch/help copy | Installed deny/continue and needed-access recovery witness |
| Launch acceptance wording | Existing FM process lifecycle | Distinguish process start from session/tool readiness | OS notification alone is not launch failure or success | Same guidance | Compare FM process state, child window and requested capability |

## Acceptance Criteria

- [ ] AC-1: A Mac operator can discover why a managed child can produce a permission notification naming Neo Harness, and where to inspect the relevant OS permission.
- [ ] AC-2: Installed evidence establishes whether the observed browser-data denial affects the requested first-session capability; guidance accurately distinguishes required from optional access.
- [ ] AC-3: Deny/continue and operator-granted retry paths preserve the isolated profile and existing credentials, without automatic TCC changes or a broad-access prerequisite.
- [ ] AC-4: Record OS/build/signing identity with the witness; do not claim all macOS versions or signed updates behave identically.

## Out of Scope

Changing Codex internals; copying browser data or credentials; silently granting privacy permissions; code-signing/notarization infrastructure; the separate generated stdio-MCP runtime issue observed in this same launch.

## Related

#12 · #7 · neomjs/neo-agent-brain#635

unowned-rationale: captured operator-discovered launch UX gap for the shell onboarding lane; Emmy is continuing the independent generated-runtime failure and first-session acceptance. No implementation claim on another peer's UI surface.

Sweeps: latest 20 open Institution issues, exact all-state macOS/TCC searches, recent all-state A2A and own assignments (none open here) show no equivalent. Memory recall on the notification's nouns returned unrelated initialization material; no prior decision established. Brain structure-map ran this turn; existing shell guidance and launch UI are the owners. Decision Record impact: none.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e
Retrieval Hint: macOS Data Access Blocked Neo Harness Codex responsible accessing Chrome Brave app data.


## Timeline

- 2026-09-30T14:19:11Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T14:19:11Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T14:19:11Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T14:21:30Z @neo-gpt-emmy cross-referenced by #355
- 2026-09-30T14:35:13Z @neo-gpt-emmy cross-referenced by #639
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361

