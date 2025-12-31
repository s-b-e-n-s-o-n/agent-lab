# Sample Evaluation Walkthrough

Complete example of evaluating CodeRabbit comments on a PR.

## Scenario

PR #456: "Add user authentication to API endpoints"
Branch: `feature/LIN-789-auth`

CodeRabbit posted 8 comments.

## Step 1: Auto-Detect and Fetch

```bash
# Get current branch
$ git branch --show-current
feature/LIN-789-auth

# Find the PR
$ gh pr list --head feature/LIN-789-auth --json number,title
[{"number": 456, "title": "Add user authentication to API endpoints"}]

# Get repo info
$ gh repo view --json nameWithOwner --jq '.nameWithOwner'
myorg/myrepo
```

## Step 2: Fetch CodeRabbit Comments

```bash
# Inline comments
$ gh api repos/myorg/myrepo/pulls/456/comments --paginate \
    --jq '[.[] | select(.user.login | test("coderabbit"; "i")) | {
      id: .id, path: .path, line: .line, body: .body
    }]'

# Found 6 inline comments

# Top-level comments
$ gh api repos/myorg/myrepo/issues/456/comments --paginate \
    --jq '[.[] | select(.user.login | test("coderabbit"; "i"))]'

# Found 2 top-level comments (summary + walkthrough)
```

## Step 3: Create Tracker File

Since we have 8 comments (≥3), create `.coderabbit-tracker.md`:

```markdown
# CodeRabbit Review Tracker

PR: #456 - Add user authentication to API endpoints
Branch: feature/LIN-789-auth
Generated: 2025-01-15 14:30

## Summary

| Category | Total | Pending |
|----------|-------|---------|
| Critical | 1 | 1 |
| Warning | 2 | 2 |
| Suggestion | 3 | 3 |
| Nitpick | 2 | 2 |
| **Total** | **8** | **8** |

## Comments

| ID | Category | File:Line | Status | Thread ID |
|----|----------|-----------|--------|-----------|
| C1 | critical | src/auth.ts:42 | pending | 11111111 |
| W1 | warning | src/api.ts:88 | pending | 22222222 |
| W2 | warning | src/middleware.ts:15 | pending | 33333333 |
| S1 | suggestion | src/utils.ts:23 | pending | 44444444 |
| S2 | suggestion | src/types.ts:56 | pending | 55555555 |
| S3 | suggestion | src/config.ts:12 | pending | 66666666 |
| N1 | nitpick | src/index.ts:5 | pending | 77777777 |
| N2 | nitpick | src/test.ts:34 | pending | 88888888 |
```

## Step 4: Evaluate Each Comment

### C1 - Critical: Hardcoded Secret

**CodeRabbit said:**
> The JWT secret is hardcoded in the source code. This is a security risk.
> Consider using environment variables.

**Run agents:**

**Repo Archeologist:**
- No similar patterns found (good - this is the only place)
- Git blame shows this was added in this PR
- No comments explaining why

**Behavior Validator:**
- Issue is real - secret would be exposed in repo
- Fix is correct - env vars are the right approach

**Architecture Reviewer:**
- Fix aligns with 12-factor app principles
- Other configs already use env vars

**Test Safety:**
- Need to update test setup to provide mock secret
- No breaking changes

**Security Performance:**
- CRITICAL: Hardcoded secrets are a major security risk
- Must fix before merge

**Decision:** Accept

**Response:**
```
**Accepted** - Critical security issue, fixing immediately.

Moving JWT secret to environment variable. Also adding to `.env.example`
for documentation.

Fixed in commit abc1234.
```

---

### W1 - Warning: Missing Error Handling

**CodeRabbit said:**
> The `verifyToken` function doesn't handle the case where the token is malformed.
> This could throw an unhandled exception.

**Run agents:**

**Behavior Validator:**
- Issue is real - jwt.verify throws on malformed tokens
- Fix is correct - need try/catch

**Test Safety:**
- Tests exist but don't cover malformed token case
- Need to add test

**Decision:** Accept

**Response:**
```
**Accepted** - Good catch.

Added try/catch and returning 401 for malformed tokens.
Added test case for this scenario.
```

---

### W2 - Warning: Input Validation

**CodeRabbit said:**
> The `userId` parameter should be validated before querying the database.

**Run agents:**

**Repo Archeologist:**
- Found validation middleware at src/middleware/validate.ts
- All routes use this middleware
- This endpoint has validation in route definition

**Behavior Validator:**
- Input IS validated - by middleware before this function
- Function is internal, always receives validated input

**Decision:** Skip

**Response:**
```
**Skipped** - Validated by middleware.

The `userId` is validated by our validation middleware before reaching this handler.
See: `src/middleware/validate.ts:23` and route definition at `src/routes/users.ts:45`.

All internal functions assume validated input - validation happens at the boundary.
```

---

### S1 - Suggestion: Use Optional Chaining

