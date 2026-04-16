## Directory Tree

```
your-project/
│
├── CLAUDE.md                          # (1) Master project context file — edit this first
├── .mcp.json                          # (2) MCP server integrations (GitHub, Postgres, GCS)
│
└── .claude/
    ├── settings.json                  # (3) Team permissions, deny list, auto-hooks
    ├── settings.local.json            # (4) Personal overrides — gitignored
    │
    ├── commands/                      # (5) Custom slash commands → /project:NAME
    │   ├── pr-review.md               #     /project:pr-review
    │   ├── security-scan.md           #     /project:security-scan
    │   ├── deploy-staging.md          #     /project:deploy-staging
    │   ├── k8s-debug.md               #     /project:k8s-debug [namespace]
    │   └── cost-estimate.md           #     /project:cost-estimate
    │
    ├── rules/                         # (6) Path-scoped rules — auto-applied by file type
    │   ├── k8s.md                     #     Applied when editing k8s/**/*.yaml
    │   ├── terraform.md               #     Applied when editing infra/**/*.tf
    │   ├── migrations.md              #     Applied when editing migrations/versions/**
    │   └── ci.md                      #     Applied when editing .github/workflows/**
    │
    ├── agents/                        # (7) Specialist subagents — isolated context windows
    │   ├── security-auditor.md        #     Deep security review with CWE refs
    │   ├── cost-optimizer.md          #     GCP FinOps with quantified savings
    │   ├── k8s-reviewer.md            #     Manifest review + YAML patches
    │   ├── terraform-reviewer.md      #     IaC review, destruction risk, GCP patterns
    │   └── incident-responder.md      #     Live triage, comms, post-mortem coordination
    │
    └── skills/                        # (8) Auto-invoked multi-file capability packages
        ├── terraform-plan/
        │   ├── SKILL.md               #     Validate → plan → security → cost → checklist
        │   └── checklist.md           #     Pre-apply safety gates (referenced via @)
        ├── dockerfile-lint/
        │   ├── SKILL.md               #     hadolint + manual security review flow
        │   └── best-practices.md      #     Full rule set: CRITICAL → LOW with examples
        └── incident-runbook/
            ├── SKILL.md               #     10 runbooks (RB-001–010) with kubectl commands
            └── runbook-template.md    #     Stakeholder comms + full post-mortem template
```

---

## File Reference

### (1) `CLAUDE.md` — Project Context

The single most important file. Loaded into Claude's system prompt at the start of every session.

**What to put in it:**
- Stack overview (language, framework, database, infra toolchain)
- Build / test / lint / deploy commands
- Architecture decisions and non-obvious constraints
- Coding conventions Claude cannot infer from reading the code
- Team workflow (branch naming, PR process, deploy gates)

**What NOT to put in it:**
- Anything already enforced by a linter or formatter config
- Long documentation you can link to instead
- Generic advice — be specific to your project

> Keep it under 200 lines. Beyond that, instruction adherence drops.

---

### (2) `.mcp.json` — MCP Server Integrations

Connects Claude Code to external tools and data sources.

Bundled servers:

| Server | Purpose |
|--------|---------|
| `github` | Read PRs, issues, repos — needs `GITHUB_TOKEN` env var |
| `postgres` | Query your database directly — needs `DATABASE_URL` env var |
| `gcs` | Read/list GCS buckets — needs `GOOGLE_CLOUD_PROJECT` env var |
| `filesystem` | Read local project files outside the working directory |
| `fetch` | Fetch URLs (docs, APIs, status pages) |

Add or remove servers to match your stack. All credentials are injected via environment variables — never hardcode them here.

---

### (3) `settings.json` — Permissions and Hooks

Two sections:

**`permissions`** — what Claude Code can and cannot run:

```json
"allow": ["Bash(kubectl get:*)", "Bash(terraform plan:*)", ...],
"deny":  ["Bash(terraform apply:*)", "Bash(kubectl delete:*)", ...]
```

The deny list prevents destructive operations without explicit human confirmation. Adjust to match your team's risk tolerance.

