---
id: 554
title: 'REFERENCES_TICKET links a message to whatever node carries the raw ticket string, never to the ticket'
state: OPEN
labels:
  - bug
  - ai
assignees: []
createdAt: '2026-09-26T19:36:08Z'
updatedAt: '2026-09-26T19:36:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/554'
author: neo-opus-vega
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
# REFERENCES_TICKET links a message to whatever node carries the raw ticket string, never to the ticket

## Context

Found while rebuilding the pruned message edges (#538, the 2026-09-26 apply) and confirmed by Euclid's REM audit: the rebuild reported 89,984 `relatedTickets` fields with a missing target against 313 linkable, and the ones it could link were mostly ticket numbers that happened to match a tag concept. Live graph at 19:32Z (read-only): 319 `REFERENCES_TICKET` edges in total; 310 of them target a bare number (`N`), the rest `vN.N`, `ADR-N`, a repo name and one `ISSUE:N`. Not one targets a ticket node. The mc-server log culls the rest continuously: `Culling hallucinated edge mapping: MESSAGE:… -> neomjs/neo#19273 (count was 1)`.

## The Problem

The mailbox projection links each `relatedTickets` string as the edge target verbatim (`MailboxService.mjs:2884–2885`, `linkOptionalMailboxEdge(messageId, t, 'REFERENCES_TICKET', …)` → `GraphService.linkNodes`), and `linkNodes` only asks whether a node with that id exists. Messages carry tickets as `owner/repo#N`; no node has that id, so the edge is skipped and later culled. When a sender wrote a bare number, it matches the tag concept a `taggedConcepts` entry with the same digits created (`ensureTaggedConceptNode`), so the message's "ticket" edge points at a tag. `rebuildMessageEdges.mjs` (#543) inherits both: its `nodeExists` test is the same unqualified id lookup, which is why its receipt carries the 89,984 missing targets and why the 227 links it declined at the rerun were tag collisions (#538's AC-3 receipt).

The consequence is the one the mailbox lane is for: the graph holds no message → ticket edge, so nothing that walks from a ticket to the swarm's conversation about it (Golden Path support, `who_is_online`'s review trail by PR, the Dream's session ↔ ticket topology) sees the mailbox.

## The Architectural Reality

- The ticket-side ids come from the ingestors, not from the message: the corpus projection writes issue chunks as `file-resources/content/archive/issues/<release>/chunk-N/issue-N.md` nodes, and `IssueIngestor` writes its own issue/PR nodes (id convention to be read at the fix, first step below).
- `MailboxService#resolveRelatedPullRequestStates` (`:3733`) already parses ticket numbers out of `relatedTickets` strings for the PR-state read; the resolver the edge needs is the same parse plus a lookup by the ingestor's convention.
- Labels are on the node (`json_extract(data, '$.label')`), and #553 adds an index on them, so "is this node a ticket" is a cheap qualifier.

## The Fix

1. Read the ticket node id convention the ingestors write for `owner/repo#N` (and for a bare `#N`, whose repo is the message's default) and put one resolver beside `resolveRelatedPullRequestStates`.
2. The projection links `REFERENCES_TICKET` to the resolved ticket node only, and never to a node whose label is `CONCEPT`; an unresolvable reference stays unlinked and is counted, not culled later.
3. `rebuildMessageEdges.mjs` uses the same resolver for its `REFERENCES_TICKET` pass, so a rebuild after this lands links the 27,372 messages that carry the field.

## Acceptance Criteria

- [ ] AC-1: a message with `relatedTickets: ['neomjs/neo#19273']` gets a `REFERENCES_TICKET` edge to the ingested ticket node for that issue, and a message whose `relatedTickets` number equals an existing tag concept's id gets no edge to that concept (unit arms, red-first).
- [ ] AC-2: `rebuildMessageEdges --types REFERENCES_TICKET` dry run on the local plane reports linkable > 0 against ticket nodes and 0 links to `CONCEPT` nodes.
- [ ] AC-3: no `Culling hallucinated edge mapping: MESSAGE:… -> owner/repo#N` line is logged for a message written after the fix is deployed.

## Out of Scope

- The other three rebuilt types (#538 delivered them).
- Ticket nodes for repositories the ingestors do not cover.

## Related

#538 · #543 · #64 · #553

unowned-rationale: found and measured while closing the REM lane; I hold #552/#553's post-merge receipts first. Any seat may take it; I take it when those are done if it is still open.

Live latest-open sweep: checked the latest 20 open Brain issues at 19:31Z; no equivalent. A2A claim sweep: the mailbox reads at 19:09Z–19:30Z carried claims on #255, #19285 and the wake hooks, none on this scope. Memory sweep: `query_raw_memories` timed out (mc-server saturated, the #552 symptom). Own-assignment sweep: none of my open Brain tickets covers it.

Origin Session ID: 27467eea-851e-486b-a0ca-55f744b67fdf
Retrieval Hint: "REFERENCES_TICKET raw relatedTickets string target tag concept collision missing target rebuildMessageEdges resolver"

## Timeline

- 2026-09-26T19:36:09Z @neo-opus-vega added the `bug` label
- 2026-09-26T19:36:10Z @neo-opus-vega added the `ai` label

