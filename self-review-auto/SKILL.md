---
name: self-review-auto
description:
  Unattended variant of self-review — runs every pass, applies every fix it
  finds, and repeats each pass with fresh subagents until one finds zero
  defects (each pass has its own iteration cap, 1–7). Diff-only passes share
  one review, then parallel fixers each own a disjoint set of files in the
  same working tree (no worktrees, no merges), then fresh reviewers verify.
  Reports iterations, time and tokens per pass and per agent call. Load only
  when asked for an automatic / auto-fix / unattended self-review; use
  self-review for the interactive one.
---

# Self-Review (Auto)

The same passes as `self-review`, run without asking: every finding is fixed
in place, and each pass repeats until a fresh reviewer finds nothing left to
fix.

**The goal is zero defects per pass:** a pass is done when a fresh reviewer
running its full methodology finds nothing. Every rule below serves that;
when one seems to conflict with it, the goal wins.

The stance, the pass methodologies and the severities are those of
`self-review`; this skill changes only the control flow. Read
`${CLAUDE_PLUGIN_ROOT}/skills/self-review/SKILL.md` → "Stance" once, and
give each subagent the pass reference by absolute path.

## Usage

```
/skills:self-review-auto                  # Uncommitted changes
/skills:self-review-auto HEAD~3..HEAD     # A range ending at HEAD, plus the fixes
/skills:self-review-auto --staged         # Staged changes
```

## References

The pass methodologies live in
`${CLAUDE_PLUGIN_ROOT}/skills/self-review/references/pass-*.md` (shared with
`self-review`, not copied). Resolve that to an absolute path before
spawning subagents; their working directory is the reviewed repository.

## Ground rules

- **Never commit, push, open PRs or create tracking issues.** Fixes stay in
  the working tree for the user to review. No worktrees, branches or temp
  commits either: every agent works in the one working tree.
- **Fix, don't ask.** Every finding is fixed, however small, large or
  inconvenient. The only exception is a finding that *cannot* be fixed in
  this working tree: the fix belongs in another PR or repository, or it
  needs a product decision the code and docs can't settle. Such a finding is
  *deferred*: recorded with its reason and not re-reported later, but it
  keeps its pass from counting as clean (see "Status" in Step 4). "Out of
  scope", "too large" or "a follow-up" are not reasons to defer.
- **Fresh context.** Each agent is a new `Agent` call (`general-purpose`),
  never a `SendMessage` continuation and never a fork. Only the deferred
  lists are carried between rounds, so deferred findings are not
  re-reported.
- **Iteration ceiling per pass:** the Max column in Step 4 (3 for any pass
  without one). When the last allowed iteration still found something, stop
  that pass with status `ceiling` (it did not reach zero; more defects may
  remain). Judgment passes (reuse, architecture, nits) never reach zero, so
  extra iterations only buy new opinions; passes whose fixes can create real
  defects (security, testing, correctness) get more.
- **One writer per file at any moment.** Parallel fixers each own a disjoint
  set of files and edit nothing else; everything that crosses files is done
  by one agent at a time. Reviewers are read-only.
- **Never end your turn while an agent is running.** Your final message is
  the Step 6 report; every message before it must carry a tool call. If an
  `Agent` call comes back as still running in the background, block on it
  (the task-output tool with blocking, or a wait/monitor tool) instead of
  replying. A turn that ends with work in flight stops the review until
  someone resumes you.
- **No idle time between calls.** Launch agents in the foreground (several
  in one message when they run in parallel) so each call returns its
  result. Never wait with `sleep`, polling loops over output files, or
  "until the file stops changing" monitors; if a call does come back
  backgrounded, make one blocking wait on it. Agent time should account for
  nearly all of the review's wall time; report the gap if it doesn't.
- **Model tier:** the Hygiene and File Size verify reviewers run on the
  smallest, fastest model tier the `Agent` tool offers in this environment
  (whatever is listed and available; don't name a model that isn't). If no
  smaller tier is available, they use the same model as everything else.
  Every other agent uses the caller's model. When the caller pins one model
  for every agent, that wins.
- **Tests:** agents run only the test files that cover the files changed,
  never whole suites, end-to-end, mutation or stress runs. Step 5 runs the
  full validation once.

## Step 1: Fix the scope

Record the scope once; every round reviews it as it stands *now*, so it
includes the fixes earlier rounds made.

