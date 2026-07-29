# Setup

Line-by-line setup for Feature Loop, from zero to a first running loop. Every step is a
command plus a **checkpoint** (what you should see, and what to do if you don't).

> **If you are an LLM setting this up:** execute the steps in order, run every checkpoint,
> and do not proceed past a failed checkpoint — fix it using the "if not" instruction first.
> Steps marked **[human]** need input only your user can provide (accounts, API keys,
> browser logins); ask for exactly that input, then continue.

**What you'll have at the end:** nine slash-skills in your coding agent
(`/ship-feature`, `/ticket-intake`, `/plan-feature`, `/build-feature`, `/mr-review`,
`/address-review`, `/resolve-conflicts`, `/ticket-report`, `/land`), a forge CLI that can raise
and comment on MRs/PRs, at least one external reviewer wired up, and — if you want ticket-driven
runs — an authorized Atlassian MCP connection.

---

## Part 1 — Prerequisites

### Step 1.1: Check the basics

```bash
git --version && jq --version && curl --version | head -1
```

**Checkpoint:** three version lines, no "command not found".
**If not:** install the missing one — macOS: `brew install git jq curl` · Debian/Ubuntu:
`sudo apt-get install -y git jq curl`.

### Step 1.2: Check your coding agent

Feature Loop runs inside an AI coding agent. Claude Code is supported natively:

```bash
claude --version
```

**Checkpoint:** a version number prints.
**If not:** install Claude Code from https://claude.com/claude-code, or skip to
**Part 6 (other agents)** if you use Cursor, Codex CLI, Gemini CLI, OpenCode, etc.

### Step 1.3: Check your forge CLI — GitLab or GitHub

Pick the one your repos live on (both is fine):

```bash
glab --version   # GitLab
gh --version     # GitHub
```

**Checkpoint:** a version for at least one of them.
**If not:** macOS: `brew install glab` or `brew install gh` · other platforms:
https://gitlab.com/gitlab-org/cli or https://cli.github.com.

### Step 1.4 [human]: Authenticate the forge CLI

```bash
glab auth login   # GitLab — or:
gh auth login     # GitHub
```

Follow the interactive prompts (browser login or token).

**Checkpoint:**

```bash
glab auth status   # or: gh auth status
```

prints "Logged in". **If not:** re-run the login; for self-hosted GitLab pass
`--hostname <your-host>`.

---

## Part 2 — Install the skills (Claude Code)

Two options. Option A is one command and gets updates; Option B is a plain copy you own.

### Option A: as a plugin

Inside Claude Code, run:

```
/plugin marketplace add ohyesgocool/feature-loop
/plugin install feature-workflow@feature-loop
```

### Option B: copy the skill folders

```bash
git clone https://github.com/ohyesgocool/feature-loop.git /tmp/feature-loop
cp -r /tmp/feature-loop/skills/* ~/.claude/skills/
```

### Step 2.1: Verify the install

Start a **new** Claude Code session (skills load at session start), then type `/` and look
for the skills, or ask: *"Do you have the /ship-feature skill?"*

**Checkpoint:** `/ship-feature`, `/ticket-intake`, `/plan-feature`, `/build-feature`,
`/mr-review`, `/address-review`, `/resolve-conflicts`, `/ticket-report` and `/land` are all
available.
**If not (Option A):** run `/plugin` and confirm `feature-workflow` shows as installed and
enabled. **If not (Option B):** `ls ~/.claude/skills/` must list the nine folders, each
containing a `SKILL.md`; re-run the copy if any are missing.

---

## Part 3 — Wire up the external reviewers

`/mr-review` sends your MR diff to reviewer models that did not write the code. One key
enables one reviewer; set at least one. Everything else in Feature Loop works without any
key — only `/mr-review` (and the review stage inside `/ship-feature`) needs them.

### Step 3.1 [human]: Get the key(s)

- **OpenAI** (GPT correctness reviewer): create a key at
  https://platform.openai.com/api-keys
- **xAI** (Grok quality reviewer, optional): create a key at https://console.x.ai

### Step 3.2: Set the environment variables

**Option A — shell profile** (works for every agent). Append to `~/.zshrc` or `~/.bashrc`:

```bash
export OPENAI_API_KEY="sk-your-key-here"
export XAI_API_KEY="xai-your-key-here"     # optional
```

Then reload: `source ~/.zshrc` (or open a new terminal).

**Option B — Claude Code settings** (applies to every project, survives shell changes).
Merge into `~/.claude/settings.json` (create the file if absent; if it exists, add the keys
inside the existing `env` object rather than replacing the file):

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-your-key-here",
    "XAI_API_KEY": "xai-your-key-here"
  }
}
```

Never commit these keys to any repository.

### Step 3.3: Verify the reviewers respond

```bash
curl -sS https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.5","messages":[{"role":"user","content":"Reply with exactly: ok"}]}' \
  | jq -r '.choices[0].message.content // .error.message'
