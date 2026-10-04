---
id: 684
title: 'The Fleet is the one Brain surface without GitLab: no token, host or clone auth per seat'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T13:28:13Z'
updatedAt: '2026-10-04T11:07:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/684'
author: neo-opus-grace
commentsCount: 3
parentIssue: null
subIssues:
  - '[x] 710 A seat''s repository records its forge, and a GitLab slug may name nested groups'
  - '[x] 712 A seat''s PAT is presented only to the forge host it was stored for'
  - '[x] 727 A GitLab seat runs gitlab-workflow with its own token, host and project'
  - '[x] 729 A seat''s forge decides which workflow server it starts with'
  - '[x] 448 The add-agent form defines a GitLab seat on its own instance'
  - '[x] 755 A GitLab seat''s repositories default to its own instance'
subIssuesCompleted: 6
subIssuesTotal: 6
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# The Fleet is the one Brain surface without GitLab: no token, host or clone auth per seat

## Context

**The goal is parity: the Fleet is the one Brain surface without GitLab.** The operator, 2026-10-01: the Brain's GitLab support is proven in production use (a self-hosted instance, through MCP and KB ingestion), so the Fleet Manager should match it. GitLab is often self-hosted, so a seat needs the instance's host and API URL plus an access token.

Today the cockpit offers GitHub agents only (Institution #245 defers GitLab "until the Brain's GitLab credential injection" exists), and `gitlab-workflow` carries `unsupportedReason: 'FleetLifecycleService has no GitLab credential injection contract'`. The plane token, `NEO_MCP_REMOTE_TOKEN`, is already either a GitHub or a GitLab PAT (`auth.mode` `github-pat` | `gitlab-pat`, #231). #659 records the default this ticket extends: the forge follows the seat's repository.

## The Problem

A seat whose repository lives on GitLab cannot be provisioned or launched:

- The registry holds one credential per agent, documented and enforced as its GitHub PAT (`FleetRegistryService.defineAgent`).
- The clone presents that credential only to `https://github.com` (`provisionAgentRepo.mjs` `gitCloneCommand`); any other remote clones without auth, so a private GitLab repository fails.
- The plan refuses an enabled `gitlab-workflow`, and nothing injects `NEO_GITLAB_*` into a seat.

## The Architectural Reality

- The server side already supports a self-hosted instance (`ai/mcp/server/gitlab-workflow/configBase.mjs`): `gitlab.hostUrl` ← `NEO_GITLAB_HOST` (URL, default `https://gitlab.com`), `gitlab.token` ← `NEO_GITLAB_PAT`, `gitlab.projectPath` ← `NEO_GITLAB_PROJECT`. `GitLabClient` derives its endpoint as `${hostUrl}/api/graphql`, so an instance under a path (`https://host/gitlab`) works by putting the path in the host URL.
- The operator's names map onto that: `GITLAB_HOST` → `NEO_GITLAB_HOST`, `GITLAB_ACCESS_TOKEN` → `NEO_GITLAB_PAT`, `GITLAB_API_URL` → derived. A separate API-URL leaf is needed only for an instance that serves its API outside `<host>/api`; none is known yet.
- The Brain's other GitLab-capable surfaces, which the Fleet reuses rather than duplicates:
  - KB tenant-repo ingestion mirrors any git host. `ai/services/knowledge-base/helpers/gitMirror.mjs` authenticates HTTPS through a transient askpass environment and SSH through a key reference, from the shared credential grammar in `tenantRepoAccessContract.mjs`, which also gives `assertCleanCloneUrl` and `deriveRepoSlugFromCloneUrl` (the source of `NEO_GITLAB_PROJECT`).
  - Plane auth's `gitlab-pat` mode validates against a self-managed API (`NEO_AUTH_GITLAB_API_BASE_URL`).
- The GitHub path to mirror: the registry's encrypted credential store, `startAgentProvisioned` resolving the PAT once before any checkout, the spawn injecting it as `GH_TOKEN`, and per-harness rendering (Codex `env_vars`; Claude Desktop through #669's Code-tab rows).

## The Fix

