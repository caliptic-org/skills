# Caliptic API Cheatsheet

Every endpoint expects `Authorization: Bearer <token>` plus a workspace
context header (`X-Workspace-Slug` or `X-Workspace-ID`). URL-prefixed
workspace routes also accept the workspace ID inline as `{wsId}`.

## Issues

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/issues` | List, filterable by `issue_type`, `status`, `priority`, `assignee_id`, `project_id`, `sprint_id`, `limit`, `offset` |
| POST | `/api/issues` | Create. Body: `{title, description, issue_type, status, priority, assignee_type, assignee_id, parent_issue_id, project_id, due_date, catalog_item_id?, form_data?}` |
| GET | `/api/issues/{id}` | Detail (workspace check is automatic) |
| PUT | `/api/issues/{id}` | Update — partial fields |
| DELETE | `/api/issues/{id}` | Soft delete |
| GET | `/api/issues/{id}/timeline` | Audit log (status changes, assignments, comments) |
| GET | `/api/issues/{id}/comments` | Comments |
| POST | `/api/issues/{id}/comments` | Add comment. Body: `{content, type?}` |
| POST | `/api/issues/{id}/subscribe` | Watch this issue |
| POST | `/api/issues/{id}/reactions` | Emoji react. Body: `{emoji}` |
| GET | `/api/issues/{id}/attachments` | File attachments |
| POST | `/api/issues/{id}/rerun` | Re-dispatch the agent task |

## Change Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/issues/{id}/cab-reviews` | CAB member approval states |
| POST | `/api/issues/{id}/cab-reviews` | Add a reviewer. Body: `{reviewer_id}` |
| POST | `/api/issues/{id}/cab-reviews/decision` | Submit your own decision. Body: `{decision: approved\|rejected\|abstained, notes?}` |
| GET | `/api/workspaces/{wsId}/freeze-windows` | Active & upcoming freeze windows |
| POST | `/api/workspaces/{wsId}/freeze-windows` | New freeze. Body: `{name, starts_at, ends_at, reason?}` |

## Release Management (new — migration 105)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/issues/{id}/release-detail` | Release header + change_count + pir_count |
| PATCH | `/api/issues/{id}/release-phase` | Build/deploy status, target_date, release_notes, rollback_plan |
| GET | `/api/issues/{id}/release-changes` | Changes currently in this release's scope |
| POST | `/api/issues/{id}/release-changes` | Add a change. Body: `{change_id}` — must be `issue_type='change'` in the same workspace |
| DELETE | `/api/issues/{id}/release-changes/{changeId}` | Remove a change from scope |
| GET | `/api/issues/{id}/release-pirs` | All PIR records (multiple are allowed) |
| POST | `/api/issues/{id}/release-pirs` | Submit a PIR. Body: `{outcome, success_rating, what_went_well, what_went_wrong, lessons_learned, open_improvement?}` |
| DELETE | `/api/issues/{id}/release-pirs/{pirId}` | Delete a PIR record (owner only) |

## Improvement (CSI)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workspaces/{wsId}/improvements` | List |
| POST | `/api/workspaces/{wsId}/improvements` | Create |
| PATCH | `/api/workspaces/{wsId}/improvements/{id}` | Update (covers status changes and PDCA fields) |
| DELETE | `/api/workspaces/{wsId}/improvements/{id}` | Delete (owner/admin) |
| GET | `/api/workspaces/{wsId}/improvements/{id}/evidence` | Sources + spawned issues (loop trace) |
| POST | `/api/workspaces/{wsId}/improvements/{id}/spawn-change` | New RFC from an approved/implementing improvement. Body: `{title, change_type, risk_level?, impact_level?, project_id?}` |

## Problem & RCA

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workspaces/{wsId}/problems` | List of problems |
| PATCH | `/api/issues/{id}/rca` | Update RCA. Body: `{root_cause, workaround}` |
| GET | `/api/issues/{id}/related-incidents` | Incidents linked to this problem |
| POST | `/api/issues/{id}/link-problem` | Link an incident to a problem |
| POST | `/api/issues/{problemId}/spawn-improvement` | Open a CSI improvement from problem RCA. Body: `{title?, description?, category?}` |

## Contract Management

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workspaces/{wsId}/contracts` | List |
| POST | `/api/workspaces/{wsId}/contracts` | Create |
| PATCH | `/api/workspaces/{wsId}/contracts/{id}` | Update |
| DELETE | `/api/workspaces/{wsId}/contracts/{id}` | Delete |
| GET | `/api/workspaces/{wsId}/contracts/{id}/attachments` | Attached PDF/DOCX files |
| POST | `/api/workspaces/{wsId}/contracts/{id}/attachments` | **multipart** — form field name `file` |
| DELETE | `/api/workspaces/{wsId}/contracts/{id}/attachments/{attId}` | Delete |
| POST | `/api/workspaces/{wsId}/contracts/{id}/analyze` | Trigger an agent task. Body: `{agent_id, prompt?}` — creates an issue and assigns the agent |

