---
id: 448
title: The add-agent form defines a GitLab seat on its own instance
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T14:02:40Z'
updatedAt: '2026-10-02T14:23:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/448'
author: neo-opus-grace
commentsCount: 0
parentIssue: 684
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 755 A GitLab seat''s repositories default to its own instance'
  - '[ ] 442 Carry the new Brain pin with forge-correct MCP controls'
blocking: []
---
# The add-agent form defines a GitLab seat on its own instance

## Context

On 2026-10-01 the operator asked for GitLab parity in the Fleet, which neomjs/neo-agent-brain#684 tracks. The point is that a seat whose repository lives on a GitLab instance, often self-hosted, works like a GitHub seat. The Brain half has merged:
- a GitLab PAT per seat (neomjs/neo-agent-brain#712);
- clone auth scoped to the seat's host;
- injection of `gitlab-workflow` (neomjs/neo-agent-brain#727);
- forge-aware MCP defaults (neomjs/neo-agent-brain#729).

#442 / PR #445 pins that Brain (`a9dd22f`), and the card follows the seat's forge.

#245 deferred the cockpit half: *"GitLab agent definitions: they need the Brain's GitLab credential injection first"*. That prerequisite is now met, and no ticket carries the cockpit half.

## The Problem

An operator whose repository is on GitLab cannot define a seat from the cockpit.
- The add-agent form collects a GitHub account and an `owner/repo` slug.
- `AddAgentFlow.repoOf` turns that slug into `https://github.com/<slug>.git` and rejects anything that is not exactly `owner/repo`, so a nested group (`group/sub/project`) and a self-hosted host cannot be entered.
- The definition intent never carries `forge` or `forgeHost`, so every cockpit seat is a GitHub seat.

The Repositories card has the same limit. It sends `{repoSlug}` only, so the Brain records a github.com clone URL even for a GitLab seat (`setRepos` runs no seat-forge check).

This also blocks a product-path sitting for neomjs/neo-agent-brain#684's installed arms (AC-2: a private GitLab clone; AC-3: `gitlab-workflow` reaching the instance).

## The Architectural Reality

**The Brain contract at the pinned `a9dd22f`.**
- `FleetRegistryService.defineAgent` takes `forge` (`'github'` by default, or `'gitlab'`) and `forgeHost`.
  - `forgeHost` is required with `gitlab` and must be the instance's bare `https` origin.
  - A GitLab seat's public definition carries both fields; a GitHub seat carries neither.
- A GitLab repository entry needs its clone URL, because "a self-hosted host cannot be derived" (`FleetManager` `repoCoordinates`).
  - The entry records `forge: 'gitlab'`.
  - `assertOnSeatForge` refuses a repository that is off the seat's forge or off its `forgeHost`.
  - A GitLab slug may name nested groups; a GitHub slug is exactly `owner/repo`.
- `githubUsername` stays the required identity field for every seat (`FleetLifecycleService` reads it as the agent identity), so a GitLab seat sends its username there.

**The Institution today.**
- `apps/agentos/view/fleet/instances/AddAgentForm.mjs` shows a heading "GitHub account", a username field, a token field (placeholder `github_pat_…`) and a working-repository field (placeholder `owner/repo`).
- `apps/agentos/util/AddAgentFlow.mjs`:
  - `createDefineAgentIntent` builds the public intent;
  - `submitDefineAgent` runs `defineAgent`, then `assignRepo` → `setRepo({id, cloneUrl, repoSlug})`;
  - `repoOf` is GitHub-only.
- `apps/agentos/view/fleet/detail/AgentReposContainer.mjs` fires `configIntent` with `repos: [..., {repoSlug}]`.
- The shell credential window already reads "GitHub or GitLab PAT" (`harness/credentialPrompt.mjs`), so no shell change is needed.
- `AgentDefinition` gains `forge` in PR #445. `forgeHost` is not yet a field.

## The Fix

1. **The form names its forge.**
   - A GitHub / GitLab choice heads the account section, GitHub by default, and the heading follows the choice.
   - GitLab adds an instance field: the https origin, prefilled `https://gitlab.com` and editable for self-hosted instances.
   - The token placeholder and the repository placeholder (`owner/repo` / `group/project`) follow the forge.
2. **The flow carries the forge, never a clone URL.**
   - `createDefineAgentIntent` adds `forge: 'gitlab'` and `forgeHost` for a GitLab seat only, so a GitHub intent stays byte-identical.
   - `repoOf` checks the slug against the forge's shape (GitHub exactly `owner/repo`; GitLab may name nested groups) and answers `{repoSlug}` only. The Brain composes the clone URL from the seat (neomjs/neo-agent-brain#755), so the Body's own `https://github.com/<slug>.git` composition is deleted too.
   - The rejection text names the forge's shape.
   - The readback guard accepts the public `forge` and `forgeHost`.
3. **The Repositories card follows the seat.**
   - A new entry stays `{repoSlug}` for both forges.
   - The field accepts nested groups on a GitLab seat, and its placeholder follows the forge.
   - **Carried-forward entries go whole.** `getOtherRepos` maps each existing entry to `{cloneUrl, repoSlug}`, which drops the `forge` the Brain stores for a GitLab repository. Any add or remove on a GitLab seat would then re-send its repositories as GitHub: a nested slug is refused, and a two-segment one is recorded under the wrong forge. The card carries every entry whole instead (@neo-opus-ada).

No new module and no new authority. The Brain composes and validates every coordinate; the cockpit only names the forge and the instance the operator chose.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Add-agent form account section | `defineAgent`'s `forge` / `forgeHost` (Brain `a9dd22f`) | A GitHub / GitLab choice. GitLab adds the instance origin, and the heading and placeholders follow the forge. | GitHub, the default, renders as today. | form JSDoc | component / unit arm |
| `AddAgentFlow.createDefineAgentIntent` | the same | A GitLab seat adds `forge: 'gitlab'` and `forgeHost`. | A GitHub intent is unchanged, with neither key. | method JSDoc | unit: both intents |
| `AddAgentFlow.repoOf` | `repoCoordinates`; the seat-aware default (neomjs/neo-agent-brain#755) | `{repoSlug}` for both forges. GitLab may name nested groups; the Brain composes the clone URL from the seat. | GitHub keeps exactly `owner/repo`. A malformed slug is `null`, with the forge's shape in the reason. | method JSDoc | unit: nested group accepted, no `cloneUrl` sent, GitHub unchanged |
| Repositories card `repos` entries | `setRepos`; the seat-aware default (neomjs/neo-agent-brain#755) | A new entry is `{repoSlug}`, and a GitLab seat accepts nested groups. Existing entries are carried forward whole, `forge` included. | GitHub entries are unchanged. | card JSDoc | unit: GitLab entry shape; `forge` survives an add and a remove |

## Acceptance Criteria

- [ ] AC-1: Defining a GitLab seat sends `forge: 'gitlab'` and the instance origin, and its working repository as its slug. Against the Brain-bound registry, the readback confirms the seat and a clone URL on its instance (E2E, no real GitLab call).
- [ ] AC-2: A GitHub definition's intent is unchanged, and its repository entry is `{repoSlug}`; the readback's clone URL is unchanged (unit + E2E).
- [ ] AC-3: A nested GitLab group is accepted. A Brain refusal of the definition or the repository reaches the form's status in the Brain's words (unit + E2E).
- [ ] AC-4: A GitLab seat's Repositories card adds a repository as `{repoSlug}` and removes one through `setRepos`. Every carried-forward entry keeps its `forge` (unit, red-first: one GitLab entry, remove another row, `forge` survives in the intent).
- [ ] AC-5: Existing Accounts unit, component and E2E checks and the Darwin visuals pass. The visual input stamp agrees.

## Out of Scope

- The installed sitting against a real GitLab instance. It stays in neomjs/neo-agent-brain#684 (AC-2/AC-3); this leaf gives it a product path.
- Renaming the Brain's `githubUsername` field.
- An instance whose API lives outside `<host>/api`; none is known (neomjs/neo-agent-brain#684).
- An unknown-forge fallback on the config card (PR #445's review note).

## Avoided Traps

- **Deriving the forge from the host.** A self-hosted GitLab can sit on any host, and so can GitHub Enterprise. The operator names the forge; the Brain refuses a mismatch.
- **A second clone-URL authority.** The Brain's seat-aware verbs compose the clone URL for both forges (neomjs/neo-agent-brain#755), so the cockpit composes none, and its GitHub composition is deleted.

## Deltas after filing

- **Intake Stage 2 (2026-10-02).** The first draft had the cockpit compose `<forgeHost>/<slug>.git`. The Brain's `setRepo` / `setRepos` already hold the seat's forge and `forgeHost`, so the composition moved there (neomjs/neo-agent-brain#755). That deleted Fix 4 (`AgentDefinition.forgeHost`), because nothing reads it any more.

## Decision Record impact

None.

## Related

#245 (the deferral) · #442 / PR #445 (the pin and forge-aware card) · #407 (Repositories card) · neomjs/neo-agent-brain#684 (parity epic; its installed arms need this path) · neomjs/neo-agent-brain#755 (the seat-aware clone URL) · neomjs/neo-agent-brain#712, neomjs/neo-agent-brain#727, neomjs/neo-agent-brain#729.

Blocked by PR #445 (the pin and `AgentDefinition.forge`), and by an Institution Brain pin that carries neomjs/neo-agent-brain#755.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-02T14:01Z. No equivalent.
A2A in-flight sweep: last 30 messages, all read-states, to 14:00Z. No claim on GitLab seat definition.
MC sweep: `add-agent form GitHub only owner/repo cannot define a GitLab seat Accounts cockpit forgeHost clone URL`, 6 results. It found the 10-01 #684 filing ("Institution #245 only defers") and no decision against this.
Own-assignment sweep: #443, #436, #414, #11. None overlapping.
Exact: `gh search issues --owner neomjs "GitLab seat add-agent"`, 0 results; open remote branches: none on this scope.
Structure (1c): no new file. Edits stay in their owning modules (`AddAgentFlow`, `AddAgentForm`, `AgentReposContainer`, `AgentDefinition`).

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("GitLab seat add-agent form forge forgeHost nested group clone URL Repositories card cockpit parity")`


## Timeline

- 2026-10-02T14:02:41Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:02:42Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T14:02:42Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:02:42Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:02:52Z @neo-opus-grace added parent issue #684
- 2026-10-02T14:02:56Z @neo-opus-grace marked this issue as being blocked by #442
- 2026-10-02T14:06:16Z @neo-opus-grace cross-referenced by #755
- 2026-10-02T14:20:02Z @neo-opus-grace marked this issue as being blocked by #755
- 2026-10-02T14:22:49Z @neo-opus-grace cross-referenced by PR #756
- 2026-10-02T14:53:50Z @neo-opus-grace cross-referenced by PR #450

