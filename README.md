# Claude Feature Workflow

**Loop engineering for [Claude Code](https://claude.com/claude-code): dump a feature brief, get back a planned, built, independently-reviewed, merge-ready change — and a system that gets better every time it runs.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-blue)](#install)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#contributing)

```
                        /ship-feature  (the loop, one command)
┌─────────────────────────────────────────────────────────────────────────┐
│ /plan-feature ──→ /build-feature ──→ /mr-review ──→ /address-review     │
│     (plan)           (code + MR)     (external AI      (triage, fix,    │
│                                        reviewers)        reply)         │
│                          ▲                ▲                 │  │        │
│                          │                └── re-review ────┘  │        │
│                          │                   until clean       │        │
│                   blind-spot ledger  ◄─── "why I missed it" ───┘        │
│                   (the loop learns)              │                      │
│                                                  ▼                      │
│                                    MERGE GATE  --merge=ask|auto|never   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       ▼
                              /land  (pipelines → migrations →
                               deploy order → canary → SHIPPED)
```

## Why

AI coding agents are good at writing code and terrible at grading their own homework. This project structures the work the way a disciplined engineering team would:

- **A product owner** who studies the codebase and challenges the brief before anything is built (`/plan-feature`)
- **An engineer** who executes the plan surgically, commits granularly, and always ships the MR with a reviewable description (`/build-feature`)
- **Independent reviewers** — *different models* (OpenAI GPT, xAI Grok) that never saw the code being written, so they don't share the author's blind spots (`/mr-review`)
- **An honest author response** — every review comment triaged against the real diff, fixed or dismissed *with reasons, on the record* (`/address-review`)
- **A release engineer** who takes the merge to production off a written runbook and doesn't call it shipped until the canary is clean (`/land`)
- **A retro that actually changes behavior** — every valid finding's "why I missed it" is written to a ledger the builder reads before the next feature, so reviewers stop finding the same category of bug twice

`/ship-feature` chains the middle four into a self-running loop with an explicit merge gate. Every skill also works standalone.

## Quick start

```
/plugin marketplace add ohyesgocool/claude-feature-workflow
/plugin install feature-workflow@claude-feature-workflow
```

Then, in any repo:

```
/ship-feature Add CSV export to the reports page — finance needs month-end data in Excel
```

It asks its clarifying questions up front (the loop's one designed touchpoint), then: plan → build → MR → external review → fixes → re-review until clean → **READY TO MERGE**. Say "merge it", or run with `--merge=auto` to skip the gate.

Prefer manual installation? Clone and copy:

```bash
git clone https://github.com/ohyesgocool/claude-feature-workflow.git
cp -r claude-feature-workflow/skills/* ~/.claude/skills/
```

New Claude Code sessions pick the skills up automatically.

## The skills

| Skill | Role | Standalone use |
|---|---|---|
| [`/ship-feature`](skills/ship-feature/SKILL.md) | the loop, one command | "ship this feature" |
| [`/plan-feature`](skills/plan-feature/SKILL.md) | product owner + architect | "plan this" |
| [`/build-feature`](skills/build-feature/SKILL.md) | disciplined implementer | "build this plan" |
| [`/mr-review`](skills/mr-review/SKILL.md) | independent external reviewers | "review this MR" |
| [`/address-review`](skills/address-review/SKILL.md) | honest triage + resolution | "address the review" |
| [`/land`](skills/land/SKILL.md) | post-merge release engineer | "land the merge" |

### `/ship-feature` — the loop

Runs plan → build → review → address, re-reviewing until **convergence**: a round with zero valid findings (or all reviewer verdicts "No blocking issues"), every thread answered, CI green. Then the merge gate:

| Flag | Behavior |
|---|---|
| `--merge=ask` *(default)* | stops at **READY TO MERGE** and waits for your approval |
| `--merge=auto` | merges when converged + CI green (`glab mr merge` / `gh pr merge`) |
| `--merge=never` | reports and leaves the MR open |
| `--max-rounds=N` *(default 3)* | honest exit if reviewers keep finding things |
| `--autonomous` | planner records defaults instead of asking questions |

Built-in safeguards: **no-progress detection** (a finding that survives two rounds stops the loop instead of thrashing — a human call beats a fourth attempt), **honest convergence** (threads are never resolved just to make the loop exit), and the sub-skills' hard stops (architectural concern, red build, red pipeline, merge conflicts) always surface to you. The loop makes routine iteration autonomous — it never makes judgment calls autonomous.

### `/plan-feature` — product owner + architect

Give it a few paragraphs on what you want. It studies the codebase first (never plans from memory — every integration point cited as `file:line`), frames the feature by **what the user achieves** rather than what gets built, asks only the genuinely blocking questions (batched, each with a recommended answer), audits every proposed UI element against *less is more*, proves the design is buildable against real code, and writes a phased plan where each phase is small and self-contained enough for a coding agent to execute without coming back with questions. Output: a plan file in `docs/plans/`.

### `/build-feature` — disciplined implementer

Branches **fresh from main**, reads the plan line by line, re-grounds every phase in the current code, then builds with clean-code discipline: SOLID applied not recited, reuse before writing (more code = more maintenance surface), surgical changes only. Two ledgers keep it honest — a **hack ledger** (a forced workaround exists only with a `HACK:` comment naming the constraint) and a **synchronization ledger** (async by default; every deliberate sync point records the invariant it protects). Every phase gates on type-check + tests; commits are granular; the finish line is always an MR with a fixed four-section description: **Problem / Solution / Technical details / Review notes**.

### `/mr-review` — independent external reviewers

Sends the MR's title, description, and diff to each configured reviewer and posts each brief **verbatim** as its own MR comment: OpenAI GPT for a correctness-first implementation brief (severity-ranked, `file:line`-cited, fixes and tests included), xAI Grok for a quality review (DRY, cohesion, naming, testability). Review-only. Large diffs are truncated *with disclosure* — omitted files are named to the reviewers so they can't produce false "file is missing" findings. A reviewer is enabled simply by its API key being present.

### `/address-review` — honest triage + resolution

Correlates every comment **line by line against the actual diff**, then verdicts: **VALID** (fix it), **PARTIAL** (right instinct, wrong specifics — fix the real concern), **INVALID** (dismissed, citing the specific context the reviewer couldn't see; burden of proof is on dismissal). For each valid finding it answers *why I missed it* with a real blind-spot category, implements the fixes (one commit per comment), pushes on a green build, and replies to every thread. Then it feeds the ledger ↓.

### The blind-spot ledger — how the loop learns

`docs/reviews/blind-spots.md`, committed to *your* repo:

```markdown
# Blind-spot ledger
Tally: Error path ×4 · Consistency ×2 · Edge case ×1

- [2026-07-03 !87 C1] Error path — every new I/O path gets its failure
  branch coded and tested before the happy path ships.
```

`/address-review` appends a forward-looking rule per valid finding; `/build-feature` reads the ledger before coding and re-checks the diff against the top categories in its verification sweep. The success metric: external reviewers stop finding the same category twice. Because it's a committed file, the learning is shared by every agent session — and every human — that touches the repo.

### `/land` — post-merge release engineer

Merged is not shipped. `/land` executes your project's landing runbook (`docs/landing.md` — deploy order, how migrations apply, health checks, smoke checks, canary window, rollback path; drafted automatically on first run from your CI config and confirmed with you). Every step is tagged `[claude]` (executed and probed directly) or `[human]` (production access only you hold — you get the exact commands, and it waits). It deploys in order with health checks between services, holds a canary window after smoke checks, and claims **SHIPPED** only when the canary closes clean. Rollback is prepared and offered with evidence — never auto-executed.

## Configuration

### What you need (and what you don't)

| Requirement | Needed for | Where to get it |
|---|---|---|
| **Claude Code** | everything — planning, building, triage, orchestration | [claude.com/claude-code](https://claude.com/claude-code) |
| **`glab` CLI** (GitLab) or **`gh` CLI** (GitHub), authenticated | MRs/PRs: raising, reading, commenting, merging | `glab auth login` / `gh auth login` |
| **`OPENAI_API_KEY`** | the GPT reviewer in `/mr-review` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| **`XAI_API_KEY`** *(optional)* | the Grok reviewer in `/mr-review` | [console.x.ai](https://console.x.ai) |
| `jq`, `curl` | `/mr-review` API calls | preinstalled on most systems / `brew install jq` |

> **You do not need a separate Claude/Anthropic API key.** The skills run inside Claude Code under your existing login. The only external keys are for the independent reviewers — and with neither set, everything except `/mr-review` still works fully.

### Reviewer settings

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `OPENAI_API_KEY` | to enable the GPT reviewer | — | correctness review |
| `OPENAI_MODEL` | no | `gpt-5.5` | which OpenAI model reviews |
| `XAI_API_KEY` | to enable the Grok reviewer | — | quality review |
| `XAI_MODEL` | no | `grok-4.3` | which xAI model reviews |
| `MR_REVIEW_MAX_DIFF_CHARS` | no | `120000` | diff size limit sent to reviewers |
| `SHIP_MERGE_MODE` | no | `ask` | default merge gate for `/ship-feature` |

### Where to put the keys

**Option A — Claude Code settings** (recommended; applies everywhere):

```json
// ~/.claude/settings.json
{
  "env": {
    "OPENAI_API_KEY": "sk-...",
    "XAI_API_KEY": "xai-..."
  }
}
```

**Option B — shell profile** (`~/.zshrc` / `~/.bashrc`):

```bash
export OPENAI_API_KEY="sk-..."
export XAI_API_KEY="xai-..."
```

Never commit keys to a repository. The skills only ever reference them as environment variables and never print them.

## FAQ

**Does it work with GitHub?** Yes. The skills are written GitLab-first (`glab`), and each MR-touching skill carries a "Forge note" mapping every command to its `gh` equivalent — Claude applies the mapping automatically when the repo is on GitHub.

**Can I use just one or two skills?** Yes — each is standalone. `/address-review` works on any MR with comments (including a colleague's); `/mr-review` can review a hand-written change; `/land` runs off any merged MR.

**What does a loop cost?** Claude usage under your existing Claude Code plan, plus reviewer API tokens per round (one OpenAI and/or one xAI call per round, diff-sized). `--max-rounds` caps it; the Ship Report shows how many rounds convergence took.

**How autonomous is it, really?** Deliberately bounded. The loop automates *iteration*; it never automates *judgment*. Human touchpoints: the planner's questions at the start (skippable with `--autonomous`), any genuine architectural concern, a red build or pipeline, and the merge gate (unless you opt into `--merge=auto`).

**Can reviews run automatically on every MR instead of on demand?** Yes — the prompt templates in [`skills/mr-review/SKILL.md`](skills/mr-review/SKILL.md) are drop-in for a webhook service (receive the MR event, fetch the diff, call the reviewers, post the briefs). Pair that with `/address-review` and the loop runs itself on every MR your team opens.

## Contributing

The skills are plain Markdown prompts — reading one is understanding it, and improving one is a text edit. Issues and PRs welcome: sharper review prompts, more reviewer backends, better convergence heuristics, forge-specific fixes. If you add a reviewer, keep the contract: independent model, verbatim posting, keys via env only.

## License

[MIT](LICENSE)
