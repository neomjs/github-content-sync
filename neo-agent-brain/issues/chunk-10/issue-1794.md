---
id: 1794
title: A merged Caddyfile never reaches the local ingress
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T08:16:47Z'
updatedAt: '2026-11-16T13:30:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1794'
author: neo-opus-ada
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
blocking: []
closedAt: '2026-11-16T13:30:25Z'
---
# A merged Caddyfile never reaches the local ingress

## Context

@neo-opus-vega found #1763 (classified gateway records for Caddy 502s, merged 2026-11-15 14:52Z) not live, although the plane runs the cut that contains it. I re-measured on 2026-11-16 at 07:4xZ:

```text
$ docker inspect -f '{{.State.StartedAt}}' neo-local-agent-os-ingress-1
2026-11-15T11:29:56.473382382Z
$ docker exec neo-local-agent-os-ingress-1 ls -la /etc/caddy/
-rw-r--r--    0 root     root           586 Nov  5 19:08 Caddyfile
$ docker exec neo-local-agent-os-ingress-1 cat /etc/caddy/Caddyfile
cat: can't open '/etc/caddy/Caddyfile': No such file or directory
$ ls -la deploy/cloud/Caddyfile.local-agent-os        # host checkout, 254fda1a
-rw-r--r--  1 …  1337 Nov 16 00:55 deploy/cloud/Caddyfile.local-agent-os
```

The ingress serves the config it loaded at its 11:29Z start, from before #1763. Its health check (`nc -z 127.0.0.1 8080`) stays green. The new file validates on the local `caddy:2-alpine` (v2.11.3).

The same mechanism hit twice before: `kb-config.yaml` on 2026-09-23 (a restarted kb-server kept link count 0 and the old 6,373 bytes; defect-note fingerprint `bf63314cc9d33dd4`), and @neo-opus-grace recorded a deleted Caddyfile inode on the ingress on 2026-11-07 while tracing #904. This is the third sighting, so the defect is promoted.

## The Problem

A checkout writes a tracked file as a new inode. A single-file bind mount keeps the old one, so a running container goes on reading a deleted file, and `caddy reload` then cannot read its config at all. The local cut recreates the four Neo services and leaves `chroma` and `ingress` alone on purpose. So every Caddyfile change merges green and never deploys, and nothing reports it.

## The Architectural Reality

- `deploy/cloud/docker-compose.local-agent-os.yml`, `ingress`: `volumes: !override` binds the single file `./Caddyfile.local-agent-os` to `/etc/caddy/Caddyfile`, and the command is `caddy run --config /etc/caddy/Caddyfile --adapter caddyfile`. The service sits behind `profiles: [ingress]`.
- The same file binds `./kb-config.yaml:/app/kb-config.yaml:ro` twice, once on kb-server, the 2026-09-23 case.
- The cut is an operator-machine script outside this repository. It recreates `mc-server kb-server orchestrator fleet-server` with `up -d --no-deps --no-build --force-recreate`, so the kb binds are re-bound on every cut. Only the ingress is never recreated.

## The Fix

*Narrowed on 2026-11-16 at implementation: `kb-config.yaml` stays a file bind (point 3).*

1. The ingress binds its config folder, never the file: the local Caddyfile moves to `deploy/cloud/ingress-local/Caddyfile`, bound read-only at `/etc/caddy`. A directory bind shows a checkout's new inode.
2. The ingress runs `caddy run --watch`, so a changed Caddyfile reloads on its own. A reload that fails keeps the running config and logs the error, so a bad merge cannot take the ingress down.
3. A compose-shape spec fails on any new single-file bind in the local profile. `kb-config.yaml` is its one reasoned exception. Every cut force-recreates both of its consumers, which re-binds it, and its readers resolve `<neoRootDir>/kb-config.yaml`, so a folder there would move a resolver path.

## Acceptance Criteria

- [ ] The ingress binds `deploy/cloud/ingress-local/` read-only at `/etc/caddy` and runs `caddy run --watch`; a compose-shape spec fails on a new single-file bind beside `kb-config.yaml`.
- [ ] Live, on a throwaway Caddy container with the same bind and flags: a Caddyfile replaced the way a checkout writes it (a new inode) is served without a recreate, with the reload line from its log as the receipt. The plane's ingress only gets the new bind at its one-time recreate, the next AC.
- [ ] Post-merge: after a one-time `--force-recreate` of `ingress`, #1763's `mc_gateway` record appears on a forced 502.

## Out of Scope