**`hooks`** — commands that run automatically on file save:

| Trigger | Action |
|---------|--------|
| Edit any `*.py` | `ruff format` + `ruff check --fix` + `mypy` |
| Edit any `*.tf` | `terraform fmt` |
| Edit any `k8s/**/*.yaml` | `kubectl apply --dry-run=client` |
| Write any `Dockerfile*` | `hadolint` |

Hooks give instant feedback at edit time — catching issues before they reach CI.

---

### (4) `settings.local.json` — Personal Overrides

Gitignored. Each engineer maintains their own copy for local overrides:
- Unlocking operations blocked by team settings (e.g., allow `terraform apply` for your dev environment)
- Local environment variables (`KUBECONFIG`, `GOOGLE_CLOUD_PROJECT`)
- Personal model preference

---

### (5) `commands/` — Custom Slash Commands

Every `.md` file here becomes a `/project:NAME` slash command inside Claude Code.

| Command | What It Does |
|---------|-------------|
| `/project:pr-review` | Reviews `git diff main...HEAD` — outputs blockers, warnings, suggestions |
| `/project:security-scan` | Full audit: secrets, CVEs, IAM, container, K8s posture |
| `/project:deploy-staging [service]` | Guided deploy: pre-flight checks → build → push → ArgoCD sync → verify |
| `/project:k8s-debug [namespace]` | Diagnoses failing pods: OOMKilled, CrashLoop, ImagePull, probe failures |
| `/project:cost-estimate` | Estimates GCP cost delta from current Terraform + K8s diff |

Commands use `!` backtick syntax to inject real shell output into the prompt before Claude sees it — so Claude is reviewing actual live data, not guessing.

**Adding your own command:**
```bash
# Creates /project:db-migrate
cat > .claude/commands/db-migrate.md << 'EOF'
---
description: Check migration status and generate upgrade plan
---
## Current Migration Head
!`uv run alembic current`

## Pending Migrations
!`uv run alembic history --indicate-current | head -20`

Review the pending migrations and advise on: ordering, risk level, whether downgrade() is complete.
EOF
```

---

### (6) `rules/` — Path-Scoped Rules

Rules are auto-applied when Claude edits files matching their domain. They contain the actual
patterns, YAML templates, and HCL blocks to use — not vague advice.

| File | Applies When Editing | Enforces |
|------|---------------------|---------|
| `k8s.md` | `k8s/**/*.yaml` | SecurityContext, resource limits, probes, labels, NetworkPolicy, ESO for secrets |
| `terraform.md` | `infra/**/*.tf` | `prevent_destroy`, least-priv IAM, Workload Identity, required labels, budget alerts |
| `migrations.md` | `migrations/versions/**` | Reversible `downgrade()`, CONCURRENT indexes, unsafe change playbook, staging gate |
| `ci.md` | `.github/workflows/**` | WIF over SA keys, SHA-pinned actions, pipeline stage order, Trivy scan, coverage gate |

Rules are the difference between Claude generating compliant code on the first try versus you
correcting it in review.

---

### (7) `agents/` — Specialist Subagents

Each agent runs in its own isolated context window. It focuses on one domain, does deep work,
and returns a structured report without polluting your main session.

| Agent | Model | Specialisation |
|-------|-------|----------------|
| `security-auditor` | Opus | CRITICAL→LOW findings with CWE refs, exact code fixes, merge recommendation |
| `cost-optimizer` | Sonnet | Quantified savings ($X/mo), GCP pricing anchors, CUD opportunities, risk flags |
| `k8s-reviewer` | Sonnet | Manifest checklist, GKE Autopilot compliance, ready-to-apply YAML patches |
| `terraform-reviewer` | Sonnet | Destruction risk first, IAM review, GCP resource patterns, pre-apply checklist |
| `incident-responder` | Opus | Triage framework, 7 common patterns, stakeholder comms, post-mortem coordination |

Opus is used for security and incident response where depth and adversarial thinking matter.
Sonnet is used for review agents where speed and cost efficiency are more important.