| Invocation | Scope command each round |
|---|---|
| none | `git diff HEAD` plus untracked files (`git ls-files --others --exclude-standard`) |
| `<A>..HEAD` (or `<A>..`) | `git diff <A>` plus untracked files |
| `--staged` | `git diff --cached`; you stage the edited paths after each integration check (`git add -- <paths>`), agents never stage |

- A range whose end is not `HEAD`: stop and ask — fixes can't land on a
  commit that isn't checked out.
- `--staged` when a staged file also has unstaged edits: stop and ask —
  staging a fix would sweep the unstaged edits in.
- Empty scope: say so and stop.

Create a scratch directory outside the repository (`mktemp -d`). Note the
repository root, the changed-file list, the repo's cheapest documented check
(a typecheck or lint script from `CLAUDE.md`, `AGENTS.md`, `README` or
`package.json`/`Makefile`/`justfile`, scoped to the affected packages if the
repo supports that), how to run a single test file, and, if documented:
the repo's **structural checks** (dependency boundaries, cycle or layering
checks: fast and repo-wide) and its **incremental mutation command** (one
that mutates only changed files).

**Scope snapshot.** Before each review round or sequential iteration, write
the scope to `<scratch>/scope-<n>.diff`: the scope command's output, then
each untracked file as a new-file diff (`git diff --no-index /dev/null
<file>`), and the changed-file list with line counts (`wc -l`) to
`<scratch>/files-<n>.txt`. Agents read these instead of re-running git; they
still open whole files when a finding needs context.

## Step 2: Start historical intelligence

Spawn one background agent (`run_in_background: true`) with the prompt from
`self-review` Step 1b (`pass-11-historical-intel.md`, the changed files, the
repo root). It is informational and is not looped. Keep its report for
pass 9. Record its duration and tokens from its completion notification.

## Step 3: File-size gate

Pass 7 starts with a script, not an agent: from `files-<n>.txt`, list the
changed source files at or above 300 lines (skip generated files, lock
files, fixtures and snapshots). None → pass 7 is `clean` with 0 iterations
and is left out of stage B. Otherwise its reviewers look only at those
files. Rerun the gate before each verify round; a pass-7 file that dropped
below 300 lines leaves the list.

## Step 4: The passes

Run the stages in this order. Pass numbers keep the `self-review` numbering;
Correctness runs last of the passes so it re-checks every fix; stage D then
checks Correctness's own fixes.

| Stage | # | Pass | Max | Reference |
|---|---|---|---:|---|
| A | 0 | Code Reuse | 2 | `pass-00-code-reuse.md` |
| A | 1 | Architecture | 3 | `pass-01-architecture.md` |
| B | 2 | Security | 3 | `pass-02-security.md` |
| B | 3 | Type Safety | 3 | `pass-03-type-safety.md` |
| B | 4 | Testing | 3 | `pass-04-testing.md` |
| B | 5+6+8 | Hygiene | 1 | `pass-05-ai-slop.md`, `pass-06-nits.md` and `pass-08-comments.md`, together |
| B | 7 | File Size | 2 | `pass-07-file-size.md` — only the files the Step 3 gate lists |
| C | 12 | Documentation | 2 | `self-review` Steps 2e and 2f (stale-duplicate half): update the High/Medium docs and stale restatements the diff makes wrong |
| C | 10 | Cross-PR | 2 | `pass-10-cross-pr.md` — only when the branch has an open PR (`self-review` Step 2d); otherwise *skipped*. Changes needed in the other PR are deferred |
| C | 13 | Adversarial cold read | 2 | `self-review` Step 2g — only when a model of a different family is reachable from the shell and the code may leave the environment; otherwise *skipped*. Each iteration sends the current diff, confirms or refutes each point against the code, and fixes the ones that hold |
| C | 9 | Correctness | 7 | `pass-09-correctness.md`, plus the historical report (wait for its notification before starting pass 9; if it failed, say so in the prompt) and `self-review` Step 2f's "Claims That Don't Hold" half |
| D | 3, 4, 5+6+8 | Settle | 1 | Only if Correctness changed code: Type Safety, Testing and Hygiene on the files Correctness touched (see "Stage D") |

Stages A and C reach beyond the diff (repo-wide search, callers, docs,
other PRs), so they run one pass at a time with the sequential loop. Stage
B's passes judge the diff itself, so they share one review and fix in
parallel. Reuse and Architecture go first because their fixes restructure
code that stage B then reviews.

