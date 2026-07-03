# Feature Loop

**Loop engineering for AI coding agents: dump a feature brief, get back a planned, built, independently-reviewed, merge-ready change — and a system that gets better every time it runs.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/format-Agent%20Skills-blue)](#works-with)
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

AI coding agents are good at writing code and terrible at grading their own homework. Feature Loop structures the work the way a disciplined engineering team would:

- **A product owner** who studies the codebase and challenges the brief before anything is built (`/plan-feature`)
- **An engineer** who executes the plan surgically, commits granularly, and always ships the MR with a reviewable description (`/build-feature`)
- **Independent reviewers** — *different models* (OpenAI GPT, xAI Grok) that never saw the code being written, so they don't share the author's blind spots (`/mr-review`)
- **An honest author response** — every review comment triaged against the real diff, fixed or dismissed *with reasons, on the record* (`/address-review`)
- **A release engineer** who takes the merge to production off a written runbook and doesn't call it shipped until the canary is clean (`/land`)
- **A retro that actually changes behavior** — every valid finding's "why I missed it" is written to a ledger the builder reads before the next feature, so reviewers stop finding the same category of bug twice

`/ship-feature` chains the middle four into a self-running loop with an explicit merge gate. Every skill also works standalone.

## Works with

The six skills are plain-Markdown operating procedures in the Agent Skills format (a folder with a `SKILL.md`). There is nothing model-specific in them — any coding agent that can follow a Markdown procedure and run shell commands can execute the loop:

- **Claude Code** — supported natively: install as a plugin or drop the folders into `~/.claude/skills/` (see below).
- **Other agents** (Cursor, OpenAI Codex CLI, Gemini CLI, OpenCode, …) — point your agent at `skills/<name>/SKILL.md` as the instruction source for that step: load it as a custom command, rule, or prompt file per your tool's convention. Where a skill mentions an optional harness feature (a task tracker, parallel search subagents), it degrades gracefully to plain equivalents.

The hard dependencies are deliberately boring: a git forge CLI (`glab` or `gh`) and `curl` + `jq` for the external reviewer calls. The reviewer side is vendor-neutral too — any OpenAI-compatible chat-completions endpoint can be a reviewer.

## Quick start

**Claude Code (native):**

```
/plugin marketplace add ohyesgocool/feature-loop
/plugin install feature-workflow@feature-loop
```

Then, in any repo:

```
/ship-feature Add CSV export to the reports page — finance needs month-end data in Excel
```

It asks its clarifying questions up front (the loop's one designed touchpoint), then: plan → build → MR → external review → fixes → re-review until clean → **READY TO MERGE**. Say "merge it", or run with `--merge=auto` to skip the gate.

**Manual / other agents:**

```bash
git clone https://github.com/ohyesgocool/feature-loop.git
# Claude Code:
cp -r feature-loop/skills/* ~/.claude/skills/
# Any other agent: register each skills/<name>/SKILL.md as a command/rule/prompt
```

## The skills

| Skill | Role | Standalone use |
|---|---|---|
| [`/ship-feature`](skills/ship-feature/SKILL.md) | the loop, one command | "ship this feature" |
| [`/plan-feature`](skills/plan-feature/SKILL.md) | product owner + architect | "plan this" |
| [`/build-feature`](skills/build-feature/SKILL.md) | disciplined implementer | "build this plan" |
| [`/mr-review`](skills/mr-review/SKILL.md) | independent external reviewers | "review this MR" |
| [`/address-review`](skills/address-review/SKILL.md) | honest triage + resolution | "address the review" |
| [`/land`](skills/land/SKILL.md) | post-merge release engineer | "land the merge" |
| [`/loop-stats`](skills/loop-stats/SKILL.md) | the loop's own dashboard | "is the loop working?" |

### `/ship-feature` — the loop

Runs plan → build → review → address, re-reviewing until **convergence**: a round with zero valid findings (or all reviewer verdicts "No blocking issues"), every thread answered, CI green. Then the merge gate:

| Flag | Behavior |
|---|---|
| `--merge=ask` *(default)* | stops at **READY TO MERGE** and waits for your approval |
| `--merge=auto` | merges when converged + CI green (`glab mr merge` / `gh pr merge`) |
| `--merge=never` | reports and leaves the MR open |
| `--max-rounds=N` *(default 3)* | honest exit if reviewers keep finding things |
| `--autonomous` | planner records defaults instead of asking questions |
| `--resume` | continue an interrupted loop from its recorded stage and round |

The loop is **crash-safe**: state is persisted to `.loop/<slug>.state.json` (gitignore `.loop/`) at every stage boundary, every stage is idempotent on re-entry, and a loop that died with its session resumes instead of restarting.

Built-in safeguards: **no-progress detection** (a finding that survives two rounds stops the loop instead of thrashing — a human call beats a fourth attempt), **honest convergence** (threads are never resolved just to make the loop exit), and the sub-skills' hard stops (architectural concern, red build, red pipeline, merge conflicts) always surface to you. The loop makes routine iteration autonomous — it never makes judgment calls autonomous.

### `/plan-feature` — product owner + architect

Give it a few paragraphs on what you want. It studies the codebase first (never plans from memory — every integration point cited as `file:line`), frames the feature by **what the user achieves** rather than what gets built, asks only the genuinely blocking questions (batched, each with a recommended answer), audits every proposed UI element against *less is more*, proves the design is buildable against real code, and writes a phased plan where each phase is small and self-contained enough for a coding agent to execute without coming back with questions. Output: a plan file in `docs/plans/`.

### `/build-feature` — disciplined implementer

Branches **fresh from main**, reads the plan line by line, re-grounds every phase in the current code, then builds with clean-code discipline: SOLID applied not recited, reuse before writing (more code = more maintenance surface), surgical changes only. Two ledgers keep it honest — a **hack ledger** (a forced workaround exists only with a `HACK:` comment naming the constraint) and a **synchronization ledger** (async by default; every deliberate sync point records the invariant it protects). Every phase gates on type-check + tests; commits are granular with 50-character imperative subjects (the classic seven rules); the finish line is always an MR with a fixed four-section description: **Problem / Solution / Technical details / Review notes**.

### `/mr-review` — independent external reviewers

Sends the MR's title, description, and diff to each configured reviewer and posts each brief **verbatim** as its own MR comment: OpenAI GPT for a correctness-first implementation brief (severity-ranked, `file:line`-cited, fixes and tests included), xAI Grok for a quality review (DRY, cohesion, naming, testability). Review-only. A reviewer is enabled simply by its API key being present.

Three things make the reviews efficient rather than noisy:

- **Static pre-pass first** — gitleaks, semgrep (when installed) and a built-in migration lint run before any LLM call; machine-checkable findings post once, and LLM tokens go to judgment calls.
- **Context pack** — reviewers get the project's conventions and the plan's recorded decisions alongside the diff, killing the context-gap false positives that waste triage time. Hard rule: the pack informs, it never defends the diff.
- **Delta re-review** — round 2+ reviews only the changes since the last reviewed commit, with each prior finding's outcome attached, so reviewers verify fixes instead of re-raising them. Large diffs are still truncated *with disclosure* so nothing gets reported as falsely missing.

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

### `/loop-stats` — is the loop actually working?

The loop records itself: every run appends outcome, rounds-to-convergence, lead time, and token spend to `docs/loops/metrics.jsonl`; every triage appends per-reviewer valid/partial/invalid tallies and finding categories. `/loop-stats` turns that into answers an engineering lead actually asks: is convergence getting faster, which reviewer is earning its tokens (the precision scorecard — a reviewer under ~40% precision is generating noise), what does a loop cost, and — the key one — do blind-spot lessons actually stop their category from recurring. It ends with the single highest-leverage change the numbers support. Strictly read-only, never fabricates a number, and says "sample too small" instead of inventing trends.

### `/land` — post-merge release engineer

Merged is not shipped. `/land` executes your project's landing runbook (`docs/landing.md` — deploy order, how migrations apply, health checks, smoke checks, canary window, rollback path; drafted automatically on first run from your CI config and confirmed with you). Every step is tagged `[agent]` (executed and probed directly) or `[human]` (production access only you hold — you get the exact commands, and it waits). It deploys in order with health checks between services, holds a canary window after smoke checks, and claims **SHIPPED** only when the canary closes clean. Rollback is prepared and offered with evidence — never auto-executed.

## Configuration

### What you need (and what you don't)

| Requirement | Needed for | Where to get it |
|---|---|---|
| **An AI coding agent** (Claude Code supported natively) | planning, building, triage, orchestration | [claude.com/claude-code](https://claude.com/claude-code), or [your tool of choice](#works-with) |
| **`glab` CLI** (GitLab) or **`gh` CLI** (GitHub), authenticated | MRs/PRs: raising, reading, commenting, merging | `glab auth login` / `gh auth login` |
| **`OPENAI_API_KEY`** | the GPT reviewer in `/mr-review` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| **`XAI_API_KEY`** *(optional)* | the Grok reviewer in `/mr-review` | [console.x.ai](https://console.x.ai) |
| `jq`, `curl` | `/mr-review` API calls | preinstalled on most systems / `brew install jq` |

> **No extra LLM key is needed for the author side.** The skills run under whatever your coding agent already uses (e.g. your Claude Code login). The only additional keys are for the *independent reviewers* — and with neither set, everything except `/mr-review` still works fully.

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

**Shell profile** (`~/.zshrc` / `~/.bashrc`) — works for every agent:

```bash
export OPENAI_API_KEY="sk-..."
export XAI_API_KEY="xai-..."
```

**Claude Code users** can alternatively keep them in the `env` block of `~/.claude/settings.json`:

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-...",
    "XAI_API_KEY": "xai-..."
  }
}
```

Never commit keys to a repository. The skills only ever reference them as environment variables and never print them.

## FAQ

**Is this Claude-specific?** No. Claude Code is supported natively (plugin + skills directory), but the skills are plain Markdown procedures with no model-specific instructions — see [Works with](#works-with). The reviewer side speaks the OpenAI-compatible chat-completions API, so reviewers are swappable too.

**Does it work with GitHub?** Yes. The skills are written GitLab-first (`glab`), and each MR-touching skill carries a "Forge note" mapping every command to its `gh` equivalent — the agent applies the mapping automatically when the repo is on GitHub.

**Can I use just one or two skills?** Yes — each is standalone. `/address-review` works on any MR with comments (including a colleague's); `/mr-review` can review a hand-written change; `/land` runs off any merged MR.

**What does a loop cost?** Author-side usage under your coding agent's existing plan, plus reviewer API tokens per round (one OpenAI and/or one xAI call per round, diff-sized). `--max-rounds` caps it; the Ship Report shows how many rounds convergence took.

**How autonomous is it, really?** Deliberately bounded. The loop automates *iteration*; it never automates *judgment*. Human touchpoints: the planner's questions at the start (skippable with `--autonomous`), any genuine architectural concern, a red build or pipeline, and the merge gate (unless you opt into `--merge=auto`).

**Can reviews run automatically on every MR instead of on demand?** Yes — the prompt templates in [`skills/mr-review/SKILL.md`](skills/mr-review/SKILL.md) are drop-in for a webhook service (receive the MR event, fetch the diff, call the reviewers, post the briefs). Pair that with `/address-review` and the loop runs itself on every MR your team opens.

## Contributing

The skills are plain Markdown prompts — reading one is understanding it, and improving one is a text edit. Issues and PRs welcome: sharper review prompts, more reviewer backends, better convergence heuristics, forge-specific fixes, portability notes for more agent harnesses. Two contracts to keep: skills stay agent-agnostic (harness-specific features only with a graceful fallback), and reviewers stay independent (different model from the author, verbatim posting, keys via env only).

## License

[MIT](LICENSE)
