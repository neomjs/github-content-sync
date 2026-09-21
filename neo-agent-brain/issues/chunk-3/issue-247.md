---
id: 247
title: 'Bulk-acquire a revision''s blobs before the ingest reads them, one negotiation per chunk'
state: CLOSED
labels:
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-08-30T01:09:29Z'
updatedAt: '2026-08-30T06:53:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/247'
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
closedAt: '2026-08-30T06:53:55Z'
---
# Bulk-acquire a revision's blobs before the ingest reads them, one negotiation per chunk

Delivered leaf of #65. #65 stays open on its two plane-bound ACs (AC-4 full first-ingest budget, AC-5 cold-mirror threshold), which no unmerged head can satisfy; this ticket owns the implementable half so the PR has a close-target it actually discharges.

Filed at @neo-gpt-emmy's RA-2 on PR #245: the delivered change needs its own close-target and an explicit Contract Ledger.

## The Problem

`cloneIfMissing` builds the tenant mirror `--filter=blob:none`, so no blob is local until something asks. `readRevisionFile` asks for exactly one, and git answers with a lazy promisor fetch — **one network round trip per file**. A first ingest reads the whole tree.

Measured cold against `neomjs/neo`, off any live volume:

| arm | files | wall clock | per file |
|---|---|---|---|
| sequential `git show` (shipped path) | 500 | **234 s** | 0.468 s |
| one `git fetch` + reads | 500 | **7 s** | 0.014 s |
| whole revision | 23,187 | **28 s** | 0.001 s |

**~3.0 hours → 28 seconds** for one repo's first ingest. The container plane independently measured 0.42 s/file, so two instruments on different hardware agree. The blobs transferred are identical; only the round-trip count changes.

Every new ingestion tenant pays the sequential bill on its first ingest, and again on any re-clone — which is why this matters now rather than later.

## The Architectural Reality

- Blobs reach this mirror through two authenticated tiers once this lands: `prefetchRevisionBlobs` in bulk, and `gitMirror.readRevisionFile`'s per-file promisor fetch as the fallback. The graph and tree reads are answered from the filter-complete commit graph and stay credential-free.
- `tenantRepoIngestEnvelopeBuilder.buildFilePayloads` loops those reads, sequentially, for both the full and incremental envelopes.
- `git fetch-pack --stdin` — the obvious batching primitive — **cannot speak smart-HTTPS at all** (`protocol 'https' is not supported`), which is why two earlier attempts at this recorded a non-result as a result. `git fetch origin <oids…>` is the primitive that works.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `gitMirror.prefetchRevisionBlobs({mirrorRoot, tenantId, repoSlug, revision, sourcePaths, credentialRef, chunkSize})` | This ticket; #65 for the measurement | Resolves the missing blob OIDs for **exactly `sourcePaths`** and fetches them in one `git fetch origin <oids…>` per chunk. Returns `{status, requested, missing, chunks, reason}`. | **Never rejects.** `status` is `prefetched` (a fetch ran), `already-local` (nothing missing — a warm or non-partial mirror), or `unavailable` (something refused; `reason` carries the code). An empty `sourcePaths`, or paths absent from the revision, return `already-local` with `missing: 0`. | JSDoc | `gitMirror.spec.mjs` — cold/warm/scoped/degrade/chunking arms, mutation-proven |
| `sourcePaths` scope | #65 AC-7 (incremental sync untouched) | The whole tree is **never** implied. | An incremental sync prefetches only its changed paths, so steady-state polling does not inherit the first-ingest cost. | JSDoc | asserted: an unrequested blob stays missing after a scoped prefetch |
| `credentialRef` | `readRevisionFile`'s existing contract | Forwarded to the fetch, which is the authenticated hop. | Omitted ⇒ anonymous, correct for a public remote and for already-local blobs. Against a private remote an anonymous prefetch fails and degrades to per-file reads. | JSDoc | builder arm asserts the ref reaches the prefetch |
| `chunkSize` | This ticket | OIDs per `git fetch` invocation; default 1000. | 🔴 **Positive-integer domain, enforced.** ~23k OIDs at 41 bytes is ~950 KB of argv, past `ARG_MAX` on macOS. A non-positive or non-integer value falls back to the default — `index += 0` would otherwise never advance and hang the ingest this exists to accelerate. | JSDoc | a `chunkSize: 0` arm returns rather than hanging; a `chunkSize: 2` arm proves the loop |
| `buildFilePayloads` call seam | This ticket | One prefetch, before any read, scoped to that envelope's path set, carrying `credentialRef`. | Called optionally (`?.`) so partial GitMirror doubles stay simple; a contract test asserts the real primitive exposes it. | — | `tenantRepoIngestEnvelopeBuilder.spec.mjs` — full + incremental arms; removing the sole call reddens 3 |

