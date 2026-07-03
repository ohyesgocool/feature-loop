---
name: address-review
description: |
  Triage external code-review feedback on a merge request and resolve it end-to-end, automatically:
  decide which comments are valid against the actual diff (line by line), explain why each valid one
  was missed during coding, defend the correct decisions reviewers got wrong, then implement the
  fixes, push, and reply to every comment thread — all in one pass, without stopping to ask.
  Built for the loop where an AI coding agent writes the code, you raise an MR, and reviewer
  agents comment with less project context than the author has.
  Invoking the skill is the go-ahead for the whole loop (triage → implement → push → reply); the only
  hard stop is a red build. Valid comments get marked done on the MR, invalid ones explained.
  Use when: addressing MR/PR review comments, "address the review", "incorporate review feedback",
  triaging reviewer-bot comments, deciding which review comments are worth acting on.
argument-hint: "[MR number or URL] — optional; defaults to the current branch's open MR(s)"
---

# Address Review

Take review comments from the agents that reviewed an MR (or any reviewer), check each
one against the **actual code you changed**, and decide honestly what holds. The reviewers have less
context than you do — your job is to use your fuller context to separate real findings from
context-gaps, **without** hiding behind "they didn't have context" to wave away valid bugs.

This skill runs the **whole loop end-to-end, without stopping to ask** — invoking it is the
go-ahead for all four stages:
1. **Triage report** — every comment addressed in a fixed format: verdict, why, and for valid ones, **why you missed it while coding** (this is the point of the skill — be specific and honest).
2. **Fix plan** — what to change, in priority order, each item tracing back to a comment.
3. **Implement** — code every VALID + PARTIAL comment, per the house rules, running type-check + tests.
4. **Push + reply** — push the commits, then post each verdict back to its MR thread (fixed ones marked done, invalid ones explained) and resolve the fixed/dismissed threads.

Print the triage report and fix plan as you go so the user can see the reasoning (and interrupt if
they disagree) — but **do not pause for permission**; flow straight through to push + reply. The one
hard stop is a **safety abort**: if type-check or tests stay red, do not push — report and wait.

If the user provided arguments: `$ARGUMENTS` — see Step 1 for how to interpret them. With no
arguments, auto-detect the current branch's open MR(s) (Step 1).

> **Forge note.** This skill is written for GitLab (`glab`). On GitHub, the process is identical —
> substitute: `glab mr view --comments` → `gh pr view --comments`; the discussions API →
> `gh api repos/:owner/:repo/pulls/<n>/comments` (review comments) and `.../issues/<n>/comments`;
> `glab mr diff` → `gh pr diff`; replies → `gh api .../pulls/<n>/comments -f body=... -F in_reply_to=<comment_id>`;
> resolving threads → GraphQL `resolveReviewThread`.

## Live state

**Current branch:**
!`git branch --show-current`

