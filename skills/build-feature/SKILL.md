---
name: build-feature
description: |
  Execute a feature plan phase by phase, as a disciplined coding agent: branch fresh from main
  (always — pull latest, never branch off a stale branch), read the plan fully line by line,
  re-ground every phase in the current code before writing, then build with clean-code discipline —
  SOLID, DRY, reuse-first (more code = more maintenance surface), async/concurrent by default with
  a synchronization ledger for every spot that must be sequential, a hack ledger so nothing
  sneaky ships unflagged, and sound database design (constraints, indexes, transactions,
  migration discipline). Type-check + tests gate every phase; commits are granular (one per
  logical change); every finished plan ends with the branch pushed and an MR/PR raised whose
  description always has four sections — Problem / Solution / Technical details / Review notes.
  Two hard stops only: an architectural concern (ask the user, with a recommendation) and a red
  build. Everything else runs continuously through all phases, MR/PR included.
  Use when: implementing a plan from /plan-feature, "build this plan", "start the coding",
  "execute the plan", or a well-scoped coding task handed directly.
argument-hint: "[path to plan file — defaults to the newest docs/plans/*.md; or an inline task description]"
---

# Build Feature

Take a plan (usually the file `/plan-feature` produced) and turn it into working, clean, committed
code — phase by phase, continuously, without drive-by refactoring and without silent shortcuts.
You are the coding agent the plan was written for: your job is faithful, high-craft execution, and
honest escalation the moment the plan and the architecture disagree.

The skill runs end-to-end with **exactly two hard stops**:
1. **Architectural concern** — the plan asks for something that fights the existing architecture,
   or you discover mid-build that a design decision is wrong. Stop, explain, recommend, ask.
2. **Red gate** — type-check, tests, or build stay red and you cannot fix them within the phase.
   Stop, report, never push red.

Everything else — phase after phase, commit after commit — flows without pausing for permission.

If the user provided arguments: `$ARGUMENTS` — a plan file path, or an inline task description for
work small enough to skip formal planning. With no arguments, use the newest file in `docs/plans/`
(confirm the filename in your first status line so a wrong guess is caught immediately).

Terminology: "MR" below means merge request / pull request — same rules on GitLab and GitHub.

---

## Process

### Step 1: Read the plan — fully, line by line

Read the **entire** plan file top to bottom: What changes, How, Product framing, Decisions, UI
inventory, every phase block, Risks, and the Recon appendix. Do not skim to the phases — the
Decisions section tells you what was already ruled out, and the appendix is your inherited context.

Build the working checklist (TodoWrite): one item per phase, in plan order, plus a final
"verification sweep" item. If `$ARGUMENTS` was an inline task instead of a plan, write the
checklist from the task and do a compact recon of your own (Step 3 covers it) — but for anything
multi-phase or cross-repo, recommend running `/plan-feature` first rather than improvising.

### Step 2: Branch fresh from main — always

Never build on a stale base and never branch off another feature branch:

1. `git status` — the tree must be clean. If it's dirty with work that isn't yours, **stop and
   report** (another session may own this checkout; a worktree may be the answer). Don't stash
   someone else's work.
2. `git checkout main && git pull` — latest main, no exceptions.
3. `git checkout -b feat/{plan-slug}` — named after the plan. All work lands via a branch and an
   MR (Step 6), even routine single-phase work; the branch costs nothing and protects main from a
   half-done state.
4. If the feature spans repos, repeat per repo, and honor the plan's deploy ordering when you
   later push (e.g. backend before frontend).

### Step 3: Re-ground every phase in the current code

The plan was written against the code as it was; verify it still holds before writing anything:

- Open every file the phases cite and confirm the anchors (`file:line`) still match — main may
  have moved since planning. Trivial drift (line numbers, renames) you absorb silently; real drift
  (the integration point changed shape, the utility was removed) gets recorded as a **deviation**
  and, if it changes the design, routed to the architectural stop.
- Understand how each phase's change *fits* the surrounding architecture: who calls this code, what
  patterns the neighboring code uses (error handling, logging, naming, layering), which existing
  utility already does part of the job. You write code that looks like it was always there.
- If at any point — now or mid-build — a plan decision looks architecturally wrong (wrong layer,
  duplicated concept, a pattern the codebase deliberately avoids), **stop and ask**, with your
  recommended resolution and why. Do not quietly "improve" the plan and do not quietly build what
  you believe is a mistake.

### Step 4: Execute phase by phase

For each phase, in order, without pausing between green phases:

