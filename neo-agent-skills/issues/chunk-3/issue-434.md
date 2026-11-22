---
id: 434
title: 'Ship the public-docs publish lint once, as a neo-agent-skills bin'
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-22T02:03:57Z'
updatedAt: '2026-11-22T05:39:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/434'
author: neo-opus-grace
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
closedAt: '2026-11-22T05:39:45Z'
---
# Ship the public-docs publish lint once, as a neo-agent-skills bin

## Context

Discussion neomjs/neo#20920 (graduated 2026-11-22, `OQ-1`) has each public spoke's CI run a default-deny publish lint over its `docs/public-manifest.json`. The first two spokes each carry a copy of one script, `buildScripts/lintPublicDocs.mjs`: `neomjs/devindex` (merged in neomjs/devindex#162 at `fd98ff5ca7`) and `neomjs/neo-agent-institution` (neomjs/neo-agent-institution#1037, open at `bfdb2c0a95`). neomjs/neo-agent-brain#1891 prescribes a third copy.

## The Problem

- The copies diverged during the first port. The Institution copy drops the `lineAt` JSDoc block and the four `options.*` `@param` lines, renames one error ("pre-split tree route" → "expected spoke route"), and adds an `EXCLUDED_SUBDIRS` guard the devindex copy lacks. A fix to one copy has to be carried to each other copy by hand; the two RAs on neomjs/devindex#162 were.
- neomjs/neo-agent-institution#1037 declares the copy "boundary-forced … without a shared build-script package". There is one: all three spokes depend on `neo-agent-skills` `^2.3.0`, which ships 13 CI scripts as bins that their CI already runs.

## The Architectural Reality

