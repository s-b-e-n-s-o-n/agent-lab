# Evaluation Criteria Reference

Detailed criteria for evaluating CodeRabbit comments across all five analysis agents.

## Nitpick Fast-Track (Auto-Fix)

**AI time is cheap.** For simple nitpicks, fix immediately instead of documenting as "low priority":

```
Nitpick received
    │
    ▼
Is it a simple fix? (< 10 lines, single file, no behavior change)
    ├─ NO → Run full agent evaluation
    └─ YES ─┐
            ▼
Would it break something?
    ├─ YES ─┐
    │       ▼
    │   Is it still a valid improvement?
    │       ├─ YES → Create Linear backlog issue
    │       │        Response: "@coderabbitai Deferred - tracked in LIN-XXX"
    │       └─ NO → Skip
    └─ NO ─┐
           ▼
Is the current code intentional? (handles edge case, API quirk, etc.)
    ├─ YES → Skip with evidence
    │        Response: "@coderabbitai Skipped - Intentional: {reason}"
    └─ NO ─┐
           ▼
FIX IT NOW
    Response: "@coderabbitai Accepted - Fixed in {commit}"
```

**Auto-fix examples:**
| Nitpick | Action |
|---------|--------|
| "Use Zod .refine() instead of if-check" | Fix it |
| "Use optional chaining" | Fix it |
| "Markdown table formatting" | Fix it |
| "Import order" | Fix it |
| "Type assertion handles API inconsistency" | Skip (intentional) |
| "Defensive null check for edge case" | Skip (intentional) |
| "Migrate to new API format" (breaking) | Defer → Linear backlog |
| "Refactor to use shared util" (large scope) | Defer → Linear backlog |

## Quick Decision Matrix

| Signal | Accept | Modify | Skip | Discuss |
|--------|--------|--------|------|--------|
| Issue is real | Yes | Yes | No | Unclear |
| Fix is correct | Yes | No | - | Unclear |
| Fits architecture | Yes | Needs work | No | Unclear |
| Has tests/safe | Yes | Needs tests | - | - |
| No security risk | Yes | Yes | Risk! | - |

## Agent A: Repo Archeologist

### Evidence to Find

| What to Look For | Where to Look | What It Means |
|-----------------|---------------|---------------|
| Same pattern elsewhere | `grep -r` codebase | Pattern is intentional |
| Existing abstraction | `utils/`, `helpers/` | Should use existing |
| Recent changes | `git log <file>` | Context on design |
| Reverted attempts | `git log --diff-filter=R` | Previous failure |
| Design comments | Source comments | Intentional design |

### Decision Criteria

**Skip if:**
- Same pattern exists in 2+ other places
- Git history shows intentional choice
- Existing abstraction exists and is being followed
- Previous attempt at change was reverted

**Accept if:**
- Pattern is inconsistent with rest of codebase
- Better abstraction exists that should be used
- Code appears to be accidental, not intentional

**Discuss if:**
- History is unclear
- Pattern is inconsistent but might be intentional

## Agent B: Behavior Validator

### Validation Checklist

| Check | Pass | Fail |
|-------|------|------|
| Issue actually occurs | Code path executes | Dead code/unreachable |
| Not handled elsewhere | Truly missing | Handled by caller/framework |
| Fix is complete | Addresses root cause | Only symptom |
| No new bugs | Same or better behavior | Regression |
| Edge cases covered | All paths work | New failure modes |

### Edge Cases to Consider

- **Null/undefined inputs**: What if values are missing?
- **Empty collections**: Arrays with 0 items
- **Boundary values**: 0, -1, MAX_INT, empty string
- **Concurrent access**: Race conditions
- **Error states**: External service failures
- **Type coercion**: JavaScript implicit conversions

### Decision Criteria

**Skip if:**
- Issue doesn't actually occur in practice
- Handled by caller/wrapper/framework
- Current behavior is intentional

**Modify if:**
- Issue is real but fix is incomplete
- Fix introduces new edge case failures
- Better fix approach exists

**Accept if:**
- Issue is real and fix is correct
- All edge cases handled
- No regressions introduced

