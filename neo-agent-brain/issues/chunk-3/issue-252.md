---
id: 252
title: Retain the pre-Brain container cohort as a rollback target
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-30T16:42:32Z'
updatedAt: '2026-08-30T16:46:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/252'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 12 Receive Agent OS deployment and prove the Brain image'
closedAt: '2026-08-30T16:46:44Z'
---
# Retain the pre-Brain container cohort as a rollback target

## Context

The canonical local Agent OS is healthy, but its four Neo-derived services still run images built from Engine revision `467fd122f3dbb92700d41bcafa81c75a9cb3ccfc`, before the Brain repository cut. Each image currently has only its mutable `latest` tag.

A fresh named backup bundle is successful and restorable, but data rollback and code rollback are separate authorities. The next image build retargets `latest`; without retaining the current image IDs first, the operation that proves the Brain image can erase the only conveniently selectable pre-cut code cohort.

This is the missing rollback half consumed by #12. That ticket explicitly excludes image-pin implementation, so this work is a separate one-PR/operator-action leaf rather than scope added to Vega's ticket.

## The Problem

The four current image IDs are content-addressed and resident, but no durable cohort binds them to one rollback name and the verified data bundle:

| service | current image ID | current source |
|---|---|---|
| `orchestrator` | `sha256:9e251be213f9d8cad4c0394b4f5e51548471327dec9cb417ceb1233ca45e50c9` | Engine `467fd122f3` |
| `mc-server` | `sha256:8b3171ccdae621e331891e31e6d4e98b2de395262b52ded7170afeb5362aa0bc` | Engine `467fd122f3` |
| `kb-server` | `sha256:aeed879596b2752c21897e6d002db12fc936a1fa6bff29111854d3434f57a2f4` | Engine `467fd122f3` |
| `fleet-server` | `sha256:5143610d54069090175e69673394abcc899ba9e870f12a5078e83546ad908b97` | Engine `467fd122f3` |

A digest written only in a ticket does not retain a local image against pruning. A local tag without a receipt does not identify the Compose project, config cohort, persistent volumes, or backup bundle it belongs with. Both are required.

## The Architectural Reality

- Compose project `neo-local-agent-os` owns the running service identities and ten named volumes.
- `/app/.neo-revision`, `org.neomjs.image.requested-ref`, and `org.opencontainers.image.revision` all agree on the pre-cut Engine SHA.
- `org.opencontainers.image.source` names `https://github.com/neomjs/neo.git` on all four images.
- The existing backup bundle records KB, Memory Core, graph, collection lineage, and embedding identity. It is the data half of rollback, not an image-retention mechanism.
- Docker tagging is metadata-only: it does not recreate a container, touch a named volume, or change the image bytes.

Mandatory structure-map gate: `npm run --silent ai:structure-map -- --files --loc` passed. This leaf introduces no repository file or `.mjs` placement; deployment transaction precedent remains under `ai/scripts/maintenance/` and `ai/examples/cloud-deployment/`.

## The Fix

Before any Brain image build:

1. Add one immutable local tag per exact image ID, using the common cohort suffix `pre-brain-cut-467fd122f3`.
2. Persist a non-secret `container-cohort.json` receipt beside the current named backup bundle.
3. Read back every tag, digest, label, project/config identity, and volume name from Docker.
4. Prove the operation changed no running container ID, health state, or volume identity.

The receipt is machine-local deployment state and must not be committed to the repository.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| four rollback tags (new) | exact running image IDs above | each tag resolves to its pre-cut image ID | any mismatch blocks the Brain build | none | `docker image inspect` readback |
| `container-cohort.json` (new) | current Docker inspection + named backup receipt | binds project, source revision, service digests/tags/config hashes, volume names, and backup bundle | missing/incomplete receipt blocks cutover | inline schema fields in receipt | JSON readback + field assertions |
| running Compose cohort | Docker container labels | container IDs and health remain unchanged by retention | any recreation/change is failure | none | pre/post `docker inspect` |
| named volumes | Compose project labels | exact volume-name set remains unchanged | any added/removed volume is failure | none | pre/post volume census |

## Decision Record impact

`none` — this makes the existing rollback requirement executable; it does not change deployment topology.

## Acceptance Criteria

- [ ] `orchestrator`, `mc-server`, `kb-server`, and `fleet-server` each have an immutable local `pre-brain-cut-467fd122f3` tag resolving to the exact image ID recorded above.
- [ ] The named backup used by the receipt is fresh, successful, integrity-clean, and `restorable: true` at capture time.
- [ ] `container-cohort.json` records `schemaVersion`, `capturedAt`, Compose project, Engine source repository/revision, backup bundle name, the ten volume names, and per-service image ID, repository digest, rollback tag, config hash, requested ref, OCI revision, and OCI source.
- [ ] Receipt values are read back from Docker and the backup receipt; no value is inferred from tag naming.
- [ ] Pre/post container IDs and health states are byte-for-byte unchanged.
- [ ] Pre/post Compose-project volume-name sets are identical.
- [ ] No image build, container recreation, Compose down, volume mutation, or data restore occurs in this leaf.

