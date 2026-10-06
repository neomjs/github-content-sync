---
id: 896
title: The operator's declared forge PAT may also serve the fleet surface
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-06T11:12:51Z'
updatedAt: '2026-10-06T13:29:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/896'
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
blockedBy:
  - '[x] 19422 ADR 0038 §2.5.1: the operator''s one PAT is declared for MC and /fleet'
blocking:
  - '[x] 571 Plane attach carries the fleet credential that plane-first Add needs'
closedAt: '2026-10-06T13:29:11Z'
---
# The operator's declared forge PAT may also serve the fleet surface

## Context

On 2026-10-06 the operator ruled on neomjs/neo-agent-institution#571 AC-1 and chose option (d), recorded in [6014865586](https://github.com/neomjs/neo-agent-institution/issues/571#issuecomment-6014865586): every peer keeps its own GitHub account and PAT, and the operator's one PAT (tobiu) both sends the operator's A2A and defines seats on the plane, which records the operator as their owner. Two design reads converged on it: [Vega](https://github.com/neomjs/neo-agent-institution/issues/571#issuecomment-6014865586) and [Mnemosyne](https://github.com/neomjs/neo-agent-institution/issues/571#issuecomment-6014907024). The work splits three ways: neomjs/neo-agent-institution#571 is the shell's carrier (Ada), neomjs/neo#19422 amends the ledger, and this leaf is the rule's code.

## The Problem

In plane mode the relay presents the fleet-surface credential to `/fleet`, for the wake stream and for plane-first Add (#881). In a PAT attach, the shell's plane-MCP bearer is the operator's forge PAT. The one-PAT rule (the operator on 10-03 and 10-05; #858) leaves the operator no other credential. `assertFleetPlaneAdmissionBearerClass` refuses an admission bearer whose bytes equal the plane-MCP bearer, and `devFleetServer` won't boot on it. So under the operator's ruling, plane-first Add has no credential it is allowed to present.

For a forge PAT the refusal protects nothing. AuthService's github-pat and gitlab-pat paths skip `aud` by design ("PATs carry no audience claim", `AuthService.mjs:593`, `:965`), so the plane already admits the operator's PAT on `/fleet` from whoever presents it. The rule's own stated reason is plane mints: "byte-identity is exactly how one mint silently serves two audiences" (`fleetServer.mjs:703-711`, and the `planeAdmissionBearer` leaf doc at `configBase.mjs:381-391`). A forge PAT is not a plane mint.

## The Architectural Reality

- **Leaves** (`ai/configBase.mjs:352-410`):
  - `fleet.planeBearer` / `planeBearerFile`: the plane-MCP bearer;
  - `fleet.planeAdmissionBearer` / `planeAdmissionBearerFile`: the fleet-surface bearer;
  - `fleet.admissionTokenFile`: the bootstrap/healthcheck token, held only for comparison.
- **Resolver and assertion** (`ai/services/fleet/fleetServer.mjs:683-747`). `resolveFleetPlaneAdmissionBearer` returns `''` when neither admission leaf is set. `assertFleetPlaneAdmissionBearerClass` throws on equality with the plane bearer or with the bootstrap token.
- **The relay boot** (`ai/services/fleet/devFleetServer.mjs:206`) calls the assertion. It arms the wake stream and `createPlaneFleetClient` with the result. An empty credential leaves the stream on poll and refuses every define.
- **Forge-PAT admissions are the only ones stamped with `authSource`:** `'github-pat'` (`AuthService.mjs:1233`) and `'gitlab-pat'` (`:833`). `local-bearer`, `seat-token`, `oidc` and `proxy-header` are not forge PATs.
- **The class reaches the relay as a declaration.** It is never inferred from equal bytes: the shell takes it from the plane's own admission verdict at probe time and exports it beside the bearer (neomjs/neo-agent-institution#571, per Mnemosyne's read).

## The Fix

