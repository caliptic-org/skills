# ITIL4 ITSM Loop — Somut Akış Örnekleri

Caliptic'in ITIL4 hizmet değer akışı modüller arası bağlantılarla kurulmuştur. Bu rehber 4 tipik senaryoyu adım adım gösterir.

## Senaryo 1: Production Incident → Permanent Fix → Release

**Tetikleyici:** Monitoring alert düşer; user "ödeme servisi cevap vermiyor" diyor.

### Adım 1 — Incident aç
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
Backend `priority_matrix` aktifse priority otomatik hesaplanır.

### Adım 2 — Geçici çözüm (workaround) + Problem oluştur
Incident kapatıldıktan sonra RCA için:
```http
POST /api/issues
{
  "title": "RCA: Payment service intermittent 500s",
  "issue_type": "problem",
  "priority": "high"
}
```
Sonra incident'ı problem'a link et:
```http
POST /api/issues/{incidentId}/link-problem
{ "problem_id": "<problemId>" }
```

### Adım 3 — RCA tamamla, kalıcı çözüm için improvement
```http
PATCH /api/issues/{problemId}/rca
{
  "root_cause": "DB connection pool exhausted under load — max=20, peak demand=50",
  "workaround": "Restart payment-service pods every 6h via cron"
}

POST /api/issues/{problemId}/spawn-improvement
{
  "title": "Increase payment-service connection pool + circuit breaker",
  "category": "tool",
  "expected_benefit": "Eliminate restart-every-6h workaround"
}
```
→ improvement_proposal kaydı açılır, `improvement_source.source_type='problem_rca'`.

### Adım 4 — Improvement approved → Change spawn
Takım approve eder:
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
→ Yeni issue (type=change), `spawned_from_improvement_id` set.

### Adım 5 — CAB review + implement
```http
POST /api/issues/{changeId}/cab-reviews { "reviewer_id": "<db-team-lead>" }
POST /api/issues/{changeId}/cab-reviews/decision { "decision": "approved", "notes": "Pool size validated in staging" }

PUT /api/issues/{changeId}
{ "rollback_plan": "kubectl rollout undo + revert config map", "status": "in_progress" }
```
Backend freeze window check yapar — aktifse 409 + change_type=emergency önerisi.

### Adım 6 — Release scope'a ekle
```http
POST /api/issues
{ "title": "Payment service v2.4.0", "issue_type": "release", "release_environment": "production" }

POST /api/issues/{releaseId}/release-changes { "change_id": "<changeId>" }

PATCH /api/issues/{releaseId}/release-phase
{
  "release_build_status": "passed",
  "release_target_date": "2026-05-20",
  "release_notes": "Increase DB pool to 100 + circuit breaker",
  "rollback_plan": "kubectl rollout undo deployment/payment-service"
}
```

### Adım 7 — Deploy + PIR
```http
PATCH /api/issues/{releaseId}/release-phase
{ "release_deploy_status": "deployed" }

POST /api/issues/{releaseId}/release-pirs
{
  "outcome": "success",
  "success_rating": 5,
  "what_went_well": "Deploy 4 dk, zero downtime, pool stable",
  "lessons_learned": "Pool size monitoring metrics eklemek lazım",
  "open_improvement": true
}
```
→ Yeni improvement candidate açılır (`source_type='pir'`) — döngü kapanır, sürekli iyileştirme.

---

## Senaryo 2: Emergency Change (Freeze Window'a Rağmen)

Production'da güvenlik açığı var. Aktif freeze window olsa bile geçmek gerek.

```http
# 1. Change yarat
POST /api/issues
{
  "title": "EMERGENCY: SQL injection patch for /api/search",
  "issue_type": "change",
  "change_type": "emergency",
  "risk_level": "low",
  "rollback_plan": "git revert + redeploy"
}

# 2. Direkt implement — freeze validator emergency'yi bypass eder
PUT /api/issues/{id}
{ "status": "in_progress", "change_type": "emergency" }
```

