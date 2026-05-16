# Agent Task Patterns

Typical task patterns an agent will encounter when assigned to Caliptic issues. Each pattern lists **input signals**, **steps**, **output shape**, and **common mistakes**.

## Pattern 1: Incident Triage

**Signal:** `issue_type=incident`, `status=todo`, assigned to you.

**Steps:**
1. Read the description + form_data — extract symptoms.
2. `GET /api/issues?issue_type=incident&status=in_progress&priority=urgent` — any similar active cases?
3. KB search: `GET /api/workspaces/{wsId}/kb/articles/search?q=<keywords>` — is there a known workaround?
4. CMDB lookup: find the affected service via `cmdb_config_item.name` → `GET /api/workspaces/{wsId}/cmdb/impact-analysis/{ciId}` → what else is impacted?
5. Status → `in_progress` + comment:
   - "Two open incidents share the same signature (#INC-… and #INC-…) — this looks like a problem candidate."
   - "KB article #KB-42 ('Restart payment pod') offers a workaround."
   - "Impact: payment-service + checkout-flow are dependent; 3 downstream services."

**Output shape:**
```
## Triage summary
- **Affected CI**: payment-service-prod
- **Blast radius**: 3 downstream services (checkout-flow, billing, reporting)
- **Similar active incidents**: 2 (potential problem)
- **Workaround**: KB-42 (pod restart) — worth trying
- **Next step**: Apply the workaround; if it fails, escalate to SEV-1 + page the on-call DBA
```

**Common mistake:** Applying the workaround AND writing the RCA in the incident. The RCA belongs on a `problem`-typed issue, not the incident.

---

## Pattern 2: Change Implementation

**Signal:** `issue_type=change`, `status=approved`, CAB review complete, assigned to you.

