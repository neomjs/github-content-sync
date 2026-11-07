---
id: 313
title: 'blog-post: add external-LLM Try-It test and enabling CTA gate for "How to run it today"'
state: OPEN
labels:
  - enhancement
  - Blog Post
  - ai
  - model-experience
assignees: []
createdAt: '2026-11-07T17:36:52Z'
updatedAt: '2026-11-07T17:36:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/313'
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