```

**Checkpoint:** prints `ok`. **If not:** the printed error message tells you why — an
invalid key, no billing, or a model name your account can't access (set `OPENAI_MODEL` to
one you can; see Step 3.4). Repeat for xAI with `https://api.x.ai/v1/chat/completions`,
`$XAI_API_KEY`, and model `grok-4.3`.

### Step 3.4: Optional overrides

Only set these if the defaults don't fit your account:

```bash
export OPENAI_MODEL="gpt-5.5"              # default
export XAI_MODEL="grok-4.3"                # default
export MR_REVIEW_MAX_DIFF_CHARS="120000"   # default
export SHIP_MERGE_MODE="ask"               # default: ask | auto | never
```

---

## Part 3.5 — Connect Jira (optional — needed only for `/ship-feature PROJ-123`)

Skip this entire part if you drive the loop from written briefs. Nothing else depends on it.

### Step 3.5.1: Register the Atlassian MCP server

```bash
claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp/authv2 --scope user
```

### Step 3.5.2 [human]: Authorize it

Run `/mcp` in an **interactive** Claude Code session, select `atlassian`, and complete the
browser login. This is OAuth — it cannot be completed in a headless or piped session, and there
is no API-token alternative in these skills.

**Checkpoint:** ask the agent *"list your Atlassian tools"*. You should see `getJiraIssue`,
`getTransitionsForJiraIssue`, `addCommentToJiraIssue`, `transitionJiraIssue` and
`getAccessibleAtlassianResources`.
**If not:** `/mcp` will show the server as needing authorization — re-run it. Until the tools
bind, `/ticket-intake` stops with this command rather than guessing at the ticket's contents.

### Step 3.5.3: Optional Jira overrides

```bash
export JIRA_SITE_URL="https://yourcompany.atlassian.net"  # only for multi-site accounts
export JIRA_STATUS_DONE="Ready for release"               # default: Done > Closed > Resolved
export JIRA_STATUS_PARTIAL="Needs Info"                   # default: In Review > In Progress
```

These express a *preference* among the transitions your workflow actually offers — they never
invent one. If your project's Done sits behind `/land` rather than the merge, set
`JIRA_STATUS_DONE` to your pre-release status and let `/land` finish the job.

---

## Part 4 — Optional but recommended

### Step 4.1: Static analysis tools

`/mr-review` runs these before spending LLM tokens, and skips them silently if absent:

```bash
brew install gitleaks semgrep    # macOS; see each tool's docs for other platforms
```

**Checkpoint:** `gitleaks version` and `semgrep --version` both print.

### Step 4.2: Ignore loop state in your project

In each repo where you'll run loops:

```bash
echo ".loop/" >> .gitignore
```

(`.loop/` holds transient resume-state and the run's requirements list. The only files Feature
Loop commits to your repo are plans under `docs/plans/` and, for `/land`, `docs/landing.md`.)

---

## Part 5 — First run

### Step 5.1: Dry-run the planner (no keys, no MR, reversible)

In Claude Code, inside any project you work on:

```
/plan-feature Add a --version flag to the CLI that prints the package version
```

**Checkpoint:** it explores the codebase, possibly asks 1–3 questions, and saves a plan to
`docs/plans/`. Nothing else is touched.