**CodeRabbit said:**
> Consider using optional chaining: `user?.profile?.name` instead of
> `user && user.profile && user.profile.name`

**Run agents:**

**Repo Archeologist:**
- Codebase already uses optional chaining elsewhere
- This appears to be older pattern

**Architecture Reviewer:**
- Modern JS, we should use optional chaining
- Consistent with rest of codebase

**Decision:** Accept

**Response:**
```
**Accepted** - Good modernization suggestion.

Updated to use optional chaining for consistency with rest of codebase.
```

---

### S2 - Suggestion: Extract Magic Number

**CodeRabbit said:**
> Consider extracting the magic number `3600` to a named constant like `TOKEN_EXPIRY_SECONDS`.

**Run agents:**

**Repo Archeologist:**
- Found constants file at src/constants.ts
- Other magic numbers are already extracted there

**Architecture Reviewer:**
- Consistent with our pattern for config values

**Decision:** Accept

**Response:**
```
**Accepted** - Aligns with our constants pattern.

Added `TOKEN_EXPIRY_SECONDS` to `src/constants.ts`.
```

---

### S3 - Suggestion: Add JSDoc

**CodeRabbit said:**
> Consider adding JSDoc documentation to the `authenticateUser` function.

**Run agents:**

**Repo Archeologist:**
- Mixed patterns in codebase
- Public APIs have JSDoc, internal functions don't always
- This is a public API function

**Decision:** Accept

**Response:**
```
**Accepted** - This is a public API, should be documented.

Added JSDoc with parameter and return type documentation.
```

---

### N1 - Nitpick: Import Order

**CodeRabbit said:**
> Consider organizing imports: external modules first, then internal modules.

**Run agents:**

**Repo Archeologist:**
- We have eslint-plugin-import configured
- This file wasn't linted? Check why.

**Decision:** Accept (with note)

**Response:**
```
**Accepted** - Fixed.

Also investigated why lint didn't catch this - file was missing from lint config.
Added to lint includes.
```

---

### N2 - Nitpick: Trailing Whitespace

**CodeRabbit said:**
> There's trailing whitespace on line 34.

**Run agents:**

(Simple issue, minimal analysis needed)

**Decision:** Accept

**Response:**
```
**Accepted** - Fixed.
```

## Step 5: Update Tracker

Final state of `.coderabbit-tracker.md`:

```markdown
# CodeRabbit Review Tracker

PR: #456 - Add user authentication to API endpoints
Branch: feature/LIN-789-auth
Generated: 2025-01-15 14:30
Last Updated: 2025-01-15 15:45

## Summary

| Category | Total | Fixed | Skipped |
|----------|-------|-------|---------|
| Critical | 1 | 1 | 0 |
| Warning | 2 | 1 | 1 |
| Suggestion | 3 | 3 | 0 |
| Nitpick | 2 | 2 | 0 |
| **Total** | **8** | **7** | **1** |

## Comments

| ID | Category | File:Line | Status | Thread ID |
|----|----------|-----------|--------|-----------|
| C1 | critical | src/auth.ts:42 | fixed | 11111111 |
| W1 | warning | src/api.ts:88 | fixed | 22222222 |
| W2 | warning | src/middleware.ts:15 | skipped | 33333333 |
| S1 | suggestion | src/utils.ts:23 | fixed | 44444444 |
| S2 | suggestion | src/types.ts:56 | fixed | 55555555 |
| S3 | suggestion | src/config.ts:12 | fixed | 66666666 |
| N1 | nitpick | src/index.ts:5 | fixed | 77777777 |
| N2 | nitpick | src/test.ts:34 | fixed | 88888888 |
```

## Step 6: Summary to User

```
CodeRabbit Review Complete for PR #456

Results:
- 7 accepted and fixed
- 1 skipped (W2 - validation handled by middleware)

Key actions:
- C1: Fixed critical security issue (hardcoded secret)
- W1: Added error handling for malformed tokens
- W2: Skipped - validated by middleware
- S1-S3: Code quality improvements implemented
- N1-N2: Minor fixes

All responses posted to CodeRabbit threads.
Tracker saved to .coderabbit-tracker.md
```

## Conversation Example

If CodeRabbit responds to our skip on W2:

**CodeRabbit:**
> Thanks for the clarification. However, defense in depth suggests adding
> validation at multiple layers. What if the middleware is bypassed?

**Our response (Round 2):**
```
**Discussing** - Good point about defense in depth.

You're right that multiple validation layers are safer. However:

1. The middleware cannot be bypassed - it's applied at the route level
2. Adding redundant validation would be inconsistent with our patterns
3. We test the middleware thoroughly

For internal functions, we document the contract (validated input expected)
rather than re-validating. This keeps code DRY and tests focused.

If we were to add defense in depth, it would be a codebase-wide change
not scoped to this PR. Created LIN-999 to discuss with team.
```

**STOP** - Report to user and wait for instruction to continue.
