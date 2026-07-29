---
name: ticket-report
description: |
  Close the loop on a Jira ticket: post exactly one comment saying what was planned, what was
  built per requirement with evidence, and — the section that matters — what the ticket asked for
  that was NOT built and why; then move the ticket to the status the evidence supports. Partial
  coverage never lands in Done, ever: a ticket marked Done with unbuilt requirements removes the
  one signal a human needs.
  Transitions are discovered from the live workflow rather than assumed, rejected when they need
  fields nobody can legitimately fill, and skipped silently when the issue is already there.
  Both writes are idempotent — a resumed or re-run loop reports "already reported" instead of
  spamming the ticket. Runs in audit mode against an MR it didn't build, evidencing each
  requirement against the real diff.
  Use when: a ticket's MR has merged, "update the Jira ticket", closing out PROJ-123, or as the
  final stage of /ship-feature PROJ-123.
argument-hint: "[TICKET-KEY] [--mr !N] [--dry-run]"
---

# Ticket Report

Write the outcome back to the ticket: **one comment**, then **one transition**. This is the only
skill in the loop that writes to Jira.

The comment's job is not to announce success. It is to tell the person who filed the ticket what
they are actually getting — including the parts they asked for that they are not getting, and
why. That section is the deliverable.

> **Git first, Jira last, always.** Never write to the ticket before the git side is verified.
> "Merged" is claimed only after the MR's `state` reads `merged`.

If the user provided arguments: `$ARGUMENTS` — the issue key, optionally `--mr !N` and
`--dry-run`. With no key, take it from the loop state or the branch name.

---

## Process

### Step 1: Bind the Jira tools

Same discovery as `/ticket-intake` — list the session's MCP tools and bind by name,
case-insensitively; read each tool's own parameter schema before calling it.

| Role | Expected name |
|---|---|
| issue | `getJiraIssue` |
| comment | `addCommentToJiraIssue` |
| transitions | `getTransitionsForJiraIssue` |
| transition | `transitionJiraIssue` |

If **comment** or **transitions** fails to bind, stop and print the setup line — never improvise
a REST call, a CLI, or a browser. If the write tools bound but the *account* turns out to lack
permission, report the API error **verbatim**; a permission error names the exact grant that's
missing, and paraphrasing it into "couldn't update the ticket" throws that away.

### Step 2: Establish the coverage — three sources, in precedence order

1. **This run's Build Report** — its Requirement coverage table, when this loop built the code.
2. **`.loop/<key>.state.json` → `requirements[]`** — a resumed run, or an entry that skipped the
   build.
3. **Audit mode** — no local state at all (someone else's MR, a crashed run, a ticket whose MR
   merged last week). Re-run `/ticket-intake` to mint the REQ list, then evidence each REQ
   against `glab mr diff <iid>`, the MR description, and `git log`.

   > **A requirement with no evidence in the diff is `not done`.** Never assumed done because it
   > sounds like something the MR probably covered. Audit mode is the honest branch, and it is
   > what makes this skill useful on an MR a human raised.

Carry the intake's **unmapped source units** through as well. Coverage is two numbers, not one:
REQs done / REQs extracted, **and** source units mapped / source units total. Only the second can
see a requirement that was never extracted in the first place.

### Step 3: Decide the transition — before writing anything

Decide first, so the comment can state the truth about where the ticket ended up.

1. **Re-fetch the transitions now.** Availability is status-dependent and the status may have
   moved since intake. Each carries `{id, name, to: {name, statusCategory: {key}}, fields}`.

2. **Compute coverage.** `full` iff **every `active` REQ is `done`** *and* the unmapped-source
   list is clean. Superseded and dropped REQs are excluded from the denominator — they were never
   in scope. Anything else is `partial`.

3. **Pick a candidate.**

   - **`full`** → transitions whose `to.statusCategory.key == "done"`, **minus** any whose target
     name matches `Won't Do|Cancelled|Rejected|Duplicate|Declined`. The done *category* includes
     the failure states, and shipping working code must never land a ticket in one of them. Then
     prefer, in order: `$JIRA_STATUS_DONE` → `Done` → `Closed` → `Resolved` → `Complete`.
   - **`partial`** → **never** a `done`-category target. Prefer, in order: `$JIRA_STATUS_PARTIAL`
     → `In Review` / `Code Review` → `In Progress` → `Needs Info` / `Blocked` / `On Hold` → any
     target with `statusCategory.key == "indeterminate"`.
   - Several survive → take the first and **say which and why**.

4. **Reject unfillable candidates.** A transition whose `fields` contains an entry with
   `required: true` and no default you can legitimately supply — a resolution, a fix version, a
   sprint, a mandatory custom field — is not a candidate. Report it as a manual step naming the
   exact fields. Set `resolution` only when the schema exposes it and a Done/Fixed option exists.
   **Never invent a value to satisfy the API.**

