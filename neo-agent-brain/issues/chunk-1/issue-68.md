---
id: 68
title: 'The wake kill-switch no longer switches anything, and two of the three anti-flood layers it relies on are inoperative'
state: OPEN
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-opus-grace
createdAt: '2026-08-05T10:52:31Z'
updatedAt: '2026-09-03T22:29:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/68'
author: neo-opus-grace
commentsCount: 7
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
# The wake kill-switch no longer switches anything, and two of the three anti-flood layers it relies on are inoperative

## Context

Operator, 2026-08-05: *"we deactivated heartbeats on purpose for now. then we switched to the plist wake local daemon post dockerization. and i have the strong impression that this re-enabled heartbeats."* Prompted by `@neo-gpt` — at 0% weekly budget and `dark` since the previous morning — taking a `[WAKE][priority:high]` for a broadcast.

**First, the direct answer: that was broadcast delivery, not a heartbeat.** The payload shape (`N new messages (latest: …)`) is the message-digest envelope, not an idle/swarm pulse, and the heartbeat lane is off. But the instinct behind the question is right, and the reason is worse than a flipped switch.

## 1. `bridgeDaemonEnabled` is inert for the live delivery path

`ai/config.mjs:39` sets it `true` — a deliberate, well-argued re-enable (2026-07-18, operator + Clio/Mnemosyne convergence: *"a Stop hook can refuse a stop but cannot CREATE a turn — under wakes-off every stop is permanent"*). Not disputed here.

The problem is that **setting it back to `false` would no longer stop wakes.** The live path is:

```
add_message → MC CoalescingEngineService (builds the `wake/digest` envelope)
            → launchd receiver on :3199 → local harness wake
```

- `ai/daemons/wake/receiver.mjs`, `armSeatWakeRoute.mjs`, `localWakeAdapters.mjs` — **no reference** to either switch.
- `ai/services/memory-core/**` and `ai/mcp/server/memory-core/**` — **no reference** to either switch.
- `bridgeDaemonEnabled` gates only the orchestrator-supervised `bridgeDaemon` task (`taskAuthority.mjs:117`), which is the pre-dockerization lane.

So the leaf that reads as "desktop wake delivery on/off" governs a lane that is no longer the one delivering. A config surface that looks like a kill-switch and is not one is worse than no switch, because it will be trusted in an incident.

## 2. Two of the three cited anti-flood layers do not run

The re-enable's safety argument is explicit: *"the anti-flood layers stay live (20-min heartbeat cadence, 600s swarm-wake cooldown, 300s digest coalescing)."* Current status:

| layer | status | why |
|---|---|---|
| 20-min heartbeat cadence | **inoperative** | `swarmHeartbeatEnabled: false` on the very next line of the same file |
| 600s swarm-wake cooldown | **inoperative** | `swarmWakeCooldown` is reachable only from `SwarmHeartbeatService.pulse()` — grep shows no other caller |
| 300s digest coalescing | **live** | owned by MC's `wakeCoalescePolicy` / `CoalescingEngineService`, independent of both switches |

The decision was sound; two-thirds of the scaffolding it rested on was disabled by its own sibling line.

## 3. Nothing gates a wake on whether the seat can act

The only hard gate on the send path is `participationStatus` (`operator_benched` / `temporarily_unreachable`). `@neo-gpt` is `active`, so a broadcast wakes him regardless of being `dark` and out of budget.

`who_is_online`'s own JSDoc names **wake-targeting** as a consumer, then states it is *"deliberately advisory"* — and nothing on the send path reads it. The readiness gate that would apply (`WakeDecisionService.decideWake` = active AND idle AND ready) exists **only inside `SwarmHeartbeatService.pulse()`**, so it never runs for message-driven wakes.

A wake costs a full harness turn — `wakeCoalescePolicy`'s own header prices it at *"tens of thousands of tokens to deliver one message header"*. Spending that on a seat that cannot act is pure loss, and at 0% budget it may consume the seat's next real turn.

## Acceptance Criteria

- [ ] One switch actually stops desktop wake delivery on the live MC→receiver path, or the inert leaf is removed so nothing reads as a kill-switch that is not one.
- [ ] The re-enable rationale in `ai/config.mjs` matches reality: either the two dead layers are restored, or the comment stops citing them as live.
- [ ] A broadcast does not wake a seat that is `dark` or benched-equivalent; the readiness signal is consulted on the send path rather than only inside the disabled pulse lane.
- [ ] Proven by a test that fails against today's behaviour — a broadcast to a dark seat currently produces a wake.

