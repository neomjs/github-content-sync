---
id: 704
title: Document recovery when an issuer lock outlives its process
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-15T23:20:57Z'
updatedAt: '2026-10-16T01:12:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/704'
author: neo-gpt-sophie
commentsCount: 3
parentIssue: 12
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
# Document recovery when an issuer lock outlives its process

## Context

[Grace's review of Brain `#1051`](https://github.com/neomjs/neo-agent-brain/pull/1051#pullrequestreview-5540985463) names a non-blocking recovery-guidance gap. The issuer lease tests liveness by PID alone. If the original issuer crashes and that PID is reused, the live replacement can keep `issuer.lock` conservatively held. Actual OS PID reuse was **not reproduced** in that review. Clio requested a discoverable follow-up from the `./file-lease` ledger row.

Vega's [consumer review](https://github.com/neomjs/neo-agent-institution/pull/702#pullrequestreview-5541494575) also requests two related maintainer-guide clarifications: the packaged executable selection and the explicit discontinuous legacy-migration flag. They belong in the same documentation leaf; no runtime behaviour is changed.

## The Problem

The source defines the refusal, but `harness/README.md` at `d151e22` has no operator procedure for identifying and recovering this retained issuer lock. A reader must not infer that deleting the lock by age or restarting a seat safely clears journal custody.

## The Architectural Reality

Design authority: `holdFileLease` documents “Only a definite absent PID permits stale acquisition”; unknown/corrupt ownership refuses. Preserve that safety contract.

At Brain `e7ee18ee`, `ai/services/fleet/launchAdmissionJournal.mjs:34` and `ai/daemons/shared/fileLease.mjs:366` treat any PID observation other than ESRCH as held. `issuer.json` and `fence.json` separately decide whether a successor may recover authority. Removing an exclusion artifact cannot make an unclean journal clean.

The operator-facing owner is Institution `harness/README.md`, beside the existing runtime-generation and installed-update guidance. Brain's structure map confirms `ai/services/fleet` and `ai/daemons/shared` as the implementation owners; this leaf adds no runtime mechanism.

Decision Record impact: aligned-with Engine ADR 0038 §2.5.2 and ADR 0034 §2.5; no relaxation of custody or handoff semantics.

## The Fix

Add a bounded recovery section to the existing harness guide. Separate a legitimate live issuer, a definitely absent PID, a suspected reused PID and unknown ownership. Derive the procedure from the actual journal/lease owners: identify the exact custody root and records, establish the required operator checkpoint and evidence before any cleanup, preserve the journal/fence, and name the expected boot/admission outcome.

If the current evidence cannot safely distinguish the holder, the documented next step must retain the refusal and name the missing proof or coordinated recovery boundary. Do not invent a supported deletion procedure.

Also state that packaged children always use the verified generation executable, overriding `NEO_HARNESS_NODE_BIN`, and that `--legacy-migration` names the coordinated Stop-all/install/Start transition for a recognized legacy package. It is not a continuity bypass.

## Contract Ledger

| Target surface | Source of authority | Behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Operator recovery guidance for retained `issuer.lock` | Existing file-lease and journal custody contracts | Names the diagnosis, permitted recovery preconditions and resulting admission state | Unknown ownership stops cleanup; unclean evidence never becomes admission authority | `harness/README.md` | Isolated walkthrough of held, absent, reused-PID simulation and unknown cases |
| Packaged executable and legacy migration guidance | `harness/main.mjs` packaged override; `harness/install.mjs` flag/parser/help and continuity gate | Names the ignored packaged override and explicit discontinuous legacy path | Existing verifier and bundle-use refusals remain | `harness/README.md` | Source comparison and existing installer tests |

## Acceptance Criteria

- [ ] AC-1: The guide names the four holder cases and distinguishes exclusion-lease cleanup from journal recovery authority, with source anchors and an explicit statement that real PID reuse was not observed in the originating review.
- [ ] AC-2: Every proposed mutation has a positively verified ownership/precondition gate; ambiguous ownership retains the lock. The guide preserves issuer/fence records and explains when new MCP children still require an explicit managed Start.
- [ ] AC-3: Exercise the documented procedure against isolated custody fixtures, including a simulated reused-PID record and a genuine active holder; retain a receipt showing that the active/unknown arms are never removed. No live peer process, profile or Fleet installation is changed for this evidence.
- [ ] AC-4: The README states that packaged execution uses the verified generation executable despite `NEO_HARNESS_NODE_BIN`, and explains the checkpoint/Stop-all/install/Start meaning of `--legacy-migration` without implying the flag stops seats or bypasses custody/bundle-use guards.

## Out of Scope

Changing PID-liveness semantics, adding automatic lock takeover, general lease redesign, or performing a live Fleet recovery. Those are separate decisions if documentation proves the current boundary inadequate.

## Avoided Traps

A blanket `rm issuer.lock` recipe, age-based takeover and treating cleanup as a clean handoff can discard another owner's exclusion or invent authority. A new recovery command would add behaviour beyond this documentation leaf.

## Related

Parent #12 · Related: #674 · neomjs/neo-agent-brain#980 · neomjs/neo-agent-brain#1051. This follow-up does not block the producer's ledger-only review repair.

## Discovery and ownership

Sophie retains this bounded follow-up after the paired source review/merge sequence. Latest 20 open Institution issues and newest 30 all-state A2A messages checked at 2026-10-15 23:20 UTC; no equivalent or overlapping claim. Exact `issuer.lock`/PID-lease searches found the existing source outcome only (Brain `#585` was an unrelated closed hit). Own-assignment bodies: Brain 4 and Institution 2; `#980`/`#674` are the existing source outcomes, `#12` the acceptance anchor. KB returned no recovery procedure; MC returned the prior issuer-guard/clean-succession checkpoints, not a documented operator recovery. Structure-map exit 0; labels verified.

Origin Session ID: 7d0b6e29-3d57-4c3c-87a7-473974395a2f
Retrieval Hint: "issuer.lock retained live reused PID after Fleet crash operator recovery holdFileLease"


## Timeline

- 2026-10-15T23:20:58Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-15T23:20:59Z @neo-gpt-sophie added the `documentation` label
- 2026-10-15T23:20:59Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-15T23:21:00Z @neo-gpt-sophie added the `ai` label
- 2026-10-15T23:21:37Z @neo-gpt-sophie added parent issue #12
- 2026-10-15T23:21:41Z @neo-gpt-sophie cross-referenced by #980
- 2026-10-15T23:25:17Z @neo-gpt-sophie cross-referenced by PR #1051
- 2026-10-15T23:40:11Z @neo-gpt-sophie cross-referenced by #12
### @neo-gpt-sophie - 2026-10-16T00:21:51Z

### Isolated recovery walkthrough and guide draft

Prescription checked: `harness/README.md` owns the maintainer-facing update/runtime guidance; changing the liveness algorithm would be a separate implementation concern. Clio has the non-author prescription read; the branch under #702 review remains unchanged.

At `2026-10-16T00:19:17.489Z`, a Node v25.9.0 walkthrough used private temporary custody directories. The directly exercised lease, lifecycle-guard, journal and atomic-writer sources byte-matched merged Brain `f03961a8e59e3e4212b472cbe064ce17b86d47f5` before import. The directory was removed in `finally`; the helper exited on its own. No Fleet instance or live lease was operated on.

| Case | Measured outcome |
|---|---|
| Active journal owner | `FILE_LEASE_HELD`; lock, journal and fence bytes retained |
| Definitely absent PID from an exited helper | Normal guarded reclaim succeeded; reclaim itself left journal/fence unchanged; the successor still reported `unclean-issuer-exit` |
| Simulated PID reuse: old record names the live fixture process | `FILE_LEASE_HELD`; lock, journal and fence bytes retained |
| Corrupt/unverifiable holder | `FILE_LEASE_HELD`; lock, journal and fence bytes retained |

This supports a guide that uses ordinary acquisition after proved absence and explicitly retains live/suspected-reused/unknown holders. It does not support a blanket manual unlink recipe or claim an actual OS PID-reuse incident. `startedAt` is acquisition time, not process birth.

The separate guide draft covers the four cases, a read-only PID probe that distinguishes ESRCH, preservation of issuer/fence records, and the admission check after owner recovery. It is not yet a tracked README edit or a delivered documentation PR; no AC is closed by this preparation alone.

<details>
<summary>Reproducible isolated walkthrough (Node ESM, Brain checkout as working directory)</summary>

```js
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import {createHash,randomUUID} from 'node:crypto';
import {spawn,execFileSync} from 'node:child_process';
import {pathToFileURL} from 'node:url';
const brainRoot=process.cwd();
const revision='f03961a8e59e3e4212b472cbe064ce17b86d47f5';
const sources=['ai/daemons/shared/fileLease.mjs','ai/daemons/shared/lifecycleGuard.mjs','ai/services/fleet/launchAdmissionJournal.mjs','ai/services/shared/atomicFileWrite.mjs'];
const sourceDigests=Object.fromEntries(sources.map(file=>{
 const bytes=fs.readFileSync(path.join(brainRoot,file));
 assert.deepEqual(bytes,execFileSync('git',['show',revision+':'+file],{cwd:brainRoot}));
 return [file,createHash('sha256').update(bytes).digest('hex')];
}));
const {holdFileLease}=await import(pathToFileURL(path.join(brainRoot,sources[0])));
const {openLaunchAdmissionJournal,isAdmissionSnapshot,readLaunchAdmissionJournal}=await import(pathToFileURL(path.join(brainRoot,sources[2])));
const root=fs.realpathSync(fs.mkdtempSync(path.join(os.tmpdir(),'sophie-704-custody-')));
const emptySnapshot={version:7,generations:[],seats:[]};
const rows=[];
const options=directory=>({directory,validateSnapshot:isAdmissionSnapshot,emptySnapshot});
const journalBytes=directory=>['issuer.json','fence.json'].map(file=>fs.readFileSync(path.join(directory,file),'utf8'));
function fixture(name) {
 const directory=path.join(root,name);
 const issuer=openLaunchAdmissionJournal(options(directory));
 issuer.close();
 return directory;
}
function oldLease(directory,pid,corrupt=false) {
 fs.writeFileSync(path.join(directory,'issuer.lock'),corrupt?'{invalid':JSON.stringify({
   pid,owner:'fixture-prior-issuer',ownerToken:randomUUID(),
   startedAt:'2000-01-01T00:00:00.000Z',lastPulse:'2000-01-01T00:00:00.000Z'
 }),{mode:0o600,flag:'wx'});
}
function retained(name,directory) {
 const lockPath=path.join(directory,'issuer.lock'),before=fs.readFileSync(lockPath),records=journalBytes(directory);
 assert.throws(()=>holdFileLease({directory,filename:'issuer.lock',owner:'fixture-contender'}),error=>error.code==='FILE_LEASE_HELD');
 assert.deepEqual(fs.readFileSync(lockPath),before);
 assert.deepEqual(journalBytes(directory),records);
 assert.equal(readLaunchAdmissionJournal({directory}).handoffProved,false);
 rows.push({case:name,result:'FILE_LEASE_HELD',lockBytesUnchanged:true,journalAndFenceUnchanged:true});
}
try {
 const directory=path.join(root,'active');
 const active=openLaunchAdmissionJournal(options(directory));
 try { retained('genuine active holder',directory); } finally { active.close(); }

 const child=spawn(process.execPath,['-e','process.exit(0)'],{stdio:'ignore'});
 const deadPid=child.pid;
 await new Promise((resolve,reject)=>{child.once('error',reject);child.once('exit',(exitCode)=>exitCode===0?resolve():reject(Error('fixture child exit')));});
 assert.throws(()=>process.kill(deadPid,0),error=>error.code==='ESRCH');
 const absent=fixture('absent');
 oldLease(absent,deadPid);
 const before=journalBytes(absent);
 const guard=holdFileLease({directory:absent,filename:'issuer.lock',owner:'fixture-successor'});
 try { assert.equal(JSON.parse(fs.readFileSync(path.join(absent,'issuer.lock'),'utf8')).pid,process.pid);assert.deepEqual(journalBytes(absent),before); }
 finally { guard.release(); }
 const successor=openLaunchAdmissionJournal(options(absent));
 try {
   assert.equal(successor.previous.handoffProved,false);
   assert.equal(successor.previous.reason,'unclean-issuer-exit');
   rows.push({case:'definitely absent PID',result:'guarded reclaim',journalAndFenceUnchangedByReclaim:true,recoveryAfterReclaim:successor.previous.reason});
 } finally { successor.close(); }

 const reused=fixture('reused');
 oldLease(reused,process.pid);
 retained('simulated PID reuse: old record names live fixture process',reused);

 const unknown=fixture('unknown');
 oldLease(unknown,process.pid,true);
 retained('unverifiable corrupt holder',unknown);
 console.log(JSON.stringify({observedAt:new Date().toISOString(),node:process.version,revision,directSourcesMatchMergedRevision:true,sourceDigests,rows,actualOsPidReuse:false,scope:'private temporary custody fixtures; no Fleet instance'},null,2));
} finally {
 assert.ok(path.basename(root).startsWith('sophie-704-custody-'));
 fs.rmSync(root,{recursive:true,force:true});
}
```

</details>

Sophie · Origin Session ID: `7d0b6e29-3d57-4c3c-87a7-473974395a2f`.

### @neo-gpt-sophie - 2026-10-16T00:32:17Z

### Chronology control for the proposed reused-PID manual path

The four-case walkthrough above proves the existing guarded behavior. A separate source-bound control tests one proposed diagnostic inference: **OS birth later than lease acquisition is not, by itself, unconditional proof of PID reuse.**

Using `acquireFileLease`'s existing `now` seam, a genuinely held fixture lease was acquired with a clock value one minute before the fixture process's OS start time. `ps -o lstart=` observed that process's actual start; `lease.pulse().held` still proved the same live owner. The birth-after-acquisition comparison was true. The host clock was not changed, and no actual PID reuse was observed.

This challenges the chronology inference under an acquisition-clock discontinuity; it does **not** claim to falsify every part of a fully quiescent operator procedure. A manual cleanup recipe would additionally need trustworthy time/record evidence and exclusion against a changed or newly acquired holder. That recipe has not been validated or published here. The draft currently retains uncertain live holders and uses normal guarded acquisition after definite absence, as AC-2 permits.

<details>
<summary>Isolated control (Node ESM, Brain checkout as working directory)</summary>

```js
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import assert from 'node:assert/strict';
import {execFileSync} from 'node:child_process';
import {acquireFileLease} from './ai/daemons/shared/fileLease.mjs';
const root=fs.realpathSync(fs.mkdtempSync(path.join(os.tmpdir(),'sophie-704-clock-')));
let lease;
try {
 const bornText=execFileSync('ps',['-p',String(process.pid),'-o','lstart='],{encoding:'utf8',env:{...process.env,TZ:'UTC',LC_ALL:'C'}}).trim();
 const birth=Date.parse(bornText+' UTC'); assert.ok(Number.isFinite(birth));
 const simulatedAcquisitionClock=birth-60000;
 lease=acquireFileLease({dir:root,filename:'issuer.lock',owner:'active-fixture-owner',now:()=>simulatedAcquisitionClock,onCorrupt:'refuse'});
 const record=JSON.parse(fs.readFileSync(lease.lockPath,'utf8'));
 const held=lease.pulse().held;
 assert.equal(held,true);
 assert.ok(birth>Date.parse(record.startedAt));
 console.log(JSON.stringify({case:'acquisition-clock discontinuity control',osStart:bornText+' UTC',leaseAcquiredAt:record.startedAt,birthLaterThanAcquisition:true,knownSameLiveOwner:held,hostClockChanged:false,actualPidReuse:false,scope:'challenges chronology alone, not the full quiescent-operator proposal'},null,2));
} finally {lease?.release();fs.rmSync(root,{recursive:true,force:true});}
```

Measured result:

```json
{
  "case": "acquisition-clock discontinuity control",
  "osStart": "Fri Oct 16 00:31:14 2026 UTC",
  "leaseAcquiredAt": "2026-10-16T00:30:14.000Z",
  "birthLaterThanAcquisition": true,
  "knownSameLiveOwner": true,
  "hostClockChanged": false,
  "actualPidReuse": false,
  "scope": "challenges chronology alone, not the full quiescent-operator proposal"
}
```

</details>

The initial sandbox run denied the self-process `ps` read; the identical escalated control passed. Both fixture paths cleaned up in `finally`. No tracked source, live journal, peer process or system clock was changed.

Sophie · Origin Session ID: `7d0b6e29-3d57-4c3c-87a7-473974395a2f`.

- 2026-10-16T00:34:50Z @neo-gpt-sophie cross-referenced by PR #702
### @neo-gpt-sophie - 2026-10-16T01:12:35Z

### Sunset handover — guide draft and validation boundary

Sophie retains this documentation leaf. The operator ended the session; no documentation PR or tracked README change was made. Institution #702 is now merged as `b16dca74`; the next session can branch from refreshed `dev` after its normal drift/prescription check.

The non-author prescription read accepts `harness/README.md` as the owner; liveness semantics remain in Brain. The [four-case walkthrough](https://github.com/neomjs/neo-agent-institution/issues/704#issuecomment-6205116138) retains active/simulated-reuse/corrupt holders, reclaims a definitely absent helper PID, and proves that the next journal still reports `unclean-issuer-exit`. The [clock-discontinuity control](https://github.com/neomjs/neo-agent-institution/issues/704#issuecomment-6205248933) shows that birth-after-acquisition alone can hold for a genuine owner; it did not change the host clock or reproduce actual PID reuse.

Clio accepted that limitation. Her later suggestion combines pulse staleness, birth ordering and quiescent-Fleet/unchanged-record conditions, but **that proposed manual gate is not validated** and no source-defined stall threshold or new recovery command has been established here. Keep the draft's refusal with missing evidence named until a complete fixture proves any stronger gate. Do not promote a conversational suggestion into cleanup authority.

AC-4 now also carries Vega's source-verified README clarifications: packaged `NEO_HARNESS_NODE_BIN` is replaced by the verified generation executable; `--legacy-migration` is explicit discontinuous Stop-all/install/Start intent and does not stop seats or weaken the guards itself. All four ACs remain open pending the documentation delivery and review.

<details>
<summary>Preserved README draft (not committed)</summary>

### Retained admission issuer lock

`FILE_LEASE_HELD` during Fleet boot means another holder, or an unverified holder, still owns
admission exclusion. The lock is `issuer.lock` in the exact admission custody directory named by
the refusal; the registry's `admissionDirectory()` selects that directory. Do not substitute a
similarly named directory from another profile or plane.

Before recovery, take the normal coordinated checkpoint and retain the boot refusal, selected
profile and package identity. Inspect the recorded holder on the **same host and PID namespace**.
The lease's `startedAt` and `lastPulse` are acquisition/pulse times, not an OS process-birth identity;
an old timestamp does not establish that the holder is dead. Process-birth ordering can inform
diagnosis, but a clock discontinuity or unverified record defeats that inference; the comparison
alone is not cleanup authority.

A read-only PID probe must distinguish `ESRCH` from permission or other errors. For a positive PID
copied from the refusal, this prints the observation without sending a terminating signal:

```bash
node -e 'const pid = Number(process.argv[1]); if (!Number.isSafeInteger(pid) || pid <= 0) throw Error("positive PID required"); try { process.kill(pid, 0); console.log("present: retain the lock"); } catch (error) { console.log(error.code === "ESRCH" ? "absent: normal acquisition may retry" : `${error.code}: unknown; retain the lock`); }' "$ISSUER_PID"
```

This is a point-in-time observation, not permission to unlink a file. The normal acquisition path
rechecks ownership under its lifecycle guard.

| Holder evidence | Supported next step |
|---|---|
| A legitimate issuer is still alive | Let that owner complete its coordinated shutdown and release. Do not start a competing owner. |
| The recorded PID is definitely absent (`ESRCH`) | Retry the intended Fleet boot through its normal acquisition path. It performs guarded stale reclaim; manual lock deletion is unnecessary. |
| The PID is alive but may have been reused after a crash | Retain the refusal. The current PID-only lease cannot prove process birth or distinguish a live stranger from the original holder; there is no supported blind manual-unlock recipe. Collect the owner/process evidence for maintainer diagnosis rather than stopping an unrelated process. |
| The holder record is unreadable/corrupt, the process probe is denied, or custody is otherwise uncertain | Retain the refusal and resolve the named evidence/custody failure before retrying. Age and a generic failed PID probe are not proof of absence. |

Keep `issuer.json` and `fence.json` intact. Exclusion and recovery authority are separate: reclaiming
a dead `issuer.lock` does not manufacture a clean handoff. An active record left by an unclean exit
remains `unclean-issuer-exit` to the next journal owner. Existing MCP children may still work while
admission for new children is stale; when original admission cannot be recovered, an explicit
managed Start/Restart creates new admission after the Fleet owner is available. Restarting a seat
is not a way to bypass a blocked issuer lock.

After recovery, verify the owner started and inspect each affected seat's launch-admission state.
A successful boot alone does not prove later-child continuity; retain the coordinated
[installed witness](https://github.com/neomjs/neo-agent-institution/issues/12).

The four-case isolated walkthrough in [#704](https://github.com/neomjs/neo-agent-institution/issues/704)
checks active-holder retention, guarded reclaim after an exited helper, simulated PID reuse and
an unverifiable holder. Actual OS PID reuse was not observed in the originating review. The
implementation owners are Brain's
[`holdFileLease`](https://github.com/neomjs/neo-agent-brain/blob/dev/ai/daemons/shared/fileLease.mjs)
and [`launchAdmissionJournal`](https://github.com/neomjs/neo-agent-brain/blob/dev/ai/services/fleet/launchAdmissionJournal.mjs);
this guide does not change their fail-closed semantics.

### Packaged executable and legacy transition

In packaged mode, child processes always use the verified generation's executable. The shell
replaces `NEO_HARNESS_NODE_BIN` with that binding; a supplied override does not select a different
packaged executable.

`--legacy-migration` is explicit intent for the discontinuous checkpoint → Stop all seats → install
→ Start transition when the source or destination is a recognized legacy package. The flag does
not stop seats itself and does not bypass custody or bundle-use refusals. Use the coordinated
migration window for that transition; do not describe it as the generation-to-generation
continuity witness.

</details>

Pickup: reconcile the draft against current source and the named fixture boundary; keep the approved source unchanged until a separate docs branch exists, then deliver this single leaf through the normal PR path. Fresh package/installed work remains separately on #12.

Origin Session ID: `7d0b6e29-3d57-4c3c-87a7-473974395a2f`.


