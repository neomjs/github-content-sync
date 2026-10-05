---
id: 533
title: 'A non-live banner shows its reason beside the pill, not only on hover'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-10-04T11:01:20Z'
updatedAt: '2026-10-05T09:37:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/533'
author: neo-opus-ada
commentsCount: 0
parentIssue: 424
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T09:37:49Z'
milestone: FM v1
---
# A non-live banner shows its reason beside the pill, not only on hover

## Context

Row 5's denominator sitting found that the cockpit bar's banner hides every non-live reason behind a hover ([Mnemosyne's read](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-5978785003), line 6; [steward disposition](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-5978806870)). All six row-5 provocations resolve to a `PLANE_REFUSALS` entry. A stranger sees a two-word pill (`pat refused`) and Connect; the sentence that says why exists only in `title` and `aria-label`.

#516 asks, for each provocation, whether the reason and the next step were readable without hovering. On `dev@4c65d45` the source already answers that, so by #424's own rule this is a leaf before the walk, not after it. I asked for the design call on 10-03 ([5971630164](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-5971630164)); Clio decided it on 10-04 (quoted below).

## The Problem

`SpineBanner.deriveSpineBanner` builds every verdict through `verdict(kind, text, title, action)`, which returns `{text, title, ariaLabel: title}`. The visible pill binds `text`, the status word; only `title` carries the sentence. For a refusal, that sentence is the product's own words (`PLANE_REFUSALS[cause].lead`, for example "… Connect again with a current one."). A boot refusal also appends the fleet child's last line as ` · <reason>`. Nothing renders the sentence visibly, so:
- the row 2 gate ("each with its reason and a next step") fails on the surface an operator reads first;
- at a narrow container (≤ 760 px) the pill itself drops to a dot, and the whole verdict becomes hover-only;
- no golden and no e2e spec contains any of the four refusal sentences ([Mnemosyne's tier-one read](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-5971601290)), so the words have no reviewable frame.

## The Architectural Reality

- `apps/agentos/util/SpineBanner.mjs`: `verdict()` inside `deriveSpineBanner`; every non-live branch of `deriveSpineBanner`; `PLANE_REFUSALS`, the four `{lead, text}` pairs from #425 and #446, whose leads reuse the connect card's `PlaneVerdict` sentences. Boot refusals append `daemon.reason`, which is developer text (config leaves, an error class). Runtime refusals pass the lead alone.
- `apps/agentos/view/fleet/cockpit/StateProvider.mjs`: the `spineBanner` derived data (`deriveBannerVerdict`).
- `apps/agentos/view/fleet/cockpit/SpineBannerComponent.mjs`: reactive `bannerTitle_` and `bannerAriaLabel_`, each mirrored onto its own attribute. It uses `text`, never `html`, because the sentence interpolates retained transport strings and an innerHTML sink would execute hostile markup.
- `apps/agentos/view/fleet/cockpit/Container.mjs`: the `fm-bar-state` row (banner + wake telltale) and the action button bound to the same verdict (Connect for `connect-plane`, otherwise Reconnect).
- Specs: `test/playwright/unit/apps/agentos/view/fleet/cockpit/spineBanner.spec.mjs` and `spineBannerPipeline.spec.mjs`.
- **Design authority, the record this changes:** `apps/agentos/design/institution-header-detail-ia.html` (2026-08-30). The state verdict is "the pill text — a status word, never a sentence"; endpoint and consequence are "the pill's title (T5 pattern, one hover away)"; narrow widths are "dots with titles — full truth stays one hover away". The component's JSDoc restates it: chrome labels are never sentences.
- **Design authority, the decision:** Clio, design lead and row 2 steward, 2026-10-04, to the row 5 steward:
  > Pill = the state word only (`refused` / `unreachable` / …), as today. Lead = one visible line beside the pill: the plane's refusal reason in its own words, truncated at one line, never a code; it appears whenever the state is not `live`. Action = Connect, the single control (Connect, not Reconnect, per #446's rule). `title` / `ariaLabel` may REPEAT the lead for assistive tech; nothing lives only there. A refusal a user must hover to read is a refusal the product hid. Detail → Connection row carries the full reason text + the next step spelled out (two densities, CARD-CONTRACT).

  It keeps the pill's label-language law. What it replaces is "one hover away" for the reason.

## The Fix

1. **The verdict gains a `lead`.** `verdict()` returns `{text, lead, title, ariaLabel, action, …}`. `lead` is the verdict's one product sentence, never a transport string, error class or config leaf. For the four refusals it is `PLANE_REFUSALS[cause].lead`; for every other non-live branch it is the sentence before its ` · <reason>` suffix. `title` stays the full sentence, retained reason included, for hover and the screen reader.
2. **The banner renders the lead visibly.** *(Shipped as its own bar component beside the pill, reference `fleet-spine-banner-lead`: at 760 px and below the pill's font size drops to 0, which would take a child lead with it.)* `SpineBannerComponent` gains a reactive `lead_`, rendered as a text child beside the status word (text, never html) and bound in `Container.mjs` from `spineBanner.lead`. It is one line, with an ellipsis on overflow, never wrapping, and it hides with the pill when the spine is live.
3. **Narrow widths.** The IA's collapse order gains the lead: it keeps its one truncated line at every width, so no reason is hover-only. Clio confirms the exact narrow form at her capture read before the PR.
4. **The detail density.** The System view's connection line (`view/system/Container.mjs` `applyConnection`, "fleet read <state> · <reason>") already carries the full reason; it gains the next step for each state it shows. If Clio names another surface at the capture read, this item follows her call.
5. **Captures.** One visual fixture frame per refusal (`plane unreachable`, `pat refused`, `account changed`, `not a plane`): #424 line 7, folded in here. The words get a reviewable frame, and #516's walk gets a reference.
6. **The design record follows.** The same PR amends `institution-header-detail-ia.html`'s fact table and narrow-width rule, and the component JSDoc, to the decision.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `SpineBanner.deriveSpineBanner` verdict, new `lead` | Clio's 10-04 decision (above) | one product sentence for every non-live verdict; never a transport string | `null` while live (the banner is hidden) | `verdict()` JSDoc | `spineBanner.spec.mjs`, one arm per branch |
| the bar's lead component (`fleet-spine-banner-lead`, beside `SpineBannerComponent`) | the component's bind contract | renders the lead visibly, as text | empty lead → nothing rendered | class JSDoc | `spineBannerPipeline.spec.mjs` + the four captures |
| `title` / `aria-label` | as today | the full sentence, retained reason included | unchanged | class JSDoc | existing arms stay green |
| System view connection line | `applyConnection` | names the next step per state shown | unchanged with no observation | method JSDoc | unit arm |
| `institution-header-detail-ia.html` | the design SSOT | fact table + narrow-width rule match the decision | — | the file | the diff, Clio's read |

## Acceptance Criteria

- [ ] AC-1: every non-live verdict of `deriveSpineBanner` carries a `lead`: one product sentence with no transport string, error class or config leaf. The four refusals carry their `PLANE_REFUSALS` leads (unit, one arm per branch).
- [ ] AC-2: the banner renders the lead visibly beside the status word, as text, on one line with an ellipsis, and hides it with the pill while the spine is live. `title` and `aria-label` still carry the full sentence (unit + pipeline spec).
- [ ] AC-3: at the narrow container width the lead still shows its truncated line; no reason is reachable only by hovering (pipeline spec at both widths).
- [ ] AC-4: the System view's connection line names the next step for each state it shows (unit).
- [ ] AC-5: four visual fixture frames, one per refusal, read by Clio before the PR (her capture read).
- [ ] AC-6: the IA sketch and the component JSDoc state the new rule; no doc still places the reason "one hover away".

## Post-Merge Validation

*(Moved here from the AC list on 2026-10-04 by the author: it is not this PR's gate.)*

- [ ] #516's walk records "readable without hovering" for each provocation, on the installed candidate that carries this leaf (cut A).

## Out of Scope

- The words of the verdicts other than the four refusals. Row 2 owns each surface's words; this leaf shows them, it does not rewrite them.
- The two cold frames, `cockpit-cold` and `home-returning-cold` (row 2's gap list on #477).
- The action rule (Connect vs Reconnect, #446): unchanged.
- The wake telltale pill and its own reason channel.

## Avoided Traps

- **Rendering `title` visibly.** For a boot refusal, `title` ends in the fleet child's developer line. The visible lead is the product sentence alone.
- **`html` for the lead.** The verdict interpolates retained transport strings; `text` only, the component's existing rule.
- **A second banner, a modal or a toast.** One pill vocabulary and one lead line. Recovery stays inline (the ROADMAP's first-run rule).
- **New copy next to the card's.** The refusal leads already are the connect card's `PlaneVerdict` sentences (#425); reuse them, never re-word them.

## Related

Parent: #424 (row 5). Serves #477 (row 2's banner states) and #516 (the walk). Builds on #425 and #446 (the refusal vocabulary and its action). #522 holds the same CARD-CONTRACT two densities. Design SSOT: `apps/agentos/design/institution-header-detail-ia.html`.

Decision Record impact: `none`. No ADR governs the chrome label law; the governing record is the design SSOT above, which this leaf amends.

Structure map: N/A (Institution cockpit view and util, no `ai/` placement).

Size: M (row 5's forecast).

Sweeps (2026-10-04):
- Live latest-open: the latest 20 open Institution issues at 11:00:33Z. Newest is #532; no banner-lead equivalent.
- A2A in-flight: the last 30 messages in all read states; no claim on the banner's lead.
- Memory Core: "cockpit banner refusal reason only visible on hover, title attribute, status pill shows two words", 5 results; no decision beyond the 08-30 IA sketch.
- Own assignments: #424, #512, #516, #521, #522, #523; none on the banner.
- Exact: `gh search issues --owner neomjs --state open` for "banner hover reason" and "pill title reason visible", none. Mnemosyne's 10-04 searches (`SpineBanner`, `refusal tooltip banner`, `banner reason visible`) also found none.

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "spine banner refusal reason visible lead beside pill hover-only title PLANE_REFUSALS row 5 line 6"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-04T11:01:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-04T11:01:22Z @neo-opus-ada added the `enhancement` label
- 2026-10-04T11:01:22Z @neo-opus-ada added the `agent-os` label
- 2026-10-04T11:01:23Z @neo-opus-ada added the `ai` label
- 2026-10-04T11:01:23Z @neo-opus-ada added the `design` label
- 2026-10-04T11:01:29Z @neo-opus-ada added parent issue #424
- 2026-10-04T11:01:30Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-04T11:10:05Z @neo-fable cross-referenced by #534
- 2026-10-04T11:47:12Z @neo-opus-ada cross-referenced by #424
- 2026-10-04T13:00:34Z @neo-opus-ada referenced in commit `6c31155` - "feat(agentos): a degraded wake keeps its word in the mark regime, so no non-live state is a bare mark (#533)

Clio's capture read (2026-10-04): at 760 px and below the fleet pill may drop to its mark because its lead carries the reason, but a degraded wake chip has no lead, so a bare mark made its state hover-only. Only a live wake now drops to its mark. The 720, 800 and 520 goldens show the word; the refusal arm asserts it."
- 2026-10-04T13:01:55Z @neo-opus-ada cross-referenced by PR #542
- 2026-10-04T13:04:32Z @neo-opus-ada referenced in commit `0955297` - "test(visual): the Golden Path pane golden shows the merged layout, at its author's request (#533)

#513 shipped the Golden Path pane's facts-first layout and re-stamped the visual baselines, but did not re-capture pane-golden-path.png, so the local visual suite failed it on clean dev (2 of 2 runs, 5026 px). Vega confirmed the frame is #513's approved layout and asked for the re-capture here. The full visual suite is now green."
- 2026-10-04T13:09:39Z @neo-opus-ada referenced in commit `0fb9d48` - "chore(merge): bring dev into the banner lead and re-stamp the visual baselines (#533)

dev gained #537, #539 (the Brain pin) and #528. The only conflict was the visual baseline stamp, which is re-stamped from the merged inputs. Visual 34/34, unit 1,380, components 8 and e2e 13 pass on the merged tree."
- 2026-10-04T13:43:13Z @neo-opus-vega cross-referenced by #544
- 2026-10-04T13:49:53Z @neo-opus-ada referenced in commit `b80050c` - "docs(agentos): the header IA records the chrome as shipped — no view buttons in the cockpit bar, the product bar's title and setup line, the detail's tabs and ledger (#533)

The operator found the design SSOT outdated: its cockpit-bar mocks, tier table and three rules still carried the Overview/Focus/Review view buttons, which left the bar on 2026-09-12 (#133) when perspectives moved to their drawer. The page now records what ships:
- the product bar's title (#107) and first-run setup line (#441);
- the cockpit bar as ControlToolbar, state and actions only;
- a healthy narrow bar of one quiet wake mark and the start icon;
- the detail's session/status/wake/capacity/runtime/repository/roster ledger over Status and Configuration tabs;
- the implementation coordinates as shipped.
A dated current-state note keeps the 08-30 captures as the problem the page answered."
- 2026-10-04T14:44:45Z @neo-gpt cross-referenced by #477
- 2026-10-04T18:17:15Z @neo-opus-ada cross-referenced by #554
- 2026-10-04T18:43:29Z @neo-opus-ada referenced in commit `4e06b79` - "chore(merge): bring dev into the banner lead and re-stamp the visual baselines (#533)"
- 2026-10-04T20:22:23Z @neo-gpt cross-referenced by PR #560
- 2026-10-05T09:37:49Z @tobiu referenced in commit `d6748e5` - "feat(agentos): a non-live banner shows its reason beside the pill, not only on hover (#533) (#542)

* feat(agentos): a non-live banner shows its reason beside the pill, not only on hover (#533)

Every non-live spine verdict carries a lead: its product sentence, with no endpoint, error class or config leaf. The bar shows the lead beside the pill on one line, which ellipsizes. Title and aria still carry the whole sentence, word for word as before.

- The lead is the row's slack (zero flex basis). The bar's buttons keep their width (itemDefaults flex none), so a narrow bar shortens the lead before any pill or button loses a pixel. Below 760 px the pills drop to marks and the lead keeps its line.
- The control bar moves into its own class, ControlToolbar, so the cockpit stays under the app-file bar. The component tree is unchanged.
- The System view's connection line names the next step for each state.
- Four refusal frames, a light frame and a 720 frame join the visual suite through a Brain-health landing. The cold, 720 and 520 goldens are refreshed for the lead.
- The IA sketch, the banner JSDoc and SCSS, and the census state the rule.

* feat(agentos): a degraded wake keeps its word in the mark regime, so no non-live state is a bare mark (#533)

Clio's capture read (2026-10-04): at 760 px and below the fleet pill may drop to its mark because its lead carries the reason, but a degraded wake chip has no lead, so a bare mark made its state hover-only. Only a live wake now drops to its mark. The 720, 800 and 520 goldens show the word; the refusal arm asserts it.

* test(visual): the Golden Path pane golden shows the merged layout, at its author's request (#533)

#513 shipped the Golden Path pane's facts-first layout and re-stamped the visual baselines, but did not re-capture pane-golden-path.png, so the local visual suite failed it on clean dev (2 of 2 runs, 5026 px). Vega confirmed the frame is #513's approved layout and asked for the re-capture here. The full visual suite is now green.

* docs(agentos): the header IA records the chrome as shipped — no view buttons in the cockpit bar, the product bar's title and setup line, the detail's tabs and ledger (#533)

The operator found the design SSOT outdated: its cockpit-bar mocks, tier table and three rules still carried the Overview/Focus/Review view buttons, which left the bar on 2026-09-12 (#133) when perspectives moved to their drawer. The page now records what ships:
- the product bar's title (#107) and first-run setup line (#441);
- the cockpit bar as ControlToolbar, state and actions only;
- a healthy narrow bar of one quiet wake mark and the start icon;
- the detail's session/status/wake/capacity/runtime/repository/roster ledger over Status and Configuration tabs;
- the implementation coordinates as shipped.
A dated current-state note keeps the 08-30 captures as the problem the page answered."
- 2026-10-05T09:37:49Z @tobiu closed this issue
- 2026-10-05T12:28:13Z @neo-opus-ada cross-referenced by #516

