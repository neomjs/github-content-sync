---
id: 209
title: Host service definitions adopt agentosRuntimeRoot and guard the root
state: CLOSED
labels:
  - bug
  - ai
  - build
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-08-28T16:56:43Z'
updatedAt: '2026-08-28T19:52:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/209'
author: neo-opus-vega
commentsCount: 0
parentIssue: 12
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-28T19:52:27Z'
---
# Host service definitions adopt agentosRuntimeRoot and guard the root

## Context

Narrow delivery leaf for #12's **AC-8** — *"Host plist/service definitions reference `agentosRuntimeRoot`; installation remains explicitly operator-owned."* #12 is the broad deployment-receive leaf with nine other ACs open, so it must not be the close target; this leaf carries the one already-delivered slice so the graph has an honest anchor.

Filed after implementation deliberately: the work is in flight at PR #208 (CI 5/5, `mergeStateStatus: CLEAN` at `96bbb12`) and the reviewer required a narrow close target rather than an unanchored agent PR.

Both host LaunchAgent templates named their `WorkingDirectory` placeholder `__NEO_REPO_ROOT__`, while ADR 0040 §2.5 *"Two root authorities, never one"* defines **`agentosRuntimeRoot`** (`learn/agentos/decisions/0040-agentos-extraction-topology.md:135`) as *"where the Agent OS itself is installed and runs"*.

## The Problem

This is a **naming** defect, not a broken path, and the distinction decides the size of the fix. The documented runbook derives the root with `export … "$(pwd -P)"`, so an operator running the guide from the Brain checkout — which the guide's own location implies — already produced the correct root. Nothing was mis-resolving in practice.

What the Engine-flavoured name left behind was a gap in the install guard:

- both plists invoke their entrypoint **relative** to `WorkingDirectory` (`ai/daemons/wake/receiver.mjs`, `ai/daemons/orchestrator/hostEdge.mjs`);
- the Engine repo no longer carries an `ai/` tree at all — measured: **0** entries under `ai/` at engine `origin/dev`, both entrypoints `ABSENT` there and `PRESENT` in Brain;
- the post-substitution check asserts only that no `__` placeholder survived, and `plutil -lint` reports `OK` on a plist whose `WorkingDirectory` is merely the wrong root.

Both checks pass, `launchctl bootstrap` succeeds, and the agent silently never launches. The runbook already documents this exact failure class for array-index replacement (*"there is no install-time diagnostic and the failure only shows up as an agent that silently never runs"*) — the wrong-root case is the same shape in a different dimension, which is why the rename ships with the assertion that closes it.

## The Architectural Reality

- `deploy/host/com.neomjs.agent-os-wake.plist` and `deploy/host/com.neomjs.agent-os-host-edge.plist` are placeholder templates; the operator materializes them outside the repository.
- `ai/scripts/lifecycle/local-agent-os/README.md` owns the substitution procedure: `plutil -replace WorkingDirectory -string "${…}"` by dictionary key, followed by `plutil -lint` and a placeholder-survival assertion.
- Substitution is **operator-owned** by design (ADR 0040 §2.5 assigns the runtime root; privileged install stays outside CI), so the guard belongs in the runbook block, not in a repo-side check.
- Relative `ProgramArguments` are re-asserted by the procedure itself (`plutil -replace ProgramArguments -json "[…, \"ai/daemons/…\"]"`), so `WorkingDirectory` is load-bearing for correctness rather than cosmetic.

## The Fix

1. Rename the placeholder to `__AGENTOS_RUNTIME_ROOT__` in both templates, with a comment recording why the root is not the Engine clone.
2. Rename the runbook variable to `AGENTOS_RUNTIME_ROOT`, citing ADR 0040 §2.5 and the relative-resolution reason.
3. Add an entrypoint assertion to **both** install blocks, after the existing placeholder check, so a wrong-root install fails loud at install time instead of at launch.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `deploy/host/*.plist` `WorkingDirectory` placeholder | ADR 0040 §2.5 (`0040-…:135`) | named `__AGENTOS_RUNTIME_ROOT__`; comment states the root is not the Engine clone | none — a template placeholder has no runtime fallback | the local-agent-os runbook | `plutil -lint` OK on both; token present once per template |
| local-agent-os runbook substitution block | ADR 0040 §2.5; operator-owned install | `AGENTOS_RUNTIME_ROOT` replaces `NEO_REPO_ROOT`; asserts the resolved root carries the invoked entrypoint | fails closed before `launchctl bootstrap` | same file | positive control at Brain root PASS, negative control at Engine root FAIL |
| installed LaunchAgent behaviour | launchd | unchanged | n/a | n/a | templates are copied at install time; running agents unaffected until the operator re-runs the block |

