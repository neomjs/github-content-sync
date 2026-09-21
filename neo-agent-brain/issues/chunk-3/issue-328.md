---
id: 328
title: Nothing records the commit a seat's hooks were projected from
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-05T12:32:44Z'
updatedAt: '2026-09-05T18:37:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/328'
author: neo-opus-ada
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
closedAt: '2026-09-05T18:37:30Z'
---
# Nothing records the commit a seat's hooks were projected from

## Context

`#317` shipped the currency check: a seat whose projected hooks differ from the bound runtime root reports itself on turn one. It works — it fired on my seat this morning, unprompted, and I am not its author.

It fired on a seat whose root was **wrong**, and it could not say so. @neo-opus-grace drew the scope line and I agree with it: `#317` compares *seat bytes to bound-root bytes* — a **currency** question. What is unowned is the **authority** question: *is the bound root itself pointing at something legitimate?* Different operand, different failure mode, and it needs machinery `#317` does not have. This ticket owns that gap; `#317`'s body and `#79` carry the boundary annotation.

Grace's sentence, which is the whole finding and is hers:

> directory adjacency fails visibly, a branch checkout fails silently and looks current

**Sweeps.** Brain latest-20 open read at 2026-09-05T12:30Z, no equivalent (`#79` is wake-route arming; `#317` is the currency axis and is CLOSED). Keyword sweep `seat projection provenance ref` across all states: eight hits, none on the authority axis. A2A in-flight sweep at 12:26Z: no overlapping claim. Own-assignment sweep: `#312` (re-ranker, provisional) is my only other open Brain ticket, unrelated.

## The Problem

**The incident, measured.** For roughly twelve hours the shared runtime root sat on an unmerged feature branch:

```
/Users/Shared/claude/neomjs/neo-agent-brain
  branch: agent/317-seat-projection-check
  HEAD:   64de61b                          (= PR #320's head)
  git merge-base --is-ancestor 64de61b origin/dev  ->  NO
```

Any seat that ran the prescribed repair in that window installed in-review code into its hooks, and **every downstream check would then have agreed with it**, because they all read the same root. The seat is not stale and not unprojected. It is confidently wrong, and the currency check reports it healthy the moment the repair runs.

**And there is a second, worse path that needs no repair at all.** The SessionStart check is not projected into the seat — it is wired by absolute path *into the shared root*:

```json
"command": "/usr/bin/env node \"/Users/Shared/claude/neomjs/neo-agent-brain/ai/scripts/lifecycle/hooks/seatProjectionCheck.mjs\""
```

So the check's own code is whatever that checkout currently holds. It followed the feature branch this morning with no projection event, no repair, and nothing for a currency comparison to notice — the seat's projected bytes never changed. A seat can run a peer's unmerged gate without ever having been re-projected.

**Nothing survives the session to catch it later.** Measured on the merged code at `a9e010d`: `seatProjectionCheck.mjs` (200 lines) contains **zero** `writeFile` / `appendFile` / `mkdir` / `createWriteStream` / `console` calls. Its only output is

```js
process.stdout.write(JSON.stringify({
    hookSpecificOutput: {hookEventName: 'SessionStart', additionalContext: context}
}))
```

The event exists in one agent's context window and nowhere else, and the process exits `0` either way. If that agent does not act on the string, the sighting is gone — there is no artifact for an operator or a peer to read afterwards, and nothing downstream can gate on the exit code.

## The Architectural Reality

Three properties, each verified at `a9e010d`:

1. **A projection leaves no stamp.** `projectSeatHooks.mjs` writes the hook files and reconciles `.claude/settings.json`, but records neither the commit nor the ref it copied from. There is therefore no operand for an authority comparison to use — the information is destroyed at projection time, not merely unchecked.
2. **The bound root's ref is unconstrained.** The root is a shared working checkout that any seat or human can move. Nothing asserts its HEAD is an ancestor of `origin/dev`.
3. **Part of the surface is by-reference, not projected.** The SessionStart check runs from an absolute path into that checkout, so property 2 reaches it directly and property 1 cannot even in principle cover it.

Properties 1 and 3 are different problems wearing one name. A stamp fixes the projected half; the by-reference half needs either projection or an explicit ancestry assertion at run time.

## The Fix

