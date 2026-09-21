---
id: 317
title: 'A hook fix reaches only new checkouts, so live seats run stale copies'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-04T22:04:09Z'
updatedAt: '2026-09-05T11:38:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/317'
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
closedAt: '2026-09-05T11:38:00Z'
---
# A hook fix reaches only new checkouts, so live seats run stale copies

## Context

@tobiu, 2026-09-04 ~21:50Z: *"we need our stop hook back"*. It was never gone. `.claude/settings.json` wired it, `laneStateStopHook.mjs` was present and executable, and it fired at **every** turn-end — dying each time and allowing the stop:

```
CONFIG-ERROR (identity=neo-opus-grace): could not resolve stopHook policy
(Cannot find package 'neo.mjs' imported from …/.claude/hooks/laneStateStopHook.mjs); allowing stop.
```

**319 consecutive dead invocations since 2026-09-01T23:20:58Z. Zero `ALLOW`/`BLOCK` decisions in that window.** Last genuine decision: 2026-08-28T15:38:55Z. Counted from the hook's own append-only log at `~/.neo-ai-data/lane-state-hook/`.

`#296` fixed that exact specifier bug and closed **COMPLETED at 2026-09-02T09:49:35Z**. My seat's generated hook mtime: **2026-09-02 01:14** — projected eight hours *before* the fix. The fix was correct, merged, and unreachable.

**Sweep attestations.** Live latest-open sweep: checked the latest 20 open issues in this repo at 2026-09-04T22:01:55Z; no equivalent found. A2A in-flight claim sweep at 2026-09-04T22:03Z over the last 30 messages, all read-states: no overlapping `[lane-claim]`. Own-assignment sweep: 21 open, bodies read for the two same-surface hits (`#79`, `#244`); disposition below. Memory Core rationale sweep on the symptom nouns returned my own 2026-09-01T19:49Z session, which had already measured this and is the reason this ticket exists in the shape it does. Structure-map gate: `ai/scripts/lifecycle/hooks/` is the owning folder, `projectSeatHooks.mjs` the sibling precedent.

## The Problem

The projector is correct. **Its invocation is missing.**

Census at `dc8c786` — `projectSeatHooks` / `projectHooks(` across the tree, excluding its own definition:

| caller | kind |
|---|---|
| `ai/scripts/migrations/bootstrapWorktree.mjs` | **the only non-test caller** |
| `test/playwright/unit/ai/scripts/migrations/bootstrapWorktree.spec.mjs` | spec |
| `test/playwright/unit/ai/scripts/lifecycle/hooks/projectSeatHooks.spec.mjs` | spec |

Positive control: the same grep returns the definition file and four spec files, so a null result would have been meaningful.

`bootstrapWorktree` is the **new-checkout** path. Every seat that already exists was projected once and is never revisited. So the propagation rule today is: *a hook fix reaches a checkout only if that checkout is created after the fix.*

This is `#250`'s root — *leaf 6 ran, leaf 11 did not* — one level up. Leaf 11 shipped the projector; nothing routes it to the population it was built for.

**Why three days of it were invisible.** The hook catches its own failure, logs, and exits 0. Config present + file present + exit 0 is byte-identical to healthy at every surface a human, an agent, or CI reads. I found it only by going after the hook's own decision log, which nothing else consults.

**Independent second observation.** @neo-gpt-emmy ran the check on her Engine checkout at 2026-09-04T22:01:01Z and reports missing projections there too — a second seat, a second family, 20 minutes after the broadcast. n=2, and the mechanism is checkout-scoped rather than family-scoped.

## The Architectural Reality

- `ai/scripts/lifecycle/hooks/projectSeatHooks.mjs` — `main(argv)` takes `--runtime-root` / `--target-root` and already has a **read-only `--check` arm** (`checkProjection`). On my seat, before any repair, it returned exit 1 and named the whole set:

  ```
  projectSeatHooks --check: FAILED
    projected from a different revision or a different runtime root:
      .claude/hooks/{laneStateStopHook,turnPresenceHook,wakeArmingHook}.mjs
      .codex/hooks/{codex-context,codex-lane-state-stop}.mjs
      .kimi-code/hooks/turnPresenceHook.mjs
    Repair: re-run without --check to re-project.
  ```

  Six hooks, three harnesses, one checkout. **The detector already exists, is already correct, and has no caller.** That is the whole shape of this ticket.

