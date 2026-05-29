# Workflow — deployment quality gate

Event-driven Workflow that triggers SRG evaluation on deployment
and auto-remediates on failure verdict.

---

## Trigger

**Type:** Event — Deployment event
**Source:** Davis deployment event or custom BizEvent from CI/CD pipeline
**Filter:** entity.name matches frontend service

---

## Actions

### Step 1 — Trigger SRG evaluation
**Type:** HTTP request
**Method:** POST
**URL:** `https://{tenant}.live.dynatrace.com/api/v2/siteReliabilityGuardian/executions`
**Body:**
```json
{
  "guardianId": "{{ guardian_id }}",
  "startTime": "{{ deployment.startTime }}",
  "endTime": "{{ now() }}",
  "variables": {
    "service": "{{ deployment.entityName }}"
  }
}
```

### Step 2 — Wait for verdict
**Type:** Event listener
**Event:** `guardian.validation.finished`
**Timeout:** 10 minutes

### Step 3 — Branch on verdict

**Branch A — PASS:**
- Send Slack notification: deployment succeeded, SRG passed

**Branch B — WARN:**
- Send Slack notification: degradation detected, continuing to monitor
- Create Jira ticket: `[WARN] Deployment degradation — {service} {version}`

**Branch C — FAIL:**
- HTTP request: trigger rollback API
- HTTP request: create Jira incident ticket `[P1] Deployment failure — {service} {version}`
- Slack: page on-call channel
- Send DT problem notification

---

## Notes

- The async nature of SRG is critical: Step 1 returns an execution ID,
  not the verdict. Step 2 must listen for the verdict event separately.
- Candidates who design this synchronously (expecting the verdict from Step 1)
  have not used SRG in a real pipeline.
- Warning verdicts do not block deployment — they serve as a noise buffer
  between a healthy pass and a hard failure requiring rollback.