## Decision Record impact

`aligned-with ADR 0040` — adopts §2.5's existing vocabulary at a surface that predated it. No ADR text changes; no amendment or supersession.

## Acceptance Criteria

- [ ] Both host plist templates name the runtime root with the `agentosRuntimeRoot` vocabulary; zero `NEO_REPO_ROOT` references remain anywhere in the repository.
- [ ] Every new ADR citation on the changed surfaces reads **§2.5** (the section that defines `agentosRuntimeRoot`), not §2.7 (custody).
- [ ] Both plist templates still pass `plutil -lint`.
- [ ] Both runbook install blocks assert that the resolved runtime root carries the entrypoint that plist invokes, positioned after the existing placeholder-survival check.
- [ ] That assertion is proven in both directions: it passes with the Brain root and fails with the Engine root.
- [ ] Installation remains explicitly operator-owned — no repo-side or CI-side install step is introduced.
- [ ] *(post-merge, operator-observable)* on the next host install, `plutil -p` on both installed plists shows a `WorkingDirectory` containing `ai/daemons/`.

## Out of Scope

- `deploy/cloud/Dockerfile` and the AC-2/AC-5 build topology — ruled to Shape B (nested Cloud package owns the build) and tracked under #12.
- AC-7's stale rollback bundle; Wave-0 bundle/pin work.
- Installing or restarting any LaunchAgent.
- Any change to `hostEdgeProfile.mjs` posture or the plists' environment keys.

## Avoided Traps

- **Treating it as a broken path and rewriting the derivation.** `$(pwd -P)` from the Brain checkout was already correct; the oversized fix would have replaced a working derivation and obscured the actual gap.
- **Trusting `plutil -lint` as path proof.** It validates structure; it reports `OK` on a syntactically perfect plist pointing at a root that cannot serve it.
- **Shipping a guard without proving it can fail.** A guard tested only in the passing direction cannot be distinguished from an inert one, so the negative control at the Engine root is part of the evidence, not decoration.
- **Moving installation into CI to make it verifiable.** Privileged install is operator-owned; the assertion had to live in the operator's own block.

## Related

- Parent (non-closing): #12 — AC-8 is one of its ten ACs.
- PR #208 — the delivered implementation.
- #198 — *Replace Engine root projections with package imports*: the same post-split family, a different surface (module resolution vs. host service definitions). Checked for overlap; the plist `WorkingDirectory` contract is not addressed there.
- ADR 0040 §2.5 — `agentosRuntimeRoot` definition and the two-root authority.

Live latest-open sweep: checked the latest 20 open issues at 2026-08-28T16:55:35Z plus a 30-message A2A recency scan; no equivalent ticket and no competing in-flight claim.

Origin Session ID: 182fafce-e5d1-418b-afba-a96e90312a4c

Retrieval Hint: `query_raw_memories("agentosRuntimeRoot host plist wrong-root install guard AC-8")` · commit anchor `96bbb12`

## Timeline

- 2026-08-28T16:56:44Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-28T16:56:45Z @neo-opus-vega added the `bug` label
- 2026-08-28T16:56:45Z @neo-opus-vega added the `ai` label
- 2026-08-28T16:56:45Z @neo-opus-vega added the `build` label
- 2026-08-28T16:56:46Z @neo-opus-vega added the `agent-os` label
- 2026-08-28T16:57:14Z @tobiu referenced in commit `0c0dc63` - "docs(deploy): cite the ADR section that defines agentosRuntimeRoot (#209)

The runtime-root vocabulary is defined in ADR 0040 §2.5 ("Two root
authorities, never one", the definition sits at line 135 of the ADR); §2.7 is
the later custody section. All three changed surfaces cited §2.7, so a
mechanically correct guard would have taught the wrong coordinate on every
future read.

Citation only — the runtime-root mechanism, the placeholder rename and both
entrypoint assertions are unchanged.

Raised as RA-1 by @neo-gpt-emmy in the PR 208 review; verified independently
against the ADR's section boundaries before applying."
- 2026-08-28T16:58:27Z @tobiu cross-referenced by PR #208
- 2026-08-28T19:52:27Z @tobiu closed this issue
- 2026-08-28T20:15:32Z @neo-opus-vega cross-referenced by #12

