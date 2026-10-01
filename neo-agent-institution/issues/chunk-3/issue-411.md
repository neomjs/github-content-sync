---
id: 411
title: 'The cockpit sets a seat''s own plane credential, and the agent config names the Fleet instead of local services'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T20:15:06Z'
updatedAt: '2026-10-01T21:04:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/411'
author: neo-opus-vega
commentsCount: 1
parentIssue: 7
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
# The cockpit sets a seat's own plane credential, and the agent config names the Fleet instead of local services

## Context

Brain PR neomjs/neo-agent-brain#728 (resolves neomjs/neo-agent-brain#699) puts a seat's Memory Core and Knowledge Base on the plane the Fleet serves. Each seat presents its own plane credential, never its checkout PAT. The credential is stored through a new credential-bearing wire verb, `setPlaneCredential({id, credential})`.

Once a Brain pin carries #728, every repo-bearing seat that the installed FM starts on the local plane refuses until its credential is set. The cockpit cannot set one today.

## The Problem

The shell's credential ingress knows two verbs. `projectPublicCredentialIntent` (`harness/fleetCapability.mjs`) projects `defineAgent` and `connectTenant` and answers anything else with `null`, so the request is rejected as "malformed public intent". No cockpit surface offers the action. The agent config's "Memory & knowledge" chips still say "Local services" for the default target, which on a Fleet that serves a plane now means that plane.

## The Architectural Reality

- `apps/agentos/fleet/installFleetBridge.mjs:246` builds `registryBridge` from the Brain's `FLEET_WIRE_METHODS`, so `registryBridge.setPlaneCredential` exists as soon as the pin carries #728. On pin 8 it is absent.
- `harness/fleetCapability.mjs` `projectPublicCredentialIntent` decides what public intent a credential-bearing verb may carry. The credential then comes from the main-owned prompt, `harness/credentialPrompt.mjs` (`SUBMIT_TEXT`, fixed copy).
- `apps/agentos/view/fleet/detail/AgentConfigComponent.mjs` `createTargetChoices` renders the target chips. `mcpTarget: null` is labelled "Local services".
- `credentialIngress: 'shell'` (`apps/agentos/app.mjs:108`) is the installed FM. Worker ingress is the browser and dev path.

## The Fix

1. The shell projects `setPlaneCredential` to `{id}`, a valid seat id only. The credential comes from the prompt.
2. For this verb, the prompt asks for the seat's own PAT without repository access, never its checkout PAT. Submit reads "Set".
3. The agent config's "Memory & knowledge" section names the default target "This fleet", like the launcher chip above it, and gains a "Plane credential · Set" action. The controller calls `registryBridge.setPlaneCredential({id})` and shows the outcome: stored, or rejected with its reason. A bridge without the verb says the Fleet does not offer it. Worker ingress says to set it from the installed Fleet Manager.
4. This ships before the pin that carries #728. On pin 8 it is inert.

## Contract Ledger

The Brain producer and its validation rules are neomjs/neo-agent-brain#699's ledger (PR neomjs/neo-agent-brain#728); this side consumes them.

| Target surface | Source of authority | Behavior | Fallback | Evidence |
|---|---|---|---|---|
| Card intent `{id, planeCredential: true}` | `AgentConfigComponent.onCardClick`, offered by `createPlaneCredentialRow` | Fired only for a seat on this fleet whose harness reaches a remote Memory Core. | Tenant seats and Antigravity get no chip, and a forged click fires nothing. | `Accounts.spec` |
| Shell projection and credential custody | `harness/fleetCapability.mjs` `projectPublicCredentialIntent`; main's credential provider (`harness/credentialPrompt.mjs`) | Every renderer field except `id` is dropped. The credential comes only from main's prompt, which asks for the seat's identity-only PAT. | A malformed intent is refused before any prompt or network access. | `fleetCapability.spec`: the projection arm, plus the capability arm over #728's contract |
| Outcomes | `ConfigIntentRoundTrip.runPlaneCredentialIntent`, over the Brain's `{status, reason}` | `stored` reads "Plane credential stored." `rejected` shows the Brain's own reason. A canceled prompt or a transport error reads "was not stored". No verb on the bridge reads "does not take plane credentials yet"; worker ingress reads "from the installed Fleet Manager". | — | `configIntentRoundTrip.spec` |
| Definition record | — | Not written by this intent. | — | `configIntentRoundTrip.spec` (record unchanged) |
| Adoption boundary | Brain pins | Inert on pins before neomjs/neo-agent-brain#728: the verb is absent and the request is refused before any prompt or network access. Merges before any pin that carries #728. | — | `fleetCapability.spec`: the arm with a contract that lacks the verb |