## Acceptance Criteria

- [x] A revision's blobs are acquired in one negotiation per chunk rather than one round trip per file.
- [x] The prefetch is scoped to the paths about to be read; an incremental sync does not prefetch the full revision.
- [x] It never rejects: a refused or impossible prefetch leaves the per-file read path exactly as capable as before.
- [x] `status` distinguishes a warm mirror (`already-local`) from a prefetch that could not run (`unavailable`) — a boolean would collapse the two.
- [x] `credentialRef` reaches the fetch, which is the only authenticated hop.
- [x] `chunkSize` has an enforced positive-integer domain; an out-of-domain value cannot stall the loop.
- [x] 🔴 **The production seam is mutation-convicted:** removing the sole call site, moving it after the reads, widening its scope to the whole revision, or dropping the credential each redden a focused arm.

## Out of Scope

- **The two plane-bound ACs**, which stay with #65: a full first-ingest budget measured on the container plane, and a threshold assertion that fails at 0.42 s/file. Both need a plane where ingestion completes, and the second needs a genuinely cold mirror this plane can no longer produce.
- **Making the prefetch outcome observable in production.** The sole caller currently discards `{status, reason}`, and neither the builder nor `TenantRepoSyncService` has a bounded diagnostic surface to consume it through. Inventing one belongs to whoever owns that surface, not to a perf leaf.
- **Provider-side or transport changes.** `--filter=blob:none` stays; this is the read path, not the clone.

## Avoided Traps

- **Reporting a failed instrument as a negative result.** #65 records two prior Option-A attempts as tested-and-unhelpful. Both were invalid: one had `fetch-pack --stdin` fail so *both* arms ran sequentially (12.42 s vs 12.55 s, read as "batching does not help"), the other timed warm reads in both arms and returned `0s`.
- **Probing absence with a command that fetches the object under test.** `ls-tree --long` fetches blobs to report sizes and `cat-file -e` triggers the promisor fetch it is being asked to test for. The fetch-free probes are `GIT_NO_LAZY_FETCH=1` plus `git rev-list --objects --missing=print`.
- **Trusting a zero the mechanism cannot produce a non-zero for.** A `rev-list` against an invalid ref yields an empty stream that `grep -c` reports as `0`, indistinguishable from *nothing missing*.
- **Proving the primitive and calling the seam done.** The `gitMirror.spec` arms pass with the builder's sole call deleted; only a builder-driven arm recording call order convicts the wiring.


## Timeline

- 2026-08-30T01:09:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-30T01:09:31Z @neo-opus-vega added the `ai` label
- 2026-08-30T01:09:51Z @neo-opus-vega cross-referenced by PR #245
- 2026-08-30T01:11:00Z @neo-opus-vega referenced in commit `845e4c2` - "docs(kb): the read path describes two acquisition tiers, bulk first (#247)

readRevisionFile's JSDoc claimed it was the ONLY operation reaching the
network, which prefetchRevisionBlobs made false. Both authenticate; the
per-file read is now documented as the fallback tier rather than the sole
path."
- 2026-08-30T01:32:38Z @neo-opus-vega referenced in commit `91e6289` - "docs(kb): every acquisition anchor describes both tiers (#247)

Four durable-source anchors still said the per-file read was the only or the
last network hop, which prefetchRevisionBlobs made false: the clone rationale
in gitMirror, both credentialRef params in the envelope builder, and the sync
service's own note that every blob re-authenticates per file.

An earlier pass fixed two anchors and left these, so the contradiction was
half-repaired - worse than untouched, because the corrected surfaces implied
the rest had been checked."
- 2026-08-30T06:53:55Z @tobiu closed this issue

