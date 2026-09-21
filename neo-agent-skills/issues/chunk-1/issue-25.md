---
id: 25
title: 'Substrate byte-budget guard as a shared baseline job, not per-repo copies'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-08-31T06:45:43Z'
updatedAt: '2026-08-31T09:48:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/25'
author: neo-opus-grace
commentsCount: 0
parentIssue: 14
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-31T09:48:43Z'
---
# Substrate byte-budget guard as a shared baseline job, not per-repo copies

## Context

Sub of #14, and filed against my own mistake: I built this guard as an Engine-local workflow in `neomjs/neo#17911` / PR #17912 — **the exact duplication this Epic exists to end**, two days after the Epic said so. @tobiu caught it against this package's README (*"Consuming repositories depend on this package; they never carry a copy of it"*). PR #17912 is converted to draft and will be superseded by this leaf.

The guard measures the per-turn agent substrate byte budget. Past **24,576 bytes** a harness silently truncates the **bottom** of the file, so a seat loses the tail of its own rules with nothing reporting it — the failure cannot be observed from inside the seat that suffers it.

## The Problem

**Two repositories are at the cliff right now, and one of them has no guard at all.** Measured 2026-08-31 against `dev`, via the GitHub contents API and confirmed against local clones:

| repo | `AGENTS.md` | `.claude/CLAUDE.md` | `.agents/ANTIGRAVITY_RULES.md` | headroom |
|---|---|---|---|---|
| `neo` | **24,574** | **24,574** (symlink → `AGENTS.md`) | 3,734 | **2 bytes** |
| `devindex` | **24,272** | — | — | **304 bytes** |
| `neo-agent-brain` | — | — | — | n/a |
| `neo-agent-skills` | — | — | — | n/a |
| `neo-agent-institution` | — | — | — | n/a |

`devindex` is 304 bytes from silently truncating its own agent rules and nothing is watching it. That is this leaf's argument in one row: a per-repo guard protects the repo whose author happened to write one.

**The absent-target contract is the real design work, and it is why this is not a copy-paste.** My Engine implementation treats a missing target as an **error** (`Required substrate file ${file} not found` → exit 1). Ported unchanged into a shared baseline job, it would red the Brain, Skills and Institution for correctly having no substrate files. A baseline job that three of five consumers cannot adopt is not a baseline.

**Why it must run from outside the caller workspace.** My Engine version does `actions/checkout` then `npm run check-substrate-size`, so it executes the caller's own copy with `PER_FILE_LIMIT_BYTES = 24576` as a constant in that checked-out file. A PR raising the limit, or dropping an entry from `TARGET_FILES`, passes its own guard. This package already solved that for `source-comment-archaeology`: install a pinned release into `runner.temp` and run it from there. My only defence was a sentence in the failure message telling people not to — prose, not a gate.

## The Architectural Reality

