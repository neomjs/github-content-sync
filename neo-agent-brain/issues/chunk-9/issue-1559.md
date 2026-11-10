---
id: 1559
title: Remove output-only revision/publication and duplicate requestPublish from add_message task inputSchema
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T02:11:44Z'
updatedAt: '2026-11-10T02:39:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1559'
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
closedAt: '2026-11-10T02:39:04Z'
---
# Remove output-only revision/publication and duplicate requestPublish from add_message task inputSchema

## Problem Statement

In [`ai/mcp/server/memory-core/openapi.yaml` (lines 2620–2654)](https://github.com/neomjs/neo-agent-brain/blob/dev/ai/mcp/server/memory-core/openapi.yaml#L2620-L2654), the `add_message` request body schema declares `requestPublish` at the top level (`properties.requestPublish: $ref: '#/components/schemas/RequestPublishPayload'`, matching `.agents/skills/blog-post/references/outbound-post.md §3`), and then inside `properties.task.properties` (`lines 2648–2654`) declares three additional `$ref` properties:

1. `task.properties.requestPublish` (`$ref: '#/components/schemas/RequestPublishPayload'`) — a duplicate advertisement of top-level `properties.requestPublish` (while `properties.task` already sets `additionalProperties: true` at line `2624`).
2. `task.properties.revision` (`$ref: '#/components/schemas/RequestPublishRevision'`) — a server-initialized output/storage record (`MailboxService.mjs:3453-3461`) already declared on `add_message`'s `'200'` response (`openapi.yaml:2692-2693`).
3. `task.properties.publication` (`$ref: '#/components/schemas/RequestPublishPublicationState'`) — a server-initialized output/storage record (`MailboxService.mjs:3463-3473`) already declared on `add_message`'s `'200'` response (`openapi.yaml:2694-2695`).

Worse than duplicating schema bytes, advertising `task.revision` and `task.publication` on `add_message.inputSchema` is actively misleading: `MailboxService.mjs:144` defines `TASK_ENVELOPE_ALLOWED_FIELD_SET = new Set(['id', 'state', 'assignee', 'expiresAt', 'fallback', 'requestPublish'])`, and `MailboxService.mjs:3416-3421` **refuses** any call that passes `task.revision` or `task.publication` with `refused: unknown-field`.

Because `SwaggerParser.validate` dereferences every `$ref` in place (`ToolService.mjs:159`), those three nested `$ref`s under `task.properties` bloat `add_message.inputSchema` from `1,830 B` to `5,718 B` (`-3,888 B` via `SwaggerParser` + `stripSchemaDescriptions` on `memory-core/openapi.yaml`; `3,646 B -> 1,724 B` in Antigravity's projected `add_message.json`; `14,471 B -> 6,657 B` / `-7.8 KB` in `get_mcp_tool_handbook`'s described schema).

Graduated from [`neomjs/neo` Discussion #20680](https://github.com/orgs/neomjs/discussions/20680) (Track 2 — Option G).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ai/mcp/server/memory-core/openapi.yaml` (`paths['/mailbox/messages'].post.requestBody.content['application/json'].schema.properties.task.properties`) | `neomjs/neo#20680` Track 2 Option G | Drop `requestPublish`, `revision`, and `publication` from `add_message`'s advertised `task.properties` input schema | Top-level `requestPublish` remains the advertised input (`openapi.yaml:2620`), and `'200'` response `properties.{revision, publication}` remain unchanged (`openapi.yaml:2686-2689`). `task.requestPublish` is still accepted at runtime under `additionalProperties: true` (`MailboxService.mjs:3415-3425`), and the top-level `requestPublish` payload wins when both are present. `task.revision` and `task.publication` stay refused (`refused: unknown-field`, `MailboxService.mjs:144,3416-3421`). | `ai/mcp/server/memory-core/openapi.yaml` | `test/playwright/unit/ai/services/memory-core/MailboxService.spec.mjs` (`AC-4 & AC-5`) |

## Avoidance / Falsification Receipt

- **What would falsify this change?** If `MailboxService.addMessage` accepted caller-supplied `task.revision` or `task.publication` on admission, or if removing `task.properties.{requestPublish, revision, publication}` broke `task: { state: 'Submitted', requestPublish: payload }` admission.
- **V-B-A verification:**
  - `MailboxService.mjs:144,3416-3421` explicitly rejects `task.revision` and `task.publication` as `refused: unknown-field`.
  - `openapi.yaml:2624` sets `additionalProperties: true` on `properties.task`, and `MailboxService.mjs:3415,3423-3425` continues to accept both top-level `requestPublish` and `task.requestPublish` (top-level wins when both are supplied).
  - `components.schemas.{RequestPublishPayload, RequestPublishRevision, RequestPublishPublicationState}` remain intact in `openapi.yaml:6774-6929` and on `add_message`'s `'200'` response (`openapi.yaml:2692-2695`), satisfying `MailboxService.spec.mjs:8831-8895` (`AC-4 & AC-5`).

## Acceptance Criteria

- [ ] Remove `requestPublish`, `revision`, and `publication` from `paths['/mailbox/messages'].post.requestBody.content['application/json'].schema.properties.task.properties` in `ai/mcp/server/memory-core/openapi.yaml` (`lines 2648–2654`), keeping top-level `properties.requestPublish` (`line 2620`) and response `properties.{revision, publication}` (`lines 2692–2695`).
- [ ] Add unit-test assertions in `test/playwright/unit/ai/services/memory-core/MailboxService.spec.mjs` (`AC-4 & AC-5`) verifying that `addMessageSchema.properties.requestPublish` is defined, `addMessageSchema.properties.task.properties` does not advertise output-only `revision` or `publication` (nor duplicate `requestPublish`), and `MailboxService.addMessage` refuses caller-supplied `task.revision` / `task.publication` with `refused: unknown-field`.
- [ ] Verify `test/playwright/unit/ai/services/memory-core/MailboxService.spec.mjs` and `test/playwright/unit/ai/mcp/schemaCompaction.spec.mjs` pass.

---
Authored by Maryam (Gemini 4 Argon, Antigravity). Session `473e18db-b717-4cc8-99be-5eede1f07c60`. 🧭


## Timeline

- 2026-11-10T02:11:44Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T02:11:45Z @neo-gemini-maryam added the `bug` label
- 2026-11-10T02:11:46Z @neo-gemini-maryam added the `ai` label
- 2026-11-10T02:11:46Z @neo-gemini-maryam added the `performance` label
- 2026-11-10T02:11:46Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-10T02:11:46Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-10T02:14:53Z @neo-gemini-maryam cross-referenced by PR #1561
- 2026-11-10T02:30:28Z @neo-fable-mnemosyne cross-referenced by #1562
- 2026-11-10T02:33:03Z @neo-fable-mnemosyne cross-referenced by #1563
- 2026-11-10T02:39:04Z @tobiu referenced in commit `449f55e` - "fix(mcp): remove output-only revision/publication and duplicate requestPublish from add_message task inputSchema (#1559) (#1561)"
- 2026-11-10T02:39:04Z @tobiu closed this issue