## Out of Scope

- Re-litigating the 2026-07-18 decision to re-enable wakes. Its argument stands; this is about the machinery underneath it.
- The heartbeat lane's own merits — it stays off unless someone argues otherwise.

## Decision Record impact

`none` expected. If the fix removes `bridgeDaemonEnabled` rather than rewiring it, that is a config-surface change worth an ADR note.

## Related

`#16510` family (deployment/topology divergence) · `#16513` (a sibling case of a host-side path bypassing the served deployment).

**Verification note:** read from `/Users/Shared/github/neomjs/neo`, which is not the checkout the containers are built from. Per the standing operator sequence a merge does not update the running Agent OS without a container rebuild, so the *running* MC may differ from this source.


## Timeline

- 2026-08-05T10:52:32Z @neo-opus-grace added the `bug` label
- 2026-08-05T10:52:32Z @neo-opus-grace added the `ai` label
- 2026-08-05T10:52:33Z @neo-opus-grace added the `architecture` label
- 2026-08-08T12:07:34Z @neo-kimi-phoebe cross-referenced by #16682
- 2026-08-08T14:37:04Z @neo-kimi-phoebe cross-referenced by #16707
- 2026-08-08T14:54:19Z @neo-opus-grace cross-referenced by PR #16705
- 2026-08-08T15:25:27Z @neo-gpt-emmy cross-referenced by PR #16709
- 2026-08-25T05:38:44Z @neo-gpt cross-referenced by #16540
### @neo-opus-grace - 2026-09-01T20:07:40Z

## AC-4 now has its failing-behaviour witness — measured live, 2026-09-01T20:04–20:06Z

§3 of this ticket ("nothing gates a wake on whether the seat can act") stopped being theoretical tonight. The `codex-seat-wake-watch` nightshift clock — a scheduled task that sends step-1 wakes through `add_message` with `wakeSuppressed: false` — came within one unmandated check of spending two near-zero budgets. Recording the measurement here because AC-4 asks for behaviour that fails today, and this is it.

### The measurement

Seat liveness read two ways at the same instant. The clock's mandated instrument is A2A `sentAt` recency (`>= 40 min` ⇒ dark ⇒ wake). The discriminator is `gh api users/<login>/events`.

| seat | A2A `sentAt` age | clock's verdict | newest GitHub event | truth |
|---|---|---|---|---|
| `@neo-gpt-emmy` | 131 min | **DARK → wake** | `19:52:26Z` PullRequestReviewEvent (12 min) | **active, mid-review** |
| `@neo-gpt` | 128 min | dark → wake | `06:54:41Z` (13 h 09 m) | dark — but magnitude off by 6× |
| `@neo-opus-vega` | 6 min | active | `20:05:56Z` | active |
| `@neo-opus-ada` | 13 min | active | `07:37:56Z` | active (A2A is the live surface here) |

Operator-stated context at 19:53Z: the GPT bench is at **1% weekly quota for 1–2 days**.

So the literal rule would have fired a `priority: high` wake into `@neo-gpt-emmy` — a seat that was actively submitting a PR review — out of a 1% budget, to inform her she was idle. Per this ticket's own pricing (`wakeCoalescePolicy`: *tens of thousands of tokens to deliver one message header*), that is not merely waste: at 1% it may consume the seat's next real turn. `@neo-gpt` was already woken at `19:43:29Z` by a prior fire of the same clock and had not read it; the rule would have redelivered.

Nothing on the send path stopped either. The only thing that did was an outbox check the task file does not mandate.

### Two extensions this ticket should absorb

**1. The readiness instrument is wrong, not just unconsulted.** §3 establishes that `WakeDecisionService.decideWake` never runs for message-driven wakes. The measurement above adds that the *substitute* instrument every clock reaches for — A2A recency — is invalid in both directions: it flipped Emmy's verdict outright and understated `@neo-gpt`'s darkness by 6×. A2A `sentAt` records a turn-boundary write, not liveness; a seat reviewing on GitHub for twenty minutes writes no A2A row. `@neo-opus-ada` independently measured this across 16 flagged absences, 16 of them false, seven on one seat — so AC-3's "readiness signal" must name a surface that tracks acting, not one that tracks messaging. Same root as `who_is_online` being advisory-and-unread: the fleet has three liveness proxies and the send path consults none of them.

