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
  - neo-fable-mnemosyne
createdAt: '2026-10-04T11:10:04Z'
updatedAt: '2026-10-18T17:18:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/534'
author: neo-fable-mnemosyne
commentsCount: 2
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

- 2026-10-04T11:10:04Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-04T11:10:06Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-04T11:10:06Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-04T11:10:06Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-04T11:10:06Z @neo-fable-mnemosyne added the `testing` label
- 2026-10-04T11:10:13Z @neo-fable-mnemosyne added parent issue #351
- 2026-10-04T11:10:14Z @neo-fable-mnemosyne marked this issue as being blocked by #523
- 2026-10-04T11:10:16Z @neo-fable-mnemosyne marked this issue as being blocked by #481
- 2026-10-04T11:10:30Z @neo-fable-mnemosyne added this to the **FM v1** milestone
- 2026-10-04T11:10:53Z @neo-fable-mnemosyne cross-referenced by #351
- 2026-10-04T11:16:32Z @neo-fable-mnemosyne cross-referenced by #535
- 2026-10-04T11:34:04Z @neo-gpt-sophie cross-referenced by #14
- 2026-10-04T11:44:36Z @neo-opus-ada cross-referenced by PR #537
- 2026-10-04T12:12:12Z @neo-gpt-emmy cross-referenced by #481
- 2026-10-04T12:42:07Z @neo-fable-mnemosyne cross-referenced by #540
- 2026-10-04T12:44:13Z @neo-fable-mnemosyne cross-referenced by PR #541
### @neo-fable-mnemosyne - 2026-10-04T14:40:37Z

## Half 1: step list and receipt table

Prepared by the row's steward; first written 2026-10-04 from `dev@194c9d7`, **refreshed 2026-10-17 from `dev@fa1a460`** (seven setup-card PRs landed between: #555, #558, #561, #613, #621, #623, #721). The walker is a seat that neither built nor designed the card (so not Mnemosyne, not Clio); Sophie did the 10-04 stranger read, so a seat that has never seen the card reads freshest.

### What half 1 needs before it starts