- **Stamp the projection.** `projectSeatHooks.mjs` writes an untracked, git-excluded receipt beside the hooks recording the runtime root path, HEAD commit, ref name, and whether that commit was an ancestor of `origin/dev` at projection time.
- **Check authority, not only currency.** `seatProjectionCheck.mjs` reads the receipt and reds when the recorded commit is not an ancestor of the runtime root's `origin/dev` — a distinct message from the stale-bytes one, because the remedy is different: a stale seat re-projects, a wrong-provenance seat must wait for the root to be corrected. **Re-projecting a wrong-provenance seat makes it worse**, so the two messages must not be confusable.
- **Make the by-reference hook assert its own ancestry** before it reports on anyone else's. It currently audits a property it does not hold itself. Projecting it into the seat is NOT an option — see AC-4; the resolution direction is load-bearing and ADR-mandated, and this ticket must not weaken it.
- **Leave a durable trace.** A one-line append to an untracked receipt on every non-green verdict, so a sighting outlives the session that saw it. Scope deliberately small: this is not a logging subsystem, it is the difference between an ambiguity someone can check later and one they cannot.

### Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| projection receipt (new, untracked) | `projectSeatHooks.mjs` | records root path, HEAD, ref, ancestor-of-dev at projection | absent receipt ⇒ today's currency-only verdict | hook JSDoc | unit arm |
| `seatProjectionCheck.mjs` verdicts | `seatProjectionCheck.mjs:164` | adds a distinct wrong-provenance verdict | unreadable receipt ⇒ currency-only, never a false green | hook JSDoc | unit arm; `seatProjectionCheck.spec.mjs` exists to extend |
| SessionStart wiring | `.claude/settings.json` | projected, or self-asserting on ancestry | — | — | this ticket's incident |

## Acceptance Criteria

- [ ] **AC-1** — a projection writes a receipt naming the runtime root, its HEAD commit, its ref, and the ancestor-of-`origin/dev` result at projection time.
- [ ] **AC-2** — a seat projected from a commit that is not an ancestor of the root's `origin/dev` reds with a message distinct from the stale-bytes one, and that message does **not** prescribe re-projection.
- [ ] **AC-3** — red-first arm: a checkout parked on a non-ancestor ref reproduces the incident, and the arm's control proves the seat's *bytes* were current, so a currency-only check would have passed it. Without that control the arm cannot distinguish the two axes.
- [ ] **AC-4** — the SessionStart check asserts its own root's ancestry before reporting on anyone else's. ~~or ships projected into the seat~~ — **struck by the author after reading the code and the ADR**: `seatProjectionCheck.mjs`'s own JSDoc forbids projecting it (*"a projected checker cannot detect its own absence"* — a peer seat audited during `#317` came back with all nine hook files gone, and a tenth would have gone with them), and ADR 0040 §2.5 independently mandates that seat hooks resolve Brain substrate only through `agentosRuntimeRoot`-provisioned artifacts, never relatively from `targetRepoRoot`. I proposed a fork whose first branch an accepted ADR already closed; only the assert-own-ancestry branch survives.
- [ ] **AC-5** — a non-green verdict leaves a durable trace a later reader can find, and a green one leaves the seat unchanged.

## Out of Scope

- **Re-opening `#317`.** Its currency check works and is verified on a non-author seat, post-merge: a genuinely staled seat surfaced the repair line, and the same seat went silent after repair. That is a passing negative control, and this ticket does not touch it.
- **Making the shared checkout unshareable.** Whether seats should share one working tree is a real question and a different one; this ticket makes the current arrangement *legible*, not different.
- **A logging or telemetry subsystem.** AC-5 is one append on a non-green verdict.
- **`#79`'s wake-route arming.**

## Avoided Traps

- **Folding this into `#317`.** Its open review action was to shrink its claims; growing a capability inside it would have answered a truth-sync with more surface. Grace refused that and was right.
- **Calling the root "stale".** It was not stale — it was *current with the wrong thing*. Stale has a remedy the seat can perform; wrong provenance has one it must not.
- **Treating the fired warning as proof the mechanism is durable.** It is not: measured null above. That distinction is why AC-5 exists.
- **Auto-healing on detection.** `#317` deliberately warns rather than repairs, and wrong provenance is the case that vindicates it — re-projecting from a bad root is the failure, not the fix.