1. **Restate the goal** in one line (your running commentary for the user).
2. **Reuse pass first.** Before writing any new function, component, table, or endpoint, grep for
   the existing thing that already does it or almost does it. Extending an existing utility beats
   a new sibling; a new sibling beats a copy-paste. More code is more maintenance surface — the
   best line of code is the one you didn't write.
3. **Write the code** per the Coding Standards below, touching only what the phase names. Every
   changed line traces to the plan (or to a recorded deviation).
4. **Gate the phase.** Run the type checker and the test suite after each meaningful change and
   always at phase end — **unpiped**, output to a log file, grep the log after; never gate on a
   pipe's exit code. Fix reds inside the phase. New tests must survive CI conditions (bare env,
   mocked heavy imports), not just your machine.
5. **Commit granularly**: one commit per logical change — a phase containing three distinct
   changes gets three commits, never one catch-all. Each message carries the right prefix
   (`feat:` / `fix:` / `refactor:`) and says specifically what that commit does; a reviewer
   reading `git log --oneline` should be able to follow the build step by step. Never commit
   `.env`, secrets, or generated noise.
6. **Phase report** (a few lines, printed as you go): what was built, files touched, gate status,
   plus any entries added to the hack ledger or synchronization ledger this phase.

## Coding Standards

**Architecture & cleanliness**
- **SOLID, applied not recited**: single responsibility per function/module; extend rather than
  modify stable code; depend on the narrow interface the caller needs, not the concrete kitchen
  sink. But no speculative abstraction — an interface with one implementation and no second one
  planned is complexity, not design.
- **DRY with judgment**: extract on the second real duplication, not the first coincidental one.
- **Clean code**: intention-revealing names, small functions, early returns over nesting, no dead
  code left behind by your own changes, comments only for constraints the code can't express.
  Match the existing style of the file even where you'd personally differ.
- **Surgical**: don't refactor, reformat, or "improve" code the phase doesn't touch. If you see
  something broken nearby, note it in the final report — don't fix it uninvited.

**No hacks — and a ledger for the unavoidable ones**
- No workarounds, no swallowed errors, no `any`-casts to silence the compiler, no sleep-based
  fixes, no copy-pasted near-duplicates to dodge a small refactor. Solve root causes.
- If a genuine constraint forces a hack (an upstream bug, a missing API, a deadline decision in
  the plan), you may write it **only flagged**: a `HACK:` comment at the site stating the
  constraint and the clean alternative, plus an entry in the **hack ledger** in the final report.
  An unflagged hack is a defect of this skill's output, even if it works.