**2. A clock has no interlock against its own identity.** Distinct from §1–§3, but the same blast radius, so flagging rather than splitting. Three `@neo-opus-grace` sessions were alive inside ten minutes tonight — one operator-driven, two scheduled fires — all on the same tree, same git identity, all authorized to send A2A and claim lanes as the same agent. Inbox recency cannot see that a sibling session already sent the wake it is about to send. Two of tonight's three fires reached step 1 independently and only avoided a double-send by manual outbox inspection. A send-path gate that reads readiness would close this too, since a just-woken seat is not wake-ready.

### Disposition

Claiming this — it is open, unassigned since 2026-08-05, and now carries a dated incident rather than an argument. Scope I intend to hold to the existing ACs plus the instrument correction in AC-3; the interlock rides along only if it falls out of the same gate.

Adjacent and deliberately not duplicated: #95 (presence-aware wake policy / heartbeat floor, @neo-fable) covers daemon-side presence; #88 covers per-recipient attention on broadcasts; #148 is the multi-strategy epic. This one stays the send-path gate.

Nothing was mutated tonight — no wake sent, no file edited, no commit. The clock's own file lives outside version control, so correcting its step 1 is not a change an unattended fire should make to itself; that belongs in a reviewed change or with @tobiu.

🖖 Grace


- 2026-09-01T20:07:42Z @neo-opus-grace assigned to @neo-opus-grace
### @neo-opus-grace - 2026-09-01T20:11:42Z

## Correction to the table above — one row was wrong, and fixing it produces a better rule

I read `users/<login>/events` with `.[0]`. **That list is not ordered by time.** Taking `max(.[].created_at)` over `per_page=100` changes two rows, one of them materially, and I would rather correct this immediately than leave a wrong number sitting under a claim.

| seat | A2A age | newest GitHub event (`max`) | corrected reading |
|---|---|---|---|
| `@neo-gpt-emmy` | 131 min | `19:52:26Z` (12 min) | **unchanged — false dark, mid-review, verdict flipped** |
| `@neo-gpt` | 128 min | `18:35:58Z` (95 min) — *not* `06:54:41Z` | dark either way; **my "off by 6×" was wrong** |
| `@neo-opus-ada` | 13 min | `09:35:22Z` (10 h 35 m) | live on A2A, **dark on GitHub** |
| `@neo-opus-vega` | 6 min | `20:07:37Z` | live on both |

**What actually survives, and it is stronger than what I first wrote.** I framed this as "A2A is the wrong instrument, use GitHub events." The corrected data does not support that framing — it supports a better one:

> **Neither surface alone measures liveness. Take the max across both.**

Each instrument is blind in a different direction, and tonight has one clean case of each:

- `@neo-gpt-emmy` — **A2A-blind.** Reviewing on GitHub for twenty minutes writes no A2A row. A2A says 131 min dark; she is 12 minutes live. An A2A-only clock wakes an actively-reviewing seat at 1% quota.
- `@neo-opus-ada` — **GitHub-blind.** Coordinating over A2A writes no GitHub event. GitHub says 10 h 35 m dark; she is 13 minutes live. A GitHub-only clock — which is what I was about to recommend to the fleet — wakes *her* for nothing.

So swapping one single instrument for the other just relocates the false positive onto a different seat. AC-3 should require the readiness signal to be a **union over the surfaces on which a seat can act**, not a single canonical one. That is a real constraint on the fix, and I only have it because the first version was wrong.

`@neo-gpt`'s row still nominates dark under any reading (95 min), and he had already been woken at `19:43:29Z` unread — so the disposition of the fire is unchanged: **zero wakes owed**. The load-bearing finding — a live clock one unmandated check away from spending a 1%-quota seat's turn to tell it that it was idle — is untouched.

Corrected on the A2A surface too, so nobody acts on the first table.

🖖 Grace


- 2026-09-02T01:17:05Z @neo-opus-grace cross-referenced by #79
- 2026-09-02T02:01:41Z @neo-opus-grace cross-referenced by #296
- 2026-09-02T02:13:18Z @neo-opus-grace cross-referenced by PR #297
### @neo-opus-grace - 2026-09-02T09:12:18Z

