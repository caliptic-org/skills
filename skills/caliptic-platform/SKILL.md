---
name: caliptic-platform
description: Comprehensive operating guide for the Caliptic ITSM platform. Attach this skill to an agent and it gains end-to-end knowledge of Caliptic's domain model (issues, changes, releases, problems, contracts, CMDB, sprints), the ITIL4 service value chain (incident → problem → change → release → PIR → improvement), the junction relationships that wire those modules together, the REST API surface, and the behavioural rules expected of agents acting as first-class teammates. Designed to bind to the workspace's default supervisor — a shared knowledge base every default agent inherits. Equips the agent to triage incidents, write RCAs, raise change requests, manage release scope, submit PIRs, spawn CSI improvements, analyse contracts, and operate inside Caliptic's API and architectural conventions.
---

# Caliptic ITSM Platform — Agent Operating Guide

This skill gives an AI agent everything it needs to act as a **first-class teammate** inside Caliptic. Agents are assigned to issues, leave comments, change status, and create related records — all while honouring **ITIL4 discipline** and Caliptic's architectural rules.

## 1. What is Caliptic?

**An AI-native ITSM platform** — Linear/Jira-shaped but with AI agents as first-class citizens. Built for ITIL4-aligned operations, multi-tenant SaaS + on-premise. Every query is filtered by `workspace_id`; the role model is member/admin/owner.

**Core concepts:**
- **Workspace** — tenant scope, the boundary every piece of data revolves around (bound via `X-Workspace-Slug` or `X-Workspace-ID` header).
- **Issue** — the atomic unit of work; its `issue_type` decides shape: `task`, `incident`, `problem`, `change`, `service_request`, `release`. One polymorphic table.
- **Agent** — an AI teammate. Can be assignee just like a human member; can comment, change status, react. `assignee_type='agent'` + `assignee_id=<agent.id>`.
- **Project** — issue grouping; agent access can be constrained per project (`project_access`).
- **Sprint** — time-boxed iteration inside a project; distinct from an ITIL4 release (sprint = team cadence, release = production deploy).

## 2. The ITIL4 Service Value Loop

Caliptic's defining feature: modules are physically wired together (via migration 105's junction tables):

```
[Incident] ──┐
[Problem]  ──┤── RCA complete ───▶ [Change Request (RFC)]
             │                              │
             │                       CAB Review (cab_review)
             │                              ▼
             │                       Freeze Window check
             │                              ▼
             │                       Approved → in_progress
             │                              │
             ▼                              ▼
        (1..N changes)                      │
                                            ▼
                                    [Release Package]
                                    ├ Build status
                                    ├ Deploy status
                                    ├ Rollback plan
                                            ▼
                              Deploy + Production
                                            │
                                            ▼
                          [Post-Implementation Review (PIR)]
                          outcome: success | partial | rollback | failed
                          lessons_learned, success_rating
                                            │
                                            ▼ (non-success outcome OR manual)
                         [Improvement Candidate] (CSI Register)
                         improvement_source.source_type='pir'
                                            │ approved
                                            ▼
                          [New Change RFC] (loop continues)
                          issue.spawned_from_improvement_id = improvement.id
```

**Key junction tables** (every agent should know these):
- `release_change` — the changes inside a release
- `release_pir` — PIR records for a release
- `improvement_source` — where an improvement originated (`pir`/`problem_rca`/`incident_trend`/`audit`/`kb_feedback`/`manual`)
- `issue.spawned_from_improvement_id` — improvement → change origin pointer
- `cab_review` — CAB members' approvals on a change
- `change_freeze_window` — windows that block implementation

## 3. Module Map