1. **A forge credential per seat.** The registry stores a GitLab PAT for a seat whose repository is on GitLab, encrypted like the GitHub PAT and never echoed. The "every agent holds its GitHub PAT" rule becomes "every agent holds its forge's PAT".
2. **Host and project from the repository.** `NEO_GITLAB_HOST` and `NEO_GITLAB_PROJECT` derive from the seat's clone URL (`https://gitlab.example.com/group/sub/project.git` → host and `group/sub/project`), with an explicit host override for an instance whose web and API hosts differ.
3. **Clone auth, reused.** The clone authenticates a GitLab host the way KB ingestion already does: the transient askpass environment and the shared credential grammar from `gitMirror.mjs` and `tenantRepoAccessContract.mjs`, scoped to that seat's host, never through a config file. No second credential mechanism.
4. **Injection and rendering.** The spawn carries `NEO_GITLAB_PAT` (secret) and the host and project (non-secret). Each harness renders them its way. The plan's `unsupportedReason` goes, and `mcpDeclarationRefusal` then admits `gitlab-workflow`.
5. **Two slots, not one.** `NEO_MCP_REMOTE_TOKEN` authenticates to the plane (`read_user`); the forge PAT authenticates to the forge API (`api`). An operator may enter the same GitLab PAT in both. The Fleet never reuses one for the other implicitly, because the plane would then hold a token wider than it needs.

## Acceptance Criteria

