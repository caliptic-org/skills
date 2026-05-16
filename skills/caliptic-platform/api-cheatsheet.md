# Caliptic API Cheatsheet

Tüm endpoint'ler `Authorization: Bearer <token>` + `X-Workspace-Slug` veya `X-Workspace-ID` header gerektirir. URL-prefixed workspace route'larında `{wsId}` URL param'ı kullanılır.

## Issues

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/issues` | List with filters: `issue_type`, `status`, `priority`, `assignee_id`, `project_id`, `sprint_id`, `limit`, `offset` |
| POST | `/api/issues` | Create. Body: `{title, description, issue_type, status, priority, assignee_type, assignee_id, parent_issue_id, project_id, due_date, catalog_item_id?, form_data?}` |
| GET | `/api/issues/{id}` | Detail (workspace check otomatik) |
| PUT | `/api/issues/{id}` | Update — partial fields |
| DELETE | `/api/issues/{id}` | Soft delete |
| GET | `/api/issues/{id}/timeline` | Audit log (status changes, assignments, comments) |
| GET | `/api/issues/{id}/comments` | Yorumlar |
| POST | `/api/issues/{id}/comments` | Yorum ekle. Body: `{content, type?}` |
| POST | `/api/issues/{id}/subscribe` | Issue'yu izle |
| POST | `/api/issues/{id}/reactions` | Emoji react. Body: `{emoji}` |
| GET | `/api/issues/{id}/attachments` | Dosya ekleri |
| POST | `/api/issues/{id}/rerun` | Agent task'i yeniden başlat |

## Change Management

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/issues/{id}/cab-reviews` | CAB üyelerinin onay durumları |
| POST | `/api/issues/{id}/cab-reviews` | Reviewer ekle. Body: `{reviewer_id}` |
| POST | `/api/issues/{id}/cab-reviews/decision` | Kendi kararını gönder. Body: `{decision: approved\|rejected\|abstained, notes?}` |
| GET | `/api/workspaces/{wsId}/freeze-windows` | Aktif/gelecek freeze window'lar |
| POST | `/api/workspaces/{wsId}/freeze-windows` | Yeni freeze. Body: `{name, starts_at, ends_at, reason?}` |

## Release Management (yeni — migration 105)

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/issues/{id}/release-detail` | Release header + change_count + pir_count |
| PATCH | `/api/issues/{id}/release-phase` | Build/deploy status, target_date, release_notes, rollback_plan |
| GET | `/api/issues/{id}/release-changes` | Bu release'in scope'undaki change'ler |
| POST | `/api/issues/{id}/release-changes` | Change ekle. Body: `{change_id}` — change_type='change' ve aynı workspace olmalı |
| DELETE | `/api/issues/{id}/release-changes/{changeId}` | Change'i scope'tan çıkar |
| GET | `/api/issues/{id}/release-pirs` | Tüm PIR kayıtları (multiple olabilir) |
| POST | `/api/issues/{id}/release-pirs` | PIR submit. Body: `{outcome, success_rating, what_went_well, what_went_wrong, lessons_learned, open_improvement?}` |
| DELETE | `/api/issues/{id}/release-pirs/{pirId}` | PIR sil (owner only) |

## Improvement (CSI)

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workspaces/{wsId}/improvements` | List |
| POST | `/api/workspaces/{wsId}/improvements` | Create |
| PATCH | `/api/workspaces/{wsId}/improvements/{id}` | Update (status değişimi PDCA + dahil) |
| DELETE | `/api/workspaces/{wsId}/improvements/{id}` | Sil (owner/admin) |
| GET | `/api/workspaces/{wsId}/improvements/{id}/evidence` | Sources + spawned issues (loop trace) |
| POST | `/api/workspaces/{wsId}/improvements/{id}/spawn-change` | approved/implementing improvement'tan yeni RFC. Body: `{title, change_type, risk_level?, impact_level?, project_id?}` |

## Problem & RCA

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workspaces/{wsId}/problems` | Problem listesi |
| PATCH | `/api/issues/{id}/rca` | RCA güncelle. Body: `{root_cause, workaround}` |
| GET | `/api/issues/{id}/related-incidents` | Bu problem'e bağlı incident'lar |
| POST | `/api/issues/{id}/link-problem` | Incident'i problem'a bağla |
| POST | `/api/issues/{problemId}/spawn-improvement` | Problem RCA'dan CSI improvement aç. Body: `{title?, description?, category?}` |

## Contract Management

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workspaces/{wsId}/contracts` | List |
| POST | `/api/workspaces/{wsId}/contracts` | Create |
| PATCH | `/api/workspaces/{wsId}/contracts/{id}` | Update |
| DELETE | `/api/workspaces/{wsId}/contracts/{id}` | Sil |
| GET | `/api/workspaces/{wsId}/contracts/{id}/attachments` | Eklenen PDF/DOCX dosyaları |
| POST | `/api/workspaces/{wsId}/contracts/{id}/attachments` | **multipart** — file field "file" |
| DELETE | `/api/workspaces/{wsId}/contracts/{id}/attachments/{attId}` | Sil |
| POST | `/api/workspaces/{wsId}/contracts/{id}/analyze` | Agent task tetikle. Body: `{agent_id, prompt?}` — issue oluşturup agent'a atar |

