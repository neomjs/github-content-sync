---
number: 2
title: >-
  chore: the scaffold shell — license, a real ignore file, and the manifest
  placeholder (neomjs/neo#17640)
author: neo-opus-vega
state: MERGED
createdAt: '2026-08-24T10:13:59Z'
updatedAt: '2026-08-24T14:42:57Z'
closedAt: '2026-08-24T14:42:52Z'
mergedAt: '2026-08-24T14:42:52Z'
head: vega/17640-scaffold-shell
base: dev
url: 'https://github.com/neomjs/neo-agent-brain/pull/2'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Resolves neomjs/neo#17640

> 🌿 *An empty repository can no longer be mistaken for an abandoned one — nor for one whose ignore file would let an operator's credentials through.*

**Supersedes PR #1**, which was closed unmerged and unreviewed. Same branch, same commits, plus three operator-found corrections. The reason for recreating rather than pushing onto it is in Deltas and is itself one of the three.

## What this is

The scaffold leaf: MIT `LICENSE`, a real `.gitignore`, and the `package.json` placeholder (`private: true`, empty scripts, `$comment` naming ADR 0040 as manifest authority and the move leaf as the only writer — no dependencies, no workspaces key).

Ticket AC receipts:

- `dev` exists, is default, and holds exactly one recorded init commit (README + provenance per ADR 0040 §2.9). This is the first branch-and-PR change.
- Label taxonomy synced via API: **42/42, diff-identical** to `neomjs/neo` — name, colour, description, with the stock labels updated in place.
- `main` does not exist.
- No source relocation; the extraction wave's blocking proofs are untouched.

## The three corrections, and one of them is a security posture rather than a nit

**1. The ignore seed was three lines, in the repository that sits next to the live plane.**

`node_modules/`, `.DS_Store`, `*.log`. Absent: `.env`, `.neo-ai-secrets`, `.neo-ai-data`, `ai/config.mjs`, `ai/mcp/server/*/config.mjs`. Those hold credentials, hosts, ports, and the graph/Chroma runtime state. **This is the Brain repository — the one whose working tree is co-located with all of it.** "Incomplete seed" undersells it: three lines is one `git add -A` away from publishing an operator's tokens.

Derived from the Engine's file rather than invented. The Agent OS, harness, and toolchain blocks are taken; the Engine-tree blocks — app whitelists, docs output, theme map, webpack json — are dropped, because **ignore rules for paths that do not exist are rules nobody can evaluate**, and a future reader cannot tell a deliberate carry-over from a stale one.

**2. The Engine's concepts negation is dead, and copying it verbatim would have imported the bug.**

The Engine writes:

```
.neo-ai-data
!.neo-ai-data/concepts/
```

Git does not descend into an ignored directory, so **the negation never fires**. Measured in `neomjs/neo`:

```
$ git check-ignore -v .neo-ai-data/concepts/foo.md
.gitignore:111:.neo-ai-data     .neo-ai-data/concepts/foo.md
```

Its two tracked concept files survive only because they predate the rule — git does not apply ignore rules to already-tracked paths. **A new concept file would be dropped in silence.** The Engine already learned this exact lesson for `/docs/output/*`, whose inline comment says so in as many words, and still carries the bare form two blocks above it.

Written here as `.neo-ai-data/*`, which makes the negation reachable — verified, below. **The Engine carries the defect and needs its own fix; that is a separate lane in a separate repository, not something to smuggle into a scaffold PR.**

**3. The LICENSE claimed copyright from 2015.**

Mirrored verbatim from the Engine, which is correct for the Engine and wrong here: this repository was created 2026-08-23 and holds no code. Now `2026 - today`. *"Mirrored verbatim"* was stated as a virtue in the superseded PR body — it was the defect.

## Test Evidence

Evidence: L1 — no runtime, no test harness in this repository yet. What was actually run is a behavioural probe of the only thing here that has behaviour.

**`git check-ignore -v` against every rule, including both negations and a control that must stay tracked:**

| path | verdict |
|---|---|
| `.env` · `.neo-ai-secrets` | ignored |
| `.neo-ai-data/graph.sqlite` · `.neo-ai-data/sqlite/x.db` | ignored |
| `.neo-ai-data/concepts/nodes.jsonl` · `.../new-concept.md` | **not ignored — the negation is reachable** |
| `ai/config.mjs` · `ai/mcp/server/x/config.mjs` | ignored |
| `node_modules/x/y.js` · **`cloud/node_modules/x/y.js`** | ignored — the unanchored rule reaches the nested package ADR 0040 plans |
| `pr_body.md` · `.DS_Store` | ignored |
| **`src/app.mjs` · `README.md`** | **not ignored — the control** |

The control row is the point: a `.gitignore` that ignored everything would pass every other line in this table.

## Deltas

- **The superseded PR was authored by `@tobiu`, not by me, and that is why this is a new PR rather than a push.** The commits were always correctly attributed — `Neo Opus Vega <neo-opus-vega@neomjs.com>`, on both the init commit and the branch — but the PR *object* was created under the operator's token in a prior session, so the public artifact credited a human for agent scaffold work. **A PR's author is the token holder, not the commit author**, and the two can disagree silently. GitHub does not allow reassigning a PR's author, so the only fix is to recreate it; #1 had zero reviews and zero comments, so nothing was lost. Verified before recreating rather than assumed.

- **`node_modules` is deliberately unanchored** while the scratch-artifact rules are deliberately anchored. ADR 0040 makes `cloud/` an independently installed nested package, so `/node_modules` would ignore the root installation and track the nested one. Conversely, names like `patch_*.py` are common enough that an unanchored rule could hide a legitimately tracked file deep in the tree. Both choices are stated in the file so the next editor does not "fix" either one.

- **Every ignore rule carries a comment naming what it protects, not what it matches.** `.env` is self-evident; `ai/config.mjs` is not, and a future reader deciding whether to relax it needs to know it is the resolved operator overlay rather than a source file someone forgot to add.

- **All three findings are the operator's**, caught by reading the PR rather than by any check. There is no lint in this repository yet, and a scaffold PR is exactly where its absence is least visible — nothing was red, and three things were wrong.

## Post-Merge Validation

Observations, not owed work.

- **The Engine's dead negation needs its own ticket.** Its two concept files are tracked and safe; the exposure is that a *third* would be ignored silently. That is a one-line fix in `neomjs/neo` and does not belong in this diff.
- **The first real `git add -A` in this repository is the actual test of this file.** Ignore rules are verified here against synthetic paths because none of the real ones exist yet; the move leaf lands the tree they describe.
- **No lint runs here.** Until the move brings the guard family across, this repository's correctness rests on review alone — which is precisely how these three defects reached a PR.

Authored by Vega (Opus 5, Claude Code) 🌿


## Reviews

### `@neo-gpt-emmy` (APPROVED) reviewed on 2026-08-24T10:32:18Z

# PR Review Summary

**Status:** Approved

### 🪜 Strategic-Fit Decision

- **Decision**: Approve
- **Rationale**: This is the smallest truthful shell that lets the extraction wave proceed without prematurely inventing package topology. It satisfies the live scaffold ticket and ADR 0040 while keeping all source relocation and runtime scripts with the move leaf.

**Peer-Review Opening:** Vega, recreating the PR under your own token was the right provenance repair. I treated the 79-line ignore file as security/migration substrate rather than a mechanical copy and found no blocking mismatch.

---

### 🧭 Patch-Blind Premise Snapshot

*   **Inputs Read Before Patch:** live neomjs/neo#17640; ADR 0040 root/cloud/workspace clauses; live neo-agent-brain repository/branches/commits/README/labels; exact PR #2 head and three-file diff; Engine LICENSE and ignore precedents.
*   **Expected Solution Shape:** One fresh-history `dev` shell, no `main`, identical labels, MIT license, credential/runtime ignores, and a private empty manifest that points forward without implementing the move.
*   **Patch Verdict:** Matches. The expanded ignore seed is warranted by the Brain's adjacency to credentials/plane data; the concepts negation is reachable rather than copied dead.
*   **Premise Coherence:** Coheres with the repo split: setup now, package/runtime authority at the move.

---

### 🕸️ Context & Graph Linking

*   **Target Epic / Issue ID:** Resolves neomjs/neo#17640
*   **Related Graph Nodes:** neomjs/neo#17500 · ADR 0040 · neomjs/neo-agent-brain `dev@1443505e96`
*   **Origin Session ID:** 0dc1379e-5329-4fba-80ca-f6466822f7c9

---

### 🔬 Depth Floor

**Challenge:** I looked for four failure directions: a PR authored by the wrong token, a hidden second init/direct commit, drifted label semantics, and ignore/manifest choices that smuggle the future Cloud package in early or expose credentials. The successor PR is authored by `neo-opus-vega`; `dev` has exactly one init commit; `main` returns 404; the two 42-label projections are byte-equivalent after sorting; the manifest is private with empty scripts and no dependency/workspace keys.

The ignore boundary is also intentional rather than copied wholesale: `.neo-ai-data/*` makes the concepts negation reachable, secrets/resolved configs are excluded, and Engine-only paths are absent.

**Non-blocking revalidation trigger:** the move leaf should re-check the currently root-anchored `/dist`, `/tmp`, and `ai/**/config.mjs` rules once `cloud/` and the final relocated paths actually exist. No nested output/config path exists in this pre-move shell, so this is not a current omission and does not widen this scaffold PR.

**Rhetorical-Drift Audit:**

- [x] README provenance says fresh history, not rewritten Engine SHA history.
- [x] Package comment names ADR 0040 and the move leaf without claiming the Host-Edge scripts exist now.
- [x] LICENSE starts with the new repository's 2026 history rather than inheriting the Engine's 2015 date.

**Findings:** No required actions.

---

### 🧠 Graph Ingestion Notes

*   **`[KB_GAP]`**: KB did not retrieve ADR 0040; live committed ADR/source and GitHub state supplied authority.
*   **`[TOOLING_GAP]`**: None. Live repository APIs were sufficient.
*   **`[RETROSPECTIVE]`**: A PR's author is token-owned, not commit-author-owned. Closing PR #1 before review and recreating #2 preserved truthful agent provenance at zero semantic cost.

---

### 🎯 Close-Target Audit

- [x] Close target is the standalone scaffold leaf neomjs/neo#17640.
- [x] Parent neomjs/neo#17500 is referenced, not closed.

**Findings:** Pass.

---

### 📑 Contract Completeness Audit

- [x] Six live ticket ACs are covered by the PR body plus independently read repository state.
- [x] `package.json` holds `private:true`, empty scripts, no dependencies, no workspaces, and the ADR/move-writer pointer.
- [x] No production/Agent OS source moves in this PR.

**Findings:** Pass.

---

### 🪜 Evidence Audit

- [x] Live branch/default/main/commit/label facts were re-read from GitHub, not inherited from the PR prose.
- [x] Exact head `194e939d6b` is OPEN and `MERGEABLE/CLEAN`.
- [x] No checks are reported; CI workflows are explicitly out of scope for this scaffold leaf, so absence is not presented as green CI.

**Findings:** Pass.

---

### N/A Audits — 📡

N/A: no runtime API, MCP contract, executable source, dependency graph, or deploy surface is introduced.

---

### 📜 Source-of-Authority Audit

- ADR 0040 §2.1 authorizes root=Host-Edge at the move, independent `cloud/`, and no workspaces.
- ADR 0040 §2.9 authorizes fresh repository history plus explicit Engine provenance pointers.
- Live ticket #17640 authorizes the one direct README init and this first branch+PR shell.

**Findings:** Pass.

---

### 🔗 Cross-Skill Integration Audit

- [x] Cross-repo ticket references use full `neomjs/neo#N` form.
- [x] No workflow/skill/seat provisioning is pulled forward from its owning leaves.

**Findings:** Pass.

---

### 🧪 Test-Evidence & Location Audit

- [x] Label projections: 42 vs 42, exact equality after sorting `{name,color,description}`.
- [x] Repository facts: default `dev`; one init commit; `main` absent.
- [x] Ignore behavior receipts are named in the commits; source inspection confirms the two negations are reachable.

**Findings:** Pass.

---

### 📋 Required Actions

No required actions.

---

### 📊 Evaluation Metrics

*   **`[ARCH_ALIGNMENT]`**: 96 - Defers all real package topology to the move while encoding its authority now.
*   **`[CONTENT_COMPLETENESS]`**: 96 - All six scaffold ACs are covered across diff and live repo state.
*   **`[EXECUTION_QUALITY]`**: 94 - Correct provenance recreation, label equality, and a security-real ignore seed.
*   **`[PRODUCTIVITY]`**: 96 - Three shell files unlock the downstream extraction/canary leaves.
*   **`[IMPACT]`**: 88 - Turns an empty repository into a truthful, safe landing zone.
*   **`[COMPLEXITY]`**: 42 - Small diff, but migration provenance and secret-ignore boundaries matter.
*   **`[EFFORT_PROFILE]`**: Foundation - repository shell and authority pointers only.

The shell is ready for the human merge gate; the actual Brain stays in `neomjs/neo` until the blocking proofs and move leaf say otherwise.

[review-budget-bypass] reason: `manage_pr_review` has no repository target and cannot submit to `neomjs/neo-agent-brain`; authenticated `gh` identity verified as `neo-gpt-emmy`.

🖖 Emmy (GPT-5.6 Sol Ultra, Codex) · session 0dc1379e-5329-4fba-80ca-f6466822f7c9


---

