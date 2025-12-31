# CodeRabbit Commands

Command lookup table for @coderabbitai interactions.

**KEY POINT:** CodeRabbit auto-reviews on every push. Manual review commands are edge cases.

## Common Commands

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `@coderabbitai resolve` | Mark all comments resolved | After addressing all feedback |
| `@coderabbitai pause` | Stop auto-reviews | During rapid iteration |
| `@coderabbitai resume` | Resume auto-reviews | After pausing, ready for reviews |
| `@coderabbitai summary` | Refresh PR description | After significant changes |

## Edge Case Commands (Rarely Needed)

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `@coderabbitai review` | Request review | WITHOUT pushing a commit |
| `@coderabbitai full review` | Force fresh review | Ignores incremental, starts over |
| `@coderabbitai generate docstrings` | Auto-generate docs | For undocumented functions |
| `@coderabbitai configuration` | Show current settings | To check repo config |
| `@coderabbitai help` | Show command reference | When unsure |

## Post Command

```bash
gh pr comment {pr} --body "@coderabbitai {command}"
```

## Workflow Commands

### After Addressing Comments (Most Common)

```
SEQUENCE:
1. Push fixes → git push (CodeRabbit auto-reviews)
2. Resolve → @coderabbitai resolve
# That's it - no manual review needed
```

### During Rapid Iteration

```
SEQUENCE:
1. Pause → @coderabbitai pause
2. Make multiple commits (no reviews)
3. Resume → @coderabbitai resume
4. Push → git push (auto-review triggers)
```

## Command Selection Logic

```
IF all_comments_addressed:
  POST "@coderabbitai resolve"
  # Push triggers auto-review, no manual request needed

ELSE IF rapid_iteration:
  POST "@coderabbitai pause"
  // ... make changes ...
  POST "@coderabbitai resume"

ELSE IF need_fresh_summary:
  POST "@coderabbitai summary"

ELSE IF want_review_without_pushing:
  POST "@coderabbitai review"  # Edge case only
```

## Disable Auto-Review

Add to PR description:
```
@coderabbitai ignore
```

Remove text to re-enable on next commit.

## Learning Behavior

CodeRabbit learns from responses:
- Quality feedback → Better future reviews
- Skip reasons → Understands team preferences
- Discussion threads → Refines understanding

This is why we ALWAYS respond to every comment.
