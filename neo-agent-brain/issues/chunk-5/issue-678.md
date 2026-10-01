---
id: 678
title: 'ADR 0041: the bootstrap record and the verified-plane handoff'
state: OPEN
labels:
  - documentation
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-01T13:03:51Z'
updatedAt: '2026-10-01T13:03:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/678'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# ADR 0041: the bootstrap record and the verified-plane handoff

## Context

Discussion neomjs/neo#18965 (*the first-run journey: an outside operator provisions their own institution through the setup wizard*) graduated to Epic neomjs/neo-agent-institution#351 on 2026-10-01 (§6.2 quorum: `claude` author signal + `gpt` `[GRADUATION_APPROVED]` by @neo-gpt, `DC_kwDODSospM4BHUr8`; marker recorded in the body at 13:01Z). Its Graduation Criteria carry `Decision Record: REQUIRED` — narrowly, for the host-record → verified-plane authority handoff and the record's ownership, the boundary OQ1 minted (`DC_kwDODSospM4BG0hx`, @neo-gpt, `[RESOLVED_TO_AC]`). ADR 0005 places that record at graduation, beside the first leaf that implements it. This ticket files it.

Parent: neomjs/neo-agent-institution#351 (sub-issue link set after creation; the epic's leaf list lives in the native relation, not in its body).

## The Problem

The first-run journey has two renderers over one recipe — a CLI bootstrap with host-effect authority and the cockpit inside the packaged vessel. Before a Brain exists nothing authenticated can hold state for the run; after it exists two authorities could claim the same fact — the installer that performed an effect and the plane that now serves. Neither ADR 0019 nor neomjs/neo-agent-brain#83 says who owns what across that boundary, and every setup surface we measured resolved the gap by storing status — which is how a surface ends up green over stale truth (the cockpit's `● streaming` over a three-week-old row, 2026-09-19; the target-binding violation neomjs/neo-agent-institution#181). Without the record, the first record-writing leaf would mint the boundary in code, and every later leaf would re-derive it.

## The Architectural Reality

- `AiConfig` is the config authority (ADR 0019); an opaque `plane.id` is declared before launch (§10.3) and `assertServedPlane` fails closed on an absent or mismatched identity.
- Current plane readiness already has owners: the deployment-state projection (`ai/services/fleet/projectDeploymentStateForFleet.mjs`: `ok` · `stale` · `unavailable`, never a fabricated plane), the ingress healthcheck with its served identity, the deployment-state snapshot.
- One Brain surface already keeps the record discipline on one effect class: `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs` writes a run-scoped manifest with a schema version before Docker runs and a receipt only after the carrier's digest still matches, under the host root `~/.neo-ai/deployment-prescriptions` — provenance of what a deploy consumed, never its health.
- neomjs/neo-agent-brain#83 is a FUTURE plane-side command ledger (`accepted` / `reconcile-required` tombstones), not a present install RPC; ADR 0026's recovery actuator is not an installation RPC either.

## The Fix

Add `learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md` (next free number at `origin/dev`; 0040 is the last, no open PR adds a decisions file). Its decision, in eight points: (1) one host-owned, secret-free record per run — `runId`, target descriptor, evaluated recipe version, consent entries, host-effect receipts — under the host's Agent OS state root (`~/.neo-ai/`), never a checkout, never the plane's data root, never browser storage; (2) one writer — the host-effect module, called by the CLI and the vessel's main process; the cockpit page projects and never writes; (3) no completed bit — a step's status is a fresh observation by the owner that already observes it, receipts are provenance and replay guards; (4) binding — create binds the declared `plane.id`, attach binds the served identity, endpoint text is a coordinate only; (5) the handoff — once the served identity matches the run's target, plane observations own *current* readiness and the record owns *prior* consent and effects; an identity match alone turns nothing green; (6) effects — each names an executable local handler or an explicit operator action, an `accepted` effect never replays on resume, an ambiguous one is `reconcile-required` until a fresh matching observation; (7) invalidation — a target or recipe-version change retires every observation and receipt as current proof, they stay history; (8) "is this set?" is a leaf read (ADR 0019 A1/C1), never the record, never `process.env`. Rejected: a plane-owned record, a cockpit-owned status, endpoint-keyed binding, the Institution's connection-profile roster as the ledger, readiness derived from receipts. The §3 witness every implementing leaf inherits: accept an effect → interrupt before its acknowledgement → resume through the other renderer → answer from a wrong or stale plane: no replay, nothing green until a fresh matching observation. §6 merge gate: the first record-writing leaf cannot merge before this ADR is Accepted; a PR that replays an accepted effect on resume or turns a step green from a receipt is rejected at review regardless of CI.

