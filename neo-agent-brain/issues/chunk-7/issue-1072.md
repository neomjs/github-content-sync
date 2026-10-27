---
id: 1072
title: Neural Link results deliver declared images as MCP image content
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-10-16T23:10:37Z'
updatedAt: '2026-10-17T00:25:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1072'
author: neo-opus-grace
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
closedAt: '2026-10-17T00:18:17Z'
---
# Neural Link results deliver declared images as MCP image content

## Context

On 2026-10-16 I drove the private Stage avatar lab (Neo's own WebGL2 engine) only through Neural Link MCP tools, the way the engine's contract tells a model to: read the scene, write one envelope, wait for the draw receipt, look at the frame. The last step failed on our side of the wire. One `waitForSceneDraw` with inspection returned **6,132,012 characters** of JSON text for one 1024×561 frame. Most of that is two RGBA8 artifacts as base64. Claude Code could not show the result and wrote it to a file. Seeing the frame took 7 tool calls plus a one-off decoder script (base64, then a bottom-to-top row flip, then a PNG encode). The same frame as a PNG is 103,481 bytes.

## The Problem

The Neural Link is the Brain's channel into the Body, and it cannot show a model a single pixel: every result is JSON text. A model that renders a scene cannot look at what it rendered, so it cannot close its own loop (change → look → judge → change) without writing tools first. The client side is not the gap. Claude Code already renders MCP `image` content to the model: the browser pane's screenshots arrive that way.

## The Architectural Reality

- `ai/mcp/server/BaseServer.mjs` `formatToolResult` (`dev` fa34303e, line 520) turns every object result into one pretty-printed `{type: 'text'}` block plus `structuredContent`. No Brain MCP server emits `image` content: a `mimeType` grep over `ai/mcp` finds nothing.
- `ai/mcp/server/neural-link/Server.mjs` extends `BaseServer` and does not override `formatToolResult`. `call_method` (`toolService.mjs:23` → `InstanceService.callMethod`) returns the App-side method's value as is.
- `@modelcontextprotocol/sdk` `^1.32.1`. `ImageContent` `{type: 'image', data, mimeType}` is part of the protocol.

## The Fix

1. Add a pure helper, `ai/mcp/server/neural-link/imageContent.mjs`, with `liftImageArtifacts(result)`. It walks plain objects and arrays once, as `JSON.stringify` already does. When an object has `mimeType` `image/png|jpeg|webp|gif` and a non-empty base64 `data` string, its `data` becomes an image block, and the JSON keeps `{...artifact, contentIndex: n}` in its place. The walk is copy-on-write: a result with no artifact comes back as the same object.
2. Override `Server.formatToolResult(result)`: lift the artifacts, pass the presented result to `super.formatToolResult`, then append the image blocks after the text block. Error results go through unchanged.

Pre-Flight (structural fast-path): `imageContent.mjs` follows the sibling `verifyBridgeToken.mjs`: a pure helper the server consumes, with its own spec under `test/playwright/unit/ai/mcp/server/neural-link/`. There is no novel directory choice. Structure map (`npm run ai:structure-map -- --files --loc`): `ai/mcp/server/neural-link` has 14 files, and `Server.mjs` is 92 code LOC.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Neural Link tool result envelope (`call_method` and every other NL tool) | MCP `ImageContent`; `BaseServer.formatToolResult` | An object with an image `mimeType` and base64 `data` becomes `content[n]` `{type: 'image', data, mimeType}`. The text JSON and `structuredContent` keep `contentIndex: n` and drop `data`. | No such object: the envelope is identical to today's | helper + override JSDoc | unit spec; live witness with the Stage companion |

## Decision Record impact

None. No ADR governs MCP result content types. The change is additive, opt-in by result shape, and limited to the Neural Link server: `BaseServer` and the other servers are untouched.

## Acceptance Criteria

- [x] A result with no image artifact formats exactly as `BaseServer.formatToolResult` formats it, and the walk returns the same object.
- [x] A nested `{mimeType: 'image/png', data}` becomes an image block after the text block. The JSON text and `structuredContent` carry `contentIndex` and no `data`.
- [x] Unsupported mime types, non-string or empty `data`, class instances and error results are not lifted.
- [x] An artifact above the size bound (5 MiB decoded), declared wider or taller than 2000 px, or with invalid base64 stays inline. The pixel bound exists because, once a request carries more than 20 images, Anthropic's API refuses any larger image, and with it the whole turn. There is no depth or count cap: either would strand a deep or late artifact silently.
- [x] Live witness: a draw receipt requested as PNG from the Stage avatar lab arrives through this server as an image block the model sees.

## Out of Scope

- Producing images. The Stage companion, neomjs/neo-stage#124, encodes its receipts.
- Window screenshots, a new Neural Link tool, and `BaseServer` behaviour for the other servers.
- Whether Codex and Gemini CLI render MCP image content. The pointer metadata stays in the text either way; each harness gets checked when one of its seats first asks for a PNG.

## Avoided Traps

- Guessing images from long base64 strings. Only artifacts that declare a `mimeType` are lifted.
- Lifting in `BaseServer` for every server. No other server produces images, and the Neural Link is the Body's channel.
- Keeping the bytes in `structuredContent` as well. A client that reads it would pay the base64 text cost this ticket removes.

## Related

- neomjs/neo-stage#27: the body as a platform, the MX tools.
- neomjs/neo-stage#54: the Neural Link tool surface as a decision record.
- neomjs/neo-stage#39: completed-frame inspection, the producer of these artifacts.

Live latest-open sweep: checked the latest 20 open issues of neomjs/neo-agent-brain and neomjs/neo-stage at 2026-10-16T23:09:39Z, plus all-state A2A from the last hour; no equivalent. Adjacent, not overlapping: neomjs/neo-stage#121 (per-actor scene description) and neomjs/neo-stage#122 (invariance-check sandbox); neither returns an image a model can see.
MC sweep: `query_raw_memories` "inspection color rgba8-base64 bottom-to-top decode pngjs view frame waitForSceneDraw tool result too large" (6 results), `query_summaries` "Neural Link screenshot image vision MCP image content block model sees rendered frame" (8 results, noise). No prior decision found. Live `gh search` over neo, Brain and Stage for "neural link screenshot", "MCP image content", "image content block", "neural link vision" and "mimeType image/png": no hits.
Own-assignment sweep: 3 open in neo-agent-brain (#684 GitLab fleet auth, #18 recovery probe, #54 external-plane recovery), none overlapping

Origin Session ID: 69f6ebf6-5b58-404b-acc1-4e6e4ed5998e
Retrieval Hint: "Neural Link image content block model sees receipt frame pixels"

🖖 Grace (Opus 5.5, Claude Code) · session 69f6ebf6-5b58-404b-acc1-4e6e4ed5998e


## Timeline

- 2026-10-16T23:10:38Z @neo-opus-grace added the `enhancement` label
- 2026-10-16T23:10:38Z @neo-opus-grace added the `ai` label
- 2026-10-16T23:10:38Z @neo-opus-grace added the `model-experience` label
- 2026-10-16T23:10:38Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-16T23:18:37Z @neo-opus-grace cross-referenced by PR #1073
- 2026-10-16T23:46:20Z @neo-opus-grace referenced in commit `f839b9a` - "feat(neural-link): an image declared wider or taller than 2000 px stays inline (#1072)"
- 2026-10-17T00:00:43Z @neo-opus-grace referenced in commit `ef78ebd` - "feat(neural-link): the SDK's image schema judges an artifact's base64, and the bound counts decoded bytes (#1072)"
- 2026-10-17T00:00:43Z @neo-opus-grace referenced in commit `ca05edd` - "feat(neural-link): image blocks append without one call argument per image (#1072)"
- 2026-10-17T00:18:17Z @tobiu referenced in commit `c7bc06c` - "feat(neural-link): results deliver declared images as MCP image content (#1072) (#1073)

* feat(neural-link): results deliver declared images as MCP image content (#1072)

A Neural Link result that carries an image artifact - a plain object declaring an image mimeType with its bytes as base64 data, as a draw receipt's frame does - now reaches the model as an MCP image block after the JSON text. The JSON keeps the artifact's other fields and a contentIndex pointer. A result without artifacts, and every error, formats exactly as BaseServer formats it.

* feat(neural-link): an image declared wider or taller than 2000 px stays inline (#1072)

* feat(neural-link): the SDK's image schema judges an artifact's base64, and the bound counts decoded bytes (#1072)

* feat(neural-link): image blocks append without one call argument per image (#1072)"
- 2026-10-17T00:18:17Z @tobiu closed this issue
- 2026-10-17T00:27:43Z @neo-fable-mnemosyne cross-referenced by #1075
- 2026-10-17T01:43:22Z @neo-gemini-julia cross-referenced by PR #1076

