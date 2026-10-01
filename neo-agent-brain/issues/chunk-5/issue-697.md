---
id: 697
title: A cloud placement is a bundle the operator runs on the target
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-01T15:16:03Z'
updatedAt: '2026-10-01T15:16:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/697'
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
# A cloud placement is a bundle the operator runs on the target

## Context

Epic neomjs/neo-agent-institution#351, Concept §1 as the operator declared it on 2026-09-30: *the first run provisions their own instance through the setup wizard, on this machine by default or as a cloud deployment they provision, which the wizard prepares as a placement and never as a service of ours*. The Discussion's OQ6 (*a remote plane: the recipe's effects run on the server; transport, auth, and what the cockpit may honestly show about a run it does not host*) was dispositioned onto this leaf: the cloud deployment is a **placement** the wizard prepares — compose + env + secret files on the target — the recipe's effects run where the plane runs, and the cockpit shows a run it does not host through Concept §2's projection rule.

Parent: neomjs/neo-agent-institution#351 (sub-issue link set after creation). Upstream: #679 (the recipe, the record and the host-effect module), #685 (the probe runs ON the target: `target.kind: 'remote-json'`), #686 (the preset's env set), the credential-step leaf (secret files), neomjs/neo-agent-brain#83 (a FUTURE admitted remote transport — until it exists a remote host effect is an operator action, ADR 0041 §2.6).

## The Problem

Nothing today prepares a plane for a machine the operator is not sitting at: `SharedDeployment.md` and `DeploymentCookbook.md` (sections 1–10, no remote recipe) assume the operator runs compose on the host; the Connect door attaches to a plane that exists. An outside operator who chose "cloud" in the wizard has no bundle to carry, no probe to run there, and no honest way to see that run in the cockpit.

## The Architectural Reality

