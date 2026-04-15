# Incident Runbook Templates

Used by @SKILL.md. Contains stakeholder comms and post-mortem template.

---

## Stakeholder Communication Templates

### SEV-1 / SEV-2: Initial Declaration (within 10 minutes)

```
🔴 INCIDENT DECLARED — SEV-[1/2]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Time:     [HH:MM UTC]
Service:  [service name]
Impact:   [what users cannot do — be specific]
          Example: "Users cannot log in. All authentication requests returning 500."
Status:   INVESTIGATING
IC:       [your name]
Bridge:   [Zoom/Meet link]
Ticket:   [JIRA/Linear ticket URL]

Next update: [HH:MM UTC — 15 min from now for SEV-1, 30 min for SEV-2]
```

### Status Update (recurring while incident is open)

```
🟡 INCIDENT UPDATE — SEV-[X] — [HH:MM UTC]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Service:   [service name]
Status:    INVESTIGATING / MITIGATING / MONITORING
Impact:    [current state — is it improving, stable, or worsening?]
           Example: "Error rate down from 95% → 40%. Some users recovering."
Working theory: [your current hypothesis]
Actions:   [what was done since last update]
Next:      [what is happening right now]
ETA:       [estimate or "unknown — will update at HH:MM"]
```

### Mitigation Applied (not yet resolved)

```
🟡 MITIGATION APPLIED — SEV-[X] — [HH:MM UTC]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Service:  [service name]
Action:   [what was done — e.g., "Rolled back to v1.4.2"]
Impact:   [current state — e.g., "Error rate returning to baseline, monitoring for 15 min"]
Status:   MONITORING
Next:     Confirm stable, then resolve. Post-mortem will follow.
```

### Resolution

```
✅ INCIDENT RESOLVED — SEV-[X] — [HH:MM UTC]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Service:   [service name]
Duration:  [start time] → [end time] = [X hours Y minutes]
Impact:    [who was affected and for how long — quantify if possible]
Root cause: [1–2 sentences — plain language]
Fix:       [what stopped the bleeding]
Post-mortem: [name] to draft within 48h. Review scheduled: [date/time]
Ticket:    [link]
```

### Customer-Facing Status Page Update

(Keep technical jargon out — business-level language only)

**Investigating**:
> We are aware of an issue affecting [service/feature]. Our team is actively investigating.
> We will provide an update by [HH:MM UTC].

**Identified**:
> We have identified the cause of the issue affecting [service/feature] and are working on a fix.
> Some users may [experience X]. We will provide an update by [HH:MM UTC].

**Monitoring**:
> A fix has been applied. We are monitoring to confirm full recovery.
> [Service/Feature] should be returning to normal. We'll confirm resolution shortly.

**Resolved**:
> This incident has been resolved. [Service/Feature] is fully operational.
> We apologise for the disruption. A full post-mortem will be published within 72 hours.

---

## Post-Mortem Template

Copy this template into your post-mortem document. Complete within 48 hours of resolution.
Blameless — focus on systems, processes, and safeguards, not individuals.