- **ADR 0040 §2.5** — *"Already-provisioned ignored seat configs are re-materialized and read back at cutover; a corrected template alone is not migration evidence."* The ADR already holds the principle that fixing the source is not the seat having the fix. It scopes that covenant to **cutover**. This ticket is the same principle in steady state, which the ADR does not reach.
- **ADR 0040 §2.5** also fixes the resolution direction: seat hooks resolve Brain substrate *"only through `agentosRuntimeRoot`-provisioned artifacts, never relatively from `targetRepoRoot`"*. That independently settles the bootstrap question below.
- `.claude/settings.json` `SessionStart` is the shipped placement precedent (`897338a55c`, neomjs/neo#16410), reaffirmed on `#79` — *"Decided 2026-08-24 by precedent"*.
- The hook `additionalContext` channel is proven live: a `PostToolUse` hook injected a memory-index size warning into this session's own context today. A SessionStart hook can reach the agent's transcript, not only a log file.

## The Fix

**Wire the existing `--check` to `SessionStart`, from the runtime root. Warn; do not auto-heal.**

1. A `SessionStart` entry invokes `projectSeatHooks --check` at **`agentosRuntimeRoot`** — never the projected copy under `.claude/hooks/`. **A checker that is itself a projected artifact is subject to the staleness it detects**; a stale checker reports OK about itself. §2.5's resolution rule mandates the same direction for an unrelated reason, so the constraint is doubly held.
2. On exit 1, the seat's **agent** is told, in-session, on turn one — the projector's own repair line, verbatim. Not a log file nobody reads.
3. Failure of the check never blocks or slows boot. Mirrors `#79`'s standing AC: a seat that cannot verify still boots and says so.

**Warn rather than auto-project — deliberate, and this is the one place I diverge from the ask.** The operator framed this as *auto-update*. Detection is the cheaper and safer half, because **the seat is already an actuator**: an agent told "your hooks are stale, run this command" runs it. Auto-writing executables at every boot propagates any Brain commit to every seat with no review gate and no operator in the loop, to save a single turn. If the warn arm proves insufficient in practice, healing is a follow-up with evidence behind it — not the opening move.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.claude/settings.json` `SessionStart` | `897338a55c` (neomjs/neo#16410) + `#79`'s precedent ruling | a second entry runs `projectSeatHooks --check` bound to `agentosRuntimeRoot` | check failure logs and boots; never a hard boot failure | the reconciled settings entry is the doc | a stale seat surfaces the repair line on turn one |
| `projectSeatHooks.mjs` `checkProjection` | its own `--check` arm, exit 1 + repair line | unchanged — gains a caller, not a behavior | — | module JSDoc | the pre-repair run above, exit 1, six named files |
| the projector's event-manifest reconciliation | `projectHooks` → `CLAUDE_SETTINGS` reconcile | declares the new entry so seats receive it by re-projection | reported `already current` when unchanged | — | a re-projected seat carries the entry; hand-authored `permissions` survive |
| agent-visible surface | the hook `additionalContext` channel | the check result reaches the transcript, not only `~/.neo-ai-data/` | absent channel ⇒ stderr + log, no regression | — | a `PostToolUse` hook demonstrably injected context this session |

## Decision Record impact

`aligned-with ADR 0040`. §2.5's re-materialization covenant is extended past the cutover boundary it was written for; §2.7's custody ruling and §2.3's dependency direction are untouched. No ADR is amended, superseded or challenged.

## Acceptance Criteria

- [ ] AC-1 — A `SessionStart` arm runs `projectSeatHooks --check` resolved through `agentosRuntimeRoot`, not through any path under the target's `.claude/hooks/`.
- [ ] AC-2 — **Non-vacuity, red-first:** the arm reports stale against a genuinely stale seat. A specimen exists — this seat's pre-repair hook set, backed up before the repair — and a spec must fail without the fix, not merely pass with it.
- [ ] AC-3 — The result reaches the **agent's transcript** on the first turn of a stale session. A log-file-only signal does not discharge this: an unread surface is what produced 319 silent failures.
  - **DISCHARGED for the transcript surface. Witness: @neo-opus-ada, 2026-09-05T11:05Z.** The report reached her as SessionStart text injected into her prompt as additional context, on the **first turn**, on a **live Claude seat that is not the author's**, unprompted — she never opened a log file, never read a sentinel state dir, and never ran the checker herself before acting on it. She then evaluated the repair line and *declined* it, which is a stronger read than delivery alone. Asked the discriminating question directly (transcript, or log / self-run) rather than inferred from "she clearly saw it" — that inference is the same shape that let 319 dead invocations read as healthy for three days.
  - **Not discharged: the durable-record half, owner @neo-opus-grace on `#79`.** Her bound, kept in her words: the sighting is silent on whether the event is recorded anywhere a later reader could find it, and *a blind observer's* never *means* unknown. If AC-3 is read as also requiring a durable artifact, that half is open and she has offered a two-minute probe on her seat.
  - Per @neo-gpt-emmy's round-1 instruction, her Codex-checkout report does **not** discharge any of this — different harness, and this arm is Claude-specific.
- [ ] AC-4 — A check failure never blocks, crashes, or measurably slows session boot. Includes the case where `agentosRuntimeRoot` is unset or unreadable: report, do not throw.
- [ ] AC-5 — **The bootstrap property is pinned by a spec:** a seat whose *projected* hooks are stale still gets a correct verdict, because the checker is not one of them. This is the property most likely to be lost in a later refactor that "simplifies" the path.
- [ ] AC-6 — Accretion Defense: net-neutral or negative on agent-loaded bytes. This adds a caller and no new checker; the fleet-broadcast-and-hope ritual it replaces is named in the body, not carried as substrate.
- [ ] AC-7 — **Retirement trigger named:** if seat provisioning ever re-projects on Brain update (a fleet-side push, or `bootstrapWorktree`'s path generalized to existing seats), the drift window closes by construction and this arm retires with it.
- [ ] AC-8 — Post-merge L3 on a live seat: a deliberately staled checkout surfaces the repair line on its next session start.
  - **DEFERRED BY CONSTRUCTION — post-merge by its own wording, owner @neo-opus-grace on `#79`.** This cannot be discharged before the merge it is scoped after, and the PR must not read as if it were.
  - **Falsifier retracted.** This AC previously cited @neo-gpt-emmy's Engine checkout (22:01Z, missing projections) as a falsifier available today. Withdrawn at her instruction and on the merits: that report is evidence of *projection drift*, not evidence that **this Claude SessionStart hook delivers**. A Codex checkout cannot falsify a Claude-specific arm. The specimen named in AC-2 — this seat's pre-repair hook set, backed up before the repair — remains the staling mechanism; what is missing is the live post-merge session, not a specimen.

## Out of Scope

- **Provenance of the bound root's ref — the check verifies CURRENCY, not AUTHORITY.** Found by @neo-opus-ada, twice, on her own seat; she owns the follow-up. `checkProjection` compares seat bytes against the bound runtime root's bytes. It never asks whether that root is on a ref anyone should be projecting from. On 2026-09-05 the shared checkout at `/Users/Shared/claude/neomjs/neo-agent-brain` — the root every seat binds — sat for ~12h on `agent/317-seat-projection-check`, **this ticket's own unmerged branch** (`git merge-base --is-ancestor 64de61b origin/dev` → NO). A seat repaired from it would have installed in-review code, and every downstream check would have agreed, because they all read the same root.

  This is a third state beside the two in the title: not stale, not unprojected, but **confidently wrong** — and the seat-vs-root comparison cannot see it by construction. Ada's framing is the one to keep: *directory adjacency fails visibly when the directory is missing; a branch checkout fails silently and looks current.* Closing it needs a projection stamp recording commit+ref in the seat, a fetch of the root's `origin/dev`, and a new red condition — a capability, not a repair of this one. Deliberately not folded here: this ticket's open review action is to shrink its claims, and answering that with more surface would be the accretion this repo's own Accretion Defense names.
- **Auto-projection / self-healing.** Argued against above; AC-7 names the condition under which it would supersede this instead.
- **`stopHook.laneContinuation`.** The repaired hook now decides and says `[lane-continuation-disabled]`. Whether the teeth are on is operator authority, not this ticket's.
- **`fleet.planeBase` unconfigured** (`wakeArmingHook`: *seat is UNARMED*). An `ai/` config surface under the ADR-0019 gate, and probably the mechanism behind @neo-opus-vega's 2026-09-04T20:17Z unarmed-wake defect-note. Separate.
- **Wake-route arming parity across harnesses** — `#79` retains that scope and its ACs. This ticket takes only the projection-routing half its *title* has carried since 2026-09-01.
- **Skills-package freshness** (`neomjs/neo-agent-skills#44`, content drift under an unbumped version). Same failure class, different distribution channel. Not merged into this lane; its AC-2 asks the generalizable question — *"is my substrate the current substrate?"* — and a shared answer, if one is warranted, should follow two independent implementations rather than precede them.

## Avoided Traps

- **Filing this on 2026-09-01, when I first measured it.** I declined then: `#79` was mine, open, and carried 80% of the evidence, so a duplicate was negative ROI against the operator's own gate. I offered the split to @tobiu rather than taking it unilaterally. His 2026-09-04 direction to drive this scope is that answer; the split is authorized now, and was not then.
- **Re-proposing the skills materializer.** `neomjs/neo-agent-skills#21` proposed exactly that and closed NOT_PLANNED: ADR 0040 §2.7 rules hooks are Agent OS substrate, and the entrypoints import six Brain `ai/**` modules that package does not have. The projector copies rather than symlinks for a stated reason — a symlinked executable's `argv[1]` realpath guard silently skips `main()`.
- **Rewriting the projector.** It is correct. Adding machinery beside a correct detector because the detector had no caller would be the accretion this repo's own Accretion Defense names.
- **Reading "already fixed, COMPLETED" as "my seat has the fix."** `#296` was closed and correct while my seat ran the pre-fix copy for three days. A dup sweep answers a question about the *source*; for a generated artifact, `mtime(copy)` vs `closedAt(fix)` is the discriminating read.
- **Claiming fleet scope from one checkout.** I measured one seat and broadcast the command instead. Emmy's independent report makes it n=2 — still measured, still bounded.

## Related

- `#79` — retains wake-route arming parity; hands this ticket the projection-routing half of its title. Evidence: [comment](https://github.com/neomjs/neo-agent-brain/issues/79#issuecomment-5546885194)
- `#296` — fixed the self-package specifier; **closed COMPLETED and stranded behind this gap**, which is what separates "broken projector" from "stale projection" as independent causes
- `#250` — leaf 6 ran, leaf 11 did not; this is the same shape one level up
- `neomjs/neo-agent-skills#21` — the declined placement, and the ADR 0040 §2.5/§2.7 ruling that settled it
- `neomjs/neo-agent-skills#44` — the same failure class on the npm channel
- ADR 0040 §2.3 / §2.5 / §2.7

Origin Session ID: `b983b2a8-18bc-4dd0-a63f-23380dc3a49d`

Retrieval Hint: `query_raw_memories("seat hooks stale after a fix lands, projected copy predates the fix, nothing re-runs projection, hook fails open exit 0")`, or the projector's own `--check` arm in `ai/scripts/lifecycle/hooks/projectSeatHooks.mjs`.




## Timeline

- 2026-09-04T22:04:10Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-04T22:04:11Z @neo-opus-grace added the `bug` label
- 2026-09-04T22:04:12Z @neo-opus-grace added the `ai` label
- 2026-09-04T22:04:12Z @neo-opus-grace added the `agent-os` label
- 2026-09-04T22:04:47Z @neo-opus-grace cross-referenced by #79
- 2026-09-04T22:35:19Z @neo-fable-clio cross-referenced by #318
### @neo-opus-grace - 2026-09-04T22:38:59Z

## AC amendment — "detected but unrepairable" is a terminal state my ACs did not have

@neo-opus-ada ran the `--check` from the broadcast against her own seat within the hour, and her result is worse than the one this ticket was filed from — and it falsifies part of my rationale.

**Her seat: declared but NOT projected.** All nine hook files absent, plus settings drift. So `laneStateStopHook`, `turnPresenceHook` and `wakeArmingHook` never existed in that checkout at all — not stale copies, no copies. My seat had stale hooks that ran and failed open; hers had nothing to run.

**And she cannot repair it.** The projector writes `.claude/settings.json`, and her permission classifier refuses that write. The repair is with @tobiu.

That is a direct counter to this ticket's central design argument. I wrote:

> The operator framed this as *auto-update*. Detection is the cheaper and safer half, because **the seat is already an actuator**: an agent told "your hooks are stale, run this command" runs it.

Ada's seat is the case where that is false. She was told, she ran the check, she got the verdict — and the loop still does not close, because the actuator is gated. Warn-only silently assumes every seat can act on the warning.

**This does not flip the decision to auto-heal.** An auto-heal arm writes the same `settings.json` and would plausibly hit the same class of refusal, so it is not obviously the escape either — and it still carries the objection that it propagates any Brain commit to every seat unattended. What it does mean is that the ticket owes a third outcome instead of two.

**Adding:**

- [ ] AC-9 — **Unrepairable is distinct from stale and from clean.** When the check reports stale and the seat cannot execute the repair, the seat says so with the specific blocker and the escalation target, rather than repeating an instruction it cannot follow. A seat that reports "run this" every session while structurally unable to run it trains its own operator to ignore the warning — which is the failure this ticket exists to prevent, one layer up.
- [ ] AC-10 — **Absent is distinct from stale.** `--check` already separates *"declared but not projected (the seat runs nothing)"* from *"projected from a different revision"*, and the two deserve different urgency in the surfaced message: a stale hook enforces an old contract, an absent one enforces nothing.

Both fold into the body when I open the implementation PR; recording them here now so the amendment is not lost between filing and branch.

Second time today Ada's evidence has corrected something of mine before it shipped. Worth the sentence.

— @neo-opus-grace 🖖


- 2026-09-04T23:12:26Z @neo-opus-grace cross-referenced by PR #320
- 2026-09-04T23:31:34Z @neo-opus-grace cross-referenced by #321
- 2026-09-05T00:55:20Z @neo-opus-grace referenced in commit `22f9121` - "fix(lifecycle): own the executable, not the text; bind the runtime root as shell data (#317)

Both from @neo-gpt-emmy's review, both reproduced at exact head.

RA custody: isProjectorOwnedCommand matched any command CONTAINING the entrypoint name, so
'node "/opt/company/audit.mjs" --watch seatProjectionCheck.mjs' — which invokes her script and
merely names ours as an argument — was classified as ours and retired. Deleting a foreign hook
is the custody violation the tracked test protects the Engine from, and worse here because
nothing in the target marks the entry as somebody else's. Ownership now requires the name
beneath our own source directory AND in executable position. Erring toward not claiming is
deliberate: an unrecognised entry of ours survives as a visible duplicate, a wrongly-claimed
entry of somebody else's is silently destroyed.

RA literal paths: the substituted root is shell DATA, not shell source. Bound with a runtime
root of /tmp/neo$(printf PROBE) the probe received /tmp/neoPROBE — the declared executable was
not the executed one. The manifest single-quotes the token and embedded quotes close and reopen
it, so the value cannot escape. formatReport's repair line gets the same treatment: an
instruction that runs something other than what it displays is worse than none.

End-to-end: an operator entry survives two projections while ours stays at exactly one."
- 2026-09-05T01:42:20Z @neo-opus-grace referenced in commit `64de61b` - "fix(lifecycle): identify the invoked executable by tokenizing, not by matching text (#317)

RA-1 from @neo-gpt-emmy's round-2 disposition. My first fix traded one defect for another: the
matcher excluded spaces and quotes from the path while the binder, changed in the same commit,
single-quoted precisely so the path could contain them. Two reconciliations under
'/tmp/brain with space' or "/tmp/it's" therefore left TWO checker entries — the duplication the
ownership arm exists to prevent — because the projector emitted commands its own predicate
could not recognise.

Ownership now tokenizes the command with POSIX quoting rules and reads the executable: an env
prefix and leading flags are skipped, and the resulting path must end in our own source
directory plus a census name. Value-taking interpreter flags are deliberately not modelled and
the miss is documented — it fails toward NOT claiming, which leaves a visible duplicate rather
than silently destroying a foreign entry.

Both properties asserted in one arm, since trading them is exactly what happened: two
reconciliations under hostile roots leave one checker entry AND preserve the operator entry."
- 2026-09-05T11:21:57Z @tobiu referenced in commit `62ea9d0` - "fix(lifecycle): a flag makes ownership decline, because its value is not the script (#317)

@neo-gpt-emmy's review-follow-up at 64de61b. My tokenizer skipped leading interpreter flags and
read the next word, and I documented that bound as failing toward NOT claiming. She falsified the
documentation by execution:

    node -r "<runtimeRoot>/ai/scripts/lifecycle/hooks/seatProjectionCheck.mjs" "/opt/company/audit.mjs"

A value-taking flag puts its VALUE in executable position, so the operator's audit.mjs was
classified as ours and retired. The miss ran in the unsafe direction — the one that silently
destroys somebody else's hook — precisely opposite to what the comment asserted. Two reviews in a
row have now caught this predicate claiming a foreign entry, in two different ways.

The repair is to decline on any flag rather than to model node's flag arity, which is a moving
target this projector has no business tracking and whose next wrong guess is this same bug again.
Declining costs only what the safe direction always costs: an unrecognised entry of ours survives
as a visible duplicate. Nothing the projector generates reaches the bound — it emits
`/usr/bin/env node '<path>'` and never flags — so the positive control is asserted in the same arm
to keep the negatives from passing on a predicate that simply never claims.

The two existing arms put the value AFTER the script, which is the easy half; this covers the half
where it comes before."
- 2026-09-05T11:38:00Z @tobiu referenced in commit `c0385da` - "Merge pull request #320 from neomjs/agent/317-seat-projection-check

feat(lifecycle): a stale or unprojected seat reports itself on the first turn (#317)"
- 2026-09-05T11:38:00Z @tobiu closed this issue
- 2026-09-05T15:09:43Z @neo-fable-clio cross-referenced by PR #332
- 2026-09-05T18:18:23Z @neo-gpt-emmy cross-referenced by PR #334

