---
id: 313
title: 'blog-post: add external-LLM Try-It test and enabling CTA gate for "How to run it today"'
state: CLOSED
labels:
  - enhancement
  - Blog Post
  - ai
  - model-experience
assignees: []
createdAt: '2026-11-07T17:36:52Z'
updatedAt: '2026-11-08T02:41:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/313'
author: neo-gemini-julia
commentsCount: 1
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
closedAt: '2026-11-08T02:41:52Z'
---
# blog-post: add external-LLM Try-It test and enabling CTA gate for "How to run it today"

## Context

While reviewing Part 2 of the *How an AI team remembers* blog series (`neomjs/neo#20563`, `learn/blog/context-compaction-recovery-for-ai-coding-agents.md`), the operator asked us to run a direct reader/LLM adoption test:
> *"you, as a LLM => would you be excited to try it out after reading the blog post (assuming you don't have our turn based memory and agent os)? does the blog guide you on HOW to try it out?"*

Both Part 1 (`learn/blog/persistent-memory-for-ai-coding-agents.md`) and Part 2 (`learn/blog/context-compaction-recovery-for-ai-coding-agents.md`) passed our existing `blog-post` gates (`neo-agent-skills#262` claim verification, flavors 1–5, cross-family review), yet both ended their `## How to run it today` section with copy that unintentionally **discouraged** interested readers rather than enabling them:
- Part 1 (`persistent-memory-for-ai-coding-agents.md` line 121): *"There is no package and no one-command install for it today."*
- Part 2 (`context-compaction-recovery-for-ai-coding-agents.md` line 122): *"Two of its checks need nothing from us... The checks of the mailbox, the handover and the recent turns need a store that keeps them."*

## The Problem

`.agents/skills/blog-post/references/blog-authoring-guide.md` §1 currently defines the CTA in one line:
> `- **CTA** — the one question the piece leaves the reader holding, plus a single concrete next step.`

And `neo-agent-skills#250` adds the stranger *click* test at the **top** of the funnel (title, subtitle, first paragraph). Neither gate checks the **bottom** of the funnel (`## How to run it today` / adoption CTA):
1. An external engineer or LLM who gets excited by the post's thesis reaches `## How to run it today` wanting to **try it right now** and **adapt it to their own needs**.
2. Leading that section with what *cannot* be run without our internal stack (or omitting copy-pasteable install/setup steps and the path to `neomjs/neo-agent-skills`, `neomjs/neo-agent-brain`, and `neomjs/neo-agent-institution` Fleet Manager, which is close to v1 and used daily by our team) turns a high-intent reader away at the finish line.

## The Architectural Reality

- Design authority: `"CTA — the one question the piece leaves the reader holding, plus a single concrete next step."` (`.agents/skills/blog-post/references/blog-authoring-guide.md` line 16) and `"The review is the product — it is what makes the post trustworthy, and it is the thesis in motion."` (line 48).
- `.agents/skills/blog-post/SKILL.md` (`8` lines) routes to `.agents/skills/blog-post/references/blog-authoring-guide.md` (`59` lines, `4,931` B).
- Structure-map check (`ticket-create` §1c): ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (exit 0); `.agents/skills/blog-post/references/blog-authoring-guide.md` in `neomjs/neo-agent-skills` owns blog authoring and review gates.

## The Fix

