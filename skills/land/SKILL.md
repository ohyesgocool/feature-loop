---
name: land
description: |
  Post-merge landing: take merged MR(s) to verified-in-production by executing the project's
  landing runbook — watch the main pipeline(s) to green, apply or hand off migrations, deploy
  in the runbook's order, then canary-verify (health checks, smoke checks) before declaring
  shipped. Runbook-driven: reads docs/landing.md; on first run it drafts one by inspecting the
  repo (CI config, migrations, health endpoints) and confirms it with you. Every step is tagged
  [claude] (executed directly) or [human] (prod-locked — handed to you as exact commands and
  waited on, never assumed). Hard stops: red pipeline, failed migration, failed canary,
  unconfirmed human step. Never auto-rolls-back; prepares the revert and asks.
  Use when: "land this", "deploy the merge", after /ship-feature merges, "take it to prod",
  running post-merge follow-ups (migrations, deploy order, canary).
argument-hint: "[MR ref(s) or 'latest'] — defaults to the most recently merged MR on this repo"
---

# Land

Merged is not shipped. Landing is everything between the merge commit and "verified in
production": pipelines, migrations, deploys in the right order, and proof that the thing
actually works where users are. This skill executes that sequence the way a careful release
engineer would — from a written runbook, step by step, verifying each step before the next,
and stopping loudly the moment reality disagrees.

Two kinds of steps, always explicit:
- **`[claude]`** — you execute and verify it directly (watch a pipeline, call a health
  endpoint, run a migration command you have access to).
- **`[human]`** — production access the user holds and you don't (a psql session into a locked
  VPC, a cloud console action). You hand over the **exact** command or click-path, then wait
  for their confirmation. A human step is never assumed done, inferred done, or skipped.

## The runbook — `docs/landing.md`

The landing sequence is project knowledge, not skill knowledge, so it lives in the repo where
the team can review and version it. The skill reads `docs/landing.md`, which defines:

```markdown
# Landing runbook

## Deploy order
1. {repo/service} — {how it deploys: CI on main / command}
2. …

## Migrations
- How they're applied: {auto on deploy | command `…` | [human] manual psql against prod}
- Where they live: {path}

## Health checks            <!-- [claude] -->
- {GET https://…/health → expect 200}
- {command → expected output}

## Smoke checks             <!-- the 2–3 flows that prove the system is alive -->
- {curl/CLI per flow → expected result}

## Canary window
- {N} minutes: {what to watch — error logs command, dashboard, re-run health}

## Rollback
- {revert MR + redeploy | pinned previous version | [human] steps}
```

**First run, no runbook:** draft one by inspecting the repo — CI config (`.gitlab-ci.yml` /
`.github/workflows/`) for how deploys trigger, the migrations directory for how schema changes
apply, the codebase for health endpoints — present the draft, confirm it with the user (this
one-time setup is a legitimate stop), commit it. Every later landing is hands-off up to the
`[human]` steps.

---

## Process

### Step 1: Resolve what's landing

From `$ARGUMENTS` (MR refs, or `latest`) or by finding the most recently merged MR on this
repo (`glab mr list --merged` / `gh pr list --state merged`). Then collect the landing
payload for each MR:

- **Migrations** and their apply method — from the MR description's Technical details section
  and the diff itself (never trust the description alone; grep the diff for migration files).
- **Cross-repo siblings** — if the MR description names a deploy order with sibling MRs in
  other repos, all of them land together, in that order. Verify each sibling is merged; a
  half-merged set is a hard stop.
- **Deferred items / follow-ups** from the build or ship report — surfaced in the final
  report, not silently dropped.

### Step 2: Load (or draft) the runbook

Read `docs/landing.md`. Missing → the first-run drafting flow above. Present the **landing
plan**: the ordered checklist for this specific landing, each step tagged `[claude]` or
`[human]`, before executing anything. Do not pause for approval — the plan is printed so the
user can interrupt, and the `[human]` steps are natural checkpoints anyway.

### Step 3: Execute, in order, verifying every step

For each repo/service in the runbook's deploy order:

1. **Pipeline to green** — poll the main branch pipeline for the merge commit
   (`glab ci status` / `gh run watch`; poll with patience, report progress once per ~2 min).
   Red pipeline = hard stop with the failing job's log tail.
2. **Migrations before or per the runbook's ordering** —
   - `[claude]` migrations: run the runbook's command, then **verify the effect** (the
     migration's own success output plus a functional probe where possible — the app-level
     query or endpoint that would fail if the schema change didn't land).
   - `[human]` migrations: print the exact command block (file path, target, any ordering
     between multiple files), wait for confirmation, then still run the functional probe —
     confirmation is trusted, verification is free.
3. **Deploy** per the runbook (often "CI on main already did it" — then this step is
   verifying the deploy actually rolled out: new version live, container restarted, etc.).
4. **Health checks** for that service before moving to the next one in the order — deploy
   order exists because later services depend on earlier ones being *actually up*, not
   *probably up*.

### Step 4: Canary

After the last service is up: run the runbook's smoke checks (the 2–3 real flows), then hold
the canary window — re-check health and scan whatever the runbook says to watch (error logs,
5xx counts) at the start and end of the window.

**Canary failure is a hard stop, and rollback is a recommendation, not a reflex:** report
what failed with evidence, prepare the rollback per the runbook (draft the revert MR, name
the redeploy command) — and ask. A wrong auto-rollback during a partial deploy can double the
damage; the human makes that call with your evidence in front of them.

### Step 5: Landing report

```
## Landing Report — {MR(s)}

Outcome: SHIPPED ✅ | STOPPED at {step} ({reason})

Sequence:
  1. [claude] {repo} pipeline → green (4m12s)
  2. [human]  psql migration 046 → confirmed by user · probe ✅
  3. [claude] {repo} deploy verified · health ✅
  4. [claude] smoke checks 3/3 ✅ · canary window 10m clean

Follow-ups: {deferred items carried forward · runbook updates worth making · none}
```

If anything in the runbook proved stale during the landing (an endpoint moved, a command
changed), update `docs/landing.md` in the same session and note it — the runbook only stays
trustworthy if every landing leaves it correct.

---

## Rules

- **The runbook is the authority.** No step skipped, reordered, or improvised. If this
  landing needs something the runbook doesn't cover, that's a runbook edit (visible, committed)
  — not an ad-hoc action.
- **Verify every step; trust no green checkmark you didn't probe.** A pipeline passing is not
  the service being up; a migration "applied" is not the schema being right. Each step has a
  probe and the probe result goes in the report.
- **Human steps are waited on, never assumed.** And even after confirmation, run the
  functional probe — people fat-finger psql sessions too.
- **Half-landed is a hard stop.** Sibling MR unmerged, migration failed mid-order, one service
  deployed and the next red — stop and report exactly where the system stands; the report's
  "Sequence" section is what makes a stopped landing recoverable.
- **No destructive operations the runbook doesn't authorize.** Dropping, truncating, force
  redeploying over pinned versions — only if the runbook explicitly says so, or the user does.
- **Rollback is prepared, offered, and human-approved.** Never executed on your own judgment.
- **Shipped means canary-verified.** Not "deploy command exited 0". The report claims SHIPPED
  only after smoke checks pass and the canary window closes clean.