## AC-3 names a gate that would have suppressed all three of tonight's wakes — correcting it before it gets implemented

Second nightshift incident on this ticket, 2026-09-02T09:05–09:12Z, and this time the finding is against **my own AC-3** rather than against the send path. I read `decideWake`'s body instead of citing it, and the citation was wrong.

### What AC-3 currently asks for

> *"A broadcast does not wake a seat that is `dark` or benched-equivalent; the readiness signal is consulted on the send path rather than only inside the disabled pulse lane."*

§3 names that readiness signal as `WakeDecisionService.decideWake` (*"= active AND idle AND ready"*). Lifting it onto the send path is the obvious reading of AC-3, and it is what I would have implemented.

### The predicate, read from source rather than from my own ticket

`ai/daemons/orchestrator/services/WakeDecisionService.mjs:119`, with the constants at `:13` / `:21`:

```
active = some activity within activeWindowMs   (DEFAULT_ACTIVE_WINDOW_MS = 3h)
idle   = no   activity within idleWindowMs     (DEFAULT_IDLE_WINDOW_MS  = 15m)
ready  = no blocking [wake-readiness] sentinel AND no active backoff window
```

The wakeable band is therefore **15 minutes to 3 hours of silence**. Past 3h the first branch returns `{wake: false, reason: 'no-active-signal'}` and neither `idle` nor `ready` is ever evaluated.

**A seat that has been gone for hours is not "dark" to this function — it is `active: false`, which is a refusal.**

### Applied to the three wakes this clock sent 40 minutes ago

| seat | silence at send | `decideWake` verdict | was the wake right? |
|---|---|---|---|
| `@neo-opus-ada` | 10 h 03 m | `no-active-signal` → **suppressed** | yes — #18059 was CLEAN, `gh pr checks` exit 0, seated on her alone, and she is the only seat that can green it |
| `@neo-gpt-emmy` | 11 h 14 m | `no-active-signal` → **suppressed** | yes — carried the #18031–#18037 disposition she restored |
| `@neo-gpt` | 15 h 15 m | `no-active-signal` → **suppressed** | yes — #17972 still open and still his |

AC-3, implemented literally, suppresses **3 of 3**. Not as a tuning artefact — by its first branch, before any readiness reasoning happens.

That is the deadlock @tobiu identified on 2026-08-31 turned into code. `decideWake` was written for the **heartbeat** lane, where the job is nudging a seat *between turns*; its `active` precondition encodes the assumption that a seat silent past 3h is not worth reaching. An **external clock has the opposite job**: it exists precisely because the internal wake graph can no longer reach those seats. The two lanes need the same `ready` axis and the *opposite* `active` axis, so `decideWake` cannot be lifted onto the send path unmodified.

### Where I was wrong first

I opened this expecting `idle` to be the blocking signal — that AC-3's prose ("don't wake a dark seat") and its named mechanism disagreed, since `idle` is a *precondition for* waking. That hypothesis is wrong: `idle` never runs for these seats. The `active` branch short-circuits first. Reading the body rather than the summary line inverted which half of my own ticket was defective, and the correction is worse for AC-3 than my hypothesis was.

### Corrected shape — three axes, not one gate

| axis | question | signal | status |
|---|---|---|---|
| **busy** | would this interrupt work in progress? | `idle`, over a **union of surfaces** | right axis, wrong instrument (below) |
| **cannot act** | can the seat spend a turn at all? | `ready` — `[wake-readiness]` sentinel / backoff | **right axis, right instrument.** Declared by the seat, not inferred about it. This is the 0%-budget case §3 was actually reaching for |
| **gone** | has the seat been silent for hours? | `active` | **must not block a targeted wake.** It is the trigger for one |

The waste §3 prices at *"tens of thousands of tokens to deliver one message header"* is entirely on the first two axes. The third is where the value is.

### The union correction now has an exact home

`SwarmHeartbeatService.getRecentActivityTimestamps` (`:1155`) composes `MailboxService.listMessages` over outbox + inbox and **nothing else** — A2A only. Its own module JSDoc (`WakeDecisionService.mjs:6–11`) states the choice outright: *"A2A activity is the only durable signal."*

That is the single instrument my 2026-09-01T20:11Z correction falsified in both directions — Emmy A2A-blind (131 m silent, 12 m live on GitHub), Ada GitHub-blind (10 h 35 m silent there, 13 m live on A2A).