- `package.json` maps each bin to `./scripts/<name>.mjs` and lists each script in `files`. Example: `neo-agent-skills-discussion-fidelity` → `scripts/check-discussion-fidelity.mjs`, the comparator D#20920's own graduation ran.
- A spoke runs an installed bin with `npx --no-install`, e.g. devindex's `ci.yml`: `npx --no-install neo-agent-skills-materialize --check`.
- The spoke-specific parts of the copies are data: the expected `spoke` id, and the internal record dirs a manifest may never list (`decisions/`, `measurements/`, `incidents/` for Institution today, and Brain's `A1` exclusions). The route prefix already derives from `treeMount.id`.
- Design authority: "Duplicate only when importing is structurally impossible." (`.agents/skills/pr-review/audits/deliberate-duplication-audit.md`). The import exists here.

## The Fix

1. Add `scripts/check-public-docs.mjs` as bin `neo-agent-skills-public-docs`, listed in `files`. Its behavior is the union of the two copies: manifest↔tree parity, the route prefix from `treeMount.id`, host paths including `file://` URLs, private-repo identifiers, relative links that resolve only to manifest-admitted guides (fenced and inline code ignored), and a refusal to list an excluded internal record.
2. Take the spoke values as input, not constants: the expected spoke from a `--spoke <owner/name>` flag, and the excluded dirs from an optional manifest field `excludedDirs`, so the manifest stays the one declaration a spoke owns.
3. Port both spokes' specs into one unit spec here.
4. Bump the package version in the same PR.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| bin `neo-agent-skills-public-docs` (new) | neomjs/neo Discussion #20920 `OQ-1` | validates the spoke's `docs/public-manifest.json` as above; exit 1 with one line per finding | none: fails closed when the manifest or tree is missing | script JSDoc + README bin list | unit spec + a run on each spoke's live manifest |
| manifest field `excludedDirs` (new, optional) | this ticket | internal record dirs the manifest may never list | absent ⇒ nothing beyond default-deny | script JSDoc | unit spec |
| manifest field `treeMounts` (accepted shape) | Brain copy, neomjs/neo-agent-brain#1892 | folders the spoke's own tree holds: routes are `/learn/<leaf id>`, and each mount must match a folder node | exactly one of `treeMount` and `treeMounts`, else a header finding | script JSDoc + README | unit spec + the Brain head run |
| manifest field `sourceLinkRoots` (new, optional) | Brain copy (`ai/**` source links) | repository dirs outside `learn/` a guide may link into; the target must be a path git tracks | absent ⇒ no link may leave `learn/` | script JSDoc + README | unit spec |
| env `NEO_PUBLIC_DOCS_FORBIDDEN_TERMS` (new) | neomjs/neo#21847 terminal predicate (client names) | comma-separated terms; a finding names a term by its position, never its text; the summary line counts the terms checked | unset ⇒ 0 terms, stated on the summary line | script JSDoc + README | unit spec + the live control |

## Decision Record impact

`aligned-with` neomjs/neo Discussion #20920 `OQ-1`: each spoke's CI still runs the lint; only where the code lives changes.

## Acceptance Criteria

- [ ] `neo-agent-skills-public-docs` ships in the package (`bin` + `files`) and passes on the live manifests of `neomjs/devindex` (26 guides) and `neomjs/neo-agent-institution` (7 guides).
- [ ] One unit spec covers every rejection case of both current spoke specs: drift, route prefix, host paths including `file://` and Windows, private repos, a missing link target, an unlisted link target, an excluded record.
- [ ] The spoke id and the excluded dirs are inputs; no spoke name appears in the script.
- [ ] The PR bumps the package version.

## Out of Scope

- Deleting the spoke copies. Each spoke retires its own `buildScripts/lintPublicDocs.mjs` in the PR that bumps its `neo-agent-skills` pin; neomjs/neo-agent-brain#1891 can call the bin directly if this lands first.
- Hub-side validation in `neomjs/pages` (neomjs/pages#158).

## Avoided Traps

- A shared-fixture parity spec across the copies: the audit's pinning clause is for twins whose import is structurally impossible, and here it is not.

## Related

neomjs/neo#21847 · neomjs/devindex#162 · neomjs/neo-agent-institution#1036 · neomjs/neo-agent-institution#1037 · neomjs/neo-agent-brain#1891 · neomjs/pages#158

Live latest-open + closed sweep: checked the latest 20 open + open and closed search ("public docs lint", "public-manifest", "lintPublicDocs", "shared script spoke") in this repository at 2026-11-22T02:03Z, plus an org-wide open search; no equivalent or prior exclusion found. Backlog: 41 open (orange); no owning epic here, the outcome epic is neomjs/neo#21847.
A2A in-flight claim sweep: the 30 newest messages at 02:02Z; the adjacent claim is neomjs/neo-agent-brain#1891 (01:48Z), none on a shared lint.
MC sweep: 2 queries on the problem's nouns (CI guard scripts copied per repository, diverging), 9 results, no prior decision found.
Own-assignment sweep: 3 open, none overlapping.
Structure map: n/a (Brain-hosted); sibling precedent `scripts/check-discussion-fidelity.mjs`.

Origin Session ID: 4b82310c-7c8c-4e15-b6ff-0acb8919e99e
Retrieval Hint: `lintPublicDocs copied per spoke; neo-agent-skills-public-docs bin; public-manifest excludedDirs`

## Timeline

- 2026-11-22T02:03:57Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-22T02:03:59Z @neo-opus-grace added the `enhancement` label
- 2026-11-22T02:03:59Z @neo-opus-grace added the `ai` label
- 2026-11-22T02:04:00Z @neo-opus-grace added the `refactoring` label
- 2026-11-22T02:04:00Z @neo-opus-grace added the `build` label
- 2026-11-22T02:05:31Z @neo-opus-grace cross-referenced by PR #1037
### @neo-opus-grace - 2026-11-22T02:08:36Z

**Handover (Grace, still the owner).** Not started: the session that filed this ended at a quiet point before the lane, so the next session picks it up.

**The copies, as of 02:10Z:**
- `neomjs/devindex` `buildScripts/lintPublicDocs.mjs`: merged (neomjs/devindex#162, `fd98ff5ca7`). Its reviewed head `463267a` is the base to port: link targets resolve only to manifest-admitted guides, `file://` host paths fail, and the route prefix derives from `treeMount.id`.
- `neomjs/neo-agent-institution` `buildScripts/lintPublicDocs.mjs`: neomjs/neo-agent-institution#1037, changes requested (restore the dropped JSDoc; correct its Deltas claim). It adds `EXCLUDED_SUBDIRS`, which becomes the manifest's `excludedDirs` here.
- `neomjs/neo-agent-brain` `ai/scripts/lint/lint-public-docs.mjs`: neomjs/neo-agent-brain#1892, open, a third copy.

**The fork with Julia** (DM): (a) the spokes ship their copies now and all retire with this bin, or (b) they wait for it. My recommendation is (a): a Skills release cannot merge before the operator is back, and the Brain restore should not wait on it. The bin is needed either way, so the work does not depend on her answer.

**Pickup:**
1. `ticket-intake` (self-authored drift probe), then a `[lane-claim]`.
2. Branch off `dev`. Port the union into `scripts/check-public-docs.mjs`, add the bin and `files` entries, and port both spokes' specs into one.
3. Bump the version in the same PR. Run this repository's CI lint command locally against `origin/dev`, not only the unit tests.
4. Acceptance probe: run the bin against devindex `dev` (26 guides) and the Institution head (7 guides).
5. Request a Gemini reviewer, since the lint is Julia's design.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 4b82310c-7c8c-4e15-b6ff-0acb8919e99e

- 2026-11-22T02:31:22Z @neo-opus-grace cross-referenced by PR #435
- 2026-11-22T02:42:01Z @neo-opus-grace referenced in commit `210d4b3` - "docs(scripts): a source link root must be one the hub's rewriters map back to the spoke (#434)

The Portal's rewriteRepoLinks and middleware-v2's LinkHelper map ai/ to neo-agent-brain and every other root to neo, so a sourceLinkRoots entry outside that mapping passes the lint and still renders as a dead link."
- 2026-11-22T05:39:45Z @tobiu referenced in commit `c9fd789` - "feat(scripts): one public-docs lint serves every spoke, as bin neo-agent-skills-public-docs (#434) (#435)

* feat(scripts): one public-docs lint serves every spoke, as bin neo-agent-skills-public-docs (#434)

Each public spoke carried its own copy of the default-deny publish lint, and the
copies drifted within an hour of the first port. The bin holds the union of the
devindex, Institution and Brain copies and takes every spoke-specific value as
input: the spoke id from --spoke, internal record dirs from the manifest's
excludedDirs, link roots outside learn/ from sourceLinkRoots, and client names
from NEO_PUBLIC_DOCS_FORBIDDEN_TERMS. It accepts both mount shapes in use
(treeMount, Brain's treeMounts).

Three changes against the copies: a source link must reach a path git tracks
rather than one on disk, a finding names a forbidden term by its position so a
public CI log never repeats a client name, and each scan reads a line once.

Minor bump to 2.4.0: a new bin that consumers opt into.

* docs(scripts): a source link root must be one the hub's rewriters map back to the spoke (#434)

The Portal's rewriteRepoLinks and middleware-v2's LinkHelper map ai/ to neo-agent-brain and every other root to neo, so a sourceLinkRoots entry outside that mapping passes the lint and still renders as a dead link."
- 2026-11-22T05:39:47Z @tobiu closed this issue

