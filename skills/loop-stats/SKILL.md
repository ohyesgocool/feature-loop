---
name: loop-stats
description: |
  Render what the feature loop's telemetry actually says: rounds to convergence, per-reviewer
  precision (the scorecard), token spend, lead times, and — the key question — whether the
  blind-spot ledger is working (repeat finding-categories declining or not). Reads the two
  files the loop maintains in the repo: docs/loops/metrics.jsonl and
  docs/reviews/blind-spots.md. Strictly read-only; never fabricates a number.
  Use when: "loop stats", "how is the loop performing", "reviewer scorecard",
  "is the ledger working", "what does a loop cost us", after a few loops have run.
argument-hint: "[--last N] — optional; defaults to all recorded runs"
---

# Loop Stats

Answer, with numbers, the questions an engineering lead asks about the loop: *is it
converging faster, are the reviewers earning their tokens, and is the learning real?*

Data sources (both maintained by the loop, both committed to the repo):
- `docs/loops/metrics.jsonl` — `run` records (from `/ship-feature`) and `triage` records
  (from `/address-review`)
- `docs/reviews/blind-spots.md` — the lessons ledger with its category tally

## Process

1. **Parse defensively.** Read `metrics.jsonl` with jq line by line; count and report
   malformed lines, never crash on them. Apply `--last N` to run records if given. If a file
   is missing or empty, say exactly which file and which skill produces it
   (`/ship-feature` → run records, `/address-review` → triage records + ledger) and stop —
   no stats theater over no data.

2. **Compute and render four small tables, each with a one-line verdict under it.** A verdict
   must cite its numbers; where a field is absent, show `n/a` — never invent.

   **Delivery** — runs by outcome (merged / ready / stopped-with-reason), median and trend of
   rounds-to-convergence, lead-time trend.
   *Verdict example: "convergence steady at 2 rounds; the two stopped runs were both
   red-pipeline, not review thrash."*

   **Reviewer scorecard** — per reviewer across triage records: precision =
   (valid + partial) / total, finding volume, trend. Call it out plainly when a reviewer sits
   under ~40% precision across 5+ runs — that's tokens spent generating noise; recommend
   tuning its prompt with the recurring false-positive patterns, or dropping it.

   **Cost** — tokens per run and per round, by reviewer, trend. Pair with rounds: a rising
   token trend with flat rounds means diffs are growing, not the loop degrading.

   **Learning (the ledger check)** — join triage `categories` over time against the ledger's
   tally: for each category with a written lesson, does it recur in triage records from runs
   *after* the lesson date? Recurring ⇒ the lesson isn't landing — say so and name it
   (candidate for promotion into the project's agent-instructions file). Gone quiet ⇒ the
   ledger is doing its job — say that too; teams should see their system winning.

3. **Close with the one thing to change** — the single highest-leverage adjustment the numbers
   support (tune reviewer X's prompt, split large MRs, promote lesson Y to a standing rule).
   One recommendation, not a list; if the data genuinely supports none, say the loop is
   healthy and stop.

## Rules

- **Read-only.** This skill never writes, commits, or edits anything.
- **No fabricated numbers.** Missing field ⇒ `n/a`; thin data (< 3 runs) ⇒ say the sample is
  too small for trends and show raw values only.
- **Insights over dumps.** Four small tables with verdicts beat one exhaustive one. Every
  verdict cites the numbers it stands on.