### Sequential loop (stages A and C)

```
for each pass:
  iteration = 0; deferred = []; started = now (date +%s)
  loop:
    iteration += 1
    write scope snapshot
    result = Agent(iteration prompt)       # fresh subagent, reviews and fixes
    log the call: pass, iteration, found, fixed, duration_ms, total_tokens
    deferred += result.new_deferred
    if result.found == 0:                  # nothing new: fixed and deferred are 0
      status = clean / clean (N deferred); stop
    if iteration == max for this pass: status = ceiling; stop
  record wall time = now - started
```

### Stage B: shared review, partitioned fixes, fresh verify

```
active = stage B passes (minus File Size if the gate listed nothing)
round = 1; started = now
write scope snapshot
findings = Agent(shared review prompt)          # B1: one reviewer, all active passes
loop:
  log the round's review calls as iteration <round> of each pass they covered
  record each pass's reviewer-deferred findings; for each active pass with
    no new findings this round: status = clean / clean (N deferred); drop it
  if findings:
    B2: cross = findings marked cross-file
        if cross: Agent(cross-file fixer prompt, cross)   # serial, one agent
    B3: local = the rest; partition by file (below)
        Agents(file fixer prompt, one per group, in ONE message)   # parallel
        returned needs_cross_file findings -> one more cross-file fixer
    B4: integration check (below)
  for each pass still active: if round == its max: status = ceiling; drop it
  if active is empty: B6 mutation gate; stop
  round += 1
  write scope snapshot; rerun the Step 3 gate
  findings = Agents(verify reviewer prompt, one per active pass, in ONE message)
                                                 # B5: fresh, read-only, parallel
record stage B wall time = now - started
```

**B1, shared review.** One read-only agent loads the whole scope once and
applies every active pass's reference in turn, then sweeps again asking
"anything new?" until a sweep adds nothing (at most 3 sweeps). One load of
the diff serves every pass. Its reviewer is anchored on what it already
said, which is why every later round uses fresh verify reviewers.

**Classifying findings.** Each finding is `local` (the fix edits only its
own file, plus that file's own test file) or `cross-file` (a rename, a
moved or extracted helper, a file split, a changed signature or export used
elsewhere, a new shared module, or any fix that edits two source files). A
File Size split is always cross-file.

**B2, cross-file fixer.** One agent applies every cross-file finding, one
at a time. These run before the parallel fixers because they move the code
that local fixes target; each local finding quotes its line so a fixer can
re-locate it after lines move.

**B3, partition.** Group the local findings by file. A source file and its
own test file go in the same group. Then pack the groups into at most 4
fixer jobs with roughly equal finding counts; no file appears in two jobs.
Each file fixer edits only the files it owns. A fix that needs any other
file is not made; the fixer returns it as `needs_cross_file`, and one
cross-file fixer applies those after the parallel fixers finish. File
fixers don't run checks or tests: another fixer may be mid-edit in the same
package. B4 checks the combined result.

**B4, integration check.** After all fixers in the round finish:

1. Confirm the fixers stayed in their lanes: every path that changed since
   the round's snapshot is either owned by a fixer job or was edited by a
   cross-file fixer. Anything else is a violation; record it in the log and
   name it in the report.
2. Run the cheap check once, the structural checks, and the test files
   covering every file changed this round. A fix that moves code (splits,
   extractions, shared types) is the usual source of a new import cycle;
   catch it here, not in the final validation.
3. On failure, run one repair agent (repair prompt) with the failure
   output, then rerun the check and tests. Up to 2 repair attempts; if it
   still fails, the repair agent reverts the edits it can attribute to the
   failing finding and defers that finding with the failure as the reason.
   Never leave the tree failing a check that passed before the round.
4. `--staged` only: stage the paths edited this round.

