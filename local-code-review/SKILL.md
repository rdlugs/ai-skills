---
name: local-code-review
description: Review uncommitted local code changes (working directory + staged) in a git repo, hunting for bugs, security issues, missing test coverage, and blast radius — everything downstream the change could break — then report findings grouped by severity and lay out an ordered plan to fix them. Use this skill whenever the user asks to review their changes, check their diff, look over what they wrote before committing, sanity-check a patch, trace what a change might affect, or says anything like "review my code", "can you look at this before I push", "did I break anything", "what does this touch", or "check my changes" — even if they don't say the words "code review". Also use it when the user is about to commit, open a PR, or asks whether their work-in-progress looks correct.
---

# Local Code Review

Review the user's uncommitted work the way a careful teammate would: read the actual diff, understand the surrounding code before judging it, and report only what matters.

## Scope

Default to **uncommitted changes**: both the working directory and the staging area. That is what the user is about to commit, so that is what deserves scrutiny.

If the user names a different scope (a branch comparison, the last N commits, a specific file), follow their lead instead.

## Workflow

### 1. Establish the diff

```bash
git status --short              # what's touched, including untracked
git diff                        # unstaged changes
git diff --staged               # staged changes
```

Untracked files never show up in `git diff`, and new files are frequently the ones with the real problems. Read them in full with `cat`.

If there are no changes, say so plainly rather than inventing something to review.

### 2. Read for context, not just for the diff

A diff hunk shows what changed, not whether it's correct. Before flagging anything, open the surrounding function or file. Most false positives come from reviewing a hunk in isolation — a "missing null check" is often handled by the caller three lines above the hunk boundary.

Where a change touches an interface, grep for its callers. A renamed parameter or a changed return type is only safe if every call site agrees.

### 3. Map the blast radius

The diff tells you what changed. The blast radius tells you what *else* now behaves differently — and it's where the expensive bugs live, because nothing in the diff looks wrong. The user changed one function; the breakage is in a caller they forgot about, or a consumer in another service, or a cached value that is now stale.

Work outward from each changed symbol:

```bash
git diff --name-only && git diff --staged --name-only   # changed files
grep -rn "symbol_name" --include=*.<ext> .              # direct references
```

For every function, class, constant, or config key the diff touches, find its references and decide whether each one still holds. Follow the chain: if a caller's own behavior changed as a result, its callers are in the radius too. Stop when the effect is contained — when a change is genuinely local, say so, because that's information the user wants.

Watch specifically for these, since they escape a grep of the source tree:

- **Contract changes.** Function signatures, return types and their nullability, exception types raised, HTTP response shapes, event payloads. Anyone who depended on the old shape is now broken, including callers you can't see — other repos, mobile clients, third-party consumers.
- **Data layer.** Schema migrations, column renames, index removals. Does the migration run against production-sized data without locking? Is there a window where old code reads a column new code stopped writing?
- **Behavior under absence.** Removed validation, removed authorization checks, changed defaults. A changed default silently rewrites the behavior of every caller who omitted the argument.
- **Shared and global state.** Module-level mutable values, singletons, caches, connection pools. A cached value keyed on data whose shape just changed will serve stale or wrong entries.
- **Config and environment.** New environment variables or feature flags that exist locally but not in deployed environments; changed timeouts, retry counts, or resource limits.
- **Concurrency and ordering.** Work moved into or out of a lock, transaction boundary, or async context. Code that was serialized may no longer be.

Report blast radius findings in the same severity groups as everything else. An unhandled downstream caller is a bug — it just happens to live outside the diff. When a change ripples widely, give the user the map: which call sites you checked, which ones are fine, which ones aren't.

### 4. Hunt in three passes

**Bugs & correctness.** Off-by-one errors, inverted conditions, unhandled error paths, resource leaks (unclosed files, connections, subscriptions), mutation of shared state, race conditions in concurrent code, silently swallowed exceptions, changed behavior the rest of the code doesn't expect.