- [x] AC-1: a seat defined on a GitLab repository stores a GitLab PAT and never returns it; a GitHub seat is unchanged. *(#712 AC-1, PR #739.)*
- [ ] AC-2: the clone of a private GitLab repository authenticates with that PAT, to that host only. *(Unit evidence: #712 AC-3's `gitCloneCommand` matrix, PR #739. Open for the installed clone, #712 AC-4.)*
- [ ] AC-3: the seat's `gitlab-workflow` reaches the self-hosted instance from the derived host and project (unit arms per harness renderer; one installed arm against a real instance). *(Unit evidence: #727 AC-1–3, PR #742, and #729's forge defaults, PR #749. Open for the installed arm, #727 AC-4.)*
- [x] AC-4: `NEO_MCP_REMOTE_TOKEN` and the forge PAT stay separate slots. *(`FleetLifecycleService.spec`: a repository PAT cannot take the plane slot, plus #727 AC-2's resident envelope.)*

## Out of Scope

- The cockpit's GitLab form (Institution #245 consumes this).
- A seat with repositories on both forges (rare; a follow-up if it appears).
- A separate API-URL leaf until an instance needs it.

## Decision Record impact

`aligned-with ADR 0019`: host and project are per-seat values the Fleet injects into the child; the server keeps reading its leaves.

## Related

#659 (GitHub on by default; the forge follows the repository), Institution #245, #231 (plane PAT modes), #571 (seat layout), #682 (multiple repositories per seat).

## Sweeps

Live latest-open sweep: latest 20 open Brain issues at 2026-10-01T13:27:29Z, no equivalent. `gh search issues --owner neomjs` for "GitLab credential injection", "gitlab-workflow Fleet credential", "NEO_GITLAB_PAT Fleet", "gitlab PAT seat": only Institution #245's deferral and #659. MC sweep ("GitLab agent cannot be launched by the Fleet; self-hosted GitLab host and personal access token per seat; plane login takes a GitHub or GitLab PAT"), 6 results: plane PAT modes confirmed; no decision on the Fleet side. Own-assignment sweep: #659 is adjacent, nothing overlapping. Structure map (this session, exit 0): `ai/services/fleet` and `ai/mcp/server/gitlab-workflow`; no new module required.

Owner: @neo-opus-grace (taken 2026-10-01 once the seat path cleared). All six leaves are merged: #710, #712, #727, #729, #755 (through #756) and neomjs/neo-agent-institution#448 (through neomjs/neo-agent-institution#450, 2026-10-02), so the cockpit can now define a GitLab seat end to end. What remains is one installed sitting against a real GitLab instance, which covers AC-2 and AC-3 (#712 AC-4 and #727 AC-4 name this epic as their residual owner). The sitting needs an installed package carrying Institution `cf4b54f` or later, which pins Brain `447d96e`. From the operator it needs a GitLab instance, a private test project and an `api` PAT.

Body revised 2026-10-01 ~13:40Z: framed as parity, and the clone fix reuses KB ingestion's credential code (operator input).

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "Fleet GitLab seat self-hosted NEO_GITLAB_HOST NEO_GITLAB_PAT clone auth forge credential per seat"

🖖 Grace · @neo-opus-grace · Claude Opus 5.5 · Claude Code




## Timeline

- 2026-10-01T13:28:15Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T13:28:15Z @neo-opus-grace added the `ai` label
- 2026-10-01T13:28:15Z @neo-opus-grace added the `architecture` label
- 2026-10-01T13:28:15Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T13:30:55Z @neo-opus-grace changed title from **The Fleet cannot run a GitLab seat: no GitLab token, host or clone auth per seat** to **The Fleet is the one Brain surface without GitLab: no token, host or clone auth per seat**
- 2026-10-01T13:31:25Z @neo-fable-clio cross-referenced by #685
- 2026-10-01T13:32:10Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T14:25:59Z @neo-opus-grace assigned to @neo-opus-grace
### @neo-opus-grace - 2026-10-01T14:27:29Z

**Claimed, with intake findings and a split.** Read at Brain `dev@741f9f3`.

**What the code says**
- **Registry:** `credentials.enc` already holds `{agentId: pat}`. The bytes are forge-neutral; only the docs say "GitHub PAT". AC-1 needs no new store.
- **Injection:** `FleetLifecycleService` puts every seat's PAT under `credentialEnvVar` (`GH_TOKEN`, `:610`). A GitLab seat needs `NEO_GITLAB_PAT` there instead, plus the non-secret `NEO_GITLAB_HOST` / `NEO_GITLAB_PROJECT`, inside the env-key distinctness check (`:517-529`).
- **Plan:** dropping `gitlab-workflow`'s `unsupportedReason` (`managedAgentWorkspacePlan.mjs:70`) admits Codex and Claude Code. Claude Desktop keeps refusing its secret env until #669, exactly like `github-workflow`.
- **Clone:** `gitCloneCommand` scopes its credential helper to `https://github.com` (`provisionAgentRepo.mjs:34-41`).
- **A blocker this body missed: nested GitLab groups cannot be registered.** `setRepo` (`FleetManager.mjs:411-435`) takes a two-segment `<owner>/<repo>` slug only (`assertRepoSlug`, `deriveAgentRepoPath.mjs:66-68`), and its clone-URL check demands that exact path, so `group/sub/project` is refused. The checkout path `<root>/<owner>/<repo>` needs a rule for more segments. #682's `setRepos` shares that helper.
- **The forge cannot be read off a self-hosted URL.** `gitlab.example.com` and a GitHub Enterprise host look alike, so the seat's repository needs an explicit `forge` (`github` by default, or `gitlab`).

**Fix 3, corrected.** This is the stage-2 check on my own prescription. The Fleet already has an env-only credential delivery: the inline, host-scoped git credential helper, with no file on disk. `gitMirror`'s askpass is a private KB function that writes a temp script. Importing it would couple fleet → knowledge-base for the weaker shape. The fix instead points the Fleet's own helper at the seat's forge host (username `oauth2` for GitLab), and reuses only the URL grammar (`assertCleanCloneUrl`). "No second mechanism" still holds; the Fleet's is the one.

**Split: three Brain PRs, each with its own sub filed when it starts**
1. **The forge model:** the repository's `forge`, GitLab slugs with nested groups and their checkout path, and host and project derived from the clone URL. Touches `setRepo` / `setRepos`. #682 shares the slug helper, so I'll agree the shape with Ada first.
2. **Clone auth** for the seat's forge host.
3. **Injection, plan and rendering:** `NEO_GITLAB_*` replaces `GH_TOKEN` for a GitLab seat, `unsupportedReason` goes, plus the Codex and Claude Code arms. The Claude Desktop arm rides #669.

Institution #245 consumes 1 and 3.

**Leaf 1 amended (Ada, 2026-10-01 15:03Z, checked against #683's head and #681).** It lands after #683 merges, not folded into it.
- **The reserve is a map.** Check segment 0 against `RESERVED_OWNERS` in `deriveAgentRepoPath.mjs` (`harness`, plus #681's `memory`), not against a `harness` literal.
- **"Listed twice" becomes path containment.** Today `setRepos` compares slug strings, which is enough only while every slug has two segments. With nested groups, `acme/tools/cli` would clone inside `acme/tools`. GitHub and GitLab `acme/tools`, or gitlab.com and a self-hosted `group/project`, would share one directory. So refuse over the seat's derived checkout paths (the working repo plus the extras): none may equal or contain another.
- **A GitLab entry needs an explicit clone URL.** `repoCoordinates`' `https://github.com/<slug>.git` default stays GitHub-only, for the same reason the forge is recorded: the host cannot be derived.

🖖 Grace (Claude Opus 5.5, Claude Code)


- 2026-10-01T17:52:15Z @neo-opus-grace cross-referenced by #710
- 2026-10-01T17:57:28Z @neo-opus-grace cross-referenced by PR #711
- 2026-10-01T18:10:28Z @neo-opus-grace cross-referenced by #712
### @neo-opus-grace - 2026-10-01T18:24:46Z

**Leaf status, and leaf 3 re-scoped after reading the code** (Brain `dev@2e46930`; ADR 0019 read in full for the env half).

- **Leaf 1:** #710 / PR #711. The forge and nested GitLab slugs; CI green, in review.
- **Leaf 2:** #712, branch `grace/712-forge-credential-host` (`7651f79`). `defineAgent` records a GitLab seat's `forge` and `forgeHost` beside the PAT, and the clone presents the PAT to that origin only. Red-first: 8 arms red on the pre-change sources, all green here. The PR opens once #711 merges, because it builds on `REPO_FORGES`.

The original leaf 3 ("injection, plan and rendering") misses three facts. They split it in two.

**Leaf 3: the seat's forge decides which workflow server starts on.** Since #698, `github-workflow` defaults on. On a GitLab seat it would start without a token, while `gitlab-workflow` stays off.
- `resolveMcpMatrix`, `normalizeMcpOverrides` and `defaultMcpMatrix` (`src/fleet/contract/mcpServers.mjs`) already take a `catalog` argument. So: each workflow entry carries its `forge`, and `mcpCatalogFor(forge)` turns on only the matching one. The GitHub catalog is exactly today's.
- The seat's catalog must reach every call site. These are `FleetRegistryService` (four) and `FleetLifecycleService.resolveResidentMcpEnvironment` (one), plus the Institution's `AgentConfigComponent` (three). Normalizing against one catalog and resolving against another would silently drop an operator's explicit toggle.
- `setRepo` refuses a working repository off the seat's forge, or off its `forgeHost` for a GitLab seat. The project then follows from the working slug.

**Leaf 4: a GitLab seat starts `gitlab-workflow` with its own token, host and project.**
- **Spawn env.** A GitLab seat's PAT goes to `NEO_GITLAB_PAT`, not `credentialEnvVar` (`GH_TOKEN`), beside `NEO_GITLAB_HOST` (`forgeHost`) and `NEO_GITLAB_PROJECT` (the working slug). All three join the reserved slots that a launch env may not name (`FleetLifecycleService.mjs:485-493`).
- **Resident envelope — a latent crash.** `resolveResidentMcpEnvironment` (`:1758-1786`) maps each enabled server to its config module's `exportEnv`, and that map has no `gitlab-workflow` entry. Only `unsupportedReason` keeps it from throwing today. The fix adds the module and excludes the three per-seat names from the exported set. Plane members still travel, as ADR 0019 §10.7 sanctions (process serialization through `exportEnv`). The host's own `NEO_GITLAB_*` must never reach a seat, which would be B5 and a credential leak. The exclusion becomes a descriptor field (`seatEnv`) instead of the hard-coded name list.
- **Plan and rendering.** The descriptor already declares the runtime, required and secret env (`managedAgentWorkspacePlan.mjs:67-73`); only `unsupportedReason` goes. Codex, Claude Code and Claude Desktop render from descriptors, so each needs a proof arm, not new code. Kimi and OpenCode have fixed server lists (`generateKimiSeatConfig.mjs:15`, `generateOpenCodeSeatConfig.mjs:17`), so the plan refuses `gitlab-workflow` there, with its reason.

Each leaf gets its own sub when it starts. Leaf 3 lands after #711 and #712. The Institution consumer is Ada's surface (Accounts, #245), and I'll ask her there.

🖖 Grace (Claude Opus 5.5, Claude Code)


- 2026-10-01T18:54:48Z @neo-opus-ada cross-referenced by #721
- 2026-10-01T19:31:34Z @neo-opus-grace cross-referenced by #725
- 2026-10-01T20:10:26Z @neo-opus-grace cross-referenced by #727
- 2026-10-01T20:23:48Z @neo-opus-grace cross-referenced by #729
- 2026-10-02T08:10:29Z @neo-opus-grace cross-referenced by PR #739
- 2026-10-02T09:19:05Z @neo-opus-grace cross-referenced by PR #742
- 2026-10-02T11:37:50Z @neo-opus-grace cross-referenced by PR #749
- 2026-10-02T13:08:31Z @neo-gpt-emmy cross-referenced by #442
- 2026-10-02T13:18:16Z @neo-opus-grace cross-referenced by #751
- 2026-10-02T13:34:04Z @neo-gpt-emmy cross-referenced by PR #445
- 2026-10-02T14:02:41Z @neo-opus-grace cross-referenced by #448
- 2026-10-02T14:06:16Z @neo-opus-grace cross-referenced by #755
- 2026-10-02T14:22:49Z @neo-opus-grace cross-referenced by PR #756
- 2026-10-02T14:36:49Z @neo-opus-grace cross-referenced by #760
- 2026-10-02T14:53:50Z @neo-opus-grace cross-referenced by PR #450
- 2026-10-03T18:25:41Z @neo-opus-grace cross-referenced by #822
- 2026-10-03T18:25:44Z @neo-opus-grace cross-referenced by #823
- 2026-10-04T10:57:20Z @neo-opus-ada cross-referenced by #571
- 2026-10-04T11:03:05Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-grace - 2026-10-04T11:07:05Z

## The two-forge follow-up: owner's disposition (2026-10-04)

The Out of Scope line "a seat with repositories on both forges (rare; a follow-up if it appears)" has appeared. Ada's and Vega's seats work on the org's GitHub and also on a forge outside the org ([#571 gap 11](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5971277938), [Vega's walk](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5978816452)).

**No new slot in this epic's store, and no leaf under it.** In both seats the second forge is not a repository the Fleet clones. Every declared repository is on GitHub. The second forge is reached by the seat's own tools, with three keys that live in the clone's `.env` today. This epic types one forge per seat because the Fleet acts on it: clone auth, the `gitlab-workflow` host, and the slot separation of AC-4. A forge the Fleet never acts on would gain nothing from a typed slot, and it would widen what the store holds. The operator's direction for gap 11 already gives such keys a home: a Fleet-written per-seat `.env` that keeps operator-added keys, which the Fleet never rewrites.

**One consequence the gap-11 contract must also cover.** After the move, the seat's harness MCP config is the Fleet's render. That render carries servers for the declared forge only: `managedAgentWorkspacePlan.mjs:326–328` refuses `gitlab-workflow` for any seat not defined with forge `gitlab`. So the second forge's MCP server entry has to survive the same way its keys do. The proposal is the same never-rewrite rule, extended from operator-added `.env` keys to operator-added MCP server entries. Otherwise the seat moves and silently loses the tools that use those keys. That belongs to #571's contract (Ada + Emmy), not to this epic.

**What would reopen it here:** a seat whose *declared* repositories span two forges. Then the Fleet clones from both, and AC-2/AC-3 apply per repository host. That would be the follow-up leaf, under this epic, with the same no-implicit-reuse rule as AC-4.

**The epic's own state:** all six leaves are merged. AC-2 and AC-3 still wait on their one installed sitting against a real GitLab instance, which needs the operator's instance, a private test project and an `api` PAT. It stays open for that sitting and is outside FM v1's rows.

🖖 Grace (Claude Opus 5.5, Claude Code) · owner



