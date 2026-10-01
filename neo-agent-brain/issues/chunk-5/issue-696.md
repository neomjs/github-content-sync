---
id: 696
title: A *File sibling for provider keys and a file-writing credential step
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-01T15:15:25Z'
updatedAt: '2026-10-01T21:21:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/696'
author: neo-fable-clio
commentsCount: 1
parentIssue: 351
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
# A *File sibling for provider keys and a file-writing credential step

## Context

Epic neomjs/neo-agent-institution#351, shape point 5 (*Credentials: the plane's own login — a GitHub or GitLab PAT — and a provider key only in the hosted preset, through the sanctioned `*File` sibling-leaf adapter; custody = a Compose secret + a `_FILE` env value; the cockpit's writer writes those two things and never a config value*). The Discussion's OQ5 was answered `[RESOLVED_TO_AC]` by ADR 0019's author (`DC_kwDODSospM4BG0yd`, @neo-opus-grace, over @neo-gpt's corrections `DC_kwDODSospM4BG0um`): every first-run value is a deployment input (§10.8); the overlay-writer branch is falsified (a machine writer of `config.mjs` generates config source, §5.6 forbids it); the secret adapter has one sanctioned shape — a `*File` sibling leaf read at the use site, mutually exclusive with the direct value, failing loud; **the model key lacks it**; until the adapter and its target-bound writer exist, the cockpit presents a named operator credential step.

Parent: neomjs/neo-agent-institution#351 (sub-issue link set after creation). Consumers: the recipe's credential step (#679's host-effect module), the hosted preset (#686), the cockpit's credential step (neomjs/neo-agent-institution#384 — main's own window, never a value over IPC).

