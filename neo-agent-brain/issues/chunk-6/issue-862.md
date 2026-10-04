---
id: 862
title: A Fleet seat starts on its declared model and reasoning effort
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T19:26:43Z'
updatedAt: '2026-10-04T19:32:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/862'
author: neo-opus-vega
commentsCount: 0
parentIssue: 571
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
# A Fleet seat starts on its declared model and reasoning effort

Sub of #571 · design read: Clio, [#571 comment 5983426039](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5983426039) · Institution half: neomjs/neo-agent-institution#559

## Context

On 2026-10-04, while the eight peers joined the FM roster, the operator asked that a Fleet-started harness start on the seat's model and thought level (for example Opus 5.5 at Max). Today it starts on the harness default, and only a hand adjustment inside the harness fixes it. A seat can therefore run below its intended level without anyone seeing it. All eight roster seats are desktop harnesses: five `claude-desktop` and three `codex-desktop` (local registry read, 2026-10-04).

## The Problem

No seat record carries a model or a reasoning effort. `FleetRegistryService.configureAgent` accepts `harnessType`, `mcpServers`, `mcpTarget` and the declared commit identity. Neither `deriveHarnessLaunchSpec` nor `prepareManagedAgentWorkspace` writes either value.

## The Architectural Reality

- **Declared vs derived** already has one precedent: the commit identity in `configureAgent` (`gitName`/`gitEmail`, where `null` returns to derivation), from #829 and Institution #524.
- **The launch surfaces Fleet owns:** `deriveHarnessLaunchSpec` composes the `claude-code` arguments itself, and the Codex families read the Fleet-managed `config.toml`. `convergeJsonSetting` in `prepareManagedAgentWorkspace` is the precedent for a value someone else set: it refuses instead of replacing.
- **Per-family surfaces**, read 2026-10-04:

  | Family | Model | Effort | Evidence |
  | --- | --- | --- | --- |
  | `claude-code` | `--model` | `--effort` (low, medium, high, xhigh, max) | `claude --help`, CLI 2.1.212 |
  | `codex`, `codex-desktop` | `config.toml` `model` | `model_reasoning_effort` | the bundled codex binary carries the key. Codex Desktop starts `codex app-server --listen` with no model flags, and a running seat's `config.toml` carries both keys. `prepareManagedAgentWorkspace.spec` already preserves them in the Fleet-managed file |
  | `claude-desktop` | the app's, per session | the app's, per session | the app starts every Code session with its own `--model` and `--effort` (five running sessions read 2026-10-04), and [Claude Code's settings docs](https://code.claude.com/docs/en/settings) rank `--model` above the `model` key in any file. The app remembers the last effort per profile (`ccd-effort-level`), so a new profile starts on the app's default |

- **Codex's efforts depend on the model.** Its model catalog names each model's supported efforts; `learn/agentos/ModelStats.md` records `ultra` and `max` on the GPT seats' model. A Fleet-side list would drift, and ADR-0019 §3 rules out hidden defaults. The harness therefore keeps its own default and its own catalog.

## The Fix

1. The seat record gains `model` and `reasoningEffort`, declared through `configureAgent`'s curated intent. `null` returns to the harness default.
2. Start carries the declared values into what the seat's harness reads at launch:
   - `claude-code`: `--model` and `--effort` in its launch arguments.
   - `codex` and `codex-desktop`: `model` and `model_reasoning_effort` in the Fleet-managed `config.toml`. A different value someone else set is refused with its reason, never replaced.
   - `claude-desktop`: nothing. The app passes its own values on every session, so the seat's rows read `set per session in the app`, and the record refuses a declaration for that family.
3. A read gives Configuration the values to offer, asked of the harness: Codex through its model catalog, `claude-code` through the effort levels its CLI names and the model ids it accepts. A declared value outside that set refuses Start in the design read's words.

`prepareManagedAgentWorkspace.mjs` is 1,987 code lines, so the `config.toml` step may belong in a sibling module. The PR names what it removes, or why nothing.

## Acceptance Criteria

- [ ] AC-1: `configureAgent` sets and clears `model` and `reasoningEffort` and refuses them for `claude-desktop`, and the public definition reads them back. Unit.
- [ ] AC-2: Start passes the declared values to `claude-code` as arguments and writes them into the `config.toml` of `codex` and `codex-desktop`. A different `config.toml` value someone else set is refused with its reason. Unit, per family.
- [x] AC-3: probe `claude-desktop` before writing for it. Result, 2026-10-04: negative. The app passes `--model` and `--effort` on every session, and those flags outrank any settings file, so the family leaves AC-2. Writing another app's local storage from outside is not a contract, so the remembered choice is never a write target either.
- [ ] AC-4: the offered values come from the harness, never from a Fleet list. A declared value outside them refuses Start with `start refused: model <x> is not available — change it in Detail › Configuration`. Unit, and the PR names each family's source.
- [ ] AC-5 (post-merge, installed): on the next #12 candidate, a `codex-desktop` seat declared at a model and effort its catalog offers starts on them, read inside the harness.

## Out of Scope

- The Configuration rows and the card's refusal line: Institution #559.
- Deriving the seat's family from a declared model: #700's surface (Sophie), Clio's fold in the same design read.
- `opencode`, `antigravity` and `kimi-code`, which no roster seat runs. That includes `generateKimiSeatConfig`'s `defaultModel = 'kimi-code/k3'`.

## Related

#571 (epic) · #829 and Institution #524 (the declared-vs-derived precedent) · #700 · Institution #12 (the candidate the installed witness rides)

Decision Record impact: aligned-with ADR-0019 (per-seat registry data, no AiConfig leaf, no hidden default).

Live latest-open sweep: latest 20 open issues in `neo-agent-brain` and `neo-agent-institution` at 19:26Z, plus a title/body search for model and effort; no equivalent found.
A2A in-flight sweep: the last 30 messages (16:57Z–19:23Z); no claim on a seat's model or effort.
MC sweep: "harness starts on its default thought level, peers less smart unless the operator adjusts the model inside the harness", 6 results, no prior decision found.
Own-assignment sweep: 10 open in Brain, 3 in Institution, none overlapping. #768 is about how a `claude-desktop` seat's wake is armed, not its launch configuration.

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "seat model reasoning effort launch configuration Start harness default"


## Timeline

- 2026-10-04T19:26:44Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T19:26:45Z @neo-opus-vega added the `enhancement` label
- 2026-10-04T19:26:45Z @neo-opus-vega added the `ai` label
- 2026-10-04T19:26:45Z @neo-opus-vega added the `agent-os` label
- 2026-10-04T19:26:58Z @neo-opus-vega cross-referenced by #559
- 2026-10-04T19:27:15Z @neo-opus-vega added parent issue #571

