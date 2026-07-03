# Claude Feature Workflow

Three [Claude Code](https://claude.com/claude-code) skills that cover the full lifecycle of shipping a feature — from a rough idea to a merged, reviewed change:

```
/plan-feature  →  /build-feature  →  /address-review
   (plan)            (code)           (review loop)
```

Each skill is a disciplined, mostly-autonomous agent with deliberate pause points. Together they form a pipeline: the planner writes a plan file, the builder executes it phase by phase and raises the MR/PR, and the review skill triages and resolves whatever your reviewers (human or AI) throw at it.

## The skills

### `/plan-feature` — product-owner + architect planner

Give it a few paragraphs on what you want to build. It:

1. **Studies your codebase first** — structure, conventions, the nearest existing feature, reusable pieces — and never plans from memory. Every integration point is cited as `file:line`.
2. **Frames the feature by what the user achieves** — outcome, not mechanism — and names the smallest version that delivers it.
3. **Challenges the brief** — one batched round of only the genuinely blocking questions, each with a recommended answer. Defaultable decisions are made and *recorded* so you can veto by reading.
4. **Audits every UI element** against "less is more": each button/field/label must name what the user achieves by it, or it's cut. Empty/loading/error states are designed in the plan, not discovered during coding.
5. **Proves feasibility** against real code, then writes a phased plan — summary of what changes, summary of how, then small self-contained phases (goal, files, elaborate changes, verify step, commit message) that a coding agent can execute without coming back with questions.

Output: a plan file in `docs/plans/`, ready for `/build-feature`.

### `/build-feature` — disciplined plan executor

Takes the plan file (defaults to the newest in `docs/plans/`) and:

- Branches **fresh from main** — always pulls latest, never builds on a stale base.
- Reads the plan **whole, line by line**, then re-grounds every phase in the current code before writing.
- Codes with clean-code discipline: SOLID applied not recited, DRY with judgment, reuse-first (more code = more maintenance surface), surgical changes only.
- **No unflagged hacks**: a forced workaround exists only with a `HACK:` comment and an entry in the hack ledger.
- **Async by default, synchronized by justification**: blocking I/O on hot paths is treated as a bug; every deliberate sync point (transaction, unique constraint, idempotency key, lock) goes in a synchronization ledger with the invariant it protects.
- Gates every phase with type-check + tests (unpiped, logged), commits granularly (one commit per logical change), and finishes with a full verification sweep — including a grep audit of every consumer of every changed surface.
- **Always raises the MR/PR** when green, with a fixed four-section description: **Problem / Solution / Technical details / Review notes** (the riskiest change, named — "nothing in particular" is banned).

Two hard stops only: an architectural concern (it asks, with a recommendation) and a red build (it never pushes red).

### `/address-review` — review-comment triage and resolution

Built for the loop where AI writes the code, you raise an MR, and reviewer agents (or humans) comment with less context than the author has. It:

- Pulls every comment, splits compound notes, and correlates each one **line by line against the actual diff** — never judging from the reviewer's summary alone.
- Assigns honest verdicts: **VALID** (fix it), **PARTIAL** (right instinct, wrong specifics — fix the real concern), **INVALID** (dismissed, citing the specific context the reviewer couldn't see).
- For every valid finding, answers **"why I missed it"** with a real blind-spot category — the learning signal is the point.
- Implements all valid fixes (separate commit per comment), pushes on a green build, and replies to every thread on the MR — fixed ones marked done with the commit, invalid ones explained. One pass, no pausing; the only hard stop is a red build.

## Install

### Option 1: as a plugin (one command)

```
/plugin marketplace add ohyesgocool/claude-feature-workflow
/plugin install feature-workflow@claude-feature-workflow
```

### Option 2: copy the skills

```bash
git clone https://github.com/ohyesgocool/claude-feature-workflow.git
cp -r claude-feature-workflow/skills/* ~/.claude/skills/
```

New Claude Code sessions will pick them up automatically.

## Requirements & notes

- **Forge CLI**: the MR steps use `glab` (GitLab) with `gh` (GitHub) equivalents noted inline. `/address-review` is written GitLab-first; a note at the top of the skill maps each command to its `gh` equivalent.
- **House rules travel with your project**: all three skills read your project's `CLAUDE.md` and encode its constraints (migration conventions, deploy ordering, test policy) instead of hardcoding any. Edit the `SKILL.md` files freely — they're prompts, not code.
- The skills reference each other (`/build-feature` hands off to `/address-review`), so installing all three gives the full loop; each also works standalone.

## License

MIT