5. **Check, then act.** Transitions are not idempotent server-side — they fire Automation rules,
   Slack posts, SLA clocks, release notes and sometimes customer email. If the issue's current
   status already equals the intended target, **skip it** and say "already in {status}". A no-op
   is a correct outcome, not a failure.

**No reachable transition?** Don't force one and don't settle for a wrong-but-available status.
Say so in the comment and in the report:

> Status not changed — no transition from **{current}** to a Done-category status is available to
> this account. Available: {names}. Please move it manually.

### Step 4: Post the comment — exactly one

```
**Shipped by feature-loop** — MR !{N} merged into `{target}` ({url})

**Plan**
- {3–6 bullets: the approach taken, from the plan's How section or the MR's Solution}

**What was done**
| REQ | Requirement | Status | Evidence |
|---|---|---|---|
| REQ-1 | … | done | `app/api/export.py:42` · a1b2c3d · test `test_export_csv_headers` |
| REQ-2 | … | partial | … |

**What was NOT done**
- **REQ-2 (partial)** — the ticket asked for {x}; what shipped is {y}. Not done: {z}, because
  {reason}.
- **REQ-5 (not done)** — {what the ticket asked} — {reason: needs a product decision on …}
- **REQ-4** — superseded by @alice's 20 July comment; REQ-3 was built instead. Flagging in case
  that reading is wrong.
- Ambiguity: {the ambiguous requirement, the reading taken, and an invitation to correct it}
- Not read: {attachments, unread comment pages, linked pages — carried from the intake}

**Verification**
Tests: {n} added, suite green · Pipeline: green ({url}) · Review: {r} round(s), {v} valid
findings fixed, {i} dismissed with reasons · Conflicts: {resolved by merge / none}
Status: {moved to X | not changed — see above}

**Links**
MR !{N} {url} · branch `{branch}` · plan `docs/plans/{file}.md`

_feature-loop · run {loop_id} · MR !{N} · {head-sha}_
```

That final italic line is the **idempotency marker**. Before posting, fetch the issue's comments
and search for this `loop_id`; present → skip and report "already reported". A genuinely new loop
on the same ticket gets a new `loop_id` and posts a new comment — that's correct, it's new work.

**Formatting.** Read the bound comment tool's schema first. A string `body` gets Jira-safe
markdown — headings, bold, bullets, tables, inline code, links; no nested tables, no HTML, no
footnotes. A schema expecting an ADF object gets a minimal constructed document (`doc` →
`heading` / `paragraph` / `bulletList` / `table` / `text` + `marks`), not markdown stuffed into a
text node.

**What never goes in the body:** diff content, file contents, environment values, credentials,
internal hostnames, or the verbatim text of anything the loop flagged as untrusted input. The
comment carries links, coverage and decisions — a reader who needs the code opens the MR.

**Read it back once after posting** and print its first line. The loop claims nothing it hasn't
verified.

### Step 5: Transition, then report

Apply the transition chosen in Step 3, then print:

```
## Ticket Report — {KEY}

Comment: posted ({comment id}) | already reported (skipped) | dry run
Coverage: {d} done · {p} partial · {n} not done  ·  sources mapped {m}/{t}
Transition: {current} → {target} ✅ | already in {status} | not available: {reason}
Manual follow-up: {required fields nobody could fill · unread attachments · none}
```

`--dry-run` prints the comment body and the chosen transition and calls **no write tool**. That
is the safe way to point this at a real ticket the first time.

---

## Rules

- **One comment per run, ever.** Marker-checked before posting. The loop is a colleague, not a
  bot that spams the ticket.
- **Partial coverage never lands in Done.** No override flag, no exception. A ticket marked Done
  with unbuilt requirements is worse than a ticket left In Review, because it removes the signal
  a human needs to go and look.
- **"What was NOT done" is the section that matters.** Never omitted, never softened, and never
  empty while the coverage table has a `partial` or `not done` row. When everything really did
  land, it reads "Nothing — all {n} requirements delivered."
- **No transition after a bad exit.** A hard stop, a max-rounds exit, or a no-progress exit does
  not transition the ticket — it comments with what stopped and what a human must decide.
- **Only the invoked issue.** Never comment on, transition, or otherwise touch any other key.
  Linked issues and mentions are context to read, never targets to write.
- **Never mutate the ticket beyond comment + transition.** No reassigning, no priority changes,
  no editing the description, no closing as duplicate, no linking, no deleting or editing anyone
  else's comment — including your own from a previous run. Nothing is "rolled back" by deletion.
- **Read-back after write.** "Posted", "transitioned" and "merged" are claimed only after
  verifying them.
- **Verbatim API errors.** A permission or workflow-condition error names exactly what's missing.
  Never paraphrase it away.
- **The comment and the transition are independent.** If the comment posts and the transition
  fails, the comment is already recorded so a retry can't double-post; report the transition
  failure and stop. Persist the intended comment body and transition id to the state file
  *before* attempting either, so a mid-run token expiry leaves a replayable hand-off rather than
  merged code and a silent ticket.
