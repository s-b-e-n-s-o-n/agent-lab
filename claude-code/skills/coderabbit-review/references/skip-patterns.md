# Skip Patterns

Decision tree and category lookup for skipping CodeRabbit suggestions.

## Decision Tree

```
START
  │
  ▼
Is there a security concern?
  ├─ YES → DO NOT SKIP (accept)
  └─ NO ─┐
         ▼
Is this pattern used elsewhere intentionally?
  ├─ YES → SKIP: intentional
  │        Evidence: 2+ similar patterns in codebase
  └─ NO ─┐
         ▼
Does an existing abstraction apply?
  ├─ YES → SKIP: intentional
  │        Evidence: Team convention, documented pattern
  └─ NO ─┐
         ▼
Is this out of scope for this PR?
  ├─ YES → SKIP: out_of_scope
  │        Evidence: Tracking issue ID
  └─ NO ─┐
         ▼
Is this a known trade-off we've accepted?
  ├─ YES → SKIP: tradeoff
  │        Evidence: Performance/simplicity reasoning
  └─ NO ─┐
         ▼
Does a framework/library require this pattern?
  ├─ YES → SKIP: framework
  │        Evidence: Documentation link
  └─ NO ─┐
         ▼
Did CodeRabbit misunderstand the context?
  ├─ YES → SKIP: false_positive
  │        Evidence: What was missed (validation elsewhere, etc.)
  └─ NO ─┐
         ▼
Is this a style preference with no functional difference?
  ├─ YES → SKIP: style
  └─ NO → CONSIDER ACCEPTING
```

## Skip Categories

| Category | Key | Template | Required Evidence |
|----------|-----|----------|-------------------|
| **Duplicate** | `duplicate` | Auto-skip | None (auto-detected) |
| Intentional pattern | `intentional` | SKIPPED_INTENTIONAL | 2+ file:line refs |
| Out of scope | `out_of_scope` | SKIPPED_OUT_OF_SCOPE | Issue ID |
| Trade-off accepted | `tradeoff` | SKIPPED_TRADEOFF | Benefit vs cost |
| Framework requirement | `framework` | SKIPPED_FRAMEWORK | Docs link |
| False positive | `false_positive` | SKIPPED_FALSE_POSITIVE | What CR missed |
| Style preference | `style` | SKIPPED_STYLE | None |
| Backwards compat | `backwards_compat` | SKIPPED_FALSE_POSITIVE | Consumer info |
| Test coverage | `test_coverage` | SKIPPED_FALSE_POSITIVE | Test file:line |
| Known tech debt | `tech_debt` | SKIPPED_OUT_OF_SCOPE | TODO/Issue ref |

### Duplicate (Auto-Skip)

Duplicates are auto-detected and skipped without agent evaluation:

```json
{
  "skip_reason": "duplicate",
  "evidence": [],
  "reasoning": "CodeRabbit marked as duplicate"
}
```

**Auto-response:** `@coderabbitai Acknowledged - duplicate`

## Evidence Requirements

### intentional
```json
{
  "skip_reason": "intentional",
  "evidence": [
    {"file": "src/utils/helpers.ts", "line": 42},
    {"file": "src/api/handlers.ts", "line": 88}
  ],
  "reasoning": "Pattern used for performance in hot paths"
}
```

### out_of_scope
```json
{
  "skip_reason": "out_of_scope",
  "evidence": [
    {"issue": "LIN-456", "title": "Refactor validation layer"}
  ],
  "reasoning": "This PR focuses on auth flow only"
}
```

### tradeoff
```json
{
  "skip_reason": "tradeoff",
  "evidence": [
    {"type": "performance", "metric": "10x improvement"},
    {"type": "scale", "context": "1000 req/s"}
  ],
  "reasoning": "Readability cost acceptable for performance gain"
}
```

### framework
```json
{
  "skip_reason": "framework",
  "evidence": [
    {"framework": "Next.js", "docs": "https://nextjs.org/docs/..."}
  ],
  "reasoning": "Required for server components"
}
```

### false_positive
```json
{
  "skip_reason": "false_positive",
  "evidence": [
    {"type": "validation", "location": "src/middleware/validate.ts:23"},
    {"type": "guarantee", "description": "Caller ensures non-null"}
  ],
  "reasoning": "Input validated by middleware before reaching this code"
}
```

## Invalid Skip Reasons

DO NOT use these as skip reasons:

| Invalid | Why | Instead |
|---------|-----|--------|
| "I don't want to" | No evidence | Find evidence or accept |
| "It works fine" | Not a reason | Explain why it's correct |
| "Too much effort" | Avoid work | Create issue, use out_of_scope |
| "I don't understand" | Confusion | Use discuss, ask for clarification |
| "CodeRabbit is wrong" | No evidence | Provide evidence for false_positive |
| (no reason given) | Silent skip | Always provide reason |

## Skip Reason Selection Logic

```python
def select_skip_reason(agent_outputs):
    # Check for patterns in codebase
    if repo_archeologist.similar_patterns >= 2:
        return "intentional", repo_archeologist.evidence

    # Check for framework requirements
    if architecture_reviewer.framework_requirement:
        return "framework", architecture_reviewer.docs_link

    # Check for false positive indicators
    if behavior_validator.validation_elsewhere:
        return "false_positive", behavior_validator.validation_location

    # Check for scope issues
    if architecture_reviewer.requires_broader_refactor:
        return "out_of_scope", create_tracking_issue()

    # Check for trade-offs
    if security_performance.performance_critical:
        return "tradeoff", security_performance.metrics

    # Style preference (last resort)
    if all_agents.no_functional_concern:
        return "style", None

    # If no valid skip reason, don't skip
    return None, None
```
