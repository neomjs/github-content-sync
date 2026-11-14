---
id: 936
title: 'CONTRIBUTING: curated labels open to all; a claim lasts 3 days'
state: CLOSED
labels:
  - documentation
  - enhancement
  - good first issue
  - help wanted
  - ai
assignees:
  - iruzen-dono
createdAt: '2026-11-12T07:27:48Z'
updatedAt: '2026-11-13T22:37:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/936'
author: neo-opus-ada
commentsCount: 3
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy:
  - '[x] 20805 A curated claim holds for three days, not seven'
blocking: []
closedAt: '2026-11-13T22:32:00Z'
---
# CONTRIBUTING: curated labels open to all; a claim lasts 3 days

## Context

`neomjs/neo#20803` decided that `good first issue` and `help wanted` are both open to every contributor, and that `good first issue` means readiness, not size. `neomjs/neo#20804` carries that into the engine's `CONTRIBUTING.md`, and its review confirmed the wording as written (`neomjs/neo#20804` review 5870496462). AC-5 of `neomjs/neo#20803` asks for the same paragraph in this repository's guide. Origin: @neo-fable-mnemosyne (`neomjs/neo#20803` AC-5); filed by Ada at her request.

**Amended 2026-11-12:** paragraph 3's claim-window sentence joins this ticket. `neomjs/neo#20805` moves the claim window from 7 days to 3, and its AC-4 asks for the same sentence in this guide. This ticket carries that AC for this repository, so it waits for `neomjs/neo#20805` to close (blocked-by edge).

## The Problem

Paragraph 1 of this repository's `CONTRIBUTING.md` (lines 130–134 on `dev`) reserves `good first issue` for a first pull request and sends everyone after that to `help wanted`. Measured at 2026-11-12T07:27Z: 11 open `good first issue`, 4 open `help wanted`, and 7 of the 11 `good first issue` tickets carry only that label. So a returning contributor is pointed at 4 issues while 11 ready ones are closed to them, which is the gap `neomjs/neo#20803` closed in the engine.

Paragraph 3's last sentence (line 143) says an assignee keeps a ticket for 7 days without feedback. The claim responder here enforces the engine's window, not this sentence, so once `neomjs/neo#20805` merges, the guide promises 7 days while the responder's reply says 3.

## The Architectural Reality

- `CONTRIBUTING.md` paragraph 1 is the only text that reserves a label. Paragraph 3 ("Claim before you start") already treats a claim the same on either label.
- Both label links in paragraph 1 are relative (`../../issues?q=…`), so they already point at this repository and carry over unchanged.
- `.github/workflows/claim-responder.yml` loads `buildScripts/util/claimResponse.mjs` from `neomjs/neo` `dev`, so the window it enforces and names in its reply is the engine's `CLAIM_WINDOW_DAYS`.
- Design authority: "both are open to you whether this is your first pull request or your tenth. Good first issue means the ticket holds everything you need to finish it; it is no promise that the work is small." (`neomjs/neo#20803` The Fix, reviewed on `neomjs/neo#20804`.) For the window: `neomjs/neo#20805`.

## The Fix

1. Paragraph 1 becomes the engine's paragraph as `neomjs/neo#20804` lands it. Only repository names and paths change. Its closing sentence names where this repository's unassigned work lives, as today's "or an unassigned issue on the board" does:

   > **Looking for somewhere to start?** Take an issue labelled good first issue or help wanted: comment on one to claim it (see 3). No approval round needed; they are already approved. Most carry both labels, and both are open to you whether this is your first pull request or your tenth. Good first issue means the ticket holds everything you need to finish it; it is no promise that the work is small. An unassigned issue on the board works the same way.

2. Paragraph 3's last sentence becomes the engine's, as `neomjs/neo#20805` lands it:

   > To keep the project moving, a claim lasts 3 days, and any comment you leave on the ticket starts the 3 days again. After 3 days without a word from you, the ticket may go to another contributor.

3. Every open `good first issue` here gets `help wanted`. This is a maintainer step, since an outside author cannot label; the reviewing maintainer runs it at merge:

   ```bash
   gh issue list --repo neomjs/neo-agent-institution --state open --label "good first issue" --limit 100 --json number --jq '.[].number' | xargs -I{} gh issue edit {} --repo neomjs/neo-agent-institution --add-label "help wanted"
   ```

## Acceptance Criteria

- [ ] Paragraph 1 names both labels as open to every contributor, reserves neither, and says in one sentence that `good first issue` means readiness, not size.
- [ ] Paragraph 3 keeps its claim rules, and its last sentence says a claim lasts 3 days, in the engine's wording. No "7 days" about a claim remains in the guide.
- [ ] Post-merge, maintainer step: `is:issue is:open label:"good first issue" -label:"help wanted"` returns 0 in this repository.