## CMDB

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workspaces/{wsId}/cmdb/cis` | All CIs |
| POST | `/api/workspaces/{wsId}/cmdb/cis` | Create a CI. Body: `{name, ci_class, status?, environment?, parent_ci_id?, criticality?}` |
| GET | `/api/workspaces/{wsId}/cmdb/cis/{ciId}` | Detail |
| PATCH | `/api/workspaces/{wsId}/cmdb/cis/{ciId}` | Update |
| DELETE | `/api/workspaces/{wsId}/cmdb/cis/{ciId}` | Delete |
| GET | `/api/workspaces/{wsId}/cmdb/topology?root_ci_id=<uuid>&depth=3` | Nodes + edges |
| GET | `/api/workspaces/{wsId}/cmdb/impact-analysis/{ciId}` | "If this CI breaks, what's affected?" |
| GET | `/api/workspaces/{wsId}/cmdb/dependencies/{ciId}` | "What does this CI depend on?" |
| GET | `/api/workspaces/{wsId}/cmdb/relations` | All relationship edges |
| POST | `/api/workspaces/{wsId}/cmdb/relations` | Create a relation. Body: `{source_ci_id, target_ci_id, relationship_type: depends_on\|hosted_on\|connected_to\|part_of\|uses\|backs_up}` |

## SLA

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workspaces/{wsId}/sla-policies` | All policies |
| POST | `/api/workspaces/{wsId}/sla-policies` | New policy |
| GET | `/api/issues/{id}/sla` | This issue's SLA records (response + resolution stages) |

## Service Catalog

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workspaces/{wsId}/catalog/categories` | Categories |
| GET | `/api/workspaces/{wsId}/catalog/items` | Service catalog items |
| POST | `/api/workspaces/{wsId}/catalog/items` | New item |
| GET | `/api/workspaces/{wsId}/catalog/items/{itemId}` | Detail (form schema included) |

Open a service request: `POST /api/issues` with `issue_type='service_request'` + `catalog_item_id` + `form_data`.

## Agents (workspace members)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/agents` | Workspace agents |
| POST | `/api/agents` | Create an agent (admin/owner) |
| GET | `/api/agents/{id}` | Detail |
| PUT | `/api/agents/{id}` | Update (instructions, model, custom_env, etc.) |
| GET | `/api/agents/{id}/skills` | Assigned skills |
| PUT | `/api/agents/{id}/skills` | Replace the skill set. Body: `{skill_ids: [...]}` |
| GET | `/api/agents/{id}/tasks` | Task history |
| POST | `/api/agents/{id}/archive` | Soft delete |

## Skills

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/skills` | Workspace skills |
| POST | `/api/skills` | New skill. Body: `{name, description, content, config?}` |
| GET | `/api/skills/{id}` | Detail |
| PUT | `/api/skills/{id}` | Update |
| GET | `/api/skills/{id}/files` | Supporting files |
| PUT | `/api/skills/{id}/files` | Upsert a file. Body: `{path, content}` |
| POST | `/api/skills/import` | Import from URL (ClawHub, GitHub). Body: `{url}` |

## Inbox

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/inbox` | Notification feed |
| GET | `/api/inbox/unread-count` | Unread count |
| POST | `/api/inbox/{id}/read` | Mark as read |
| POST | `/api/inbox/mark-all-read` | Mark all as read |
| POST | `/api/inbox/{id}/archive` | Archive |

## Chat (direct conversation with an agent)

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/chat/sessions` | New chat session. Body: `{agent_id, title?}` |
| GET | `/api/chat/sessions` | List |
| GET | `/api/chat/sessions/{id}` | Detail |
| DELETE | `/api/chat/sessions/{id}` | Archive (soft) |
| DELETE | `/api/chat/sessions/{id}/permanent` | Hard delete (cascades messages) |
| GET | `/api/chat/sessions/{id}/messages` | Messages |
| POST | `/api/chat/sessions/{id}/messages` | Send a message — triggers an agent reply task. Body: `{content}` |
| DELETE | `/api/chat/sessions/{id}/messages/{messageId}` | Delete a single message |

## Workflows (visual flow engine)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/workflows` | Workflow list |
| POST | `/api/workflows` | Create. Body: `{name, workflow_type: automation\|process, config}` |
| POST | `/api/workflows/{id}/events` | Push an event (workflow trigger) |
| GET | `/api/workflows/{id}/runs` | Run history |
