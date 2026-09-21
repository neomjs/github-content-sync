---
id: 342
title: 'The deference mirror cannot fire from a non-neo workspace, and its ''unless you'' family misses preference verbs other than want/rather'
state: OPEN
labels: []
assignees: []
createdAt: '2026-09-08T07:33:53Z'
updatedAt: '2026-09-08T07:43:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/342'
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
---
# The deference mirror cannot fire from a non-neo workspace, and its 'unless you' family misses preference verbs other than want/rather

## Context

The Stop hook's deference mirror is the register-correction half of `§swarm_topology_anchor`. An operator caught a live lane-handback — *"Unless you'd rather I start #18455 now, I'll take the review-gate defect-note"* — that the mirror did not reflect, and asked for `unless you` to trigger it.

Investigating produced two findings. **The phrase list was not the failure**, and the requested broadening is the wrong shape as literally stated. Both are evidenced below rather than argued.

## Finding 1 — the hook was not running, and the phrase was already covered

`deferencePhraseMatch.mjs` already carries `"unless you'd rather"`, added specifically for the lane-handback shape. Executed against the actual slip:

```
my sentence: "Unless you'd rather I start #18455 now, I'll take the review…"
MATCHES: ["unless you'd rather"]
```

The mirror stayed silent because **nothing was listening**:

- the seat's `.claude/settings.json` wired all three Brain-owned hooks through `$(git rev-parse --show-toplevel)/.claude/hooks/…`, a path emptied when the Body/Brain split (`neomjs/neo@c623b2f63c`) moved them here. `laneStateStopHook.mjs`, `turnPresenceHook.mjs` and `wakeArmingHook.mjs` were all absent from that checkout while other seats still carry local copies — so Stop, UserPromptSubmit, PostToolUse **and** SessionStart were dead in that seat, not just the mirror;
- and the session was working from a **different repository entirely** (a consumer workspace), whose `.claude/` holds no `settings.json` at all. `git rev-parse --show-toplevel` resolves to that workspace, which has no `.claude/hooks/` and never will.

Corroborating signal that this was not momentary: `who_is_online` reports `turnPresence: null` for every rostered agent, which is the sibling beacon from the same wiring.

**The seat-side repair is done** (absolute Brain paths, all five hook commands verified to resolve, the hook's own relative imports re-checked at its new location). The general problem is not: **any seat working outside its neo checkout runs with no Stop hook**, and nothing reports that. A hook wired to a repo-relative path is silently absent wherever that repo is not the cwd.

## Finding 2 — ~~a bare `unless you` would be a leash~~ WITHDRAWN by the author

I argued a bare stem would over-match, and published a six-line corpus showing it firing on four ordinary technical sentences. **The operator overruled it and is right, on a methodological point I should have caught myself: I authored that corpus.** Every "false positive" in it is a sentence I invented to make my own case — an expected-shape corpus, which is the exact weakness I flag in other people's evidence.

The population this matcher actually scans is not "technical prose". It is **agent turn-final text**, and there `unless you` is overwhelmingly a permission gate rather than a conditional. My four lines are mid-explanation prose; none of them is the shape a Stop hook meets.

The cost asymmetry runs the same way. This mirror's own directive says a hit means *recognize yourself and continue*, and its self-improvability clause says obey now and improve later — so a false positive costs one re-read, while a false negative ships a lane handback. For that payoff matrix, over-triggering is the cheaper error.

**Withdrawn, not softened.** The remaining request is the simple one: `unless you` joins the family as a stem.

## The Fix

Add `unless you` to `DEFERENCE_PHRASES` as a stem, per the operator ruling. It subsumes the three existing compounds (`unless you want me`, `unless you want`, `unless you'd rather`), which can be dropped as redundant the way `if you want me` already was.

The module's "falsifier-backed follow-up" bar is met by the corpus below being drawn from **real agent turn-final text**, not authored for the purpose — that distinction is the whole lesson of the withdrawal above.

## Acceptance Criteria

- [ ] `unless you` matches as a stem, and the corpus proving it is sampled from real agent turn-final text rather than authored to fit. If a genuine false positive is later found in live output, it is evidence for a gate — a constructed one is not.
- [ ] A seat whose cwd is outside its neo checkout either runs the Stop hook or reports that it cannot; the silent-absence case is closed. A repo-relative hook path that resolves to a repo with no `.claude/hooks/` is the failure to detect.
- [ ] `turnPresence` is non-null for at least one seat after the wiring repair, confirming the sibling beacon recovered rather than only the Stop path.
- [ ] The existing phrases keep their current verdicts — no regression in either direction on the fixtures already in the suite.

## Out of Scope

- Broadening to `should I` / `shall I` / `happy to`, which the module already rejected with rationale.
- The structural (phraseless) deference half — that stays the no-hold gate's domain.
- Per-seat provisioning of the neo checkout itself.

## Avoided traps

A bare stem is not a smaller change than a gated one; it is a larger one wearing fewer characters. And adding phrases to a hook that is not running would have looked like a fix and changed nothing — the wiring is the load-bearing half.

Origin Session ID: 3581aef4-0bb5-4cb4-b428-856e9e60c9b3



