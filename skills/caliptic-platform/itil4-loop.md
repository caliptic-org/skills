# ITIL4 ITSM Loop — Concrete Flow Examples

Caliptic's ITIL4 service value chain is wired together with cross-module
junction tables. This guide walks four typical scenarios step by step.

## Scenario 1: Production Incident → Permanent Fix → Release

**Trigger:** Monitoring alert fires; a user reports "the payment service isn't responding."

### Step 1 — Open an incident
```http
POST /api/issues
{
  "title": "Payment service 500 errors",
  "issue_type": "incident",
  "priority": "urgent",
  "urgency": "high",
  "itil_impact": "extensive",
  "assignee_type": "agent",
  "assignee_id": "<on-call-agent-id>"
}
```
If `priority_matrix` is enabled the priority is computed automatically.

### Step 2 — Apply a workaround + open a problem
After closing the incident, for the RCA:
```http
POST /api/issues
{
  "title": "RCA: Payment service intermittent 500s",
  "issue_type": "problem",
  "priority": "high"
}
```
Then link the incident to the problem:
```http
POST /api/issues/{incidentId}/link-problem
{ "problem_id": "<problemId>" }
```

### Step 3 — Complete the RCA, open a CSI improvement for the permanent fix
```http
PATCH /api/issues/{problemId}/rca
{
  "root_cause": "DB connection pool exhausted under load — max=20, peak demand=50",
  "workaround": "Restart payment-service pods every 6h via cron"
}

POST /api/issues/{problemId}/spawn-improvement
{
  "title": "Raise payment-service connection pool + add circuit breaker",
  "category": "tool",
  "expected_benefit": "Eliminate the restart-every-6h workaround"
}
```
→ An `improvement_proposal` record is created with `improvement_source.source_type='problem_rca'`.

### Step 4 — Improvement approved → spawn a change
The team approves:
```http
PATCH /api/workspaces/{wsId}/improvements/{improvementId}
{ "status": "approved" }

POST /api/workspaces/{wsId}/improvements/{improvementId}/spawn-change
{
  "title": "Raise DB pool to 100 + add Hystrix breaker",
  "change_type": "normal",
  "risk_level": "medium",
  "impact_level": "low"
}
```
→ A new issue (`type=change`) is created with `spawned_from_improvement_id` set.

### Step 5 — CAB review + implement
```http
POST /api/issues/{changeId}/cab-reviews { "reviewer_id": "<db-team-lead>" }
POST /api/issues/{changeId}/cab-reviews/decision { "decision": "approved", "notes": "Pool size validated in staging" }

PUT /api/issues/{changeId}
{ "rollback_plan": "kubectl rollout undo + revert config map", "status": "in_progress" }
```
The backend performs a freeze-window check — if one is active you'll get a 409 with a hint to escalate `change_type=emergency`.

### Step 6 — Add to release scope
```http
POST /api/issues
{ "title": "Payment service v2.4.0", "issue_type": "release", "release_environment": "production" }

POST /api/issues/{releaseId}/release-changes { "change_id": "<changeId>" }

PATCH /api/issues/{releaseId}/release-phase
{
  "release_build_status": "passed",
  "release_target_date": "2026-05-20",
  "release_notes": "Raise DB pool to 100 + circuit breaker",
  "rollback_plan": "kubectl rollout undo deployment/payment-service"
}
```

### Step 7 — Deploy + PIR
```http
PATCH /api/issues/{releaseId}/release-phase
{ "release_deploy_status": "deployed" }

POST /api/issues/{releaseId}/release-pirs
{
  "outcome": "success",
  "success_rating": 5,
  "what_went_well": "4-minute deploy, zero downtime, pool stable",
  "lessons_learned": "We should add pool-size monitoring metrics",
  "open_improvement": true
}
```
→ A new improvement candidate is opened (`source_type='pir'`) — the loop closes and continuous improvement continues.

---

## Scenario 2: Emergency Change (despite an active Freeze Window)

There's a security vulnerability in production. We must push through even though a freeze window is active.

