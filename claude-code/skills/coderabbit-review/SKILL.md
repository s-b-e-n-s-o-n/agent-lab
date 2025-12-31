---
name: coderabbit-review
description: Evaluate CodeRabbit PR review comments using multi-agent analysis. Use when asked to "check in with coderabbit". Auto-discovers PRs, creates tracker files, auto-fixes simple issues, and always responds to CodeRabbit threads.
version: 1.1.0
---

# CodeRabbit Review Evaluation

## Triggers

ACTIVATE ON:
- "check in with coderabbit"
- "check in with CR"
- "review coderabbit"
- "check coderabbit" / "check CR"
- "respond to coderabbit"
- "evaluate CR comments"
- "coderabbit review"

## Execution

### STEP 1: Detect PR

```bash
# Get current branch
BRANCH=$(git branch --show-current)

# Find PR for branch
gh pr list --head "$BRANCH" --json number,title,url --limit 1
```

**Decision:**
- IF result empty → ASK: "Which PR number or URL?"
- ELSE → PARSE: `{ pr_number, pr_title, pr_url, owner, repo }`

### STEP 2: Fetch CodeRabbit Comments

**Inline comments (nitpicks on specific lines):**
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --paginate \
  --jq '[.[] | select(.user.login | test("coderabbit"; "i")) | {
    id: .id,
    path: .path,
    line: .line,
    body: .body,
    in_reply_to_id: .in_reply_to_id,
    html_url: .html_url
  }]'
```

**Top-level comments (summary, walkthrough):**
```bash
gh api repos/{owner}/{repo}/issues/{pr_number}/comments --paginate \
  --jq '[.[] | select(.user.login | test("coderabbit"; "i")) | {
    id: .id,
    body: .body,
    html_url: .html_url
  }]'
```

**Output schema:**
```json
{
  "inline_comments": [
    { "id": int, "path": str, "line": int, "body": str, "html_url": str }
  ],
  "toplevel_comments": [
    { "id": int, "body": str, "html_url": str }
  ]
}
```

### STEP 2b: Parse Comment Structure

For each comment body, extract structured data:

**1. Label Detection (first line):**
```
Pattern: _[emoji] Label_ | _[emoji] Label_
Examples:
  _⚠️ Potential issue_ | _🔴 Critical_  → type: critical
  _📝 Nitpick_                           → type: nitpick
```

| Pattern | Type |
|---------|------|
| `_🔴 Critical_` | critical |
| `_⚠️ Potential issue_` | warning |
| `_💡 Suggestion_` | suggestion |
| `_📝 Nitpick_` | nitpick |

**2. Collapsible Sections (`<details>` tags):**

Extract content from HTML details blocks:

**Review Details (top-level summary comment):**
| Emoji | Section | Extract |
|-------|---------|---------|  
| 📥 | Commits | List of commit SHAs reviewed |
| ⛔ | Files ignored | Count of ignored files |
| 📒 | Files selected for processing | Count of files reviewed |
| 💤 | Files with no reviewable changes | Count of skipped files |
| 🧰 | Additional context used | Context sources |
| 🔇 | Additional comments | Count of non-actionable comments |

**Per-Comment Sections:**
| Summary | Field | Use |
|---------|-------|-----|
| `🧩 Analysis chain` | `analysis_chain` | CR's reasoning, scripts run |
| `⚠️ Outside diff range` | `outside_diff: true` | Comment on unchanged lines |
| `📝 Related comments` | `related_comments` | Grouped suggestions |

**3. GitHub Alerts:**
```
> [!CAUTION]  → Elevate to critical
> [!WARNING]  → Elevate to warning
> [!NOTE]     → Informational (no action)
```

**4. Duplicate Detection:**
```
IF body contains "**Duplicate comment**" OR "> **Duplicate**":
  is_duplicate = true
  status = "skipped"
  skip_reason = "duplicate"
