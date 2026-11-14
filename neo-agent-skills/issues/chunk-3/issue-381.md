---
id: 381
title: Set scheduledFor on dated requestPublish payloads in outbound-post
state: OPEN
labels:
  - documentation
  - enhancement
  - good first issue
  - contributor-experience
  - ai
assignees: []
createdAt: '2026-11-14T04:41:30Z'
updatedAt: '2026-11-14T04:41:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/381'
author: neo-gemini-maryam
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
# Set scheduledFor on dated requestPublish payloads in outbound-post

## Context

`neomjs/neo-agent-skills#332` (shipped in PR `neomjs/neo-agent-skills#333`) added the prose `D-1 18:00 CET` lock rule for dated publications to `.agents/skills/blog-post/references/outbound-post.md` §3 (`line 67`), while leaving mechanical publish-gate enforcement to `neomjs/neo-agent-brain`.

`neomjs/neo-agent-brain#1695` (implemented in PR `neomjs/neo-agent-brain#1696`, amending [ADR 0042: Fleet Manager Publish Gate](https://github.com/neomjs/neo-agent-brain/blob/dev/learn/agentos/decisions/0042-fm-publish-gate.md) §1.2 and §4.2 and `ai/mcp/server/memory-core/openapi.yaml` `RequestPublishPayload.properties.scheduledFor`) adds mechanical `scheduledFor` enforcement across `MailboxService` (`addMessage`, `editPublishRequest`, `attestPublishRevision`) and `publishExecutor` (`refused: content-lock-window` after `D-1 18:00 Europe/Berlin`, plus non-destructive `refused: schedule-not-reached` holds before day `D`).

During review of `neomjs/neo-agent-brain#1696` ([review `5901602753`](https://github.com/neomjs/neo-agent-brain/pull/1696#pullrequestreview-5901602753)), `@neo-opus-vega`'s Cross-Skill Integration Audit surfaced the caller-side gap: `.agents/skills/blog-post/references/outbound-post.md` §3 (`lines 69–120` on `dev` `8ac3306`) carries the `D-1 18:00` rule as prose, yet its canonical `requestPublish` JSON recipe (`lines 69–98`) and `Field & Revision Invariants` (`lines 100–120`) never set or mention `scheduledFor`. Because `scheduledFor` is optional in `RequestPublishPayload` (omitted or `null` for undated routine posts), an agent following `outbound-post.md` verbatim submits dated posts without `scheduledFor`, leaving the mechanical `D-1 18:00 Europe/Berlin` lock and pre-release hold dormant.

- **Live latest-open + closed sweep:** checked latest 30 open issues + total open count (`36` open — orange band; no open `blog-post` / `outbound-post` / `requestPublish` ticket owns this caller recipe) and ran closed search (`outbound-post OR scheduledFor OR requestPublish OR content-one-day-early`, reading `#332` `Out of Scope` and `Avoided Traps`) + `ask_knowledge_base(type: 'ticket')` at `2026-11-14T04:40Z`; no equivalent ticket or prior exclusion found (`#332` explicitly deferred publish-gate enforcement to `neomjs/neo-agent-brain`).
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', projection: 'compact', limit: 30})` at `2026-11-14T04:40Z`; no competing `[lane-claim]` or `[lane-intent]` on `outbound-post.md` or `scheduledFor`.
- **MC sweep:** `query_raw_memories({query: 'outbound-post requestPublish recipe never sets scheduledFor dated post lock dormant', detail: 'summary', nResults: 5})`, `5` results, no prior exclusion found.
- **Own-assignment sweep:** `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-skills`, none overlapping.
- **Meta-skill, Turn-Memory Pre-Flight & Structure-Map sweep (§1b, §1c):** Read `.agents/skills/create-skill/SKILL.md` (`references/skill-authoring-guide.md`), `.agents/skills/turn-memory-pre-flight/SKILL.md` (`references/turn-memory-pre-flight-workflow.md`), and ran `npm run ai:structure-map -- --files --loc` in `neomjs/neo-agent-brain` (`93` folders, `231,845` code LOC). `.agents/skills/blog-post/SKILL.md` (`11` lines, `1,140` B) stays untouched (`0` B router delta); the change lives strictly in the on-demand reference payload `.agents/skills/blog-post/references/outbound-post.md` (`158` lines, `22,768` B against `perFilePayloadBudget: 25000` and `maxPositiveDeltaBytes: 250`) and states caller workflow discipline rather than duplicating MCP schema mechanics (`skill-authoring-guide.md` §2 *Tool Mechanics Live in the Tool Description, Not the Skill*).

## The Problem

1. `.agents/skills/blog-post/references/outbound-post.md` §3 (`line 67`) instructs agents: *"For any dated publication, lock final text, media, and cross-family attestation (`§4`) by 18:00 CET on D-1 before the operator's publish click"*, but the `requestPublish` example payload immediately below (`lines 69–98`) omits `"scheduledFor"`.
2. `Field & Revision Invariants` (`lines 100–120`) lists `schemaVersion`, `channel`, `media`, `hypothesis`, `format` & `inReplyTo`, and `revisionDigest`, without naming `scheduledFor` — so a drafting or reviewing agent has no prompt in the loaded skill reference to populate `scheduledFor` (`'YYYY-MM-DD'` or explicit-offset ISO-8601) on dated posts, omit/`null` it on undated routine posts, or know that `scheduledFor` sits outside the 6-field `revisionDigest` tuple (`{account, channel, format, text, media, inReplyTo: inReplyTo ?? null}`).

## The Architectural Reality

Design authority: "For any dated publication, lock final text, media, and cross-family attestation (`§4`) by 18:00 CET on D-1 before the operator's publish click (urgent incident or security notices follow operator direction):" (`.agents/skills/blog-post/references/outbound-post.md:67`, paired with `neomjs/neo-agent-brain` `learn/agentos/decisions/0042-fm-publish-gate.md` §1.2 and §4.2).

- **Router boundary (`.agents/skills/blog-post/SKILL.md`):** `11` lines (`1,140` B), untouched (`0` B delta).
- **Payload boundary (`.agents/skills/blog-post/references/outbound-post.md`):** `158` lines (`22,768` B on `dev` `8ac3306`; `perFilePayloadBudget: 25000`, `maxPositiveDeltaBytes: 250`). Owned by `neomjs/neo-agent-skills` and materialized into consumer repos on `npm install`.
- **Tool-description vs. skill boundary (`create-skill/references/skill-authoring-guide.md:112–119`):** Exact wire validation rules (calendar-rollover rejection, naive timestamp rejection, `refused: scheduledFor` vs `refused: content-lock-window`) already live in `ai/mcp/server/memory-core/openapi.yaml` (`RequestPublishPayload.properties.scheduledFor`) and `0042-fm-publish-gate.md` §1.2 / §4.2. `outbound-post.md` §3 only needs the **caller-side discipline**:
  1. Include `"scheduledFor": "2026-11-18"` (illustrative `'YYYY-MM-DD'` target date) in the §3 `requestPublish` JSON recipe;
  2. Add a concise `scheduledFor` bullet in `Field & Revision Invariants` (`ADR 0042 §1.2, §1.4, §4.2`) stating that **every dated publication must set `scheduledFor`** (`'YYYY-MM-DD'` or ISO-8601 with `'Z'` / `'±HH:MM'`, evaluated against `Europe/Berlin` `CET`/`CEST` so `MailboxService` enforces the `D-1 18:00 Europe/Berlin` lock and the executor holds pre-approved posts until release time), undated routine posts omit it or set `null`, urgent incident/security notices follow operator direction, and `scheduledFor` stays outside the 6-field `revisionDigest` tuple;
  3. Update §4 (`The Cross-Family Review & Attestation`, item 1) so the reviewer checklist verifies that a dated post sets `scheduledFor` matching its target publication date;
  4. Trim verbose prose in `outbound-post.md` (for example, tightening the two 64-hex hash examples in line 118 or redundant parentheticals) so `npm run lint` (`scripts/lint-skill-corpus.mjs` `perFilePayloadBudget: 25000` and `maxPositiveDeltaBytes: 250`) passes cleanly without needing `[skill-growth-justified: ...]`, and bump `version` in `package.json` + `package-lock.json` (`scripts/check-version-bump.mjs`).

## The Fix

1. **Update `.agents/skills/blog-post/references/outbound-post.md`:**
   - **§3 intro (`line 67`):** Align `"18:00 CET on D-1"` to `"18:00 Europe/Berlin (CET/CEST) on D-1"` and state that every dated publication sets `scheduledFor` on `requestPublish`.
   - **§3 JSON recipe (`lines 69–98`):** Add `"scheduledFor": "2026-11-18"` to the `requestPublish` object (noting in the invariant bullet that `revisionDigest` in the example remains unchanged because `scheduledFor` is outside the 6-field digest tuple).
   - **§3 `Field & Revision Invariants` (`lines 100–120`):** Add a compact **`scheduledFor` (`ADR 0042 §1.2, §1.4, §4.2`)** bullet covering when callers set `'YYYY-MM-DD'` / explicit-offset ISO-8601 vs `null`/omitted, that `scheduledFor` sits outside the 6-field `revisionDigest` tuple, and that `MailboxService` + the Fleet executor enforce the `D-1 18:00 Europe/Berlin` lock and pre-schedule hold (urgent incident or security notices follow operator direction).
   - **§4 item 1 (`lines 128–134`):** Add a clause to check that dated posts carry the matching `scheduledFor` value.
   - **Byte hygiene:** Tighten surrounding prose in `outbound-post.md` so `scripts/lint-skill-corpus.mjs` passes both `perFilePayloadBudget` (`25000`) and `maxPositiveDeltaBytes` (`250`) cleanly.
2. **Bump package version (`package.json` and `package-lock.json`):** Increment patch version per `scripts/check-version-bump.mjs`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/blog-post/references/outbound-post.md` (§3 recipe + invariants, §4 reviewer check) | this ticket; `neomjs/neo-agent-brain#1695` / PR `neomjs/neo-agent-brain#1696`; `0042-fm-publish-gate.md` §1.2, §4.2 | Every dated `requestPublish` sets `scheduledFor` (`'YYYY-MM-DD'` or explicit-offset ISO-8601, `Europe/Berlin` `CET`/`CEST` lock at `D-1 18:00`, outside the 6-field `revisionDigest`); reviewer checks `scheduledFor` on dated posts | Undated routine posts omit `scheduledFor` or set `null`; urgent incident/security notices follow operator direction | `.agents/skills/blog-post/references/outbound-post.md` | `npm run lint` and `npm test` in `neomjs/neo-agent-skills` |

Decision Record impact: `aligned-with` `neomjs/neo#adr:skill-anatomy-and-authoring-contract` and `neomjs/neo-agent-brain#adr:fm-publish-gate` (`0042-fm-publish-gate.md` §1.2, §4.2).

## Acceptance Criteria

- [ ] AC-1 `.agents/skills/blog-post/references/outbound-post.md` §3 includes `"scheduledFor"` in the canonical `requestPublish` JSON recipe and documents `scheduledFor` in `Field & Revision Invariants` (required on dated publications as `'YYYY-MM-DD'` or explicit-offset ISO-8601 under `Europe/Berlin` `CET`/`CEST` `D-1 18:00` lock and pre-release hold; `null`/omitted for undated routine posts; outside the 6-field `revisionDigest`; urgent incident/security notices follow operator direction).
- [ ] AC-2 `.agents/skills/blog-post/references/outbound-post.md` §4 (`What the reviewer checks`) includes verifying that dated posts set `scheduledFor` to the intended publication date/timestamp.
- [ ] AC-3 `.agents/skills/blog-post/SKILL.md` is untouched (`0` B delta), `package.json` and `package-lock.json` bump their version (`scripts/check-version-bump.mjs`), and `npm run lint && npm test` pass cleanly within `lint-skill-corpus.mjs` payload and delta budgets.

## Out of Scope

- Modifying `.agents/skills/blog-post/SKILL.md` router text or `skills.manifest.json` budgets.
- Backend `MailboxService`, OpenAPI schema, or `publishExecutor` implementation (owned by `neomjs/neo-agent-brain#1695` / PR `neomjs/neo-agent-brain#1696`).
- Re-documenting low-level parser/executor error codes or implementation internals already covered by `RequestPublishPayload.properties.scheduledFor` in `neomjs/neo-agent-brain`.

## Avoided Traps

- **Adding `scheduledFor` to the `revisionDigest` tuple description:** `scheduledFor` is intentionally excluded from the 6-field RFC 8785 tuple (`{account, channel, format, text, media, inReplyTo: inReplyTo ?? null}`) so re-dating a post without changing its content preserves existing digests (`0042-fm-publish-gate.md` §1.2, §1.4).
- **Hardcoding `"CET"` without `"Europe/Berlin (CET/CEST)"`:** Summer publications observe `CEST` (`UTC+2`), and the publish gate evaluates `D-1 18:00` in `Europe/Berlin`.
- **Bloating `SKILL.md` or breaching `perFilePayloadBudget` / `maxPositiveDeltaBytes`:** Keep `.agents/skills/blog-post/SKILL.md` untouched and trim verbose example prose in `outbound-post.md` so `npm run lint` passes within existing corpus budgets (`250` B positive delta cap).
- **Forgetting the version bump:** `neomjs/neo-agent-skills` CI (`scripts/check-version-bump.mjs`) requires bumping `version` in both `package.json` and `package-lock.json`.

## Related

- `neomjs/neo-agent-brain#1695` / PR `neomjs/neo-agent-brain#1696` (mechanical `scheduledFor` lock and pre-release hold in `MailboxService` and `publishExecutor`)
- `neomjs/neo-agent-skills#332` / PR `neomjs/neo-agent-skills#333` (initial prose `D-1 18:00` rule in `blog-post`)
- `neomjs/neo-agent-skills#168` (`outbound-post.md` introduction)

unowned-rationale: Ready for a contributor; maintainers provide planning and review (`@neo-gemini-maryam` will review the pull request it draws).

Origin Session ID: 5e41d607-04fd-4b94-af3b-27313b02d063
Retrieval Hint: "outbound-post.md requestPublish recipe scheduledFor dated publication D-1 18:00 Europe/Berlin lock"

Authored by Maryam (Gemini 4 Argon, Antigravity). Session 5e41d607-04fd-4b94-af3b-27313b02d063. 🧭

## Timeline

- 2026-11-14T04:41:32Z @neo-gemini-maryam added the `documentation` label
- 2026-11-14T04:41:32Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-14T04:41:32Z @neo-gemini-maryam added the `good first issue` label
- 2026-11-14T04:41:33Z @neo-gemini-maryam added the `contributor-experience` label
- 2026-11-14T04:41:33Z @neo-gemini-maryam added the `ai` label
- 2026-11-14T04:48:40Z @neo-gemini-maryam cross-referenced by PR #1696
- 2026-11-14T09:43:55Z @github-actions cross-referenced by #50

