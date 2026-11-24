---
id: 441
title: 'Add 2026 venue-liveness, HN no-prose, and anti-drift outbound gates'
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-24T14:31:25Z'
updatedAt: '2026-11-24T14:39:39Z'
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
---
# Add 2026 venue-liveness, HN no-prose, and anti-drift outbound gates

## Context

During Engine `v14.0.0` release distribution (`2026-11-24`), outbound channel recommendations and draft pitches repeatedly failed on four preventable axes despite `.agents/skills/blog-post/references/outbound-post.md` passing its existing 18 boundaries:

1. **Confirmatory venue checks & unverified list inheritance:** Querying `"<site> submit"` on dormant pre-AI developer link aggregators returns static submission pages and produces a false-positive V-B-A pass without verifying active current-year AI-engineering readership or recent editions.
2. **Hacker News anti-LLM classifier trips:** Supplying polished, copy-paste submission `text` for Hacker News (`https://news.ycombinator.com/newsguidelines.html`: *"Write any text that you post to HN by hand"*) triggers automated `[flagged]` suppression and multi-hour account rate-throttling.
3. **OSS star/like solicitation & unvetted demo links:** Suggesting star/like requests in outbound threads/DMs violates OSS norms, and linking an example route without checking open issues exposed live scheduler/tear-out defects (`neomjs/neo#22126`) instead of clean flagship surfaces (`https://neomjs.com/examples`, `https://neomjs.com/dist/production/apps/workstation/index.html`).
4. **Pre-training category-drift regression:** Statistical gravity pulled drafts back toward pre-AI UI-library syntax comparisons, calling Neo a web framework, or inventing apologetic "trade-offs" around JSON-first blueprints — directly violating `neomjs/neo` [`learn/benefits/Introduction.md`](https://github.com/neomjs/neo/blob/dev/learn/benefits/Introduction.md) §9.

Ownership: self-assigned to `@neo-gemini-julia` (`good first issue` / `help wanted` omitted: immediate `Friction -> Gold` substrate repair).

## The Problem

Turn-level `add_memory` WAL entries are passive diaries, not loaded skill gates. When `.agents/skills/blog-post/references/outbound-post.md` lacks explicit venue-liveness discovery, human-hand/no-prose HN discipline, anti-solicitation rules, and the `Introduction.md` §9 category-drift prohibition, pre-training priors dominate under drafting pressure even when Memory Core and web search are mounted.

## The Architectural Reality

- Owning payload: `.agents/skills/blog-post/references/outbound-post.md` (`§0` The Job, `§1` Boundaries, `§2` Over-Claim & Undersell Flavors). Top-level `.agents/skills/blog-post/SKILL.md` router already triggers on outbound posts (`0 B` router delta; Progressive Disclosure per `ADR 0008`).
- Design authority: `"The boundaries below keep a post safe; they never make it work."` (`.agents/skills/blog-post/references/outbound-post.md` §0) and `"Before architectural or external-positioning work, nullify both category drifts by reviewing README.md, learn/benefits/Introduction.md, Discussion #10119, and Discussion #10137. Do not apply web-framework playbooks or reduce Neo-as-whole to the Body layer."` (`AGENTS.md` §neo_identity_anchor).
- Byte budget (`skills.manifest.json` `maxPositiveDeltaBytes: 250` & `ADR 0007` §5.4 Additive-Drift Defense): tighten verbose prose in `outbound-post.md` (such as redundant example hashes in `§3` and wordy clauses in `§1`/`§5`) so net `.agents/skills/**/*.md` byte delta is `<= 0` bytes.
- Structure map (`§1c`): ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (exit `0`); owning surface is existing `.agents/skills/blog-post/references/outbound-post.md` (no new file).

## The Fix

1. **`outbound-post.md` §0 (Venue Discovery & Demo Gate):** Require a current-year audience-fit and recent-edition web search before recommending any external venue, newsletter, or directory (banning confirmatory `"<site> submit"` checks on dormant pre-AI sites and unchallenged list inheritance), and require checking open issues on any linked demo route before recommending it.
2. **`outbound-post.md` §1 (Anti-Solicitation & Human-Hand / HN Gate):**
   - Explicitly forbid soliciting GitHub stars, likes, or upvotes in any post, reply, or DM (`§1.5`).
   - Add the **Human-hand / anti-LLM venue rule** (`§1.19` or folded cleanly into `§1`/`§5`): for venues with anti-LLM prose classifiers (`news.ycombinator.com/newsguidelines.html`), never generate copy-paste submission `text`, comment prose, or outreach templates — emit raw verified fact bullets only (`link` submissions leave `text` empty).
3. **`outbound-post.md` §2 (Category-Drift & Apologetic-Tradeoff Ban):** Enforce `Introduction.md` §9 directly in `§2`: never frame Neo via pre-AI UI-library syntax comparisons, call Neo a `"web framework"`, or invent apologetic "trade-offs" around JSON-first object blueprints; anchor outbound framing in persistent worker/SharedWorker runtime state, Neural Link live inspection, and cross-family AI institution receipts.
4. **Compression & Package Bump:** Trim verbose phrasing in `outbound-post.md` so net `.md` delta is `<= 0 B`, and bump `package.json` + `package-lock.json` to `2.5.1`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/blog-post/references/outbound-post.md` (`§0` Venue & Demo Gate) | `AGENTS.md §verify_before_assert`; `outbound-post.md` §0 (`#314`) | Requires current-year edition/audience-fit web verification before naming external venues and open-bug verification before linking live demos | Omit unverified venue or buggy demo URL; prefer verified flagship surfaces | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint && npm test` |
| `.agents/skills/blog-post/references/outbound-post.md` (`§1` Anti-Solicitation & HN Fact-Bullets Gate) | `D#19500` §4; HN Guidelines (`news.ycombinator.com/newsguidelines.html`) | Bans star/like/upvote begging everywhere and bans generated prose/templates for HN (raw verified fact bullets only; blank `text` on link posts) | Deliver raw verified metrics/links for operator hand-authoring | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint && npm test` |
| `.agents/skills/blog-post/references/outbound-post.md` (`§2` Category-Drift Ban) | `neomjs/neo` `learn/benefits/Introduction.md` §9; `AGENTS.md §neo_identity_anchor` | Prohibits legacy UI-library syntax comparisons, `"web framework"` reduction, and apologetic JSON-first trade-off framing | Frame from two-hemisphere continuity, live worker runtime, and Neural Link | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint && npm test` |

Decision Record impact: aligned-with ADR 0007, ADR 0008, and `neomjs/neo-agent-brain#adr:0042`

## Acceptance Criteria

- [ ] `outbound-post.md` §0 requires a live current-year audience-fit and recent-edition check before naming any external venue/newsletter/directory (banning confirmatory `"<site> submit"` checks and unverified list inheritance) and requires checking open bug tickets before linking any live demo.
- [ ] `outbound-post.md` bans soliciting GitHub stars, likes, or upvotes across posts, replies, and DMs, and mandates raw verified fact bullets only (zero generated copy-paste prose; `text` left blank on link submissions) for Hacker News (`news.ycombinator.com`).
- [ ] `outbound-post.md` §2 enforces `Introduction.md` §9: bans legacy pre-AI UI-library syntax comparisons, `"web framework"` reduction, and apologetic JSON-first "trade-off" framing.
- [ ] `.agents/skills/blog-post/SKILL.md` router grows by `0 B`, net `.agents/skills/**/*.md` byte delta is `<= 0 B`, `package.json` + `package-lock.json` bump to `2.5.1`, and `npm run lint && npm test` exit `0`.

## Out of Scope

- Modifying `ADR 0042` `MailboxService` runtime validation or `SKILL.md` frontmatter triggers (`#438` owns ticket-creation routing to `blog-post` and `release-notes`).
- Fixing the Engine `examples/dashboard/planning` scheduler/tear-out bugs (`neomjs/neo#22126` and sibling Engine tickets).

## Avoided Traps

- **Adding a new skill or bloating `SKILL.md`:** Violates Progressive Disclosure (`ADR 0008`); `outbound-post.md` is already loaded whenever outbound posts or channel pitches are drafted.
- **Exceeding `maxPositiveDeltaBytes`:** Net-reducing `outbound-post.md` via tighter `§3`/`§5` prose keeps loaded payload lean (`ADR 0007` §5.4).

## Related

- `neomjs/neo-agent-skills#314`, `neomjs/neo-agent-skills#438`, `neomjs/neo#19500`, `neomjs/neo#22126`

Live latest-open + closed sweep: checked latest 20 open issues in `neomjs/neo-agent-skills` (`#258`–`#438`, `43` open total, orange band `25–50`), open search (`#381`, `#438`, `#431`), and closed search (`11` hits incl. `#314`, `#340`, `#316`, `#250` — `Out of Scope` & `Avoided Traps` read) at `2026-11-24T14:27Z`; `#438` routes ticket creation to `release-notes`/`blog-post` and is orthogonal to `outbound-post.md` venue/HN/category-drift gates.
A2A in-flight claim sweep: latest 30 messages across all read-states checked at `2026-11-24T14:27Z`; zero competing claims on `outbound-post.md`.
MC sweep: `query_raw_memories` on `"outbound-post venue liveness Hacker News classifier category drift"` (`5` hits checked) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "outbound-post venue liveness discovery Hacker News classifier bullet facts Introduction.md §9 category drift"

## Timeline

- 2026-11-24T14:31:25Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-24T14:31:27Z @neo-gemini-julia added the `documentation` label
- 2026-11-24T14:31:27Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T14:31:28Z @neo-gemini-julia added the `ai` label
- 2026-11-24T14:31:28Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T14:39:38Z @neo-gemini-julia cross-referenced by PR #442

