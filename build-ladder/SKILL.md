---
name: build-ladder
description:
  Drive an approved ladder to completion unattended — per phase a builder
  subagent, an automatic self-review, the full checks once, then commit, PR,
  merge and a time report, then the next phase. Load when asked to build,
  execute, run or "autodrive" a ladder (or a range of its phases) without
  supervising each step.
---

# Build Ladder

You are the coordinator. Subagents build and review; you own the gates, the
git operations, the timing and what the user hears. Each phase goes through
the same loop:

```
kickoff → build → (live smoke) → review → checks → ship → time report → next
```

The loop is tuned for wall time without reviewing less: builders run only
scoped checks, the review fixes in parallel and gates itself, and the full
(slow) checks run exactly once per phase, after the review.

## Usage

```
/skills:build-ladder specs/my-feature.ladder.md            # every phase not yet complete
/skills:build-ladder specs/my-feature.ladder.md 4-7        # phases 4 to 7
/skills:build-ladder specs/my-feature.ladder.md --stop-after 7
```

## Authorisation

Building a ladder unattended commits, pushes, opens PRs and merges. Most
users' standing rule is "commit only when asked", so confirm once at the
start, in one message, that the user wants this loop to commit, push, open a
PR and merge each phase without asking. That answer covers this ladder only.

Everything else outward-facing still needs its own explicit yes: real
credentials or tokens, paid cloud resources, writes to trackers or other
repositories, deleting data, force pushes, editing CI workflows the token
can't merge. Ask once, name exactly what will be created and what it costs,
and record the answer (memory, if available) so later phases don't re-ask.

## Step 0: Preflight (once)

Read, and keep for every prompt you write:

- The ladder: status table, every phase you will run, its **Notes** (the
  standing gate and earlier decisions), and the spec(s) it points at.
- The repo's `CLAUDE.md` / `AGENTS.md` / `README`: the full validation
  command (and how to run it headless if it opens windows), the scoped
  per-package checks, structural checks (dependency boundaries), the
  incremental mutation command, stress mode, test budgets, lint and style
  rules, and anything the repo forbids touching.
- The base branch (from the repo's conventions or the user's), the merge
  method, and whether PR checks gate the merge or local checks are the gate.
- Shared-machine constraints: other projects' containers, processes or
  sessions on the host that must never be touched.

Then write down the **rules block** you will paste into every builder,
review and fixer prompt: never commit, push, stash, reset or discard; files
and directories that are off-limits; the lint/style rules; "never run two
heavy test runs (e2e, mutation, full validate) at once"; how to treat
shared-host resources; never read secret files or print tokens; "if a
permission prompt is denied, don't work around it, report it".

**Model choice.** Use the user's primary model via the runner's model
option or a pinned role profile. Before branching, if the user approved a
fallback for this repo, resolve its exact installed model ID and verify
that the runner can select it for builders, reviewers and fixers without
changing the primary profiles. Record both models and the role names in
the run checkpoint. For example, when the user approves Sol 6.2 with the
latest Claude Sonnet as fallback, preflight both before starting. Do not
send repo content to an unapproved provider. Apply the quota handoff
below to every role, including the review driver.

## Step 1: Kickoff (per phase)

1. Start the phase timer: `date +%s` into a scratch file. Every later stage
   records its own end time the same way.
2. Fresh base: `git fetch origin <base>`, fast-forward the local base if the
   tree is clean, then `git checkout -b <branch> origin/<base> --no-track`.
   Confirm `git rev-parse HEAD` equals `git rev-parse origin/<base>`, and
   that the previous phase's merge is in it.
3. Branch name: the repo's convention (for example
   `<handle>/<phase-slug>`).

## Step 2: Build

Launch one builder subagent (background is fine; you have nothing else to
do on this tree meanwhile). Its prompt:

