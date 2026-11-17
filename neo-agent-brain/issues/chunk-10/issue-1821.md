---
id: 1821
title: 'Measure the ranking out of sample: new questions, a fresh index, a BM25 arm'
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-11-16T22:09:43Z'
updatedAt: '2026-11-17T00:16:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1821'
author: neo-fable-clio
commentsCount: 2
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
# Measure the ranking out of sample: new questions, a fresh index, a BM25 arm

## Context

The measurement of #1800 (`learn/agentos/measurements/kb-ranking-2026-11-16.md`) and the repair of #1806 in #1820 moved the shipped ranking from 26 to 33 of the balanced 40 (28 to 39 of 49). Part 3 of the memory series (neomjs/neo#21089, pull request neomjs/neo#21093) publishes those numbers, marks them as in-sample and says in public that the next measurement is owed. Five cold reads of that post named the same gaps: numbers chosen with the questions in view, an index three months stale on source, no proper lexical baseline, no codebase but ours.

One of those gaps closed on 16 Nov at 22:05Z with a drafting-seat check: **BM25 over the same files scores 34 of 40 and 42 of 49**, above ripgrep by distinct words (30, 36) and above the repaired ranking (33, 39). BM25 and the repaired ranking miss disjoint questions: either one finds all 40.

## The Problem

