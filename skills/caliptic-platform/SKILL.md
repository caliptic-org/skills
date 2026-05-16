---
name: caliptic-platform
description: Caliptic ITSM platformunun bütüncül kullanım rehberi. Bu skill'i bir agent'a verirsen — agent Caliptic'in domain model'ini (issues, changes, releases, problems, contracts, CMDB, sprints), ITIL4 hizmet değer akışını (incident → problem → change → release → PIR → improvement), modüller arası bağlantıları, API endpoint'lerini ve agent davranış kurallarını eksiksiz bilir. Default supervisor'a bağlanmak için tasarlandı — workspace genelinde her görevde geçerli ortak bilgi tabanı. Issue oluşturma, RCA yazma, change request açma, release scope yönetimi, PIR submit, CSI improvement spawn etme, sözleşme analizi gibi işleri Caliptic'in API'leri ve mimari ilkeleriyle yapar.
---

# Caliptic ITSM Platform — Agent Rehberi

Bu skill bir AI agent'ın Caliptic platformunda **first-class teammate** olarak çalışması için gereken her şeyi içerir. Agent issue'lara atanır, yorum yazar, status değiştirir, ilişkili kayıt açar; bunu yaparken **ITIL4 disiplinini** ve Caliptic'in mimari ilkelerini gözetir.

## 1. Caliptic Nedir?

**AI-native ITSM platformu** — Linear/Jira benzeri ama AI agent'lar first-class. Türk ITIL4 ekosistemine yönelik kurulmuş, çok kiracılı (multi-tenant) SaaS + on-premise. Her veri sorgusu `workspace_id` filtreli; member/admin/owner rol modeli.

