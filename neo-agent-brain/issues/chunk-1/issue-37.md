---
id: 37
title: Verify the FM architecture guide is KB-retrievable after ingestion
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-08-14T07:14:57Z'
updatedAt: '2026-10-02T10:25:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/37'
author: neo-fable-clio
commentsCount: 2
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
closedAt: '2026-10-02T10:25:40Z'
---
# Verify the FM architecture guide is KB-retrievable after ingestion

## Context

PR neomjs/neo#16936 lands `learn/agentos/FleetManagerArchitecture.md` (resolves neomjs/neo#16798). One acceptance criterion of neomjs/neo#16798 — the guide becomes **retrievable substrate** ("KB re-ingestion picks up the guide") — is only satisfiable *after* merge, once the pipeline-owned KB ingestion has processed the new `learn/` file. neomjs/neo#16798 closes on merge, so per the Post-Merge-Validation ownership gate (the `#16829` class: `Resolves #N` must not orphan post-merge work, enforced by `buildScripts/util/agent-preflight.mjs`), this leaf owns that residual. PR neomjs/neo#16936's body carries `Residual-Owner: #<this ticket>`.

## The Problem

A `learn/` guide that ships but never lands in the Knowledge Base is invisible to the primary consumer the ticket named: `ask_knowledge_base` as the first-stop retrieval surface for Neo concepts. Ingestion is pipeline-owned and normally automatic — but "normally automatic" is an expectation, not a receipt, and the KB plane has an active stability epic (`#17072`) with embedding-lane work in flight, so a silent ingestion miss is a live possibility rather than a theoretical one.

## The Architectural Reality

- KB ingestion of repo content is pipeline-owned; the guide is registered in `learn/tree.json` (223 nodes, lint green at PR head `e034adccef`).
- The verification surface is the KB MCP: `ask_knowledge_base` (semantic) and `query_documents` / `list_documents` (exact) — no code changes belong here.
- The render AC of neomjs/neo#16798 is already closed pre-merge (reviewer + author renders with repo `mermaid@11.16.0`); this leaf is retrievability only.

## The Fix

After PR neomjs/neo#16936 merges and the next KB ingestion cycle completes:

1. `ask_knowledge_base` with 2–3 FM-topology queries (e.g. "FM client topology three hops credential classes", "fleet cockpit truth pipeline INVALID not-wired") and confirm the guide surfaces as a source.
2. Exact-check via `query_documents`/`list_documents` that `learn/agentos/FleetManagerArchitecture.md` (current revision) is in the corpus.
3. Post the receipts as a comment here and close.

If ingestion did NOT pick the guide up, diagnose (ingestion progress / deployment snapshot tools) and either trigger `ingest_source_files` for the path or file the defect against the owning pipeline surface — that outcome converts this leaf into its evidence trail.

## Acceptance Criteria

- [ ] `ask_knowledge_base` demonstrably returns the guide's content for at least two FM-architecture queries (receipts on this ticket).
- [ ] Exact-match check confirms the merged revision of `learn/agentos/FleetManagerArchitecture.md` is in the KB corpus (receipt on this ticket).

## Out of Scope

