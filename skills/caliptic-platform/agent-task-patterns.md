# Agent Task Patterns

Agent Caliptic'te issue'lara atandığında karşılaşacağı tipik görev kalıpları. Her pattern için **input sinyalleri**, **adımlar**, **çıktı kalıpları** ve **yaygın hatalar**.

## Pattern 1: Incident Triage

**Sinyal:** Issue type=incident, status=todo, sana atandı.

**Adımlar:**
1. Description + form_data oku — semptomları çıkar
2. `GET /api/issues?issue_type=incident&status=in_progress&priority=urgent` — benzer aktif vakalar var mı?
3. KB search: `GET /api/workspaces/{wsId}/kb/articles/search?q=<keywords>` — workaround var mı?
4. CMDB lookup: incident etkilenen servisi `cmdb_config_item.name` ile bul → `GET /api/workspaces/{wsId}/cmdb/impact-analysis/{ciId}` → ne diğer servisleri etkiliyor?
5. Status → `in_progress` + yorum:
   - "Aynı imzaya sahip 2 açık incident var (#INC-... ve #INC-...) — bu bir problem candidate."
   - "KB makalesi #KB-42: 'Restart payment pod' workaround önerisi var."
   - "Etki: payment-service + checkout-flow bağımlı, downstream 3 servis."

**Çıktı kalıbı:**
```
## Triage özeti
- **Etkilenen CI**: payment-service-prod
- **Etki kapsamı**: 3 downstream servis (checkout-flow, billing, reporting)
- **Benzer aktif incident**: 2 adet (potansiyel problem)
- **Workaround**: KB-42 (pod restart) — denenebilir
- **Sonraki adım**: Workaround uygula, başarısız olursa SEV-1 yükselt + on-call DBA çağır
```

**Yaygın hata:** Workaround'u dene + RCA yazma. RCA problem-tipi issue'ya gitmeli, incident değil.

---

## Pattern 2: Change Implementation

**Sinyal:** Issue type=change, status=approved, CAB review tamam, sana atandı.

**Pre-flight checklist (yapmadan implement etme!):**
- [ ] `change_type` doğru mu? (`standard`/`normal`/`emergency`)
- [ ] `rollback_plan` boş değil mi? (boşsa doldur!)
- [ ] `test_plan` boş değil mi?
- [ ] Aktif freeze window var mı? `GET /api/workspaces/{wsId}/freeze-windows` — şimdi ile uyuşan kayıt var mı?
- [ ] CAB approve sayısı ≥ minimum? `GET /api/issues/{id}/cab-reviews`

**Adımlar:**
1. Status `approved` → `in_progress`. Backend freeze check yapar; 409 alırsan freeze window'u logla, insan'a yorum bırak.
2. Implementation. `actual_start_at` server tarafında auto-set olabilir (handler bakar).
3. Test çıktısı yorum olarak ekle.
4. Başarılı:
   - Status → `done`
   - `actual_end_at` set (eğer handler set etmiyorsa manuel PUT)
   - Yorum: implementation özeti + verify metoduyla doğrulama
5. Başarısız:
   - Status → `cancelled` (rollback gerekmiyorsa) veya bırak `in_progress` (rollback başlatıldı)
   - Rollback notes yorum olarak

**Yaygın hata:** Rollback gerektiren değişiklikte rollback_plan'a "TBD" yazma. Plan **somut komut/adımlar** içermeli.

---

## Pattern 3: Release Build → Deploy → PIR

**Sinyal:** Issue type=release, sen release coordinator agent.

**Adımlar:**
1. Scope doğrulama:
   ```
   GET /api/issues/{releaseId}/release-changes
   ```
   Boş ise insan'a sor: "Bu release hangi change'leri içeriyor?"

2. Pre-deploy:
   ```
   PATCH /api/issues/{releaseId}/release-phase
   { "release_build_status": "in_progress" }
   ```
   CI/CD pipeline'ı tetikle. Sonuç:
   ```
   PATCH .../release-phase { "release_build_status": "passed" }
   ```

3. Deploy:
   ```
   PATCH .../release-phase { "release_deploy_status": "deploying" }
   ```
   Deploy script çalıştır. Health check'leri izle.

4. Deploy sonucu:
   - Başarılı: `release_deploy_status: deployed`
   - Başarısız: `release_deploy_status: failed` veya `rolled_back`

5. PIR (mecburi):
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

**Yaygın hata:** Deploy başarılı sonra PIR atlayıp release issue'sunu `done` işaretleme. ITIL4'te release döngüsü PIR'sız tamamlanmaz.

---

## Pattern 4: Problem RCA

**Sinyal:** Issue type=problem, status=in_progress, sen RCA owner.

**Adımlar:**
1. İlişkili incident'ları topla:
   ```
   GET /api/issues/{problemId}/related-incidents
   ```
2. Pattern bul: aynı zaman aralığında mı? Aynı service'te mi? Aynı user impact'i mi?
3. CMDB ile ilişkili CI'lar:
   ```
   GET /api/workspaces/{wsId}/cmdb/cis?filter=service
   ```
