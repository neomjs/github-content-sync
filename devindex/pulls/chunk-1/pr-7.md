---
number: 7
title: Consume the agent skill substrate as an npm dependency
author: neo-opus-grace
state: CLOSED
createdAt: '2026-08-26T10:51:41Z'
updatedAt: '2026-08-26T10:53:24Z'
closedAt: '2026-08-26T10:53:24Z'
mergedAt: null
head: feat/consume-agent-skills
base: main
url: 'https://github.com/neomjs/devindex/pull/7'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Refs neomjs/neo#17798

`devindex` consumes the canonical agent skill substrate. Skills arrive as **`neo-agent-skills@^0.1.1`** and are projected by that package's postinstall linker into two **untracked** surfaces. No skill bytes enter this repository's git.

This repo was the original evidence for why the canonical store exists: it carried a hand-copied `AGENTS.md` differing from canonical in 8 hunks — two of them semantic — and **no `.agents/skills` at all**. Not behind; forked in one place and empty in another, with nothing reporting either.

## What lands

| path | what |
|---|---|
| `package.json` | `neo-agent-skills@^0.1.1` in devDependencies; the materializer **appended** to the existing postinstall chain |
| `package-lock.json` | the pin `npm ci` resolves from |
| `.gitignore` | `.agents/skills` and `.claude/skills/` — projection, never committed |
| `.github/workflows/substrate-sync.yml` | asserts the projection actually happened |

**The postinstall is appended, not replaced.** `linkMarkedForWorkerScope` and `pullDevIndexData` still run first; the materializer is chained after them. Replacing that entry would have broken this repo's build, and it is the kind of thing a template-shaped change gets wrong by default.

Two surfaces, deliberately different shapes: `.agents/skills` is **one directory symlink** (harness-neutral discovery — Codex, Antigravity, any fork), `.claude/skills` is **per-skill links** (the manifest-declared Claude façade). A skill can be opted out of the façade and cannot be opted out of a directory symlink.

## Verification

The materializer was exercised directly rather than through a full install, so this repo's data-pull step was not triggered:

```
neo-agent-skills-materialize  → .agents/skills → ../node_modules/neo-agent-skills/.agents/skills
                                .claude/skills → 37 links
--check                       → exit 0
git ls-files                  → 0 skill bytes tracked
```

## Not in scope

**`AGENTS.md` is untouched.** Its divergence from canonical is real and is a different ticket: the committed constitution is the *contributor* surface and keeps separate custody from the maintainer constitution, which projects into seat substrate rather than into a repo's tree.

Corpus rules — budgets, manifest coherence, reference integrity — do not run here. They run once, in `neomjs/neo-agent-skills`. This repo runs exactly one check: did the projection happen.

Authored by Grace (Claude Opus 5, Claude Code). Session f27af939-3cec-4f52-a67d-e4e8786fed08.
