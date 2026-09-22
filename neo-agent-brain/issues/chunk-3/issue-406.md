---
id: 406
title: Deploy defaults and guides still pin the Agent OS to the Engine repo
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - build
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-21T14:18:01Z'
updatedAt: '2026-09-22T09:11:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/406'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-22T09:11:59Z'
---
# Deploy defaults and guides still pin the Agent OS to the Engine repo

## Context

The Agent OS source moved from `neomjs/neo` to `neomjs/neo-agent-brain` on 2026-08-26/27 (Brain `11552b0` received it; Engine `c623b2f63c` removed `ai/`). PR #258 (#256) moved the four canonical compose builds onto the Brain repository. Two defaults and four documented commands did not move with them, and all six still point the build at the Engine. Found 2026-09-21 while tracing why the live cohort runs Engine `467fd122f3` (#237's open AC, gated on #253).

## The Problem

An operator following the documented deploy path today gets a hard build failure, and the message does not name the cause. Measured at 2026-09-21T14:14Z:

- `PipelineWiring.md:56` says `export NEO_REVISION=$(git ls-remote https://github.com/neomjs/neo.git dev | cut -f1)`. That resolved to Engine `a2e5b33cde2dd510e8cf67acc04209ec13e5c5e7`.
- Compose (`deploy/cloud/docker-compose.yml:84/223/392/626`) fetches `NEO_REF` from its own default, `https://github.com/neomjs/neo-agent-brain.git`. Fetching that Engine SHA from the Brain remote: `fatal: remote error: upload-pack: not our ref a2e5b33c…`. The Brain remote advertises 0 refs containing it.
- `ai/examples/cloud-deployment/deploy-pipeline.sh:124` defaults `NEO_REPO_URL` to `https://github.com/neomjs/neo.git` and resolves the selector against it (`:143`, `:177`), then exports only `NEO_REVISION` (`:206`) — so the script itself manufactures the same Engine-SHA-into-Brain-fetch mismatch on every run that does not set `NEO_REPO_URL`.
- `deploy/cloud/Dockerfile:15` defaults `ARG NEO_REPO_URL=https://github.com/neomjs/neo.git`; its header comment (`:11`) still describes it as "neo repository (MIT-licensed) to clone from". A bare `docker build` fetches the Engine and then the builder stage runs `node ./ai/scripts/setup/initServerConfigs.mjs` with `SERVICE_ENTRYPOINT=ai/…`: Engine `origin/dev` has 0 entries under `ai/` (`git ls-tree origin/dev ai/daemons/orchestrator/daemon.mjs` → nothing), so that default can no longer produce any image.

It fails closed, which is the only reason it has not shipped a wrong image. It still means the documented path cannot deploy the Brain without an undocumented variable, and the reference pipeline script cannot deploy it at all.

## The Architectural Reality

Six surfaces name the Engine as the Agent OS source; all live in the Brain repository:

| surface | line | what it does today |
|---|---|---|
| `deploy/cloud/Dockerfile` | `:15` (`ARG NEO_REPO_URL=…/neo.git`), `:11` comment | default source for the `source-git` stage |
| `ai/examples/cloud-deployment/deploy-pipeline.sh` | `:124` | default remote the selector is resolved against |
| `learn/agentos/cloud-deployment/PipelineWiring.md` | `:56`, `:180` | the operator's documented pin command |
| `learn/agentos/cloud-deployment/Day0Tutorial.md` | `:57` | same pin command, four lines after `:43` clones `neo-agent-brain.git` |
| `ai/scripts/lifecycle/local-agent-os/README.md` | `:125` | same pin command for the local plane |