### Step 5.2: Run a small loop end to end

Pick something genuinely small for the first loop:

```
/ship-feature Add a --version flag to the CLI that prints the package version
```

**Checkpoint:** plan → branch → commits → MR raised → reviewer brief(s) posted as MR
comments → findings triaged and fixed → **READY TO MERGE** prompt.
**If the review stage fails:** the report names the reviewer and the API error — almost
always Step 3.3 unfinished. Everything up to the MR still stands; fix the key and run
`/mr-review` on the open MR.

### Step 5.3: First ticket-driven run (only if you did Part 3.5)

Start read-only. This touches nothing and verifies the whole Jira wiring:

```
/ticket-intake PROJ-123
```

**Checkpoint:** it prints the ticket's status and comment counts, a numbered requirements table
where every row carries a verbatim quote from the ticket, and a "Sources not mapped to a
requirement" block naming anything it didn't turn into a requirement (including attachments,
which it never downloads). Read that last block — it's where a misread ticket shows up.
**If it stops instead:** it will name the reason — unauthorized MCP (redo Step 3.5.2), an
ambiguous Atlassian site (set `JIRA_SITE_URL`), or an Epic (name a child issue).

Then a cautious end-to-end, with the merge gate closed:

```
/ship-feature PROJ-123 --merge=never
```

**Checkpoint:** branch `feat/PROJ-123-<slug>`, a plan under `docs/plans/` containing a
**Requirements coverage** section, an MR whose Technical details carry the per-requirement
table, reviewer briefs posted and triaged — and the run ending with the MR open, plus a Jira
comment saying so. Once that reads right, drop the flag: `/ship-feature PROJ-123` runs the same
loop through the merge and moves the ticket.

---

## Part 6 — Other agents (Cursor, Codex CLI, Gemini CLI, OpenCode, …)

The skills are plain Markdown procedures; nothing in them is Claude-specific.

1. Clone the repo (Step under Option B above).
2. Register each `skills/<name>/SKILL.md` as your tool's equivalent of a custom command,
   rule, or prompt file (e.g. a Cursor command, a prompt your CLI agent loads on demand).
3. Do Parts 1, 3, and 4 exactly as written — they're agent-independent shell setup.
4. Where a skill mentions an optional harness feature (task tracker, search subagents), your
   agent uses its own equivalent or a plain fallback; the skills spell those out inline.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Skills don't appear in `/` | session predates install | start a new session |
| `/mr-review` says "no reviewer configured" | env vars not visible to the agent | `echo $OPENAI_API_KEY` in the agent's own shell; redo Step 3.2 for the option you chose |
| Reviewer call fails with model error | account lacks the default model | set `OPENAI_MODEL` / `XAI_MODEL` to a model you can access |
| MR steps fail with 401/404 | forge CLI not authenticated for this host | Step 1.4 with the right `--hostname` |
| Loop restarted from zero after a crash | state file was deleted or branch changed | keep `.loop/` between sessions; resume runs on the feature branch |
| `/ticket-intake` says no Atlassian tools | MCP server registered but not authorized | `/mcp` in an interactive session (Step 3.5.2); it will not fall back to guessing |
| It picked the wrong Jira site | multi-site Atlassian account | set `JIRA_SITE_URL` to the site you meant |
| Ticket didn't move to Done | coverage was partial, or the workflow has no reachable Done from the current status | read the comment's Verification line — it names which, and lists the transitions that *were* available |
| Ticket transition needs a field | the Done screen requires a resolution / fix version | the report names the exact fields; set them by hand once, or point `JIRA_STATUS_DONE` at a status without a screen |
| `/resolve-conflicts` stopped instead of resolving | hard-stop class: lockfile, migration, binary, delete/rename | resolve by hand — the branch is exactly as it was, and the report prints the anchor sha |
| Two open MRs for one ticket | a stale branch was never closed | close or merge one; the loop refuses to guess which is "the" MR |

Stuck on something this file doesn't cover? Open an issue — see
[CONTRIBUTING.md](CONTRIBUTING.md) for what to include.
