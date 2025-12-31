# Tracker Format

JSON schema and template for `.coderabbit-tracker.md`.

## Create Condition

```
IF total_comments >= 3 → CREATE tracker
ELSE → SKIP tracker creation
```

## File Location

```
{repo_root}/.coderabbit-tracker.md
```

## ID Assignment

### Priority 1: CodeRabbit Labels (Preferred)

Parse the first line of comment body for explicit labels:

| Pattern | Prefix | Category |
|---------|--------|----------|
| `_🔴 Critical_` | `C` | Critical |
| `_⚠️ Potential issue_` | `W` | Warning |
| `_💡 Suggestion_` | `S` | Suggestion |
| `_📝 Nitpick_` | `N` | Nitpick |

Also check for GitHub alert syntax:
| Alert | Category |
|-------|----------|
| `> [!CAUTION]` | Critical |
| `> [!WARNING]` | Warning |

### Priority 2: Keyword Fallback

If no explicit label found, use keyword matching:

| Prefix | Category | Indicators in CR comment |
|--------|----------|-------------------------|
| `C` | Critical | "security", "vulnerability", "injection", "data loss", "crash" |
| `W` | Warning | "bug", "error", "missing", "should", "validation" |
| `S` | Suggestion | "consider", "could", "might", "improvement", "refactor" |
| `N` | Nitpick | "nit", "style", "naming", "formatting", "minor" |

### Duplicate Detection

If comment contains `**Duplicate comment**` or `> **Duplicate**`:
- Set `is_duplicate: true`
- Set `status: skipped`
- Set `skip_reason: duplicate`
- Auto-respond: `@coderabbitai Acknowledged - duplicate`

Sequential numbering: `C1, C2, C3...`, `W1, W2, W3...`, etc.

## Status Values

| Status | Meaning |
|--------|--------|
| `pending` | Not yet evaluated |
| `in-progress` | Currently working on |
| `fixed` | Accepted and implemented |
| `skipped` | Decided not to implement |
| `modified` | Implemented differently |
| `discussing` | In conversation with CR |

## Data Schema

```json
{
  "pr": {
    "number": 123,
    "title": "Feature title",
    "branch": "feature/LIN-456-auth",
    "url": "https://github.com/..."
  },
  "review_details": {
    "commits_reviewed": ["abc123", "def456"],
    "files_ignored": 1,
    "files_selected": 20,
    "files_no_changes": 2,
    "additional_comments": 26
  },
  "generated": "2025-01-15T10:30:00Z",
  "last_updated": "2025-01-15T14:45:00Z",
  "summary": {
    "critical": {"total": 1, "pending": 0, "fixed": 1, "skipped": 0, "modified": 0, "discussing": 0},
    "warning": {"total": 3, "pending": 1, "fixed": 1, "skipped": 1, "modified": 0, "discussing": 0},
    "suggestion": {"total": 5, "pending": 2, "fixed": 2, "skipped": 0, "modified": 1, "discussing": 0},
    "nitpick": {"total": 4, "pending": 0, "fixed": 3, "skipped": 1, "modified": 0, "discussing": 0}
  },
  "comments": [
    {
      "id": "C1",
      "type": "critical",
      "cr_label": "🔴 Critical",
      "file": "src/auth.ts",
      "line": 42,
      "thread_id": 12345678,
      "status": "fixed",
      "is_duplicate": false,
      "duplicate_of": null,
      "outside_diff": false,
      "title": "SQL injection risk",
      "cr_body": "User input is passed directly...",
      "analysis_chain": "Script executed: grep -r 'query'...",
      "assessment": "Valid critical issue",
      "response": "Accepted - Fixed in abc1234",
      "thread_url": "https://github.com/..."
    }
  ]
}
```

## Markdown Template

```markdown
# CodeRabbit Review Tracker

PR: #{pr.number} - {pr.title}
Branch: {pr.branch}
Generated: {generated}
Last Updated: {last_updated}

## Summary

| Category | Total | Pending | Fixed | Skipped | Modified | Discussing |
|----------|-------|---------|-------|---------|----------|------------|
| Critical | {summary.critical.total} | {summary.critical.pending} | {summary.critical.fixed} | {summary.critical.skipped} | {summary.critical.modified} | {summary.critical.discussing} |
| Warning | {summary.warning.total} | {summary.warning.pending} | {summary.warning.fixed} | {summary.warning.skipped} | {summary.warning.modified} | {summary.warning.discussing} |
| Suggestion | {summary.suggestion.total} | {summary.suggestion.pending} | {summary.suggestion.fixed} | {summary.suggestion.skipped} | {summary.suggestion.modified} | {summary.suggestion.discussing} |
| Nitpick | {summary.nitpick.total} | {summary.nitpick.pending} | {summary.nitpick.fixed} | {summary.nitpick.skipped} | {summary.nitpick.modified} | {summary.nitpick.discussing} |

## Comments

| ID | Category | File:Line | Status | Thread ID |
|----|----------|-----------|--------|----------|
{FOR comment IN comments}
| {comment.id} | {comment.type} | {comment.file}:{comment.line} | {comment.status} | {comment.thread_id} |
{END FOR}

## Details

{FOR comment IN comments}
### {comment.id} - {comment.title} ({comment.status})
**File:** {comment.file}:{comment.line}
**CodeRabbit said:**
> {comment.cr_body}

**Our assessment:** {comment.assessment}
**Response:** {comment.response}
**Thread:** {comment.thread_url}

---

{END FOR}
```

## Update Operations

### Update status
```python
def update_status(tracker, comment_id, new_status, response=None):
    comment = tracker.comments.find(id=comment_id)
    old_status = comment.status
    comment.status = new_status
    if response:
        comment.response = response
    # Update summary counts
    tracker.summary[comment.type][old_status] -= 1
    tracker.summary[comment.type][new_status] += 1
    tracker.last_updated = now()
```

### Add assessment
```python
def add_assessment(tracker, comment_id, assessment, response):
    comment = tracker.comments.find(id=comment_id)
    comment.assessment = assessment
    comment.response = response
    tracker.last_updated = now()
```

## User Reference

Use IDs in conversation:
- "work on N1"
- "skip all nitpicks"
- "what's the status of C1?"
- "mark S1-S3 as fixed"