**Invoking an agent:**
```
Review this PR for security issues.
```
Claude will automatically route to `security-auditor` based on its description trigger.
Or invoke explicitly: `Use the security-auditor agent to review infra/main.tf`

---

### (8) `skills/` — Auto-Invoked Skill Packages

Skills are like commands but smarter: they trigger automatically based on context, and they
bundle supporting reference files alongside the skill definition via `@file` references.

| Skill | Auto-Triggers On | Supporting File |
|-------|-----------------|----------------|
| `terraform-plan` | "terraform", "tf plan", "infra change", `*.tf` in context | `checklist.md` — pre-apply safety gates |
| `dockerfile-lint` | "docker build", "Dockerfile", "base image", "container image" | `best-practices.md` — CRITICAL→LOW rule set with full examples |
| `incident-runbook` | "incident", "outage", "prod is down", "SEV-", "on-call" | `runbook-template.md` — comms templates + post-mortem template |

The key difference from commands: **skills are packages**. The `@checklist.md` reference in
`terraform-plan/SKILL.md` pulls in the full checklist document when the skill runs.
Commands are single files. Skills are directories.

---

## What to Commit vs Gitignore

```
# Commit — team config
CLAUDE.md                      ✓
.mcp.json                      ✓
.claude/settings.json          ✓
.claude/commands/              ✓
.claude/rules/                 ✓
.claude/agents/                ✓
.claude/skills/                ✓

# Gitignore — personal config
.claude/settings.local.json    ✗
.claude/.session*              ✗
.claude/session-*              ✗
```

The entire point of committing `.claude/` is that every engineer on the team gets the same
AI behaviour. Treat it like any other team configuration — review changes in PRs.

---

## Customisation Guide

### Adapting to Your Stack

This template is written for Python / FastAPI / GKE / GCP. To adapt it:

1. **`CLAUDE.md`** — replace stack, commands, and gotchas with your own
2. **`rules/`** — update language-specific rules (swap `ruff`/`mypy` for `eslint`/`tsc` etc.)
3. **`settings.json` hooks** — update file matchers and commands for your toolchain
4. **`commands/`** — update `kubectl`/`gcloud`/`argocd` references to your orchestration tools
5. **`.mcp.json`** — add/remove MCP servers to match your data sources

### Adding a New Command

1. Create `.claude/commands/your-command.md`
2. Add frontmatter: `description:` (shown in `/help`)
3. Use `!` backtick syntax to inject shell output
4. Use `$ARGUMENTS` to accept parameters: `/project:your-command somearg`

### Adding a New Agent

1. Create `.claude/agents/your-agent.md`
2. Add frontmatter: `description:` (trigger text), `capabilities:`, `model:`
3. Write a focused system prompt for one domain
4. Keep it under 150 lines — agents need focused context, not encyclopedias

### Adding a New Skill

1. Create `.claude/skills/your-skill/SKILL.md` with frontmatter `name:` and `description:`
2. Add supporting files in the same directory
3. Reference them inside SKILL.md with `@./supporting-file.md`
4. The `description:` field controls auto-invocation — be specific about trigger phrases

---
```
# frontmatter examples:
 
arguments:
  - name: output_path
    hint: "Where should the file be saved? Provide a full path like /home/claude/report.md"
    required: true

  - name: content_topic
    hint: "What should the file contain? Describe the content or paste it directly."
    required: true

  - name: file_format
    hint: "What format? Options: markdown, plain text, JSON, YAML"
    required: false
    default: "markdown"

allowed-tools:
  - create_file   # ✅ only write new files
  - view          # ✅ only read/check directories
  # bash_tool     ❌ blocked → no shell commands (rm, mv, etc.)
  # str_replace   ❌ blocked → no overwriting existing files****
```
## Related Resources

- [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code/overview)
- [Claude Code .claude Directory Explorer](https://code.claude.com/docs/en/claude-directory)
- [MCP Server Registry](https://github.com/modelcontextprotocol/servers)
- [Anthropic Prompt Engineering Guide](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)