1. **One new leaf**, `fleet.planeBearerClass` (`NEO_FLEET_PLANE_BEARER_CLASS`, string, default `''` = undeclared). It holds the plane bearer's credential class, as the plane's admission verdict named it. It is not a plane member, and its JSDoc names the values that change behaviour (the forge-PAT sources).
2. **`resolveFleetPlaneAdmissionBearer`:** an explicitly declared admission bearer (direct or file) still wins. When none is declared and `planeBearerClass` is a forge-PAT source, the admission bearer is the plane bearer itself. The shell exports one secret, declared once.
3. **`assertFleetPlaneAdmissionBearerClass`:** equality with the plane bearer passes only under a declared forge-PAT class. Undeclared, or any other class, still throws as it does today. The bootstrap-token comparison doesn't change, so a forge PAT that is the deployment's bootstrap PAT still refuses.
4. **Docs:** the two admission leaves, the assertion, `planeFleetClient.mjs:19` and the local-agent-os lifecycle README's credential-class paragraph each say so in one sentence.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleet.planeBearerClass` (new leaf) | ADR 0038 §2.5.1 row 1 as amended (neomjs/neo#19422); ADR 0019 §5 | declares the plane bearer's class from the plane's admission verdict | `''`: undeclared, exactly today's behaviour | leaf JSDoc | unit: config parity + default |
| `resolveFleetPlaneAdmissionBearer` | same | a declared admission leaf wins; else the plane bearer under a declared forge-PAT class; else `''` | `''` keeps the relay honestly unarmed | JSDoc | unit, every branch |
| `assertFleetPlaneAdmissionBearerClass` + the relay boot | same | equality passes only under a declared forge-PAT class; equality with the bootstrap token always refuses | throws, and the boot refuses (unchanged) | JSDoc + README class paragraph | unit, red first |

## Acceptance Criteria

- [ ] AC-1: With `planeBearerClass` empty, every existing spec of the resolver, the assertion and the relay boot passes unchanged.
- [ ] AC-2: With a declared forge-PAT class (`github-pat` or `gitlab-pat`) and no admission leaf set, the admission bearer resolves to the plane bearer and the assertion returns it (unit).
- [ ] AC-3: The red-first controls still throw (unit):
  - equal bytes with no class declared;
  - equal bytes under a non-forge class (`local-bearer`, `seat-token`);
  - a declared forge-PAT bearer that equals the bootstrap token.
- [ ] AC-4: An explicitly declared admission bearer still wins over the derivation. It still refuses an alias unless the class is a declared forge PAT (unit).
- [ ] AC-5: Through the composed admission chain, a declared forge-PAT plane bearer resolves as the relay's fleet credential, is admitted on `/fleet/events` and drives cold catch-up (unit, composed server with a doubled auth). The relay boot hands that one resolved value to both the wake stream and `createPlaneFleetClient` (the existing dev-entry ratchet).

Not claimed here: the installed proof that plane-mode Add defines a seat owned by the operator. That is neomjs/neo-agent-institution#571's AC-4 (packaged held run).

## Out of Scope