- The cloud profile's ingress (`docker-compose.yml` binds `./Caddyfile` the same way). No cloud plane is live, and its deploy path differs; the local overlay replaces that bind.
- A folder bind for `kb-config.yaml` (The Fix, point 3).
- The operator-machine cut script itself. With the folder bind and `--watch`, it needs no ingress step.

## Avoided Traps

- Recreating the ingress on every cut: it would fix the symptom but cost a 502 window on each cut, and it leaves `caddy reload` broken in between.
- A cut step that compares hashes and recreates on a difference: that logic lives outside the repository and repeats what a directory bind gives for free.

## Related

#1763 (the change that never deployed) · #904 (the 2026-11-07 sighting) · #84 (the local hard cut)

Live latest-open + closed sweep: latest 20 open, plus open and closed searches (`ingress Caddyfile mount`, `single-file bind mount stale config`, `bind mount recreate stale`) at 2026-11-16T08:14Z. Out of Scope read on #425 and #253; no equivalent or prior exclusion found. Backlog band: 160 open (red); no owning ticket to fold into.
A2A in-flight sweep: latest 30 messages, no claim on the ingress; @neo-opus-vega's report goes to the owner of the cut.
MC sweep: "ingress keeps serving an old Caddyfile after a checkout": Vega's 2026-11-16 diary and Grace's 2026-11-07 inode note; no ticket.
Own-assignment sweep: 6 open (#1781, #1777, #1531, #571, #133, #1508), none on the ingress.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): `deploy/cloud` is outside `ai/`; the compose files own the change, and no new `.mjs` placement arises beyond the spec.

Labels: no `good first issue` or `help wanted`, because the live AC needs the operator machine's running plane.
Ownership: @neo-opus-ada, who owns the local cut.

Origin Session ID: a63efafd-7284-4d9c-aac1-4257cfb33300

Retrieval Hint: "single-file bind mount stale inode after checkout" · "ingress Caddyfile never deployed" · "caddy reload fails missing config"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-16T08:16:48Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-16T08:16:50Z @neo-opus-ada added the `bug` label
- 2026-11-16T08:16:50Z @neo-opus-ada added the `ai` label
- 2026-11-16T08:16:50Z @neo-opus-ada added the `agent-os` label
### @neo-opus-ada - 2026-11-16T08:25:31Z

**Pending operator go: the one-time ingress recreate that makes #1763 live.** I own the cut, and I want it now. My seat's auto mode refused it as a destructive action on 2026-11-16 at about 07:55Z, and I have not routed it around that refusal.

- Measured state: the ingress started 2026-11-15T11:29:56Z, and its bound Caddyfile has link count 0 and reads ENOENT. The 1337-byte host file validates on the local `caddy:2-alpine` (v2.11.3).
- The recreate: from `deploy/cloud`, the cut's own compose invocation (its `--env-file` and three `-f` files) plus `--profile ingress`, then `up -d --no-deps --no-build --force-recreate ingress`. That costs a few seconds of 502s on `127.0.0.1:3102`.
- Afterwards I check the mount inside the container (link count 1, 1337 bytes) and look for a `mc_gateway` record on the next 502.

This fix's own PR makes the step unnecessary from then on (directory bind plus `--watch`). Until it merges, any Caddyfile change needs this recreate.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-16T08:26:00Z @neo-opus-ada cross-referenced by #1777
- 2026-11-16T08:31:13Z @neo-opus-ada cross-referenced by PR #1798
- 2026-11-16T09:27:51Z @neo-opus-ada referenced in commit `0c67f2d` - "test(deploy): the bind guard reads the effective local profile (#1794)"
- 2026-11-16T09:40:53Z @neo-opus-ada referenced in commit `a756a41` - "test(deploy): the bind guard reads Compose's own render of the profile (#1794)"
- 2026-11-16T13:30:25Z @tobiu referenced in commit `a2a621c` - "fix(deploy): a merged Caddyfile reaches the local ingress (#1794) (#1798)

* fix(deploy): a merged Caddyfile reaches the local ingress (#1794)

The plane cut never recreates the ingress, and its single-file bind kept
the inode a checkout deleted, so #1763's Caddyfile never deployed. The
local Caddyfile moves into deploy/cloud/ingress-local/, bound read-only
at /etc/caddy, and Caddy runs with --watch. A compose-shape spec fails
on any new single-file bind beside kb-config.yaml, whose consumers
every cut force-recreates.

* test(deploy): the bind guard reads the effective local profile (#1794)

* test(deploy): the bind guard reads Compose's own render of the profile (#1794)"
- 2026-11-16T13:30:27Z @tobiu closed this issue

