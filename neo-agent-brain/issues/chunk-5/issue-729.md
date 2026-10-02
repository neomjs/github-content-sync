---
id: 729
title: A seat's forge decides which workflow server it starts with
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T20:23:47Z'
updatedAt: '2026-10-02T12:46:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/729'
author: neo-opus-grace
commentsCount: 1
parentIssue: 684
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 727 A GitLab seat runs gitlab-workflow with its own token, host and project'
blocking: []
closedAt: '2026-10-02T12:44:04Z'
---
# A seat's forge decides which workflow server it starts with

## Context

This is a leaf of #684. It was re-scoped in [comment 5937817911](https://github.com/neomjs/neo-agent-brain/issues/684#issuecomment-5937817911) and lands **after #727**, which made `gitlab-workflow` runnable on a GitLab seat. Since #698, `github-workflow` starts on for every seat. A GitLab seat therefore starts it with no GitHub token, and `gitlab-workflow` stays off unless the operator enables it by hand. This leaf makes the seat's forge decide which workflow server starts.

## The Problem

- **The defaults are forge-blind.** `MCP_SERVERS` carries each workflow server's `forge` (#727), but `defaultEnabled` ignores it.
- **Resolution and normalization must agree on one catalog per seat.** `normalizeMcpOverrides` drops a value equal to its default. If a seat were normalized against one catalog and resolved against another, an operator's explicit toggle would silently flip.
- **`setRepo` accepts a working repository on any forge** (#710). A GitLab seat can name a GitHub working repository, which #727 then gives no project. A GitHub seat can name a GitLab one, and its PAT never reaches that repository.

## The Architectural Reality

- **The seam already exists.** `defaultMcpMatrix(catalog)`, `resolveMcpMatrix(overrides, catalog)` and `normalizeMcpOverrides(overrides, catalog)` all take the catalog, which defaults to `MCP_SERVERS`.
- **Brain call sites:**
  - `FleetRegistryService.defineAgent` and `configureAgent` (normalize, plus the refusal's resolve);
  - `FleetLifecycleService.resolveResidentMcpEnvironment`;
  - `prepareManagedAgentWorkspace` (its `resolveMatrix` seam).
- **The public definition carries `forge`** (#712), so the cockpit's `AgentConfigComponent` can pass the same catalog. That is Institution work, and @neo-opus-ada scoped it to after this leaf and a pin.
- **`setRepo` already reads the seat's row** for the collision rule (#711).

## The Fix

1. `mcpCatalogFor(forge = 'github')` in `src/fleet/contract/mcpServers.mjs` returns the catalog in which a workflow server defaults on only when its `forge` matches. For GitHub its values are today's `MCP_SERVERS` defaults: an equal, separately frozen catalog, and no consumer relies on its identity.
2. Every Brain call site above passes `mcpCatalogFor(<seat>.forge)`.
3. `setRepo` refuses a working repository whose forge is not the seat's, and, on a GitLab seat, a clone URL that is not on `forgeHost`. The reason is written for the operator, because #724 shows it verbatim in Accounts.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `mcpCatalogFor(forge)`, a new export of `src/fleet/contract/mcpServers.mjs` | the catalog | The seat's forge's workflow server defaults on; the other forge's defaults off | `github`: today's `MCP_SERVERS` values | JSDoc | `mcpServers` unit arms |
| `setRepo` working-repository forge rule | `FleetManager` | Refuses a repository off the seat's forge or `forgeHost`, and writes nothing | none (refusal) | `setRepo` JSDoc | `FleetManager.spec` arm |

## Acceptance Criteria

- [x] AC-1: `mcpCatalogFor('github')` carries today's catalog values. `mcpCatalogFor('gitlab')` turns `gitlab-workflow` on and `github-workflow` off, and leaves the core servers unchanged (unit).
- [x] AC-2: a GitLab seat defined with `mcpServers` omitted or `null` resolves `gitlab-workflow` on and `github-workflow` off, stores `null`, and starts with that matrix. An explicit override survives normalization on both `defineAgent` and `configureAgent` (unit).
- [x] AC-3: `setRepo` refuses each of these and writes nothing: a GitHub working repository on a GitLab seat, a GitLab one off the seat's `forgeHost`, and a GitLab one on a GitHub seat (unit).
- [x] AC-4: every existing GitHub arm passes unchanged.

## Out of Scope

- The cockpit's catalog calls and GitLab form (Institution, Ada).
- A seat whose repositories span both forges.
- The extra repositories' forge, which `setRepos` leaves free; they clone unauthenticated off the PAT's host.

## Decision Record impact

`none`: there is no config leaf. Aligned with #659's ruling that the forge follows the seat's repository.

## Related

Parent #684 · #727 (blocks) · #712 · #710 / PR #711 · #724 · #698 · #659 · Institution #245

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T20:23:06Z. No equivalent; #727, #712 and #710 are the stack this builds on.
- Exact: `gh search issues --owner neomjs` for "mcpCatalogFor", "workflow server default forge" and "working repository forge setRepo" found only #684, whose comment 5937817911 is this leaf's design.
- MC sweep: the #727 sweep this session ("GitLab seat cannot run gitlab-workflow ...") and #659's forge-follows-the-repository ruling; no conflicting decision.
- A2A: the 8 newest, all read-states, to 20:19:02Z. No claim on this scope. Ada scoped the Institution consumer to after this leaf.
- Own-assignment: #684, #710, #712, #727 (the stack); none carries the default flip.
- Structure map: `src/fleet/contract` + `ai/services/fleet`. No new module.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "mcpCatalogFor forge workflow server default GitLab seat github-workflow off setRepo working repository forge"

🖖 Grace (Claude Opus 5.5, Claude Code)




## Timeline

- 2026-10-01T20:23:48Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T20:23:48Z @neo-opus-grace added the `ai` label
- 2026-10-01T20:23:48Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T20:23:48Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T20:23:59Z @neo-opus-grace added parent issue #684
- 2026-10-01T20:24:01Z @neo-opus-grace marked this issue as being blocked by #727
- 2026-10-01T20:27:14Z @neo-opus-ada cross-referenced by #730
### @neo-opus-grace - 2026-10-01T20:29:02Z

**Built** on `grace/729-forge-workflow-default` at `f94899d`, stacked on #727 (`93dac53`) → #712 → #711. Its PR opens when the stack below merges.

- **Catalog:** `mcpCatalogFor(forge)` in the shared contract. GitHub's catalog is today's defaults; GitLab's flips the two workflow servers. An unknown forge throws, prototype names included. It is exported through `neo-agent-brain/fleet-contract` for the cockpit.
- **Call sites:** `defineAgent` and `configureAgent` normalize and resolve against the seat's catalog, as do `resolveResidentMcpEnvironment` and `prepareManagedAgentWorkspace`.
- **setRepo:** the working repository stays on the seat's forge, and on its GitLab instance. The reasons are written for the operator, and they keep the `FleetManager.setRepo:` prefix that #724 carries into Accounts. One clone-URL host parser (`provisionAgentRepo.cloneUrlHost`) now serves this rule and #727's project derivation.
- **Evidence:** four new arms, each red on #727's sources and green here; the registry spec fails at file load there, since `mcpCatalogFor` doesn't exist yet. Fleet suites: 1180 passed, 2 skipped, and only the 5 environment-only reds.

🖖 Grace (Claude Opus 5.5, Claude Code)

- 2026-10-01T20:29:35Z @neo-opus-grace referenced in commit `2b2e1d9` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog; setRepo keeps the working repository on the seat's forge and GitLab instance, with an operator-worded reason. One clone-URL host parser (provisionAgentRepo.cloneUrlHost) serves the rule and the GitLab project derivation."
- 2026-10-01T20:33:12Z @neo-opus-grace referenced in commit `e47d198` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog; setRepo keeps the working repository on the seat's forge and GitLab instance, with an operator-worded reason. One clone-URL host parser (provisionAgentRepo.cloneUrlHost) serves the rule and the GitLab project derivation."
- 2026-10-01T20:45:16Z @neo-opus-grace referenced in commit `548a31f` - "fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T08:14:45Z @neo-opus-grace referenced in commit `6205d4c` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog; setRepo keeps the working repository on the seat's forge and GitLab instance, with an operator-worded reason. One clone-URL host parser (provisionAgentRepo.cloneUrlHost) serves the rule and the GitLab project derivation."
- 2026-10-02T08:14:46Z @neo-opus-grace referenced in commit `db23043` - "fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T08:48:15Z @neo-opus-grace referenced in commit `237097f` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog; setRepo keeps the working repository on the seat's forge and GitLab instance, with an operator-worded reason. One clone-URL host parser (provisionAgentRepo.cloneUrlHost) serves the rule and the GitLab project derivation."
- 2026-10-02T08:48:15Z @neo-opus-grace referenced in commit `842939f` - "fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T09:19:05Z @neo-opus-grace cross-referenced by PR #742
- 2026-10-02T09:19:29Z @neo-opus-grace referenced in commit `bda8c21` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog; setRepo keeps the working repository on the seat's forge and GitLab instance, with an operator-worded reason. One clone-URL host parser (provisionAgentRepo.cloneUrlHost) serves the rule and the GitLab project derivation."
- 2026-10-02T09:19:30Z @neo-opus-grace referenced in commit `816ada3` - "fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T11:06:53Z @neo-opus-grace cross-referenced by #727
- 2026-10-02T11:12:29Z @neo-opus-grace referenced in commit `12a32b4` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog. setRepo keeps the working repository on the seat's forge and on its exact GitLab instance, with an operator-worded reason. One predicate, provisionAgentRepo.isOnInstance (an https clone by exact origin, an ssh or scp-style clone by host), serves both the rule and the GitLab project derivation."
- 2026-10-02T11:12:29Z @neo-opus-grace referenced in commit `2db717d` - "fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T11:37:47Z @neo-opus-grace referenced in commit `b122fdd` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog. setRepo keeps the working repository on the seat's forge and on its exact GitLab instance, with an operator-worded reason. One predicate, provisionAgentRepo.isOnInstance (an https clone by exact origin, an ssh or scp-style clone by host), serves both the rule and the GitLab project derivation."
- 2026-10-02T11:37:47Z @neo-opus-grace referenced in commit `e204c14` - "fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T11:37:50Z @neo-opus-grace cross-referenced by PR #749
- 2026-10-02T12:44:04Z @tobiu referenced in commit `a9dd22f` - "feat(fleet): a seat's forge decides which workflow server it starts with (#729) (#749)

* feat(fleet): a seat's forge decides which workflow server it starts with (#729)

mcpCatalogFor(forge) gives each forge its own defaults, and every Brain matrix call site resolves and normalizes a seat against its forge's catalog. setRepo keeps the working repository on the seat's forge and on its exact GitLab instance, with an operator-worded reason. One predicate, provisionAgentRepo.isOnInstance (an https clone by exact origin, an ssh or scp-style clone by host), serves both the rule and the GitLab project derivation.

* fix(fleet): setRepos words its refusals for the operator, who reads them in Accounts (#729)

Ada's defect-note 67ec73a3: two setRepos reasons named the setRepo API verb, and #724 now renders them verbatim in the cockpit."
- 2026-10-02T12:44:04Z @tobiu closed this issue
- 2026-10-02T12:46:20Z @neo-opus-grace cross-referenced by #684
- 2026-10-02T13:08:31Z @neo-gpt-emmy cross-referenced by #442
- 2026-10-02T14:02:41Z @neo-opus-grace cross-referenced by #448
- 2026-10-02T14:06:16Z @neo-opus-grace cross-referenced by #755
- 2026-10-02T14:22:49Z @neo-opus-grace cross-referenced by PR #756

