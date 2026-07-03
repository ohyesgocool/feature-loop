---
name: ship-feature
description: |
  The full loop, one command: dump a feature brief and it plans (/plan-feature), builds and
  raises the MR (/build-feature), gets independent external-AI reviews (/mr-review), triages
  and fixes the findings (/address-review), then re-reviews — looping until the review comes
  back clean — and exits at a merge gate: --merge=ask (default) stops and asks you to approve
  the merge, --merge=auto merges when converged + CI green, --merge=never just reports.
  Safeguards: max review rounds (default 3), no-progress detection (same finding surviving two
  rounds stops the loop instead of thrashing), and honest convergence (threads are never
  resolved just to make the loop exit). Human touchpoints: the planner's clarifying questions
  at the start (skippable with --autonomous), a genuine architectural concern, a red build,
  and the merge gate itself.
  Use when: "ship this feature", "run the whole loop", "build and merge this", dumping a
  feature brief you want taken all the way to a merge-ready (or merged) MR.
argument-hint: "[feature brief or plan path] [--merge=ask|auto|never] [--max-rounds=N] [--autonomous]"
---

# Ship Feature

Run the whole feature lifecycle as one supervised loop:

```
brief ─→ PLAN ─→ BUILD (+MR) ─→ REVIEW ─→ ADDRESS ─→ converged? ──no──┐
                                   ▲                      │            │
                                   └──────────────────────┴── round+1 ─┘
                                                          │yes
                                                          ▼
                                                     MERGE GATE
                                              ask / auto / never
```

You are the loop supervisor. The stages themselves are the four skills — invoke them, don't
re-implement them. Your job is sequencing, state, convergence judgment, and honest exits.

## Flags