## Agent C: Architecture Reviewer

### Layer Boundaries

| Layer | Can Import | Cannot Import |
|-------|------------|---------------|
| UI/Components | Services, Utils, Types | Database, External APIs |
| Services/API | Domain, Utils, Types | UI, Database directly |
| Domain | Utils, Types | UI, Services, Database |
| Data/Repository | Database, Types | UI, Services, Domain |
| Utils | Types only | Everything else |

### Coupling Indicators

| Indicator | Healthy | Unhealthy |
|-----------|---------|----------|
| Imports | 3-7 per file | 15+ per file |
| Circular deps | None | Any |
| Cross-module | Through interfaces | Direct implementation |
| Abstraction | Appropriate level | Over/under abstracted |

### Decision Criteria

**Skip if:**
- Suggestion violates layer boundaries
- Would increase coupling significantly
- Inconsistent with established patterns
- Creates circular dependencies

**Modify if:**
- Direction is right but approach is wrong
- Needs to go through proper abstraction layer
- Should follow existing patterns

**Accept if:**
- Respects architecture
- Reduces coupling
- Consistent with patterns
- Makes future changes easier

## Agent D: Test & Safety Checker

### Test Coverage Requirements

| Change Type | Required Tests |
|-------------|----------------|
| Bug fix | Regression test |
| New feature | Unit + integration |
| Refactor | Existing tests pass |
| API change | Contract tests |
| Security fix | Security test |

### Risk Assessment Matrix

| Factor | Low Risk | Medium Risk | High Risk |
|--------|----------|-------------|----------|
| Test coverage | >80% | 50-80% | <50% |
| Users affected | Internal only | Some users | All users |
| Reversibility | Easy rollback | Manual rollback | No rollback |
| Dependencies | None | Internal only | External |

### Decision Criteria

**Accept if:**
- Good test coverage exists
- Change is backwards compatible
- Low rollout risk
- Easy to verify

**Modify if:**
- Tests needed first
- Breaking change needs migration
- Feature flag recommended
- Gradual rollout needed

**Skip if:**
- Would break existing consumers
- No way to safely rollout
- Risk too high for benefit

## Agent E: Security & Performance

### Security Checklist

| Category | Check | Red Flag |
|----------|-------|----------|
| Injection | Parameterized queries | String concatenation with user input |
| XSS | Output encoding | Raw HTML from user |
| Auth | Session validation | Trust client-side auth |
| Data | Minimal exposure | Return more than needed |
| Logging | No secrets | Passwords/tokens in logs |
| Errors | Generic messages | Stack traces to users |

### Performance Checklist

| Category | Good | Bad |
|----------|------|-----|
| Queries | Batched, indexed | N+1, full scans |
| Async | Parallel when possible | Sequential always |
| Memory | Bounded, streamed | Unbounded, all in memory |
| Caching | Appropriate TTL | No cache or cache everything |
| I/O | Async, non-blocking | Sync in hot paths |

### Decision Criteria

**Reject immediately if:**
- Introduces injection vulnerability
- Removes authentication check
- Logs sensitive data
- Bypasses authorization

**Accept if:**
- Improves security posture
- No performance regression
- Maintains observability
- Scales appropriately

**Modify if:**
- Security improvement but perf regression
- Needs rate limiting
- Needs additional validation

## Synthesis: Final Decision

After all agents report, synthesize:

1. **Critical issues** (security, correctness) override other concerns
2. **Consistency** with codebase patterns is important
3. **Evidence** from git history and existing code matters
4. **Risk** must be balanced against benefit
5. **Tests** must support any accepted change

### Final Decision Tree

```
Is there a security risk?
├── Yes → Reject or require fix
└── No → Continue

Is the issue real?
├── No → Skip (false positive)
└── Yes/Maybe → Continue

Is the fix correct?
├── No → Modify (provide alternative)
└── Yes → Continue

Does it fit architecture?
├── No → Skip or Modify
└── Yes → Continue

Is it safe to deploy?
├── No → Modify (add tests, flag)
└── Yes → Accept
```