The compose files are already correct (#256 / PR #258): `NEO_REPO_URL: ${NEO_REPO_URL:-https://github.com/neomjs/neo-agent-brain.git}` on all four Neo-derived services. The Dockerfile's `#16635` guard (full-SHA-only `NEO_REF`) and the `/app/.neo-revision` integrity gate are unaffected and stay.

**A prior decision is being reversed here, on purpose.** #256 (mine) scoped the Dockerfile default out as a "generic default, intentionally untouched … kept deliberately for standalone reuse". That premise is dead: the builder stage's `ai/` paths do not exist in the Engine tree, so the "generic" default has no tree it can build. A default that can never succeed is not generality, it is a trap with a delay.

## The Fix

1. `deploy/cloud/Dockerfile:15` → `ARG NEO_REPO_URL=https://github.com/neomjs/neo-agent-brain.git`; rewrite the `:11` comment to say what it is: the Agent OS repository the image packages.
2. `ai/examples/cloud-deployment/deploy-pipeline.sh:124` → same default, so the selector resolves against the repository compose will fetch from.
3. The four documented pin commands (`PipelineWiring.md:56`, `:180`; `Day0Tutorial.md:57`; `local-agent-os/README.md:125`) → `git ls-remote https://github.com/neomjs/neo-agent-brain.git dev | cut -f1`.
4. One unit spec beside `test/playwright/unit/deploy/PackageBoundary.spec.mjs` that reads exactly these deploy surfaces and asserts none names `neomjs/neo.git` as a source — scoped to the deploy surfaces, never to `ai/` at large (see Avoided Traps).

No new `.mjs` runtime file. Structure-map gate run (`npm run ai:structure-map -- --files --loc`): `ai/examples/cloud-deployment` is the owning folder of the script; the spec joins the existing `test/playwright/unit/deploy/` siblings. Placement N/A otherwise.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ARG NEO_REPO_URL` (`deploy/cloud/Dockerfile:15`) | this ticket; reverses #256 Out of Scope | defaults to the Brain repository | `--build-arg` / compose arg still override in either direction | Dockerfile header comment | spec (Fix 4) + `docker compose config` rendering |
| `NEO_REPO_URL` default (`deploy-pipeline.sh:124`) | this ticket | resolves the selector against the Brain repository | env `NEO_REPO_URL` override unchanged | `PipelineWiring.md` | spec (Fix 4); a resolved Brain SHA is fetchable by compose (AC-2) |
| documented pin command (three guides, four sites) | this ticket | names the Brain repository | n/a | the guides themselves | spec (Fix 4) |

**Decision Record impact:** `none` — ADR 0014 (cloud deployment topology) does not name the source repository; #256 was a ticket-level decision, reversed above with its falsifier.

## Acceptance Criteria

- [ ] **AC-1** — The six sites in the Architectural Reality table name `https://github.com/neomjs/neo-agent-brain.git`; `git grep -n 'neomjs/neo.git' -- deploy ai/examples/cloud-deployment learn/agentos/cloud-deployment ai/scripts/lifecycle/local-agent-os` returns nothing.
- [ ] **AC-2** — Outside CI, with `NEO_REPO_URL` unset: `deploy-pipeline.sh`'s resolution step yields a SHA that `git fetch --depth 1 https://github.com/neomjs/neo-agent-brain.git <sha>` accepts (control: today's default yields one the Brain remote rejects with `not our ref`).
- [ ] **AC-3** — A unit spec under `test/playwright/unit/deploy/` reddens if any of those surfaces regains `neomjs/neo.git`, and stays green on the corrected tree; its scope is the listed surfaces only, so `corpusProjectionFreshness.sourceRepository` and the KB's Engine corpus source (which legitimately name the Engine) are outside its read.
- [ ] **AC-4** — `Day0Tutorial.md` clones and pins the same repository.
- [ ] **AC-5** — The Dockerfile comment at `:11` no longer describes the argument as the "neo repository".

## Out of Scope

- The cut of the live cohort to Brain-built images — #253 owns that transaction.
- The `#16635` full-SHA guard and the `/app/.neo-revision` integrity gate — correct, untouched.
- Any surface that names the Engine as a *content* source (KB corpus projection, `NEO_ORCHESTRATOR_CORPUS_SOURCE_REPOSITORY`): those are right.

## Avoided Traps

- **A repo-wide lint for `neomjs/neo.git`.** The orchestrator's corpus projection reports `sourceRepository: https://github.com/neomjs/neo.git` by design (the KB ingests the Engine's content). A blanket grep would flag correct code; the spec reads the deploy surfaces only.
- **Keeping the Dockerfile "generic".** See the Architectural Reality: the builder stage hard-codes `ai/` paths, so the Dockerfile is Brain-specific whatever its default says.
- **Treating fail-closed as fine.** The failure message (`not our ref <sha>`) names neither repository nor the doc line that produced the SHA; an operator has to know the split to decode it.

## Related

- #256 / PR #258 — compose moved to the Brain source; this ticket finishes the same move on the surfaces #256 left out.
- #12 (closed) — proved the Brain image; its notes recorded the "profile gap" this ticket's script/docs half is the remainder of.
- #253 — the live-cohort cut (Engine `467fd122f3` still running); #237 is `blocked_by` it.
- #246 — the same post-split class on a different surface (fleet content roots checkout-relative).
- neomjs/neo#16635 — the full-SHA guard, whose message this ticket's fix stops triggering for the wrong reason.

**Live latest-open sweep:** latest 20 open in `neomjs/neo-agent-brain`, created-descending, at 2026-09-21T14:15:35Z — no equivalent (#253 is the cut, #246 a different surface). **A2A in-flight claim sweep:** 30 most recent messages at 14:16Z; lane-claims in the window are neo #19036 (@neo-opus-grace), neo #19040/#19041 (@neo-opus-ada) and my own withdrawn Brain #237 claim — no overlap. **Memory Core rationale sweep:** surfaced #256's deliberate scoping of the Dockerfile default (2026-08-30) and #12's "profile gap" note — carried above as the reversed decision, not re-derived. **Own-assignment sweep:** my open Brain tickets (#23, #64, #65, #237, #362) own none of these surfaces.

Origin Session ID: 7739f08e-6139-4d6f-b533-86044f255ba3

Retrieval Hint: `query_raw_memories("deploy-pipeline NEO_REPO_URL default Engine repository after Brain split not our ref")`

## Timeline

- 2026-09-21T14:18:01Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-21T14:18:03Z @neo-opus-vega added the `bug` label
- 2026-09-21T14:18:03Z @neo-opus-vega added the `documentation` label
- 2026-09-21T14:18:03Z @neo-opus-vega added the `ai` label
- 2026-09-21T14:18:03Z @neo-opus-vega added the `build` label
- 2026-09-21T14:18:04Z @neo-opus-vega added the `agent-os` label
- 2026-09-21T20:27:08Z @neo-opus-vega cross-referenced by PR #407
- 2026-09-22T09:11:59Z @tobiu referenced in commit `2f365a0` - "Merge pull request #407 from neomjs/vega/406-brain-source-defaults

fix(deploy): the source defaults and guides name the Brain, not the Engine (#406)"
- 2026-09-22T09:11:59Z @tobiu closed this issue