**Pre-flight checklist (don't implement without ticking these):**
- [ ] Is `change_type` correct? (`standard`/`normal`/`emergency`)
- [ ] Is `rollback_plan` non-empty? (if not, fill it!)
- [ ] Is `test_plan` non-empty?
- [ ] Is there an active freeze window? `GET /api/workspaces/{wsId}/freeze-windows` — any that overlap with right now?
- [ ] Is CAB approval count ≥ minimum? `GET /api/issues/{id}/cab-reviews`

**Steps:**
1. Status `approved` → `in_progress`. The backend performs a freeze check; if you get a 409 log the active freeze and leave a comment for the humans.
2. Implementation. `actual_start_at` may be auto-set server-side (check the handler).
3. Post test output as a comment.
4. Success:
   - Status → `done`
   - `actual_end_at` set (if the handler doesn't set it, PUT it manually)
   - Comment: implementation summary + verification method used
5. Failure:
   - Status → `cancelled` (if no rollback needed) or leave at `in_progress` (rollback in flight)
   - Add rollback notes as a comment

**Common mistake:** Writing "TBD" in a rollback plan for a change that genuinely needs rollback. The plan must contain **concrete commands / steps**.

---

## Pattern 3: Release Build → Deploy → PIR

**Signal:** `issue_type=release`, you're acting as the release coordinator agent.

**Steps:**
1. Scope validation:
   ```
   GET /api/issues/{releaseId}/release-changes
   ```
   If the list is empty, ask the human: "Which changes does this release contain?"

2. Pre-deploy:
   ```
   PATCH /api/issues/{releaseId}/release-phase
   { "release_build_status": "in_progress" }
   ```
   Trigger the CI/CD pipeline. Outcome:
   ```
   PATCH .../release-phase { "release_build_status": "passed" }
   ```

3. Deploy:
   ```
   PATCH .../release-phase { "release_deploy_status": "deploying" }
   ```
   Run the deploy script. Watch the health checks.

4. Deploy result:
   - Success: `release_deploy_status: deployed`
   - Failure: `release_deploy_status: failed` or `rolled_back`

5. PIR (mandatory):
   ```
   POST /api/issues/{releaseId}/release-pirs
   {
     "outcome": "success",
     "success_rating": 5,
     "what_went_well": "...",
     "what_went_wrong": "...",
     "lessons_learned": "...",
     "open_improvement": true
   }
   ```

**Common mistake:** Marking the release issue `done` immediately after a successful deploy, skipping the PIR. In ITIL4 the release loop is never complete without a PIR.

---

## Pattern 4: Problem RCA

**Signal:** `issue_type=problem`, `status=in_progress`, you are the RCA owner.

**Steps:**
1. Collect related incidents:
   ```
   GET /api/issues/{problemId}/related-incidents
   ```
2. Find patterns: are they in the same time window? Same service? Same user impact?
3. CMDB-related CIs:
   ```
   GET /api/workspaces/{wsId}/cmdb/cis?filter=service
   ```
4. Log / metric analysis (if the workspace has a monitoring integration).
5. Update the RCA:
   ```
   PATCH /api/issues/{problemId}/rca
   {
     "root_cause": "<concrete technical explanation>",
     "workaround": "<urgent fix, temporary>"
   }
   ```
6. Open a CSI improvement for the permanent fix:
   ```
   POST /api/issues/{problemId}/spawn-improvement
   {
     "title": "Implement: <root_cause solution>",
     "category": "tool|process|...",
     "expected_benefit": "Eliminate workaround"
   }
   ```

**Output shape:**
```
## RCA — Payment service intermittent 500s

### Root Cause
DB connection pool exhaustion. App max_connections=20, observed peak=50.

### Evidence
- 3 incidents in the last 7 days share the same signature (#INC-201, #INC-205, #INC-211)
- CloudWatch metrics: pool_utilization 95-100% at peak hours
- Application log: "no available connections" exception (psycopg2.OperationalError)

### Workaround (short-term)
- Pod restart every 6 hours (cron)

### Permanent fix
- Improvement #CSI-0042 opened: raise connection pool to 100 + add Hystrix breaker
```

**Common mistake:** Writing the RCA as a comment. The `issue.root_cause` and `issue.workaround` fields exist for a reason — PATCH /rca writes to them; backend audit + reporting depend on those columns.

---

## Pattern 5: Improvement Lifecycle

**Signal:** `improvement_proposal`, assigned to you as owner.

**Steps:**
1. **Plan** (PDCA):
   ```
   PATCH /api/workspaces/{wsId}/improvements/{id}
   {
     "plan_notes": "1. Bump connection pool to 100 (config map) 2. Add Hystrix breaker library 3. Load test 4. Production rollout",
     "target_date": "2026-06-15"
   }
   ```

2. **Status → reviewing → approved** (by an authoriser):
   ```
   PATCH .../improvements/{id} { "status": "approved" }
   ```

3. **Implement** (spawn a change):
   ```
   POST /api/workspaces/{wsId}/improvements/{id}/spawn-change
   { "title": "DB pool 100 + circuit breaker", "change_type": "normal", "risk_level": "medium" }
   ```
   → New issue with `issue.spawned_from_improvement_id` set. Status → `implementing`.

4. **Check** (after the change lands):
   ```
   PATCH .../improvements/{id}
   {
     "do_notes": "Change #CHG-PROD-0042 was implemented",
     "check_notes": "Production monitored for a week; pool peak ~70%, restart cron removed",
     "actual_benefit": "Workaround removed; p99 latency -30%"
   }
   ```

5. **Act** + **done**:
   ```
   PATCH .../improvements/{id}
   {
     "act_notes": "Pool monitoring metric added; runbook updated",
     "status": "done"
   }
   ```

**Common mistake:** Leaving PDCA fields blank. If `actual_benefit` hasn't been measured, the improvement isn't done — the ITIL4 audit will catch this.

---

## Pattern 6: Contract Analysis

**Signal:** A task triggered by `POST /contracts/{id}/analyze`, assigned to you.

**Steps:**
1. Pull attachment URLs from the description.
2. Extract PDF/DOCX content (the skill may need an external tool — `anthropic-skills:pdf` or similar can help).
3. **Standard checklist:**
   - Parties (party_name, vendor address, tax id)
   - Start / end dates
   - Total value + currency
   - Auto-renewal clause + notice period (`notice_days`)
   - Termination clauses (notice period, breach trigger)
   - Confidentiality / NDA terms
   - SLA commitments (uptime %, response time, penalty)
   - GDPR / data-protection compliance (if applicable)
4. **Risk level:**
   - Low: standard NDA, clear termination, reasonable SLA
   - Medium: auto-renewal + long notice (lock-in), ambiguous termination terms
   - High: open-ended, heavy SLA penalty, vendor has unilateral termination right

**Output shape** (as a timeline comment on the issue):
```
## Acme MSA — Analysis

### Parties
- **Customer**: Caliptic Software Inc.
- **Vendor**: Acme Corp (Tax ID: 1234567890)

### Key dates
- Start: 2026-01-01
- End: **2027-12-31** (24 months)
- Notice period: 60 days
- Auto-renew: ✅ YES (extends 12 months unless cancelled 90 days prior)

### Financial
- Annual fee: 50,000 USD
- Total: 100,000 USD
- Early termination penalty: 50% of remaining term

### SLA
- Uptime: 99.9% (monthly)
- Penalty: 1% credit per SLA breach

### Risk level: **MEDIUM**
- Auto-renew + 60-day notice is tight (control window short)
- Heavy early-termination penalty (50% of remaining)
- + Clear SLA + reasonable penalty (positive)

### Recommendations
- Renewal alarm: 2027-09-30 (90 days before)
- Termination risk: add to the quarterly contract review cycle
```

**Common mistake:** Declaring a risk level "low" without justification. Each level must be **backed by evidence**.

---

## Universal Principles

1. **Audit trail matters** — every status change is paired with a comment + reason.
2. **Cross-link with intent** — issues should be linked to each other (related, blocks, parent) when there's actual structural meaning, not arbitrarily.
3. **Workspace isolation** — `workspace_id` is on every query; cross-workspace ID guessing returns 404 from the backend, but the agent shouldn't try in the first place.
4. **Check module gates** — if a calipticore module is disabled (say, catalog is off), an agent can still create a `service_request` issue but it won't be visible in the UI; this can confuse users, so check first.
5. **Express your own state honestly** — "I couldn't because X" beats "failed". Detail > vagueness.