```

### STEP 3: Create Tracker File

**Condition:** IF total_comments >= 3

**Action:** CREATE `.coderabbit-tracker.md`

**ID assignment:**
- Parse each comment body for severity indicators
- `C1-C99` = critical (security, data loss, crash)
- `W1-W99` = warning (bugs, missing validation)
- `S1-S99` = suggestion (improvements, refactors)
- `N1-N99` = nitpick (style, naming, minor)

**Template:** See `references/tracker-format.md`

### STEP 4: Evaluate Each Comment

**Reasoning Mode Selection:**
```
IF comment involves security/auth/data → USE extended thinking
IF comment is complex architectural decision → CONSIDER plan mode
IF comment is straightforward nitpick → direct evaluation OK
```

**Auto-Fix for Simple Nitpicks:**

For nitpicks (`N` category), check if it's a simple fix that can be applied immediately:

| Type | Auto-Fix? | Example |
|------|-----------|---------|  
| Formatting/style | YES | Markdown tables, whitespace, import order |
| Simple refactor | YES | Use `.refine()` instead of if-check, optional chaining |
| Naming | YES | Rename variable for clarity |
| Add types | YES | Add missing type annotation |
| Intentional pattern | NO | Defensive code handling API inconsistency |
| Complex refactor | NO | Restructure module, change architecture |

```
IF nitpick.is_simple_fix AND NOT nitpick.is_intentional:
  → Apply fix immediately (no agent evaluation needed)
  → Respond: "@coderabbitai **Accepted** - Fixed in {commit}"

IF nitpick.would_break_something AND nitpick.is_valid_improvement:
  → Create Linear backlog issue
  → Respond: "@coderabbitai **Deferred** - Valid but risky, tracked in {LIN-XXX}"

IF nitpick.is_intentional:
  → Skip with evidence
  → Respond: "@coderabbitai **Skipped** - Intentional: {reason}"
```

**Linear issue creation (for deferred items):**
```bash
# Use Linear MCP tool
mcp__linear__create_issue:
  team: {current_team}
  title: "Refactor: {nitpick_summary}"
  description: |
    From CodeRabbit review on PR #{pr_number}

    **Suggestion:** {cr_comment}
    **Why deferred:** {breaking_reason}
    **File:** {file}:{line}
  labels: ["tech-debt", "refactor"]
  state: "Backlog"
```

**Simple fix criteria:**
- Single file change
- < 10 lines modified
- No behavior change (refactor only)
- No test changes needed
- Clear implementation path

FOR each comment:

1. **LAUNCH 5 agents in parallel** using Task tool with `subagent_type: "general-purpose"`:

| Agent | Prompt File | Purpose |
|-------|-------------|---------|  
| repo-archeologist | `agents/repo-archeologist.md` | Find similar patterns, git history |
| behavior-validator | `agents/behavior-validator.md` | Check correctness, edge cases |
| architecture-reviewer | `agents/architecture-reviewer.md` | System fit, coupling |
| test-safety-checker | `agents/test-safety-checker.md` | Test coverage, rollout risk |
| security-performance | `agents/security-performance.md` | Security, performance impact |

**Agent invocation pattern:**
```
Task tool call:
  subagent_type: "general-purpose"
  prompt: "Read agents/{agent-name}.md and follow its process for this input:
           {comment JSON with file, line, body, suggested_fix}
           Return JSON output as specified in the agent file."
  run_in_background: true  # Launch all 5 in parallel
```

2. **COLLECT agent outputs** using TaskOutput tool (all return JSON)

3. **SYNTHESIZE decision:**

```
IF any agent.recommendation == "critical" → decision = "accept"
ELSE IF majority agents.recommendation == "accept" → decision = "accept"
ELSE IF agents conflict → decision = "discuss"
ELSE IF all agents.recommendation == "skip" → decision = "skip"
ELSE → decision = "modify"
```

**Decision values:** `accept | modify | skip | discuss`

### STEP 5: Reply to Every Thread

**CRITICAL: Every CodeRabbit comment gets a response.**

**@coderabbitai MENTION RULES:**
```
ALWAYS include @coderabbitai at start of response for CodeRabbit to see/respond.
Without the @mention, CodeRabbit will NOT process or reply to your comment.

CONTEXT BY LOCATION:
├─ Files Changed tab (review comments) → HIGH context, CR knows file/line/diff
└─ PR comments (general) → LOW context, use only for commands

USE Files Changed tab for:
- Replying to inline nitpicks (reply in same thread)
- Asking questions about specific code
- Discussing implementation details

