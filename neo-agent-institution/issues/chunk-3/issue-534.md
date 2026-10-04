---
id: 534
title: 'Row 1''s installed walkthrough: a cold first run reaches done'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable
createdAt: '2026-10-04T11:10:04Z'
updatedAt: '2026-10-04T14:40:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/534'
author: neo-fable
commentsCount: 1
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 481 The setup card witnesses verify through to done on the fixture plane'
  - '[x] 523 A walker can hold smoke''s isolated organism open and drive its plane'
blocking: []
milestone: FM v1
---
# Row 1's installed walkthrough: a cold first run reaches done

## Context

Row 1 of FM v1 (ROADMAP: *first run and connection*) passes when a cold first run of the installed Fleet Manager reaches, through the setup card alone, a working institution that answers a query and holds its first persisted memory (#351's terminal predicate). Rows 2 to 5 have a walk leaf each (#479, #485, #490, #516). Row 1 has none, and the card has never been driven to `done` on an installed candidate.

The planner accepted this leaf from the card's gap list on 2026-10-03 ([disposition, lines 5 and 6](https://github.com/neomjs/neo-agent-institution/issues/351#issuecomment-5971732569)): a cold host to `done` with one receipt per step, in two halves.

## The Problem

A merged PR never retires an installed check (ROADMAP accounting), so row 1 stays `blocked` whatever lands on `dev`. Three signals say the walk can fail where no unit test looks:

- **The card was built against its design page, never against a stranger.** The builder's own count on 2026-10-03 was 6 decisions across 11 rows named by recipe id ([`5971695936`](https://github.com/neomjs/neo-agent-institution/issues/351#issuecomment-5971695936)). The guided front is accepted as direction and unbuilt.
- **No test runs the card against the pinned recipe.** `test/playwright/e2e/agentos/FleetSetupCard.spec.mjs` drives a sample recipe with three effects through a recorded shell. The pinned Brain's recipe has a fourth, `verify`, and the card has never met it.
- **A refused `verify` write leaves a `run` button that does nothing** ([#481's intake](https://github.com/neomjs/neo-agent-institution/issues/481#issuecomment-5979287502)).

## The Architectural Reality

- **Candidate.** The installed bundle today is Institution `e1a9dbe`, Brain `fb40366`, engine `82bc615`; it predates the `verify` effect. `dev` pins Brain `5d466610`, which carries it. A receipt can name its Institution commit once #532 lands.
- **Host effects are not isolatable on the operator's machine.** `compose-up` is `docker compose -p <project> up -d --wait` (`ai/services/fleet/hostEffects.mjs:152` at the pin). The project name is a constant, whatever the state root. On a host whose plane is running the recipe reads the row `ok — observed; not performed by this run` and nothing runs (probed on 2026-10-04, #481); on a host whose plane is stopped it would start that same project with the walking organism's env file. So no peer consents to a host effect on the operator's daily machine.
- **Isolation.** #523 lets a walker hold smoke's isolated organism open; it boots from a stored plane record against the fixture plane. Whether the Create door is reachable in that state is the first half's first check, and its prerequisite if it is not.
- **The expected words** are the recipe's own: each step's `summary`, `status` and `reason` as `apps/agentos/view/setup/StepList.mjs` renders them, and the one action `actionFor` gives the row.

## The Fix

No feature work. Two halves, every step ending as **pass**, **fail**, **missing** or **blocked**, with a receipt: the capture time, the words the surface showed, the action it offered, and what the walker had to know or supply that the surface did not say.

1. **Peer-side, isolated** (walker: a seat that neither built nor designed the card). On the held smoke organism the walker opens the Create door and walks every question frame: placement, preset, the advanced fold, the credential window with a fixture value, and the consent rows as they read before anything runs. No host effect is consented. Per frame the receipt records the decisions asked, the words a stranger does not have, and whether the one next action is readable.
2. **`[human]`, the real host.** On a machine the operator chooses (a second Mac or a fresh VM, with Docker), a cold install of the candidate runs the card to the end: the three host effects, `served-plane`, `validation`, `verify`, `done`. Then one query is answered and the witness memory is read back in the cockpit. This is the row's installed check.

The steward prepares the step list and the receipt table before each half, and removes the isolated organism after the first.

## Acceptance Criteria

- [ ] The receipt names the candidate (Institution, Brain and engine commits), the machine, the placement and the preset.
- [ ] Half 1: a non-builder walks the card's question frames on the isolated organism, one receipt per frame, recorded on #351. No host effect runs, and afterwards the operator's app profile and the team's plane are unchanged.
- [ ] Half 2, `[human]`: on a machine the operator chooses, a cold run reaches `done ok` through the card alone, one receipt per step, with the elapsed time to the first persisted memory (#14's read).
- [ ] After `done`: one query is answered and the witness memory is readable from the cockpit, with its receipt.
- [ ] Each failed or missing step reaches a planner as a `defect-note:` with its receipt and becomes a leaf under #351 only after the walk. #351's `Row state:` line is updated and broadcast as the row report.

## Out of Scope

- **Changing any surface.** A failed step gets its own leaf after the walk.
- **The enrollment half** (Add Agent, Start, the five readbacks): neomjs/neo-agent-brain#571's own walk, stewarded by Ada and Emmy.
- **The guided front** and **the new-attempt exit**: their own leaves, behind the stranger read and #481.
- **A hosted or cloud placement** (neomjs/neo-agent-brain#697): outside v1 unless the operator decides otherwise.

## Avoided Traps

- **The builder walking her own card.** A builder knows which row is wired; that blindness is what the walk removes (neomjs/neo#19384).
- **Running `compose-up` beside the live plane to "test the real thing".** It targets the live plane's own compose project; only the recipe's observation rule keeps it from acting there.
- **Calling the row passed from a fixture run.** The fixture half proves the frames read; only the real host proves the predicate.

## Related

#351 (the row's epic) · #481 · #523 (isolation) · #532 (the candidate's own revision) · #14 (the first-persistence instrument; its event is decided there) · siblings #479, #485, #490, #516 · neomjs/neo-agent-brain#571 (the enrollment half)

Sweeps (2026-10-04):
- Live latest-open Institution sweep: the latest 20 open issues, created-descending, read at 11:09Z (#533 … #476). No row-1 walk leaf; rows 2 to 5 have theirs.
- A2A: the last 14 messages of all read states. No competing claim on a row-1 walk.
- Memory Core: one query on the unwalked card. Prior record: installed-host execution and density stay with #351 (the #464 review); no decision against a walk leaf.
- Own-assignment: #481, #475, #391 read; none is a walk leaf.
- Structure map: N/A, an acceptance walk with no file placement.

Decision Record impact: `none`.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "row 1 installed walkthrough setup card cold first run done ok real host human row isolated question frames"



## Timeline

- 2026-10-04T11:10:04Z @neo-fable assigned to @neo-fable
- 2026-10-04T11:10:06Z @neo-fable added the `enhancement` label
- 2026-10-04T11:10:06Z @neo-fable added the `agent-os` label
- 2026-10-04T11:10:06Z @neo-fable added the `ai` label
- 2026-10-04T11:10:06Z @neo-fable added the `testing` label
- 2026-10-04T11:10:13Z @neo-fable added parent issue #351
- 2026-10-04T11:10:14Z @neo-fable marked this issue as being blocked by #523
- 2026-10-04T11:10:16Z @neo-fable marked this issue as being blocked by #481
- 2026-10-04T11:10:30Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T11:10:53Z @neo-fable cross-referenced by #351
- 2026-10-04T11:16:32Z @neo-fable cross-referenced by #535
- 2026-10-04T11:34:04Z @neo-gpt-sophie cross-referenced by #14
- 2026-10-04T11:44:36Z @neo-opus-ada cross-referenced by PR #537
- 2026-10-04T12:12:12Z @neo-gpt-emmy cross-referenced by #481
- 2026-10-04T12:42:07Z @neo-fable cross-referenced by #540
- 2026-10-04T12:44:13Z @neo-fable cross-referenced by PR #541
### @neo-fable - 2026-10-04T14:40:37Z

## Half 1: step list and receipt table

Prepared by the row's steward on 2026-10-04, from the card's source at `dev@194c9d7`. The walker is a seat that neither built nor designed the card (so not Mnemosyne, not Clio).

### What half 1 needs before it starts

- **A packaged candidate carrying #537 and #541.** The Create door mounts only in a packaged shell: `ViewportController.mountPlaneSetup` checks `status.packaged`, and the broker answers `not-packaged` otherwise (`harness/setupBroker.mjs:183`). An unpackaged held run cannot show the door. That is the next #12 candidate.
- **A held smoke run on the fixture-plane arm** (`NEO_HARNESS_SMOKE_HOLD=1`; contract and commands on #523). It opens a window on the machine it runs on. On the operator's machine that needs his yes for the slot, as row 5's cut does.
- **The walker drives through the Neural Link**: reads the words from the component tree, presses only what this list names.

### Step 0: is the Create door reachable?

The held organism boots with a configured plane, so no card shows. From the source the path is: the shell switcher's attach-plane entry (`onAttachPlane`) → the card opens on Connect → the `Create` door button (`Panel.mjs`, `create-door-button`). Record what the walker actually had to do. `blocked` here ends the half and becomes its prerequisite.

### Frames

One receipt per frame. Nothing is consented beyond what the row itself asks.

| # | Frame | What to read |
|---|---|---|
| 1 | The door | the lede |
| 2 | `placement` | the budget line, the placement line, the row's action (`re-read`) |
| 3 | `preset` | one card per preset: name, facts, footprint, verdict with its reason, `choose`; a refused preset cannot be chosen. Choose the recommended one (the consent lands in the run's record under the smoke root) |
| 4 | `plane credential` | the button and its note; the vessel's credential window with the fixture's token; afterwards the row names a path, never a value. If the window cannot be driven through the Neural Link, the frame is `blocked`, and that is its receipt |
| 5 | `provider key` | for a local preset: the sentence that no key is needed, and no action |
| 6 | `advanced` | folded, with `unfold` |
| 7 | The rows before anything runs | `write-secrets`, `write-env`, `compose-up`, `served-plane`, `validation`, `verify`, `done`: status word, reason, the one action each offers. **Stop here: no `run` is pressed** |
| 8 | The status line and the quiet line | what they say with nothing run |

### Receipt columns

time (UTC) · the words shown, verbatim · the action offered · the decisions asked of the walker · what the walker had to know or supply that the surface did not say · `pass` / `fail` / `missing` / `blocked`

### Known before the walk

These are to confirm, with the words the surface shows:

- On a machine whose local plane runs, the three host-effect rows can read `ok` with "observed; not performed by this run". The compose project name is a constant, so the card reads the live plane's project. Record it; press nothing.
- The candidate lists the carrier row before the secret-files row until a Brain pin carries neomjs/neo-agent-brain#844.
- Several pending rows offer `run`, and the top one answers nothing: #535 and neomjs/neo-agent-brain#840 own those.

### After

`node harness/walkControl.mjs --packaged cleanup`, then the steward checks that the operator's app profile and the team's plane are unchanged and posts the table on #351.

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 577754b6-3d27-48f5-911a-434605a54220

- 2026-10-04T14:50:56Z @neo-fable cross-referenced by #547

