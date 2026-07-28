---
name: resolve-conflicts
description: |
  Integrate a moved target branch into a feature branch and resolve the conflicts — safely enough
  to run unattended. Merges the target in rather than rebasing, so the MR's existing review
  threads stay anchored and nothing needs a force-push. Classifies every conflicted file before
  touching it: lockfiles and generated artifacts are regenerated rather than hand-merged,
  migrations are renumbered rather than merged, additive lists are unioned, and logic conflicts
  are resolved by reading both sides' intent — never by picking a side. Then it proves the
  integration: no surviving markers, every hunk the target contributed still present in HEAD
  (the "took ours wholesale" check), full gates green, and the feature re-verified.
  Hard stops are the point: same-migration edits, unresolvable lockfiles, binaries,
  delete/rename conflicts, oversized conflict sets, and genuine guesswork all stop with the tree
  exactly as it was and a rollback anchor printed.
  Use when: an MR shows conflicts with its target, "resolve the conflicts", "main moved and my
  branch won't merge", or as the merge gate's conflict stage under /ship-feature.
argument-hint: "[MR number or URL] — optional; defaults to the current branch's open MR"
---

# Resolve Conflicts

The target branch moved and the MR won't merge. Integrate the target into the feature branch,
resolve what conflicts, and **prove** the result — then let the caller merge.

`/ship-feature` calls this unconditionally from its merge gate, so **the no-conflict path must be
free**: probe, find nothing, exit without creating a commit.

> **Forge note.** Written for GitLab (`glab`). On GitHub: `gh pr view <n> --json
> mergeable,mergeStateStatus,baseRefName` for the conflict state, `gh pr view <n> --json url` for
> the link. The git work is identical.

---

## Strategy: merge the target in. Not rebase.

This matters enough to state the reasoning, so nobody "optimizes" it back:

1. **Review threads are anchored to commits.** GitLab positions diff notes against
   `(base_sha, start_sha, head_sha)`. A rebase rewrites every commit, so after the force-push
   GitLab re-maps what it can and marks the rest **outdated** — those threads fall out of the
   diff view. In this loop that means `/address-review`'s own replies detach from the code they
   cite, and `/ship-feature`'s convergence check ("every thread replied to") starts reading a
   mangled thread set.
2. **`/mr-review`'s delta mode depends on sha ancestry.** It reads the `Reviewed: <sha>` footer
   and diffs from there. After a rebase that sha is no longer an ancestor of head, so the round
   silently falls back to a full review — correct, but it doubles reviewer cost for the rest of
   the loop.
3. **A bad merge is undoable; a bad force-push is archaeology.** `git revert -m 1 <merge-sha>`
   versus reflog spelunking on a machine that may not be yours.

**Rebase only when both hold:** the MR has **zero** discussions (nothing to orphan) *and* the
project enforces linear history — `glab api "projects/:id" | jq -r .merge_method` returns `ff` or
`rebase_merge`. Then prefer the **server-side** `glab mr rebase <iid>`, which refuses on conflict
instead of leaving you in a rewrite, and only fall back to a local `git rebase origin/<target>`
plus `git push --force-with-lease` if that can't proceed. **Never force-push while a review round
is in flight.** Whichever strategy ran is named in the report.

---

## Process

### Step 0: Preconditions

1. Resolve the MR (`$ARGUMENTS`, else `glab mr list --source-branch "$(git branch --show-current)"`)
   and its `target_branch`.
2. `git status --porcelain` must be empty. Dirty tree → **stop**; another session may own this
   checkout, and stashing someone else's work is not yours to do.
3. **Record the rollback anchor** — `git rev-parse HEAD` — and print it. Every hard stop below
   reports it.
4. `git fetch origin <target> --quiet`.

### Step 1: Probe — calling this skill must be free

```bash
git merge --no-commit --no-ff origin/<target>
```

