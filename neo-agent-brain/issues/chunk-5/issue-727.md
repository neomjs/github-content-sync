---
id: 727
title: 'A GitLab seat runs gitlab-workflow with its own token, host and project'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T20:10:25Z'
updatedAt: '2026-10-02T11:37:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/727'
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
  - '[x] 712 A seat''s PAT is presented only to the forge host it was stored for'
blocking:
  - '[x] 729 A seat''s forge decides which workflow server it starts with'
closedAt: '2026-10-02T11:32:28Z'
---
# A GitLab seat runs gitlab-workflow with its own token, host and project

## Context

This is a leaf of #684 (the Fleet's GitLab parity). It was re-scoped in [comment 5937817911](https://github.com/neomjs/neo-agent-brain/issues/684#issuecomment-5937817911) and is numbered leaf 4 there. **It lands before leaf 3** (the forge decides which workflow server starts on): flipping a GitLab seat's default to `gitlab-workflow` while that server is unsupported would refuse every GitLab define, and a Start would crash on the missing envelope entry below. With this leaf, an operator can enable `gitlab-workflow` explicitly. Leaf 3 then makes it the GitLab default.

It builds on #712 (the row's `forge` and `forgeHost`) and #710 (the working repository's forge and nested slug).

## The Problem

A seat bound to a GitLab instance cannot run `gitlab-workflow`:
- **The plan refuses it.** The descriptor already declares the runtime env (`NEO_GITLAB_HOST`, `NEO_GITLAB_PAT`, `NEO_GITLAB_PROJECT`, `NEO_AGENT_IDENTITY`), its required and secret env, and an `unsupportedReason` (`managedAgentWorkspacePlan.mjs` :67-72 at `dev@e6f0bb0`). `mcpDeclarationRefusal` turns that reason into a refusal.
- **Nothing injects the three values.** The spawn puts every seat's PAT under `credentialEnvVar` (`GH_TOKEN`; `FleetLifecycleService.mjs` :591). A GitLab PAT under `GH_TOKEN` is wrong for both servers.
- **A latent crash.** `resolveResidentMcpEnvironment` (:1787-1815) maps each enabled server to its config module's `exportEnv`, and the map has no `gitlab-workflow` entry. Only the plan's refusal keeps this from throwing at Start.
- **Kimi and OpenCode render fixed server lists** that name GitHub only (`generateKimiSeatConfig.mjs` :15, `generateOpenCodeSeatConfig.mjs` :17).

## The Architectural Reality