**B6, mutation gate.** When stage B ends (no pass left active), run the
repo's incremental mutation command once, if it has one. Surviving mutants
in changed code, stale ignore entries, or a mutation run that cannot start
(a test that fails in the mutation tool's sandbox copy) are findings of
pass 4 (Testing): send them through B2–B4 once, preferring a test that kills
the mutant over an ignore entry, then rerun the command once. Anything still
failing is deferred with the output as the reason. Stage D ends with the
same gate when it changed code. Report each gate's duration and result.

**B5, verify.** Fresh read-only reviewers, one per pass still active, run
in parallel on the current scope. Their findings re-enter at B2. This
restores the independence the shared review gave up and catches defects
the fixers introduced.

**Counting.** A stage B pass's iteration count is the number of rounds it
was reviewed in (the shared review is iteration 1 for every pass it
covered). A pass's "Fixed" is the number of its findings that ended up
fixed, by whichever fixer. Fixer, repair and integration-check time and
tokens are reported on their own rows, not split across passes.

### Stage D: settle Correctness's fixes

Correctness runs last, so nothing else reviews what it changed: a fix can
add an untested branch, a cast or a stray comment. If any Correctness
iteration changed code (not only docs), run one settle round:

1. Scope: only the files Correctness changed, as they stand now (their full
   diff against the review base), plus each one's own test file. Write it
   as its own scope snapshot.
2. Run verify reviewers (B5 prompt) for Type Safety, Testing and Hygiene,
   in parallel, on that scope; Hygiene on the small model tier.
3. Their findings go through B2–B4 exactly as in stage B (cross-file fixer,
   partitioned file fixers, integration check with repair).
4. Run the B6 mutation gate on the result.
5. Stop. There is no second round and no loop back into Correctness: the
   settle fixes are mechanical (tests, types, hygiene), and Step 5's full
   validation checks them.

If Correctness changed no code, stage D is `skipped (no Correctness code
changes)`. Report stage D as its own row: findings, fixed, deferred, and
`clean` when its reviewers found nothing, otherwise `settled` (fixes
applied, not re-reviewed).

### Status

`clean` means zero defects: the last iteration found nothing and nothing
was deferred during the pass. `clean (N deferred)` means nothing more can be
fixed here but N known defects remain; never report it as plain `clean`.
Check the arithmetic: for a sequential iteration or a fixer, `found ==
fixed + len(new_deferred)` (plus `needs_cross_file` for a file fixer); an
agent that reports findings it neither fixed nor deferred is a failed call.

If an agent call fails or returns no parseable result, retry it once; on a
second failure record the pass (for a fixer: every pass whose findings it
held) as *error* and move on.

### Iteration prompt (sequential loop)

```
You are iteration <i> of pass <#> (<name>) of an automatic self-review.
Work only from this prompt and the files; you have no earlier context.

Objective: drive this pass's defects to zero. This pass repeats with a
fresh reviewer after you until one finds nothing, so a finding you hold
back is only postponed. Report everything the methodology catches,
borderline findings included; don't stop at the first few and don't drop a
finding because it is minor.

Repository: <repo root>
Scope: <scratch>/scope-<n>.diff is the change as it stands now (untracked
files appear as new-file diffs); <scratch>/files-<n>.txt lists the changed
files with line counts. Open the files themselves when a finding needs
context.
Methodology: follow <absolute path to reference> exactly. Stance: <the
Stance section of self-review, verbatim>.
<pass-specific context: historical report for pass 9; PR number for pass 10>

Already deferred in this pass (do not report these again):
<deferred list, or "none">

Do this:
1. Review the scope under this pass only. Findings outside this pass's
   remit are not yours.
2. Fix every finding in place with the Edit tool, whatever its size. Don't
   refactor untouched code beyond what a finding requires.
3. Defer a finding only when it cannot be fixed in this working tree: the
   fix belongs in another PR or repository, or it needs a product decision
   the code and docs can't settle. List it under new_deferred with that
   reason. "Out of scope", "too large" or "a follow-up" are not reasons:
   fix it.
4. If you edited files, run `<cheap check or "none">` and only the test
   files covering the files you changed (`<single-test-file command>`);
   never whole suites, end-to-end, mutation or stress runs. If your edits
   broke them, fix them or revert your own edit and defer the finding.
5. <only for --staged> Stage exactly the paths you edited: `git add -- <paths>`.
6. Never commit, push, stash, reset or discard changes you did not make.

Reply with a short list of what you fixed (file:line, severity, one line
each), then exactly one JSON block:
{"pass": <#>, "iteration": <i>, "found": <n>, "fixed": <n>,
 "new_deferred": [{"file": "", "line": 0, "severity": "critical|warning|suggestion",
                   "summary": "", "reason": ""}],
 "files_changed": [], "check": "passed|failed|not run"}
```

### Shared review prompt (B1)

```
You are the shared reviewer of an automatic self-review. Work only from
this prompt and the files; you have no earlier context. You are READ-ONLY:
do not edit, stage or run anything that writes. Fixers apply your findings.

Repository: <repo root>
Scope: read <scratch>/scope-<n>.diff in full first; <scratch>/files-<n>.txt
lists the changed files with line counts. Open files when a finding needs
context.
Stance: <the Stance section of self-review, verbatim>.

Apply each of these passes in turn, following its reference exactly:
<# name: absolute path(s), one line per active pass; for pass 7, the gate's file list>

Then sweep the whole scope again under all of them, asking only "what did
I miss?". Repeat until a sweep adds nothing, at most 3 sweeps. Report
everything, borderline findings included: each pass repeats with a fresh
reviewer until one finds nothing, so a finding you hold back is only
postponed.

Already deferred (do not report again): <deferred lists, by pass, or "none">

For each finding give: pass, file, line, the quoted line (so a fixer can
find it after lines move), severity, what is wrong, the exact fix (the
replacement text or the precise change), and kind:
- "local": the fix edits only this file, or this file and its own test file;
- "cross-file": a rename, a moved or extracted helper, a file split, a
  changed signature or export used elsewhere, a new shared module, or any
  fix that edits two source files. List every file the fix touches.
Mark a finding deferred only when it cannot be fixed in this working tree
(another PR or repository, or a product decision); give the reason.

Reply with exactly one JSON block:
{"sweeps": <n>,
 "findings": [{"pass": "", "file": "", "line": 0, "quote": "", "severity": "",
               "summary": "", "fix": "", "kind": "local|cross-file", "files": []}],
 "deferred": [{"pass": "", "file": "", "line": 0, "severity": "", "summary": "", "reason": ""}]}
```

### Verify reviewer prompt (B5)

The shared review prompt with these changes: "You are the round <r> verify
reviewer for pass <#> (<name>)", only that pass's reference(s), no sweep
instruction (one full pass of the methodology), and "Other reviewers run at
the same time on other passes." Same reply JSON.

### Cross-file fixer prompt (B2)

```
You are a cross-file fixer of an automatic self-review. Work only from
this prompt and the files; you have no earlier context. You are the only
agent editing right now.

Repository: <repo root>
Apply every finding below, one at a time, with the Edit tool. Each is a
change that crosses files (rename, move, extraction, split, signature or
export change): update every caller, import, re-export, test, mutation or
coverage config entry and doc reference it affects. Re-read each file
before editing; earlier fixes move lines.

<findings, each tagged with its pass>

Then run `<cheap check or "none">` and the test files covering what you
changed (`<single-test-file command>`); never whole suites. If your edits
broke them, fix them or revert your own edit and defer the finding.
Defer only what cannot be fixed in this working tree.
Never commit, push, stash, reset or discard changes you did not make.

Reply with exactly one JSON block:
{"found": <n>, "fixed": <n>,
 "new_deferred": [{"pass": "", "file": "", "line": 0, "severity": "",
                   "summary": "", "reason": ""}],
 "files_changed": [], "check": "passed|failed|not run"}
```

### File fixer prompt (B3)

```
You are file fixer <k> of <N> in an automatic self-review. Work only from
this prompt and the files; you have no earlier context. Other fixers are
editing other files in this same working tree right now.

Repository: <repo root>
You own ONLY these files: <file list>. Edit nothing else, and don't
create, move or delete files.

Apply every finding below with the Edit tool. Each quotes its line; find it
by the quote, since lines may have moved. If a fix needs a change to any
file you don't own, don't make it: return it under needs_cross_file with
the files it needs. Defer only what cannot be fixed in this working tree.

<findings for these files, each tagged with its pass>

Do not run checks or tests (other fixers may be mid-edit; the coordinator
checks the combined result). Never stage, commit, push, stash, reset or
discard.

Reply with exactly one JSON block:
{"job": <k>, "found": <n>, "fixed": <n>,
 "needs_cross_file": [<the finding, plus "files": [...]>],
 "new_deferred": [{"pass": "", "file": "", "line": 0, "severity": "",
                   "summary": "", "reason": ""}],
 "files_changed": []}
```

### Repair prompt (B4)

```
You are the repair agent of an automatic self-review. Work only from this
prompt and the files; you have no earlier context. You are the only agent
editing right now.

Repository: <repo root>
This round's fixes broke the check below. Make it pass again without
undoing the intent of the fixes.

Failing command(s) and output: <output, trimmed to the errors>
Findings applied this round, with files: <list>

Fix the breakage with the Edit tool and rerun the failing command(s). If a
specific fix can't be made to work, revert that fix's edits (only that
fix's) and report it under new_deferred with the failure as the reason.
Never commit, push, stash, reset or discard changes you did not make.

Reply with exactly one JSON block:
{"repaired": true|false, "reverted": [<finding summaries>],
 "new_deferred": [{"pass": "", "file": "", "line": 0, "severity": "",
                   "summary": "", "reason": ""}],
 "files_changed": [], "check": "passed|failed"}
```

## Step 5: Final check

Run the repo's documented full validation command once (if it has one and
it runs in reasonable time on this machine) and report its exit code and
duration. A failure caused by the fixes gets fixed now; one with a
machine-local cause is named, not dismissed. Then `git diff --stat` for the
files the loop changed.

