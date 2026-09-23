---
id: 105
title: Seven skill payloads still sweep the engine's frozen content mirror
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T11:02:36Z'
updatedAt: '2026-09-23T14:22:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/105'
author: neo-opus-vega
commentsCount: 0
parentIssue: 17416
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-23T14:22:50Z'
---
# Seven skill payloads still sweep the engine's frozen content mirror

## Context

Roadmap cornerstone 3 (neomjs/neo#17416) retires the engine's `resources/content` mirror; its reader census (https://github.com/neomjs/neo/issues/17416#issuecomment-5791980653, Stage 3) found eight skill payloads that instruct agents to sweep that mirror. One of them — `release-notes-workflow.md` §2's analyzer — is already neomjs/neo-agent-skills#104 (@neo-opus-grace). This ticket is the other seven.

## The Problem

The mirror froze on 2026-08-26 (`git log -- resources/content` on engine `dev@9dcc4ef088`: last commit `e7874db2d2`); the live corpus is `neomjs/github-content-sync`, publishing every few hours (`267fedfd` at 07:46Z today). Seven payloads still send agents to the frozen tree, and every one of those instructions returns a *plausible* result — a grep over four-week-old files exits 0 with hits — so nothing tells the agent the sweep is blind to a month of tickets and discussions:

| payload | line | instruction today |
|---|---|---|
| `ticket-create/references/ticket-create-workflow.md` | 49–50 | `grep on resources/content/issues/` · `grep on resources/content/discussions/` as the exact/historical sweep |
| `ideation-sandbox/audits/pre-authoring-adjacency-sweep.md` | 17–18 | "Local exact sweep: search `resources/content/discussions/` and `resources/content/issues/`" |
| `epic-review/references/epic-review-workflow.md` | 33 | duplicate sweep: "Check `resources/content/issues/` (active and archived)" |
| `ticket-triage/references/ticket-triage-workflow.md` | 29 | adjacency sweep: "`grep_search` against `resources/content/issues/`" |
| `ticket-intake/references/ticket-intake-workflow.md` | 45 | fallback for exact keyword verification against the local content tree |
| `tech-debt-radar/references/tech-debt-radar-guide.md` | 23 | "`ask_knowledge_base` against the backlog (`resources/content/issues/`)" — names the mirror as the backlog's home |
| `hostile-content-quarantine/references/hostile-content-quarantine-workflow.md` | 45–47 | the ingestion-window check: `ls resources/content/discussions/ \| grep <number>` and `git log … -- resources/content/` to decide whether moderation lands before the next sync |
| `industry-friction-radar/references/industry-friction-radar-workflow.md` | 62–63 | reading pointers to `resources/content/discussions/discussion-10119.md` / `discussion-10137.md` |

The only thing currently catching what these miss is `ticket-create` §1a's mandatory live GitHub sweep — which exists precisely because the local substrates lag.

## The Architectural Reality

- Each of these is an **artifact sweep** in the sense of `ticket-create/references/decision-substrate-sweeps.md`: it answers "does a ticket exist?", and its substrate is the mirror.
- The live substitutes already exist: the GitHub sweep (`gh issue list … sort:created-desc`, `list_issues`, `gh search issues`), the Knowledge Base (`query_documents` / `ask_knowledge_base`, whose conversation rows come from the `github-content-sync` tenant once neomjs/neo-agent-brain#411 activates it), and for the quarantine's window question the corpus publisher's own run log (`gh run list --repo neomjs/github-content-sync`) plus the tenant poller's sweep.
- These are reference payloads, not `SKILL.md` routers — Progressive Disclosure is untouched (`create-skill`); the edits shorten payloads, so the skill-corpus byte budget (`scripts/lint-skill-corpus.mjs`) moves the right way.

## The Fix

One PR over the seven payloads:

1. Replace each mirror grep with the live pair — the GitHub sweep for filed artifacts, `query_documents` / `ask_knowledge_base` for exact and semantic recall — and delete the mirror path.
2. `hostile-content-quarantine` steps 1–3: publication and ingestion are two observations, neither a run timestamp — *published?* is the corpus repository's `dev` tip holding a file for the artifact (a run's input commit and its published commit differ); *ingested?* is the tenant's `lastIngestedRev` against the publishing commit, and where the checkpoint cannot answer (it read `null` beside 260 settled chunks on 2026-09-23) a direct `query_documents` probe — unknown is never "no". A published copy is itself an ingestion source until a run republishes without it, so upstream moderation alone is preventive only while nothing is published; drop the `ls`/`git log` over the mirror. *(Corrected at PR #107 review R1, @neo-gpt-emmy; the first cut compared the artifact's `createdAt` with the newest run.)*
3. `industry-friction-radar`: point at the Discussions themselves (D#10119, D#10137).
4. `tech-debt-radar`: name the backlog as the Knowledge Base's conversation rows, not a directory.
5. Leave `release-notes-workflow.md` to #104.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| the seven reference payloads above | `ticket-create` §1a (live sweep is mandatory), `decision-substrate-sweeps.md` | every duplicate/adjacency/window instruction names a live substrate; no payload names `resources/content` | none — the mirror is deleted by cornerstone 3 | the payloads themselves | AC-1, AC-2 |

## Decision Record impact

`none` — process payloads; `aligned-with` D#17846 §8.5 (consumers read the corpus, not the engine tree).

## Acceptance Criteria

- [ ] **AC-1** `grep -rn "resources/content" .agents/skills` returns only `release-notes-workflow.md` (owned by #104), `update-roadmap-workflow.md` (a MUST-NOT example) and `guide-authoring-bar.md` (a historical measurement).
- [ ] **AC-2** Each replaced instruction names its live substrate and the command or tool that runs it; the quarantine check binds the artifact to the corpus repository's published tip, keeps partial or unknown ingestion distinct from none, and treats a published copy as an ingestion source that needs containment *(re-stated at PR #107 R1; the first cut compared `createdAt` with the newest run and could call published-but-unswept content safe)*.
- [ ] **AC-3** `npm run lint` (the skill-corpus lint) passes and the total payload byte count does not grow.
- [ ] **AC-4** One `ticket-create` dry run against a ticket filed after 2026-08-26 finds it through the replaced sweep — the case the mirror could never return.

## Out of Scope

- `release-notes-workflow.md` and the analyzer's root (#104).
- A freshness lint for content paths in skill payloads (recurrence-gated; the census is the first recurrence).
- The mirror's deletion itself (cornerstone 3's retirement leaf under neomjs/neo#17416).

## Avoided Traps

- ⛔ Pointing the greps at a corpus checkout path: no seat has a declared checkout location, and a path that is right on one machine is a new frozen-mirror class on the next.
- ⛔ Keeping "the local grep is a fallback": a fallback nobody can tell is stale is the defect being fixed.

## Related

neomjs/neo#17416 (epic; this is a consumer-boundary leaf) · neomjs/neo-agent-skills#104 · neomjs/neo-agent-skills#62 (closed-state sweeps — adjacent, not this) · D#17846 §8.5 · neomjs/neo-agent-brain#411

Live latest-open sweep: latest 20 open issues in this repository at 2026-09-23T11:00Z — #104 is the release-notes sibling, excluded above; no other equivalent. A2A in-flight sweep (30 most recent, all read-states, 11:02Z): no claim on skill sweeps; #104's `[ticket-filed + owner]` (09:46Z) confirms the split. Memory Core sweep: `query_raw_memories` returned unrelated `sync_all` memories (recall degraded today). Own-assignment sweep (this repository): #90, #87, #86, #85, #80, #51 — none equivalent. Epic layer: leaf, not an epic. Meta-skill sweep (`create-skill`): reference payloads only, routers untouched. Structure map: N/A — no `.mjs`.

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727
Retrieval Hint: `query_raw_memories("skill payloads frozen mirror resources/content live sweep replacement cornerstone 3")`

Authored by Vega (Fable 5.1, Claude Code) 🌿


## Timeline

- 2026-09-23T11:02:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T11:02:37Z @neo-opus-vega added the `bug` label
- 2026-09-23T11:02:37Z @neo-opus-vega added the `documentation` label
- 2026-09-23T11:02:37Z @neo-opus-vega added the `ai` label
- 2026-09-23T11:02:38Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T11:03:46Z @neo-opus-vega added parent issue #17416
- 2026-09-23T11:29:06Z @neo-opus-grace cross-referenced by PR #106
- 2026-09-23T12:03:33Z @neo-opus-vega cross-referenced by PR #107
- 2026-09-23T12:21:32Z @neo-opus-vega referenced in commit `4dc71b8` - "chore(skills): bump to 0.1.16 (#105)"
- 2026-09-23T12:23:08Z @neo-opus-vega referenced in commit `bf3294e` - "chore(skills): the reusable baseline pins 0.1.16 (#105)"
- 2026-09-23T13:25:56Z @neo-opus-vega referenced in commit `d3d8391` - "fix(skills): the quarantine clock reads publication and ingestion as two observations (#105)"
- 2026-09-23T13:47:39Z @neo-opus-ada cross-referenced by #109
- 2026-09-23T14:22:50Z @tobiu referenced in commit `90d4f0a` - "Merge pull request #107 from neomjs/vega/105-mirror-sweeps

fix(skills): seven payloads sweep the live corpus, not the frozen engine mirror (#105)"
- 2026-09-23T14:22:50Z @tobiu closed this issue
- 2026-09-23T14:47:27Z @neo-opus-vega cross-referenced by #17416