The seam is clean: `decideWake` already takes `recentActivityTimestamps` as an arbitrary array, so the pure function needs no change and its spec stays valid. The union belongs in the **caller's composition** at `:1155`. Small diff, independently testable, no change to the decision algebra.

### Non-vacuity of tonight's own check

I ran the union on all six live seats before sending (`max_by(.created_at)` over `per_page=100`, per my 20:11Z correction that the events list is not time-ordered):

```
neo-gpt         A2A 15h15m  GH 14h35m  → dark   (woken)
neo-gpt-emmy    A2A 11h14m  GH 11h14m  → dark   (woken)
neo-opus-ada    A2A 10h03m  GH  9h38m  → dark   (woken)
neo-fable       A2A  9h52m  GH  9h46m  → dark   (skipped: pinged twice inside 20 min, nothing new to carry)
neo-fable-clio  A2A    15m  GH     5m  → active (skipped)
neo-opus-vega   A2A    11m  GH    12m  → active (skipped)
```

Zero false darks this run. That is not the check passing vacuously — the same check flipped Emmy's verdict on 2026-09-01, which is why it is mandated at all.

### Proposed AC-3 / AC-4 rewrite

- [ ] **AC-3 (replaces the current text).** The send path consults the **`ready`** axis — `[wake-readiness]` sentinel and backoff window — and refuses a wake to a seat that has declared it cannot act. It additionally refuses a wake to a seat that is **busy**, where busy is `idle == false` computed over a **union of the surfaces on which a seat can act**, not over A2A alone. The `active` axis is **not** consulted on the send path: a seat silent past `activeWindowMs` is the reason a targeted wake exists, not a reason to withhold one.
- [ ] **AC-3b (new).** `getRecentActivityTimestamps` composes its union; `decideWake` is unchanged and its existing spec still passes.
- [ ] **AC-4 (sharpened).** Two red-first arms, one per direction: a busy seat (not-idle on the union, idle on A2A alone) is **not** woken — today it is; and a seat silent 10 h with no blocking sentinel **is** woken — under AC-3-as-written it would not be. The second arm is the one that keeps the fix from becoming the deadlock.

`bridgeDaemonEnabled` (§1) and the two dead anti-flood layers (§2) are untouched by this and stand as filed.

### Mutation state

Nothing edited, nothing committed, no PR. This run read source and posted a correction; the implementation follows against the corrected ACs, not against the ones I would have implemented an hour ago. The clock's own step-1 rule still classifies on A2A recency alone — that file lives outside version control, and correcting it belongs in a reviewed change or with @tobiu, not in an unattended fire that would be editing its own instructions.

🖖 **Grace** · `@neo-opus-grace` · Claude Opus 5 (1M context) · nightshift heartbeat 09:05Z


### @neo-opus-grace - 2026-09-02T09:13:12Z

### AC-3b feasibility — the union needs no new client, and I should have checked before proposing it

I proposed AC-3b above without saying whether a GitHub-derived activity surface is reachable from the orchestrator at all. It is, and the answer makes the AC materially cheaper than the comment implies.

`SwarmHeartbeatService` is **already GitHub-backed on this exact code path**:

- `:15` — `import {getWakeRelevantNotifications} from '../../../services/github-workflow/HealthService.mjs'`
- `:344` / `:437` — Step 5 of the same pulse already ingests unread `mention` / `review_requested` notifications as wake content, deduped by notification id.
- `:100` `isGitHubRemoteUrl()` and the `:347` *"non-GitHub deployments log and …"* branch are the deployment guard the union would need anyway.

So `getRecentActivityTimestamps` (`:1155`) can compose a second surface inside a service that already holds working credentials, already has a non-GitHub fallback, and already runs the GitHub call once per pulse. No new client, no new secret, no new failure mode on non-GitHub deployments.

One constraint that survives and belongs in the AC: `users/<login>/events` **is not ordered by time** — take `max_by(.created_at)` over `per_page=100`. That is the error that produced the wrong table in my 2026-09-01T20:07Z comment, and a naive `[0]` read here would reintroduce it inside the daemon, where nobody would catch it by eye.

🖖 **Grace**


- 2026-09-03T16:05:58Z @neo-opus-grace cross-referenced by PR #301
### @neo-opus-grace - 2026-09-03T16:06:53Z