- No number exists for questions the ranking has never seen. The distinct-word order was picked after the summed order had been scored, and #1820 was verified on the fixture it was written against.
- 6 of the 40 fixture questions cannot be answered from the index at all (source and guide rows of 6 to 14 Aug; #419), so every comparison mixes ranking and freshness.
- The runner has no BM25 arm, so the committed report cannot say what the post now says.
- A textbook lexical ranker over the checkout matches the shipped ranking on this fixture. Whether lexical rescue inside `QueryService` should use it is a design question this ticket does not decide; it supplies the number.

## The Architectural Reality

- The runner `ai/scripts/diagnostics/measureKbRanking.mjs` takes `--fixture <path>`, `--all-49` and `--json-out <path>`; its fixture is `ai/scripts/diagnostics/fixtures/kbRankingQuestions.json`. Structure map (`npm run ai:structure-map -- --files --loc`, 16 Nov): both live under `ai/scripts/diagnostics/`, and a second fixture and a new arm belong there.
- The query log (`kb_query_log` through `list_agent_faqs`) listed 300 clusters on 16 Nov; 55 were picked for #1800. Many unused rows ask for history (no tracker for a checkout search) or concern repositories outside the two measured ones.
- #419 records the gate before the re-ingest (no profile manifest on the canonical plane) and owns the retirement after it. The first profile embed itself has no ticket of its own yet: asked on #419 (comment 6922366404), unanswered on 17 Nov.
- The check behind the 34: files from `rg --files` under the runner's exclusion globs in neo `459371e353` and neo-agent-brain `a2a621cb78` (7,690 files), tokens split on `[^a-z0-9_]+`, the runner's `extractGrepWords` and `STOPWORDS`, BM25 with k1 1.2 and b 0.75, one corpus over both checkouts. Its two control orders reproduce the committed report exactly: distinct words 30/40 (MRR 0.497) and 36/49 (0.499), summed count 10/40 (0.132) and 13/49 (0.152). BM25: 34/40 (MRR 0.571), 42/49 (0.566). By slice on the 40: indexed 34: 28; unindexed 6: 6; questions in words: 18 of 22; keyword queries: 16 of 18.

## The Fix

1. **A BM25 arm in the runner**, over the same files and words as the ripgrep arm, reported beside the two existing orders. The check above is the cross-check.
2. **A held-out fixture:** at least 20 new questions from the query log, chosen by a rule written down before any is scored (list order after the rows #1800 used; history questions and questions about other repositories skipped), answer files labelled by one seat without the rankers under test and verified by a seat of another family; the ranking code frozen at a named commit before the first run.
3. **Two runs, reported side by side:** on the index as it stands, and after #419's first profile embed. Each with the indexed and unindexed split, for the held-out set and for the first 49.
4. **The three reader scripts of the post as files** beside the runner (`rgrank`, `rgscore`, a BM25 ranker), so the post can link a file instead of a code block, and outside counts posted as issues are listed in the second report.
5. The second report as its own page under `learn/agentos/measurements/`, with its summary in `learn/agentos/KnowledgeBase.md`.

## Acceptance Criteria

- [ ] AC-1: The runner reports a BM25 arm, and on the committed fixture it reproduces 34 of 40 and 42 of 49 or the report says why not.
- [ ] AC-2: A held-out fixture of at least 20 verified questions is committed with its selection rule, its labeller and its verifier.
- [ ] AC-3: The second report states every arm on the held-out set at a frozen ranking commit, with the indexed and unindexed split.
- [ ] AC-4: The same report is repeated after the first profile embed of #419, or the report names that run as open with its owner.
- [ ] AC-5: The reader scripts exist as files, and any outside counts posted by then are listed in the report.

## Out of Scope

- Tuning weights on the held-out set: a set that weights are tuned on stops being held out.
- Replacing lexical rescue in `QueryService` with BM25, or a hybrid arm. The numbers come first.
- An agent-in-the-loop comparison (the same tasks with and without the index), and a run on a public codebase that is not ours. Both are the next evidence after this, and neither is this ticket.

## Related

#1800, #1806, #1820, #1807, #419, neomjs/neo#21089, neomjs/neo#21093

Decision Record impact: none.

unowned-rationale: open to a seat with the runner's context. Maryam built the runner and the fix and is the natural owner by her own choice; the BM25 check and the post's numbers are mine and I answer questions on them here. Item 3's second run is blocked by #419.

Live latest-open + closed sweep: latest 20 open (154 open, red band: no open ticket or epic owns the Knowledge Base's measurement) plus an open search (`measure ranking held-out re-ingest`) and a closed search (`knowledge base ranking measure held-out questions`) at 2026-11-16T21:59Z; the closed hit is #1800, whose Out of Scope does not exclude a second measurement; no equivalent. A2A: no claim on a re-measurement in the last 25 rows of the mailbox. Memory Core sweep (`knowledge base ranking measurement held-out questions BM25`): no prior claim. My open assignments here (#51, #53): none overlapping.

Origin Session ID: 76a8cf2d-14b8-426b-b8f1-9c3199c15c76

Retrieval Hint: "knowledge base ranking out of sample held-out fixture BM25 arm 34 of 40 fresh index"

## Timeline

- 2026-11-16T22:09:44Z @neo-fable-clio added the `enhancement` label
- 2026-11-16T22:09:45Z @neo-fable-clio added the `ai` label
- 2026-11-16T22:09:45Z @neo-fable-clio added the `testing` label
- 2026-11-16T22:09:45Z @neo-fable-clio added the `agent-os` label
### @neo-fable-clio - 2026-11-16T22:10:06Z

**The BM25 check behind the 34, as run on 16 Nov 2026 at 22:05Z.** It is a drafting-seat check for neomjs/neo#21093, not the arm this ticket asks for: it reads the fixture from `origin/dev`, lists files with `rg --files` under the runner's exclusion globs (through `RIPGREP_CONFIG_PATH`), and prints BM25 beside the runner's two orders as controls.

```text
RIPGREP_CONFIG_PATH=<file with the runner's exclusion globs> \
python3 bm25.py <neo at 459371e353> <neo-agent-brain at a2a621cb78> <a neo-agent-brain clone with origin/dev> [--all-49]

files 7690, mean length 1116 tokens, questions 40, query vocabulary 401
bm25      Hit@5 34/40  MRR 0.571
distinct  Hit@5 30/40  MRR 0.497
count     Hit@5 10/40  MRR 0.132
--all-49: bm25 42/49 (0.566) · distinct 36/49 (0.499) · count 13/49 (0.152)
```

```python
import json, math, os, re, subprocess, sys
from collections import Counter

neo_root, brain_root, brain_git = sys.argv[1:4]
all49 = '--all-49' in sys.argv

STOP = set('''about after all also an and any are as at be been before between both but by can could did do does each either for
from had has have how if in into is it its may more most must no nor not of on one only or other our out over same should so some
such than that the their them then there these they this those through to under up use used using was we were what when where
which while who why will with within without would'''.split())

fixture = json.loads(subprocess.run(['git', '-C', brain_git, 'show',
    'origin/dev:ai/scripts/diagnostics/fixtures/kbRankingQuestions.json'], capture_output=True, text=True, check=True).stdout)
questions = [q for q in fixture['questions'] if all49 or q['inBalanced40']]

def words(text):
    seen, out = set(), []
    for token in re.split(r'[^a-z0-9_]+', text.lower()):
        if len(token) >= 3 and token not in STOP and token not in seen:
            seen.add(token); out.append(token)
    return out

vocab = set(w for q in questions for w in words(q['question']))
split = re.compile(r'[^a-z0-9_]+')
docs  = []

for repo, root in (('neomjs/neo', neo_root), ('neomjs/neo-agent-brain', brain_root)):
    for rel in subprocess.run(['rg', '--files'], cwd=root, capture_output=True, text=True).stdout.splitlines():
        try:
            raw = open(os.path.join(root, rel), 'rb').read()
        except OSError:
            continue
        if b'\x00' in raw[:8192]:
            continue
        tokens = split.split(raw.decode('utf-8', 'ignore').lower())
        docs.append((repo, rel, len(tokens), Counter(t for t in tokens if t in vocab)))

N, avglen, df = len(docs), sum(d[2] for d in docs) / len(docs), Counter()
for d in docs:
    df.update(d[3].keys())

def rank(q, mode):
    ws, scored = words(q['question']), []
    for repo, rel, length, counts in docs:
        hit = [w for w in ws if counts.get(w)]
        if not hit:
            continue
        total = sum(counts[w] for w in hit)
        if mode == 'bm25':
            score = sum(math.log(1 + (N - df[w] + 0.5) / (df[w] + 0.5)) * counts[w] * 2.2 / (counts[w] + 1.2 * (0.25 + 0.75 * length / avglen)) for w in hit)
            key = (-score, f'{repo}/{rel}')
        elif mode == 'distinct':
            key = (-len(hit), -total, f'{repo}/{rel}')
        else:
            key = (-total, f'{repo}/{rel}')
        scored.append((key, f'{repo}/{rel}'))
    scored.sort()
    return [name for _, name in scored]

print(f'files {N}, mean length {avglen:.0f} tokens, questions {len(questions)}, query vocabulary {len(vocab)}')
for mode in ('bm25', 'distinct', 'count'):
    hits, rr = 0, 0.0
    for q in questions:
        answers = {f"{q['repo']}/{a['path']}" for a in q['answerFiles']}
        pos     = next((i + 1 for i, name in enumerate(rank(q, mode)) if name in answers), None)
        hits   += 1 if pos and pos <= 5 else 0
        rr     += 1 / pos if pos and pos <= 100 else 0
    print(f'{mode:9s} Hit@5 {hits}/{len(questions)}  MRR {rr / len(questions):.3f}')
```

The exclusion globs are the runner's `RIPGREP_EXCLUDE_GLOBS`, one `--glob=!…` line each. Per question on the 40, BM25 misses E002, E220, B004, B043, B086 and B138; the repaired ranking at `d3c614ffa8` misses E021, E132, B005, B023, B073, B116 and B117 (five of them are files the index has never seen).

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code · session 76a8cf2d-14b8-426b-b8f1-9c3199c15c76

- 2026-11-16T22:10:44Z @neo-fable-clio cross-referenced by #93
- 2026-11-16T22:11:03Z @neo-fable-clio cross-referenced by PR #94
- 2026-11-16T22:13:37Z @neo-fable-clio cross-referenced by #21089
- 2026-11-16T22:23:34Z @neo-fable-clio cross-referenced by PR #21093
### @neo-fable-clio - 2026-11-17T00:15:37Z

**Receipt: the second run of the committed runner, after #1820 (the "33 of 40")**

The fix's pull request states its result on all 49 questions (28 to 39). The balanced-40 number that neomjs/neo#21093 prints had no public receipt until now; a fact check of that post found the gap. This is the run, unchanged:

`node ai/scripts/diagnostics/measureKbRanking.mjs --neo-root <neo at 459371e353> --json-out <file>` in neo-agent-brain at `d3c614ffa8`, measured 2026-11-16T21:36:19Z, index 92,585 chunks, embeddings `text-embedding-qwen3-embedding-8b` through `openAiCompatible`.

| Slice | Shipped ranking | Nearest chunks | ripgrep, distinct words | ripgrep, summed count |
|---|---|---|---|---|
| Balanced 40 | 33 (MRR 0.692) | 15 (0.258) | 30 (0.472) | 10 (0.127) |
| 34 with an indexed answer file | 32 | 15 | 25 | 9 |
| 22 of those with every answer file indexed | 22 | 9 | 16 | 6 |
| 6 with no indexed answer file | 1 | 0 | 5 | 1 |
| 22 asked as a question | 19 | 6 | 13 | 5 |
| 18 keyword queries | 14 | 9 | 17 | 5 |
| All 49 (`--all-49`, 21:40:49Z) | 39 | 19 | 36 | 13 |

The shipped ranking's seven misses on the 40: E021, E132, B005, B023, B073, B116, B117. Five of them (E021, E132, B023, B073, B117) are among the six questions without an indexed answer file. Its one hit there is E032, and it comes from the lexical rescue reading the checkout (`learn/agentos/DockZoneModel.md`, `indexed: false`), not from the index.

Two things for whoever takes this ticket:

- The index grew from 92,257 chunks at the first report to 92,585 at this run through `github-content-sync`; the three baseline rows did not move.
- The vector arm's first-five slots on the 40: 115 of 200 are issue, pull request or discussion rows (74, 35, 6). I had printed 122 from a loose path filter; the post is corrected.

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code · session 51030985-fe65-4115-aa2a-731ec56e6e87


