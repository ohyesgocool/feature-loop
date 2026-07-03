---
name: mr-review
description: |
  Get independent external-AI reviews on a merge request and post them as MR comments:
  sends the MR title, description, and diff to each configured external reviewer — OpenAI GPT
  (correctness-first implementation brief) and xAI Grok (code-quality review: DRY, cohesion,
  naming, testability) — and posts each reviewer's brief as a separate MR discussion.
  The point is independence: the reviewers are DIFFERENT models from the one that wrote the
  code, so they don't share its blind spots. Review-only — never modifies code, never approves,
  never merges. Feeds /address-review, which triages and resolves the posted comments.
  Requires OPENAI_API_KEY and/or XAI_API_KEY in the environment (see Configuration).
  Use when: "review this MR", "run the AI review", "get an external review", right after
  /build-feature raises an MR, or re-reviewing after fixes were pushed.
argument-hint: "[MR number or URL] — optional; defaults to the current branch's open MR"
---

# MR Review

Run the review stage of the feature loop: collect the MR's real diff, hand it to **external**
reviewer models, and post their briefs back onto the MR as comments. You orchestrate; the
reviews themselves come from models that did not write the code — that independence is the value,
so post their output **verbatim**, never edited, softened, or summarized by you.

This skill is **review-only**: it never modifies code, pushes commits, approves, or merges.
The follow-up — deciding which findings hold and fixing them — is `/address-review`'s job.

If the user provided arguments: `$ARGUMENTS` — an MR number or URL. With no arguments,
auto-detect the current branch's open MR (`glab mr list --source-branch "$(git branch --show-current)"`).

> **Forge note.** Written for GitLab (`glab`). On GitHub: `gh pr view` / `gh pr diff` to gather
> context, `gh pr comment <n> --body-file <brief.md>` to post each review. Process identical.

## Configuration

All configuration is environment variables — set them in your shell profile, or in your agent's
settings (for Claude Code, the `env` block of `~/.claude/settings.json`); see the repo README:

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `OPENAI_API_KEY` | for the GPT reviewer | — | correctness-first implementation brief |
| `OPENAI_MODEL` | no | `gpt-5.5` | which OpenAI model reviews |
| `XAI_API_KEY` | for the Grok reviewer | — | code-quality review brief |
| `XAI_MODEL` | no | `grok-4.3` | which xAI model reviews |
| `MR_REVIEW_MAX_DIFF_CHARS` | no | `120000` | diff truncation limit sent to reviewers |

A reviewer is **enabled if its key is set** — one key runs one review, both keys run both. If
neither key is set, stop immediately with a clear message pointing at this table; do not fall
back to reviewing the code yourself (that would defeat the independence).

**Never print, echo, or log the keys.** Reference them only as `$OPENAI_API_KEY` / `$XAI_API_KEY`
inside curl headers.

---

## Process

### Step 1: Resolve the MR and gather its context

1. Resolve the MR from `$ARGUMENTS` or the current branch (as above). If there's genuinely no
   open MR, say so and stop — this skill reviews MRs, not working trees.
2. Fetch what the reviewers see:
   - Title, description, URL, project path: `glab mr view <id> -F json`
   - The full diff: `glab mr diff <id>`
3. **Truncation with disclosure.** If the diff exceeds `MR_REVIEW_MAX_DIFF_CHARS`, truncate it —
   but append an explicit note to the prompt listing every file whose diff was cut or omitted:
   `NOTE: diff truncated at N chars. Files partially or fully omitted: <list>. Do not report
   these files as empty or missing.` Reviewers reading a silently truncated diff produce false
   "file is empty/missing" findings — disclosure prevents that entire failure class.

### Step 2: Build each reviewer's prompt

Fill the templates below (at the end of this file), substituting `{{PROJECT_PATH}}`, `{{TITLE}}`,
`{{URL}}`, `{{DESCRIPTION}}`, `{{DIFF}}`. Write each completed prompt to a temp file — don't
inline megabytes into a shell argument.

### Step 3: Call the reviewers — in parallel, with patience

Both APIs are OpenAI-compatible chat completions. Build the payload with jq (safe escaping),
call with generous timeouts (big diffs on reasoning models can take minutes), run enabled
reviewers concurrently:

```bash
jq -n --rawfile prompt prompt-openai.txt --arg model "${OPENAI_MODEL:-gpt-5.5}" \
  '{model: $model, messages: [{role: "user", content: $prompt}]}' > payload-openai.json
curl -sS --max-time 600 https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" -H "Content-Type: application/json" \
  -d @payload-openai.json -o response-openai.json
```

Same shape for Grok against `https://api.x.ai/v1/chat/completions` with `$XAI_API_KEY` and
`${XAI_MODEL:-grok-4.3}`.

Extract each brief: `jq -r '.choices[0].message.content' response-X.json > brief-X.md`.

**Validate before posting**: if the response carries an `.error`, or the brief is empty or
doesn't start with the expected `## … review` heading, report that reviewer as failed (with the
API's error message, never the key) and continue with the other reviewer. Never post an error
body or an empty comment to the MR.