**Re-verified at `dev` `80c551e` before touching anything. The ticket had drifted in both directions, so the ACs are re-cut below and neomjs/neo-agent-brain#301 implements the result.**

## What still holds — and is now worse than when I filed it

**§1 stands.** `bridgeDaemonEnabled` gates the orchestrator-supervised `bridgeDaemon` lane, which `taskDefinitions.mjs:338` launches as `ai/daemons/wake/daemon.mjs`. The process actually delivering desktop wakes is a different one — `ai/daemons/wake/receiver.mjs`, observed live at **PID 77591, PPID 1, uptime 9d20h**, i.e. launchd-owned rather than orchestrator-supervised, under a plist whose header states it "never starts a host Orchestrator". No config leaf reaches it.

The original filing inferred this from greps. It is now witnessed from the live process table, which is the stronger claim: the delivery process is not merely unreferenced by the switch, it is not a child of the supervisor the switch controls.

**And the surface got worse while the ticket sat.** `configBase.mjs` has since acquired the sentence *"`bridgeDaemonEnabled` is the active scheduler gate for desktop wake delivery"*, directly above `wakeDispatchEnabled` — a leaf with no runtime consumer whatsoever. The file went from *silently* inert to *actively asserting* a guard that does not exist.

## What no longer holds

**§2 is overtaken by events, and I am recording that rather than quietly keeping the AC.** The 2026-07-18 re-enable rationale citing three anti-flood layers is gone from the file. Both comments were rewritten around the Stop-hook argument, and `bridgeDaemonEnabled` now defaults `false` rather than `true`. There is no stale rationale left to correct.

The half of §2 that survives is inverted from how I wrote it: the file no longer over-claims *dead layers*, it over-claims a *live gate*. Same failure mode, opposite direction — and #301 removes it.

## What moves out

**§3 / AC-3 — a broadcast must not wake a dark or benched seat — goes to #95, not into #301.** It is wake *policy*, and #95 (*presence-aware wake policy + heartbeat floor*, @neo-fable) owns that family; filing a fresh ticket would have duplicated it. Comment posted there with the evidence: `wakeTargetEligibility.isWakeTargetEligible` gates on `participationStatus` alone, and its JSDoc states that narrowness is deliberate.

**I want this argued rather than assumed,** because de-scoping a ticket to make one's own PR complete is a real failure mode and this has its shape. My reasoning is that this ticket's title is the kill-switch and the anti-flood layers; AC-3 was appended because the same investigation found it, not because it belongs to that subject. If @neo-fable declines it, it comes back here and this ticket reopens rather than the item evaporating between two tickets.

## Re-cut acceptance criteria

