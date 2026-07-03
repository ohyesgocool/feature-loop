---
name: plan-feature
description: |
  Product-owner + architect planning agent: take a rough brief (a few paragraphs on what to
  build), study the actual codebase first, frame the feature by what the USER achieves, challenge
  the brief and ask only the clarifying questions that genuinely block planning (with a
  recommendation each), audit every proposed UI element against "less is more", verify the design
  is buildable against real code (file:line), then produce a phased, executable plan — summary of
  what changes, summary of how, then small technically-elaborate phases a coding agent can pick up
  and run without extra context. The plan is saved as a markdown file for handoff.
  One deliberate pause: the clarifying-questions gate. Everything else runs end-to-end.
  Use when: planning a new feature, "plan this", "create a plan for", "how should we build X",
  turning a product idea into an implementation plan, before starting any multi-phase build.
argument-hint: "[feature brief — paragraphs on what to build, or a path to a brief file]"
---

# Plan Feature

Turn a rough product brief into a plan that is **grounded in the real codebase**, **framed by what
the user achieves**, and **broken into phases small enough that a coding agent can execute each one
without asking anything**. You are acting as product owner first, architect second, and only then
as the person who writes the task list.

The skill runs in one pass with **exactly one allowed pause**: Step 4's clarifying-questions gate.
Before that gate you gather context; after it you never stop — UI audit, feasibility check, plan
writing, and file save flow straight through.

If the user provided arguments: `$ARGUMENTS` — the brief, or a path to a file containing it. With
no arguments, use the brief from the conversation (the paragraphs the user just wrote).

---

## Process

Follow the steps in order. Do not skip Step 2 — a plan written from memory of the codebase instead
of the codebase itself is the failure mode this skill exists to prevent.

### Step 1: Ingest the brief

Read the brief from `$ARGUMENTS` (inline text or a file path) or from the conversation. Restate it
in 2–4 sentences: what is being asked for, in your own words. If the restatement already exposes a
contradiction or an impossibility, note it — it becomes a Step 4 question.

Also load the house context that constrains any plan:
- The project's `CLAUDE.md` and any global one (deployment model, style rules, commit discipline).
- Any persistent memory or notes for constraints relevant to the touched area — deploy ordering,
  migration conventions, prior decisions on the same feature area.

### Step 2: Codebase recon — never plan from memory

Study how this codebase actually does things **before** forming any opinion on design. Use
parallel Explore agents for breadth (structure, naming conventions, similar features) and direct
Reads for the specific areas the feature will touch. Establish, with `file:line` references:

- **Structure** — where this kind of feature lives (routes, services, components, schema, jobs).
- **The nearest existing feature** — the closest thing already built. Its shape is the default
  shape for the new feature; deviating from it needs a stated reason.
- **Reusable pieces** — existing utilities, components, endpoints, tables that already do part of
  the job. Reuse first, generic second, new code last. Grep before proposing anything new.
- **Data model** — current tables/columns the feature reads or extends; whether a migration is
  needed (and check how this project ships migrations — some apply standalone idempotent SQL
  files manually rather than auto-migrating; follow the convention you find).
- **Integration points** — the exact files/functions the new code will plug into.
- **Blast radius** — every consumer of anything you'll change (a grep pass per touched surface, so
  the plan doesn't leave one wired path and one silently broken path).

Keep the recon output as a short map — it feeds Steps 5–7 and goes into the plan's appendix.

### Step 3: Product framing — what does the user achieve?

Before any technical design, answer as a product owner:

- **Who** is the user of this feature (which role, in which moment of their workday)?
- **What do they achieve** — the outcome in their words, not the mechanism? "Sees a dashboard" is
  a mechanism; "knows within 10 seconds whether yesterday's campaign worked" is an achievement.
- **What is the smallest version that delivers that achievement?** Everything beyond it is a
  candidate for cutting or deferring — name those cuts explicitly.