```
Build phase <n> (`<slug>`, "<title>") of <ladder path>. You're on <branch>,
freshly cut from origin/<base>. Leave everything uncommitted; a review and
commit follow.

First read: <repo instructions>, the phase section in full, the ladder's
Notes (standing gate, earlier decisions), the spec sections it cites, and
the code earlier phases built that this one extends (name the files or
patterns: ports, fakes, contract tests, how a package is added).

Deliver everything in Delivers; make every Done-when check pass. Mark the
phase in-progress in the status table now and complete (with today's date)
when done; record decisions and accepted limits as a new ladder Note.
Update docs the change makes wrong. Tests are the top priority.

Checks — scoped only (the coordinator runs the full suite after review):
- per touched package: typecheck, lint, unit tests with coverage at the
  repo's threshold; the structural/boundary check; integration tests only
  for packages that have that tier and that you touched.
- a short targeted stress run of the test files you added or changed (a few
  seeds, ideally with some CPU load), to catch flakes now.
- do NOT run the full validation, full mutation or full stress.

Before you hand off, a correctness self-check: walk every Done-when item
and the Notes' gotchas against the code — callers, failure paths, partial
failures and cleanup, crash/restart windows, teardown, hostile inputs —
and fix what doesn't hold.

<rules block>
<phase-specific hard rules, e.g. "never call the real cloud API; tests use
the fake; never read .env">

Report: files built; each Done-when item and how it's verified; scoped check
results; stress results; the ladder Note; deviations from the ladder and
why; anything not done.
```

Record the build end time. Read the report for deviations and anything not
done; a gap in a Done-when item goes back to the builder (`SendMessage`)
before review.

## Step 3: Live smoke (only when the phase has one and it's approved)

If the phase ships a command that touches a real external service (a cloud
API, a paid resource), run it yourself after the build and before the
review, so a mismatch with the real service is fixed and then reviewed.
Load credentials only the way the phase's own script does; never echo them;
afterwards grep the output for the secret (count only, never print it) and
confirm through the service's API that nothing it created is left. Tell the
user what was created and removed.

## Step 4: Review

Launch one driver subagent that invokes `self-review-auto` on the
uncommitted scope and follows it end to end, with:

- the model rule above, and the rules block for every agent it starts;
- **skip its final full validation** (Step 5 runs it); Cross-PR is skipped
  without an open PR; the cold read is skipped unless a second model family
  is available and the code may leave the machine;
- a short builder summary (what the phase added, the ladder Note's
  decisions) so reviewers don't re-derive intent;
- "record wall-clock start/end for each pass, round and stage";
- "never end your turn while an agent is running".

Don't touch the tree while it runs. Record the review end time. Read its
deferred findings: anything critical or warning-level that is fixable here
goes back to a fixer before Step 5.

## Step 5: Checks (once, after the review)

Run these yourself, in order, one heavy run at a time:

1. If the review touched the lockfile: a frozen-lockfile install.
2. The full validation (headless if needed). Report the exit code and task
   counts from the output, not a tail.
3. The incremental mutation command on changed code. Survivors, stale
   ignore entries, or a run that can't start go to one fixer agent
   (prefer a test that kills the mutant over an ignore entry), then rerun.
4. Targeted stress for tests changed since the build.
5. Test budgets. An over-budget task is raised only with the user's
   approval (a standing "raise to whatever is practical" counts); set it
   about 25–30% above the slowest measured run on this host and say so in
   the PR.
6. If the review changed code that talks to the external service: rerun
   the live smoke once (Step 3 rules).

Any fix in this step means rerunning validation before shipping. A failure
with a machine-local cause is named and fixed or flagged, never called
"unrelated". Record the checks end time.

## Step 6: Ship

1. `git status --short | grep '^??'`: every untracked file is meant to ship.
   Then `git add -A` and check the staged names contain no secrets, build
   output, reports or caches. Note anything staged that isn't this phase's.
