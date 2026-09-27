---
id: 296
title: Add agent gives the new seat a working repo
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T14:17:48Z'
updatedAt: '2026-09-27T16:03:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/296'
author: neo-opus-ada
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
closedAt: '2026-09-27T16:03:03Z'
---
# Add agent gives the new seat a working repo

Blocker for the FM goal "manage our AI team with FM": an agent added in FM has no working repo.

`AddAgentFlow.createDefineAgentIntent` sends `{githubUsername, harnessType, launchOwner, credential}`, and nothing in the Institution calls the wire's `setRepo`. Start then takes `startAgentProvisioned`'s no-repo branch and launches the harness in the Fleet process's own directory instead of a provisioned clone.

## Fix

Both add-agent forms capture the working repo as `owner/repo`, prefilled with `neomjs/neo`. After a confirmed define, the flow calls `setRepo`, so the first Start clones that repo and runs in it. A malformed repo is refused before any bridge contact. A repo that cannot be set is named in the outcome, never hidden.

## Acceptance Criteria

- [ ] Both forms have the Working repo field; the flow sets it through `setRepo` after the define.
- [ ] Unit arms cover the default, a named repo, a malformed one (no bridge contact) and a failed `setRepo`; the Accounts arms assert the `setRepo` call.

Related: #245 (the form redesign itself) · neomjs/neo-agent-brain#577

Authored by Ada (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-09-27T14:17:49Z @neo-opus-ada added the `bug` label
- 2026-09-27T14:17:49Z @neo-opus-ada added the `ai` label
- 2026-09-27T14:17:49Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T14:21:35Z @neo-opus-ada cross-referenced by PR #297
- 2026-09-27T14:54:18Z @neo-opus-ada referenced in commit `b14a2ae` - "fix(agentos): Add agent gives the new seat a working repo (#296)

Both forms take the repo as owner/repo, prefilled with neomjs/neo, and the flow
calls the wire's setRepo after a confirmed define, so the first Start clones
that repo and runs in it instead of the Fleet process's own directory. A
malformed repo never reaches the bridge; one that cannot be set is named in the
outcome. AddAgentFlow.assignRepo is the one repo step both forms use."
- 2026-09-27T15:14:22Z @neo-opus-ada referenced in commit `9c180d9` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
- 2026-09-27T15:31:46Z @neo-opus-ada referenced in commit `ac7de2f` - "fix(agentos): Add agent gives the new seat a working repo (#296)

Both forms take the repo as owner/repo, prefilled with neomjs/neo, and the flow
calls the wire's setRepo after a confirmed define, so the first Start clones
that repo and runs in it instead of the Fleet process's own directory. A
malformed repo never reaches the bridge; one that cannot be set is named in the
outcome. AddAgentFlow.assignRepo is the one repo step both forms use."
- 2026-09-27T15:31:47Z @neo-opus-ada referenced in commit `6e3793f` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
- 2026-09-27T15:31:47Z @neo-opus-ada referenced in commit `c9697dd` - "test(agentos): the Accounts golden rests the pointer, so the rail tooltip #295 moved over it stays out (#296)

Since the rail tooltips open to the icon's right, the Accounts tab's tooltip can
land inside the Accounts surface when the shot follows the click, and whether
it does depends on timing. The arm now dwells, leaves and waits out the hide
before the shot, as the Observatory arm does, and the golden is re-captured
over the rebased tree with the working-repo field."
- 2026-09-27T15:44:01Z @neo-opus-ada referenced in commit `584ff13` - "fix(agentos): Add agent gives the new seat a working repo (#296)

Both forms take the repo as owner/repo, prefilled with neomjs/neo, and the flow
calls the wire's setRepo after a confirmed define, so the first Start clones
that repo and runs in it instead of the Fleet process's own directory. A
malformed repo never reaches the bridge; one that cannot be set is named in the
outcome. AddAgentFlow.assignRepo is the one repo step both forms use."
- 2026-09-27T15:44:02Z @neo-opus-ada referenced in commit `6e2aed2` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
- 2026-09-27T15:44:02Z @neo-opus-ada referenced in commit `fdef2fc` - "test(agentos): the Accounts golden rests the pointer, so the rail tooltip #295 moved over it stays out (#296)

Since the rail tooltips open to the icon's right, the Accounts tab's tooltip can
land inside the Accounts surface when the shot follows the click, and whether
it does depends on timing. The arm now dwells, leaves and waits out the hide
before the shot, as the Observatory arm does, and the golden is re-captured
over the rebased tree with the working-repo field."
- 2026-09-27T15:55:33Z @neo-opus-ada referenced in commit `dfe8b45` - "fix(agentos): Add agent gives the new seat a working repo (#296)

Both forms take the repo as owner/repo, prefilled with neomjs/neo, and the flow
calls the wire's setRepo after a confirmed define, so the first Start clones
that repo and runs in it instead of the Fleet process's own directory. A
malformed repo never reaches the bridge; one that cannot be set is named in the
outcome. AddAgentFlow.assignRepo is the one repo step both forms use."
- 2026-09-27T15:55:33Z @neo-opus-ada referenced in commit `6495453` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
- 2026-09-27T15:55:33Z @neo-opus-ada referenced in commit `711643f` - "test(agentos): the Accounts golden rests the pointer, so the rail tooltip #295 moved over it stays out (#296)

Since the rail tooltips open to the icon's right, the Accounts tab's tooltip can
land inside the Accounts surface when the shot follows the click, and whether
it does depends on timing. The arm now dwells, leaves and waits out the hide
before the shot, as the Observatory arm does, and the golden is re-captured
over the rebased tree with the working-repo field."
- 2026-09-27T16:03:03Z @tobiu referenced in commit `228db07` - "Merge pull request #297 from neomjs/ada/296-add-agent-repo

fix(agentos): Add agent gives the new seat a working repo (#296)"
- 2026-09-27T16:03:03Z @tobiu closed this issue
- 2026-09-27T16:14:16Z @neo-gpt cross-referenced by PR #592

