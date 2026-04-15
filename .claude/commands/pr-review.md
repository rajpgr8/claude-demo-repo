---
description: Review the current branch diff before raising a PR — checks logic, security, tests, style
---

## Branch Info
!`git branch --show-current`

## Commits on This Branch
!`git log --oneline main...HEAD`

## Changed Files
!`git diff --name-status main...HEAD`

## Full Diff
!`git diff main...HEAD`

---

Review every changed file carefully. Structure your feedback as follows:

### BLOCKERS — must fix before merge
- Security issues: hardcoded secrets, injection risks, overly-permissive IAM
- Broken logic or data-loss risk
- Missing migration downgrade() function
- Tests deleted without replacement
- New endpoint without auth/authz dependency
- Direct os.environ usage instead of config module
- Synchronous blocking I/O inside async function

### WARNINGS — should fix
- Missing type hints on new functions
- New endpoint missing OTel span
- No error handling on external API / GCP service calls
- Hardcoded project IDs, bucket names, regions
- print() or bare logging.info() instead of structlog
- Missing __all__ on new module
- No docstring on public functions or classes

### SUGGESTIONS — nice to have
- Readability improvements
- Test coverage gaps (not blocking)
- Docstring or comment additions
- Naming consistency issues

### SUMMARY
- What this PR does well
- Biggest risk area
- Risk level: LOW / MEDIUM / HIGH
- Recommendation: APPROVE / REQUEST CHANGES / NEEDS DISCUSSION
