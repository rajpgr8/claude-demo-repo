# claude-demo-repo

```
your-project/
├── CLAUDE.md                    # ← lives at project ROOT, not inside .claude/
├── .mcp.json                    # ← also at ROOT
└── .claude/
    ├── settings.json            # Permissions, hooks, tool controls
    ├── settings.local.json      # Personal overrides (gitignored)
    ├── commands/                # Custom slash commands
    ├── rules/                   # Path-scoped rules
    ├── skills/                  # Auto-invoked capability packages
    ├── agents/                  # Specialized subagent personas
    └── hooks/                   # Event-driven automation
```

```
.claude/commands/
├── pr-review.md          → /project:pr-review
├── security-scan.md      → /project:security-scan
├── deploy-staging.md     → /project:deploy-staging
├── k8s-debug.md          → /project:k8s-debug
└── cost-estimate.md      → /project:cost-estimate
```

```
.claude/skills/
├── terraform-plan/
│   ├── SKILL.md             # Skill definition + trigger description
│   └── checklist.md         # Supporting file referenced via @
├── dockerfile-lint/
│   ├── SKILL.md
│   └── best-practices.md
└── incident-runbook/
    ├── SKILL.md
    └── runbook-template.md
```
```
.claude/agents/
├── security-auditor.md      # SAST/DAST, CVE analysis, IAM review
├── cost-optimizer.md        # FinOps, right-sizing, reservation advice
├── k8s-reviewer.md          # Manifest review, HPA/VPA, resource tuning
├── terraform-reviewer.md    # IaC review, drift detection
└── incident-responder.md    # Runbooks, triage, post-mortem drafts
```
```
.claude/rules/
├── terraform.md     # Apply when editing *.tf files
├── k8s.md           # Apply when editing manifests in /k8s/**
├── migrations.md    # Apply when editing DB migrations
└── ci.md            # Apply when editing .github/workflows/**
```