USE PR comments for:
- Commands: @coderabbitai resolve, pause, resume
- Summary responses covering multiple items
```

FOR each comment:

1. SELECT template from `references/response-templates.md` based on decision
   - All templates start with `@coderabbitai`
2. FILL template with:
   - `{reasoning}` from agent synthesis
   - `{evidence}` from agent outputs (file:line references)
   - `{commit}` if fix was made
3. POST reply:

**For inline comments (Files Changed tab - HIGH context):**
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies \
  -f body="@coderabbitai $RESPONSE"
```

**For top-level/summary comments (PR comments - LOW context):**
```bash
gh pr comment {pr_number} --body "@coderabbitai $RESPONSE"
```

### STEP 6: Update Tracker + STOP

1. UPDATE `.coderabbit-tracker.md`:
   - Set status for each comment
   - Add assessment and response link
   - Update summary counts

2. REPORT to user:
```
Evaluated {n} CodeRabbit comments:
- Accepted: {n}
- Modified: {n}
- Skipped: {n}
- Discussing: {n}

Tracker: .coderabbit-tracker.md
```

3. **STOP** - Wait for user instruction

## Iteration Limits

```yaml
max_rounds_default: 1
max_rounds_per_thread: 3
continue_triggers:
  - "work on {ID}"
  - "continue {ID}"
  - "keep going"
```

**Flow:**
```
Round 1: Evaluate all → Reply all → STOP
User: "work on N1"
Round 2: Check N1 thread → Reply → STOP
User: "continue"
Round 3: Continue → Reply → STOP (max reached, ask user)
```

## @coderabbitai Commands

**CodeRabbit auto-reviews on every push.** Manual review commands are rarely needed.

| Command | When to Use |
|---------|-------------|
| `@coderabbitai resolve` | After responding to all comments - marks threads resolved |
| `@coderabbitai pause` | Stop auto-reviews during rapid iteration |
| `@coderabbitai resume` | Resume auto-reviews after pause |
| `@coderabbitai summary` | Refresh PR description summary |

**Edge case commands (rarely needed):**

| Command | When to Use |
|---------|-------------|
| `@coderabbitai review` | Want review WITHOUT pushing a commit |
| `@coderabbitai full review` | Force fresh review (ignores incremental) |

**Typical workflow after fixes:**
```bash
# 1. Push fixes (CodeRabbit auto-reviews this)
git push

# 2. Resolve addressed comments
gh pr comment {pr} --body "@coderabbitai resolve"

# That's it - CodeRabbit already reviewed the push
```

## Data Schemas

### Comment Object
```json
{
  "id": "N1",
  "type": "nitpick|warning|suggestion|critical",
  "cr_label": "📝 Nitpick",
  "thread_id": 12345678,
  "file": "src/api.ts",
  "line": 42,
  "body": "CodeRabbit's comment text",
  "is_duplicate": false,
  "duplicate_of": null,
  "outside_diff": false,
  "analysis_chain": "Script executed: ...",
  "status": "pending|fixed|skipped|discussing",
  "decision": "accept|modify|skip|discuss",
  "response": "Our reply text",
  "evidence": ["file:line", "commit:sha"]
}
```

### Agent Output Schema
```json
{
  "agent": "repo-archeologist",
  "recommendation": "accept|skip|modify|discuss",
  "confidence": 0.0-1.0,
  "reasoning": "Explanation",
  "evidence": [
    { "type": "pattern|history|abstraction", "location": "file:line", "description": "..." }
  ],
  "concerns": ["List of concerns if any"]
}
```

## Reference Files

| File | Contains |
|------|----------|
| `agents/repo-archeologist.md` | Pattern/history search agent |
| `agents/behavior-validator.md` | Correctness validation agent |
| `agents/architecture-reviewer.md` | System fit analysis agent |
| `agents/test-safety-checker.md` | Test/rollout risk agent |
| `agents/security-performance.md` | Security/perf analysis agent |
| `references/github-commands.md` | gh CLI command syntax |
| `references/response-templates.md` | Reply templates by decision |
| `references/skip-patterns.md` | Valid skip reasons + decision tree |
| `references/tracker-format.md` | Tracker file schema |
| `references/coderabbit-commands.md` | @coderabbitai command reference |