## Out of Scope

- Building or deploying the Brain image.
- Executing rollback.
- Restoring Memory Core, Knowledge Base, graph, or Chroma data.
- Changing Compose files, image labels, deployment topology, or repository code.
- Durable kbSync/tenant-ingestion design.
- Post-cut rollback-tag survival readback; the live cutover ticket must require it after `latest` moves.

## Avoided Traps

- **Record IDs only in GitHub.** Rejected: GitHub cannot retain machine-local images against pruning.
- **Rely on `latest`.** Rejected: the next build moves it by definition.
- **Tag images without binding the data bundle.** Rejected: code and data could roll back to different moments.
- **Commit host image IDs.** Rejected: they are machine-local deployment state, not portable repository authority.
- **Rebuild first, pin afterward.** Rejected: the first mutation destroys the selection name this ticket exists to preserve.

## Related

Related: #12 · #84 · #90

Origin Session ID: `59ebb4c5-25c4-4e71-b66c-0cf8c30f9079`

Retrieval Hint: `pre-Brain rollback image cohort 467fd122 latest tag named backup`

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-08-30T16:42:32.108Z; no equivalent found.
A2A in-flight claim sweep: checked 30 messages across all read states over the last hour; no competing peer claim found.


## Timeline

- 2026-08-30T16:42:34Z @neo-gpt-emmy added the `bug` label
- 2026-08-30T16:42:34Z @neo-gpt-emmy added the `ai` label
- 2026-08-30T16:42:34Z @neo-gpt-emmy added the `testing` label
- 2026-08-30T16:42:34Z @neo-gpt-emmy added the `build` label
- 2026-08-30T16:42:35Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-30T16:42:45Z @neo-gpt-emmy assigned to @neo-gpt-emmy
### @neo-gpt-emmy - 2026-08-30T16:46:39Z

## Completion evidence — pre-cut rollback cohort retained

Executed the ticket's bounded operator action without rebuilding or recreating anything.

### Immutable tag readback

| service | rollback tag | verified image ID |
|---|---|---|
| `orchestrator` | `neo-local-agent-os-orchestrator:pre-brain-cut-467fd122f3` | `sha256:9e251be213f9d8cad4c0394b4f5e51548471327dec9cb417ceb1233ca45e50c9` |
| `mc-server` | `neo-local-agent-os-mc-server:pre-brain-cut-467fd122f3` | `sha256:8b3171ccdae621e331891e31e6d4e98b2de395262b52ded7170afeb5362aa0bc` |
| `kb-server` | `neo-local-agent-os-kb-server:pre-brain-cut-467fd122f3` | `sha256:aeed879596b2752c21897e6d002db12fc936a1fa6bff29111854d3434f57a2f4` |
| `fleet-server` | `neo-local-agent-os-fleet-server:pre-brain-cut-467fd122f3` | `sha256:5143610d54069090175e69673394abcc899ba9e870f12a5078e83546ad908b97` |

Every `docker image inspect` readback also reproduced the recorded repository digest, Engine requested/OCI revision `467fd122f3dbb92700d41bcafa81c75a9cb3ccfc`, and source `https://github.com/neomjs/neo.git`.

### Receipt and data binding

- Receipt: `backup-2026-08-30T14-56-45.665Z/container-cohort.json`
- SHA-256: `60dc92f75c6d71673732cc0b1fea8bdaf7c3c83979af62e2b811edf5f4a21fbc`
- Mode/size: `0644`, 4,221 bytes
- Backup readback: `success`, `restorable: true`, integrity-clean; completed `2026-08-30T14:58:38.540Z`
- Receipt schema: 4 services, 10 named volumes, Compose project `neo-local-agent-os`

### Negative controls

- Pre/post running container IDs: unchanged for all four services.
- Pre/post state: all four still `running` / `healthy`.
- Pre/post named-volume set: exact 10/10 match.
- No image build, Compose operation, container recreation, volume mutation, tracked repository edit, or restore occurred.

The body was corrected before closure to remove a circular gate: survival after `latest` moves is downstream cutover validation, not a #252 close condition. The native block on #12 can now resolve by closing this completed pre-cut leaf.

- 2026-08-30T16:46:44Z @neo-gpt-emmy closed this issue
- 2026-08-30T16:48:51Z @neo-gpt-emmy cross-referenced by #253
- 2026-08-30T18:54:24Z @neo-opus-vega cross-referenced by #12
- 2026-08-30T22:09:35Z @neo-gpt-emmy cross-referenced by #184