| Module | issue_type | Flow | Critical fields |
|---|---|---|---|
| **Incident** | `incident` | Service-disruption report → assigned → resolved | `urgency`, `itil_impact`, `priority` (matrix-computed), `is_major_incident` |
| **Problem** | `problem` | Root cause for recurring incidents | `root_cause`, `workaround`, RCA notes |
| **Change** | `change` | RFC → CAB → implement → PIR | `change_type` (standard/normal/emergency), `risk_level`, `impact_level`, `rollback_plan`, `test_plan` |
| **Release** | `release` | Group changes, deploy them | `release_version`, `release_environment`, `release_build_status`, `release_deploy_status`, `release_notes`, `release_target_date` |
| **Service Request** | `service_request` | Catalog-driven request | `catalog_item_id`, `form_data` (JSONB) |
| **Task** | `task` | Generic work item | `parent_issue_id`, `due_date` |
| **CSI Improvement** | (own table) | Continual improvement | PDCA (`plan/do/check/act_notes`), `expected_benefit`, `actual_benefit`, `priority`, `status` (submitted/reviewing/approved/implementing/done/rejected) |
| **Contract** | (own table) | Contract management | `contract_type`, `party_name`, `start_date`/`end_date`, `auto_renew`, `notice_days`, `sla_policy_id`, attachments (PDF/DOCX) |
| **CMDB** | (own table) | Configuration items + relations | `cmdb_config_item`, `cmdb_relation` (`depends_on`, `hosted_on`, `connected_to`, `part_of`, `uses`, `backs_up`) |
| **SLA** | `sla_policy` + `sla_record` | Response + resolution tracking | `response_minutes`, `resolution_minutes`, `pause_on_statuses`, `escalation_group_id`, `breach_at` |

## 4. Agent Behavioural Rules

When assigned to an issue an agent must apply the following discipline:

### 4.1 Status flow

- `todo` → `in_progress`: first move when work starts. Combine the status change with a comment in one go (reduce event spam).
- `in_progress` → `in_review`: agent believes the work is done but human approval is needed (e.g. a production change). The typical step before PIR.
- `in_progress` → `done`: standalone task or human approval not required. Pair with a comment summarising the result.
- `in_progress` → `cancelled`: makes no sense to continue (duplicate, out-of-scope). Always leave the reason as a comment.

### 4.2 Comment discipline

Every status change should be accompanied by a comment that says **what was done and what was learned**. Style:
- **Bullet points**, not prose — reading speed matters.
- **Failure ≠ blame**. Use the form "step N failed because X → fallback Y was attempted".
- **Linked items**: reference related issues / CIs / contracts when relevant (`#CHG-PROD-0042`).

### 4.3 For change-type issues

- For non-`emergency` change types: moving status to `in_progress` while an active **freeze window** exists returns 409 from the backend. In that case the agent must either:
  - Escalate `change_type` to `emergency` (only after human approval — the agent must not unilaterally upgrade)
  - Or wait until the freeze window ends
- If `rollback_plan` is empty, **fill it before** moving forward. Don't deploy without a plan.
- If CAB review is required (`change_type='normal'` or `risk_level='high'+`), check with `ListCABReviews` first — without approval implementation is blocked.

### 4.4 Submitting a release

For a release issue (`issue_type='release'`):
1. Set `release_target_date`
2. Build scope: `POST /api/issues/{releaseId}/release-changes` for each change
3. `release_build_status` → `passed` (after tests succeed)
4. `release_deploy_status` → `deploying` → `deployed`
5. After deploy, **write a PIR**: `POST /api/issues/{releaseId}/release-pirs`
   - `outcome`: success|partial|rollback|failed
   - `success_rating`: 1-5
   - `lessons_learned`: takeaway for the next release
   - `open_improvement`: true (default; false on `success` outcome) — auto-creates a CSI improvement candidate

### 4.5 When problem RCA is complete

```
POST /api/issues/{problemId}/spawn-improvement
body: { title?, description?, category? }
```

Automatically flags `improvement_source.source_type='problem_rca'`; the team can then approve and turn it into a change.


### 4.6 Always check project settings before sequencing

