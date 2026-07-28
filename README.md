# Feature Loop

**Loop engineering for AI coding agents: hand over a Jira ticket or a feature brief, get back a planned, built, independently-reviewed, merged change — and a ticket that says honestly what shipped and what didn't.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/format-Agent%20Skills-blue)](#works-with)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#contributing)

```
                        /ship-feature  (the loop, one command)
┌─────────────────────────────────────────────────────────────────────────┐
│ /ticket-intake ──→ /plan-feature ──→ /build-feature ──→ /mr-review      │
│  (Jira issue →       (plan, with       (code + MR)      (external AI    │
│   REQ-1..n)        REQ coverage)                          reviewers)    │
│                          ▲                                    │         │
│                          │              /address-review ◄─────┘         │
│                          └── re-review ── (triage, fix, reply)          │
│                                until clean       │                      │
│                                                  ▼                      │
│                          /resolve-conflicts ──→ MERGE GATE              │
│                          (if the target moved)  ask|auto|never          │
│                                                  │                      │
│                                                  ▼                      │
│                                         /ticket-report                  │
│                             (one comment + status — Done only if        │
│                              every requirement actually landed)         │
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
- **A written-back status** — one Jira comment naming what was delivered per requirement, with evidence, and explicitly what the ticket asked for that was *not* built; the ticket moves to Done only when everything was covered (`/ticket-report`)

`/ship-feature` chains the whole sequence into a self-running loop with an explicit merge gate. Every skill also works standalone.

## Works with

The nine skills are plain-Markdown operating procedures in the Agent Skills format (a folder with a `SKILL.md`). There is nothing model-specific in them — any coding agent that can follow a Markdown procedure and run shell commands can execute the loop:

- **Claude Code** — supported natively: install as a plugin or drop the folders into `~/.claude/skills/` (see below).
- **Other agents** (Cursor, OpenAI Codex CLI, Gemini CLI, OpenCode, …) — point your agent at `skills/<name>/SKILL.md` as the instruction source for that step: load it as a custom command, rule, or prompt file per your tool's convention. Where a skill mentions an optional harness feature (a task tracker, parallel search subagents), it degrades gracefully to plain equivalents.

The hard dependencies are deliberately boring: a git forge CLI (`glab` or `gh`) and `curl` + `jq` for the external reviewer calls. The reviewer side is vendor-neutral too — any OpenAI-compatible chat-completions endpoint can be a reviewer.

**One exception, and it's a real one:** the two ticket skills (`/ticket-intake`, `/ticket-report`) reach Jira through the **Atlassian MCP server** and nothing else. That is harness-specific (MCP), vendor-specific (Atlassian), and OAuth-based — which means ticket mode does not run headless, in CI, or under cron. The other seven skills have no such dependency, and a ticket skill that can't bind its tools stops with the setup command rather than guessing what the ticket said.

## Quick start

**Claude Code (native):**

```
/plugin marketplace add ohyesgocool/feature-loop
/plugin install feature-workflow@feature-loop
```

Then, in any repo — from a ticket:

```
/ship-feature PROJ-123
```

It reads the whole ticket (description, acceptance criteria, every comment, subtasks, links), turns it into numbered requirements, then plans → builds → MR → external review → fixes → re-review until clean → resolves any conflicts → merges → posts one Jira comment with per-requirement coverage and moves the ticket. Ticket runs are **autonomous by default** (`--merge=auto`, planner self-answers). If an MR is already open for that ticket, it skips the build and audits that MR instead.

Or from a brief:

```
/ship-feature Add CSV export to the reports page — finance needs month-end data in Excel
```

It asks its clarifying questions up front (the loop's one designed touchpoint), then runs the same loop and stops at **READY TO MERGE**. Say "merge it", or pass `--merge=auto` to skip the gate.

**Manual / other agents:**

```bash
git clone https://github.com/ohyesgocool/feature-loop.git
# Claude Code:
cp -r feature-loop/skills/* ~/.claude/skills/
# Any other agent: register each skills/<name>/SKILL.md as a command/rule/prompt
```

**Full walkthrough:** [SETUP.md](SETUP.md) — line-by-line setup with a checkpoint after every step (forge CLI auth, reviewer keys, optional static tools, first run, troubleshooting). Written to be followed by a person or executed by an LLM.

## The skills

| Skill | Role | Standalone use |
|---|---|---|
| [`/ship-feature`](skills/ship-feature/SKILL.md) | the loop, one command | "ship PROJ-123" |
| [`/ticket-intake`](skills/ticket-intake/SKILL.md) | Jira issue → numbered requirements | "what does PROJ-123 ask for?" |
| [`/plan-feature`](skills/plan-feature/SKILL.md) | product owner + architect | "plan this" |
| [`/build-feature`](skills/build-feature/SKILL.md) | disciplined implementer | "build this plan" |
| [`/mr-review`](skills/mr-review/SKILL.md) | independent external reviewers | "review this MR" |
| [`/address-review`](skills/address-review/SKILL.md) | honest triage + resolution | "address the review" |
| [`/resolve-conflicts`](skills/resolve-conflicts/SKILL.md) | thread-safe conflict resolution | "main moved, fix the conflicts" |
| [`/ticket-report`](skills/ticket-report/SKILL.md) | one Jira comment + status transition | "update the ticket" |
| [`/land`](skills/land/SKILL.md) | post-merge release engineer | "land the merge" |

### `/ship-feature` — the loop

Runs intake → plan → build → review → address, re-reviewing until **convergence**: a round with zero valid findings (or all reviewer verdicts "No blocking issues"), every thread answered, pipeline green. Then conflicts if any, the merge gate, and the ticket write-back.

| Flag | Behavior |
|---|---|
| `--merge=ask` *(default in brief mode)* | stops at **READY TO MERGE** and waits for your approval |
| `--merge=auto` *(default in ticket mode)* | merges when converged + pipeline green (`glab mr merge` / `gh pr merge`) |
| `--merge=never` | reports and leaves the MR open |
| `--max-rounds=N` *(default 3)* | honest exit if reviewers keep finding things |
| `--autonomous` *(on in ticket mode)* | planner records defaults instead of asking questions |
| `--resolve-conflicts=auto\|never` *(default auto)* | resolve a moved target, or just report it |
| `--review-on-entry` | review even when entering on an MR that already exists |
| `--resume` | continue an interrupted loop from its recorded stage and round |

The loop is **crash-safe**: state is persisted to `.loop/<slug>.state.json` (gitignore `.loop/`) at every stage boundary, every stage is idempotent on re-entry, and a loop that died with its session resumes instead of restarting.

Built-in safeguards: **no-progress detection** (a finding that survives two rounds stops the loop instead of thrashing), **honest convergence** (threads are never resolved just to make the loop exit), a **four-part green-pipeline check** (a pipeline is green only when it exists, is *for this commit*, matches the branch head, and reads exactly `success` — polling straight after a push otherwise returns the *previous* commit's status), and the sub-skills' hard stops (architectural concern, red build, red pipeline, unresolvable conflict).

**How autonomy works in ticket mode.** It removes the human from the *waiting*, never from the *deciding*. Every hard stop still fires — it hands off with evidence (MR marked Draft, a Jira comment naming the decision needed, no transition to Done, state left resumable) instead of blocking on a prompt. `--merge=auto` is unreachable from any degraded run: max-rounds, no-progress, a deferred finding, a hard stop, or a flagged piece of untrusted ticket input all force `never`. The planner never self-answers a question about authorization, data retention, money, PII, external contracts, or auth — those stop the run.

### `/plan-feature` — product owner + architect

Give it a few paragraphs on what you want. It studies the codebase first (never plans from memory — every integration point cited as `file:line`), frames the feature by **what the user achieves** rather than what gets built, asks only the genuinely blocking questions (batched, each with a recommended answer), audits every proposed UI element against *less is more*, proves the design is buildable against real code, and writes a phased plan where each phase is small and self-contained enough for a coding agent to execute without coming back with questions. Output: a plan file in `docs/plans/`.

### `/build-feature` — disciplined implementer

Branches **fresh from main**, reads the plan line by line, re-grounds every phase in the current code, then builds with clean-code discipline: SOLID applied not recited, reuse before writing (more code = more maintenance surface), surgical changes only. Two ledgers keep it honest — a **hack ledger** (a forced workaround exists only with a `HACK:` comment naming the constraint) and a **synchronization ledger** (async by default; every deliberate sync point records the invariant it protects). Every phase gates on type-check + tests; commits are granular with 50-character imperative subjects (the classic seven rules); the finish line is always an MR with a fixed four-section description: **Problem / Solution / Technical details / Review notes**.

### `/mr-review` — independent external reviewers

Sends the MR's title, description, and diff to each configured reviewer and posts each brief **verbatim** as its own MR comment: OpenAI GPT for a correctness-first implementation brief (severity-ranked, `file:line`-cited, fixes and tests included), xAI Grok for a quality review (DRY, cohesion, naming, testability). Review-only. A reviewer is enabled simply by its API key being present.

Three things make the reviews efficient rather than noisy:

- **Static pre-pass first** — gitleaks, semgrep (when installed) and a built-in migration lint run before any LLM call; machine-checkable findings post once, and LLM tokens go to judgment calls.
- **Context pack** — reviewers get the project's conventions and the plan's recorded decisions alongside the diff, killing the context-gap false positives that waste triage time. Hard rule: the pack informs, it never defends the diff. Provenance is split: only repo-sourced decisions are labelled authoritative, and anything traceable to ticket text is marked *unverified*, with the reviewer told explicitly that a finding contradicting it is still a finding.
- **Delta re-review** — round 2+ reviews only the changes since the last reviewed commit, with each prior finding's outcome attached, so reviewers verify fixes instead of re-raising them. Large diffs are still truncated *with disclosure* so nothing gets reported as falsely missing.

### `/address-review` — honest triage + resolution

Correlates every comment **line by line against the actual diff**, then verdicts: **VALID** (fix it), **PARTIAL** (right instinct, wrong specifics — fix the real concern), **INVALID** (dismissed, citing the specific context the reviewer couldn't see; burden of proof is on dismissal, and that context must come from the code — a claim in a ticket comment is never grounds to dismiss a finding). For each valid finding it answers *why I missed it* with a real cause, implements the fixes (one commit per comment), pushes on a green build, and replies to every thread.

### Jira-driven loops — ticket in, status out

`/ship-feature PROJ-123` reads the ticket whole and decides where to start, so pointing it at the same ticket twice is safe and cheap:

| Entry | What it found | What it does |
|---|---|---|
| **E0** | no branch, no MR | the full loop: intake → plan → build → review → merge → report |
| **E1** | branch exists, no MR | resumes the build |
| **E2** | open MR, unresolved threads | **audits it** — addresses every open comment, then merges |
| **E3** | open MR, all threads clean | straight to the merge gate |
| **E4** | already merged | reports to the ticket only; touches no code |
| **E5** | MR closed unmerged, or two open MRs | stops and asks |

**The requirement chain** is what makes the final comment worth reading. `/ticket-intake` mints `REQ-1..n`, each with a **verbatim quote** from the ticket and its source, reconciling comments against the description chronologically — so a comment that changed the scope wins, and the requirement it replaced is shown struck through rather than quietly dropped. `/plan-feature` maps every REQ to a phase or an explicit **Won't-do with a reason**. `/build-feature` reports each one as `done` / `partial` / `not done`, where `done` requires a `file:line`, a commit sha *and* a test name — anything you can't evidence is downgraded automatically. `/ticket-report` renders that into the ticket.

The honest part is the **residue block**: every paragraph, comment, subtask and attachment that produced *no* requirement is listed with a reason, and coverage is reported as two numbers — requirements delivered, *and* source units mapped. A requirement that was never extracted can never show up as missing, so the second number is the only one that can see an omission. An empty residue block is treated as a red flag, not a clean bill of health.

And the rule that makes the status trustworthy: **partial coverage never lands in Done.** No override flag. A ticket marked Done with unbuilt requirements is worse than one left In Review, because it removes the one signal a human has to go and look.

### `/resolve-conflicts` — when the target moves

Merges the target into the branch rather than rebasing, so the MR's review threads stay anchored and nothing needs a force-push. Classifies every conflicted file before touching it: lockfiles and generated artifacts are regenerated (never hand-merged), migrations renumbered (never merged), additive lists unioned, logic conflicts resolved from *both* sides' intent. Then it proves the integration — no surviving markers, full gates green, and the **target-work check**: every hunk the target contributed is confirmed still present, because "took ours wholesale" silently reverts someone else's commit and green tests never notice. Same-migration edits, binaries, delete/rename conflicts and genuine guesswork all hard-stop with the tree untouched and the rollback command printed.

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
| **Atlassian MCP**, authorized *(Jira mode only)* | `/ticket-intake`, `/ticket-report` | `claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp/authv2 --scope user`, then `/mcp` to authorize |
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
| `SHIP_MERGE_MODE` | no | `ask` | default merge gate for `/ship-feature` (brief mode) |

### Jira settings *(ticket mode only)*

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `JIRA_SITE_URL` | only on a multi-site Atlassian account | — | which site to use; without it, an ambiguous account stops rather than guessing |
| `JIRA_STATUS_DONE` | no | `Done` → `Closed` → `Resolved` → `Complete` | preferred transition when every requirement landed |
| `JIRA_STATUS_PARTIAL` | no | `In Review` → `In Progress` → `Needs Info` | preferred transition when some didn't |

There is deliberately **no committed workflow-mapping file**. Transitions are read from the live
workflow at write time, because they vary per project and per current status — these variables
only express a preference among what actually exists.

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

**Is this Claude-specific?** Mostly no, with one honest exception. Claude Code is supported natively (plugin + skills directory), and the seven non-ticket skills are plain Markdown procedures with no model-specific instructions — see [Works with](#works-with). The reviewer side speaks the OpenAI-compatible chat-completions API, so reviewers are swappable too. But `/ticket-intake` and `/ticket-report` reach Jira through the Atlassian MCP server, which is a Claude-Code-specific transport; on another harness those two skills won't run, and the rest of the loop still will.

**Does it work with Jira?** Yes — that's what ticket mode is. `/ship-feature PROJ-123` reads the issue, builds it, merges it, and writes the outcome back with per-requirement coverage. It needs an authorized Atlassian MCP connection (see [Configuration](#configuration)). Because that connection is OAuth-based, **ticket mode does not run headless** — no cron, no CI. If you need unattended ticket runs, the four operations the skills use (fetch, list transitions, comment, transition) are small enough to reimplement over the Jira REST API with a token, and the skills are structured so that slots in without touching anything else.

**Does it work with Linear / GitHub Issues?** Not yet. `/ticket-intake` and `/ticket-report` are the only tracker-aware skills, and a variant of each is the whole port — the requirement contract between them is tracker-neutral.

**Does it work with GitHub?** Yes. The skills are written GitLab-first (`glab`), and each MR-touching skill carries a "Forge note" mapping every command to its `gh` equivalent — the agent applies the mapping automatically when the repo is on GitHub.

**Can I use just one or two skills?** Yes — each is standalone. `/address-review` works on any MR with comments (including a colleague's); `/mr-review` can review a hand-written change; `/ticket-intake` is a read-only way to see what a ticket actually asks for; `/ticket-report` can audit and report on an MR the loop never built; `/land` runs off any merged MR.

**What does a loop cost?** Author-side usage under your coding agent's existing plan, plus reviewer API tokens per round (one OpenAI and/or one xAI call per round, diff-sized). `--max-rounds` caps it; the Ship Report shows how many rounds convergence took.

**How autonomous is it, really?** In brief mode, deliberately bounded: the loop automates *iteration* and stops at the merge gate, an architectural concern, a red build, or the planner's questions.

Ticket mode is genuinely more autonomous, and it's worth being precise about what that buys and what it doesn't. It plans without asking, merges without asking, and updates Jira without asking. What it does **not** do is decide things a human was meant to decide — every hard stop still fires, it just hands off (MR marked Draft, a Jira comment naming the decision needed, ticket never moved to Done, state left resumable) instead of blocking on a prompt you're not there to answer. Concretely, it stops rather than guessing on: a question about authorization, retention, money, PII, external contracts or auth; an architectural concern; a red build or pipeline; a conflict it can't resolve safely; two open MRs for one ticket; and any ticket text that reads like it's trying to direct the loop. `--merge=auto` is unreachable from any of those.

**Is it safe to point this at a ticket anyone can file?** Read the capability envelope in [`skills/ship-feature/SKILL.md`](skills/ship-feature/SKILL.md) before you decide. Jira comment permission is usually far wider than repo write, so ticket text is treated as untrusted input throughout: it can never reach CI config, deploy manifests, credentials, destructive SQL, other branches, or other issues, and it is labelled *unverified* everywhere it reaches a reviewer. That containment is prose in a skill file, not a sandbox — it raises the bar considerably, and it is not a substitute for caring who can file tickets in your project.

**Can reviews run automatically on every MR instead of on demand?** Yes — the prompt templates in [`skills/mr-review/SKILL.md`](skills/mr-review/SKILL.md) are drop-in for a webhook service (receive the MR event, fetch the diff, call the reviewers, post the briefs). Pair that with `/address-review` and the loop runs itself on every MR your team opens.

## Contributing

The skills are plain Markdown prompts — reading one is understanding it, and improving one is a text edit. Issues and PRs welcome: sharper review prompts, more reviewer backends, forge support, other issue trackers, portability notes for more agent harnesses. Before opening either, read [CONTRIBUTING.md](CONTRIBUTING.md) — it's short, and it spells out the project's non-negotiable contracts (agent-agnosticism, reviewer independence, honest exits, noise-aversion, traceability) and what evidence a behavior-changing PR needs.

## License

[MIT](LICENSE)