Parse from `$ARGUMENTS` (everything that isn't a flag is the brief / plan path):

| Flag | Default | Meaning |
|---|---|---|
| `--merge=ask\|auto\|never` | `ask` (or `SHIP_MERGE_MODE` env) | what happens when the loop converges |
| `--max-rounds=N` | `3` | maximum review→address rounds before an honest non-converged exit |
| `--autonomous` | off | planner answers its own clarifying questions (recorded as Decisions) instead of asking |
| `--plan=path` | — | skip planning; start from an existing plan file |

State the resolved flags in your first status line so a misparse is caught immediately.
Track the loop in TodoWrite: plan · build · round 1 (review, address) · … · merge gate.

## Stages

### Stage 1 — Plan

Invoke `/plan-feature` with the brief (skip if `--plan` given and the file exists). This is the
loop's **one designed human touchpoint at the start**: answer the planner's blocking questions
now, then walk away. With `--autonomous`, instruct the planner to convert every would-be
question into a recorded default in the Decisions section — the user reviews decisions in the
MR instead of answering questions up front.

### Stage 2 — Build

Invoke `/build-feature` with the plan. It branches fresh from main, executes the phases, and
ends with the branch pushed and the MR raised (Problem / Solution / Technical details / Review
notes). Capture the MR reference — every later stage uses it.

Its two hard stops are real and pass through the loop: an **architectural concern** pauses
everything for the user; a **red build** ends the run with a report. Don't route around either.

### Stage 3 — Review round *r*

Invoke `/mr-review` on the MR. External reviewers (whichever of OpenAI / Grok are configured)
post their briefs as MR comments. Note each reviewer's verdict line.

### Stage 4 — Address round *r*

Invoke `/address-review` on the MR. It triages every open comment against the real diff, fixes
VALID/PARTIAL findings, dismisses INVALID ones with reasons, pushes, and replies to every
thread. Record the round's tally: valid / partial / invalid, and the list of finding titles.

### Convergence check — after every round

The loop **converges** when a full round shows the work is clean:

1. The latest review round produced **zero VALID or PARTIAL findings** after triage
   (dismissed-with-reasons findings don't block), **or** every reviewer's verdict is
   "No blocking issues"; and
2. every review thread on the MR has been replied to (resolved or explicitly deferred with
   an owner); and
3. the MR's head pipeline is **green** (poll `glab api "projects/:id/merge_requests/<iid>"` →
   `.head_pipeline.status` until it settles; a failed pipeline is a red build → hard stop).

Not converged → increment the round and go back to Stage 3, subject to the safeguards below.

**Safeguards against thrash:**
- **Max rounds** (`--max-rounds`, default 3). External reviewers can generate low-value
  findings forever; three rounds is where returns go negative. Hitting the cap is an honest
  exit: report what's still open and stop — never merge past it silently.
- **No-progress detection.** If a finding you fixed in round *r* comes back in round *r+1*
  (same file/issue), or the valid-finding count doesn't drop between rounds, stop and escalate
  to the user with both rounds' tallies — the loop is oscillating, and a human call beats a
  fourth attempt.
- **Honest convergence only.** Never resolve a thread without a fix or a stated dismissal,
  never soften a triage verdict, never skip `/mr-review` "because it'll probably pass" — a
  loop that games its own exit criteria is worse than no loop.
- **Cost awareness.** Each round spends reviewer API tokens. Note the round count in the final
  report so the user sees what convergence cost.

### Merge gate — on convergence

First run the readiness checklist (all must hold, whatever the merge mode):

- Converged per the check above (findings clean + threads closed + CI green).
- The MR is mergeable: no conflicts with the target branch (`.has_conflicts == false`,
  `.merge_status`). If the target moved and there are conflicts, stop and report — resolving a
  conflicted rebase is a human-supervised task, not a loop step.
- Surface the build report's follow-ups (manual migrations to apply, cross-repo deploy order)
  — merging does not make those happen; they go in the final report either way.

Then act per the flag:

- **`ask` (default):** print the Ship Report (below) ending with **READY TO MERGE** and the
  exact merge command — then stop. The user merges, or replies "merge it" and you run the
  command. This is the safe default; auto-merge is opt-in per invocation.
- **`auto`:** merge it: `glab mr merge <iid> --remove-source-branch` (GitHub:
  `gh pr merge <n> --delete-branch`, honoring the repo's merge method). Verify the MR state is
  `merged`, then report. If the merge API refuses (permissions, approvals required, protected
  branch rules), report the refusal verbatim — don't try to bypass review requirements.
- **`never`:** report and end with the MR open — useful when merging is someone else's call.

## Ship Report (final output, always)

```
## Ship Report — {feature} (MR !{N})

Outcome: MERGED ✅ | READY TO MERGE (awaiting approval) | STOPPED ({reason})
Loop: plan {✅/skipped} · build ✅ ({n} commits) · {r} review round(s)

Rounds:
  1. {v} valid / {p} partial / {i} invalid → fixed {v+p} · verdicts: {…}
  2. …

Ledgers: hacks {n} · sync points {n} (see MR description / build report)
Follow-ups: {manual migrations + order · deploy sequence · deferred items · none}
```

## Rules

- **Orchestrate, don't re-implement.** Each stage is its skill, with its skill's rules. If a
  stage skill is missing, stop and say which one — no inline approximations of it.
- **The sub-skills' hard stops are the loop's hard stops.** Architectural concern, red build,
  red pipeline, merge conflicts: all surface to the user immediately. The loop makes routine
  iteration autonomous; it never makes judgment calls autonomous.
- **Merging is gated by the flag, and `ask` is the default.** `auto` must have been explicitly
  requested this invocation (or via `SHIP_MERGE_MODE`) — never assume it, never upgrade to it
  mid-run, and record in the report which mode ran.
- **Exits are honest.** Converged, max-rounds, no-progress, or hard stop — the report names
  which, with what's still open. "Merged" is only claimed after verifying the MR state.
- **One feature per loop.** Don't batch unrelated briefs into one run; queue them as separate
  invocations so each MR stays reviewable.
- **Keep round outputs terse.** The sub-skills print their own detail; the loop supervisor
  adds one tally line per stage, and the Ship Report at the end.