2. Confirm the branch still contains `origin/<base>`.
3. Commit with a Conventional Commits title naming the phase, a body of
   what it delivers, and the attribution line the environment specifies.
4. `git show --stat HEAD`, then push.
5. Open the PR: title as the commit; body = what it delivers (bullets),
   review summary (fix count, notable bugs fixed, deferred items),
   budgets raised and why, check results with exit codes, live-smoke
   resources created and destroyed, and "Time report: see the PR comment".
6. Merge with the repo's method once its gate is met (green PR checks, or
   your local checks when the repo has made those the gate).
7. Post the **time report** as a PR comment (template below), then check
   out the base branch and fast-forward it. Leave branches in place.

## Step 7: Report and continue

Tell the user, in a few lines: PR link, total time and the stage split,
notable bugs the review caught, deferred items that need a decision, budgets
raised, external resources created and removed. Then start the next phase,
unless the stop condition is reached.

Stop when the requested phases are merged, or the user said to stop after a
phase. After stopping, start nothing new and file no follow-ups unless asked.

## Time report (PR comment)

```markdown
## Time report: phase <n> (`<slug>`), soup to nuts

**Total: <h m s>** (branch cut <time> → merged <time>, <date>)

| Stage | Start → end | Duration |
|---|---|---:|
| Build | … | … |
| Live smoke | … | … |
| Review (self-review-auto <version>) | … | … |
| Post-review checks and fixes | … | … |
| Commit, push, PR, merge | … | … |

### Review breakdown
<one row per pass / stage-B round / stage D with start → end, duration,
iterations, result; the slowest single calls; model handoffs, aborted calls,
and quota wait (separate from active build/review time)>

### Post-review checks
<one row per step with duration and result, including failures and what
fixed them>

### Deferred review findings
<each with file, why it was deferred, and what decision it needs>
```

## Working rules

- **Never end your turn while a subagent you need is running.** Wait for
  its notification; never predict or report its result before it arrives.
- **One heavy run at a time** across you and every subagent: full
  validation, e2e, mutation and stress runs compete for the machine and
  produce false failures and budget overruns.
- **Don't duplicate a running agent's work** or edit files it owns. To
  change course, stop it at a safe point (let the current iteration
  finish) and start a new one that resumes from its report.
- **Permission denials stop the step.** Don't route around a denied action;
  tell the user what was blocked and why it was needed.
- **Primary quota hit → hand off immediately.** On an explicit usage-cap
  error, don't end the turn or ask the user to say “continue” when an
  approved fallback is ready. Confirm the failed job stopped. Checkpoint
  the branch/base, tracked and untracked changes, unfinished work, last
  passing checks, and any provider-reported reset time in scratch outside
  the repo. Preserve partial edits. Launch the matching fallback role
  **in the same turn** with that checkpoint and the same scope and rules.
  Tell the user which model is taking over. A launch is not completion.
- **Fallback abort is not a quota error.** Check for an explicit stop,
  cancellation or permission denial first. Respect it: don't relaunch, and
  cancel any wake-up for the run. For an unexplained abort with no stop in
  effect, confirm there is no live job, inspect partial edits, and retry
  the fallback at most once from the updated checkpoint. If that fails,
  both models are blocked, or no approved fallback is available, report
  the exact blocker and resume point. Don't loop or overwrite work.
- **Return to primary at a safe checkpoint.** If the provider supplies a
  reset time and an authorized scheduler is available, schedule and verify
  one retry for that time. A wake-up must not launch a second writer or
  resume a stopped run. Otherwise, say no retry was scheduled. Keep working
  on the fallback either way. Once the reset is due, retry the primary at
  the next handoff after the active fallback job finishes; switch back only
  on success. If the reset time is unknown, check once at a later phase
  boundary, not on every call.
- **Load-sensitive numbers:** budgets and stress results are measured on a
  shared machine; report the load average next to any timing you cite.
- **Keep the user's view current** with short updates at each stage change,
  not tool-by-tool narration.