- The shell's carrier, its probe and the attach copy (neomjs/neo-agent-institution#571).
- The ledger text (neomjs/neo#19422).
- The plane's forge registry (#858 and the operator-authorized one-time step).
- Option (c), a plane-minted fleet credential: it is the fallback if (d) is ever withdrawn. Option (b), a plane-minted MC credential.

## Avoided Traps

- **Inferring the class from equal bytes.** Equal bytes are exactly what the rule refuses for mints.
- **Dropping the equality check.** An undeclared alias of a minted bearer would then pass silently.
- **The shell exporting the same secret under two names.** One secret, one declaration.

Decision Record impact: depends-on ADR 0038 §2.5.1 as amended by neomjs/neo#19422; aligned-with ADR 0019 (one declared leaf, read at the use site).

Structure map: `npm run ai:structure-map -- --files --loc` at `1b69ef7`. The owning folder is `ai/services/fleet/`, plus the `fleet` leaves in `ai/configBase.mjs`. No new file.

Sweeps:
- Live latest-open: the latest 20 open Brain issues at 2026-10-06T11:09Z (#549 to #880). No equivalent.
- A2A: claims since 2026-10-05 16:00Z. None on this scope, and Ada accepted this split at 11:06Z.
- MC: the rationale sweep on the class rule and the operator PAT returned the 10-05 AC-1 trail (Ada, Emmy). No prior decision on a declared forge-PAT class.
- Own assignments: none on this surface.
- Keyword: `planeAdmissionBearer` across the org: neomjs/neo-agent-institution#571 (consumer), plus #857, neomjs/neo#17101 and neomjs/neo#17276 (closed).

Related: neomjs/neo-agent-institution#571 · #881 · #857 · #858 · #571 · neomjs/neo#19388 · neomjs/neo#17116

Origin Session ID: d8d6173a-bc55-48ea-a50a-156a0add9ff4

Retrieval Hint: "declared forge PAT class plane bearer fleet admission one operator PAT option d"



## Timeline

- 2026-10-06T11:12:52Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-06T11:12:52Z @neo-opus-vega added the `enhancement` label
- 2026-10-06T11:12:52Z @neo-opus-vega added the `ai` label
- 2026-10-06T11:12:53Z @neo-opus-vega added the `architecture` label
- 2026-10-06T11:12:53Z @neo-opus-vega added the `security` label
- 2026-10-06T11:12:53Z @neo-opus-vega added the `agent-os` label
- 2026-10-06T11:13:09Z @neo-opus-vega cross-referenced by #19422
- 2026-10-06T11:13:12Z @neo-opus-vega marked this issue as blocking #571
- 2026-10-06T11:13:12Z @neo-opus-vega marked this issue as being blocked by #19422
- 2026-10-06T11:16:05Z @neo-opus-vega cross-referenced by PR #19424
- 2026-10-06T11:17:54Z @neo-opus-ada cross-referenced by #571
- 2026-10-06T11:24:39Z @neo-opus-vega cross-referenced by PR #897
- 2026-10-06T11:24:58Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-06T11:25:24Z @neo-opus-ada referenced in commit `82404c1` - "feat(harness): the attach records the plane's class for its bearer, and the launch hands the bearer once with it (#571)

Under the operator's 2026-10-06 ruling (d), the operator's one PAT serves the plane's MCP and its
/fleet surface, so the separately stored fleet credential goes. The attach asks /fleet/probe which
class the plane admitted the bearer as and keeps that authSource in plane.json. The launch exports it
as NEO_FLEET_PLANE_BEARER_CLASS (neomjs/neo-agent-brain#896) and empties both admission variables:
the Brain derives the fleet-surface credential from a forge-PAT bearer, and nothing inherited crosses
planes. A missing or unnamed class exports '', which keeps today's rule."
- 2026-10-06T11:35:11Z @neo-opus-vega referenced in commit `fd31ed7` - "feat(fleet): the no-credential refusal names the operator's remedy (#896)

The relay's define refusal reaches the operator verbatim in Add Agent. Its old cause, "the plane-MCP bearer never dials the fleet surface", is no longer true once a forge-PAT plane bearer is declared. It now leads with the remedy (reconnect the plane with your own GitHub or GitLab PAT) and keeps the cause after it."
- 2026-10-06T11:36:06Z @neo-opus-ada cross-referenced by #898
- 2026-10-06T11:53:29Z @neo-opus-vega referenced in commit `e3ec28c` - "docs(fleet): an empty admission leaf arms through a declared forge-PAT class (#896)

Emmy's review on #897: the planeAdmissionBearer doc said empty leaves leave plane-stream consumers unarmed, which a declared forge-PAT planeBearerClass now contradicts."
- 2026-10-06T12:08:07Z @neo-opus-vega referenced in commit `d28b72f` - "feat(fleet): the refusal line reads as the design seat wrote it (#896)

Mnemosyne's design read for the FM gate: "This plane connection can't add agents: reconnect the plane with your own GitHub or GitLab PAT." The technical cause stays in parentheses."
- 2026-10-06T12:51:17Z @neo-opus-ada cross-referenced by #571
- 2026-10-06T13:29:11Z @tobiu referenced in commit `7f22b0a` - "feat(fleet): a plane bearer declared as the operator's forge PAT also serves the fleet surface (#896) (#897)

* feat(fleet): a plane bearer declared as the operator's forge PAT also serves the fleet surface (#896)

The operator's 2026-10-06 ruling (neomjs/neo-agent-institution#571,
option d) lets the operator's one PAT serve both of the plane's
audiences. A new leaf, fleet.planeBearerClass, carries the plane
bearer's class as the plane's own admission verdict names it
(authSource). Under github-pat or gitlab-pat, the fleet-surface
credential resolves to the plane bearer when no admission bearer is
declared, and the class rule accepts the equal bytes. An undeclared or
non-forge class still refuses, and the bootstrap token refuses under
any class.

* feat(fleet): the no-credential refusal names the operator's remedy (#896)

The relay's define refusal reaches the operator verbatim in Add Agent. Its old cause, "the plane-MCP bearer never dials the fleet surface", is no longer true once a forge-PAT plane bearer is declared. It now leads with the remedy (reconnect the plane with your own GitHub or GitLab PAT) and keeps the cause after it.

* docs(fleet): an empty admission leaf arms through a declared forge-PAT class (#896)

Emmy's review on #897: the planeAdmissionBearer doc said empty leaves leave plane-stream consumers unarmed, which a declared forge-PAT planeBearerClass now contradicts.

* feat(fleet): the refusal line reads as the design seat wrote it (#896)

Mnemosyne's design read for the FM gate: "This plane connection can't add agents: reconnect the plane with your own GitHub or GitLab PAT." The technical cause stays in parentheses."
- 2026-10-06T13:29:11Z @tobiu closed this issue
- 2026-10-06T13:40:46Z @neo-opus-ada cross-referenced by PR #577
- 2026-10-06T14:59:42Z @tobiu referenced in commit `75c3467` - "feat(harness): the plane attach carries the operator's PAT and its class to the fleet surface, with the Brain pin at 0b8477c8 (#571) (#577)

* feat(harness): the plane record holds the fleet credential and hands it to the fleet child (#571)

The storage half of the carrier. It is the same under either attach design still open on
the ticket.

- planeConfig: an optional fleet credential, encrypted beside the bearer in
  `plane-fleet-credential.bin` (safeStorage, 0600). It reads back with the record and is
  forgotten with it. A credential that is empty, or that equals the bearer, is refused
  before anything is written, mirroring the Brain's class rule. A record stored again
  without one drops the earlier one.
- planeEnvFragment: a launch from the record exports `NEO_FLEET_PLANE_ADMISSION_BEARER` as
  the record's credential or '', and always empties the `_FILE` sibling, so a credential
  the launch inherited never reaches the record's plane. An env bearer or env base keeps
  the env's own.
- main: the log redacts the stored fleet credential and an inherited admission bearer
  along with the other secrets.

The attach prompt, the `/fleet` probe and the card's copy follow the design call.

* feat(harness): the attach records the plane's class for its bearer, and the launch hands the bearer once with it (#571)

Under the operator's 2026-10-06 ruling (d), the operator's one PAT serves the plane's MCP and its
/fleet surface, so the separately stored fleet credential goes. The attach asks /fleet/probe which
class the plane admitted the bearer as and keeps that authSource in plane.json. The launch exports it
as NEO_FLEET_PLANE_BEARER_CLASS (neomjs/neo-agent-brain#896) and empties both admission variables:
the Brain derives the fleet-surface credential from a forge-PAT bearer, and nothing inherited crosses
planes. A missing or unnamed class exports '', which keeps today's rule.

* test(harness): the carrier composed with a real Brain root, from the Fleet probe to the class rule (#571)

* build(deps): the declared Brain pin moves to 0b8477c8, with the carrier (#571)

package.json, the lock and ci.yml's contract checkout move from f24815d6 to
Brain dev 0b8477c8, which carries neomjs/neo-agent-brain#897 (the declared
forge-PAT plane bearer) and #899. A pin past e3388e5e makes plane-mode Add
refuse without this carrier, and the carrier's composed arm needs #897, so
the two land together.

* fix(harness): the plane record's plane, identity and class read back only with the bearer they were written for (#571)

Euclid's round 1 on #577:
- RA-1: plane.json names the sha256 of the exact encrypted bearer it was
  written with, and each file is replaced whole (temp file plus rename). A
  replacement that fails between the two files, or a bearer file that is gone
  or replaced, now reads as unconfigured. It is never the new bearer with the
  old plane, identity or class. A record from before the binding carries no
  class.
- RA-2: the composed arm stores and reads back seat-token, oidc and an unknown
  acme-pat record through production code. Each derives no fleet-surface
  credential in the real Brain, and its equal bytes still refuse.

* fix(harness): the plane record is written before its bearer, so a failed replacement never pairs the new bearer with an older record (#571)

Euclid's round 2 on #577 (RA-1, the legacy case): over a record stored before the binding, writing the bearer first and then failing on plane.json left the old plane and identity with the new bearer. plane.json now goes first and names the new bearer's digest: a failure before it lands keeps the old pair whole, and one after it reads unconfigured, over a bound record or an older one alike."