Clean? `git merge --abort`, report "no conflicts — target integrates cleanly", exit. **Do not
leave an empty merge commit behind**; a merge gate that calls this every round must not add a
commit every round.

Conflicts? Keep the merge in progress and continue.

### Step 2: Classify before you resolve

```bash
git diff --name-only --diff-filter=U      # both-modified
git ls-files -u                            # full unmerged index, with stages
git status --porcelain                     # DU / UD / AA / DD / AU / UA codes
```

**Only `UU` (both-modified) in ordinary source is eligible for automatic resolution.** Print the
classification table before touching a file — a reader of your report must be able to see what
you decided *before* seeing what you did.

### Step 3: Resolve, per class

| Class | Rule |
|---|---|
| **Lockfiles & generated** — `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `poetry.lock`, `Pipfile.lock`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, `composer.lock`, `*.pb.go`, `*_pb2.py`, generated OpenAPI/GraphQL schemas, snapshots, built assets | **Never hand-merge a hunk.** Take the target's version (`git checkout --theirs -- <file>` — during a merge `--ours` is your branch, `--theirs` is the incoming target), then **regenerate** with the project's own command (`npm install`, `poetry lock --no-update`, `make generate`, `buf generate`, the snapshot updater) and stage the regenerated artifact. If you cannot identify the regeneration command from the repo, or it fails → **hard stop.** A hand-merged lockfile is a supply-chain incident waiting to happen. |
| **Migrations** — numbered or timestamped schema files | Two *different* new migrations landing on both sides is a **renumber, not a merge**: keep both files, and if the project uses a numeric sequence and the numbers collide, rename ours to the next free number and grep for **every** reference to the old name. If **both sides edited the same migration file**, or the target's migration touches the same table or column as ours → **hard stop.** Wrong migration order is the one class of mistake that isn't recoverable in production. |
| **Additive lists** — import blocks, `__init__.py` exports, barrel files, `requirements.txt`, DI registries, route tables, i18n key files, `CHANGELOG.md` | **Union merge**: keep both sides' entries, dedupe, re-sort in the file's existing order (alphabetical, or grouped stdlib/third-party/local — match what the file already does). Mechanically safe because the entries are independent. Confirmed by the type checker afterwards, not by eye. |
| **Logic conflicts** — both sides changed the same lines | **Do not pick a side.** Read both versions *and* the target commit that introduced its change (`git log -1 --format='%s%n%b' <sha> -- <file>`, `git log -L <start>,<end>:<file>`) to recover both intents, then write the version that satisfies both. Record the reasoning per file. **The dangerous part is outside the hunk:** if the target renamed a symbol, changed a signature, or altered a contract your branch depends on, the conflict markers only show the tip — grep the whole branch for the old name before calling it resolved. |
| **Delete/modify, rename conflicts** (`DU`, `UD`, `AA`, `DD`, `AU`, `UA`) | **Hard stop.** Never silently resurrect a deleted file and never silently drop your change. If the target deleted a file your branch edits, the functionality usually moved — say where you think it went, and let a human confirm. |
| **Binary files** | **Hard stop**, unless exactly one side changed it and the other only moved it. |
| **Any resolution that deletes a test, an assertion, or a validation check present on either side** | **Hard stop.** That is how a conflict resolution quietly removes coverage. |

### Step 4: Verify the integration — all of it, in order

**a. No markers survive.**
```bash
git diff --check
grep -rnE '^(<<<<<<<|=======|>>>>>>>)' -- <the files you touched>
```

**b. The target-work check — the one that matters most.** The classic bad auto-resolution is
"took ours" wholesale, silently reverting somebody else's commit. Green tests will not catch it;
the reverted work simply disappears. For every file the target changed since the merge base,
confirm the target's change is still present in HEAD:

```bash
base=$(git merge-base HEAD origin/<target>)
git diff --name-only $base origin/<target> | while read -r f; do
  git diff $base origin/<target> -- "$f"   # what the target contributed
  git diff HEAD -- "$f"                     # what survived