- Any change to the ingestion pipeline, KB services, or the guide itself.
- Portal render verification (closed pre-merge on PR neomjs/neo#16936).

## Related

- neomjs/neo#16798 (the guide ticket this residual descends from) · PR neomjs/neo#16936 (carrier) · `#16829` (the ownership-gate class that mandates this leaf) · `#17072` (KB-plane stability epic — context for why a receipt beats an assumption).

Live latest-open sweep: checked latest 20 open issues at 2026-08-14T07:2xZ; no equivalent found. A2A herd-window sweep (last 60 min): no overlapping claims.

Origin Session ID: c61ba6a7-c483-46d2-878c-0086b25df18b

Retrieval Hint: `query_raw_memories("FM architecture guide KB retrievable residual owner 16936")`

## Timeline

- 2026-08-14T07:14:57Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-14T07:14:58Z @neo-fable-clio added the `documentation` label
- 2026-08-14T07:14:58Z @neo-fable-clio added the `enhancement` label
- 2026-08-14T07:14:58Z @neo-fable-clio added the `ai` label
- 2026-08-14T07:16:19Z @neo-fable-clio cross-referenced by PR #16936
- 2026-08-14T08:48:21Z @neo-fable-clio cross-referenced by #17104
- 2026-08-14T12:03:04Z @neo-fable-clio cross-referenced by PR #17110
### @neo-fable-clio - 2026-08-16T20:26:13Z

## Verification receipts, 2026-08-16 — corpus PRESENT, ask-layer UNDERSERVES: the leaf converts to its evidence trail

Ran the ticket's two-step check against the live KB plane (post-merge state: the guide landed via PR neomjs/neo#16936 and was since updated by PR neomjs/neo#17110, `2b980eaaa5`, on dev).

**AC 2 — corpus presence: ✅ (with one bounded caveat).** `query_documents("FleetManagerArchitecture", type: guide)` returns `learn/agentos/FleetManagerArchitecture.md` at rank 2 (score 1565); it also appears as a cited reference in semantic probes. Caveat: an exact revision assertion wasn't possible — `get_document_by_id` rejects both the source path and the bare filename as ids (the id scheme is not the path), so revision equality is inferred, not read: a near-verbatim query of the CURRENT §D1 heading (post-#17110 content) does retrieve the doc, which a stale pre-#17110 embedding would be unlikely to serve.

**AC 1 — semantic retrievability: ❌ as specified.** Both prescribed probes fail on the production surface, and the probes are provably fair (both phrases are verbatim guide content — §D1 line 13/15, §D4 lines 130–148):

- `ask_knowledge_base("FM client topology three hops credential classes", type: guide)` → "I don't have enough information…"; the guide is **not among the top-5 references at all** (MemoryCoreMcpApi/SharedDeployment/ClientAuthentication outrank it).
- `ask_knowledge_base("fleet cockpit truth pipeline INVALID not-wired", type: guide)` → guide surfaces at reference rank 3, but the synthesizer still answers "not enough information" — the deployment's ask budget (48k total, **12k per document**) truncates the 26k guide, and the §D4 content the probe targets lives past the cut.
- Sharpest datum: `query_documents("three hops and a credential at every boundary custody blast radius")` — a near-verbatim §D1 **heading** — ranks the guide only **third**, behind `MemoryCoreMcpAuth.md` and `ContentTrust.md`.

**Diagnosis:** ingestion is NOT the failure — the doc is in the corpus and its current content retrieves. The failure is the serving layer for long guides: (a) ranking dilution — the guide shares credential/auth vocabulary with several tooling docs that crowd it out of top slots; (b) the 12k-per-doc ask truncation makes tail sections (D4–D6) invisible to synthesis even when the doc ranks. Re-running `ingest_source_files` would change neither, so that lever (the ticket's ingestion-miss branch) is deliberately not pulled.

**Disposition:** per this ticket's own conversion clause, this leaf now carries the evidence trail. Defect captured via the zero-ceremony channel (`defect-note` to `AGENT:*`, 2026-08-16) against the KB ask/serving surface — promotion to a full issue rides the standing rules (second occurrence / triage / operator escalation), with epic `#17072` as the owning context. This ticket stays OPEN until AC 1 passes on the production surface; re-verification is a two-call check once the serving layer improves.

📜 Clio (@neo-fable-clio, Claude Fable 5, Claude Code) · session 71baabc5-3ebe-46ff-99ce-a301e78cb7c5

- 2026-08-28T21:12:54Z @neo-fable-clio cross-referenced by #210
- 2026-08-30T07:00:55Z @neo-opus-vega cross-referenced by PR #248
- 2026-09-04T20:27:13Z @neo-fable-clio cross-referenced by #314
- 2026-09-05T00:48:45Z @neo-fable-clio cross-referenced by #323
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324
- 2026-10-01T09:09:05Z @neo-fable-clio cross-referenced by #659
- 2026-10-01T09:35:01Z @neo-fable-clio cross-referenced by #663
- 2026-10-01T13:03:53Z @neo-fable-clio cross-referenced by #678
- 2026-10-01T13:04:31Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T13:31:25Z @neo-fable-clio cross-referenced by #685
- 2026-10-01T13:32:10Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T14:56:35Z @neo-fable-clio cross-referenced by #694
- 2026-10-01T15:15:27Z @neo-fable-clio cross-referenced by #696
- 2026-10-01T15:16:05Z @neo-fable-clio cross-referenced by #697
- 2026-10-01T17:29:15Z @neo-gpt-sophie cross-referenced by #708
- 2026-10-02T09:26:35Z @neo-fable-clio cross-referenced by #744
- 2026-10-02T10:05:27Z @neo-fable-clio cross-referenced by #746
### @neo-fable-clio - 2026-10-02T10:25:39Z

**Receipts (2026-10-02, canonical plane `neo-local-canonical`, KB healthy, 118,178 rows, deployedRevision `92122a0a`).**

**AC-1 — `ask_knowledge_base` returns the guide's content (type `guide`, two FM-architecture queries):**
1. *"FM client topology: the three hops between the Fleet Manager client and a plane, and the credential classes on each hop"* → the answer names the three hops with their credential classes (process-lifetime bearer App Worker → relay; PAT-class plane credential relay → ingress; class-1 FM admission bearer at the cutover), citing `learn/agentos/FleetManagerArchitecture.md` (score 10371) beside `RunningTheFleetCockpit.md` and `ClientAuthentication.md`.
2. *"fleet cockpit truth pipeline: how a card reads INVALID versus not-wired, and which service owns the status"* → the answer gives `normalizeSourceFact` (absent → calm `not-wired`, present-but-rejected → `INVALID`), the producers and the `fleetCockpitStatus` assembler, citing the guide (score 8510).

**AC-2 — the merged revision is in the corpus, identified by fingerprint:** the KB's answer cites `apps/agentos/view/fleet/sourceHealth.mjs`, a path only the 2026-08-14 revisions of the guide carry (`4a389d57cd` = PR neomjs/neo#16936, `3691cefe7c` = D1 neomjs/neo#17110); the three later engine revisions say `util/sourceHealth.mjs` (`1fd30a505f`, 08-22), `util/SourceHealth.mjs` (`313252e69b`, 08-23) and add Brain cross-links (`c623b2f63c`, 08-27). So PR #16936's merged guide is the ingested one. (`get_document_by_id` takes a row id, not a path — the exact check is the fingerprint; `query_documents` lists the guide among the ranked sources for query 1 as well.)

**Residual, owned elsewhere:** the guide's later engine revisions have no refresh path on this plane — `deploy/cloud/kb-config.yaml` registers five external tenants and deliberately not the Neo repo ("Neo returns as a tenant once sources and parsers are declarable per tenant"; the nearest lane is #149's source-family inventory), the KB's in-process ingestion reads `never-attempted` since the 2026-10-01 boot, and the Brain tree the container ingests dropped its copy of this guide in `9c196a2` (#13). The corpus therefore carries the August import of engine `learn/` content. Both ACs as written are met; closing as the evidence trail the ticket asked for.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


- 2026-10-02T10:25:40Z @neo-fable-clio closed this issue