- **Precedent to follow, exactly:** `.github/workflows/reusable-pr-baseline.yml` → `source-comment-archaeology`. It checks out the caller head, then `npm install --prefix "${SKILLS_ROOT}" --ignore-scripts --package-lock=false --no-save "neo-agent-skills@${SKILLS_VERSION}"` into `runner.temp`, and invokes `${SKILLS_ROOT}/node_modules/.bin/...` against the caller tree.
- **Sibling scripts:** `scripts/check-ticket-archaeology.mjs` is the shape — a portable guard with a `bin` entry, no repo-local assumptions.
- **The implementation to port** carries work worth keeping: it follows symlinks via `realpathSync` (`lstat` on `neo`'s symlinked `CLAUDE.md` reports **12** bytes — the length of the string `../AGENTS.md` — which is how that surface hid), sums whole-line `@path` imports recursively as one loaded unit with a cycle guard keyed on file identity, and reports **headroom** rather than pass/fail because the drift is gradual.
- **Its entrypoint guard was repaired in review** by @neo-gpt: both operands canonicalized through `realpathSync`, because under `--preserve-symlinks-main` the one-sided form leaves `main()` unreachable and the process exits 0 with nothing measured. Six process-level arms and four verified mutations came with it. All of that ports.

## The Fix

1. `scripts/check-substrate-size.mjs` in this package, with a `bin` entry, porting the Engine implementation and its spec.
2. **Absent target → not applicable.** A target that does not exist is skipped and reported as such, never an error. Only a *breach*, or a target that exists and cannot be measured, fails.
3. A `substrate-size` job in `reusable-pr-baseline.yml` using the `source-comment-archaeology` install pattern — pinned release into `runner.temp`, run against the checked-out caller tree.
4. Stable check name so repository rules can bind it as a required context.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `substrate-size` job in `reusable-pr-baseline.yml` | This package (per #14: Skills owns non-product PR governance) | Runs a pinned guard release from `runner.temp` against the caller tree; fails only on a measured breach | None — the alternative is per-repo duplication, which #14 exists to end | this README's guard section, beside `source-comment-archaeology` | a seeded over-limit fixture reds the job; a repo with no substrate files greens it as N/A; a caller PR raising the limit in its own tree **still reds**, because the limit ships with the pinned guard |
| `neo-agent-skills-substrate-size` bin | `scripts/check-substrate-size.mjs` | Measures loaded bytes per target — symlinks resolved, `@`-imports summed — and prints headroom | Absent target is reported N/A, not failed | script header | ported spec green in this repo, including the `--preserve-symlinks-main` entrypoint arm |

Row anchors verified before assertion: `reusable-pr-baseline.yml`'s job list and the archaeology install steps were read at Skills `dev`; the per-repo sizes are the table above; the symlink/`@`-import behaviour and the entrypoint repair were measured on the Engine branch.

## Decision Record impact

`none` — completes the npm Skills SSOT this Epic already selected. It reverses nothing; it removes an instance of the duplication #14 names.

## Acceptance Criteria

- [ ] `scripts/check-substrate-size.mjs` exists here with a `bin` entry and its ported spec green.
- [ ] A missing target is reported not-applicable and exits 0; a repo with no substrate files greens the job. Asserted for a fixture with zero targets.
- [ ] A measured breach exits non-zero, and the failure names the file, its byte count, and the limit.
- [ ] Symlinked targets are measured as their resolution, not as the link — a `lstat`-only implementation reds a fixture reproducing `neo`'s 12-byte symlink.
- [ ] Whole-line `@path` imports are summed as one loaded unit, with a cycle terminating.
- [ ] The entrypoint guard canonicalizes **both** operands; a `--preserve-symlinks-main` invocation still runs `main()` rather than exiting 0 silently.
- [ ] A `substrate-size` job in `reusable-pr-baseline.yml` installs a pinned release into `runner.temp` and runs it against the caller tree — never the caller's own copy.
- [ ] **A caller PR that raises the limit in its own tree still fails**, proving the gate cannot be weakened by the diff it guards. This is the property my Engine version lacked.
- [ ] `neomjs/neo#17911` is closed as superseded and PR #17912 closed, with the Engine-local workflow and script never merged.

## Out of Scope

- Per-repository caller rollout. #14 states those have distinct owners and merge boundaries; each repo's caller is its own leaf, and the Engine's is the urgent one at 2 bytes.
- Binding the job as a required status context — repository settings, tracked under #14 and `neomjs/neo#17171`.
- Changing the 24,576 limit, or compacting any repo's substrate to fit. `neo` is at 2 bytes and `devindex` at 304; both are decisions for their owners, and **@tobiu's** for `neo` — an agent does not self-edit `AGENTS.md`.

## Avoided Traps

- **Copying the Engine implementation unchanged.** It errors on absent targets, which would make three of five consumers unable to adopt the baseline.
- **Keeping the guard in the caller workspace.** Convenient, and it lets a PR raise its own limit. The archaeology job already shows the correct shape; there is no reason to invent a second one.
- **Trusting a prose deterrent.** My version's defence against limit-raising was the sentence *"Do NOT raise a limit to make this pass."* A rule that only a comment enforces is not enforced.

## Related

Sub of #14. Related: #22, #23 (siblings on the same reusable baseline), `neomjs/neo#17911` and PR #17912 (superseded by this leaf), `neomjs/neo#17175` (the Engine ticket that surfaced the Claude load path and the 2-byte measurement), `neomjs/neo#17171` (required-context binding).

Retrieval Hint: `query_raw_memories` for "substrate size guard shared baseline duplication" · the per-repo byte table and the self-disarm demonstration are recorded on `neomjs/neo` PR #17912.

Origin Session ID: 56bc214a-5b55-41a2-848a-cdfa372abbcd


## Timeline

- 2026-08-31T06:45:44Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-31T06:45:45Z @neo-opus-grace added the `enhancement` label
- 2026-08-31T06:45:45Z @neo-opus-grace added the `ai` label
- 2026-08-31T06:45:45Z @neo-opus-grace added the `architecture` label
- 2026-08-31T06:45:46Z @neo-opus-grace added the `build` label
- 2026-08-31T06:45:56Z @neo-opus-grace added parent issue #14
- 2026-08-31T06:53:25Z @neo-opus-grace cross-referenced by #17911
- 2026-08-31T06:53:42Z @neo-opus-grace cross-referenced by PR #17912
- 2026-08-31T07:11:18Z @neo-opus-grace cross-referenced by PR #26
- 2026-08-31T07:20:53Z @neo-opus-grace cross-referenced by #27
- 2026-08-31T07:55:42Z @neo-opus-grace referenced in commit `3ada4f3` - "fix(ci): N/A means absent only, and the measurement is pinned to the caller root (#25)

@neo-gpt's RA-1 and RA-2, and both are the same defect wearing different clothes:
the negative-space contract that lets three of five consumers adopt this baseline
is also a new way to fail open, and the first implementation took it twice.

RA-1 — a bare `catch` around the presence probe absorbed EVERY filesystem error.
A permission denial, an unreadable mount, a malformed root whose parent component
is not a directory: each reported "not applicable" and exited 0 as a legitimately
empty consumer. Only ENOENT may mean absent now; anything else is an error row
and a non-zero exit.

Absent is a claim about the repository. Unreadable is a claim about the
observation, and an observation that failed supports neither verdict.

RA-2 — the measurement step invoked the binary bare and let the guard resolve its
root from cwd. Outside the checkout every target is ENOENT, so the negative-space
contract reports all-N/A and the job GREENS: a wrong root was indistinguishable
from an empty consumer. `working-directory` is now pinned to the workflow-owned
`github.workspace`, and the contract suite asserts the pin, because it is a
silent failure everywhere else.

RA-3 — README carries the guard beside source-comment archaeology: what it
measures, why loaded bytes differ from file bytes, the ENOENT-only N/A semantics,
and why the limit ships with the guard rather than the caller.

One fixture repaired along the way. `substrate isolated install` anchored its
replacement on the prose comment `# Invoked bare`, because the archaeology job
carries a byte-identical install line and something had to disambiguate. Renaming
that comment turned the mutation into a silent no-op. It now anchors on the step
name, which is contract rather than prose -- a negative fixture coupled to prose
fails the moment the prose is correct-but-different.

Mutation-verified both ways: restoring the bare catch reds the permission arm;
removing the pin reds the workflow-contract arm. 29 negative mutations plus the
guard's contract arms green. `test-lint-skill-corpus` sits at 24/25 both with and
without this change -- pre-existing, not touched here.

RA-4 (evidence-matrix levels) follows in the PR body."
- 2026-08-31T08:53:19Z @neo-opus-grace referenced in commit `9cd57a3` - "fix(ci): anchor the workflow-contract path assertions so a suffixed root reds (#25)

RA-2 on PR #26, from @neo-gpt. The workspace pin was present in the workflow and
asserted in the contract suite — as an UNANCHORED substring, so
`working-directory: ${{ github.workspace }}/docs` satisfied the assertion that
exists to forbid exactly that. A wrong root is the silent failure the pin was
added for: the guard resolves its targets from cwd, so outside the checkout every
target is ENOENT, the negative-space contract reports N/A, and the job greens
having measured nothing.

Three assertions shared the defect, not one. Every path-valued assertion in this
suite was substring-matched, so a suffix produced a different root that passed:
`working-directory`, the substrate `SKILLS_ROOT`, the archaeology `SKILLS_ROOT`,
and the substrate `ref:`. All are now anchored to end-of-line.

Non-vacuity, which is the only thing that makes this a fix rather than a claim:
the three new mutations append a suffix rather than replacing the value. A
wholesale replacement reds against an unanchored pattern too, so it cannot tell
an anchored assertion from an unanchored one — it would have passed before this
change and after it. Reverting ONLY the workspace anchor fails the suite at
exit 1 on exactly `substrate workspace pin — suffixed root` and nothing else;
restored, the suite is green at 32 mutations.

Same class as the #28 defect fixed on PR #30: substring presence cannot express
a positional rule. I shipped it in one guard while fixing it in another."
- 2026-08-31T09:48:43Z @tobiu referenced in commit `64ad325` - "Merge pull request #26 from neomjs/feat/substrate-size-shared-baseline-25

feat(ci): share the substrate byte-budget guard as a baseline job (#25)"
- 2026-08-31T09:48:43Z @tobiu closed this issue
- 2026-08-31T12:21:51Z @neo-opus-grace cross-referenced by #17175
- 2026-09-03T19:47:14Z @neo-opus-grace cross-referenced by #41
- 2026-09-07T00:14:47Z @neo-opus-grace cross-referenced by #56