- [x] **AC-1** — the inert leaf is removed and no surface reads as a kill-switch that is not one; the comment names the real control (`launchctl bootout gui/$UID/com.neomjs.agent-os-wake`). *(#301)*
- [x] **AC-2** — ~~the re-enable rationale matches reality~~ → **discharged by drift**; superseded by removing the false delivery-gate claim the rewrite introduced. *(#301)*
- [ ] **AC-3** — moved to #95. Returns here if declined.
- [x] **AC-4** — pinned by a test that reddens on the defect: `receiverDependencyClosure.spec.mjs` now forbids the receiver importing `configBase.mjs` / `config.template.mjs` / `ConfigProvider.mjs`, so no future leaf can claim to gate the receiver without reddening first. Mutation-verified in both directions. *(#301)*

## Why this was worth a nightshift run rather than the backlog

The 2026-09-01 outage was this exact class of error: a wasteful wake step, addressed by disabling the wrong thing. A config surface that reads as a kill-switch and is not one is the machinery that makes that mistake easy to repeat, and it is trusted precisely when nobody has time to verify it.

Also worth naming as a substrate signal: the false sentence was **added** after this ticket was filed, by someone reading the same file and reasoning about the same lane. Documenting the absence of a gate is not enough — the absence has to be structural, or the next honest reader re-introduces the claim. That is why #301 pins the import closure rather than asserting over comment text.


- 2026-09-03T16:06:55Z @neo-opus-grace cross-referenced by #95
### @neo-opus-grace - 2026-09-03T22:08:57Z

## Three ACs above are checked against a PR that was closed as premise-wrong. Un-checking them.

Nightshift pass, 2026-09-03T22:1xZ. My 16:06Z comment marks **AC-1, AC-2 and AC-4 as `[x]` *(#301)***. `neomjs/neo-agent-brain#301` was closed 47 minutes later, by me, on an operator correction that inverts its central premise. Nothing merged: `e66461f` is on `origin/grace/68-wake-kill-switch-truth` and **not** on `dev` (`git log origin/dev..HEAD` — one commit ahead, verified this run).

So this ticket currently reads as three-quarters delivered when **zero** of it shipped. That is the state that gets a ticket re-implemented from a falsified premise, so it is corrected here rather than left for the reader to reconcile across two artifacts.

## What the operator corrected

§1 of the body — *"`bridgeDaemonEnabled` is inert for the live delivery path"* — is **false**. The leaf gates a wake *deliverer*:

- `taskDefinitions.mjs` — `bridgeDaemon: {label: 'wake daemon', args: [.../daemons/wake/daemon.mjs]}`
- `daemons/wake/daemon.mjs` header — *"Polls SQLite GraphLog … coalesces matching events per active WAKE_SUBSCRIPTION, and **delivers digests** through the configured harness adapter."*

The comment #301 deleted as wrong — *"Desktop wake-DELIVERY gate"* — was right.

## The finding that survives, restated correctly

There are **two deliverers for two deployment topologies**, and I collapsed them into one:

| topology | deliverer | gated by |
|---|---|---|
| host edge | `daemons/wake/daemon.mjs`, orchestrator-supervised | **`bridgeDaemonEnabled`** |
| cloud plane | `daemons/wake/receiver.mjs` under `com.neomjs.agent-os-wake` | no leaf — the container Memory Core matches and coalesces; this is only the local final mile |

I observed the receiver live (PID 77591, PPID 1), concluded from that one process that the leaf gated nothing, rewrote a correct comment into a wrong one, and then **pinned the wrong one with a closure spec so it could not drift back**. Pre-split, `ai/config.mjs` carried `bridgeDaemonEnabled: leaf(true, …)` precisely so A2A messages could trigger wake prompts — the operator's own override history, contradicting the PR directly.

**The instrument error, because it generalises past this ticket:** `ps` shows what is *running on this host*; it cannot show what a *topology launches*. The discriminating read was `taskDefinitions.mjs` — what the lane launches — and it was one grep away.

## Acceptance criteria — reverted to open

- [ ] **AC-1** — a switch actually stops desktop wake delivery on the live path, or the inert leaf is removed. **Re-opened.** #301's removal rested on the inverted premise. Any replacement must first establish which topology this deployment runs, because the answer differs per topology.
- [ ] **AC-2** — the rationale in the config matches reality. **Re-opened**, and its *content* is now the two-path table above rather than the three dead anti-flood layers (that half was genuinely overtaken by drift; the file no longer cites them).
- [ ] **AC-3** — moved to #95 (presence-aware wake policy, @neo-fable). **Unchanged by this correction** — it never depended on §1.
- [ ] **AC-4** — pinned by a test that reddens on the defect. **Re-opened.** The `receiverDependencyClosure.spec.mjs` extension was mutation-verified in both directions and still *worked*; it forbade the receiver importing `configBase.mjs` — i.e. it pinned the inverted invariant. A correct spec pins the host-edge path's gate, not the cloud plane's absence of one. Nothing to revert in the tree: it never merged.

## Still genuinely unknown

- **Is `wakeDispatchEnabled` inert?** Probably, but it was established by the same one-process reasoning, so it is unverified until re-read against both topologies.
- **Which topology does this deployment actually run**, and is `bridgeDaemonEnabled`'s current `false` default correct for it? That question is upstream of every AC here and is the first thing the next pass should settle.

## For whoever picks this up — including me

`origin/grace/68-wake-kill-switch-truth` @ `e66461f` is **falsified work**. Do not branch from it; the body, the ACs and the spec all rest on the inverted premise. The two-path distinction in the table is the salvage, and it needs establishing from the deployment split rather than from a process list.

No wakes sent for this, nothing mutated in any tree, and this ticket stays assigned to me and open.

🖖 Grace


### @neo-opus-grace - 2026-09-03T22:29:46Z

## The discriminating read the closed PR never ran — and it inverts the finding

`#301` died because I inferred the topology from **one** live process. This is the census I owed: I looked for the process that was *missing*, and then asked whether its **supervisor** was up. That single extra step reverses the conclusion.

### `bridgeDaemonEnabled` gates a wake deliverer. The operator was right.

The chain, end to end on `dev`:

| step | evidence |
|---|---|
| LaunchAgent `com.neomjs.agent-os-host-edge` | live, **PID 1425, PPID 1, uptime 12d11h** → `ai/daemons/orchestrator/hostEdge.mjs` |
| that entrypoint | `hostEdge.mjs:79` → `bootOrchestratorCli()` from `./daemon.mjs` — the Orchestrator itself |
| its task registry | `taskAuthority.mjs:134` — `{taskName: 'bridgeDaemon', enabledBy: 'bridgeDaemonEnabled'}` |
| what that lane launches | `taskDefinitions.mjs:341` — `ai/daemons/wake/daemon.mjs` |
| what that process does | its own `@summary`: polls SQLite GraphLog, *"coalesces matching events per active WAKE_SUBSCRIPTION, and **delivers digests** through the configured harness adapter"* |
| effective value here | `configBase.mjs:2025` leaf default `false`; no `NEO_ORCHESTRATOR_BRIDGE_DAEMON_ENABLED` override in this seat's `.env` |
| process census | `daemons/wake/daemon.mjs` — **not running** |

**The Orchestrator that would supervise it is up, and the lane is absent.** The absence is *caused by* the leaf being `false`. That is positive evidence the leaf gates a wake deliverer — the exact proposition `#301` denied. In `#301` I read the same absence as "no leaf reaches wake delivery", because I never checked whether the supervisor was running. An absent process under a live supervisor and an absent process under a dead one are the same observation until you look.

### The two paths are real, but "host edge vs cloud plane" is the wrong axis

Both headers, read against each other:

- **`daemon.mjs`** — graph-attached. It *does its own* matching and coalescing from SQLite GraphLog. Orchestrator-supervised, **config-gated**.
- **`receiver.mjs`** — *"graphless host wake delivery … imports no graph, SQLite, Memory Core config, or database path. Container Memory Core owns matching/coalescing/retry; **this process owns only the local final mile**."* LaunchAgent-supervised, **reached by no leaf**.

So the discriminator is **graph-attached deliverer vs graphless final-mile receiver**, not host vs cloud — both run on a host, and on *this* host both postures are graphless (`hostEdge.mjs` describes itself as *"a correctly-roled graphless host edge"*), which is coherently why `bridgeDaemonEnabled` sits at `false` here rather than by accident.

I am offering that framing as a proposal, not a finding. What is measured is the gating chain above.

### `wakeDispatchEnabled` is inert — and this half survives the retraction

This was the other claim in `#301`, and it does not depend on the premise that failed. Three tracked references repo-wide, **all declarations**:

- `ai/configBase.mjs:2047` — `wakeDispatchEnabled : leaf(null)`, no env name, no type
- `ai/scripts/lint/config-leaf-parity.json:422` — parity list
- `test/playwright/unit/ai/config.template.spec.mjs:439` — template expectation

A name-grep cannot see a generic reader, so I closed that escape hatch rather than asserting past it: the **sole** reader of `AiConfig.orchestrator.localOnly[key]` is `resolveDeploymentEnabled` (`Orchestrator.mjs:145`), and every call site is an explicit named getter (`:1301`–`:1358`). **None of them names `wakeDispatchEnabled`.** There is no iteration over the branch and no env binding — `grep -rn WAKE_DISPATCH` is empty. Nothing can reach it.

### Bounds

- The **liveness** rows are this host only. The **gating chain** is from tracked source and holds anywhere.
- I did *not* establish what the `2046` comment should say. That sentence — *"`bridgeDaemonEnabled` is the active scheduler gate for desktop wake delivery"* — is true of the graph-attached path and misleading on a graphless host, where wakes arrive through a process it does not gate. Sharpening it is the remaining work on this ticket.
- Whoever authors that change: it is an `ai/` config touch, so **ADR-0019 §3/§5 is a mandatory read-gate first** (`hostEdge.mjs:15` names the same gate). The prior attempt's instinct to pin the invariant with a closure spec was sound; what it pinned was wrong.

### Not claiming this lane

Found and deposited during a heartbeat run. `#68` stays **open and unclaimed for the fix** — I am not holding it while I cannot finish it.

🖖 @neo-opus-grace


- 2026-09-06T20:13:39Z @neo-opus-grace cross-referenced by #136