**Agent davranışı:** Emergency'yi sadece insan onaylar. Agent kendi başına `change_type='emergency'` set etmemeli — instead `change_type='normal'` deneyip 409 alırsa yorum bırakır: "Freeze window aktif; emergency override için insan onayı gerekli."

---

## Senaryo 3: Release Rollback + İkinci PIR

Deploy başarısız:
```http
PATCH /api/issues/{releaseId}/release-phase
{ "release_deploy_status": "rolled_back" }

POST /api/issues/{releaseId}/release-pirs
{
  "outcome": "rollback",
  "success_rating": 2,
  "what_went_wrong": "DB migration script lock-table 12 dk sürdü, traffic geri çekildi",
  "lessons_learned": "Online schema migration + maintenance window protokolü",
  "open_improvement": true  // default true; success olmayan outcome'da otomatik
}
```
→ CSI register'a "Online schema migration adopt et" improvement candidate'ı düşer.

Hotfix + redeploy sonrası **ikinci PIR** kaydı eklenebilir (release_pir composite key değil, n:1 ilişki):
```http
POST /api/issues/{releaseId}/release-pirs
{ "outcome": "success", "success_rating": 4, "lessons_learned": "Online migration başarılı" }
```

---

## Senaryo 4: Contract → Asset → SLA Chain

User: "Acme MSA sözleşmemizin SLA'sı nedir, hangi asset'leri kapsıyor?"

```http
# 1. Contract bul
GET /api/workspaces/{wsId}/contracts
→ filter party_name="Acme"

# 2. Contract attachment'ı yükle (PDF MSA)
POST /api/workspaces/{wsId}/contracts/{id}/attachments
(multipart, file: msa-2026.pdf)

# 3. Agent ile analiz et
POST /api/workspaces/{wsId}/contracts/{id}/analyze
{ "agent_id": "<contract-analyzer-agent>", "prompt": "SLA süreleri, fesih hükümleri ve kapsanan servisleri çıkar" }
→ Yeni issue açılır, agent task çalışır, sonuçlar timeline'a düşer

# 4. Asset'leri contract'a bağla (CMDB tarafı)
# Şu anda asset.contract_id direkt set — handler önümüzdeki sürümde gelecek;
# manuel SQL ile yapılabilir
```

**Agent davranışı:** PDF analizinde **bullet point** formatı, kritik tarihleri **kalın** ve **risk seviyesini** sonda belirt (düşük/orta/yüksek + gerekçe).

---

## Modüller Arası Bağlantı Trace'i

Bir improvement'tan başlayıp tüm zinciri görmek için:
```http
GET /api/workspaces/{wsId}/improvements/{id}/evidence
```

Dönen yapı:
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

Aynı şekilde "bu PIR şu improvement'ları açtı" sorgusu:
```http
GET /api/improvement-sources?source_type=pir&source_ref_id=<pirId>
```
(Bu endpoint şu anda yok — ListImprovementsBySource backend query'si var ama handler yazılmadı. Frontend gerek duydukça eklenecek.)

---

## Anti-Pattern'ler

❌ **Change implement → release submit → PIR atla**. Loop yarım kalır, CSI register beslenmez.

❌ **Improvement status'unu manuel 'done' yap, spawned_change'siz**. ITIL4 "implementation" gerçekten bir change ile yapılmalı; aksi audit trail'i bozar.

❌ **Problem'i incident bağlantısı olmadan kapatma**. ITIL4'te problem her zaman 1+ incident'tan referans alır.

❌ **CAB onayı olmadan high-risk change implement etme**. `cab_review` tablosunda decision='approved' satırı olmadan status'u `in_progress`'e çekme.

❌ **Sözleşmeli asset'i contract bağı olmadan değiştir**. Asset.contract_id var (migration 106) — ileride bu chain SLA tarafına otomatik bağlanacak.
