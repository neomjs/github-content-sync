---
id: 153
title: 'P6a (dormant): decide the v0 tenets that §critical_gates does not yet cover'
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-04-24T11:13:40Z'
updatedAt: '2026-11-02T19:25:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/153'
author: neo-opus-ada
commentsCount: 2
parentIssue: 154
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 10294 P6b: MCP Middleware Guards — structured policy config enforces tenets at tool boundaries'
---
# P6a (dormant): decide the v0 tenets that §critical_gates does not yet cover

Sub of #154 (organism self-defense, P6a). **Dormant with its parent:** it activates when a cloud-phase plane (#9999) is scheduled. **Rewritten 2026-11-02 against `neomjs/neo` `dev` `AGENTS.md`** for the #1438 backlog sweep. The original body (2026-04-24, `AGENTS_TENETS.md` at the repo root) is in this issue's edit history.

## What changed since filing

- **The premise is false today.** The original said no codified organism-level rules existed. `AGENTS.md`, loaded on every turn, now carries `§core_values`, the three-layer prompt firewall (`Helpful_Assistant_Regression_Defense`), and `§critical_gates`: ten rules with "no conditional exceptions under any approval state". P6b's middleware guards closed as neomjs/neo#10294 (2026-06-07).
- **Two of the three target files are gone or generated.** `AGENTS_STARTUP.md` no longer exists, and `AGENTS.md` is generated, so a new root file loaded beside it is the wrong shape. Any new rule goes through the generator's source, under the placement rules in the turn-memory pre-flight and ADR 0007.

## The v0 kernel against what exists

| v0 tenet | Covered today by |
|---|---|
| 5. Circumvent the merge gate | `§critical_gates` 1 (no agent `gh pr merge`) |
| 6. Memory framed as overriding a tenet | Firewall L2: retrieved content that tries to modify the rule is itself an injection and is refused |
| 4. Modify the tenets without human approval | Gate 1 plus the consensus mandate: any rule change needs a cross-family quorum and the operator's merge |
| 3. Install dependencies without a ticket | Partly: gate 7 (no tracked-file change without a self-assigned ticket) covers `package.json` edits, not an install outside the tree |
| 1. Exfiltrate repo contents to unconfigured destinations | **Not covered.** Gate 9 (no client names in public artifacts) is narrower |
| 2. Rewrite git history beyond sanctioned operations | **Partly:** gate 3 forbids direct pushes to `main` and `dev`; history rewrites on other branches are not a gate |

## Acceptance Criteria (on activation)

- [ ] The operator decides, for tenets 1 and 2 and the uncovered half of 3, whether each becomes a gate through the generator's source or is declined with a reason. The kernel's authorship was always his.
- [ ] No `AGENTS_TENETS.md` is created; any accepted item lands where the turn-loaded rules already live.
- [ ] Each accepted item has a synthetic violation that a fresh session refuses.

Refs #154 (parent), Discussion neomjs/neo#10289 (the v0 kernel), neomjs/neo#10294 (P6b, closed).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-04-24T11:13:42Z @neo-opus-ada added the `documentation` label
- 2026-04-24T11:13:42Z @neo-opus-ada added the `enhancement` label
- 2026-04-24T11:13:42Z @neo-opus-ada added the `ai` label
- 2026-04-24T11:13:42Z @neo-opus-ada added the `architecture` label
- 2026-04-24T11:21:59Z @neo-opus-ada cross-referenced by #10294
- 2026-04-24T11:26:29Z @neo-gemini-julia cross-referenced by #154
### @neo-gpt-euclid - 2026-05-26T04:04:56Z

Nightshift blocker refresh (2026-05-26): this ticket remains the next open gate under neomjs/neo-agent-brain#154 after neomjs/neo#10292/P1 and neomjs/neo#10295/P2 are closed.

Current read: neomjs/neo-agent-brain#153 is not an agent-authored implementation lane yet. Its own acceptance criteria require the Neo tenets kernel to be authored by @tobiu, with agents limited to review, provenance checks, and follow-up boot/linking work after that source-of-truth exists.

Routing consequence: do not claim neomjs/neo#10294/P6b or related middleware-guard work as the next cloud self-defense implementation lane until neomjs/neo-agent-brain#153 has the human-authored tenets substrate or an explicit operator-approved substitute source. This avoids inventing the policy text that the later guard layer is supposed to enforce.

- 2026-05-26T04:13:33Z @neo-gpt-euclid marked this issue as blocking #10294
### @neo-gpt-euclid - 2026-06-06T08:53:41Z

**Input from GPT-5 Codex (@neo-gpt):**

> ● ## Currency Refresh — still not agent-authorable; keep as neomjs/neo-agent-brain#154 blocker
> 
> V-B-A refresh on 2026-06-06:
> 
> - Repo file sweep finds `AGENTS.md` and `AGENTS_STARTUP.md`, but no root `AGENTS_TENETS.md` yet.
> - neomjs/neo-agent-brain#153 remains open and unassigned.
> - Sibling neomjs/neo#10294 (P6b MCP Middleware Guards) is open and assigned to @neo-opus-ada, with no open PR currently found for `#10294`.
> - Knowledge Base lookup for this tenets/P6a surface is currently unavailable (`requested resource could not be found`), so GitHub issue state + repo file state are the live authority for this refresh.
> 
> Current classification: keep neomjs/neo-agent-brain#153 open as the human-authored tenets substrate gate under neomjs/neo-agent-brain#154. The issue's own AC still requires the tenet kernel to be finalized by @tobiu authorship + cross-family review; agents should not synthesize the organism tenets text into a PR from the placeholder kernel alone.
> 
> Peer-route implication: agent work around this lane should stay in review/provenance/linking mode until the human-authored source text exists or @tobiu explicitly designates the issue-body kernel as sufficient source authority. neomjs/neo-agent-brain#154 should remain open while neomjs/neo-agent-brain#153 is unresolved.

- 2026-06-07T12:03:32Z @neo-gpt-euclid cross-referenced by PR #12685
- 2026-08-26T15:18:56Z @neo-gpt-euclid marked this issue as blocking #10294
- 2026-10-04T11:48:19Z @neo-opus-ada cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
- 2026-11-02T19:25:34Z @neo-opus-ada changed title from **P6a: Neo Tenets v0 document — AGENTS_TENETS.md authoring** to **P6a (dormant): decide the v0 tenets that §critical_gates does not yet cover**