```markdown
# Post-Mortem: [Service] [SEV Level] Incident — [YYYY-MM-DD]

**Status**: DRAFT | IN REVIEW | FINAL
**Severity**: SEV-[1/2/3]
**Author**: [name]
**Reviewers**: [names]
**Review Meeting**: [date/time/link]

---

## Summary

[2–4 sentences. What happened, what was the user-facing impact,
how was it detected, and how was it resolved. Write for a non-technical reader.]

---

## Impact

| Metric | Value |
|--------|-------|
| Start time | [HH:MM UTC YYYY-MM-DD] |
| End time | [HH:MM UTC YYYY-MM-DD] |
| Duration | [X hours Y minutes] |
| Users affected | [number or percentage] |
| Requests failed | [count or % error rate at peak] |
| Features affected | [list] |
| Revenue impact | [$X] (if applicable) |
| SLO impact | [X% of 30-day error budget consumed] |

---

## Timeline

All times UTC.

| Time | Event |
|------|-------|
| HH:MM | [Earliest symptom — when did it actually start, from logs?] |
| HH:MM | Alert fired: [alert name] |
| HH:MM | On-call [name] acknowledged |
| HH:MM | SEV-[X] declared, IC assigned |
| HH:MM | [Key investigation step] |
| HH:MM | [Hypothesis formed: "suspect recent deploy of v1.4.3"] |
| HH:MM | [Action taken: "rolled back to v1.4.2"] |
| HH:MM | Error rate began recovering |
| HH:MM | Error rate at baseline — incident resolved |
| HH:MM | Post-mortem initiated |

---

## Root Cause

[Technical explanation. Enough detail for an engineer unfamiliar with the system to
understand what went wrong and why it had the impact it did.

Example:
"A deploy of v1.4.3 introduced a connection pool configuration change that set
pool_size=100 per pod. With 8 pods running, this created 800 concurrent connection
slots against a CloudSQL instance with max_connections=200. Once traffic ramped up
post-deploy, all connection slots were exhausted within 3 minutes, causing all pods
to return 500 errors for any database-dependent request."]

---

## Contributing Factors

[What made this possible or made it worse? No blame — focus on system/process gaps.]

1. **[Factor]**: [Explanation]
   Example: "No canary deployment — v1.4.3 rolled out to 100% of pods simultaneously,
   giving no window to detect the connection spike before full impact."

2. **[Factor]**: [Explanation]
   Example: "CloudSQL connection count not in our alerting dashboard — we had no
   visibility into connection exhaustion until pods started returning 500s."

3. **[Factor]**: [Explanation]
   Example: "pool_size was a new configuration parameter added without documentation
   or a code review checklist item covering connection pool sizing."

---

## Detection

How was the incident detected?
- [ ] Automated alert (which alert, how quickly after incident start?)
- [ ] Customer report (how long after incident start?)
- [ ] Engineer noticed (during normal work?)
- [ ] External monitoring service

Time from incident start to detection: [X minutes]
Is this acceptable? If not, what alert should have fired earlier?

---

## What Went Well

[Genuine positives — what worked? This section matters for morale and for recognising
things we should preserve.]

1. [e.g., "On-call response time was under 3 minutes — excellent."]
2. [e.g., "ArgoCD rollback completed in 4 minutes — fast and reliable."]
3. [e.g., "Incident comms were clear and frequent — stakeholders were well-informed."]

---

## What Went Poorly

[Honest assessment of what slowed resolution or made impact worse.]

1. [e.g., "Alert threshold was set too high (5%) — fired 18 minutes after impact started."]
2. [e.g., "Rollback procedure was not documented — IC had to look up ArgoCD commands."]
3. [e.g., "No runbook for DB connection exhaustion — diagnosis took 30 extra minutes."]

---

## Action Items

Concrete, assigned, time-bound. No vague "improve monitoring" without specifics.

| Priority | Action | Owner | Due Date | Ticket |
|----------|--------|-------|----------|--------|
| P1 | Add canary stage to prod deploy pipeline (5% → 25% → 100% with 5min gates) | [name] | [date] | TICKET-XXX |
| P1 | Add CloudSQL connection count alert: fire at 80% of max_connections | [name] | [date] | TICKET-XXX |
| P1 | Add connection pool sizing to CLAUDE.md and PR review checklist | [name] | [date] | TICKET-XXX |
| P2 | Document ArgoCD rollback procedure in incident runbook | [name] | [date] | TICKET-XXX |
| P2 | Add pool_size validation test: assert pool_size * expected_replicas < db.max_connections | [name] | [date] | TICKET-XXX |
| P3 | Create CloudSQL connection count dashboard panel | [name] | [date] | TICKET-XXX |

---

## Lessons Learned

[3–5 key takeaways for the team. What does this incident teach us about
our system, our processes, or our tooling?]

1. [e.g., "Configuration changes that affect system-wide resource consumption (connections,
   threads, memory) need explicit capacity analysis before deploy."]

2. [e.g., "Canary deployments are non-optional for services with stateful dependencies.
   We were lucky to catch this with a rollback — data corruption scenarios would not
   be recoverable this way."]

3. [e.g., "We need runbooks for every known failure mode BEFORE incidents happen,
   not after. Time-to-runbook during an incident is wasted time."]

---

## Appendix

### Relevant Log Queries

```bash
# Error spike during incident window
gcloud logging read \
  'resource.type="k8s_container"
   AND severity=ERROR
   AND timestamp >= "YYYY-MM-DDTHH:MM:00Z"
   AND timestamp <= "YYYY-MM-DDTHH:MM:00Z"' \
  --limit=200 --format=json

# Connection errors specifically
gcloud logging read \
  'resource.type="k8s_container"
   AND jsonPayload.message=~"too many connections|connection refused|pool"' \
  --freshness=3h --limit=100
```

### Metrics Snapshots

[Link to Cloud Monitoring dashboard screenshot showing the incident period]
[Link to any relevant BigQuery queries on billing/error data]

### Related Incidents

[Link to any previous similar incidents for pattern analysis]
```

---

## Runbook Review Cadence

After each incident, review this runbook:
- Did RB-00X for this failure pattern exist? If not: create it.
- Was the existing runbook accurate? If not: update it.
- Did the runbook get the team to resolution faster? If not: improve it.

Runbooks that are never used rot. Schedule a 30-min quarterly runbook review with the team.
