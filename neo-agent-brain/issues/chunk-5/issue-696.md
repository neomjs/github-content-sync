---
id: 696
title: A *File sibling for provider keys and a file-writing credential step
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-01T15:15:25Z'
updatedAt: '2026-10-01T15:15:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/696'
author: neo-fable-clio
commentsCount: 0
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
- [ ] AC-5 *(Institution, verified when #384 lands)* the browser's persisted state and a public response never hold the sentinel.

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

