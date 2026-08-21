---
name: mr-fix
description: >
  Fetch unresolved code review comments from a GitLab MR or GitHub PR, plan the fixes,
  implement them after approval, verify, self-review the diff in a bounded fix loop, and commit. Use this skill whenever the user
  mentions fixing, addressing, resolving, or implementing MR comments, PR comments,
  review findings, review feedback, reviewer suggestions, or code review results —
  and whenever they type /mr-fix, /mr-fix 298, or /mr-fix MR 298. Also use it when the
  user pastes review comments and asks to act on them, or says things like "address the
  review on 298", "fix what the reviewer flagged", or "handle the MR feedback".
---

# mr-fix

Turns code review feedback into reviewed, formatted, committed changes. Language- and environment-agnostic: it detects the forge, the container setup, and the toolchain before it touches anything.

## Step 0 — Preflight (detect everything; assume nothing)

Run this block first. It answers every environmental question the rest of the skill depends on. Nothing below is hardcoded — if a check comes back empty, the skill adapts rather than failing.

```bash
# --- Forge: which platform, which CLI, is it authed? ---
git remote get-url origin
git branch --show-current
git status --porcelain

command -v glab && glab auth status 2>&1 | tail -2
command -v gh   && gh   auth status 2>&1 | tail -2

# --- Runner: is this project containerized? ---
ls docker-compose.yml docker-compose.yaml compose.yml compose.yaml 2>/dev/null
docker compose ps --services --status running 2>/dev/null

# --- Toolchain: what actually exists in this repo? ---
ls composer.json package.json vendor/bin/pint vendor/bin/php-cs-fixer 2>/dev/null
ls pyproject.toml uv.lock poetry.lock Pipfile requirements.txt setup.py tox.ini 2>/dev/null
ls -d .venv venv 2>/dev/null
command -v uv poetry pipenv hatch pdm rye conda 2>/dev/null
command -v oco
```

If `pyproject.toml` exists, read it — `[tool.ruff]`, `[tool.black]`, `[tool.mypy]`, `[tool.pytest.ini_options]` tell you what the project actually uses, which beats guessing from what happens to be installed globally.

Derive four facts, then reuse them everywhere:

**1. Platform + CLI.** Remote host contains `gitlab` → GitLab/`glab`; contains `github` → GitHub/`gh`. Self-hosted GitLab often has a custom domain — if the host matches neither, check `glab api /version` against the remote host before asking.

If the matching CLI is missing or unauthenticated, say exactly which (`glab not found` vs `glab found but not logged in`) and give the one command that fixes it (`glab auth login`). Do not attempt to scrape the MR from the web.

**2. Command runner.** Determine the prefix for *every* command this skill runs. Call it `$RUN`. It composes in two layers — container first, then language environment.

*Layer 1 — container:*