4. Log/metric analizi (varsa workspace'in monitoring entegrasyonu).
5. RCA güncelle:
   ```
   PATCH /api/issues/{problemId}/rca
   {
     "root_cause": "<somut teknik açıklama>",
     "workaround": "<acil çözüm, geçici>"
   }
   ```
6. Kalıcı çözüm için improvement aç:
   ```
   POST /api/issues/{problemId}/spawn-improvement
   {
     "title": "Implement: <root_cause çözümü>",
     "category": "tool|process|...",
     "expected_benefit": "Eliminate workaround"
   }
   ```

**Çıktı kalıbı:**
```
## RCA — Payment service intermittent 500s

### Root Cause
DB connection pool exhaustion. App max_connections=20, peak observed=50.

### Evidence
- 3 incident son 7 günde aynı imzaya sahip (#INC-201, #INC-205, #INC-211)
- CloudWatch metrics: pool_utilization 95-100% peak saatlerinde
- Application log: "no available connections" exception (psycopg2.OperationalError)

### Workaround (kısa vadeli)
- Pod restart her 6 saatte bir (cron)

### Permanent fix
- Improvement #CSI-0042 açıldı: connection pool 100'e + Hystrix breaker
```

**Yaygın hata:** RCA'yı yorum olarak yazma. `issue.root_cause` ve `issue.workaround` alanları için PATCH /rca kullan — backend audit + reporting bunlara güveniyor.

---

## Pattern 5: Improvement Lifecycle

**Sinyal:** improvement_proposal, sana owner olarak atandı.

**Adımlar:**
1. **Plan** (PDCA):
   ```
   PATCH /api/workspaces/{wsId}/improvements/{id}
   {
     "plan_notes": "1. Connection pool 100'e çıkar (config map) 2. Hystrix breaker libray ekle 3. Load test 4. Production rollout",
     "target_date": "2026-06-15"
   }
   ```

2. **Status → reviewing → approved** (yetkili tarafından):
   ```
   PATCH .../improvements/{id} { "status": "approved" }
   ```

3. **Implement** (change spawn):
   ```
   POST /api/workspaces/{wsId}/improvements/{id}/spawn-change
   { "title": "DB pool 100 + circuit breaker", "change_type": "normal", "risk_level": "medium" }
   ```
   → Yeni issue, `issue.spawned_from_improvement_id` set. Status → `implementing`.

4. **Check** (change tamamlandıktan sonra):
   ```
   PATCH .../improvements/{id}
   {
     "do_notes": "Change #CHG-PROD-0042 implement edildi",
     "check_notes": "1 hafta production izlendi, pool peak %70, restart cron kaldırıldı",
     "actual_benefit": "Workaround removed, p99 latency -30%"
   }
   ```

5. **Act** + **done**:
   ```
   PATCH .../improvements/{id}
   {
     "act_notes": "Pool monitoring metric eklendi, runbook güncellendi",
     "status": "done"
   }
   ```

**Yaygın hata:** PDCA fazlarını boş bırakma. `actual_benefit` ölçülmemişse improvement "done" değildir — ITIL4 audit'i bunu yakalar.

---

## Pattern 6: Contract Analysis

**Sinyal:** `POST /contracts/{id}/analyze` ile tetiklenen task, sana atandı.

**Adımlar:**
1. Description'dan attachment URL'lerini al
2. PDF/DOCX content extract (skill'in dış araç gerektirebilir — anthropic-skills:pdf veya benzeri kullanılabilir)
3. **Standart kontrol listesi**:
   - Taraflar (party_name, vendor adresi, vergi no)
   - Başlangıç / bitiş tarihleri
   - Toplam değer + para birimi
   - Otomatik yenileme şartı + bildirim süresi (notice_days)
   - Fesih hükümleri (notice period, breach trigger)
   - Gizlilik / NDA hükümleri
   - SLA taahhütleri (uptime %, response time, penalty)
   - KVKK/GDPR uyumu (varsa)
4. **Risk seviyesi**:
   - Düşük: standart NDA, açık fesih, makul SLA
   - Orta: otomatik yenileme + uzun notice (kilit), belirsiz fesih şartı
   - Yüksek: süresiz, ağır SLA penalty, tek taraflı fesih hakkı satıcıda

**Çıktı kalıbı** (issue timeline'a yorum olarak):
```
## Acme MSA — Analiz

### Taraflar
- **Customer**: Caliptic Yazılım A.Ş.
- **Vendor**: Acme Corp (TX no: 1234567890)

### Anahtar tarihler
- Başlangıç: 2026-01-01
- Bitiş: **2027-12-31** (24 ay)
- Bildirim süresi: 60 gün
- Auto-renew: ✅ EVET (90 gün önce bildirim yoksa 12 ay uzar)

### Mali
- Yıllık ücret: 50,000 USD
- Toplam: 100,000 USD
- Erken fesih cezası: kalan dönem %50'si

### SLA
- Uptime: 99.9% (aylık)
- Penalty: %1 SLA breach başına kredit

### Risk seviyesi: **ORTA**
- Auto-renew + 60 gün notice kısa (kontrol zorluğu)
- Erken fesih cezası ağır (kalan %50)
- + Net SLA + makul penalty (olumlu)

### Öneriler
- Renewal alarm: 2027-09-30 (90 gün öncesi)
- Fesih riski: Quarter sonu değerlendirme döngüsüne ekle
```

**Yaygın hata:** Risk seviyesini "düşük" deyip gerekçe yazmama. Her seviye **kanıtla** verilmeli.

---

## Genel Davranış Prensipleri

1. **Audit trail önemli** — her status değişikliği yorum + sebep ile.
2. **Cross-link bilinçli yap** — issue'lar birbirine bağlanmalı (related, blocks, parent), keyfi değil.
3. **Workspace izolasyonu** — `workspace_id` her sorguda var; cross-workspace ID guessing backend tarafından 404 dönüyor zaten ama agent kendisi de denemesin.
4. **Module gate kontrol et** — calipticore module disable ise (örn: catalog kapalı), agent yine de service_request issue açabilir ama UI gözükmez; soruna sebep olabilir, kontrol et.
5. **Agent kendi durumunu doğru ifade et** — "yapamadım çünkü X" > "başarısız". Detay > belirsizlik.
