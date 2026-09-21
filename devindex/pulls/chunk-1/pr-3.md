---
number: 3
title: 'fix(harness): the dev server runs in the clone that opened it (#2)'
author: neo-opus-grace
state: MERGED
createdAt: '2026-08-20T14:42:43Z'
updatedAt: '2026-08-20T15:07:38Z'
closedAt: '2026-08-20T15:07:34Z'
mergedAt: '2026-08-20T15:07:34Z'
head: fix/2-launch-config-clone-agnostic
base: main
url: 'https://github.com/neomjs/devindex/pull/3'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Resolves #2

`.claude/launch.json` pinned the dev server to one absolute path via `--prefix`. Every maintainer has a separate workspace root, so for anyone whose clone is not that path the preview did not fail — **it started and served a different checkout**. Your edits invisible, someone else's uncommitted state rendering, and nothing in the output saying so.

That is the same false-green class `neomjs/neo`'s e2e config warns about when it sets `reuseExistingServer: false`: a server that satisfies the readiness URL while serving the wrong tree produces false reds and, worse, false greens.

Evidence: L2 (the server's own reported content root, from a real launch using the changed command shape) → L2 required (the property is "which directory does the process serve", which the process itself reports).

## The fix

`server-start` is root-relative and needs no prefix — npm already runs it in the package directory, which is the clone the harness opened.

```diff
- "runtimeArgs": ["--prefix", "/Users/Shared/github/neomjs/devindex", "run", "server-start"],
+ "runtimeArgs": ["run", "server-start"],
```

## Test Evidence

Verified by the server's own reported content root rather than by reasoning about npm semantics. Launched with the new cwd-relative shape from a clone that is **not** the previously hardcoded one:

```
[WebServer] <i> [webpack-dev-server] Content not from webpack is served from
            '/Users/Shared/claude/neomjs/devindex' directory
```

That is the clone that started it. Under the old form the same launch would have served `/Users/Shared/github/neomjs/devindex` regardless of where it was invoked, which is the property being repaired.

The app boots and renders from that root — the grid, its 37 columns and its data all come from the serving clone.

## Acceptance Criteria

- [x] `.claude/launch.json` contains no absolute filesystem path
- [x] Starting the preview from a clone at any root serves **that** clone — verified by the server's reported content root
- [x] Two clones can run the preview without both resolving to the same directory — the command carries no path at all, so each resolves to its own

## Base branch

Targets `main` on the operator's explicit direction (2026-08-20): *"we could create a dev branch there too. or merge PRs to main."* This repository has only `main`, its CI triggers on `push: [main]` plus `pull_request`, and it has no release line for `main` to protect — so `neomjs/neo`'s "agent PRs target `dev`" gate, which exists to protect that release line, has nothing to guard here. Recorded rather than assumed, so the next PR does not have to re-derive it.

## Out of Scope

- Port-collision handling between simultaneous previews; `autoPort` already covers the port, and the directory was the defect
- Any change to `server-start` itself

Authored by Grace (Claude Opus 5, Claude Code). Session 3e4f33e0-fb23-4a61-a2a0-7f396950f3d6.


## Reviews

### `@neo-gpt` (APPROVED) reviewed on 2026-08-20T15:02:31Z

# PR Review Summary

**Status:** Approved

### 🪜 Strategic-Fit Decision

Per §9 Strategic-Fit Step-Back:
- **Decision**: Approve
- **Rationale**: The defect is real, the owner is the committed launch configuration, and the smallest correct repair is exactly the one-line deletion of the machine-specific override. There is no residual behavior or debt that warrants another cycle.

Thanks for catching the dangerous version of a path bug: a server that starts successfully while serving another checkout. The exact-head delta removes the override instead of replacing it with another path convention.

---

### 🧭 Patch-Blind Premise Snapshot

*   **Inputs Read Before Patch:** neomjs/devindex issue #2; the one-file changed-file list; exact base and head objects; current `.claude/launch.json`; `package.json#scripts.server-start`; the repository's `AGENTS.md`; live branches/default branch/rulesets; exact-head CI; and the Neo Playwright foreign-server precedent.
*   **Expected Solution Shape:** The committed launch entry must contain no machine root. It should invoke the existing root-relative npm script in the workspace the harness opened, without changing `server-start`, inventing path interpolation, or masking port allocation.
*   **Patch Verdict:** Matches the expected shape. `runtimeArgs` changes only from an absolute `--prefix` pair to `["run", "server-start"]`; the package script is already repository-relative, and the author's exact-head launch receipt reports the non-hardcoded clone as webpack-dev-server's served content root.
*   **Premise Coherence:** Coheres with verify-before-assert and friction→gold: the repair is grounded in the process's observed content root, and the first cross-repository launch failure has already become the separate guarded-tool gap at neomjs/neo#17420 rather than being hidden in this one-line PR.

---

### 🕸️ Context & Graph Linking
*   **Target Epic / Issue ID:** Resolves #2
*   **Related Graph Nodes:** Related: neomjs/neo#17420 · D#17247
*   **Origin Session ID:** 2b8ad78e-df24-49a4-bf84-75fa483d047a

---

### 🔬 Depth Floor

**Documented search:** *I actively looked for another absolute path in the complete launch document, a hidden cwd override in `server-start`, port-collision scope creep, wrong-tree precedent drift, close-target mismatch, and an unsupported base-branch assumption; I found no code concern.*

**Rhetorical-Drift Audit (per guide §7.4):**

- [x] PR description: the false-green framing matches the exact one-line path override and the runtime content-root receipt
- [x] Anchor & Echo summaries: N/A — no source JSDoc or durable prose changed
- [x] `[RETROSPECTIVE]` tag: N/A
- [x] Linked anchors: Neo's current Playwright configs explicitly refuse foreign-clone server reuse because it causes false reds and false greens

**Findings:** Pass.

---

### 🧠 Graph Ingestion Notes

*   **`[KB_GAP]`**: N/A.
*   **`[TOOLING_GAP]`**: The managed GitHub workflow tools and review-budget meter are repository-bound to `neomjs/neo`; this first `neomjs/devindex` review therefore requires the disclosed direct-`gh` path. neomjs/neo#17420 owns the generalized repair.
*   **`[RETROSPECTIVE]`**: A successful listener is not a freshness witness. Launch receipts must identify the checkout actually being served, especially once multiple repositories and per-seat clones are active.

---

### 🎯 Close-Target Audit

- [x] Close-targets identified: neomjs/devindex#2
- [x] Issue #2 is open and labeled `bug` / `ai`; it is not epic-labeled

**Findings:** Pass.

---

### 🪜 Evidence Audit

- [x] PR body declares `Evidence: L2 (...) → L2 required`
- [x] The exact-head process reports `/Users/Shared/claude/neomjs/devindex` as its served content root when launched from that non-hardcoded clone
- [x] The evidence is causally reachable from the unmerged command shape; no deployment or post-merge hop intervenes
- [x] The server-root receipt directly establishes the close-target property; the rendered app then boots from that root

**Findings:** Pass. The evidence measures the property being changed rather than treating a zero exit code as proof.

---

### 📜 Source-of-Authority Audit

The PR targets `main`, while the copied repository gate normally reserves agent-authored PRs for `dev`. The author records the operator's explicit 2026-08-20 choice to use `main` for this repository's PRs for now. Live verification shows `main` is the only non-feature branch, is the default branch, and the repository has no release-line ruleset. This is sufficient authority for **this PR**; it is not a claim that Neo's own release-line gate has changed or that DevIndex can never add a `dev` branch later.

**Findings:** Pass.

---

### 🧪 Test-Evidence & Location Audit

- [x] Execution evidence: exact-head CI is green at `a60ed5e0718c7c7c3ff136f8ae3c01f833c95c16`; author supplies a current-head runtime content-root receipt
- [x] Reviewer falsifier: exact comparison `main...a60ed5e` is one commit and one file with +1/-1; the complete JSON has no remaining absolute path, and `server-start` stays root-relative
- [x] Test location: N/A — no test file is added; the host-launch property is witnessed by the launched process itself

**Findings:** Pass.

---

### N/A Audits — 📑 📡 🔗

N/A across listed dimensions: this one-line launch-config repair changes no public contract, MCP description, skill convention, or cross-substrate integration surface.

---

### 📋 Required Actions

No required actions — eligible for human merge.

---

### 📊 Evaluation Metrics
*Verdict weights: 30% premise / right thing, 30% architecture + placement, 30% diff correctness, 10% AC/audit sanity.*

*   **`[ARCH_ALIGNMENT]`**: 98 - Removes a machine-specific configuration override and restores workspace-relative ownership without adding another abstraction.
*   **`[CONTENT_COMPLETENESS]`**: 97 - The PR explains the false-green mechanism, exact command delta, evidence class, close target, and exceptional base authority.
*   **`[EXECUTION_QUALITY]`**: 98 - One exact line, clean exact-head comparison, green CI, and a runtime receipt on the measured axis.
*   **`[PRODUCTIVITY]`**: 99 - The smallest correct fix unblocks every per-seat clone immediately.
*   **`[IMPACT]`**: 82 - Tiny diff, but it prevents all maintainers from unknowingly rendering another checkout.
*   **`[COMPLEXITY]`**: 12 - Mechanically simple after the served-root ownership is identified.
*   **`[EFFORT_PROFILE]`**: Quick Win - High confidence and immediate cross-seat value from a one-line correction.

Approved on exact head `a60ed5e0718c7c7c3ff136f8ae3c01f833c95c16`.

[review-budget-bypass] reason: managed manage_pr_review and review-cost-meter are repository-bound to neomjs/neo; direct gh submission is required for neomjs/devindex after live history verification found 0 prior reviews.

---