### Step 4: Post each brief as its own MR discussion

One discussion per reviewer, verbatim:

```bash
glab api "projects/:id/merge_requests/<iid>/discussions" -X POST -F "body=@brief-openai.md"
```

Re-runs post **new** comments (a re-review after fixes is a new review — that's correct); note in
your report if earlier review comments from a previous run are still open on the MR.

### Step 5: Report and hand off

Report, in this shape:

```
## MR Review — !{N} "{title}"

Reviewers: {OpenAI gpt-5.5 ✅ · Grok grok-4.3 ✅ / failed: reason}
Diff sent: {N} files, {M} chars {(truncated — X files disclosed as omitted)}
Verdicts: OpenAI → {verdict line} · Grok → {verdict line}
Comments posted: {links or discussion refs}
```

Close with the handoff: run `/address-review` to triage the findings — that skill decides
line-by-line which comments hold, implements the valid ones, and replies to every thread.

---

## Rules

- **Independence is the product.** Post reviewer output verbatim. You may add the truncation
  disclosure to the *prompt*, but never edit, filter, or soften what comes *back*. If you
  disagree with a finding, that's `/address-review`'s argument to make — on the record.
- **Review-only.** No code changes, no commits, no approvals, no merge, no thread resolution.
- **Keys never leak.** Env-var references only; redact anything that could echo them; API error
  bodies are safe to show, request headers are not.
- **No silent truncation.** Anything cut from the diff is named to the reviewers and in your
  report. A reviewer misled about what it saw produces confidently wrong findings.
- **Fail one, run the other.** A failed reviewer degrades the run, it doesn't abort it — unless
  no reviewer is configured at all, which stops with the configuration table.
- **Don't review it yourself.** Even if both APIs are down. Your context wrote this code; the
  external eyes are the point. Report the outage instead.

---

## Prompt templates

### OpenAI — general implementation brief

```
Review this merge request and produce exactly one Markdown implementation brief.

Project: {{PROJECT_PATH}}
Title: {{TITLE}}
URL: {{URL}}
Description:
{{DESCRIPTION}}

Diff:
{{DIFF}}

The comment will be copied into another LLM to implement the fixes, so make it
actionable, self-contained, and specific enough to hand off directly.

Rules:
- Return only the review brief content. Do not wrap it in a code fence.
- Do not ask follow-up questions.
- Do not produce inline comments.
- Focus on correctness, security, data loss, race conditions, missing tests,
  maintainability, DRY violations, edge cases, and performance.
- Every issue must cite file paths and line numbers when available from the diff.
- Rank issues by severity: Critical, High, Medium, Low.
- For each issue, state the problem, why it matters, and the recommended fix.
- Include concrete test coverage that should be added.
- If there are no meaningful issues, say that clearly and mention residual test risk.
- Do not approve, merge, or claim the changes are safe.

Use this exact structure:

## AI MR review - implementation brief

**Verdict:** Request changes | Needs follow-up | No blocking issues

### Issues

1. **<Severity> - <short title>** (`path/to/file.ext:Lx-Ly`)
   - Problem: <what is wrong>
   - Impact: <why this matters>
   - Recommended fix: <what the implementer should change>
   - Tests to add/update: <specific tests>

### Test coverage to add

### Notes
```

### Grok — quality review brief

```
Review this merge request and produce exactly one Markdown quality review brief.

Project: {{PROJECT_PATH}}
Title: {{TITLE}}
URL: {{URL}}
Description:
{{DESCRIPTION}}

Diff:
{{DIFF}}

The comment will be copied into another LLM or handed to the author, so make it
actionable, self-contained, and specific enough to implement directly.

Rules:
- Return only the review brief content. Do not wrap it in a code fence.
- Do not ask follow-up questions.
- Do not produce inline comments.
- Focus on code quality gaps that are easy to miss in a correctness-first
  review: DRY violations, over-abstraction, under-abstraction, naming,
  cohesion, coupling, testability, readability, maintainability, consistency
  with local patterns, error handling shape, and complexity.
- Also flag correctness, security, data loss, race conditions, missing tests,
  and performance issues when they are visible from the diff.
- Every issue must cite file paths and line numbers when available from the diff.
- Rank issues by severity: Critical, High, Medium, Low.
- Avoid nitpicks unless they indicate a repeated pattern or future maintenance
  cost.
- For each issue, state the problem, why it matters, and the recommended fix.
- Include concrete test coverage that should be added or updated.
- If there are no meaningful issues, say that clearly and mention residual test
  risk.
- Do not approve, merge, or claim the changes are safe.

Use this exact structure:

## Grok MR quality review - implementation brief

**Verdict:** Request changes | Needs follow-up | No blocking issues

### Issues

1. **<Severity> - <short title>** (`path/to/file.ext:Lx-Ly`)
   - Problem: <what is wrong>
   - Impact: <why this matters>
   - Recommended fix: <what the implementer should change>
   - Tests to add/update: <specific tests>

### Test coverage to add

### Notes
```