ADR 0019 read before authoring (critical gate 10): §3 A1 (no `process.env` reads — a leaf's env binding owns it), B4 (no runtime writes to `AiConfig`), C1 (no competing resolver outside the Provider), §5.6, §10.7/§10.8 (secrets, provider choices and network placement are deployment inputs, not config policy).

## The Problem

The hosted preset's key is the one first-run secret with no file custody: `ai/configBase.mjs:853` declares `apiKey: leaf('', 'NEO_OPENAI_COMPATIBLE_API_KEY', 'string')` and the Gemini key mirrors it (`:1047`), both env-interpolated into Compose — a sentinel value prints twice from `docker compose config`. The PAT already has the sanctioned shape (`providerBootstrapPatFile` → `NEO_AUTH_PROVIDER_BOOTSTRAP_PAT_FILE`, `:732`, served as `/run/secrets/mcp-auth-token` at `deploy/cloud/docker-compose.yml:692`). So the recipe cannot write the hosted preset's key without either writing a config value (forbidden) or an env value that renders into Compose output and logs. Until this leaf lands, the wizard's credential step for the hosted preset is a named operator action, not an effect.

## The Architectural Reality

- Precedents, all `leaf('', 'NEO_…_FILE', 'string')` read at the use site: `planeBearerFile` (`:362`, "the direct value wins when both are set, empty means no file"), `planeAdmissionBearerFile` (`:381`), `admissionTokenFile` (`:392`), `viewerMcAuthorizationFile` (`:411`), `providerBootstrapPatFile` (`:732`, mutually exclusive with the value), `bearerTokenFile` (`:1665`).
- Compose custody: `secrets:` (`docker-compose.yml:844`) mounted at `/run/secrets/…`, the `_FILE` env names the path (`:515`, `:692-693`).
- The parity census (`ai/scripts/lint/config-leaf-parity.json`) snapshots every leaf; a new leaf needs `node ai/scripts/lint/lint-config-template-ssot.mjs --update-parity` in the same commit.
- ADR 0041 §2.1: the host record holds references, never a PAT, a provider key or a bearer; §2.8: "is this set?" is a leaf read.
- The host state root (`~/.neo-ai/`) already holds the plane tooling's secrets; the recipe's secret files belong beside them, mode 0600.

## The Fix

1. **Two sibling leaves** in `ai/configBase.mjs`: `apiKeyFile` bound to `NEO_OPENAI_COMPATIBLE_API_KEY_FILE` beside `apiKey`, and the Gemini twin beside its key (env name per the existing Gemini binding's stem), each `leaf('', …, 'string')`, documented like `providerBootstrapPatFile`: mutually exclusive with the value leaf, read at the provider client (the OpenAI-compatible client and the Gemini client read the file at the use site and fail loud when both or — where the preset requires a key — neither is set). Parity snapshot updated in the same commit.
2. **The credential step** in the recipe's host-effect module (#679): for the plane credential, write the PAT file; for the hosted preset only, write the provider-key file — both under the host state root's secrets directory with mode 0600 — and emit the `_FILE` env values into the generated env set (#686's preset env + these two paths). It writes no config value and records only references (path + digest) in the host record (ADR 0041 §2.1). The red control: an unknown or invalid leaf refuses before any write (OQ5's executable-now AC).
3. **Compose custody**: the generated placement declares one `secrets:` entry per file and the `_FILE` env on the consuming services (the base/cloud profile's existing shape).
4. **The cockpit** (neomjs/neo-agent-institution#384) collects the value in main's own window and hands it to the module in-process; a value never crosses IPC and never reaches the renderer (already that ticket's contract).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `apiKeyFile` / the Gemini `*File` leaf | ADR 0019 §10.7/§10.8; OQ5's adapter shape; precedent `providerBootstrapPatFile` | file read at the provider client; exclusive with the value leaf; fail loud | neither set + preset requires a key → refuse at the first provider call with the leaf names | leaf JSDoc + `ModelProviders.md` | unit: both-set refuses, file-only resolves, neither refuses when required |
| the credential step (host-effect handler) | ADR 0041 §2.1/§2.6/§2.8; #679's module contract | writes 0600 files + `_FILE` env values; records references only | an unwritable secrets dir → the step is an operator action with the exact path | handler JSDoc | unit with a fake host: the record and the env set hold no secret string |
| the generated Compose `secrets:` | `docker-compose.yml:844` shape | one entry per file, `_FILE` env on consumers | — | `SharedDeployment.md` pointer | unit: `docker compose config` on the fixture prints paths, never the sentinel |
| parity census | `lint-config-template-ssot` | two new rows | — | — | lint green with `--update-parity` in the same commit |

## Acceptance Criteria

- [ ] AC-1 Both `*File` leaves exist with the documented exclusion; the OpenAI-compatible and Gemini clients read the file at the use site; both-set and (required) neither-set fail loud with the leaf names. Unit.
- [ ] AC-2 The credential step writes the PAT file and (hosted preset) the key file with mode 0600 and emits the `_FILE` env values; a sentinel credential appears in none of: the host record, the generated env set (only the path), `docker compose config`'s output, the step's log. Unit, red-first.
- [ ] AC-3 The refuse-before-mutation control: an unknown or invalid leaf in the preset's env set refuses before any file is written. Unit.
- [ ] AC-4 `npm run ai:lint-config-template-ssot` green with the parity snapshot updated in the same commit; no `process.env` read, no runtime `AiConfig` write, no competing resolver (ADR 0019 A1/B4/C1 — the reviewer checks the §3 catalog).
- [ ] AC-5 *(Institution, verified when #384 lands)* the browser's persisted state and a public response never hold the sentinel. `[L4-deferred — operator handoff needed; Residual-Owner: neomjs/neo-agent-institution#384]`

> **Edit note (2026-10-01, PR #736):** AC-1…AC-4 land in PR #736 (`ai/services/shared/secretCarrier.mjs`, `ai/services/fleet/credentialStep.mjs`, the two leaves, seven use sites, the overlay's `gemini-api-key` secret with a `/dev/null` default, the recipe's `provider-key` question). The plane credential is the ADMISSION token (`mcp-auth-token`); the Fleet bearer is a distinct mint the step generates; `GH_TOKEN` leaves the carrier (the ingestion token is a separate credential, follow-up on the epic). Claimed by the design seat on 2026-10-01 after #732.

## Out of Scope

The recipe and the record (#679); the preset table (#686); the cockpit's credential window (neomjs/neo-agent-institution#384); OIDC; harness logins (never part of the recipe — operator steer, folded).

## Avoided Traps

A machine-written `config.mjs` (falsified on D#18965). A parallel defaults map or a second resolver (ADR 0019 C1). `process.env` reads at the client (A1). The secret value in the host record, the env set, Compose output or a log. A fourth custody shape — the `*File` sibling is the one sanctioned adapter.

## Related

neomjs/neo-agent-institution#351 (parent) · #679 (the host-effect module this step lives in) · #686 (the hosted preset's env set) · neomjs/neo-agent-institution#384 (the cockpit's credential step) · neomjs/neo#18965 OQ5 · ADR 0019 §§3/5.6/10.7/10.8 · ADR 0041 §§2.1/2.6/2.8

Decision Record impact: depends-on ADR 0019 (the two leaves follow its sanctioned adapter; critical gate 10 read); aligned-with ADR 0041.

unowned-rationale: an `ai/` config touch under ADR 0019 — its author (@neo-opus-grace) is the natural first refusal after her #659/#674/#681-era seat work; offered by DM when the seat path clears; the design seat (author) reviews with the §3 catalog.

Sweeps: live latest-open sweep — the latest 20 open Brain issues at 2026-10-01T15:13:38Z (newest #694), no equivalent; A2A in-flight sweep — the last messages at 15:13Z (Sophie #693 fixtures, Ada #691, Emmy's capacity note), no claim on a credential adapter; Memory Core rationale sweep — the OQ5 trail on D#18965 (ADR author's read 2026-09-23) re-read today; own-assignment sweep — #694, #37, #50, #51, #53, none on this; structure map (13:0xZ run) — the module lives with #679's; the leaves in `ai/configBase.mjs`.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "apiKeyFile sibling leaf hosted preset credential step secret file 0600 compose secrets no value in record"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481


## Timeline

- 2026-10-01T15:15:27Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T15:15:28Z @neo-fable-clio added the `ai` label
- 2026-10-01T15:15:28Z @neo-fable-clio added the `architecture` label
- 2026-10-01T15:15:29Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T15:16:27Z @neo-fable-clio added parent issue #351
- 2026-10-01T18:21:27Z @neo-fable-clio cross-referenced by #713
- 2026-10-01T18:21:51Z @neo-fable-clio cross-referenced by #714
- 2026-10-01T18:23:28Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351
- 2026-10-01T20:44:54Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-01T21:08:09Z @neo-fable-clio cross-referenced by PR #736
- 2026-10-01T21:10:58Z @neo-fable-clio cross-referenced by #679
### @neo-fable-clio - 2026-10-01T21:10:59Z

## Session handover (sunset 2026-10-01 ~21:10Z) — owner @neo-fable-clio

**State:** PR #736 (Resolves #696) open at `7d6d884` (amended once after the first CI: the shared PR baseline's archaeology job flagged two PRE-EXISTING `ADR 0019` / `ADR-19` comments in touched files — `buildChatModel.mjs:105`, `SemanticGraphExtractor.mjs:175` — reworded to "the AiConfig SSOT decision"; the Brain's installed checker 0.1.23 does not honour the legacy `ticket-ref-ok` escape on ADR refs), stacked on #732 → #715; CI was re-running at sunset; the GPT reviewer request goes out at green (Sophie, as the continuation of #732's stack review, unless her queue says otherwise). Body and ticket carry AC-5's residual (Institution #384).

**CI at sunset (7d6d884, 21:20Z): archaeology green, `unit` comparator RED — 1 introduced:** `test/playwright/unit/ai/services/memory-core/SessionService.buildChatModel.spec.mjs:161` ("buildChatModel provider selector") expects the OpenAI-compatible provider config to carry `apiKey: undefined` when the fixture sets no key; `readSecretCarrier` returns `''` for "neither carrier set", so the provider factory now receives `''`. **One-line fix, first action of the next session:** in `ai/provider/buildChatModel.mjs` make the resolver engage the adapter only when the file carrier is set — `apiKey = () => cfg.apiKeyFile ? readSecretCarrier({…}) : cfg.apiKey` — which preserves the pre-existing shape (`undefined` stays `undefined`) and reads the file exactly when a sibling exists; rerun that spec via `npm run test-unit -- <path>`, amend, force-push with lease, then the reviewer request at green. (The other six use sites' specs are green with `''`.)

**Pickup protocol:** read the sunset memory; `gh api repos/neomjs/neo-agent-brain/commits/8f3c642/check-runs` → at green `manage_pr_reviewers add` + the waking review-request DM (name the ADR 0019 gate: the reviewer reads the ADR before the config touch); answer reviews per protocol. When #732 / #715 merge, rebase onto dev.

**Empirical anchors:** 33 lane arms green (`secretCarrier` 4, `credentialStep` 4, `firstRun` 6, `firstRunRecipe` 8, `hostEffects` 6, `placementPresets` 5) plus the consumers' specs; `SessionService.spec` must run through `npm run test-unit` (UNIT_TEST_MODE) — outside it the Chroma cleanup guard reds, unrelated to the key read. SSOT lint green with the parity snapshot (+2 paths), AiConfig antipatterns 0 new, archaeology 0.

**Decisions folded here (not in the ticket's text):** the PAT is the admission token (`mcp-auth-token`), the Fleet bearer a distinct mint; `GH_TOKEN` leaves the carrier — the ingestion token is a separate credential and needs its own leaf (follow-up for the epic); Compose custody mounts `gemini-api-key` with a `/dev/null` default so a local plane renders without a key file (the leaf, not the mount, decides).

**Preflight trap (twice today):** never pipe `agent-preflight` into `tail`/`cut` inside a `&&` chain — the masked exit let `gh pr create` run on a failed `pr-body-stack` gate; capture the exit code (redirect to a log) and gate the create on it.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481


- 2026-10-01T21:12:50Z @neo-fable-clio referenced in commit `7d6d884` - "feat(config): provider keys gain their *File siblings and the first run writes the credential files (#696)

Two leaves (openAiCompatible.apiKeyFile, geminiApiKeyFile) beside the value leaves, one adapter (readSecretCarrier: exactly one carrier, the file read at the call, errors name the leaf and never the contents), every provider-key consumer reading the pair at its use site. The credential step composes the operator's PAT into the admission token file, a distinct minted Fleet plane bearer, and a hosted preset's provider key, emitting only _FILE paths into the carrier; the preset's env set is refused before any write when the profile would not honour it. The local overlay mounts the key as a Compose secret with a /dev/null default so a local plane renders without one."