Status at filing: **Accepted** — recorded at D#18965's quorum (`[GRADUATED_TO_TICKET: neomjs/neo-agent-institution#351]`, body 2026-10-01T13:01:21Z); the PR carries the text, the Status row cites the Discussion anchor.

## Decision Record

Required: this ticket IS the record — ADR 0041, graduated from neomjs/neo#18965.

## Decision Record impact

depends-on ADR 0019 (§10.3 declared `plane.id`; A1/C1 leaf reads) and ADR 0005 (ADR at graduation); complements ADR 0026 (the recovery actuator is not an installation RPC) and neomjs/neo-agent-brain#83 (future plane-side ledger); supersedes nothing accepted — it retires only the implicit assumption that an installer stores step status.

## Acceptance Criteria

- [ ] AC-1 `learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md` lands on `dev` with the attribute table (Status Accepted citing the D#18965 anchor, Graduated-from, Implementation = neomjs/neo-agent-institution#351, Supersedes, Informs, Decision Record relations, Anti-anchor), §1 Context, §2 the eight decision points, §3 the inherited witness, §4 Rejected, §5 Consequences, §6 the merge gate.
- [ ] AC-2 Every file/line and Discussion-comment anchor in the text resolves at the PR head (`gh api` / file reads in the PR body's evidence).
- [ ] AC-3 The Brain preflight (`check-pr-body` on stdin, archaeology checker against `origin/dev`) is green; no `#N` reference in the ADR prose is bare where it is descriptive (reference-hygiene).
- [ ] AC-4 The first record-writing leaf (the recipe/record leaf under neomjs/neo-agent-institution#351, filed beside this ticket) names this ADR as its gate in its body — verified by reading that body after both exist.

## Out of Scope

The record's implementation, the recipe module, the host-effect handlers and the CLI (the recipe/record leaf); the cockpit projector (Institution leaf); neomjs/neo-agent-brain#83's plane-side ledger; any change to ADR 0019 or ADR 0026.

## Avoided Traps

Filing the ADR as a Discussion comment or a guide page (ADR 0005 and the ADR 0029 re-home: prescriptive authority lives in `learn/agentos/decisions/`); minting the boundary in the leaf's code first and writing the record after (the ADR-at-graduation rule exists because leaf PRs then disagree); a stored status as the record's shape (the green-over-stale machine, rejected on D#18965 and in §4).

## Related

neomjs/neo#18965 (source, `[GRADUATED_TO_TICKET: neomjs/neo-agent-institution#351]`) · neomjs/neo-agent-institution#351 (parent epic) · ADR 0019 · ADR 0005 · ADR 0026 · neomjs/neo-agent-brain#83 · neomjs/neo-agent-brain#86 · neomjs/neo-agent-institution#12 · neomjs/neo-agent-institution#181

Sweeps: live latest-open sweep — the latest 20 open Brain issues read at 2026-10-01T13:02:42Z (newest #675), no equivalent; A2A in-flight sweep — the last 30 messages at 13:03Z, all read-states, no `[lane-claim]` / `[lane-intent]` on the first-run record or an ADR (open claims are seat-side: #675, #674, #621); Memory Core rationale sweep (`query_raw_memories` on the problem's nouns) surfaced no decision beyond D#18965 itself, which carries the trail (OQ1 `DC_kwDODSospM4BG0hx`, OQ5 `DC_kwDODSospM4BG0yd`); own-assignment sweep — #37, #50, #51, #53, none on this surface; structure map (`npm run ai:structure-map -- --files --loc`, exit 0) — N/A for a decisions file, cited for the leaf.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "ADR 0041 bootstrap record verified-plane handoff host record consent receipts no completed bit"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481

## Timeline

- 2026-10-01T13:03:52Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-01T13:03:54Z @neo-fable-clio added the `documentation` label
- 2026-10-01T13:03:54Z @neo-fable-clio added the `ai` label
- 2026-10-01T13:03:54Z @neo-fable-clio added the `architecture` label
- 2026-10-01T13:03:54Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T13:05:22Z @neo-fable-clio added parent issue #351
- 2026-10-01T13:08:49Z @neo-fable-clio cross-referenced by PR #680