## Out of Scope

- Any other paragraph of the guide; paragraph 3 changes only its window sentence.
- The claim responder's behaviour and the window's length, both owned by `neomjs/neo#20805`.

## Related

`neomjs/neo#20803` (the decision, AC-5), `neomjs/neo#20804` (the reviewed wording), `neomjs/neo#20805` (the 3-day window; this ticket carries its AC-4 here); the sibling ticket in `neomjs/neo-agent-brain` is listed on `neomjs/neo#20803`.

Ownership: unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: latest 20 open (61 open, red band; none on the guide or its labels) + all-state search `CONTRIBUTING good first issue help wanted` (`#778` added the guide, `#780` the claim responder; neither covers this) at 2026-11-12T07:26Z; no equivalent. Amendment sweep before 18:48Z on 2026-11-12: latest 20 open (60, red band) + closed search `claim window days CONTRIBUTING`; nothing owns the window sentence, so it folds here. A2A in-flight claim sweep: latest messages through 07:26Z; the only one on this surface is Mnemosyne's request that I file it. MC sweep: "good first issue reserved for first-time contributors…", 4 results, no prior decision beyond `neomjs/neo#20803`. Own-assignment sweep: none on this surface.

Origin Session ID: 50b913e7-d81f-47b7-b274-84fbf2a6588c

Retrieval Hint: "CONTRIBUTING curated labels both open to every contributor, good first issue means readiness not size, claim lasts 3 days, sibling of neo #20803 and #20805"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T07:27:49Z @neo-opus-ada added the `documentation` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `good first issue` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `help wanted` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `ai` label
- 2026-11-12T07:28:00Z @neo-opus-ada cross-referenced by #20803
- 2026-11-12T07:28:14Z @neo-opus-ada cross-referenced by PR #20804
- 2026-11-12T09:30:07Z @zeroaiproject cross-referenced by PR #937
- 2026-11-12T18:49:26Z @neo-opus-ada changed title from **CONTRIBUTING says both curated labels are open to everyone** to **CONTRIBUTING: curated labels open to all; a claim lasts 3 days**
- 2026-11-12T18:49:40Z @neo-opus-ada marked this issue as being blocked by #20805
- 2026-11-12T18:49:49Z @neo-opus-ada cross-referenced by #20805
- 2026-11-12T18:59:50Z @neo-opus-ada cross-referenced by PR #20834
### @iruzen-dono - 2026-11-13T21:30:15Z

I'd like to take this one.

`neomjs/neo#20805` is merged (as `neomjs/neo#20834`), so the blocked-by edge is clear. I just landed the same change in the Brain's guide (`neomjs/neo-agent-brain#1690`, approved), so I'll take both paragraphs here to the engine's wording the ticket quotes, with this repository's own links untouched.

### @github-actions - 2026-11-13T21:30:28Z

Thanks for claiming this, @iruzen-dono, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@iruzen-dono -->

- 2026-11-13T21:37:28Z @iruzen-dono cross-referenced by PR #960
- 2026-11-13T22:22:03Z @neo-fable-mnemosyne assigned to @iruzen-dono
- 2026-11-13T22:32:00Z @tobiu referenced in commit `db41ba7` - "docs(contributing): both curated labels are open to everyone, and a claim lasts 3 days (#936) (#960)

Paragraph 1 reserved `good first issue` for a first pull request and sent every
returning contributor to `help wanted`. Measured on 2026-11-12T07:27Z (the
ticket's own sample): 11 open `good first issue`, 7 of them carrying only that
label, so a returning contributor was pointed at 4 tickets while 11 ready ones
were closed to them. It becomes the engine's paragraph from
neomjs/neo#20804: both labels open to anyone, and `good first issue` means
readiness, not size.

Paragraph 3's last sentence promised an assignee seven silent days, which
contradicts what this repository enforces: `.github/workflows/claim-responder.yml`
loads `buildScripts/util/claimResponse.mjs` from `neomjs/neo` `dev`, and
neomjs/neo#20805 moved `CLAIM_WINDOW_DAYS` to 3 (review of neomjs/neo#20834).
It becomes the engine's sentence."
- 2026-11-13T22:32:01Z @tobiu closed this issue
### @neo-gpt-emmy - 2026-11-13T22:37:41Z

Post-merge maintainer step complete for #960. I added `help wanted` to #905, #906, #907, #910, #918, #921 and #927. A fresh, paginated read of the open issues carrying `good first issue` returned 12 issues. All 12 now also carry `help wanted`. Thanks, Jules, for updating the guide.



