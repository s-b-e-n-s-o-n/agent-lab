# GitHub CLI Commands

## Prerequisites

```bash
gh auth status  # Check auth
gh auth login   # Login if needed
```

## Command Reference

### DETECT_PR
Find PR for current branch.

```bash
BRANCH=$(git branch --show-current)
gh pr list --head "$BRANCH" --json number,title,url --limit 1
```

Output: `[{"number": 123, "title": "...", "url": "..."}]`

### GET_REPO_INFO
Get owner/repo from current directory.

```bash
gh repo view --json nameWithOwner --jq '.nameWithOwner'
```

Output: `owner/repo`

### FETCH_INLINE_COMMENTS
Get CodeRabbit inline review comments (nitpicks on specific lines).

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate \
  --jq '[.[] | select(.user.login | test("coderabbit"; "i")) | {
    id: .id,
    path: .path,
    line: .line,
    body: .body,
    in_reply_to_id: .in_reply_to_id,
    html_url: .html_url
  }]'
```

Output schema:
```json
[{
  "id": 12345678,
  "path": "src/api.ts",
  "line": 42,
  "body": "Comment text...",
  "in_reply_to_id": null,
  "html_url": "https://github.com/..."
}]
```

### FETCH_TOPLEVEL_COMMENTS
Get CodeRabbit top-level comments (summary, walkthrough).

```bash
gh api repos/{owner}/{repo}/issues/{pr}/comments --paginate \
  --jq '[.[] | select(.user.login | test("coderabbit"; "i")) | {
    id: .id,
    body: .body,
    html_url: .html_url
  }]'
```

Output schema:
```json
[{
  "id": 12345678,
  "body": "## Summary\n...",
  "html_url": "https://github.com/..."
}]
```

### REPLY_TO_INLINE
Reply to an inline review comment thread.

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="{response_text}"
```

Parameters:
- `{owner}` - Repository owner
- `{repo}` - Repository name
- `{pr}` - PR number
- `{comment_id}` - ID of the comment to reply to
- `{response_text}` - Your reply text

### REPLY_TO_TOPLEVEL
Create a new top-level PR comment.

```bash
gh pr comment {pr} --body "{response_text}"
```

### POST_CODERABBIT_COMMAND
Send a command to CodeRabbit.

```bash
gh pr comment {pr} --body "@coderabbitai {command}"
```

Commands: `resolve`, `review`, `full review`, `pause`, `resume`, `summary`

### GET_PR_FILES
List files changed in PR.

```bash
gh pr view {pr} --json files --jq '.files[].path'
```

### COUNT_CR_COMMENTS
Count CodeRabbit inline comments.

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate \
  --jq '[.[] | select(.user.login | test("coderabbit"; "i"))] | length'
```

### GET_THREAD_REPLIES
Get replies to a specific comment thread.

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate \
  --jq '[.[] | select(.in_reply_to_id == {comment_id})]'
```

### CHECK_RATE_LIMIT

```bash
gh api rate_limit --jq '.resources.core'
```

## Variable Substitution

| Variable | Source |
|----------|--------|
| `{owner}` | `gh repo view --json owner --jq '.owner.login'` |
| `{repo}` | `gh repo view --json name --jq '.name'` |
| `{pr}` | From DETECT_PR or user input |
| `{comment_id}` | From FETCH_INLINE_COMMENTS output |
| `{response_text}` | From response template |

## Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| 404 | PR/repo not found | Verify PR number and repo access |
| 403 | No access | Run `gh auth login` |
| 422 | Invalid params | Check comment_id is top-level (not reply) |
| 429 | Rate limited | Wait and retry |