**Async & concurrency — async by default, sync by justification**
- I/O never blocks: async/await end to end (`async` handlers, no sync file/network/DB calls on a
  request path). All async calls wrapped in `try`/`catch` (or the project's error pattern).
- Independent work runs **concurrently**: fan out with `Promise.all` / `asyncio.gather` (bounded
  where the list is unbounded), background jobs or a scheduler for slow work instead of making a
  user wait, fire-and-track rather than fire-and-forget.
- Where correctness genuinely requires sequencing — read-modify-write races, ordering guarantees,
  once-only side effects, multi-row invariants — **make it synchronized deliberately**: a DB
  transaction, a unique constraint + upsert, an idempotency key, a queue, a lock. Choose the
  narrowest mechanism that guarantees the invariant.
- Every such spot goes in the **synchronization ledger**: location, the invariant it protects, and
  the mechanism. Synchronization is never the lazy default and never invisible.

**Database design**
- Right types, `NOT NULL` where the domain says so, foreign keys, unique constraints for real
  uniqueness invariants (they're also your race protection), indexes for the query paths the
  feature actually adds — and no speculative indexes.
- Multi-statement writes that must land together run in a transaction.
- Schema changes follow the project's migration convention — check the repo: numbered sequence?
  standalone idempotent SQL applied manually? auto-migrate on deploy? Match it. Regardless of
  convention, keep migrations idempotent and additive-first: add → backfill → cut over → drop
  later; and remember Postgres rejects subqueries in `ALTER COLUMN ... USING`.
- Destructive operations (drop, irreversible rewrite) need an explicit plan line authorizing them;
  otherwise raise it, don't run it.

### Step 5: Verification sweep — before declaring done

After the last phase, verify the whole, not just the parts:

1. **Full gates, unpiped**: complete test suite and type-check to a log file; run the production
   build for frontend work (type-check and unit tests both miss framework-boundary breaks — e.g.
   React Server Component serialization — that only the build catches).
2. **Consumer audit**: for every surface you changed, grep for **all** its consumers and confirm
   each still works — one wired path and one silently broken path reads as "works" until a user
   hits it. This is a required pass, not a spot check.
3. **Drive it**: exercise the feature's main flow end to end — the plan's success criteria, not
   just green tests.
4. **Plan reconciliation**: walk the plan's phase list and UI inventory once more; anything
   skipped, deferred, or deviated is stated in the report — never silently dropped.

### Step 6: Push, raise the MR, and report

Push the feature branch **only when every gate is green** — if anything is red, stop at the report
and say so plainly. If tests were red before your work began, don't push past them without asking.

Then **raise the MR — always, without asking** (finishing the plan is the trigger; invoking the
skill authorized it). Use `glab mr create` on GitLab or `gh pr create` on GitHub, targeting the
plan's base branch (usually `main`); for cross-repo work, one MR per repo, each description naming
the deploy order and linking its sibling. The description follows this exact structure — all four
sections, every time:

```
## Problem
{What problem does this MR solve — the user/product problem it exists for, not "implemented the
plan". 1–3 sentences a reviewer can judge the whole MR against.}

## Solution
{The approach chosen to solve it, in plain terms — and what was deliberately NOT done (cut or
deferred, per the plan) so a reviewer doesn't flag it as forgotten.}

## Technical details
{What was actually done, concretely: key files/modules touched, schema changes and how to apply
them, new endpoints/jobs/components, entries from the synchronization ledger, deploy ordering if
cross-repo.}

## Review notes
{What reviewers should specifically look at or know before reviewing: the riskiest change, any
flagged hack and the constraint that forced it, close-call decisions where you want a second
opinion. Never "nothing in particular" — every MR has a riskiest part; name it.}
```

Then report:

```
## Build Report — {plan name}

Branch: feat/{slug} ({N} commits) · pushed ✅ · MR: !{N} {url} → main
Gates: type-check ✅ · tests ✅ ({n} passed) · build ✅ · consumer audit ✅

### Phases
1. {phase} — done ({commit sha}) {· deviation: …}
2. …

### Hack ledger
{Each flagged hack: file:line — constraint — clean alternative. Or "empty" — the goal.}

### Synchronization ledger
{Each deliberate sync point: file:line — invariant — mechanism. "Empty" only if truly none.}

### Deviations from plan
{Every place reality differed and what you did. Or "none".}

### Follow-ups
{Migrations to apply + order · deploy ordering across repos · anything noticed but deliberately
not touched.}
```

Close with the concrete next action: which migration to apply and where, deploy order if
cross-repo, and that the MR awaits review (`/address-review` picks up from there once reviewers
comment). Raising the MR is the skill's last outward action — don't merge it, deploy, or apply
migrations yourself.

---

## Rules

- **Fresh main, always.** Every feature branch starts from just-pulled main. Building on a stale
  base or a sibling branch is a setup error, not a shortcut.
- **The plan is read whole, then followed — or challenged, never silently edited.** Faithful
  execution plus recorded deviations plus the architectural stop. The one thing you may not do is
  build something other than the plan without saying so.
- **Two hard stops only.** Architectural concern → ask with a recommendation. Red gate → report,
  never push. Everything else runs continuously; the user can interrupt anytime.
- **Reuse before write.** Proposing new code requires the grep that found nothing to reuse. More
  code is more maintenance surface.
- **Clean code or flagged code — nothing in between.** Root-cause fixes by default; a forced hack
  exists only with its `HACK:` comment and ledger entry.
- **Async by default, synchronized by justification.** Blocking I/O on a hot path is a bug;
  unprotected read-modify-write is a worse one. Both ledgers ship with the report — an empty sync
  ledger on a feature with shared writes is a smell, question it.
- **Every changed line traces to the plan.** No drive-by refactors, no adjacent "improvements", no
  style rewrites. Orphans your change created, you clean; pre-existing mess you report.
- **Granular commits, and the MR is the finish line.** One commit per logical change, each message
  specific enough that the log reads as the build's story. A finished plan without its MR isn't
  finished — push and raise it without asking, all four description sections (Problem / Solution /
  Technical details / Review notes) filled honestly, none skipped, none boilerplate.
- **Gates are unpiped and total.** Type-check + tests per change, full suite + build + consumer
  grep audit before done, logs greppable. Pass-locally isn't pass-CI — mock heavy imports and
  check the bare-env story for new tests.
- **DB changes are idempotent and additive-first**, per the project's migration convention:
  transaction where atomicity matters, constraints as invariants, no destructive statement the
  plan didn't explicitly authorize.
- **Done means verified.** The feature was driven end to end against the plan's success criteria,
  every consumer surface was grepped, and the report states what was NOT done as clearly as what
  was.
