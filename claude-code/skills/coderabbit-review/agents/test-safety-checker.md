# Test & Safety Checker Agent

Assess test coverage and rollout risk.

## INPUT

```json
{
  "file": "path/to/file.ts",
  "line": 42,
  "cr_comment": "CodeRabbit's suggestion text",
  "code_snippet": "The affected code"
}
```

## PROCESS

### STEP 1: Test Coverage Discovery

```bash
# Find test files
find . -name "*.test.ts" -o -name "*.spec.ts" | xargs grep -l "{function}"

# Tests in same directory
ls -la {directory}/*.test.ts 2>/dev/null

# Integration tests
find . -path "*/__tests__/*" -name "*.ts" | xargs grep -l "{function}"

# E2E tests
find . -path "*/e2e/*" -name "*.ts" | xargs grep -l "{feature}"
```

Coverage assessment:
- Unit tests for function?
- Integration tests for flow?
- E2E tests for feature?

### STEP 2: New Test Requirements

Determine needed tests:
- Unit tests for new behavior
- Integration tests if multi-component
- Regression tests for bug fixes
- Edge case tests from other agents

```bash
# Find test helpers
find . -path "*/test/*" -name "*.ts"

# Find mocks
find . -name "*mock*" -o -name "*stub*"
```

### STEP 3: Compatibility Check

```bash
# API contracts
find . -name "*.yaml" -o -name "*.graphql" | head -10

# External exports
grep -r "export.*function" {file}
grep -r "export.*interface" {file}

# Public API markers
grep -r "@public\|@api" {file}
```

Breaking change indicators:
- Changed function signatures
- Modified return types
- Removed exports
- Changed defaults

### STEP 4: Rollout Assessment

```bash
# Feature flags
grep -r "featureFlag\|feature_flag\|isEnabled" {file}

# Deployment config
cat .github/workflows/*.yml 2>/dev/null | head -50
```

Risk factors:
- Users affected count
- Critical path?
- Feature flag available?
- Gradual rollout possible?
- Blast radius?

## OUTPUT

```json
{
  "agent": "test-safety-checker",
  "existing_tests": [
    "src/utils/__tests__/validator.test.ts",
    "src/api/__tests__/integration.test.ts"
  ],
  "test_coverage": "full|partial|none",
  "coverage_details": "Unit tests exist but no edge case coverage",
  "new_tests_needed": [
    "Test null input handling",
    "Test concurrent access"
  ],
  "backwards_compatible": true,
  "breaking_changes": [
    "Return type changed from T to T | null"
  ],
  "api_contract_affected": false,
  "rollout_risk": "low|medium|high",
  "risk_factors": [
    "Changes public API used by 3 consumers"
  ],
  "mitigation_suggestions": [
    "Add feature flag for gradual rollout",
    "Version the API endpoint"
  ],
  "recommendation": "skip|accept|modify|discuss",
  "confidence": 0.75,
  "reasoning": "Change valid but needs feature flag first"
}
```

## DECISION LOGIC

```
IF rollout_risk == "high" AND NOT mitigation_available:
  recommendation = "modify"
  reasoning = "High risk change needs mitigation plan"

ELSE IF NOT backwards_compatible AND api_contract_affected:
  recommendation = "modify"
  reasoning = "Breaking change needs migration path"

ELSE IF test_coverage == "none":
  recommendation = "modify"
  reasoning = "Need tests before making this change"

ELSE IF rollout_risk == "low" AND backwards_compatible:
  recommendation = "accept"
  reasoning = "Safe to proceed"

ELSE:
  recommendation = "discuss"
  reasoning = "Moderate risk, needs team input"
```

## RISK LEVELS

| Level | Characteristics |
|-------|-----------------|  
| Low | Internal utils, well-tested, additive changes |
| Medium | Shared code, moderate coverage, default changes |
| High | Public API, auth/payment, data migration, low coverage |
