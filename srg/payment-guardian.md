# SRG — Payment service quality gate

Site Reliability Guardian configuration for the frontend payment service.
Evaluated automatically on each deployment event via Workflow trigger.

---

## Guardian: Payment services — deployment gate

**Entity scope:** `frontend` service
**Baseline window:** 7 days prior to deployment
**Evaluation window:** 60 minutes post-deployment
**Verdict on fail:** Block pipeline, trigger rollback Workflow

---

## Objectives

### 1. Error rate
- **DQL:**
  ```
  fetch spans, from: now()-1h
  | filter service.name == "frontend"
  | summarize error_rate = round(toDouble(countIf(isError == true)) / count() * 100, decimals: 2)
  ```
- **Comparison:** Absolute
- **Warning:** > 1%
- **Fail:** > 2%
- **Rationale:** Hard ceiling — even a relative increase from 0.1% to 0.15% is noise, but crossing 2% absolute indicates a real regression.

### 2. P99 latency
- **DQL:**
  ```
  fetch spans, from: now()-1h
  | filter service.name == "frontend"
  | summarize p99_ns = percentile(duration, 99)
  | fieldsAdd p99_ms = round(toDouble(p99_ns) / 1000000, decimals: 2)
  | fields p99_ms
  ```
- **Comparison:** Relative
- **Warning:** > 10% above baseline
- **Fail:** > 20% above baseline
- **Rationale:** Relative threshold accounts for traffic seasonality. A 50ms increase at midnight vs peak load has different significance.

### 3. Database span p99
- **DQL:**
  ```
  fetch spans, from: now()-1h
  | filter isNotNull(db.statement) or isNotNull(db.system)
  | summarize p99_ms = round(toDouble(percentile(duration, 99)) / 1000000, decimals: 3), by: { service.name }
  | filter service.name == "frontend"
  ```
- **Comparison:** Relative
- **Warning:** > 20% above baseline
- **Fail:** > 30% above baseline
- **Rationale:** DB span latency spiking is the earliest signal of connection pool exhaustion — this objective would have caught the Oracle incident before customer impact.

---

## Integration with Workflow

```
Trigger: Deployment event (Davis or BizEvent from CI/CD)
  → Action 1: POST /api/v2/siteReliabilityGuardian/executions
  → Action 2: Listen for guardian.validation.finished event
  → Branch:
      PASS  → Slack notification: deployment succeeded
      WARN  → Slack notification: degradation detected, monitoring
      FAIL  → Trigger rollback API
            → Create Jira incident ticket
            → Page on-call via Slack
```

---

## Real-world context

This SRG configuration was designed based on a production incident where
Oracle DB connection pool exhaustion caused payment card transaction failures.
The DB span p99 objective (objective 3 above) directly addresses this failure mode —
a spike in database span latency would trigger a warning before the pool fully
exhausts and customers begin seeing failures.