```http
# 1. Create the change
POST /api/issues
{
  "title": "EMERGENCY: SQL injection patch for /api/search",
  "issue_type": "change",
  "change_type": "emergency",
  "risk_level": "low",
  "rollback_plan": "git revert + redeploy"
}

# 2. Implement directly — the freeze validator bypasses emergency types
PUT /api/issues/{id}
{ "status": "in_progress", "change_type": "emergency" }
```

**Agent behaviour:** Only a human can declare a change `emergency`. An agent should not set `change_type='emergency'` unilaterally — instead, try `change_type='normal'` first and, on the resulting 409, leave a comment: "Freeze window is active; emergency override requires human approval."

---

## Scenario 3: Release Rollback + Second PIR

Deploy failed:
```http
PATCH /api/issues/{releaseId}/release-phase
{ "release_deploy_status": "rolled_back" }

POST /api/issues/{releaseId}/release-pirs
{
  "outcome": "rollback",
  "success_rating": 2,
  "what_went_wrong": "The DB migration script held a table-level lock for 12 minutes; traffic had to be drained",
  "lessons_learned": "Adopt an online schema migration + maintenance-window protocol",
  "open_improvement": true  // default true; on non-success outcomes auto-fires
}
```
→ A "Adopt an online schema migration" improvement candidate lands in the CSI register.

After the hotfix + redeploy a **second PIR** record can be added (`release_pir` is an n:1, not a composite-key relation):
```http
POST /api/issues/{releaseId}/release-pirs
{ "outcome": "success", "success_rating": 4, "lessons_learned": "Online migration succeeded" }
```

---

## Scenario 4: Contract → Asset → SLA Chain

User: "What's the SLA on our Acme MSA, and which assets does it cover?"

```http
# 1. Find the contract
GET /api/workspaces/{wsId}/contracts
→ filter party_name="Acme"

# 2. Upload an attachment (MSA PDF)
POST /api/workspaces/{wsId}/contracts/{id}/attachments
(multipart, file: msa-2026.pdf)

# 3. Trigger an agent analysis
POST /api/workspaces/{wsId}/contracts/{id}/analyze
{ "agent_id": "<contract-analyzer-agent>", "prompt": "Extract SLA durations, termination clauses, and covered services" }
→ A new issue is opened, the agent task runs, results land in the timeline

# 4. Tie assets to the contract (CMDB side)
# Currently asset.contract_id is set directly via PATCH; a fully-featured
# handler is on the roadmap. For now this can be done with a manual SQL
# update or the asset update endpoint.
```

**Agent behaviour:** PDF analysis should use **bullet points**, **bold** critical dates, and finish with a **risk level** (low/medium/high) + rationale.

---

## Tracing Cross-Module Relationships

To start from an improvement and walk the whole chain:
```http
GET /api/workspaces/{wsId}/improvements/{id}/evidence
```

Response shape:
```json
{
  "sources": [
    { "source_type": "problem_rca", "source_ref_id": "<problemId>", "note": "Spawned from problem" }
  ],
  "spawned_issues": [
    { "id": "<changeId>", "number": 42, "title": "Implement: ...", "issue_type": "change", "change_number": "CHG-WS-0042", "status": "in_progress" }
  ]
}
```

A symmetrical "which improvements did this PIR spawn?" lookup is partially supported:
```
GET /api/improvement-sources?source_type=pir&source_ref_id=<pirId>
```
(The handler isn't yet exposed — the `ListImprovementsBySource` query exists in the backend; the route will be added as the frontend needs it.)

---

## Anti-Patterns

❌ **Don't skip PIR after implementing a change + submitting a release.** The loop stays half-open and the CSI register goes unfed.

❌ **Don't mark an improvement `done` without a spawned change.** In ITIL4, "implementation" must happen through a real change record; bypassing this breaks the audit trail.

❌ **Don't close a problem without at least one linked incident.** In ITIL4 a problem always references one or more incidents.

❌ **Don't implement a high-risk change without CAB approval.** Don't push status to `in_progress` unless `cab_review` has a `decision='approved'` row.

❌ **Don't update a contracted asset without checking the contract binding.** `asset.contract_id` exists (migration 106); future versions will auto-link this chain to the SLA system.