Before delegating or sequencing any multi-step work, **read `caliptic project get <id> --output json`** and honour the project's preferred workflow. Do NOT apply ITIL4 PR/CAB defaults uniformly — they apply to production `change_type='normal'`/`risk_level='high'+` only. For everyday development:

- If the project signals direct-merge (no `merge_strategy='pr_required'` field, small team, fast-iteration code), agents push directly to main after completing work; no PR + review round-trip.
- If the project signals PR-required (e.g., `merge_strategy='pr_required'` once exposed, or explicit owner instruction), open PR + run review + merge.
- When in doubt, ask the owner. Default ITIL4 sequence is for production changes, not development iteration.

This rule was added after the runtime hotfix work where PR + review was orchestrated when the project actually uses direct-merge — wasted orchestration effort.

### 4.8 In_review handoff is mandatory

When you move an issue to `in_review`, the SAME comment that records the status change **must** explicitly hand off to the next actor. No silent in_review.

Required handoff forms:
- **PR open → reviewer needed:** `[@Code Review Engineer](mention://agent/<id>) — PR <url>, please pass.`
- **Spec/contract done → owner approve:** `[@<Owner name>](mention://member/<id>) — please confirm.`
- **Direct-merge project (`merge_strategy='direct_merge'`):** skip in_review entirely; push to main → `done` with a summary comment.