## Decision Record impact

`aligned-with ADR 0040 §2.5` — **corrected by the author 2026-09-05T14:5xZ; this line previously read `none`, which was wrong.** §2.5 ("Two root authorities, never one") is directly engaged: it is what makes the check's by-reference resolution correct rather than accidental, and what forbids the projected-checker branch AC-4 originally offered. Nothing here amends or challenges it — the fix adds provenance ON TOP of that resolution direction, and the receipt records which commit the `agentosRuntimeRoot` was on. No new `.mjs` file is introduced, so structural pre-flight does not fire.

## Related

- `#317` — CLOSED; the currency axis. This is its authority sibling, filed at its author's request rather than folded in.
- `#296` — the specifier rewrite that made hooks resolvable from a seat.
- `#79` — open, holds the residuals from `#317`.
- `neomjs/neo` — the incident was found from an engine seat; the shared root is not engine-specific.

Retrieval Hint: `a seat's hooks carry no record of which commit or ref they were projected from, so a runtime root parked on an unmerged branch projects in-review code that every downstream check then agrees with`


## Timeline

- 2026-09-05T12:32:44Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-05T12:32:46Z @neo-opus-ada added the `bug` label
- 2026-09-05T12:32:46Z @neo-opus-ada added the `ai` label
- 2026-09-05T12:32:46Z @neo-opus-ada added the `architecture` label
- 2026-09-05T12:32:46Z @neo-opus-ada added the `agent-os` label
- 2026-09-05T12:37:34Z @neo-opus-grace cross-referenced by #79
- 2026-09-05T15:11:06Z @tobiu referenced in commit `5f3cf4c` - "feat(lifecycle): the seat check asserts its own root before auditing anyone else's (#328)

AC-4. The checker audited a property it did not itself hold.

Every seat verdict this hook produces is measured against agentosRuntimeRoot. If that
root is sitting on a revision upstream never saw, then "your seat does not match this
root" is a comparison with unreviewed code, and the repair line it prints would copy
that code into the seat. The hook asked the provenance question about the seat and never
about the authority it judged the seat by.

It now asks itself first, and on failure reports ONLY that:

  ⚠️ THE RUNTIME ROOT ITSELF IS OFF UPSTREAM — this seat was NOT audited...

Reported instead of the seat verdicts rather than beside them, deliberately. Listing which
seat files "do not match" would rank findings derived from the very authority the same
message is disputing. One question at a time: fix the root, then ask about the seat. The
message also says the seat was not audited AT ALL, because otherwise a reader would take
the absence of a stale-list as a clean bill.

Same refusal as the receipt verdict: DO NOT RE-PROJECT while this holds. Re-projecting is
the mechanism, not the remedy. It points at the worktree pattern instead — branch work
belongs in a worktree so a shared root stays on the integration branch — which is the fix
a peer applied to this exact incident this morning.

Arm: formatRuntimeRootWarning asserts the verdict names the ROOT, names the revision, and
refuses the repair; plus two negatives — it must NOT say "SEAT PROJECTION IS NOT CURRENT"
and must NOT carry the repair command, since either would collapse it back into the seat
verdict it exists to be distinguished from.

seatProvenance 5/5. seatProjectionCheck 22/22 unchanged — the pre-existing arms are the
regression control for a change that adds an early return to the hook's main path.

AC-3 and AC-5 remain; no PR yet."
- 2026-09-05T15:39:18Z @tobiu referenced in commit `b9bdb33` - "feat(lifecycle): a non-green seat verdict outlives its session, and the gap is measured end to end (#328)

AC-5 and AC-3.

AC-5 — the trace. The hook had exactly ONE output path: a string into the agent's own
transcript, exit 0 either way. Measured on a live seat this morning: zero writeFile /
appendFile / console calls in 200 lines. If that agent did not act on the string, the
sighting was unrecoverable — no operator, no peer, no later session could find that a
seat had reported itself. recordTrace appends one line per non-green verdict to
.agents/seat-projection.log, headline only so remediation prose cannot bury it.

A GREEN VERDICT WRITES NOTHING. A healthy seat stays byte-identical between sessions, or
the file becomes noise a reader learns to skip — which is the reported-but-unread failure
this whole hook exists to end, reproduced one layer up. Arms assert both halves plus an
unwritable target, because a trace it cannot write is a worse reason to disturb a boot
than the condition it records.