**Security.** User-controlled input reaching a sink without validation: SQL string concatenation, shell commands built from strings, path traversal in file operations, deserialization of untrusted data, rendering unescaped input. Also: secrets, tokens, or keys committed as literals; authentication or authorization checks removed or weakened; logging that captures sensitive data.

**Test coverage.** Does new behavior have a test? Does changed behavior have a test that would have caught the change if it were wrong? A test that passes both before and after a bug fix is not a test of that bug fix. Point to the specific untested branch, not "add more tests."

### 5. Filter before reporting

Ask of each finding: if the user shipped this, what actually goes wrong? If the answer is "nothing, it's just not how I'd write it," drop it. Style preferences dressed up as findings make a review harder to act on and train the user to skim.

Keep a finding when it identifies a concrete failure mode, and say what that failure mode is.

## Output

Report inline in chat, grouped by severity. Use exactly these three groups, and omit any group that is empty:

```
## Critical
Will break or expose something. Fix before committing.

## Worth fixing
Real problems, but not blockers.

## Consider
Judgment calls the user may reasonably decline.
```

Each finding takes this shape:

**`path/to/file.py:42`** — one-line statement of the problem.
Then a sentence or two on *why* it fails: the input that triggers it, the state that breaks, the check that's missing. Show the offending line if it clarifies. Suggest the fix concretely — a corrected line beats a paragraph describing one.

Open with a single sentence orienting the user: how many files changed, what the change appears to do, and how far it reaches — "contained to this module" or "touches 6 call sites across 3 packages, plus the `/v2/users` response shape." Close with an honest verdict — "this looks ready to commit" is a valid and useful review outcome, and saying it when true is what makes the critical findings credible when they appear.

## Fix plan

Whenever there is at least one finding in **Critical** or **Worth fixing**, close the review with a plan. A list of problems leaves the user to work out the order themselves, and order matters — fixing a caller before fixing the signature it calls means touching the same file twice.

Skip the plan entirely when the only findings are **Consider** items, or when there are no findings at all. A plan for optional suggestions is noise.

Structure it as ordered steps, not a restatement of the findings:

```
## Fix plan
1. [file:line] — what to change, in one line.
2. ...
```

Sequence by dependency first, severity second. Contract changes come before their call sites. Schema migrations come before the code that reads the new shape. A fix that makes another finding disappear should come before you spend a step on the finding it dissolves — and say so: "this also resolves #4."

Group steps that touch the same file into one step. Note which steps are independent, so the user knows what can be parallelized or split across commits. If a fix needs a decision only the user can make — restore the old default, or update every caller? — state the fork rather than silently choosing.

Then say what verifies the fix: the test to add, the existing test to run, the manual check. A plan that ends without a way to confirm it worked isn't finished. If a finding was a missing test, the fix *is* the test — name the branch it should cover.

End by offering to apply it: "Want me to work through this?" Don't apply anything unasked. If the user takes you up on it, follow the plan in order and re-verify the diff afterward rather than assuming the edits landed cleanly.

## Example finding

**`api/users.py:87`** — SQL injection via the `sort` query parameter.

The parameter is interpolated straight into the `ORDER BY` clause, so `?sort=id;DROP TABLE users--` executes. Parameterized queries don't cover column names, so allowlist instead:

```python
SORT_COLUMNS = {"id", "name", "created_at"}
if sort not in SORT_COLUMNS:
    raise ValueError(f"invalid sort column: {sort}")
```

**`core/pricing.py:12`** — default changed from `include_tax=True` to `False`; three callers omit the argument.

`checkout.py:203`, `invoice.py:88`, and `reports/monthly.py:41` all call `total()` without passing `include_tax`, so each now returns pre-tax amounts. `checkout.py` is the one that matters — it charges the customer. Either restore the old default and make the new behavior opt-in, or update all three call sites explicitly.

## Notes

Review the change, not the file. Pre-existing problems in untouched lines are out of scope unless the change makes them newly reachable — in which case they're now the change's problem and belong in the review.