- **ADR 0019 §10.7:** a resident stdio child receives its plane slots through `ConfigProvider.exportEnv()` at Start, as process serialization. A per-seat value is not a config value. If the Fleet exported the host's own `gitlab.token` / `hostUrl` / `projectPath` leaves into a seat, that would be B5 and a credential leak (the operator's GitLab PAT in a seat's env). The exclusion is therefore part of the contract, not an optimization.
- **Every per-seat value is already known at Start:** the PAT (the registry, bound to `forgeHost` by #712), `NEO_GITLAB_HOST` = the row's `forgeHost`, and `NEO_GITLAB_PROJECT` = the working repository's slug when that repository is on the seat's GitLab instance. A GitLab seat whose working repository is elsewhere leaves it unset (the server's own empty default) until leaf 3's `setRepo` rule makes that state unreachable.
- **The reserved-slot guard** (:485-493) keeps credential classes in distinct env slots, and a launch env may not name a reserved slot.

## The Fix

1. **Per-seat env.** For a GitLab seat, the spawn sets `NEO_GITLAB_PAT` (its PAT), `NEO_GITLAB_HOST` (`forgeHost`) and `NEO_GITLAB_PROJECT` (the working slug, only for a clone on that instance), and no `GH_TOKEN`. A GitHub seat is unchanged. All three join the reserved slots.
2. **Envelope.** `resolveResidentMcpEnvironment` maps `gitlab-workflow` to its config module and excludes the per-seat names from `exportEnv`. The hard-coded exclusion list becomes a descriptor field (`seatEnv`), so the same rule covers `GH_TOKEN`.
3. **Plan.** The descriptor's `unsupportedReason` goes. The plan refuses `gitlab-workflow` on a seat that is not bound to GitLab, and on Kimi and OpenCode (their fixed lists), each with its reason.
4. **Rendering.** Codex, Claude Code and Claude Desktop render from the descriptor, so each gets a proof arm, not new code.

## Contract Ledger

*Backfilled during review (PR #742, Round 1, RA-2). Evidence is L2 (unit); the installed arm is AC-4's.*

| Target surface | Source of authority | Behavior | Failure / fallback | Docs | Evidence |
|---|---|---|---|---|---|
| A GitLab seat's spawn env (`FleetLifecycleService.start`) | the registry row's `forge` + `forgeHost` (#712) and the seat's stored PAT | `NEO_GITLAB_PAT` = the PAT, `NEO_GITLAB_HOST` = `forgeHost`; no `GH_TOKEN` / `GITHUB_TOKEN` | a GitHub seat's env is unchanged | JSDoc | lifecycle: "a GitLab seat starts with its own PAT, instance and project…" |
| `NEO_GITLAB_PROJECT` (`gitlabProjectOf` → `isOnInstance`) | the working repository's `repoSlug` + `cloneUrl` | named only for a clone on the bound instance: an `https` clone by exact origin (scheme, host, port; IPv6 normalized), an `ssh` / scp-style clone by host, since SSH has its own port | absent, so `gitlab-workflow` has no default project | JSDoc | lifecycle: the eleven-case instance matrix |
| Reserved seat slots | the `gitlab-workflow` descriptor's `seatEnv` | a launch env naming any of the three refuses the start; so does a resident envelope carrying one | the start refuses before spawning | JSDoc | lifecycle: "SECURITY: a launch env cannot pre-load a GitLab seat slot", plus the envelope-refusal arm |
| `seatEnv` on every descriptor, and the `exportEnv` exclusion (`SEAT_ENV`) | `MANAGED_WORKSPACE_MCP_SERVER_DESCRIPTORS` plus the identity slot | the resident envelope never exports a per-seat name from the host's own config; the union replaces the hard-coded list (a superset of it) | n/a (a static set) | JSDoc at the descriptors | lifecycle: "the default producer gives the GitLab workflow server its plane slots and never a GitLab seat value" |
| Plan intent `agent.forge` (`managedAgentWorkspacePlan`) | the registry row | present only for a GitLab seat | omitted = GitHub; a GitHub plan's shape is unchanged | JSDoc | prepare: the GitHub plan arms, unchanged |
| `MCP_SERVERS` catalog `forge` field | `src/fleet/contract/mcpServers.mjs` | each workflow entry names its forge | none (#729 derives the defaults from it) | JSDoc | contract spec |
| Renderer admission (`mcpDeclarationRefusal(forge)`) | the descriptor and the harness's renderer | `gitlab-workflow` is admitted on a GitLab seat for Codex, Claude Code and Claude Desktop, and refused on a GitHub seat and on Kimi / OpenCode, each with its reason | refused before any write | JSDoc | prepare: "the GitLab workflow server runs only on a seat bound to GitLab, and only where the harness renders it" |

Boundaries: the default flip (`github-workflow` off for a GitLab seat) is #729; the installed arm is AC-4, owned by open #684.

## Acceptance Criteria

- [x] AC-1: a GitLab seat's spawn env carries `NEO_GITLAB_PAT`, `NEO_GITLAB_HOST` and `NEO_GITLAB_PROJECT`, and no `GH_TOKEN`. A GitHub seat's env is unchanged. A launch env naming any of the three refuses (unit).
- [x] AC-2: with `gitlab-workflow` enabled, the resident envelope carries the plane slots and never the host's own `NEO_GITLAB_*` values (unit, with a host value set in the export source).
- [x] AC-3: the plan admits an enabled `gitlab-workflow` on a GitLab seat for Codex, Claude Code and Claude Desktop, and each renderer forwards or references the three names. It refuses on a GitHub seat and on Kimi and OpenCode, each with its reason (unit).
- [ ] AC-4 `[L4-deferred — operator handoff needed]` (post-merge, installed): a GitLab seat's `gitlab-workflow` reads its project on a self-hosted instance. This is #684's installed arm, so the residual owner is #684.

AC-1 to AC-3 were delivered by PR #742, merged as `db9918e` on 2026-10-02 after @neo-gpt's Round-2 approval at `9995ca9`. Round 1 named the project only for a clone on the exact instance, and backfilled the Contract Ledger. AC-4 stays with #684.

## Out of Scope

- The default flip (leaf 3).
- The cockpit's GitLab form and catalog calls (Institution, Ada).
- A seat with repositories on both forges.
- A separate API-URL leaf.

## Decision Record impact

`aligned-with ADR 0019` (§10.7): per-seat values are injected by the Fleet, and the server keeps reading its own leaves. ADR 0019 was read in full this session.

## Related

Parent #684 · #712 (blocks) · #710 · #659 · #669 / #692 (the Claude Desktop Code-tab rows) · Institution #245

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T20:10:00Z. No equivalent; #712 and #684 are adjacent.
- Exact: `gh search issues --owner neomjs` for "gitlab-workflow unsupportedReason", "NEO_GITLAB_PAT seat" and "GitLab workflow server seat": none.
- MC sweep: "GitLab seat cannot run gitlab-workflow; NEO_GITLAB_PAT injection; resident MCP envelope exportEnv per-seat values excluded", 5 results. They are #684's own record and #659's design (the plan treats a disabled row as renderable and enabling as a separate contract); no prior decision against this.
- A2A: the 12 newest, all read-states, to 20:00:44Z. No claim on this scope.
- Own-assignment: #684, #710, #712 (the stack this builds on); none carries the injection.
- Structure map: `ai/services/fleet`. No new module.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "GitLab seat gitlab-workflow NEO_GITLAB_PAT NEO_GITLAB_HOST NEO_GITLAB_PROJECT exportEnv seatEnv unsupportedReason resolveResidentMcpEnvironment"

🖖 Grace (Claude Opus 5.5, Claude Code)




## Timeline

- 2026-10-01T20:10:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T20:10:26Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T20:10:27Z @neo-opus-grace added the `ai` label
- 2026-10-01T20:10:27Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T20:10:36Z @neo-opus-grace added parent issue #684
- 2026-10-01T20:10:38Z @neo-opus-grace marked this issue as being blocked by #712
### @neo-opus-grace - 2026-10-01T20:22:20Z

**Built** on `grace/727-gitlab-workflow-seat` at `93dac53`, stacked on #712 (`bff457e`) and #711 (`100c535`). The PR opens once both merge.

- **Spawn:** a GitLab seat's PAT goes to `NEO_GITLAB_PAT`, beside `NEO_GITLAB_HOST` (`forgeHost`) and `NEO_GITLAB_PROJECT`. The project is set only when the working repository is on that instance; the seat gets no `GH_TOKEN`. All three join the reserved slots.
- **Envelope:** `resolveResidentMcpEnvironment` maps `gitlab-workflow` to its config module. Every descriptor's `seatEnv` (now the one source for the formerly hard-coded list) is excluded from `exportEnv`, so the host's own GitLab values never reach a seat. As a second guard, a reserved slot arriving in the envelope refuses the start.
- **Plan:** `unsupportedReason` is gone. `mcpDeclarationRefusal` takes the seat's `forge`, refuses `gitlab-workflow` on a seat not bound to GitLab and on Kimi and OpenCode (their fixed lists), and the plan's agent intent carries `forge` for a GitLab seat only. The catalog entries gained `forge` for leaf 3 to use.
- **Evidence:** seven new or changed arms, each red on #712's sources and green on this head. The three lifecycle arms were run one by one, because a serial describe had hidden them as "did not run" behind the first red. Fleet suites: 1176 passed, 2 skipped, and only the 5 environment-only reds that fail the same way on `dev`.

🖖 Grace (Claude Opus 5.5, Claude Code)


- 2026-10-01T20:23:48Z @neo-opus-grace cross-referenced by #729
- 2026-10-01T20:24:01Z @neo-opus-grace marked this issue as blocking #729
- 2026-10-01T20:27:14Z @neo-opus-ada cross-referenced by #730
- 2026-10-01T20:29:33Z @neo-opus-grace referenced in commit `5298471` - "feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727)

The spawn puts a GitLab seat's PAT, instance and working project in the GitLab slots instead of GH_TOKEN; the resident envelope gains the gitlab-workflow module and excludes every per-seat value from exportEnv; the plan admits the server on a GitLab seat and refuses it elsewhere and on Kimi/OpenCode."
- 2026-10-01T20:32:11Z @neo-opus-grace cross-referenced by #710
- 2026-10-01T20:33:10Z @neo-opus-grace referenced in commit `51eb97d` - "feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727)

The spawn puts a GitLab seat's PAT, instance and working project in the GitLab slots instead of GH_TOKEN; the resident envelope gains the gitlab-workflow module and excludes every per-seat value from exportEnv; the plan admits the server on a GitLab seat and refuses it elsewhere and on Kimi/OpenCode."
- 2026-10-01T20:44:52Z @neo-gpt cross-referenced by PR #711
- 2026-10-02T08:14:43Z @neo-opus-grace referenced in commit `3f61ba3` - "feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727)

The spawn puts a GitLab seat's PAT, instance and working project in the GitLab slots instead of GH_TOKEN; the resident envelope gains the gitlab-workflow module and excludes every per-seat value from exportEnv; the plan admits the server on a GitLab seat and refuses it elsewhere and on Kimi/OpenCode."
- 2026-10-02T08:48:13Z @neo-opus-grace referenced in commit `b64e019` - "feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727)

The spawn puts a GitLab seat's PAT, instance and working project in the GitLab slots instead of GH_TOKEN; the resident envelope gains the gitlab-workflow module and excludes every per-seat value from exportEnv; the plan admits the server on a GitLab seat and refuses it elsewhere and on Kimi/OpenCode."
- 2026-10-02T09:19:02Z @neo-opus-grace referenced in commit `39d4baa` - "feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727)

The spawn puts a GitLab seat's PAT, instance and working project in the GitLab slots instead of GH_TOKEN; the resident envelope gains the gitlab-workflow module and excludes every per-seat value from exportEnv; the plan admits the server on a GitLab seat and refuses it elsewhere and on Kimi/OpenCode."
- 2026-10-02T09:19:05Z @neo-opus-grace cross-referenced by PR #742
- 2026-10-02T11:07:14Z @neo-opus-grace referenced in commit `9995ca9` - "fix(fleet): a GitLab seat's project is named only for a clone on its exact instance (#727)

An https clone must carry the bound instance's exact origin, so the same host on another port no longer names its project and a bracketed IPv6 instance now does. ssh and scp-style clones still match by host, since SSH has its own port."
- 2026-10-02T11:32:28Z @tobiu referenced in commit `db9918e` - "feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727) (#742)

* feat(fleet): a GitLab seat runs gitlab-workflow with its own token, host and project (#727)

The spawn puts a GitLab seat's PAT, instance and working project in the GitLab slots instead of GH_TOKEN; the resident envelope gains the gitlab-workflow module and excludes every per-seat value from exportEnv; the plan admits the server on a GitLab seat and refuses it elsewhere and on Kimi/OpenCode.

* fix(fleet): a GitLab seat's project is named only for a clone on its exact instance (#727)

An https clone must carry the bound instance's exact origin, so the same host on another port no longer names its project and a bracketed IPv6 instance now does. ssh and scp-style clones still match by host, since SSH has its own port."
- 2026-10-02T11:32:28Z @tobiu closed this issue
- 2026-10-02T11:37:50Z @neo-opus-grace cross-referenced by PR #749
- 2026-10-02T12:46:20Z @neo-opus-grace cross-referenced by #684
- 2026-10-02T14:02:41Z @neo-opus-grace cross-referenced by #448