- **Friday's candidate: a package cut at Institution ≥ `fa1a460`.** The installed 10-14 bundle (`afabdef6`) and the 10-16 package (`51078674`, #12) both carry the three-question front (#613) but predate the fresh run (#721). Brain = the cut's pin (today `b96b4d34`, package.json:69, which carries neomjs/neo-agent-brain#844's one effect order); engine = its pin. The receipt names all three from `organism-build-info.json` `owners.*.revision` (#532 landed).
- **The hold (#523 landed):** the installed candidate launched with `NEO_HARNESS_SMOKE=1 NEO_HARNESS_SMOKE_PLANE=1 NEO_HARNESS_SMOKE_HOLD=1` (the fixture-plane arm only; anywhere else it refuses before any profile path is set). It logs `HARNESS_SMOKE_HOLD` with the candidate, the auth mode, the plane base and the smoke root, and writes `<smokeRoot>/walk/manifest.json`. It opens a window on the machine it runs on, so it takes the operator's slot — Friday's.
- **The walker drives through the Neural Link**: reads the words from the component tree, presses only what this list names.

### Step 0: is the Create door reachable?

The held organism boots attached to its fixture plane, so no card mounts at boot (`mountPlaneSetup` mounts for a packaged shell with no plane). The card comes back on request (`showPlaneSetup`, `force: true`): the shell switcher's attach-plane entry, the banner's Connect, or Home's doors. With a plane configured Home shows the keeper doors, so the expected path is the shell switcher → the card on the door `choosePrimaryDoor` picks → the `Create` door button (`Panel.mjs`, `create-door-button`). Record what the walker actually had to do. `blocked` here ends the half and becomes its prerequisite.

### Frames

One receipt per frame. Nothing is consented beyond what the frame itself asks; **no `run` and no `Run next step` is pressed.**

| # | Frame | What to read |
|---|---|---|
| 0 | Home, if the first-run face shows | the promise line, the primary `Set up your institution` with its line `A GitHub token · where it runs · start.`, the quiet `Joining a team that already runs one?` → `Connect to it` |
| 1 | The Create door's lede | `Three questions, then one action at a time. Nothing is written until you say so, and every step says what it observed.` |
| 2 | Q1, the token | the line (`One token. It lets your agents read and write your repositories, and it signs you in here.`), `Enter your token`, the help line; the app's own credential window with the fixture's token; afterwards the row names a path, never a value. If the window cannot be driven through the Neural Link, the frame is `blocked`, and that is its receipt |
| 3 | Q2, where it runs | `Measuring this machine…` → the placement line and `Use this`; `Other choices` → one card per preset: name, facts, footprint, verdict with its reason, `choose`; a refused preset cannot be chosen; the budget line (`not probed yet` → its reading); `Enter the provider key` only when the chosen preset needs one, for a local preset the sentence that no key is needed. Choose the recommended one (the consent lands in the run's record under the smoke root) |
| 4 | Q3, start | the start line, the one action `Run next step`, the start help — read, not pressed |
| 5 | The ledger | the details toggle; then the pin's 13 rows as they read before anything runs — `placement`, `preset`, `plane-credential`, `provider-key`, `advanced`, `write-secrets`, `write-env`, `compose-up`, `register-forge`, `served-plane`, `validation`, `verify`, `done`: status word, reason, a `waits for <id>` chip where one shows, and the one action the row offers (`run` · `re-check` · `write again` · `unfold` · `change` · `re-read`). **Stop here** |
| 6 | The status line and the quiet line | what they say with nothing run |

### Receipt columns

time (UTC) · the words shown, verbatim · the action offered · the decisions asked of the walker · what the walker had to know or supply that the surface did not say · `pass` / `fail` / `missing` / `blocked`

### Known before the walk

To confirm, with the words the surface shows:

- On a machine whose local plane runs, the host-effect rows can read `ok` with "observed; not performed by this run": the compose project name is a constant, so the card reads the live plane's project. Record it; press nothing.
- Before the plane is up, the `served-plane` row may read a transport code (`connect ECONNREFUSED …`) instead of a sentence — the planner's gap line on #351 (5982813310); the pinned observer still shows what the health check throws. Record the exact words.
- `register-forge` is a row now; nobody has read its words before anything runs.

### AC-5 of #475 — the stuck row → `start a fresh run` → `done`

Needs a run record with a receipt to damage: `test/playwright/e2e/agentos/FleetSetupCard.spec.mjs:255–309` sets the `write-secrets` receipt back to `pending` on disk, a re-read finds the row `reconcile-required` with `re-check` alone, re-check keeps it, the row then says `re-checked · unsettled` and offers `start a fresh run`, the lede reads `Fresh run <id> — the run <old> stays on disk as history…`, and the run reaches `done`. Half 1 runs no effect, so there is nothing to damage; this is **half 2's last step**, on the operator's test host after `done`, by the operator's word.

### After

`node harness/walkControl.mjs --packaged cleanup` (both plane ports closed, the smoke root gone), then the steward checks that the operator's app profile and the team's plane are unchanged and posts the table on #351; the `Row state:` line is updated and broadcast as the row report. Half 2's elapsed time to the first persisted memory is a stopwatch read until #14 lands.

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · first prepared in session 577754b6-3d27-48f5-911a-434605a54220, refreshed 2026-10-17


- 2026-10-04T14:50:56Z @neo-fable-mnemosyne cross-referenced by #547
- 2026-10-04T16:22:47Z @neo-fable-mnemosyne cross-referenced by #550
- 2026-10-04T17:06:23Z @neo-gpt-euclid cross-referenced by PR #549
- 2026-10-04T17:28:35Z @neo-fable-clio cross-referenced by #858
- 2026-10-04T18:34:00Z @neo-fable-mnemosyne cross-referenced by PR #555
- 2026-10-05T09:49:31Z @neo-fable-mnemosyne cross-referenced by PR #561
- 2026-10-09T01:40:29Z @neo-fable-mnemosyne cross-referenced by PR #613
- 2026-10-09T03:58:21Z @neo-fable-mnemosyne cross-referenced by PR #621
- 2026-10-09T21:54:01Z @neo-opus-vega cross-referenced by PR #960
- 2026-10-09T23:10:19Z @neo-gpt-sophie cross-referenced by PR #959
- 2026-10-10T14:07:27Z @neo-fable-clio cross-referenced by #658
- 2026-10-17T17:16:35Z @neo-fable-mnemosyne cross-referenced by #722
- 2026-10-17T18:43:00Z @neo-fable-mnemosyne cross-referenced by #727
- 2026-10-17T22:06:01Z @neo-fable-mnemosyne cross-referenced by PR #731
- 2026-10-17T22:40:28Z @neo-fable-clio cross-referenced by #724
### @neo-opus-grace - 2026-10-18T17:18:48Z

**Operator hold, 2026-10-18.** @tobiu: *"i would definitely delay any v1 FM testing that clio planned for tuesday and friday. while our current friction items might not all hit new operators, this is too much."*

So this walk (Fri 2026-10-23) and the Tuesday session are on hold. Today's install (Institution `2f9056a` / Brain `d6a905dc`) surfaced three start-path defects; each has a Brain fix in flight, and all three need one FM rebuild and install:

- No GPT seat starts: Fleet refuses its own Codex MCP rows after the runtime move (`FLEET_WORKSPACE_DIVERGENT`) — neomjs/neo-agent-brain#1136, PR neomjs/neo-agent-brain#1137.
- A seat stopped before a host reboot stays unstartable: boot recovery restamps it `exited` — neomjs/neo-agent-brain#1138.
- Neural Link is refused on every fresh Claude Desktop and Antigravity Start (`credential-missing, bridge-token`) — Julia's fix, ticket to follow.

The ACs that point at this walk (`#725`, `#726`, Brain `#1074` / `#1077` / `#1078`) keep pointing here; the date moves with the re-plan, which is the planner's once those fixes are installed.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 9c83bf6f-c435-4288-b11d-58eac7fa53df


- 2026-10-18T22:58:24Z @neo-opus-ada cross-referenced by #732
- 2026-10-25T16:28:24Z @neo-opus-ada cross-referenced by #812
- 2026-10-26T14:36:56Z @neo-fable-mnemosyne cross-referenced by #823

