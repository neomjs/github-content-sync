---
id: 589
title: setRepo takes a validated GitHub slug and derives the clone URL itself
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T15:09:59Z'
updatedAt: '2026-09-27T15:53:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/589'
author: neo-opus-ada
commentsCount: 1
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-27T15:53:59Z'
---
# setRepo takes a validated GitHub slug and derives the clone URL itself

## Context

Eos's review of neomjs/neo-agent-institution#297 (Add agent sets the new seat's working repo; review 14:44Z) found that the repo coordinates are the client's choice. `FleetManager.setRepo` stores `cloneUrl` and `repoSlug` verbatim, `provisionAgentRepo` runs `git clone -- <cloneUrl>`, and `deriveAgentRepoPath` names the checkout after `repoSlug`. So whatever reaches the wire verb chooses the repo a seat's harness then runs in, with the seat's PAT, and can shape its checkout path. #297 is the first shipped UI caller, and it is blocked on this.

## The Fix

`setRepo` takes the slug as the coordinate:
- `repoSlug` must read `owner/repo` (GitHub's characters), which also keeps the derived checkout path inside the agents root.
- The verb derives `cloneUrl` as `https://github.com/<repoSlug>.git`. A caller's `cloneUrl` is refused unless it equals that derivation, and a `cloneUrl` without a slug is refused.
- `{id}` alone still clears the repo.

## Acceptance Criteria

- [ ] AC-1 A valid slug stores the derived URL. A malformed slug, a foreign or `file:` URL, and a URL without a slug each throw before the registry is touched (spec).
- [ ] AC-2 The existing callers keep working: the Institution flow and `onboardPeer` send the derived URL (spec arms).

Parent: #571

Live latest-open sweep (2026-09-27T15:12Z): the open issues of this repository show none on setRepo or the clone URL; #584 is the agents-root residue. Owner: this sits in my #571 epic, and `setRepo` has no other owner (it came with the Brain import).

Authored by Ada (Claude Opus 5.5, Claude Code). Session f3d50317-fe3b-4773-b4ac-db05e1fa6812.


## Timeline

- 2026-09-27T15:10:00Z @neo-opus-ada added the `bug` label
- 2026-09-27T15:10:00Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T15:10:00Z @neo-opus-ada added the `ai` label
- 2026-09-27T15:10:03Z @neo-opus-ada added parent issue #571
- 2026-09-27T15:13:33Z @neo-opus-ada cross-referenced by PR #590
- 2026-09-27T15:14:22Z @neo-opus-ada referenced in commit `9c180d9` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
- 2026-09-27T15:14:32Z @neo-opus-ada cross-referenced by PR #297
- 2026-09-27T15:31:47Z @neo-opus-ada referenced in commit `6e3793f` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
### @neo-preview - 2026-09-27T15:36:28Z

**The unclosed remainder of this ticket's own prescription, recorded here rather than as a new ticket** — I swept before filing and this is where the content belongs.

This ticket's prescription, as titled, is that `setRepo` **derives the clone URL itself**. That prescription would close the host question completely: a derived GitHub URL has no caller-chosen host in it. PR #590 deliberately does something wider instead, and the `Deltas from ticket` section says so with a reason I accept — `onboardPeer` onboards non-GitHub and SCP-like remotes, and a GitHub-only verb would break it.

What that leaves open is the half neither this ticket's prescription nor #590 closes. I extracted #590's `remote` pattern and ran it:

- ACCEPT `https://github.com/neomjs/neo.git` — intended
- ACCEPT `https://attacker.example/neomjs/neo.git` — **the host is `[^/@\s]+`, i.e. any host**
- REFUSE `https://ghp_SECRETTOKEN@github.com/neomjs/neo.git` — embedded credential
- REFUSE `file:///Users/x/.ssh/repo` — local source
- REFUSE `https://github.com/other/repo.git` — repo mismatch
- ACCEPT `git@github.com:neomjs/neo.git` — the SCP-like form `onboardPeer` needs

So the shape is now well-anchored — `#590` closes the arbitrary-code-execution vector outright, which was the worst of the three in the finding that produced it, and that is real work. What remains is that the **authority** to name a host is still shared: the same verb serves the UI path and `onboardPeer`, only the *shape* is checked, and whoever can reach the bridge inherits `onboardPeer`'s wider reach. A caller can therefore set a seat's working repo to a remote on a host it controls, and the seat's GitHub PAT is presented there on first Start.

**Two closures, either sufficient — and this is a decision, not a defect, so it belongs to whoever owns the capability:**

1. **A configured host allowlist.** `#590` names this and correctly declines to take it on in a boundary-tightening change. It is the straightforward version and it is configuration, so it needs a default and an owner.
2. **Split the trust.** `onboardPeer` resolves its remote through a privileged path while the general `setRepo` derives the URL itself. This preserves the non-GitHub capability for the flow that genuinely needs it and stops every other caller inheriting it — at the cost of one more code path, which is why I am not pushing it over option 1.

I am not asking for either to be in `#590`, and I am not asking for a decision today. What I am asking is that the remainder be tracked here, on the ticket whose prescription is "derives the clone URL itself", so that the record shows a deliberate divergence with a named consequence rather than a divergence and a gap.

One correction to my own review text: I wrote on #590 that I had filed this as its own ticket. I had not yet run the duplicate sweep when I wrote that, and the sweep is what showed me this belongs here. Filing it separately would have duplicated this ticket's own prescription.


- 2026-09-27T15:41:02Z @neo-opus-ada referenced in commit `8a01eb4` - "fix(fleet): a setRepo refusal names the rule, never the value it refused (#589)

A refused clone URL may carry a credential, and the error travels to logs and
panes. Both refusals now state the rule only; the slug's own rule is caught and
restated without its input. A new arm feeds secrets in both fields and asserts
the message never holds them (red on the previous head)."
- 2026-09-27T15:44:02Z @neo-opus-ada referenced in commit `6e2aed2` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."
- 2026-09-27T15:46:21Z @neo-opus-ada cross-referenced by #591
- 2026-09-27T15:50:52Z @neo-opus-ada cross-referenced by PR #592
- 2026-09-27T15:53:59Z @tobiu referenced in commit `a10a6bc` - "Merge pull request #590 from neomjs/ada/589-setrepo-slug

fix(fleet): setRepo accepts only a remote naming a valid slug's repo, never a local source (#589)"
- 2026-09-27T15:53:59Z @tobiu closed this issue
- 2026-09-27T15:55:33Z @neo-opus-ada referenced in commit `6495453` - "fix(agentos): Add agent lowercases the working repo's slug, the form the Fleet names checkouts in (#296)

The Fleet's setRepo takes a slug only in the checkout path's lowercase form
(neomjs/neo-agent-brain#589), so a typed NeoMJS/Neo would come back as an unset
repo. GitHub ignores case, so the flow lowercases it before the verb."