- A compose file exists **and** services are running → find the app service (the one with `composer.json`, `artisan`, or `pyproject.toml` mounted; commonly `app`, `php`, `web`, `api`, or `laravel.test` under Sail — confirm from `docker compose ps --services`, don't guess). `$RUN = docker compose exec <service>`
- A compose file exists but **nothing is running** → ask whether to `docker compose up -d` or run on host. Don't decide this for the user.
- No compose file → this layer is empty.

*Layer 2 — Python environment* (only if the repo is Python). Pick the **first** that applies; the lockfile is stronger evidence than what's on `PATH`:

| Evidence | Prefix |
|---|---|
| `uv.lock`, or `[tool.uv]` in pyproject | `uv run` |
| `poetry.lock` | `poetry run` |
| `Pipfile.lock` | `pipenv run` |
| `pdm.lock` / `[tool.pdm]` | `pdm run` |
| `rye` in pyproject / `requirements.lock` | `rye run` |
| `.venv/` or `venv/` present, no lockfile | `.venv/bin/` prefix on the binary directly |
| Bare `requirements.txt`, no venv | none — but **warn** before installing anything globally |

Inside a container, the venv is usually already active, so Layer 2 is often unnecessary — check with `$RUN which python` before stacking `uv run` on top of `docker compose exec`. Stacking both when the container already activates the venv is harmless with `uv`, but breaks with `poetry` in some images.

**Once `$RUN` is fixed, never bypass it.** Every later command in this skill is written as `$RUN <command>` and must be expanded with the detected prefix.

**3. Toolchain.** Resolve per ecosystem; skip and *say so* when a tool is absent.

| | PHP | Python | JS/TS |
|---|---|---|---|
| Format | `pint`, else `php-cs-fixer fix` | `ruff format`, else `black` | `prettier` |
| Lint | — | `ruff check --fix`, else `flake8` (no autofix) | `eslint --fix` |
| Types | — | `mypy`, else `pyright` | `tsc --noEmit` |
| Test | `php artisan test`, else `phpunit`/`pest` | `pytest`, else `python -m unittest` | `vitest`/`jest` |

Committer: `oco` if on `PATH`, else write the commit message directly.

For Python, prefer the project's configured entrypoints over your memory of them: `[tool.pytest.ini_options] addopts` may already set flags, and a `Makefile` or `tox.ini` target (`make lint`, `make test`) is usually the intended interface. Use it if it exists.

**4. Working tree.** If `git status --porcelain` is non-empty, show it and ask before touching anything — uncommitted work plus automated edits is how changes get lost. If the current branch isn't the MR's source branch (known after Step 1), warn and offer to check it out.

## Step 1 — Resolve the MR

Parse the MR/PR number from the invocation: `/mr-fix 298`, `/mr-fix MR ID 298`, `!298`, `#298`, a full URL, or a bare number.

If no number was given, list open MRs authored by the current user on the current branch and offer them:

```bash
glab mr list --source-branch "$(git branch --show-current)"   # or: gh pr list --head "$(git branch --show-current)"
```

A single match → use it. Multiple or zero → ask.

## Step 2 — Fetch unresolved comments only

This is the step that matters most. Raw note lists include resolved threads, system notes ("added 1 commit"), and the user's own replies. **Filter all of those out.** Acting on a resolved thread is worse than missing one.

**GitLab** — use the API directly, since `glab mr note list` does not expose `resolved`:

```bash
glab api "projects/:id/merge_requests/<IID>/discussions?per_page=100" \
  --paginate \
  | jq '[.[] | select(.notes[0].resolvable == true and .notes[0].resolved == false)
        | {file: .notes[0].position.new_path,
           line: .notes[0].position.new_line,
           author: .notes[0].author.username,
           body: .notes[0].body,
           discussion_id: .id}]'
```

**GitHub:**

```bash
gh pr view <NUMBER> --json reviews,comments
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/comments" --paginate \
  | jq '[.[] | select(.in_reply_to_id == null) | {file: .path, line: .line, author: .user.login, body: .body}]'
```

If zero unresolved comments: say so plainly and stop. Do not invent work.

## Step 3 — Classify each finding

Before planning, sort every comment into one of:

| Class | Meaning | Action |
|---|---|---|
| **Actionable** | A concrete change is requested | Include in plan |
| **Question** | Reviewer is asking, not directing | Draft a reply, do not change code |
| **Nitpick** | Style/preference, no correctness impact | Include, but mark as low priority |
| **Out of scope** | Valid, but belongs in another MR | Flag; propose a follow-up ticket |
| **Conflicting** | Contradicts another comment | **Stop. Ask the user to arbitrate.** |

Never silently resolve a conflict by picking one side.

## Step 4 — Present the plan, then wait

Group by file. For each finding show: line, reviewer, what they asked, what you'll change, and why.

```
app/Services/BookingService.php
  L42  @jdoe  "This N+1s on large result sets"
       → Eager-load `customer` in the base query.
  L88  @jdoe  "Magic number"
       → Extract 30 into a MAX_RETRIES class constant.  [nitpick]

resources/js/stores/useBookingStore.ts
  L15  @amara "Mutating state outside an action"
       → Move the assignment into `setFilters()`.
```

Then ask: **"Implement this plan?"** Stop and wait. Do not proceed on ambiguity or silence.

## Step 5 — Implement

- One finding at a time, smallest diff that satisfies the comment
- Do not opportunistically refactor adjacent code
- If a fix turns out to require a change the reviewer did not ask for, surface it before writing it

## Step 6 — Verify

Expand `$RUN` and the tool names from Step 0. The same skill runs on the host, in a container, under `uv run`, or under both — nothing below is language-specific until you substitute.

```bash
$RUN <format>     # pint --dirty   |   ruff format .   |   black .
$RUN <lint>       #                    ruff check --fix .
$RUN <types>      #                    mypy <changed-paths>
$RUN <test>       # php artisan test --filter=X   |   pytest <path>::<test> -q
```

Concretely, a `uv` project inside Docker becomes:

```bash
docker compose exec api uv run ruff check --fix .
docker compose exec api uv run pytest tests/test_bookings.py -q
```

...and a bare `uv` project on the host is just `uv run pytest -q`.

Run the narrowest test selection that covers the changed code, not the whole suite — then the full suite once, at the end, before committing.

Skip any check whose tool Step 0 didn't find — and say which you skipped. A silently skipped test suite is worse than no test suite. If `ruff format` and `ruff check --fix` are both available, run **format after lint**: lint autofixes can leave formatting artifacts.

If tests fail, fix and re-run. Do not commit red.

## Step 7 — Self-review the diff (loop until clean)

Green tests do not mean the change is right. Before committing, review your own diff as if it were someone else's MR.

```bash
git diff
```

**Scope the review deliberately.** You are checking three things, in this order:

1. **Did each original finding actually get addressed?** Walk the Step 4 plan item by item and point at the hunk that satisfies it. A finding with no corresponding hunk is unaddressed, regardless of what the tests say.
2. **Did the fix introduce a new problem?** N+1s, unhandled nulls, swallowed exceptions, changed public signatures, altered behavior outside the reviewed lines, debug artifacts (`dd()`, `dump()`, `console.log`, `print()`, `breakpoint()`), commented-out code, secrets.
3. **Is the diff minimal?** Anything in the diff that no finding asked for is scope creep. Flag it.

Do **not** re-review the whole file, the whole codebase, or design decisions that predate this MR. That's how a two-comment fix becomes a refactor.

### The loop

Classify what the self-review turns up:

| Severity | Meaning | Action |
|---|---|---|
| **Blocker** | Wrong, unsafe, or a finding is unaddressed | Fix it. Loop. |
| **Should-fix** | Correct but sloppy — debug code, dead branch, bad name | Fix it. Loop. |
| **Consider** | Judgment call, arguably fine either way | Report, do not act |

If there are any Blocker or Should-fix findings:

1. State them plainly, grouped by file, with line references.
2. **Return to Step 5**, fixing only those findings — do not re-plan, do not revisit the reviewer's comments.
3. Re-run Step 6 (verify).
4. Re-run this step.

Iterate until the self-review yields nothing above **Consider**.

### Bound the loop

**Maximum three passes.** If pass 3 still surfaces Blockers, stop. Do not attempt a fourth.

Report to the user: what's still broken, what you tried on each pass, and why you think it isn't converging. Three failed passes usually means one of:

- The reviewer's comment is ambiguous and you're solving the wrong problem
- Two findings genuinely conflict and Step 3 missed it
- The fix requires a design change outside the MR's scope

All three need a human, not another pass. Say which one you suspect.

Also stop early — before three passes — if you notice a pass **undoing** something a previous pass did. That's oscillation, not convergence, and one more iteration will not help.

### Announce the outcome

Before moving to commit, state the result explicitly:

```
Self-review: pass 2 — clean.
  Addressed:  3/3 findings
  Fixed on pass 2:  leftover dd() in BookingService.php:51
  Consider (not acted on):  setFilters() could take a typed DTO — out of scope for this MR
```

Only then proceed.

## Step 8 — Commit

One commit per logical group, not one commit for the whole MR.

```bash
git add <files-for-this-group>
oco --yes
```

Use `oco` only if Step 0 found it. If it's absent, or it fails (`EMPTY_MESSAGE`, 405, provider unreachable), **do not retry more than once.** Write the conventional commit directly:

```bash
git commit -m "fix(booking): eager-load customer to avoid N+1"
```

Tell the user `oco` was bypassed and why.

## Step 9 — Close the loop

Print a summary: what was fixed, what was skipped and why, any drafted replies to questions.

Then offer — do not perform automatically:
- Push the branch
- Reply to each discussion thread with what changed
- Resolve the threads (`glab api --method PUT ".../discussions/<id>?resolved=true"`)

## Error handling

| Situation | Response |
|---|---|
| Two comments contradict | Stop, present both, ask user |
| Comment references code no longer in the diff | Flag as stale, skip, mention it |
| Working tree dirty at start | Show `git status`, ask before touching anything |
| Not on the MR's source branch | Warn, offer to check it out |
| Can't detect platform/repo | Ask once, then proceed |
| Forge CLI missing vs. unauthenticated | Distinguish the two; give the exact fix command |
| Compose file exists, no containers up | Ask: start them, or run on host? Never decide silently |
| Multiple candidate app services | List them, ask which; don't assume `app` |
| Formatter/test runner not found | Skip that check and name it in the summary |
| Both `uv.lock` and `poetry.lock` present | Ask which is authoritative; do not pick |
| Python project, no venv and no lockfile | Warn before any install; never `pip install` globally without consent |
| Lockfile changed as a side effect of a fix | Call it out — it belongs in the commit, but the user should know |
| Self-review still finds blockers after 3 passes | Stop. Report state, diagnose why, hand to user. Never commit |
| A pass reverts a previous pass's change | Oscillation — stop immediately, don't wait for pass 3 |
| Self-review wants to fix something no finding raised | It's `Consider`. Report it, leave the code alone |
| Reviewer asks for a breaking change | Do not implement silently — confirm scope first |