**Temel kavramlar:**
- **Workspace** — kiracı kapsamı, tüm verinin etrafında dönen bağlam (`X-Workspace-Slug` veya `X-Workspace-ID` header'ı ile bağlanır).
- **Issue** — temel iş birimi; tipi (`issue_type`) belirler: `task`, `incident`, `problem`, `change`, `service_request`, `release`. Aynı tablo polymorphic.
- **Agent** — AI teammate. İnsan member gibi assignee olabilir, comment atabilir, status değiştirebilir. `assignee_type='agent'` + `assignee_id=<agent.id>`.
- **Project** — issue gruplama; agent erişimi `project_access` ile sınırlandırılabilir (private projects).
- **Sprint** — proje altında zaman kutusu; ITIL4 release scope'undan farklı (sprint = takım iterasyon, release = production deploy).

## 2. ITIL4 Hizmet Değer Akışı (Loop)

Caliptic'in en kritik özelliği: modüller fiziksel olarak birbirine bağlıdır (migration 105 ile gelen junction tabloları):

```
[Incident] ──┐
[Problem]  ──┤── RCA tamamlandı ──▶ [Change Request (RFC)]
             │                              │
             │                       CAB Review (cab_review tablosu)
             │                              ▼
             │                       Freeze Window check
             │                              ▼
             │                       Approved → in_progress
             │                              │
             ▼                              ▼
         (1..N changes)                    │
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
                                    ▼ (non-success outcome veya manuel)
                         [Improvement Candidate] (CSI Register)
                         improvement_source.source_type='pir'
                                    │ approved
                                    ▼
                          [Yeni Change RFC] (loop devam eder)
                          issue.spawned_from_improvement_id = improvement.id
```

**Anahtar junction tabloları** (agent bunları bilmeli):
- `release_change` — bir release altındaki change'ler
- `release_pir` — bir release'in PIR kayıtları
- `improvement_source` — improvement nereden doğdu (pir/problem_rca/incident_trend/audit/kb_feedback/manual)
- `issue.spawned_from_improvement_id` — improvement → change origin pointer
- `cab_review` — change için CAB üyelerinin onayları
- `change_freeze_window` — implementation'ı bloklayan donmuş pencereler

## 3. Modül Haritası

| Modül | issue_type | Ana akış | Kritik alanlar |
|---|---|---|---|
| **Incident** | `incident` | Service disruption raporu → atanır → çözüm | `urgency`, `itil_impact`, `priority` (matrix'ten hesap), `is_major_incident` |
| **Problem** | `problem` | Recurring incident root cause | `root_cause`, `workaround`, RCA notları |
| **Change** | `change` | RFC → CAB → implement → PIR | `change_type` (standard/normal/emergency), `risk_level`, `impact_level`, `rollback_plan`, `test_plan` |
| **Release** | `release` | Change'leri grupla, deploy et | `release_version`, `release_environment`, `release_build_status`, `release_deploy_status`, `release_notes`, `release_target_date` |
| **Service Request** | `service_request` | Catalog'dan istek | `catalog_item_id`, `form_data` (JSONB) |
| **Task** | `task` | Genel iş kaydı | parent_issue_id, due_date |
| **CSI Improvement** | (kendi tablosu) | Continual improvement | PDCA (plan/do/check/act_notes), `expected_benefit`, `actual_benefit`, `priority`, `status` (submitted/reviewing/approved/implementing/done/rejected) |
| **Contract** | (kendi tablosu) | Sözleşme yönetimi | `contract_type`, `party_name`, `start_date`/`end_date`, `auto_renew`, `notice_days`, `sla_policy_id`, attachments (PDF/DOCX) |
| **CMDB** | (kendi tablosu) | Configuration items + ilişkiler | `cmdb_config_item`, `cmdb_relation` (depends_on, hosted_on, connected_to, part_of, uses, backs_up) |
| **SLA** | `sla_policy` + `sla_record` | Response + resolution süre takibi | `response_minutes`, `resolution_minutes`, `pause_on_statuses`, `escalation_group_id`, `breach_at` |

## 4. Agent Davranış Kuralları

Bir agent issue'ya atandığında **şu disiplini** uygulamalı:

### 4.1 Status akışı

- `todo` → `in_progress`: işe başlandığında ilk hareket. Tek atımda yorum + status değişikliği yap (event spam'i azalt).
- `in_progress` → `in_review`: agent kendince bitirdi ama insan onayı bekliyor (örn: production change). PIR öncesi tipik halka.
- `in_progress` → `done`: standalone task ya da insan onayı gereksiz iş. Yorum + result summary ile birlikte.
- `in_progress` → `cancelled`: yapılması anlamsız (duplicate, out-of-scope). Gerekçeyi yorum olarak yaz.

### 4.2 Yorum disiplini

Her status değişikliğinde **ne yapıldığını, ne öğrenildiğini** yorum olarak bırak. Stil:
- **Bullet points**, prozak değil — okuma hızı kritik.
- **Failure ≠ blame**. "Bu adım N nedeniyle başarısız oldu → şu fallback denendi" formatı.
- **Linked items**: ilgili issue/CI/contract referansı varsa link bırak (`#CHG-PROD-0042`).

### 4.3 Change tipindeki issue'larda

- **Change_type='emergency'** olmayan change'in status'unu `in_progress`'e geçirirken, aktif **freeze window** varsa backend 409 döner. Bu durumda agent ya:
  - change_type'ı `emergency` yapmalı (insan onayı sonrası — agent kendi başına yükseltmemeli)
  - veya freeze window'un bitişini beklemeli
- **rollback_plan** boşsa **uygulamadan önce** doldur. Plan yoksa deploy yapma.
- **CAB review** gerekiyorsa (`change_type='normal'`/`risk_level='high'+`), önce `ListCABReviews` ile durumu kontrol et — onay yoksa implementation bloklu.

### 4.4 Release submit etmek

Release issue'su (`issue_type='release'`) yapısı:
1. `release_target_date` set et
2. Scope: `POST /api/issues/{releaseId}/release-changes` ile change'leri ekle
3. `release_build_status` → `passed` (test geçtikten sonra)
4. `release_deploy_status` → `deploying` → `deployed`
5. Deploy sonrası **PIR yaz**: `POST /api/issues/{releaseId}/release-pirs`
   - `outcome`: success|partial|rollback|failed
   - `success_rating`: 1-5
   - `lessons_learned`: bir sonraki release için ders
   - `open_improvement`: true (default; success outcome'da false) — CSI register'a otomatik improvement candidate açılır

### 4.5 Problem RCA tamamlandığında

```
POST /api/issues/{problemId}/spawn-improvement
body: { title?, description?, category? }
```

Otomatik improvement_source.source_type='problem_rca' işaretlenir; takım sonradan onaylayıp change'e çevirebilir.

### 4.6 Contract analizi

Agent contract'a atandığında (`POST /api/contracts/{id}/analyze`):
1. Attachments URL'lerini description'a alır (PDF/DOCX/TXT)
2. **Standart analiz** çıkarır: taraflar, başlangıç/bitiş, değer, fesih, otomatik yenileme, gizlilik, SLA taahhütleri
3. **Risk seviyesi** belirler: düşük/orta/yüksek + gerekçe
4. Sonucu issue üzerinden timeline + comment olarak bırakır

## 5. API Hızlı Rehber

### 5.1 Authentication & workspace context

Her istek:
- `Authorization: Bearer <PAT veya JWT>`
- `X-Workspace-Slug: acme` VEYA `X-Workspace-ID: <uuid>`
- Agent ise: `X-Agent-ID: <agent.id>` + `X-Task-ID: <task.id>` (server cross-check yapar)

### 5.2 Issue CRUD

```
GET    /api/issues?issue_type=change&status=in_progress
POST   /api/issues                       — yeni issue
GET    /api/issues/{id}                  — detay
PUT    /api/issues/{id}                  — update (status, fields)
POST   /api/issues/{id}/comments         — yorum
GET    /api/issues/{id}/timeline         — audit log
```

### 5.3 ITIL4 loop endpoint'leri (en kritik)

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

# Contract analiz
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

## 6. Veri Modeli — Sık Karşılaşılan Yapılar

### Issue temel alanlar
```ts
{
  id: uuid, workspace_id: uuid, number: int,           // workspace içinde unique
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

## 7. Anti-Pattern'ler — Yapma!

❌ **Birden çok workspace'in datasını karıştırma**. Her sorgu `workspace_id` filtreli; cross-workspace ID guessing backend 404 döner.

❌ **Status'u sessizce değiştirme**. Her transition yorum + reason ile birlikte.

❌ **Emergency change yapma kararını agent kendi başına alma**. Insan onayı + audit trail için Change Advisory Board süreci.

❌ **Freeze window'u bypass etmeye çalışma**. Backend zaten engelliyor; agent gerekirse change_type değiştirmeyi insan'a önersin.

❌ **Rollback plan'sız release deploy etme**. release.rollback_plan boşsa PATCH ile doldur, sonra deploy.

❌ **PIR'sız release "done" işaretleme**. Deploy tamamlandı ≠ release tamamlandı. PIR çıktısı CSI register'ı besler.

❌ **Contract'a PDF eklerken ve agent analizinde** — attachment workspace_id ile bağlı, başka workspace'in dosyasına ulaşmaya çalışma.

## 8. Ek Dosyalar

Bu skill paketi şu destek dosyalarını içerir:
- `api-cheatsheet.md` — Tüm REST endpoint'lerinin detaylı listesi
- `itil4-loop.md` — Modüller arası akışın somut örnekleriyle anlatımı
- `agent-task-patterns.md` — Sık karşılaşılan görev kalıpları (incident triage, change implementation, vs.)

Detay için bu dosyalara bak.

---

**Kaynak referansları:**
- CLAUDE.md (proje kök dizini) — mimari kararlar
- `server/migrations/105_itil4_loop.up.sql` — ITIL4 loop şeması
- `server/internal/handler/{release,improvement_extra,contract_attachment,change}.go` — handler implementasyonları