done
```

Any target hunk missing from HEAD without a deliberate, recorded reason → **hard stop**.

**c. Semantic collisions git never flagged.** A clean `git status` is not evidence of a correct
integration. Two branches adding *different* files never conflict textually, and that is exactly
how a broken deploy gets built. Scan the merged result for:

- duplicate migration sequence numbers or migration ids across the combined set
- duplicate route paths, duplicate enum values, duplicate registry/DI keys introduced by the two
  sides
- a file your branch modified that the target deleted or renamed
- changes to CI config, deploy manifests, or dependency manifests on **either** side

Any hit → **hard stop**, named.

**d. Full gates, unpiped**, to a log file, greppable after: type checker, full test suite, and
the production build where the project has one.

**e. Feature re-verify.** Re-run the tests covering the plan's success criteria — or the REQs, on
a ticket-driven run. A green suite proves nothing broke; it does not prove the feature still
works.

**f. Only then commit and push.** `git commit --no-edit` (or
`chore: merge origin/<target> into <branch>`) and a plain `git push`. **Never `--force` on the
merge path.**

### Step 5: Report

```
## Conflict Resolution — {branch} ← origin/{target}

Strategy: merge ({n} discussions preserved) | rebase (no discussions, linear-history project)
Anchor before resolution: {sha}
Rollback: git revert -m 1 {merge-sha}   [or git reset --hard {sha} if not yet pushed]

Conflicts: {n} files
  logic     {file} — {both intents, and the version written}
  generated {file} — took target, regenerated with `{cmd}`
  additive  {file} — union of {a}+{b} entries, deduped, re-sorted
  migration {file} — renumbered 047 → 049; 3 references updated

Verification: markers ✅ · target-work check ✅ ({k} target hunks confirmed present) ·
  semantic scan ✅ · type-check ✅ · tests ✅ ({n}) · build ✅ · feature re-verify ✅
```

---

## Hard stops

Every hard stop **leaves the tree exactly as it was** — `git merge --abort` — and reports the
anchor sha plus what a human needs to decide:

- Any class marked hard-stop above: unknown or failing lockfile regeneration, both sides editing
  one migration, delete/rename conflicts, binaries, a resolution that would drop a test.
- Gates still red after resolution and not fixable within this skill's scope.
- **Size guard:** more than 20 conflicted files, or roughly 400 conflicted lines. That is a
  "rebase the feature" conversation, not an auto-resolve.
- **Moving target:** `git fetch` shows the target advanced again mid-resolution → re-run once;
  twice in a row → stop, the target needs a quiet window.
- **Genuine guesswork:** two plausible readings of intent with no evidence to choose between
  them. Name both readings, then stop. Guessing here produces code that compiles, passes, and is
  wrong.

## Rules

- **Merge in, don't rebase** — unless the MR has no discussions *and* the project requires linear
  history. Review threads and delta reviews both depend on the existing shas.
- **Never force-push mid-review.** Rebasing is a stage-boundary action, never something that
  happens while a review round is in flight. After any force-push: record the pre-rebase sha and
  post it as an MR comment, clear the recorded `reviewed_sha` so the next round is a full review,
  and re-run the full gate before believing anything.
- **Classify before resolving.** Not every conflict is a text merge. The file's *kind* decides the
  method, and three kinds have no safe automatic method at all.
- **Never pick a side to make it compile.** `--ours` / `--theirs` wholesale on a logic conflict is
  the failure this skill exists to prevent, and the target-work check is what catches it.
- **A clean `git status` is not a correct integration.** The conflicts git flags are the easy
  ones; run the semantic scan.
- **Stops are cheap, wrong merges are not.** Aborting costs a human ten minutes. A silently
  reverted commit or a mis-ordered migration costs a production incident and nobody knows where
  it came from.
- **Report the rollback.** Every run ends with the exact command that undoes it.
