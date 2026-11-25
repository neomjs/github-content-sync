---
id: 441
title: 'Add 2026 venue-liveness, HN no-prose, and anti-drift outbound gates'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-24T14:31:25Z'
updatedAt: '2026-11-24T20:13:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/441'
author: neo-gemini-julia
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
closedAt: '2026-11-24T20:13:27Z'
---
# Add 2026 venue-liveness, HN no-prose, and anti-drift outbound gates

## Context

During Engine `v14.0.0` release distribution (`2026-11-24`), outbound channel recommendations and draft pitches repeatedly failed on four preventable axes despite `.agents/skills/blog-post/references/outbound-post.md` passing its existing 18 boundaries:

1. **Confirmatory venue checks, unverified list inheritance, & unreachable discovery routing:** Querying `"<site> submit"` on dormant pre-AI developer link aggregators returns static submission pages and produces a false-positive V-B-A pass without verifying active current-year AI-engineering readership, recent dated editions (for newsletters), or recent activity/accepted listings (for directories and communities) — and `.agents/skills/blog-post/SKILL.md` did not route standalone venue/directory discovery requests into `outbound-post.md`.
2. **Hacker News human-authorship violations:** Supplying drafted copy-paste titles, submission `text`, or comments for Hacker News violates HN's published guidelines (`https://news.ycombinator.com/newsguidelines.html`, forbidding generated comments or other generated/AI-edited content).
3. **OSS star/like solicitation & untested demo gestures:** Suggesting star/like requests in outbound threads/DMs violates OSS norms, and linking a live route without exercising the specific invited reader gesture at the deployed version and searching open issues exposed live defects (`neomjs/neo#22125` on Workstation native popup close; `neomjs/neo#22126` on planning scheduler).
4. **Pre-training category-drift regression:** Statistical gravity pulled drafts back toward pre-AI UI-library syntax comparisons, calling Neo a web framework, or inventing apologetic "trade-offs" around JSON-first blueprints — directly violating `neomjs/neo` [`learn/benefits/Introduction.md`](https://github.com/neomjs/neo/blob/dev/learn/benefits/Introduction.md) §9.

Ownership: self-assigned to `@neo-gemini-julia` (`good first issue` / `help wanted` omitted: immediate `Friction -> Gold` substrate repair).

## The Problem

Turn-level `add_memory` WAL entries are passive diaries, not loaded skill gates. When `.agents/skills/blog-post/SKILL.md` does not route standalone venue-discovery requests to `outbound-post.md`, and `outbound-post.md` lacks explicit recommendation-time venue/gesture verification, HN human-authorship raw-fact-brief discipline, manual-venue vs. Fleet `requestPublish` handoff separation, and the `Introduction.md` §9 category-drift prohibition, pre-training priors dominate under drafting pressure.

## The Architectural Reality

- Owning surfaces: `.agents/skills/blog-post/SKILL.md` + `skills.manifest.json` (reworded within the `<= 220` byte description and `<= 12` line router budgets so standalone venue/directory discovery loads `outbound-post.md`) and `.agents/skills/blog-post/references/outbound-post.md` (`§0` Gate A/B, `§1`, `§2`, `§3.1–3.2`, `§4.1–4.3`).
- Design authority: `"The boundaries below keep a post safe; they never make it work."` (`.agents/skills/blog-post/references/outbound-post.md` §0), `ADR 0042` (`requestPublish` scoped to `'x'` and `'linkedin'`), HN guidelines (`news.ycombinator.com/newsguidelines.html`), and `AGENTS.md §neo_identity_anchor`.
- Byte budget (`skills.manifest.json` `maxPositiveDeltaBytes: 250` & `ADR 0007` §5.4 Additive-Drift Defense): tighten verbose prose in `outbound-post.md` so net `.agents/skills/**/*.md` byte delta is `<= 0` bytes even with the `SKILL.md` router alignment.
- Structure map (`§1c`): ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (exit `0`); owning surfaces are existing `.agents/skills/blog-post/SKILL.md` and `.agents/skills/blog-post/references/outbound-post.md`.

## The Fix

1. **Router & `outbound-post.md` §0 (Reachable Venue Discovery & Deployed Demo Gesture Gate):** Align `.agents/skills/blog-post/SKILL.md` (+ `skills.manifest.json`) within router budgets to route standalone venue/directory discovery requests to `outbound-post.md`. Split `§0` into **Gate A** (runs on standalone venue discovery AND before drafting: live current-year audience-fit + dated edition check for newsletters or recent activity/accepted listings for directories/communities; plus deployed URL + invited reader gesture execution + open-issue search, recording action and result) and **Gate B** (copy-specific fills applied when drafting).
2. **`outbound-post.md` §§1, 3–4 (Anti-Solicitation, HN Human-Authorship Brief, & Manual-Venue Handoff):**
   - Forbid soliciting GitHub stars, likes, or upvotes in any post, reply, thread, or DM (`§1.5`).
   - Enforce HN's published human-authorship rules (`news.ycombinator.com/newsguidelines.html`) across final title, submission `text`, and comments (`§1.19`): hand the operator a raw verified fact brief only (`link` submissions leave `text` blank; normal link posts omit `Show HN:`).
   - Separate Fleet-supported `requestPublish` channels (`x`, `linkedin` with `revisionDigest` and gate switches in `§3.1` / `§4.2 Path A`) from manual/human-published venues (Medium, newsletters, directories, HN in `§3.2` / `§4.3 Path B`).
3. **`outbound-post.md` §2 (Category-Drift & Apologetic-Tradeoff Ban):** Enforce `Introduction.md` §9 directly in `§2`: never frame Neo via pre-AI UI-library syntax comparisons, call Neo a `"web framework"`, or invent apologetic "trade-offs" around JSON-first object blueprints.
4. **Compression & Package Bump:** Keep net `.agents/skills/**/*.md` byte delta `<= 0 B`, and bump `package.json` + `package-lock.json` (+ `plugins/neo` manifest sync) to `2.6.1`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/blog-post/SKILL.md` & `outbound-post.md` (`§0` Gate A/B) | `AGENTS.md §verify_before_assert`; `ADR 0008`; `outbound-post.md` §0 (`#314`) | Routes standalone venue discovery within router budgets; checks dated editions (newsletters) or recent activity/listings (directories/communities) and exercises exact deployed demo URL + invited reader gesture + open-defect search | Omit unverified venue or route with an open defect on the invited gesture | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint && npm test` + editorial walkthroughs |
| `outbound-post.md` (`§§1, 3–4` HN Brief & Manual-Venue Handoff) | `D#19500` §4; `ADR 0042 §1.2`; HN Guidelines (`news.ycombinator.com/newsguidelines.html`) | Bans star/like/upvote begging; hands raw verified fact brief only for HN (title, `text`, comments human-written); separates manual-venue review/handoff packets (`§3.2`, `§4.3 Path B`) from Fleet `requestPublish`/`revisionDigest` fields (`§3.1`, `§4.2 Path A`) | Hand verified manual-venue packet directly to operator | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint && npm test` + editorial walkthroughs |
| `outbound-post.md` (`§2` Category-Drift Ban) | `neomjs/neo` `learn/benefits/Introduction.md` §9; `AGENTS.md §neo_identity_anchor` | Prohibits legacy UI-library syntax comparisons, `"web framework"` reduction, and apologetic JSON-first trade-off framing | Frame from two-hemisphere continuity, live worker runtime, and Neural Link | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint && npm test` + editorial walkthrough |

Decision Record impact: aligned-with ADR 0007, ADR 0008, and `neomjs/neo-agent-brain#adr:0042`

## Acceptance Criteria

- [ ] `.agents/skills/blog-post/SKILL.md` (+ `skills.manifest.json`) routes standalone venue/directory discovery within router budgets, and `outbound-post.md` §0 separates Gate A (dated editions for newsletters, recent activity/accepted listings for directories/communities, plus deployed demo URL + invited gesture check + open-issue search with recorded result) from Gate B copy fills.
- [ ] `outbound-post.md` bans soliciting GitHub stars, likes, or upvotes across posts, replies, and DMs (`§1.5`), enforces HN's published human-authorship rule across final title, submission `text`, and comments via a raw verified fact brief (`§1.19`), and separates manual-venue review/operator handoff (`§3.2`, `§4.3 Path B`) from Fleet `requestPublish`/`revisionDigest` fields (`§3.1`, `§4.2 Path A`).
- [ ] `outbound-post.md` §2 enforces `Introduction.md` §9: bans legacy pre-AI UI-library syntax comparisons, `"web framework"` reduction, and apologetic JSON-first "trade-off" framing.
- [ ] `.agents/skills/blog-post/SKILL.md` stays within router line/description budgets, net `.agents/skills/**/*.md` byte delta is `<= 0 B`, `package.json` + `package-lock.json` (+ `plugins/neo` manifest) bump to `2.6.1`, and `npm run lint && npm test` exit `0`.

## Out of Scope

- Modifying `ADR 0042` `MailboxService` runtime validation or expanding `requestPublish` beyond `'x'` and `'linkedin'`.
- Fixing the Engine `apps/workstation` or `examples/dashboard/planning` defects (`neomjs/neo#22125`, `neomjs/neo#22126`).

## Avoided Traps

- **Leaving standalone venue discovery unrouted in `SKILL.md` or coupling Gate A to Gate B copy fills:** Rewording `SKILL.md` within its `12`-line / `220`-byte cap and splitting Gate A from Gate B makes recommendation-only discovery work cleanly.
- **Applying Fleet `revisionDigest` / gate switches to manual venues or asserting unverified HN internal classifier mechanics:** `ADR 0042` scopes `requestPublish` to `'x'`/`'linkedin'`, and HN's published human-authorship rule is sufficient authority on its own.

## Related

- `neomjs/neo-agent-skills#314`, `neomjs/neo-agent-skills#438`, `neomjs/neo#19500`, `neomjs/neo#22125`, `neomjs/neo#22126`

Live latest-open + closed sweep: checked latest 20 open issues in `neomjs/neo-agent-skills` (`#258`–`#438`, `43` open total, orange band `25–50`), open search (`#381`, `#438`, `#431`), and closed search (`11` hits incl. `#314`, `#340`, `#316`, `#250` — `Out of Scope` & `Avoided Traps` read) at `2026-11-24T14:27Z`; `#438` routes ticket creation to `release-notes`/`blog-post` and is orthogonal to `outbound-post.md` venue/HN/category-drift gates.
A2A in-flight claim sweep: latest 30 messages across all read-states checked at `2026-11-24T14:27Z`; zero competing claims on `outbound-post.md`.
MC sweep: `query_raw_memories` on `"outbound-post venue liveness Hacker News classifier category drift"` (`5` hits checked) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "outbound-post venue liveness discovery Hacker News human authorship fact brief Introduction.md §9 category drift"

## Timeline

- 2026-11-24T14:31:25Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-24T14:31:27Z @neo-gemini-julia added the `documentation` label
- 2026-11-24T14:31:27Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T14:31:28Z @neo-gemini-julia added the `ai` label
- 2026-11-24T14:31:28Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T14:39:38Z @neo-gemini-julia cross-referenced by PR #442
- 2026-11-24T19:40:45Z @neo-gemini-julia referenced in commit `b55e5e1` - "docs(blog-post): route venue discovery and separate manual venue handoffs (#441)"
- 2026-11-24T20:02:06Z @neo-gemini-julia referenced in commit `1bd43bd` - "docs(blog-post): align manual-venue section refs and HN paraphrase (#441)"
- 2026-11-24T20:02:06Z @neo-gemini-julia referenced in commit `f21bc5d` - "chore(release): merge dev and bump version to 2.6.1 (#441)"
- 2026-11-24T20:13:27Z @tobiu referenced in commit `80633df` - "docs(blog-post): add 2026 venue-liveness, HN no-prose, and anti-drift gates (#441) (#442)

* docs(blog-post): add 2026 venue-liveness, HN no-prose, and anti-drift gates (#441)

* docs(blog-post): route venue discovery and separate manual venue handoffs (#441)

* docs(blog-post): align manual-venue section refs and HN paraphrase (#441)"
- 2026-11-24T20:13:28Z @tobiu closed this issue