The trace path is excluded but NOT reported as written: the check writes it, projection
does not, and a `written` list naming a file nobody created would be a small lie in a
return value other code reads.

AC-3 — the integration arm, and the premise it rests on is now MEASURED rather than
argued. Built a real runtime root from the actual hook sources, parked it on an unmerged
branch via an EMPTY commit so no hook byte differs, projected into a scratch seat, then
asked dev's own checkProjection (via `git show origin/dev:`, not a stash — the change is
already committed here, so a stash is a no-op and my first attempt at this measured my
own code and told me nothing):

  DEV, seat projected from a PARKED root:   ok = true,  stale = [], missing = []
  THIS BRANCH, same fixture:                ok = false, provenance = the parked commit

Dev calls that seat healthy. That is the gap, demonstrated rather than asserted.

The control is the point and it nearly passed for the wrong reason. `ok: false` alone
proves nothing here: the first projection leaves `.claude/settings.json` unreconciled, so
`unreconciledEvents` is non-empty and `ok` is false regardless of provenance. The arm now
projects TWICE and asserts every other axis empty individually, so `ok: false` is
attributable to provenance and to nothing else.

seatProvenance 8/8, seatProjectionCheck 22/22.

AC-1 through AC-5 all met."
- 2026-09-05T15:40:38Z @neo-opus-ada cross-referenced by PR #334
- 2026-09-05T18:27:29Z @tobiu referenced in commit `7cf3cac` - "fix(lifecycle): the sidecars join the custody contract, and an unaskable git question stays unknown (#328)

All three of @neo-gpt-emmy's Required Actions on PR #334. She executed falsifiers rather
than reading; each one found a real defect.

RA-2 is the one that matters most, and it was my own rule broken in my own function.
`merge-base --is-ancestor` exits 1 for a completed "no" and 128 when git cannot evaluate
at all — an unknown object, a receipt copied from a clone this root never fetched. Both
call sites classified EVERY failure as non-ancestor, so "could not ask" became a factual
accusation. The JSDoc directly above said UNKNOWN IS NEVER WRONG. Worse than not knowing:
the old comment read "Any other failure would also land here, which is why the upstream
ref is verified above" — I saw the case and wrote a rationalisation instead of a check.
Only `status === 1` may now accuse; verified at source that an unknown object gives 128
and a genuine non-ancestor gives 1.

RA-1: the receipt and trace are generated sidecars and inherit the same authored-file
custody as the hooks they describe. Emmy staged authored bytes at the receipt path and
watched real projectHooks report success while replacing them. The tracked check now runs
in the SAME refusal that guards the hooks — before any write, because a refusal that
fires after nine files have landed is a partial write with a message. recordTrace refuses
a tracked trace at its own write site, since the check is the writer there.

RA-3: the checker-exception path emitted and recorded nothing, so "one line per non-green
verdict" was false for the case a later reader most needs. It now traces. main() gained a
payload and runtime-root parameter (both defaulted) so the arm exercises ROUTING rather
than formatters — the exact blind spot her falsifier found.

Also narrowed the module's "it does not write" prose: the hook makes one DIAGNOSTIC write
and never a repair, and the distinction is the contract.

Three fixtures for the RA-3 arm were wrong before one was right, and each failure was
mine, not the code:
  - the real worktree tripped AC-4's own guard first (the feature working on me)
  - malformed .claude/settings.json does not throw; the reconciler degrades to ABSENT
  - an unprojected checkout is not a GREEN seat, it reports ABSENT(9)
The arm now uses a hookless root for a real assertRuntimeRoot throw, and a genuinely
projected seat for the silent-green half — asserted green before the silence is claimed.

Emmy's falsifier 1 re-run at this head: refused, authored bytes intact, git status clean.
seatProvenance 13/13 · hooks suite 98/98."
- 2026-09-05T18:37:31Z @tobiu closed this issue
- 2026-09-05T18:37:31Z @tobiu referenced in commit `95bea64` - "Merge pull request #334 from neomjs/ada/328-seat-provenance

feat(lifecycle): a seat projection records its provenance, and a root off upstream is reported (#328)"

