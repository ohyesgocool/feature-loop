# Contributing

Thank you for considering a contribution to Feature Loop.

One thing makes this project unusual to contribute to: **the "code" is Markdown**. Each skill
is a plain-text operating procedure that an AI coding agent executes. That means reading a
skill is understanding it, improving one is a text edit — and "does it work" can only be
proven by *running* it, not by unit tests. Contributions are judged accordingly: on clarity,
on the behavior they produce in real runs, and on whether they respect the project's contracts
(below).

## What contributions are welcome

### Bug reports

A "bug" here is a skill doing the wrong thing in a real run: a loop that games its own
convergence, a reviewer prompt that produces noise, a step that breaks on GitHub when it works
on GitLab, an instruction agents consistently misread.

- Search existing issues first. Duplicates will be closed with a pointer to the original.
- State: **which skill**, **which agent/harness** you ran it in, **what it did**, and **what
  it should have done**. A sanitized excerpt of the run (the skill's report output, the MR
  comment it posted) is worth more than a paragraph of description.
- Reports without enough detail to reproduce the behavior may be closed. That's not hostility
  — a behavior nobody can reproduce is a behavior nobody can fix.

### Feature suggestions

- Check [`docs/plans/`](docs/plans/) first — the roadmap lives there, and several ideas
  (plan-review stage, feature queue, parallel worktrees, post-landing watch) are already
  designed and deliberately deferred. A suggestion that's already on the roadmap will be
  linked there and closed.
- Search existing issues for duplicates.
- The burden of the case is on the requester: what gap does it close for an engineering team
  using the loop, and why does it belong in the core rather than in your own fork of a skill?
  Whether a feature fits the project's scope is ultimately the maintainer's call.

### Improvements that are especially wanted

- **Sharper review prompts** — measurably better findings per token (bring before/after
  evidence from real MRs).
- **More reviewer backends** — any OpenAI-compatible endpoint qualifies; keep the contract
  (see below).
- **Forge support** — better `gh` equivalences, other forges (Gitea, Bitbucket) as "Forge
  note" mappings.
- **Tracker support** — Linear, GitHub Issues, Azure Boards as `/ticket-intake` +
  `/ticket-report` variants. The requirement contract between them is tracker-neutral, so a port
  is those two files and nothing else. A REST-plus-token Jira variant that works headless is
  equally welcome.
- **Portability notes** — how to load the skills in more agent harnesses, with graceful
  fallbacks for missing features.
- **Docs** — corrections, clearer explanations, honest FAQ entries.

## The contracts (non-negotiable in review)

Every PR is checked against these; they are what make the loop trustworthy:

1. **Agent-agnostic.** No model-specific or harness-specific instruction without a graceful
   fallback ("your harness's task tracker, or a plain list"). Nothing vendor-locked.

   **The one carve-out:** `/ticket-intake` and `/ticket-report` reach Jira through the Atlassian
   MCP server and nothing else — harness-specific, vendor-specific, and OAuth-only. It is
   documented as such in the README rather than papered over. The terms of the carve-out: every
   other skill runs without it, a ticket skill that can't bind its tools **stops with the setup
   command instead of degrading silently**, and it never invents a fallback (no scraping, no
   guessed token, and above all no reconstructing a ticket's content from its key). A second
   tracker or a headless transport must meet the same terms.

2. **Reviewer independence.** Reviewers are *different models* from the author agent; their
   output is posted **verbatim** — never edited, filtered, or softened. Context given to
   reviewers informs, it never defends the diff.

   This is also why context has **provenance**. Only repo-sourced conventions and decisions may
   be labelled authoritative to a reviewer; anything traceable to ticket text is marked
   unverified, with the reviewer told a finding contradicting it is still a finding. Without
   that split, anyone who can comment on a ticket can silence both reviewers through a
   documented data path — and no PR may reopen it.

3. **Honest exits.** No change may let the loop game its own convergence: threads are never
   resolved without a fix or a stated dismissal, "merged/shipped" is only claimed after
   verification, caps and truncations are always disclosed. A finding is never dismissed on the
   strength of a claim in a ticket; dismissal requires the code.
4. **Noise-aversion.** Nothing posts a comment, report line, or file the team didn't need.
   "No issues found" comments, decorative output, and speculative configuration are all
   rejected on sight. One outward artifact per run per system: one brief per reviewer, one reply
   per thread, **one Jira comment per loop** — each guarded by an idempotency check before the
   write, because a second run must not re-notify everyone who was already told.
5. **Keys via environment only.** Never in files, never in prompts sent to reviewers, never
   echoed.
6. **Traceability.** On a ticket-driven run every requirement the ticket states is numbered with
   a verbatim quote, mapped to a phase or an explicit Won't-do, and reported back with its status
   and evidence. Every source unit that produced *no* requirement is listed too — that residue is
   the only place an omission can become visible, so removing or weakening it is not an
   optimization. **A requirement that silently disappears between intake and the ticket comment
   is the highest-severity bug in this project**, above any crash: the loop's whole claim is that
   what didn't get built is stated as clearly as what did.

## Pull requests

- **Open an issue first** for anything that changes a skill's behavior, and wait for a nod
  before investing serious effort. Typo- and docs-level fixes can go straight to a PR.
- **Small and focused.** One concern per PR. A PR that rewrites three skills to fix one bug
  will be asked to shrink.
- **Bring evidence.** For behavior changes, include what a real run produced before and after
  (sanitized report output or MR comments). The skills cannot be unit-tested; your run *is*
  the test.
- **Follow the project's own commit rules** — they're in
  [`skills/build-feature/SKILL.md`](skills/build-feature/SKILL.md) and this repo practices
  them: `<prefix>: <imperative summary>`, ≤ 50 characters, no trailing period, body only when
  the *why* needs room. PRs with a messy history may be squashed on merge.
- Review takes real maintainer time; acceptance is discretionary. A declined PR is not a
  judgment of you — it's a scope decision.

## Be respectful

- No aggression, no demands, no deadlines. Maintainers and contributors here owe you nothing;
  everything you receive is given freely.
- Read the [README](README.md) and search issues and [`docs/plans/`](docs/plans/) before
  asking a question that's already answered.
- Don't nitpick or bikeshed in threads — if a discussion has been settled, relitigating it
  costs everyone.
- If the project's direction doesn't suit you, the MIT license is your friend: fork it, bend
  it to your needs, and godspeed.

## Licensing

By contributing, you agree that your contributions are licensed under the [MIT License](LICENSE),
the same license as the project.