- **Success criteria** — how you'd verify, after building, that the user actually achieves it.

If you cannot state the achievement crisply, that is itself a Step 4 question — do not paper over
it with a generic one.

### Step 4: Challenge the brief — the clarifying-questions gate

Interrogate the brief and your recon for everything that is ambiguous, contradictory, or silently
decided. For each open point, classify honestly:

- **Blocking** — different answers produce materially different plans (scope, data model,
  who-sees-what, build-vs-defer). These go to the user.
- **Defaultable** — a sensible default exists that a senior engineer wouldn't argue with. Decide
  it yourself and **record it in the plan's Decisions section** so the user can veto by reading.

Ask **only the blocking questions**, all at once (one round, not a drip), and for every question
**include your recommended answer and why** — never present bare options. Use AskUserQuestion when
available, otherwise a numbered list. 2–5 questions is the healthy range; zero is legitimate when
the brief plus recon genuinely covers everything — in that case say so explicitly ("no blocking
questions; N defaults recorded in Decisions") and continue without pausing.

This is the only pause in the skill. Once answered (or skipped), run to the end.

### Step 5: UI audit — less is more

Only if the feature has a UI surface. For **every** proposed element — button, field, label,
toggle, tab, empty state, notification — fill in one row:

| Element | What the user achieves by it | Verdict |
|---|---|---|
| {element} | {achievement in the user's terms} | keep / cut / defer |

Rules of the audit:
- If the achievement column is empty, jargon, or "flexibility", the verdict is **cut**. Every
  element earns its place; when in doubt, leave it out — a missing control can be added when
  someone asks; a cluttered screen can't easily be unshipped.
- Copy is user language, not system language ("Check my mailbox now", not "Trigger poll").
- **Empty, loading, and error states are designed here**, per screen — not discovered during
  coding. A state you didn't plan is a state that ships broken.
- Everything that survives the audit must **work** in the plan — each kept element traces to a
  phase that wires it end-to-end, not a mock. No dead buttons, no UI written before its backend
  phase exists.

The audit table goes into the plan verbatim, cuts included — cut rows document restraint.

### Step 6: Technical & architectural feasibility

Now design the build against the Step 2 map and prove it's buildable:

- **Data flow** — from entry point to storage and back, naming the actual files/functions
  (`file:line`) each hop touches. If a hop lands somewhere you haven't read, go read it now.
- **Schema changes** — exact tables/columns, following the project's migration convention
  (numbered sequence, idempotency, manual vs auto apply — match what the repo already does).
- **Cross-repo / deploy ordering** — if the feature spans repos or services, state the safe
  deploy order (e.g. backend → migration → frontend) and which phases belong to which repo.
- **Risks** — the 2–3 things most likely to sink this design (a boundary that doesn't serialize,
  a race, a permission model gap), each with the mitigation the plan adopts.
- **Testing strategy** — foundational work (schema, permissions, retrieval) ships with regression
  tests in the same phase, not as a follow-up; permissive defaults are proven by a test.
- **Feasibility verdict** — one line: buildable as designed / buildable with the named change to
  the brief / not buildable this way (and the alternative). Never hand over a plan you couldn't
  build yourself.

If feasibility work overturns a Step 4 default, update the Decisions section — don't silently
re-plan.

### Step 7: Write the plan

Produce the plan in exactly this shape:

```
# Plan: {feature name}

{2–3 sentence restatement of the brief}
Branch: {feature branch name, or "direct to main" for routine single-phase work}

## What changes (summary)
{5–10 bullets of user-visible and structural changes — what exists after that didn't before.
Readable by a non-engineer.}

## How (approach summary)
{1–2 paragraphs: the architectural approach, what is reused vs. new, deploy ordering if
cross-repo. Readable in 60 seconds.}

## Product framing
- User & moment: {…}
- Achievement: {…}
- Smallest version: {…} · Deferred: {…}
- Success criteria: {…}

## Decisions
{Every Step 4 answer and every default you chose: "Q → answer (user)" or "Q → default: X,
because Y". This section is the veto surface — a wrong default gets caught here.}

## UI inventory
{The Step 5 audit table, kept and cut rows.}

## Phases

### Phase 1: {title}
- **Goal:** {one sentence — what exists and works after this phase}
- **Files:** {each file to create/change, with the anchor `path/to/file.ts:line` where known}
- **Changes:** {technically elaborate — the functions to add/modify, signatures, the shape of
  the data, the edge cases to handle. Enough that a coding agent starts typing, not searching.}
- **Verify:** {the concrete check — the test to write and pass, type-check, build, or the manual
  flow to drive. "It works" is not a verify.}
- **Commit:** `feat: {…}` {one logical change per commit}

### Phase 2: {…}
{…}

## Risks
{The Step 6 risks with mitigations.}

## Recon appendix
{The Step 2 map — structure, nearest feature, reusable pieces, integration points — so the
coding agent inherits the context without redoing the exploration.}
```

Phase discipline — this is what makes the plan executable:
- **Small**: one sitting each; a phase description that needs its own sub-plan is two phases.
- **Ordered and green**: each phase leaves type-check, tests, and build passing; no phase depends
  on a later one. Cross-repo phases are grouped and ordered by the deploy sequence.
- **Self-contained**: a coding agent given only that phase block plus the recon appendix can
  execute it — no "as discussed above", no unresolved "figure out the best way to…".
- **Verified**: every phase has a verify step that would actually fail if the phase were done
  wrong. Schema/permission phases include their regression tests inside the phase.
- **Backend before UI**: a UI element's phase comes after the phase that makes it functional.

### Step 8: Save and hand off

Write the full plan to `docs/plans/{yyyy-mm-dd}-{feature-slug}.md` in the repo (create the
directory if needed; if the project already keeps plans somewhere else, match that convention
instead). Then present in chat: the **What changes** and **How** summaries in full, the phase list
as one line per phase, any blocking risks — and the file path. Do not paste the entire plan into
chat; the file is the artifact.

Close with the handoff line: which phase to start with, and whether the work belongs on a feature
branch (multi-phase / cross-repo → feature branch, merged atomically) or direct to main (routine,
single-phase).

---

## Rules

- **The codebase is the source of truth, not your memory of it.** Every integration point, reused
  utility, and file path in the plan comes from Step 2 reading, cited as `file:line`. If you
  didn't read it this run, you don't get to cite it.
- **Achievement before mechanism.** No design work until Step 3 names what the user achieves. A
  feature that can't state its achievement gets questioned, not planned.
- **One question round, recommendations always.** Blocking questions go to the user once, batched,
  each with a recommended answer and why. Everything defaultable is decided and recorded in
  Decisions — silent decisions are the bug; recorded defaults are the feature.
- **Less is more, enforced per element.** Every UI element passes the achievement test or is cut.
  Empty/loading/error states are part of the plan, not an afterthought. Nothing ships decorative.
- **Nothing speculative.** No abstractions for single use, no configurability nobody asked for, no
  phase that exists "for later". The smallest version that delivers the achievement, plus named
  deferrals.
- **Reuse first.** An existing utility/component/table that does 80% of the job beats a new one
  that does 100%. Proposing new infrastructure requires showing the grep that found nothing.
- **Every phase is agent-executable.** Goal, files, elaborate changes, verify, commit — if a
  competent coding agent would need to come back with a question, the phase isn't written yet.
- **Feasibility is your signature.** The plan asserts you could build every phase yourself against
  the code you read. "Should be possible" is not a phase.
- **Respect house constraints.** Migration conventions, deploy ordering, test policy, branching
  strategy, data-privacy rules — read the project's `CLAUDE.md` and encode its constraints in the
  plan so the coding agent doesn't rediscover them.
- **The plan is the deliverable, not the build.** This skill never starts implementing. It ends at
  the saved file and the handoff line.