Update `.agents/skills/blog-post/references/blog-authoring-guide.md` (§1 CTA and §4 Cross-Family Review Bar):
1. **Expand `CTA / How to try it today` in §1 with the External Reader / LLM Try-It Test:**
   - Every technical blog post's closing `## How to run it today` / CTA section must pass two questions from the perspective of an external reader or LLM **without** our internal setup pre-installed:
     1. *Would I be excited to try this out right now, and can I easily adapt it to my own stack?*
     2. *Does the section guide me step-by-step on HOW to try it today?*
   - Specifically, when a post showcases an Agent OS / workflow capability, enable two progressive adoption paths rather than framing prerequisites as blockers:
     - **Immediate / adaptable path:** Show how the reader's agent can run or adapt the skill/pattern immediately (e.g., `npm install --save-dev neo-agent-skills && npx neo-agent-skills-materialize`, or adapting the plain-Markdown runbook in `.agents/skills/` to their own git/GitHub/local files).
     - **Full stack path:** Point interested readers clearly to [`neomjs/neo-agent-brain`](https://github.com/neomjs/neo-agent-brain) (Memory Core & Knowledge Base MCP stack) and [`neomjs/neo-agent-institution`](https://github.com/neomjs/neo-agent-institution) (Fleet Manager — our mission-control cockpit for operating a standing multi-model AI team every day).
   - **Ban the "anti-CTA" flavor:** Never lead `## How to run it today` with what does *not* work for the reader (`"only 2 of 6 checks work without our store"`, `"there is no package today"`). State what works now, how to run/customize it, and where the full stack lives.
2. **Add the Try-It check to §4 (Cross-Family Review Bar):**
   - Alongside the title/preview cold read (`#250`) and the final-version fact check (`#262`), the cross-family reviewer explicitly runs the external-LLM "Try-It" test on the closing CTA section.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/blog-post/references/blog-authoring-guide.md` §1 & §4 | this ticket; operator feedback on `neomjs/neo#20563` (2026-11-07) | Codifies the External Reader / LLM Try-It Test, progressive adoption paths (`neo-agent-skills` -> `neo-agent-brain` + `neo-agent-institution`), and bans anti-CTA framing | Pure engine/UI posts map the same two-question test to their runnable example/demo + repo quickstart | `blog-authoring-guide.md` | `npm run lint` and `npm test` in `neo-agent-skills` |

Decision Record impact: `aligned-with` ADR 0008 (payload-only update in `references/blog-authoring-guide.md`; no `SKILL.md` router change).

## Acceptance Criteria

- [ ] AC-1 `.agents/skills/blog-post/references/blog-authoring-guide.md` §1 and §4 codify the External Reader / LLM "Try-It" Test, the progressive adoption guidance (`neo-agent-skills`, `neo-agent-brain`, `neo-agent-institution`), and the ban on discouraging anti-CTA framing.
- [ ] AC-2 `npm run lint` and `npm test` pass in `neomjs/neo-agent-skills` (within the skill byte-budget guard or with justified compaction), and `package.json` version is bumped per repository release rules.

## Out of Scope

- Modifying `SKILL.md` router frontmatter.
- Rewriting historical blog posts outside the active *How an AI team remembers* series (`neomjs/neo#20563`).

## Avoided Traps

- Treating `## How to run it today` as a disclaimer block: honesty about architecture is compatible with an enabling, step-by-step quickstart that invites readers to try and customize the workflow.

## Related

- `neomjs/neo#20562` / `neomjs/neo#20563` (Part 2 blog post where this test surfaced the gap)
- `neomjs/neo-agent-skills#250` (top-of-funnel stranger title/click test; this ticket is its bottom-of-funnel adoption counterpart)
- `neomjs/neo-agent-skills#262` (claim-class verification and final-version fact check)

unowned-rationale: Filed from the `neomjs/neo#20563` review cycle so any maintainer seat can claim and ship alongside `#250`.

Live latest-open sweep: checked latest 30 open issues in `neomjs/neo-agent-skills` at `2026-11-07T17:33:55Z` (`#289`, `#282`, `#258`, `#256`, `#250`, `#248`, etc.); `#250` covers title/click traction at the top of the post, while no open issue covers the closing `How to run it today` / external-LLM Try-It CTA test.
A2A in-flight claim sweep: checked latest 30 messages via `list_messages({status: 'all', limit: 30})` at `2026-11-07T17:33:09Z`; no competing claim on this scope.
Memory Core rationale sweep: `query_raw_memories` for `"context-recovery standalone fallback blog-post How to run it today CTA neo-agent-institution"` and `ask_knowledge_base` returned no prior ticket on the Try-It CTA gate.
Own-assignment sweep: `list_issues({repo: 'neomjs/neo-agent-skills', state: 'open', assignee: 'neo-gemini-julia'})` returned 0 open issues.
Meta-skill sweep (§1b): Read `.agents/skills/create-skill/SKILL.md` and `references/skill-authoring-guide.md`; router unchanged, payload stays well under cap.

Origin Session ID: c5b059c4-5d96-46f1-8c61-184af74d34b4
Retrieval Hint: "blog-post How to run it today external LLM try-it test anti-CTA neo-agent-institution #20563"

## Timeline

- 2026-11-07T17:36:53Z @neo-gemini-julia added the `enhancement` label
- 2026-11-07T17:36:53Z @neo-gemini-julia added the `Blog Post` label
- 2026-11-07T17:36:54Z @neo-gemini-julia added the `ai` label
- 2026-11-07T17:36:54Z @neo-gemini-julia added the `model-experience` label
- 2026-11-08T02:26:46Z @neo-opus-grace cross-referenced by #314
- 2026-11-08T02:39:02Z @neo-opus-grace referenced in commit `f599406` - "feat(blog-post): the closing section passes the Try-It test, one reviewer reads as a stranger (#314)

Folds #313 in with its author's consent: blog-authoring-guide.md §1 gains the Try-It test for the closing How-to-run-it-today section (what runs today first, the progressive path, no anti-CTA opening); §4 has one reviewer run it alongside outbound-post.md §0's cold-reader check."
- 2026-11-08T02:39:03Z @neo-opus-grace cross-referenced by PR #315
### @neo-gemini-julia - 2026-11-08T02:41:51Z

Folded into neomjs/neo-agent-skills#314 (AC-5) and implemented in PR neomjs/neo-agent-skills#315 at f59940690e72b414ce03b8e8e66426f5de7c2380.

- 2026-11-08T02:41:52Z @neo-gemini-julia closed this issue
- 2026-11-08T03:14:46Z @tobiu referenced in commit `50577d9` - "feat(blog-post): an outbound post names the reader's next action and keeps its link in view (#314) (#315)

* feat(blog-post): an outbound post names the reader's next action and keeps its link in view (#314)

outbound-post.md gains §0 The Job: six fills before drafting (the action and
its number, the link in view, a trailer not the article, the level in line
one, end on the action, a reply links where its answer continues). §4.1 reads
§0 as a cold reader; §5.1 scopes link-in-first-reply to cold X accounts.
blog-authoring-guide.md §1: the CTA is the reader's next action with its link.

[skill-growth-justified: the outbound gate had no slot for effect, so posts passed 18/18 boundaries and moved no reader; §0 retires into the requestPublish payload once it carries the action and the Fleet executor records the outcome]

* feat(blog-post): the closing section passes the Try-It test, one reviewer reads as a stranger (#314)

Folds #313 in with its author's consent: blog-authoring-guide.md §1 gains the Try-It test for the closing How-to-run-it-today section (what runs today first, the progressive path, no anti-CTA opening); §4 has one reviewer run it alongside outbound-post.md §0's cold-reader check.

* fix(blog-post): the reviewer's link-reply check is scoped to cold X accounts, like §5.1 (#314)

Julia's review note: §4.1 still checked link-in-first-reply unscoped after §5.1 was scoped.

* feat(blog-post): the level line names its source, and the outbound is drafted with the post (#314)

§0.4 points the level line at learn/benefits/Introduction.md (accountable AI peers: a name, a memory, rival-lab review, the right to refuse); §1's pre-flight drafts the outbound's venues and §0 fills with the post and reviews them in the same PR.

* fix(blog-post): the level line links the Introduction by its canonical URL and the census records it (#314)"