## Acceptance Criteria

- [ ] `projectPublicCredentialIntent('setPlaneCredential', …)` returns `{id}` for a valid seat id and `null` otherwise. No renderer-supplied credential or extra field reaches the Brain request. Spec over the projection and the capability.
- [ ] The prompt for `setPlaneCredential` names the seat's own identity-only PAT and warns off the checkout PAT; its submit reads "Set".
- [ ] The config card offers "Plane credential · Set" and renders the outcome. A bridge without the verb and worker ingress each say why, and no request is sent.
- [ ] "Local services" no longer appears; the default target reads "This fleet".
- [ ] *(Post-merge, L4, installed FM, with a pin carrying Brain #728)* Setting a seat's credential from the card lets the seat start on the local plane.

## Out of Scope

- Showing whether a credential is stored. The credential never leaves the Brain, and no public read exists yet.
- A credential field for worker ingress.
- The Brain pin bump itself.

## Avoided Traps

- **A renderer password field in the shell.** It would put credential bytes in Body state; the shell prompt exists so they never get there.

## Related

neomjs/neo-agent-brain#699 · neomjs/neo-agent-brain#728 · #409 (pin 8) · neomjs/neo-agent-brain#571

Live latest-open sweep: checked the latest 20 open Institution issues at 20:14Z, plus a search for "plane credential"; no equivalent.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207
Retrieval Hint: "setPlaneCredential shell credential ingress agent config plane credential"




## Timeline

- 2026-10-01T20:15:08Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T20:15:08Z @neo-opus-vega added the `enhancement` label
- 2026-10-01T20:15:08Z @neo-opus-vega added the `agent-os` label
- 2026-10-01T20:15:09Z @neo-opus-vega added the `ai` label
- 2026-10-01T20:25:37Z @neo-opus-vega cross-referenced by PR #413
- 2026-10-01T20:27:15Z @neo-opus-vega cross-referenced by #699
- 2026-10-01T20:27:32Z @neo-opus-vega cross-referenced by PR #728
- 2026-10-01T20:45:05Z @neo-opus-vega added parent issue #7
- 2026-10-01T20:45:14Z @neo-opus-vega referenced in commit `b43ca56` - "test(harness): the plane-credential capability path runs over Brain #728's contract and refuses on a contract without the verb (#411)"
- 2026-10-01T20:54:22Z @neo-opus-vega referenced in commit `57a482c` - "test(harness): the capability fixture names the contract it injects, not its ticket (#411)"
### @neo-opus-vega - 2026-10-01T21:04:24Z

**Handover (Vega, session sunset 2026-10-01 21:0xZ).** PR #413 at `57a482c`: Sophie's two actions are addressed, and her formal re-review waits for green. Her earlier write was blocked by the reviewer-family guard (neomjs/neo-agent-brain#700).

Pickup protocol:
1. **If #412 merges first** (Ada: same seven files), rebase #413 onto `dev` and resolve. Then re-capture the three Accounts goldens on Darwin, run `npm ci` and `npm run build-themes -- -n -e dev -t all` first, then `npx playwright test -c test/playwright/playwright.config.visual.mjs -g "ccount" --update-snapshots`. Stage, run `npm run stamp-visual-baselines`, push, and send Sophie the new head.
2. **Merge order:** #413 merges before any Brain pin that carries neomjs/neo-agent-brain#728, which is approved and with the operator. Then the operator creates one identity-only PAT per seat for the local plane and sets each through "Plane credential · Set".
3. AC-5 (installed L4) stays with #7.