(Open MRs for this branch are fetched in Step 1 via `glab` through the Bash tool —
not at skill-load time, since load-time `!`-commands can't prompt for approval.)

---

## Process

Follow the steps in order. Do not skip Step 3 — line-by-line correlation is the whole job.

### Step 1: Get the review comments

Interpret `$ARGUMENTS` and the conversation:

- **MR ref** — `$ARGUMENTS` is an MR number (`42`, `!42`, `#42`) or an MR URL → **MR mode**. Pull the comments with `glab`:
  - `glab mr view <id> --comments` — human-readable MR details + discussion threads
  - **Always also pull structured discussions** — they carry the thread IDs you need to reply in-thread later (Step 8):
    `glab api "projects/:id/merge_requests/<iid>/discussions"` — `glab` expands `:id` to the current repo. Each discussion has an `id` (the **thread ID**); each note under it carries `id`, `author.username`, `body`, and (for inline comments) `position.new_path` + `position.new_line`.
- **Pasted text** — `$ARGUMENTS` (or a recent message) contains the raw review comments → **paste mode**. Use it directly. No thread IDs are available, so Step 8 can only post one summary note, not in-thread replies.
- **Empty (the default — auto-detect, don't ask)** — resolve the current branch's open MR(s) yourself and pull each one's comments. The Live-state block already lists them; else `glab mr list --source-branch "$(git branch --show-current)"`. Process **every** open MR for the branch (usually one) in MR mode. Only if there is genuinely no open MR *and* no pasted comments anywhere in the conversation do you fall back to asking for an MR number or pasted text.

Normalize whatever you got into a flat list of **discrete comments**. One reviewer note may contain several distinct points — split them. For each, record:

- **Reviewer** (attribute every comment to its author)
- **Location** — `path/to/file:line` if given, else the area it refers to, else "general"
- **The point** — what the reviewer is actually claiming or asking for
- **Thread ID** (MR mode) — the `discussion.id` this comment belongs to, so Step 8 can reply to the right thread. Split points from one reviewer note share that note's thread ID.

Resolve threads: if a comment was already replied to or marked resolved on the MR, note it and don't re-litigate.

### Step 2: Reconstruct the diff (the ground truth)

You judge comments against the code, not against the reviewer's description of it. Get the real diff:

1. Detect the base branch: `glab mr view <id> -F json 2>/dev/null` → `target_branch`; else `glab repo view -F json 2>/dev/null` → `default_branch`; else `git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||'`; else `main`.
2. Get the diff:
   - **MR mode:** `glab mr diff <id>`
   - **Paste mode:** if the current branch has an open MR, use `glab mr diff <id>`; otherwise `git fetch origin <base> --quiet && git diff origin/<base>...HEAD`.
3. List the changed files and read each one **at the changed regions** (use Read on the files, not just the diff hunks — you need surrounding context, callers, and types to judge correctly).

If the review references code that isn't in this diff, say so — the comment may be about pre-existing code or a stale revision.

### Step 3: Correlate each comment to the code, line by line

For **every** comment, go to the cited location in the real diff and read what the code actually does there. Do not evaluate from the reviewer's summary alone. For each comment, establish:

- **What the code at that line actually does** — in your own words, grounded in the file you just read.
- **Whether the reviewer's claim matches reality** — is the bug/issue actually present? Does the suggested change actually improve it? Would it break something the reviewer can't see?

This is where your context advantage matters: you can see the caller that already validates the input, the type that makes the null case impossible, the test that covers it, the intentional reason for the "weird" line. Use it. But see the anti-rationalization rule below.

### Step 4: Verdict + root cause per comment

Assign each comment exactly one verdict:

- **VALID** — a real issue. The code is wrong, risky, unclear, or worse than an easy alternative, and the reviewer is right.
- **PARTIAL** — right instinct, wrong specifics: correct that something's off but mis-diagnoses it, overstates severity, or proposes a fix that's wrong while the underlying concern is real.
- **INVALID** — the comment is wrong. The code is correct as-is, almost always because the reviewer lacked context you have.

For **VALID** and **PARTIAL**, fill in **Why I missed it** — the single most important field. Be specific and honest. Pick the closest blind-spot category and explain the actual cause:

| Category | Typical cause |
|---|---|
| Edge case | empty / null / boundary / single-element / unicode case not considered |
| Error path | happy path coded; failure/exception branch skipped |
| Trust boundary | assumed an input was validated/sanitized upstream when it wasn't |
| State / concurrency | race, ordering, stale read, shared mutable state |
| Invariant | relied on an assumption the code doesn't actually guarantee |
| Performance | N+1, redundant work, work inside a loop, unbounded growth |
| Consistency | copy-paste drift, one branch updated and not its twin |
| Spec misread | built what was literal, not what was intended |
| Test gap | shipped without a test that would have caught this |

"Why I missed it" must name a real cause, not "oversight" or "I should have been more careful." If the coding happened earlier in this same conversation, recall the actual reasoning; if not, reconstruct the most likely cause from the shape of the change. State which you're doing if it's a reconstruction.

For **INVALID** and **PARTIAL**, fill in **What the reviewer couldn't see** — the specific context that makes them wrong (the validating caller, the type guarantee, the intentional tradeoff, the covering test). Cite it concretely.

Assign **Severity** (High / Medium / Low) to VALID and PARTIAL comments — drives plan ordering.

### Step 5: Present the triage report

Output in exactly this shape:

```
## Review Triage — {MR !N "title" | current branch}

Reviewers: {names} · {X} comments · Diff: {N} files, +{a}/-{b}
Valid: {n}  ·  Partial: {m}  ·  Invalid: {k}    →  fixing {n+m}, dismissing {k}
By reviewer: {reviewer A} {valid}/{total} valid · {reviewer B} {valid}/{total} valid

---

### [C1] VALID · {one-line title}
- **Reviewer / location:** {reviewer} · `app/api/example.py:142`
- **Comment:** "{verbatim if short, else tight summary}"
- **The code in question:** {what those lines actually do — your read of the real diff}
- **Verdict & why:** Valid — {grounded reasoning that cites the line}
- **Why I missed it:** {category} — {specific, honest cause}
- **Severity:** High

### [C2] INVALID · {one-line title}
- **Reviewer / location:** {reviewer} · `app/api/example.py:88`
- **Comment:** "{…}"
- **The code in question:** {…}
- **Verdict & why:** Invalid — {why the code is correct as written}
- **What the reviewer couldn't see:** {the specific context they lacked — cite it}

### [C3] PARTIAL · {one-line title}
- **Reviewer / location:** …
- **Comment:** "{…}"
- **The code in question:** {…}
- **Verdict & why:** Partial — the concern is real but {the mis-diagnosis / wrong fix}
- **Why I missed it:** {category} — {cause}
- **What the reviewer couldn't see:** {the part they got wrong}
- **Severity:** Medium
```

After the comments, add a short learning signal:

```
### Blind-spot patterns
{Only if a category repeats. e.g. "3 of 4 valid findings were error-path omissions —
I keep coding the happy path and skipping failure branches."}
```

### Step 6: Fix plan

List the work for every VALID and PARTIAL comment, ordered by severity (High → Low). Each item maps to a comment ID and follows the goal-driven style — change + verify:

```
## Fix Plan

1. [C1] {what to change} — `app/api/example.py:142`
   Approach: {1–2 lines}
   Verify: {the test or check that proves it's fixed}
2. [C3] {…}
   Approach: {…}
   Verify: {…}

Dismissed (no code change):
- [C2] Invalid — reviewer lacked {context}. {If the confusion signals unclear code, note the optional readability fix here.}
```

If a "wrong" comment reveals that the code is genuinely confusing (the reviewer misread it because it reads badly), surface an optional clarity fix even though the logic is correct — a reviewer's confusion is a real signal.

### Step 7: Implement the plan (automatically — no prompt)

Do **not** stop to ask. Invoking the skill is the go-ahead, so go straight from the fix plan into implementing it. Code **every VALID and PARTIAL** comment (High → Low). For a **PARTIAL**, implement the *corrected* version — the real fix for the real concern — not the reviewer's exact wrong suggestion. Leave **INVALID** comments uncoded; they get a dismissal reply in Step 8.

Follow the project's house rules (`CLAUDE.md` / `AGENTS.md` / equivalent): surgical changes only (every changed line traces to a comment), match the project's error-handling and style patterns, **run the type checker and tests after each change**, and make a **separate commit per comment** with a `fix:` / `refactor:` prefix that references the comment (e.g. `fix: guard empty item list (addresses review C1)`).

**Update the blind-spot ledger.** After the fixes, append this run's lessons to `docs/reviews/blind-spots.md` (create it on first use): one line per VALID/PARTIAL finding —
`- [<date> <MR> <comment-id>] <Category> — <the lesson, phrased as a rule for next time>` — and refresh the one-line category tally at the top of the file. Phrase lessons as forward-looking rules ("every new I/O path gets its failure branch coded before the happy path ships"), not as descriptions of the mistake. Keep the file small enough to read whole: past ~50 lessons, distill the oldest into category-level rules. Commit it as its own `docs:` commit alongside the fixes. This ledger is what `/build-feature` reads before coding — it is the loop's memory; a lesson that isn't written down gets relearned the expensive way.

**Safety abort — the one hard stop.** A green build gates the push. After the changes, the type checker and the full test suite must pass. If they stay red and you can't fix them, **STOP**: report the failure plus the commits made so far, and do **not** push or reply. This is the only thing that interrupts the automatic flow.

**Honest deferral over a reckless fix.** If a comment's fix genuinely can't be made blind — it needs a product decision, a test harness that doesn't exist, or a change far larger than the comment implies — don't force a bad edit. Leave it as a **deferred** item; Step 8 replies "acknowledged, tracking separately." Note any deferrals in your running output.

When the build is green, flow straight into Step 8 — push, then reply. No confirmation prompt.

### Step 8: Push, then reply to each comment on the MR (automatic)

Once the build is green, push the commits and post the verdict back to each comment's own thread so the reviewer sees the outcome in context — valid ones marked done, invalid ones explained. This closes the loop on the MR, not just in a local report.

**When this runs:** automatically, right after Step 7's green build — no confirmation prompt (the skill invocation already authorized it). Order matters: **`git push` first**, so the commit SHAs you reference already exist on the remote, **then** the replies.

**Which comments get a reply:** every comment triaged this run — **VALID/PARTIAL** → fixed (cite the commit), **INVALID** → dismissed with the reason, **deferred** → acknowledged. Keep each reply short, factual, and non-defensive — you're recording a decision, not winning an argument. Show the consolidated block (comment ID → thread → text you posted) **after** posting, as a record — not as a gate before it.

Reply templates — derive the text from the triage fields (Verdict & why, What the reviewer couldn't see, the commit):

- **VALID / PARTIAL (fixed):** `✅ Fixed in <short-sha> — <one line of what changed>.` For PARTIAL, name the part you adjusted vs. the reviewer's exact suggestion.
- **INVALID:** `Not changing — <why the code is correct as written>. <the specific context the reviewer couldn't see>.`
- **VALID but deferred (not done this run):** `Acknowledged, tracking separately — <why it's out of scope here>.`

Fill `<short-sha>` from the per-comment commit made in Step 7 (`git log --oneline` for that commit, or `git rev-parse --short HEAD` right after committing it).

**Post the reply (MR mode, in-thread):**
```
glab api "projects/:id/merge_requests/<iid>/discussions/<discussion_id>/notes" -X POST -f body="<reply text>"
```
(Shorthand, if available: `glab mr note create <id> --reply <discussion_id> -m "<reply text>"`.)

Once a reply is posted, optionally resolve the thread — do this for **fixed and dismissed** comments, not for deferred ones:
```
glab api "projects/:id/merge_requests/<iid>/discussions/<discussion_id>" -X PUT -f resolved=true
```

**Paste mode (no thread IDs):** you can't target individual threads. Offer to post a single summary note instead — `glab mr note create <id> -m "<summary of verdicts>"` — or skip. Don't claim you replied in-thread when you couldn't.

**Don't double-post.** On a re-run, skip any thread that already carries your reply (check the discussion's notes first), and don't touch threads that were already resolved before this run.

Close by reporting what you did: comment ID → posted / resolved / skipped, so the user can see the loop is closed.

---

## Rules

- **The diff is the source of truth, not the reviewer's words and not your memory.** Read the actual changed code in Step 3 before assigning any verdict.
- **Burden of proof is on dismissal.** Default to taking a finding seriously. Mark INVALID only when you can point to the specific code or context that makes the reviewer wrong. "I had more context" is not an argument — name *what* context and *why* it makes the code correct.
- **Do not rationalize away valid bugs.** You wrote this code; the pull toward defending it is real. If a finding is valid, say so plainly and own the blind spot. A defensive triage that protects your own code is worse than useless.
- **"Why I missed it" is the deliverable.** It must be specific, honest, and name a real cause. Generic self-blame ("should have been more careful") is a failed answer.
- **Feed the ledger.** Every valid finding's "why I missed it" becomes a written lesson in `docs/reviews/blind-spots.md` — the loop's learning is cumulative, not per-run. A triage that fixes the code but skips the ledger fixed one MR and taught nothing.
- **A wrong comment can still be a real signal.** If a reviewer misread correct code, ask whether the code reads badly. Surface the clarity fix as optional.
- **Attribute every comment** to its reviewer and track per-reviewer hit rate — over time it tells the user which reviewer to weight.
- **Split compound comments.** One reviewer note with three points is three entries.
- **Don't re-litigate resolved threads.** If the MR shows a comment already addressed or resolved, note it and move on.
- **Replies are outward-facing, but invoking the skill authorizes them.** Posting to MR threads is visible to reviewers and hard to take back — so post only as the final step of *this* run, only for comments triaged this run, only after a green build, and **never double-post** (skip any thread that already carries your reply; don't touch threads resolved before this run). Running `/address-review` is the standing go-ahead for the push + replies — you don't ask again mid-run.
- **Match the reply to the verdict, honestly.** A fixed comment says fixed (with the commit), a dismissed one says why with the context the reviewer lacked. Don't soften a real dismissal into faux-agreement, and don't claim "done" for anything you didn't actually change.
- **Run end-to-end; the only hard stop is a red build.** Don't pause for an A/B/C/D choice or a push/reply confirmation — triage, implement, push, and reply in one pass. The single exception: if type-check/tests stay red after the fixes, stop before pushing and report. The user can always interrupt mid-run to redirect.