## Step 6: Report

Lead with the pass table. Tokens and time are taken from each subagent's
usage (`total_tokens`, `duration_ms`); wall time is from `date +%s` around
each pass (stage B: around the whole stage). Your own orchestration tokens
aren't included — say so under the table.

```markdown
## Self-Review (Auto) Complete

| # | Pass | Iterations | To clean | Fixed | Deferred | Status | Wall time | Agent time | Slowest call | Tokens |
|---|------|-----------:|---------:|------:|---------:|--------|----------:|-----------:|-------------:|-------:|
| 0 | Code Reuse | 2 | 1 | 3 | 0 | clean | 4m 12s | 3m 58s | 2m 40s (#1) | 182,400 |
| … |
| B | Shared review | 1 | — | — | — | — | — | 3m 10s | — | 140,000 |
| B | Verify reviewers | 3 | — | — | — | — | — | 4m 20s | 1m 50s (r2, pass 4) | 210,000 |
| B | Cross-file fixers | 2 | — | 6 | 0 | — | — | 3m 05s | 2m 10s (r1) | 95,000 |
| B | File fixers | 7 | — | 31 | 1 | — | — | 9m 40s | 2m 30s (r1, job 2) | 260,000 |
| B | Integration checks | 3 | — | — | — | 1 repair | 2m 30s | 1m 10s | — | 40,000 |
| B | Mutation gate | 1 | — | 2 | 0 | 2 survivors killed | 6m 40s | 1m 30s | — | 30,000 |
| B | Stage B | — | — | — | — | — | 14m 30s | — | — | — |
| 7 | File Size | 0 | — | — | — | clean (gate: no file ≥ 300 lines) | — | — | — | — |
| 11 | Historical intel | 1 | — | — | — | info | 1m 40s | 1m 40s | — | 61,200 |
| 13 | Adversarial | — | — | — | — | skipped (no second model) | — | — | — | — |
| D | Settle (Correctness's files) | 1 | — | 2 | 0 | settled | 2m 40s | 3m 30s | 1m 20s (Testing) | 70,000 |
| | Final check | — | — | — | — | exit 0 | 6m 10s | — | — | — |
| | **Total** | **N** | | **N** | **N** | | **…** | **…** | | **…** |

Iterations counts every review, including the final clean one; "To clean"
is the number of iterations that found something (Iterations − 1 when
clean). Status is `clean` (zero defects), `clean (N deferred)`, `ceiling`,
`error` or `skipped`. Stage B pass rows show iterations, fixed and status
but no time or tokens; those are on the shared-review, verify, fixer and
integration rows. Agent time for parallel calls is summed; the stage B
wall time shows what the parallelism saved. Tokens are subagent tokens
only.
```

Under the table, one line: how many passes reached zero defects out of how
many ran. Then the **call log**: one line per agent call in run order
(stage, pass or job, round or iteration, found, fixed, duration, tokens),
plus any lane violations from B4, so the slowest step is visible.

Then:

- **Deferred findings** grouped by severity, each with file:line, pass and
  reason, so the user can decide on them.
- **Final check:** command, exit code and duration.
- **Evaluation:** the six dimensions and score from `self-review` →
  "Evaluation Dimensions", scored on the change *after* the fixes; each
  deferred critical/warning finding still deducts.
- **Caveat** when any pass hit the ceiling or errored, and that later fixes
  are not re-checked by earlier passes (only Correctness, which runs last,
  sees every fix, and stage D re-checks only Correctness's files under
  Type Safety, Testing and Hygiene); running the skill again re-checks everything.
- Changes are uncommitted; say that the user should review the diff.
