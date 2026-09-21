# github-content-sync

The corpus repository for the Neo.mjs organisation's GitHub conversation content — issues, pull requests and discussions, mirrored as markdown, for every relevant repository in the org.

**Configured origins:** `neo`, `neo-agent-brain`, `neo-agent-institution`, `neo-agent-skills`, `devindex`. Each gets one `<repoSlug>/` root **once its first clean emission publishes**; until then it is configured, not present. `neo` has published on the six-hourly schedule since 2026-09-20; the other four appear with the first successful run after they joined the list. The set is the `ORIGINS` list in the workflow; adding a repository is adding one token there, and its tree is created by its first emission. The emitter ships with the Brain ([`neomjs/neo-agent-brain#387`](https://github.com/neomjs/neo-agent-brain/issues/387)); a runtime pin without its corpus mode is refused at the *Verify the pinned runtime supports corpus mode* step, **before** the emitter is invoked, because a runtime without the mode would silently run a full manual sync rather than refusing.

## Why it exists

The data-sync pipeline worked while `neomjs/neo` was effectively a monorepo. It no longer is: content from the Brain, `devindex` and the Fleet Manager has no home in a single repository's tree, and mirroring it into `neomjs/neo` puts churn and noise into that repository's history and statistics.

Extracting the corpus into its own repository lets every other org repository pull the slice it needs as a git-ignored working tree, the way the skills package is already consumed.

## Design

- **[Discussion #17846](https://github.com/neomjs/neo/discussions/17846)** — the design, graduated 2026-09-19 for a producer-only first delivery.
- **[#17416](https://github.com/neomjs/neo/issues/17416)** — the epic.
- **[#18997](https://github.com/neomjs/neo/issues/18997)** — the ADR 0004 amendment that writes down the index contract this repository's layout depends on.

Two decisions from that design govern what lands here:

1. **Content is origin-qualified.** Logical identity is the tuple `(repoSlug, type, id)`, and files land under a `<repoSlug>/` root. Two repositories' issue number 1 are different objects and must survive write, read, update and removal independently.
2. **Publication is owned by this repository.** The job that writes here runs from this repo against a pinned runtime, and publishes only into this repo.

## How publication works

`.github/workflows/publish-corpus.yml` runs on a schedule and on demand. It checks the Brain out at a pinned commit, installs that commit's own lockfile, runs the emitter against this checkout **once per origin, in order**, and commits **only if every invocation exits 0**.

The contract in one line: **the emitter's exit code is the whole verdict, per origin.** Zero means all four outcomes advanced — release reference, issues, discussions, pulls. Everything else — a refusal, a held lease, a partial facet failure that left real files on disk — is nonzero, ends the run before the next origin, and publishes nothing. No step reads the emitter's output to decide, because a publisher that parses prose eventually publishes on a message it misread.

When a run does publish, the content tree and `_index.json` land in **one commit**. They are a single publication unit; a revision carrying one without the other is a corpus whose index disagrees with its files.

The pushed commit always sits directly on the current `dev` tip. If `dev` moved while the emitter ran — a reviewed merge landing mid-run — the publication commit is replayed onto the new tip (a cherry-pick; it touches only corpus paths, which nothing else writes) and the parent check runs again before the push. A conflict on a corpus path means a second writer, and the run refuses instead.

## Maintaining the runtime pin

`brain-runtime.json` holds the Brain commit this repository runs. **It is a maintained artifact, not a constant** — pinning a runtime means this repository carries a pin with the same staleness failure mode `neo-agent-institution` has with its engine pin. One file, so the question *"is this pin stale?"* has one place to look.

**To bump it:**

1. Pick the Brain commit you want and confirm it is a **full 40-character SHA** on `dev`. The workflow refuses anything else — a branch name here would make every scheduled run an untested upgrade of the thing doing the publishing.
2. Update `sha` and `pinnedAt`, and say in `note` what the bump is for.
3. Open a PR. A bump changes the code that writes this repository's contents, so it gets a review like any other.

**To tell whether it is stale:**

```bash
jq -r '.sha + "  pinned " + .pinnedAt' brain-runtime.json
gh api repos/neomjs/neo-agent-brain/commits/dev --jq '.sha + "  " + .commit.committer.date'
gh api "repos/neomjs/neo-agent-brain/compare/$(jq -r .sha brain-runtime.json)...dev" --jq '{behind: .behind_by, ahead: .ahead_by}'
```

Distance alone is not a reason to bump — a pin that publishes correctly is doing its job. Bump when the Brain ships something this job needs, or when the gap grows large enough that the eventual bump stops being reviewable.

## Branches

`dev` is the working branch and the base for pull requests. `main` is release-only.
