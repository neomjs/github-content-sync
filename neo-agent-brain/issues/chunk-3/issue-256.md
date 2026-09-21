---
id: 256
title: The canonical compose silently builds Engine source
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-08-30T19:27:37Z'
updatedAt: '2026-08-30T20:32:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/256'
author: neo-opus-vega
commentsCount: 0
parentIssue: 12
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-30T20:32:08Z'
---
# The canonical compose silently builds Engine source

## Context

Leaf of #12 (AC-2: "Docker/compose/Caddy/build contexts resolve entirely inside Brain topology"), split out so the fix has a one-PR `Resolves` target while #12 remains the proof ledger. Found during #12's image-proof leg; independently verified by @neo-gpt-emmy on current source (#12's seam-decision comment).

## The Problem

All four service builds in `deploy/cloud/docker-compose.yml` pass `NEO_REF` and `NEO_REVISION` — none passes `NEO_REPO_URL`. The Dockerfile's default for that arg is `https://github.com/neomjs/neo.git` (the Engine), kept deliberately for standalone reuse. Net effect: **the canonical Brain compose definition builds Engine source unless a CLI-only `--build-arg` override is remembered** — and a green healthcheck never surfaces which source it proves (the Dockerfile's own docblock warns that health never proves revision; the same holds for source).

## The Architectural Reality

- `deploy/cloud/Dockerfile:13-15` — `ARG NEO_REPO_URL=https://github.com/neomjs/neo.git` (generic default, intentionally untouched).
- `deploy/cloud/docker-compose.yml` — four `build.args` blocks (kb-server, mc-server, orchestrator, fleet-server) carrying the #16087 single-pin mapping but no repo URL.
- Orchestrator and fleet-server are **profile-gated**: they are absent from un-profiled `config`/`up` renders, so any render-based audit must use `--profile '*'` or it undercounts (measured: 2 vs 4 arg occurrences).

## The Fix

One line per service in the canonical compose:

```yaml
NEO_REPO_URL: ${NEO_REPO_URL:-https://github.com/neomjs/neo-agent-brain.git}
```

The compose (Brain-owned, canonical) owns the Brain default; the Dockerfile keeps its generic default; the env override remains available in both directions. Implemented on branch `vega/12-canonical-brain-source` (`cc3d1fd`), proven end-to-end in #12's run-level receipts: canonical-definition build with **no CLI override** produced four images labeled `org.opencontainers.image.source = …neo-agent-brain.git`, revision `90d41ff90a61…` in file + label from the running cohort.

## Acceptance Criteria

- [ ] `docker compose --profile '*' -f docker-compose.yml -f docker-compose.local-agent-os.yml config` renders the Brain `NEO_REPO_URL` for all four service builds with no CLI arguments.
- [ ] A canonical build with only `NEO_REVISION` exported yields images whose OCI `source` label is the Brain repo.
- [ ] The env override still switches the source in either direction.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `deploy/cloud/docker-compose.yml` — `build.args.NEO_REPO_URL` on kb-server, mc-server, orchestrator, fleet-server (new key ×4) | this ticket; #12 AC-2 ("build contexts resolve entirely inside Brain topology") | canonical builds default the source to `https://github.com/neomjs/neo-agent-brain.git` | `${NEO_REPO_URL:-…}` — an exported env value wins over the default | in-file comment at the first arg site names the silent failure this closes | full-profile render counts 4× the Brain URL with zero CLI args; head-bound build receipts on PR #258 |
| `deploy/cloud/Dockerfile` — `ARG NEO_REPO_URL` (existing; semantics unchanged) | Dockerfile docblock (generic, standalone-reusable definition) | keeps the generic Engine default; consumed only when neither compose nor env supplies a value | n/a — it IS the last fallback tier | Dockerfile ARG block | unchanged in the diff (verified by the review's synthetic-merge render) |
| `NEO_REPO_URL` env var (operator-facing override) | #16087 single-pin convention (this extends the same operator surface) | switches the built source in either direction without file edits | absent ⇒ compose default (Brain) | #256 body + PR #258 AC-3 | explicit-value path exercised: CLI-supplied Brain URL build produced identical labels; Engine direction reachable by exporting the Engine URL |

## Out of Scope

- Changing the Dockerfile's generic default.
- The rehearsal/proof procedure itself (lives in #12).

## Related

- #12 (parent; receipts comment carries the end-to-end proof)
- #16087 (the single-pin contract these args extend)

Live latest-open sweep: latest 15 open checked 2026-08-30 ~19:30 UTC — no equivalent (#237 adjacent-but-different: retry semantics, not build source). A2A sweep: no overlapping claim; executed under #12's assignment per the seam decision.

Origin Session ID: b51135ac-ec45-4689-ac45-0e99fb073069

Retrieval Hint: "canonical compose NEO_REPO_URL Brain source build arg profile-gated"


## Timeline

- 2026-08-30T19:27:39Z @neo-opus-vega added the `bug` label
- 2026-08-30T19:27:39Z @neo-opus-vega added the `ai` label
- 2026-08-30T19:28:07Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-30T19:31:06Z @neo-opus-vega cross-referenced by PR #258
- 2026-08-30T20:32:09Z @tobiu closed this issue
- 2026-08-30T20:32:09Z @tobiu referenced in commit `9947f10` - "Merge pull request #258 from neomjs/vega/12-canonical-brain-source

fix(deploy): the canonical compose builds from Brain topology (#256)"
- 2026-08-30T22:09:35Z @neo-gpt-emmy cross-referenced by #184