- ADR 0019 §10.7: the base/cloud profile (`deploy/cloud/docker-compose.yml`) is the canonical placement (`/app/.neo-ai-data`, named volumes); §10.8: secrets, provider choices, network placement and privileged capabilities are deployment inputs; the plane declares an opaque `plane.id` before launch (§10.3).
- ADR 0041 §2.4–2.5: a run binds to the declared `plane.id` and its bound data-root expectation; once the served identity and root match, plane observations own current readiness; §2.6: a remote host effect waits for an admitted transport and is an operator action until then — accepted effects never replay.
- Compose custody of secrets: `secrets:` + `_FILE` env (`docker-compose.yml:515`, `:692-693`, `:844`).
- The probe (#685) runs on the machine that bears the workload; for a remote target the CLI runs it there (`--probe --json`) and the cockpit shows that JSON.
- The record (#679) holds consent and receipts — for a remote placement the receipts are what was PREPARED (digests), not what ran.

## The Fix

1. **`preparePlacement({target: 'remote', preset, answers})`** in #679's host-effect module: writes a bundle directory under the host state root (`~/.neo-ai/setup/<runId>/placement/`): the base/cloud `docker-compose.yml` with the declared `plane.id` and plane members, `.env` from #686's preset env set, the secret files (mode 0600, from the credential step) with their `_FILE` env values and `secrets:` entries, the probe command (`<cli> --probe --json`), and a short `README` naming the four operator actions — copy the bundle to the target · run the probe there and paste its JSON back · `docker compose up -d` · paste the plane URL back. Everything is a reference or a file; the record gets digests.
2. **Operator actions, consented and receipted**: each remote effect is a recipe step of kind `effect` whose handler is "an explicit operator action" (ADR 0041 §2.6) — the recipe shows the command, the operator confirms it ran, the receipt records the confirmation and the bundle digest; nothing executes remotely from the wizard in v1.
3. **The attach half**: after `up`, the operator pastes the plane URL; the recipe's observation step authenticates with the plane's own PAT, reads the served `plane.id` and root, and binds the run (ADR 0041 §2.4–2.5) — the Connect door's existing path (`attachPlane()` precedent), now the second half of a create.
4. **The cockpit** (neomjs/neo-agent-institution#384) projects the bundle's steps as operator actions with their receipts and the pasted probe JSON; a remote run it does not host shows exactly what the record and the plane's observations say, never a local status.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `preparePlacement()` bundle | Concept §1 (placement, never a service); ADR 0019 §10.7 base/cloud profile | compose + env + secret files + probe command + README under the run's directory | an unwritable run directory → the step refuses before writing anything | module JSDoc + a `DeploymentCookbook.md` section | unit: fixture bundle complete; `docker compose config` renders; no secret string in the record |
| remote effect steps | ADR 0041 §2.6 | operator actions with consent + receipt (bundle digest, confirmation) | — | JSDoc | unit: an unconfirmed action stays `pending`, a confirmed one is `accepted` and never replays |
| the attach half | ADR 0041 §2.4–2.5; `attachPlane()` | bind on served id AND root; a same-id/different-root responder refused | mismatch → fail closed, the step names it | JSDoc | unit: the ADR §3 witness, remote arm |
| the probe on the target | #685 (`remote-json`) | the pasted JSON feeds the placement step; an unprobed target is marked `unprobed` | — | JSDoc | unit: `fitsPreset()` consumes pasted JSON |

## Acceptance Criteria

- [ ] AC-1 On a fixture preset, `preparePlacement` writes the complete bundle (compose · env · secret files 0600 · probe command · README); `docker compose config` on it renders with paths, not secret values; the record holds the bundle's digests and no secret string. Unit.
- [ ] AC-2 Remote effect steps: pending until the operator confirms; confirmed → `accepted` with its receipt; a resumed run never re-presents an accepted action as pending (ADR 0041 §3, remote arm). Unit, red-first.
- [ ] AC-3 The attach half binds the pasted plane URL to the run by served `plane.id` + root and refuses a same-id/different-root responder. Unit.
- [ ] AC-4 The bundle brought up locally in a scratch directory (the fixture plane recipe) stands up a plane whose served identity binds the run end to end. E2e on the fixture plane.
- [ ] AC-5 *(post-merge)* the first real remote placement by an outside operator recorded on the epic with its density count.

## Out of Scope

Executing anything on the target from the wizard (neomjs/neo-agent-brain#83's transport); SSH / cloud-provider APIs; TLS, DNS or ingress provisioning beyond what the base profile declares; a Neo-operated service (option E, deferred on D#18965).

## Avoided Traps

A remote execution path minted before #83's admitted transport. A bundle that embeds secret values instead of files. A "local" status shown for a remote run (Concept §2's projection rule). A second compose profile — the base/cloud profile IS the placement.

## Related

neomjs/neo-agent-institution#351 (parent) · #679 · #685 · #686 · the credential-step leaf · neomjs/neo-agent-institution#384 · neomjs/neo-agent-brain#83 · neomjs/neo#18965 Concept §1 + OQ6 · ADR 0019 §§10.3/10.7/10.8 · ADR 0041 §§2.4–2.6

Decision Record impact: aligned-with ADR 0041 and ADR 0019; none amended.

unowned-rationale: a build lane behind the Claude Desktop seat path and behind #679 (it lives in that module); the deploy-topology reader's owner (@neo-opus-vega, #585/#588) or #679's eventual owner are the natural first refusals; the design seat (author) specifies the README's four actions and reviews.

Sweeps: live latest-open sweep — the latest 20 open Brain issues at 2026-10-01T15:13:38Z (newest #694), no equivalent; A2A in-flight sweep — the last messages at 15:13Z, no claim on a remote placement; Memory Core rationale sweep — the OQ6 disposition and Concept §1 on D#18965 (re-read today), the Cookbook's sections 1–10 (no remote recipe); own-assignment sweep — #694, #37, #50, #51, #53, none on this; structure map (13:0xZ run) — the module lives with #679's.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "cloud placement bundle compose env secret files probe on the target operator actions attach half plane.id"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481

## Timeline

- 2026-10-01T15:16:05Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T15:16:06Z @neo-fable-clio added the `ai` label
- 2026-10-01T15:16:06Z @neo-fable-clio added the `architecture` label
- 2026-10-01T15:16:06Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T15:16:28Z @neo-fable-clio added parent issue #351
- 2026-10-01T18:21:27Z @neo-fable-clio cross-referenced by #713
- 2026-10-01T18:21:51Z @neo-fable-clio cross-referenced by #714
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351

