# Response Templates

Template lookup for CodeRabbit responses. Every comment gets a response.

**IMPORTANT:** All responses MUST start with `@coderabbitai` for CodeRabbit to see and respond.

## Template Selection

```
DECISION → TEMPLATE_KEY
─────────────────────────
accept   → ACCEPTED | ACCEPTED_WITH_COMMIT
modify   → MODIFIED_APPROACH
skip     → SKIPPED_{REASON}
discuss  → QUESTION | DISCUSSING
```

## Templates

### ACCEPTED

```
@coderabbitai **Accepted** - {brief_reason}
```

Variables:
- `{brief_reason}` - 1 sentence why (e.g., "Good catch, implementing this fix.")

### ACCEPTED_WITH_COMMIT

```
@coderabbitai **Accepted** - Fixed in {commit_sha}.

{description}
```

Variables:
- `{commit_sha}` - Short commit hash (7 chars)
- `{description}` - 1-2 sentences describing the fix

### ACCEPTED_CRITICAL

```
@coderabbitai **Accepted** - Critical issue, fixing immediately.

{description}
```

### WILL_FIX

```
@coderabbitai **Will fix** - {acknowledgment}

{note}
```

Variables:
- `{acknowledgment}` - "Valid point" / "Good catch"
- `{note}` - Optional brief note on approach

### MODIFIED_APPROACH

```
@coderabbitai **Modified approach** - {acknowledgment}

{explanation}

{evidence}
```

Variables:
- `{acknowledgment}` - "Agree with the concern, different solution."
- `{explanation}` - Why using alternative approach
- `{evidence}` - File:line references supporting the approach

### SKIPPED_INTENTIONAL

```
@coderabbitai **Skipped** - Intentional design.

This pattern is used intentionally because {reason}.
See similar usage in:
{evidence_list}
```

Variables:
- `{reason}` - Why the pattern exists
- `{evidence_list}` - Bullet list of `- file:line` references

### SKIPPED_OUT_OF_SCOPE

```
@coderabbitai **Skipped** - Out of scope for this PR.

Valid suggestion, tracked in {issue_ref}.
This PR focuses on {scope}.
```

Variables:
- `{issue_ref}` - Issue ID (e.g., "LIN-456")
- `{scope}` - What this PR is specifically doing

### SKIPPED_TRADEOFF

```
@coderabbitai **Skipped** - Acceptable trade-off.

The suggested approach would {benefit}, but at the cost of {cost}.
For our {context}, the current approach is preferred because {reason}.
```

Variables:
- `{benefit}` - What the suggestion would improve
- `{cost}` - What would be sacrificed
- `{context}` - Scale/use case/situation
- `{reason}` - Why current choice is better here

### SKIPPED_FALSE_POSITIVE

```
@coderabbitai **Skipped** - Not applicable here.

{explanation}:
{evidence_list}
```

Variables:
- `{explanation}` - What CodeRabbit missed
- `{evidence_list}` - Bullet list of evidence

### SKIPPED_FRAMEWORK

```
@coderabbitai **Skipped** - Framework requirement.

This pattern is required by {framework} for {reason}.
See: {docs_link}
```

Variables:
- `{framework}` - Framework/library name
- `{reason}` - Why it's required
- `{docs_link}` - Link to documentation

### SKIPPED_STYLE

```
@coderabbitai **Skipped** - Style preference, keeping current.
```

### DISAGREE

```
@coderabbitai **Disagree** - {statement}

{reasoning}

Evidence:
{evidence_list}
```

Variables:
- `{statement}` - "I think this suggestion is incorrect." / "This would introduce a bug."
- `{reasoning}` - Why the current code is correct
- `{evidence_list}` - Bullet list of evidence

### QUESTION

```
@coderabbitai **Question** - Need clarification.

{question}
```

Variables:
- `{question}` - Specific question for CodeRabbit

### DISCUSSING

```
@coderabbitai **Discussing** - Let's explore this.

{points}

What do you think about {specific_approach}?
```

Variables:
- `{points}` - Numbered list of considerations
- `{specific_approach}` - Alternative being proposed

## Template Selection Logic

```
IF decision == "accept":
  IF has_commit:
    USE ACCEPTED_WITH_COMMIT
  ELSE IF is_critical:
    USE ACCEPTED_CRITICAL
  ELSE:
    USE ACCEPTED

ELSE IF decision == "modify":
  USE MODIFIED_APPROACH

ELSE IF decision == "skip":
  SELECT by skip_reason:
    "intentional"     → SKIPPED_INTENTIONAL
    "out_of_scope"    → SKIPPED_OUT_OF_SCOPE
    "tradeoff"        → SKIPPED_TRADEOFF
    "false_positive"  → SKIPPED_FALSE_POSITIVE
    "framework"       → SKIPPED_FRAMEWORK
    "style"           → SKIPPED_STYLE

ELSE IF decision == "discuss":
  IF needs_clarification:
    USE QUESTION
  ELSE:
    USE DISCUSSING
```

## After All Responses

Post this command to mark threads resolved:
```
@coderabbitai resolve
```

No need to request review - CodeRabbit auto-reviews when you push fixes.
