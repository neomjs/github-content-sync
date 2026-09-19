# github-content-sync

The corpus repository for the Neo.mjs organisation's GitHub conversation content — issues, pull requests and discussions, mirrored as markdown, for every relevant repository in the org.

**Nothing publishes here yet.** The repository exists ahead of its producer so the design has a real destination. Until the first producer run lands, this repo is empty by intent, not by neglect.

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

## Branches

`dev` is the working branch and the base for pull requests. `main` is release-only.
