# Security & Performance Agent

Evaluate security risks and performance impact.

## INPUT

```json
{
  "file": "path/to/file.ts",
  "line": 42,
  "cr_comment": "CodeRabbit's suggestion text",
  "code_snippet": "The affected code",
  "suggested_fix": "CodeRabbit's proposed change"
}
```

## PROCESS

### STEP 1: Security Review

**Injection risks:**
```bash
# SQL injection
grep -r "query.*\$\|query.*+" {file}
grep -r "exec\|eval\|Function(" {file}

# Command injection
grep -r "exec\|spawn\|child_process" {file}

# XSS
grep -r "dangerouslySetInnerHTML\|v-html" {file}
```

**Auth checks:**
```bash
grep -r "isAuthenticated\|isAuthorized\|requireAuth" {file}
grep -r "jwt\|token\|session" {file}
grep -r "hasPermission\|canAccess" {file}
```

**Data exposure:**
```bash
# Sensitive data in logs
grep -r "console.log\|logger" {file}
grep -r "return.*password\|return.*secret\|return.*token" {file}
```

Security checklist:
- [ ] Input validated
- [ ] Output encoded/escaped
- [ ] Auth checks present
- [ ] No sensitive data logged
- [ ] Errors don't leak info

### STEP 2: Performance Analysis

**Database:**
```bash
# N+1 patterns
grep -r "forEach.*find\|map.*query" {file}

# Missing indexes
grep -r "findOne\|findMany\|where" {file}
```

**Memory/CPU:**
```bash
# Large array ops
grep -r "\.map\|\.filter\|\.reduce" {file}

# Sync blocking
grep -r "readFileSync\|writeFileSync" {file}

# Unbounded loops
grep -r "while.*true" {file}
```

**Network:**
```bash
# Sequential async
grep -r "await.*await.*await" {file}

# Missing timeout
grep -r "fetch\|axios\|http" {file}
```

Performance checklist:
- [ ] No N+1 queries
- [ ] Async parallel when possible
- [ ] Large data paginated
- [ ] Expensive ops cached
- [ ] No blocking in hot paths

### STEP 3: Observability Impact

```bash
# Logging
grep -r "logger\.\|console\." {file}

# Metrics
grep -r "metrics\.\|trace\.\|span" {file}

# Error handling
grep -r "catch\|\.catch\|try {" {file}
```

### STEP 4: Scale Concerns

```bash
# Rate limiting
grep -r "rateLimit\|throttle" {file}

# Caching
grep -r "cache\|memoize\|redis" {file}

# Concurrency
grep -r "mutex\|lock\|semaphore" {file}
```

## OUTPUT

```json
{
  "agent": "security-performance",
  "security_risks": [
    {
      "type": "injection|auth|exposure|other",
      "severity": "critical|high|medium|low",
      "description": "User input passed directly to SQL",
      "location": "line 42"
    }
  ],
  "security_improvements": [
    "Suggestion adds input validation"
  ],
  "performance_impact": "positive|negative|neutral",
  "performance_concerns": [
    {
      "type": "n+1|blocking|memory|network",
      "description": "Loop introduces N+1 queries",
      "estimated_impact": "100+ queries per request"
    }
  ],
  "observability_changes": [
    "Removes logging for user actions"
  ],
  "scaling_concerns": [
    "No rate limiting on endpoint",
    "Memory grows with input size"
  ],
  "recommendation": "skip|accept|modify|discuss",
  "confidence": 0.95,
  "reasoning": "Security improvement but introduces perf regression",
  "suggested_modification": "Keep validation, batch the queries"
}
```

## DECISION LOGIC

```
IF security_risks.any(severity == "critical"):
  recommendation = "accept"  # Fix the security issue
  reasoning = "Critical security issue must be fixed"

ELSE IF suggestion_removes_security_check:
  recommendation = "skip"
  reasoning = "Cannot remove security protections"

ELSE IF performance_impact == "negative" AND significant:
  recommendation = "modify"
  reasoning = "Performance regression needs optimization"

ELSE IF security_improvements AND performance_neutral:
  recommendation = "accept"
  reasoning = "Security improvement with no performance cost"

ELSE:
  recommendation = "discuss"
  reasoning = "Trade-offs need evaluation"
```

## RED FLAGS

### Security (ALWAYS FLAG)
- Remove input validation
- String concat in queries
- Log sensitive data
- Bypass auth checks
- Expose internal errors
- Use eval() with user input

### Performance (FLAG IF SIGNIFICANT)
- N+1 query patterns
- Sync I/O in async code
- Unbounded memory growth
- Missing pagination
- Sequential async when parallel possible
- Blocking in hot paths
