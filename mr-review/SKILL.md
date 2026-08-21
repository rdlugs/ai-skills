---
name: mr-review
description: >
  Use this skill whenever the user wants to perform a code review on a GitLab Merge Request (MR).
  Triggers include: any mention of "code review", "review MR", "review merge request", "review MR ID",
  "check MR", "review this MR", or when the user provides an MR ID (e.g. "MR 42", "!42") with or
  without an associated issue ID. This skill handles the full review lifecycle: reading prior review
  comments, fetching related issue context, performing a structured code review with impact analysis,
  and posting the result as a GitLab MR comment. Always use this skill when a GitLab MR review is
  involved — even if the user says "just a quick review" or only provides an MR ID.
---

# GitLab Code Review Skill

Reviews a GitLab MR using the `glab` CLI: fetch the diff and prior comments, optionally pull issue
context, analyze, then post findings as an MR note.

**Requirements**
- `bash` tool access
- Either the `glab` CLI (authenticated against the company's self-hosted GitLab), or `curl` + `jq` —
  see the **API fallback** appendix at the end of this file

**Command compatibility.** This skill sticks to long-stable `glab` commands. Avoid `glab mr note list`
and `glab mr note create` — the first is marked experimental and may be removed; both are absent from
older glab versions still widely installed. Use `glab mr view --comments` and `glab mr note -m` instead.

**Never type `<MR_ID>` into a shell.** Bash reads `<` as input redirection, so `glab mr view <MR_ID>`
tries to open a file named `MR_ID`, fails with a misleading error, and never runs glab at all. Bind the
inputs to variables once (Step 0) and use `"$MR_ID"` everywhere after.

---

## Step 0 — Verify tooling, auth & collect inputs

Two distinct failures live here — don't conflate them:

```bash
command -v glab || echo "NO_GLAB"
```

**If glab is missing**, don't stop and don't improvise — skip to the **API fallback** appendix at the end
of this file and carry on with the same review steps.
Everything `glab` does here is a thin wrapper over the GitLab REST API, so nothing about the review
itself changes.

**If glab exists**, check auth separately:

```bash
glab auth status
```

If auth fails, **do not ask the user to paste a token into the chat.** A token in the transcript is a
leaked token — it persists in conversation history and shell history, which defeats the whole point of
treating it as a secret. Instead, ask them to run this in their own terminal and report back:

```bash
glab auth login --hostname gitlab.mycompany.com
```

Ask for the hostname if it isn't already configured. Never assume or fall back to `gitlab.com` — this
org uses a self-hosted instance only.

### Bind the inputs

- **MR ID** (required) — e.g. `42` or `!42`. Strip the `!`.
- **Project path** (optional) — infer it first:
  ```bash
  git remote get-url origin 2>/dev/null
  ```
  If cwd is a GitLab repo, glab resolves the project automatically and `-R` can be omitted. Only ask the
  user for `group/my-repo` if inference fails or they're reviewing a repo they aren't sitting in.
- **Issue ID** (optional) — e.g. `15` or `#15`.

Set them once, with the real values substituted:

```bash
MR_ID=42
REPO=group/my-repo        # leave empty to let glab infer from cwd
ISSUE_ID=15               # optional
```

Every later command uses `${REPO:+-R "$REPO"}`, which expands to nothing when `REPO` is empty — so the
same command works inside and outside the repo.

---

## Step 1 — Fetch MR metadata, prior comments, and diff

```bash
glab mr view "$MR_ID" ${REPO:+-R "$REPO"} --comments --unresolved
glab mr diff "$MR_ID" ${REPO:+-R "$REPO"} --raw
```

`--unresolved` implies `--comments`, so one call gets metadata plus the discussions that still matter.
Add a second call with `--comments` alone if you need the resolved threads too.

**Always pass `--raw` to `mr diff`.** Without it, glab may emit ANSI color escapes into the diff, which
corrupt the hunk headers you're about to count lines from in Step 4. `--color=never` works too; `--raw`
is the flag intended for piping.

**Act on MR state — don't just read it.** Before reviewing, check and surface anything that changes the
review's meaning:

| State | What to do |
|---|---|
| Draft / WIP | Note it up front; hold suggestions lighter, the author isn't done |
| Merged or closed | Tell the user and ask whether to proceed before spending effort |
| Merge conflicts | Flag prominently — the diff may not reflect what will land |

**Prior comments.** Note resolved vs unresolved threads. Don't re-raise resolved items. Do flag
unresolved items that recur in this diff — a repeated finding is a stronger signal than a fresh one. If
there are no prior comments, continue silently.

Use branch/state/file-list as context; don't dump the raw output into chat. If the user asks about it
directly, of course answer.

---

## Step 2 — Handle large diffs before analyzing

A large MR will exceed your context and you'll silently review only the fraction that fit — which is
worse than admitting the limit, because the user thinks they got a full review.

```bash
glab mr diff "$MR_ID" ${REPO:+-R "$REPO"} --raw | wc -l
```

If the diff is large (roughly >1500 lines), don't try to swallow it whole:

1. List changed files: `glab mr diff "$MR_ID" ${REPO:+-R "$REPO"} --raw | grep '^+++'`
2. Prioritize by blast radius (see Step 4): auth/permissions → DB migrations → config & secrets →
   API contracts → business logic → tests → docs/formatting. The 🔴 categories get reviewed even if
   nothing else does.
3. Review file by file, highest priority first
4. **Tell the user which files you reviewed and which you skipped.** Put this in the posted note too.

---

## Step 3 — Fetch related issue (if Issue ID provided)

```bash
glab issue view "$ISSUE_ID" ${REPO:+-R "$REPO"}
```

Use the description and acceptance criteria to check whether the MR actually delivers what was asked.
Context only — don't summarize the issue back to the user. Skip silently if no Issue ID was given.

---

## Step 4 — Perform the review

Analyze the diff, MR description, issue context, and prior comment history across: logic correctness,
security, performance, error handling, test coverage, and impact on the target branch.

### Assess blast radius

Before writing findings, work out what this change can break beyond the lines it touches. A clean diff
with a wide blast radius is more dangerous than a messy one confined to a single leaf module, and the
author — sitting inside their own change — is the person least likely to see it.

Ask, in order:

1. **Who calls this?** For every changed function, exported symbol, or endpoint, find the callers. If the
   repo is checked out locally, actually look rather than guessing:
   ```bash
   git grep -n '<symbol_name>' -- ':!*_test.*' ':!*spec*'
   ```
   Callers outside the changed files are the blast radius. Zero callers found for a modified public
   symbol is itself a finding — either it's dead code or it's consumed by another repo.

2. **Does it cross a contract boundary?** API request/response shapes, event payloads, queue message
   formats, and shared library signatures have consumers you cannot see in this diff. Renaming a JSON
   field is a one-line diff and a production outage.

3. **Is the data change reversible?** Migrations that drop columns, backfill in place, rewrite rows, or
   add non-concurrent indexes on large tables cannot be undone by reverting the MR. Rollback safety is a
   separate question from correctness, and it is the one people forget.

4. **What's the config and secrets surface?** New env vars, changed defaults, feature flags, and IAM or
   permission changes affect every environment, not just the one the author tested in. A new required
   env var with no default breaks deploy the moment it merges.

5. **Is it in a hot path?** Changes inside request handlers, loops over unbounded collections, N+1 query
   shapes, or anything holding a lock scale with traffic. A 20ms regression is invisible in review and
   obvious at peak.

Grade the result:

| Radius | Meaning |
|---|---|
| 🟢 **Contained** | Effects stop at the changed files. Internal logic, tests, docs, formatting. |
| 🟡 **Module-wide** | In-repo callers affected. Shared helpers, internal interfaces, config defaults. |
| 🔴 **Cross-boundary** | Reaches beyond this repo or beyond this deploy — API/schema contracts, migrations, secrets, IAM, anything irreversible. |

Fold what you find into the findings themselves rather than reporting it abstractly. "This drops
`users.legacy_id`" is an observation; "This drops `users.legacy_id`, which `billing-service` still reads —
and a revert won't bring the column back" is a review. Include the grade in the posted note whenever it's
🟡 or 🔴; omit the line entirely when it's 🟢, since a contained change needs no announcement.

### Getting line numbers right

Findings are cited as `file.ext:LINE`. A wrong line number costs more trust than a missed bug does — it
tells the author the reviewer never actually looked. Unified diff output does not hand you line numbers,
so derive them carefully:

- Each hunk header `@@ -a,b +c,d @@` means the **new** file's hunk starts at line `c`
- Count forward from `c`, incrementing on context lines (` `) and added lines (`+`), **not** on removed
  lines (`-`)
- Only cite lines that appear in the diff. You cannot see the rest of the file.
- If you can't establish a line number confidently, cite the file and quote the offending line instead of
  guessing: `` config/app.rb — `secret_key = ENV.fetch(...)` ``

`glab` has no support for line-anchored inline comments, so all findings go into a single note. Don't
reach for a `--file` / `--line` flag; it doesn't exist.

### What to write

**Only surface actual findings. Omit any section that has nothing to report** — no "Issues: none" headers,
no summary paragraph, no filler.

**Blast radius** — a single line, only when 🟡 or 🔴. Name the radius and what's downstream:
`🔴 **Cross-boundary** — drops \`users.legacy_id\`, still read by \`billing-service\`; revert does not restore it.`

**Issues (blocking)** — bugs, security holes, broken logic, missing auth checks, unsafe migrations. Each as:
`- \`file.ext:LINE\` — what the problem is and why it matters`

A 🔴 radius doesn't automatically make something blocking — plenty of intentional contract changes are
correct. It raises the bar for what counts as adequately handled: an unreversible migration with no
rollback plan, or a renamed field with no note about its consumers, is a blocking issue precisely because
of the radius.

**Suggestions (non-blocking)** — code quality, naming, DRY, missing tests, minor performance. Each as:
`- \`file.ext:LINE\` — what to improve and how`

**Verdict** — exactly one:
- ✅ **Approved** — no issues found
- ⚠️ **Approved with suggestions** — no blockers, suggestions noted
- 🔁 **Changes requested** — blocking issues must be resolved before merge

---

## Step 5 — Preview & post

### Preview

Show the review in chat and ask:

> "Ready to post this to MR !{MR_ID}?"

Skip confirmation only if the user explicitly said "just post it." ("Just a quick review" is not that.)

### Compose to a file, not to the shell

Review bodies contain backticks, quotes, and `$` from code snippets. Interpolating that into a shell
command is command injection by accident — it will mangle the note at best and execute something at worst.
Always write to a file first:

```bash
REVIEW_FILE="/tmp/mr-review-${MR_ID}.md"
cat > "$REVIEW_FILE" <<'REVIEW_EOF'
<!-- claude-review -->
## Code Review — !42

🔴 **Cross-boundary** — drops `users.legacy_id`, still read by `billing-service`; revert won't restore it.

### Issues
- `path/to/file.rb:42` — ...

### Suggestions
- `path/to/other.rb:17` — ...

---
🔁 **Changes requested**

> *Reviewed on 2026-07-10*
REVIEW_EOF
```

Write the finished text into the heredoc with real values already substituted — the quoted
`<<'REVIEW_EOF'` prevents the shell expanding anything inside, which is what makes it safe for code
snippets, but it also means `$MR_ID` and `$(date)` will **not** interpolate. Get the date first:

```bash
date -u +%Y-%m-%d
```

### Re-reviews

Check the fetched comments for the `<!-- claude-review -->` marker. If a prior review exists, the note
must open with a line naming the pass and what moved, so the author isn't reading a wall of text hunting
for what's new:

```
## Code Review — !42 (2nd pass)
_Previous blockers: 2 resolved, 1 outstanding._
```

### Post

```bash
glab mr note "$MR_ID" ${REPO:+-R "$REPO"} -m "$(cat "$REVIEW_FILE")"
```

On success, give the user the direct link (substitute real values — this is for the human, not the shell):

```
https://gitlab.mycompany.com/group/my-repo/-/merge_requests/42#notes
```

If the verdict was ✅ **Approved**, offer (don't run automatically):

```bash
glab mr approve "$MR_ID" ${REPO:+-R "$REPO"}
```

---

## Error handling

| Situation | Action |
|---|---|
| `glab` not installed | Use the API fallback (appendix). Mention that installing `glab` is nicer long-term, but don't block the review on it. |
| `glab` missing **and** `GITLAB_TOKEN` unset | Ask the user to export the token in their own shell, or fall back to offline review (appendix). Never take a token in chat. |
| `glab auth status` fails | Ask user to run `glab auth login --hostname <host>` in their own terminal. Never take a token in chat. |
| 404 on MR or Issue | Confirm project path and ID. Check they're on the right host — a valid ID on the wrong instance 404s identically. |
| SSL / cert error | Self-hosted instances usually need the org CA bundle: `glab config set ca_cert /path/to/ca.pem`. Only if that's impossible, mention `glab config set skip_tls_verify true` — and say plainly that it disables certificate verification globally and shouldn't be left on. |
| Empty diff | Review description and issue only; state that no file changes were found. |
| Diff too large for context | Follow Step 2. Never review a truncated diff silently. |
| Unrecognized `glab` subcommand | The installed glab is older than the docs. Fall back to `glab mr view --comments` and `glab mr note -m`. |
| `cannot open MR_ID: No such file` | You typed a `<PLACEHOLDER>` into bash and it was read as redirection. Substitute the real value or use `"$MR_ID"`. |
| Diff full of `ESC[` escape codes | Missing `--raw` on `glab mr diff`. Re-fetch; do not attempt line numbers from colored output. |
| No prior comments | Skip silently. |
| No Issue ID given | Skip silently. |

---

## Notes

- **Always the company's self-hosted GitLab** — never substitute `gitlab.com`
- Never post a partial or errored review to the MR
- Never accept, echo, or persist a GitLab token
- Clean up: `rm -f "$REVIEW_FILE"` after posting

---

## Appendix — API fallback when `glab` is unavailable

`glab` is a convenience layer, not a dependency. Every step above maps onto a REST call, so a missing
CLI is a change of syntax, not a reason to abandon the review. Steps 2 through 4 (large-diff triage,
blast radius, line-number derivation, verdict) are unchanged — only fetching and posting differ.

### Setup

```bash
command -v curl && command -v jq          # both required
: "${GITLAB_TOKEN:?export GITLAB_TOKEN in your own shell first}"
: "${GITLAB_HOST:=gitlab.mycompany.com}"
```

Ask the user to export `GITLAB_TOKEN` themselves. Same rule as before: a token pasted into chat is a
leaked token.

The project path must be URL-encoded — `group/my-repo` becomes `group%2Fmy-repo`. Nested groups have
multiple slashes, so encode rather than hand-writing it:

```bash
MR_ID=42
PROJECT=$(printf '%s' "group/subgroup/my-repo" | jq -sRr @uri)
API="https://${GITLAB_HOST}/api/v4/projects/${PROJECT}/merge_requests/${MR_ID}"
AUTH=(--header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" --fail --silent --show-error)
```

Pass the token via a header, never inline in a URL — URLs land in server logs and shell history.

Set all four in a **single** bash invocation together with the call that uses them, or re-declare them
each time. `AUTH` is a bash array; if each command runs in a fresh shell it will silently expand to
nothing and every request comes back as anonymous `401`/`404`.

### Command mapping

| Step | `glab` | REST fallback |
|---|---|---|
| MR metadata & state | `glab mr view <id>` | `curl "${AUTH[@]}" "$API"` |
| Prior discussions | `glab mr view <id> --comments --unresolved` | `curl "${AUTH[@]}" "$API/discussions?per_page=100"` |
| Diff | `glab mr diff <id>` | `curl "${AUTH[@]}" "$API/diffs?per_page=100"` |
| Changed file list | `glab mr diff <id> --raw \| grep '^+++'` | `curl "${AUTH[@]}" "$API/diffs" \| jq -r '.[].new_path'` |
| Issue context | `glab issue view <id>` | `curl "${AUTH[@]}" ".../issues/<ISSUE_ID>"` |
| Post note | `glab mr note <id> -m ...` | see below |
| Approve | `glab mr approve <id>` | `curl "${AUTH[@]}" -X POST "$API/approve"` |

Notes on the differences that actually matter:

- **`/diffs` is paginated.** Default page size is 20 files. A large MR will silently return a partial
  file list, which is exactly the failure mode Step 2 exists to prevent — always pass `per_page=100` and
  follow pagination, checking the `x-total-pages` response header where the endpoint returns it.
- **`/diffs` is recent.** On older self-hosted instances it 404s. Fall back to
  `GET "$API/changes"`, which returns the same file diffs under a `.changes[]` key rather than at the
  top level. A 404 here means "old GitLab", not "wrong MR ID" — don't send the user chasing the ID.
- **`/approve` may 403** depending on instance tier and the token's role, even when the MR is otherwise
  reachable. Treat a 403 on approve as "not permitted," not as a failed review — the note is already
  posted by that point.
- **`/discussions`, not `/notes`.** The `notes` endpoint returns a flat list with no `resolved` field.
  Resolution state lives on discussion notes, and Step 1 depends on it:
  ```bash
  curl "${AUTH[@]}" "$API/discussions?per_page=100" \
    | jq -r '.[].notes[] | select(.resolvable and (.resolved | not)) | "\(.author.username): \(.body)"'
  ```
- **`state` and `draft`** come from the MR object (`.state`, `.draft`, `.has_conflicts`) — feed these into
  the Step 1 state table exactly as before.

### Posting the review

The heredoc-to-file discipline still applies, and the API makes it easier: `--data-urlencode` with `@file`
reads the body straight off disk, so there's no JSON escaping to get wrong and no chance of the review's
backticks or `$` reaching the shell.

```bash
curl "${AUTH[@]}" "$API/notes" \
  --data-urlencode "body@/tmp/mr-review-${MR_ID}.md"
```

`--data-urlencode` implies `POST`, so `-X POST` is redundant here (and actively harmful if a redirect
turns the request into a `GET`).

Never build the JSON body by string-concatenating the review text. A single unescaped quote in a code
snippet will either corrupt the note or fail the request.

### Last resort — offline review

If there's no network to the GitLab host at all but the repo is checked out locally, you can still review
the code, just not fetch context or post:

```bash
git fetch origin
git diff "origin/${TARGET_BRANCH}...origin/${SOURCE_BRANCH}"
```

The triple-dot diffs against the merge base, which is what the MR actually shows — a two-dot diff will
include unrelated changes that landed on the target branch since the MR was opened.

State clearly what's degraded: no MR description, no issue context, no prior comments (so resolved items
may be re-raised), and the review cannot be posted. Deliver it in chat and offer to hand the user a
copy-pasteable markdown block for the MR.