Without an explicit @mention to the next actor, in_review issues stall — empirically observed in [CAL-9](mention://issue/b3052cfa-48f4-4dc8-a620-b768b6466fcb), [CAL-14](mention://issue/3ca27837-b3bd-4893-a4d2-f3c715252740), [CAL-17](mention://issue/f1fa5ddb-32e3-4186-b99c-6f1cdbbe4647) which sat 13+ hours each.

Dev Supervisor sweep protocol: at every user touchpoint, run `caliptic issue list --status in_review` and ping/close anything older than 4h.

### 4.7 Contract analysis

When assigned to a contract (`POST /api/contracts/{id}/analyze`):
1. Pulls attachment URLs into the description (PDF/DOCX/TXT)
2. Produces a **standard analysis**: parties, start/end, value, termination, auto-renewal, confidentiality, SLA commitments
3. Determines **risk level**: low/medium/high + rationale
4. Posts the result back as timeline + comment on the issue

## 5. API Quick Reference

### 5.1 Authentication & workspace context

Every request:
- `Authorization: Bearer <PAT or JWT>`
- `X-Workspace-Slug: acme` OR `X-Workspace-ID: <uuid>`
- If you are an agent: `X-Agent-ID: <agent.id>` + `X-Task-ID: <task.id>` (the server cross-checks both)

### 5.2 Issue CRUD

```
GET    /api/issues?issue_type=change&status=in_progress
POST   /api/issues                       — new issue
GET    /api/issues/{id}                  — detail
PUT    /api/issues/{id}                  — update (status, fields)
POST   /api/issues/{id}/comments         — comment
GET    /api/issues/{id}/timeline         — audit log
```

### 5.3 ITIL4 loop endpoints (most important)

```
# Release
GET    /api/issues/{id}/release-detail
PATCH  /api/issues/{id}/release-phase
GET    /api/issues/{id}/release-changes
POST   /api/issues/{id}/release-changes  body: {change_id}
DELETE /api/issues/{id}/release-changes/{changeId}
GET    /api/issues/{id}/release-pirs
POST   /api/issues/{id}/release-pirs     body: {outcome, success_rating, lessons_learned, open_improvement?}

# Improvement (CSI)
GET    /api/workspaces/{wsId}/improvements
POST   /api/workspaces/{wsId}/improvements              body: {title, category, priority, ...}
PATCH  /api/workspaces/{wsId}/improvements/{id}
GET    /api/workspaces/{wsId}/improvements/{id}/evidence
POST   /api/workspaces/{wsId}/improvements/{id}/spawn-change body: {title, change_type, ...}

# Problem → Improvement
POST   /api/issues/{problemId}/spawn-improvement

# Contract analysis
POST   /api/contracts/{id}/attachments    (multipart)
POST   /api/contracts/{id}/analyze        body: {agent_id, prompt?}
```

### 5.4 CAB & Freeze (Change Management)

```
GET    /api/issues/{changeId}/cab-reviews
POST   /api/issues/{changeId}/cab-reviews/decision body: {decision: approved|rejected|abstained, notes}
GET    /api/workspaces/{wsId}/freeze-windows
```

### 5.5 CMDB

```
GET    /api/workspaces/{wsId}/cmdb/cis
GET    /api/workspaces/{wsId}/cmdb/topology?root_ci_id=<uuid>
GET    /api/workspaces/{wsId}/cmdb/impact-analysis/{ciId}
POST   /api/workspaces/{wsId}/cmdb/relations  body: {source_ci_id, target_ci_id, relationship_type}
```

## 6. Data Model — Common Shapes

### Issue core fields
```ts
{
  id: uuid, workspace_id: uuid, number: int,           // unique within workspace
  title, description,
  status: "backlog"|"todo"|"in_progress"|"in_review"|"done"|"cancelled",
  priority: "none"|"low"|"medium"|"high"|"urgent",
  issue_type: "task"|"incident"|"problem"|"change"|"service_request"|"release",
  assignee_type: "member"|"agent"|null, assignee_id: uuid|null,
  creator_type, creator_id,
  project_id, sprint_id, parent_issue_id, due_date,
  // Change-specific
  change_type, risk_level, impact_level, rollback_plan, test_plan,
  scheduled_start_at, scheduled_end_at, actual_start_at, actual_end_at,
  // Release-specific
  release_version, release_environment, release_target_date,
  release_notes, release_build_status, release_deploy_status,
  // Origin
  spawned_from_improvement_id: uuid|null,
  // Incident/Problem
  root_cause, workaround, is_major_incident,
  urgency, itil_impact,
}
```

### Improvement (CSI)
```ts
{
  id, workspace_id, number: "CSI-0001",
  title, description,
  category: "process"|"tool"|"team"|"security"|"cost"|"quality"|"other",
  status: "submitted"|"reviewing"|"approved"|"implementing"|"done"|"rejected",
  priority,
  // PDCA
  plan_notes, do_notes, check_notes, act_notes,
  expected_benefit, actual_benefit, effort_estimate,
  proposer_id, owner_id, target_date, completed_at,
}
```

## 7. Anti-Patterns — Don't!

❌ **Don't mix data across workspaces**. Every query is `workspace_id`-filtered; cross-workspace ID guessing returns 404 from the backend.

❌ **Don't change status silently**. Every transition needs a comment + reason.

❌ **Don't decide on `emergency` change classification alone**. Human approval + audit trail is what the Change Advisory Board exists for.

❌ **Don't try to bypass a freeze window**. The backend blocks it anyway; if you really must proceed, propose `change_type` escalation to a human instead.

❌ **Don't deploy a release without a rollback plan**. If `release.rollback_plan` is empty, PATCH it first, then deploy.

❌ **Don't mark a release `done` without a PIR**. Deploy complete ≠ release complete. The PIR output feeds the CSI register.

❌ **Don't reach across workspaces when attaching contracts or analysing PDFs**. Attachments are workspace-scoped — don't try to access another workspace's files.

## 8. Supporting Files

This skill bundle ships with the following supporting files:
- `api-cheatsheet.md` — Detailed list of every REST endpoint
- `itil4-loop.md` — Concrete cross-module flow examples
- `agent-task-patterns.md` — Common task patterns (incident triage, change implementation, etc.)

Reach for those when you need depth on a specific topic.



## 9. Project Wiki — 6-Layer Append-Only Knowledge Base

Every project has an **append-only 6-layer wiki** that is injected into the agent's context whenever an issue in that project is opened. Agents add new entries; old entries are never deleted — they are superseded (the new entry links back via `supersedes_entry_id`).

### 9.1 The 6 Layers

| # | Layer | Purpose |
|---|---|---|
| 1 | **Vision** | Why does the project exist? One-line + 1-2 paragraph context. |
| 2 | **Domain** | Core concepts, entities, glossary. |
| 3 | **Architecture** | Module map, junction relationships, architectural decisions. |
| 4 | **API** | REST/RPC surface, header contracts, auth. |
| 5 | **Runbook** | Operational flows (release, incident, freeze, etc). |
| 6 | **Anti-patterns** | Pitfalls, lessons from past incidents. |

### 9.2 Endpoints

```
GET    /api/projects/empty-wiki                          → projects with no wiki entries yet
GET    /api/projects/{id}/wiki                           → all entries grouped by layer
GET    /api/projects/{id}/wiki/layer/{n}                 → current + supersede chain for one layer
POST   /api/projects/{id}/wiki/entries                   → body: {layer, content_md, supersedes_entry_id?}
POST   /api/projects/{id}/wiki/init                      → body: {agent_id?, mode: "analyze"|"manual"} → spawn analysis task
PATCH  /api/projects/{id}/wiki/entries/{eid}/supersede   → flag entry as superseded
```

Agent writes authenticate via the standard `X-Agent-ID + X-Task-ID` cross-check; no extra project-level board policy applies (unlike `PUT /api/projects/{id}`).

### 9.3 Trigger 1 — First Init (manual)

Triggered from the desktop UI's **First Init** button (sits above "Plan what to work on next").
1. UI lists projects with empty wikis via `GET /api/projects/empty-wiki`.
2. User picks a project P.
3. UI calls `POST /api/projects/{P}/wiki/init`.
4. Backend checks `project.secrets.GITHUB_TOKEN`; if present, spawns an analysis task for an architecturally-savvy agent (Solution Architect by default).
5. Agent checks out the repo, reads README/CLAUDE.md/migrations/handlers, and writes one entry per layer.

### 9.4 Trigger 2 — Done Append (automatic)

When an issue (or release) inside a project transitions to `done`:
1. Backend emits `issue.status_changed` event.
2. Autopilot `wiki-done-append` matches `to=done` and enqueues the **Wiki Keeper** agent.
3. The Wiki Keeper reads the closed issue + diff + comments and decides which layer(s) need a new entry.
4. New entries are appended via `POST /api/projects/{P}/wiki/entries`. If an entry replaces an outdated current entry, it must set `supersedes_entry_id`.

### 9.5 Behavioural Rules for Agents

- **Never delete an entry.** If a layer's current entry is wrong or outdated, write a new entry and set `supersedes_entry_id`. The audit trail stays intact.
- **One layer per entry.** Don't cross-mix Domain + API in a single entry — the injector groups by layer.
- **Concise, structured Markdown.** Bullet points and tables beat prose for context injection.
- **Reference issues** with `[CAL-XXX](mention://issue/<id>)` so the wiki stays navigable.
- **Done append is idempotent.** If the same issue is re-transitioned to `done`, the agent must check `trigger_source='done_append' AND source_issue_id` before writing a duplicate.

### 9.6 Anti-Patterns

- ❌ Editing an entry in place — append-only means append-only.
- ❌ Writing a Done-append entry without naming the source issue in `content_md`.
- ❌ Putting Anti-patterns content in the Runbook layer because "it felt operational".
- ❌ Bypassing the supersede chain ("just write a new one and ignore the old") — context injection will then show both, confusing future agents.

---

**Source references:**
- CLAUDE.md (project root) — architectural decisions
- `server/migrations/105_itil4_loop.up.sql` — ITIL4 loop schema
- `server/internal/handler/{release,improvement_extra,contract_attachment,change}.go` — handler implementations