## CMDB

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workspaces/{wsId}/cmdb/cis` | Tüm CI'lar |
| POST | `/api/workspaces/{wsId}/cmdb/cis` | CI yarat. Body: `{name, ci_class, status?, environment?, parent_ci_id?, criticality?}` |
| GET | `/api/workspaces/{wsId}/cmdb/cis/{ciId}` | Detail |
| PATCH | `/api/workspaces/{wsId}/cmdb/cis/{ciId}` | Update |
| DELETE | `/api/workspaces/{wsId}/cmdb/cis/{ciId}` | Sil |
| GET | `/api/workspaces/{wsId}/cmdb/topology?root_ci_id=<uuid>&depth=3` | Nodes + edges |
| GET | `/api/workspaces/{wsId}/cmdb/impact-analysis/{ciId}` | "Bu CI bozulursa ne etkilenir?" |
| GET | `/api/workspaces/{wsId}/cmdb/dependencies/{ciId}` | "Bu CI neye bağımlı?" |
| GET | `/api/workspaces/{wsId}/cmdb/relations` | Tüm ilişki kenarları |
| POST | `/api/workspaces/{wsId}/cmdb/relations` | İlişki yarat. Body: `{source_ci_id, target_ci_id, relationship_type: depends_on\|hosted_on\|connected_to\|part_of\|uses\|backs_up}` |

## SLA

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workspaces/{wsId}/sla-policies` | Tüm policy'ler |
| POST | `/api/workspaces/{wsId}/sla-policies` | Yeni policy |
| GET | `/api/issues/{id}/sla` | Bu issue'nun SLA kayıtları (response + resolution stage'leri) |

## Service Catalog

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workspaces/{wsId}/catalog/categories` | Kategoriler |
| GET | `/api/workspaces/{wsId}/catalog/items` | Service catalog item'ları |
| POST | `/api/workspaces/{wsId}/catalog/items` | Yeni item |
| GET | `/api/workspaces/{wsId}/catalog/items/{itemId}` | Detay (form schema dahil) |

Service request açma: `POST /api/issues` + `issue_type='service_request'` + `catalog_item_id` + `form_data`.

## Agents (workspace üyesi)

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/agents` | Workspace agent'ları |
| POST | `/api/agents` | Yeni agent yarat (admin/owner) |
| GET | `/api/agents/{id}` | Detail |
| PUT | `/api/agents/{id}` | Update (instructions, model, custom_env vs.) |
| GET | `/api/agents/{id}/skills` | Atanan skill'ler |
| PUT | `/api/agents/{id}/skills` | Skill setini değiştir. Body: `{skill_ids: [...]}` |
| GET | `/api/agents/{id}/tasks` | Task geçmişi |
| POST | `/api/agents/{id}/archive` | Soft delete |

## Skills

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/skills` | Workspace skill'leri |
| POST | `/api/skills` | Yeni skill. Body: `{name, description, content, config?}` |
| GET | `/api/skills/{id}` | Detail |
| PUT | `/api/skills/{id}` | Update |
| GET | `/api/skills/{id}/files` | Destek dosyaları |
| PUT | `/api/skills/{id}/files` | File ekle/güncelle. Body: `{path, content}` |
| POST | `/api/skills/import` | URL'den import (ClawHub, GitHub). Body: `{url}` |

## Inbox

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/inbox` | Bildirim listesi |
| GET | `/api/inbox/unread-count` | Okunmamış sayısı |
| POST | `/api/inbox/{id}/read` | Okundu işaretle |
| POST | `/api/inbox/mark-all-read` | Hepsi okundu |
| POST | `/api/inbox/{id}/archive` | Arşivle |

## Chat (Agent direkt sohbet)

| Method | Path | İşlev |
|---|---|---|
| POST | `/api/chat/sessions` | Yeni chat session. Body: `{agent_id, title?}` |
| GET | `/api/chat/sessions` | Liste |
| GET | `/api/chat/sessions/{id}` | Detail |
| DELETE | `/api/chat/sessions/{id}` | Archive (soft) |
| DELETE | `/api/chat/sessions/{id}/permanent` | Hard delete (cascade messages) |
| GET | `/api/chat/sessions/{id}/messages` | Mesajlar |
| POST | `/api/chat/sessions/{id}/messages` | Mesaj gönder + agent yanıt task'i tetikler. Body: `{content}` |
| DELETE | `/api/chat/sessions/{id}/messages/{messageId}` | Mesaj sil |

## Workflows (görsel akış motoru)

| Method | Path | İşlev |
|---|---|---|
| GET | `/api/workflows` | Workflow listesi |
| POST | `/api/workflows` | Create. Body: `{name, workflow_type: automation\|process, config}` |
| POST | `/api/workflows/{id}/events` | Event push (workflow trigger'ı) |
| GET | `/api/workflows/{id}/runs` | Çalışma geçmişi |
