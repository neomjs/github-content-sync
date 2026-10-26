---
id: 1286
title: Publish the Neural Link's MCP Registry entry from GitHub Actions
state: CLOSED
labels:
  - enhancement
  - ai
  - github_actions
assignees:
  - neo-opus-grace
createdAt: '2026-10-26T02:01:51Z'
updatedAt: '2026-10-26T12:27:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1286'
author: neo-opus-grace
commentsCount: 0
parentIssue: 20114
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-26T12:27:48Z'
---
# Publish the Neural Link's MCP Registry entry from GitHub Actions

## Context

`io.github.neomjs/neural-link` is to be listed in the MCP Registry (neomjs/neo#20114, graduated from D#20113): the `neural-link` manifest gets its `packages` entry, and the other five stay unpublished. `#1265` added the manifests. `#1283` gave `ai/mcp/server/neural-link/server.json` its npm `packages` entry, and the operator published `neo-neural-link@1.0.0` on 2026-10-26; npm staged it, and the maintainer's approval was pending at filing time.

The operator's interactive publish was then refused (mcp-publisher 1.8.4, 2026-10-26). The registry answered 403: "You have permission to publish: io.github.tobiu/*" and "The interactive 'mcp-publisher login github' flow cannot read organization roles; instead publish from GitHub Actions (which uses OIDC), or run 'mcp-publisher login github --token <classic PAT with the read:org scope>'."

## The Problem

The registry grants `io.github.neomjs/*` only to a credential that can read the organization role. That leaves two routes:
- a classic PAT with `read:org`, which is a long-lived operator credential;
- GitHub Actions OIDC from a `neomjs` repository, which needs no secret.

Publishing is an ops step that peers run, not the operator (the operator, 2026-10-26: "running middleware updates is the job of our ai peers"). So it belongs in CI.

## The Architectural Reality

- `ai/mcp/server/neural-link/server.json` is the one declaration. Its `name` is the package's `mcpName`, and `packages[0]` names `neo-neural-link` and its version; `ai/scripts/setup/packNeuralLink.mjs` reads both.
- The registry validates a publish against npm: the version in `packages[0]` must be live there with a matching `mcpName`, or the registry answers "Package validation failed". So the registry publish follows the npm release and cannot precede it.
- `.github/workflows/` has 13 workflows at `da24eba2`, none of them a publish. Sibling precedent: `neomjs/neo-agent-skills` `.github/workflows/publish.yml` (from CI, with provenance) and `neomjs/create-app` `npm-publish.yml` (`workflow_dispatch`, OIDC).
- Structure map at `da24eba2` (exit 0): the manifest's owning folder is `ai/mcp/server/neural-link`. The workflow is CI, outside the Agent OS map.
- `mcp-publisher publish [server.json]` takes the manifest's path, and `login github-oidc` is the Actions route (both from `--help` on 1.8.4).

## The Fix

Add `.github/workflows/publish-mcp-registry.yml`:
- `on: workflow_dispatch` only. It is dispatched once the npm version is live, because an earlier run fails package validation.
- `permissions: {contents: read, id-token: write}`.
- Install `mcp-publisher` v1.8.4 from the registry's GitHub release, and verify the tarball against its pinned sha256 (`5dc2fd41…`, the release asset's digest) before running it.
- Run `mcp-publisher login github-oidc`, then `mcp-publisher publish ai/mcp/server/neural-link/server.json`.

## Decision Record impact

none.

## Acceptance Criteria

- [ ] `.github/workflows/publish-mcp-registry.yml` publishes `ai/mcp/server/neural-link/server.json` and nothing else, on `workflow_dispatch`, with `id-token: write` and no secret.
- [ ] `mcp-publisher` is pinned to v1.8.4, and its tarball's sha256 is verified before it runs.
- [ ] Post-merge: after `neo-neural-link@1.0.0` is live on npm, a dispatched run publishes `io.github.neomjs/neural-link` 1.0.0, and `https://registry.modelcontextprotocol.io/v0/servers?search=io.github.neomjs` lists it. That query returned `count: 0` at filing time.

## Out of Scope

- Publishing `neo-neural-link` to npm from CI. Trusted publishing needs the operator's one-time npmjs.com configuration, and npm's staged release needs a maintainer's approval.
- The other five servers' manifests.

## Avoided Traps

- A classic PAT with `read:org` as a repository secret: a long-lived operator credential, for a step OIDC does without.
- Triggering on `server.json` changes merged to `dev`: the run would race the npm release and fail validation.
- `releases/latest`: an unpinned binary inside a publish job.

## Related

neomjs/neo#20114 (the epic), `#1265` (the manifests), `#1283` (the package).

Live latest-open sweep: the latest 20 open issues in `neo-agent-brain` at 2026-10-26T02:01Z; no equivalent found. A2A sweep: the last 30 messages, all read states; no overlapping claim. Memory Core sweep: one problem-noun query, 10 results, none recording a registry-auth decision. Own-assignment sweep: `#684` and `#54`, both unrelated.

Origin Session ID: 5348e5ef-cc4d-4b4b-ad6b-62a5f55bc651

Retrieval Hint: `query_raw_memories("MCP Registry org namespace publish GitHub Actions OIDC neural-link")`

## Timeline

- 2026-10-26T02:01:51Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-26T02:01:52Z @neo-opus-grace added the `enhancement` label
- 2026-10-26T02:01:52Z @neo-opus-grace added the `ai` label
- 2026-10-26T02:01:52Z @neo-opus-grace added the `github_actions` label
- 2026-10-26T02:02:30Z @neo-opus-grace added parent issue #20114
- 2026-10-26T02:05:29Z @neo-opus-grace cross-referenced by PR #1287
- 2026-10-26T12:27:48Z @tobiu referenced in commit `26d39c1` - "feat(neural-link): publish the MCP Registry entry from GitHub Actions (#1286) (#1287)

The registry grants the io.github.neomjs namespace only to a credential that can
read the organization role; GitHub Actions OIDC from this repository is that
credential. Dispatched from dev once the npm version server.json names is live."
- 2026-10-26T12:27:48Z @tobiu closed this issue

