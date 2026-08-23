# API Fallback Reference

Use these REST endpoints only when the CLI (`gh` / `glab`) is unavailable or unauthenticated. Read the token from the environment — never hard-code or print it.

Token env vars to check, in order:
- GitHub: `GITHUB_TOKEN`, then `GH_TOKEN`
- GitLab: `GITLAB_TOKEN`, then `GL_TOKEN`

If none is set and no CLI works, stop and ask the user for a token or to paste the issue text. Do not proceed blind.

Check them portably — `[ -n "$GITHUB_TOKEN" ]` works everywhere. Avoid bash-only tricks like `${!var}` indirect expansion, which fail under `/bin/sh`.

---

## GitHub REST API

Base: `https://api.github.com` (for GitHub Enterprise, use `https://<host>/api/v3`).
Auth header: `Authorization: Bearer $GITHUB_TOKEN`.

### Get an issue
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/issues/NUMBER
```

### Get issue comments
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/issues/NUMBER/comments
```

### Get a PR (body contains the "Closes #N" references)
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/pulls/NUMBER
```
Parse the `body` field for `Closes #N` / `Fixes #N` / `Resolves #N`. There is also a GraphQL `closingIssuesReferences` field, but for a fallback the body scan is simpler.

### List files changed in a PR
```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/pulls/NUMBER/files
```

### Create a PR
```bash
curl -s -X POST -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/pulls \
  -d '{"title":"<title>","head":"<current-branch>","base":"<default-branch>","body":"Closes #<issue>. <summary>"}'
```
The branch must be pushed to the remote first: `git push -u origin HEAD`.

---

## GitLab REST API

Base: `https://gitlab.com/api/v4` (self-hosted: `https://<host>/api/v4`).
Auth header: `PRIVATE-TOKEN: $GITLAB_TOKEN`.

The project id must be URL-encoded, e.g. `mygroup/myproject` → `mygroup%2Fmyproject`.

### Get an issue
```bash
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/PROJECT_ID/issues/IID"
```

### Get issue notes (comments)
```bash
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/PROJECT_ID/issues/IID/notes"
```

### Get an MR (description contains "Closes #N")
```bash
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/PROJECT_ID/merge_requests/IID"
```

### Get issues closed by an MR
```bash
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/PROJECT_ID/merge_requests/IID/closes_issues"
```
This is GitLab's reliable equivalent of GitHub's `closingIssuesReferences`.

### Get MR changes
```bash
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/PROJECT_ID/merge_requests/IID/changes"
```

### Create an MR
```bash
curl -s -X POST -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/PROJECT_ID/merge_requests" \
  -d "source_branch=<current-branch>" \
  -d "target_branch=<default-branch>" \
  -d "title=<title>" \
  -d "description=Closes #<issue>. <summary>"
```
Push the branch first: `git push -u origin HEAD`.

---

## Notes

- Prefer `jq` for parsing JSON responses if available (`curl ... | jq '.title, .body'`).
- Rate limits: unauthenticated GitHub requests are heavily limited, which is another reason to always use a token.
- For self-hosted instances, only the base URL changes; the paths are the same.