# Caliptic Skills

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

Caliptic ITSM platformu için resmi skill koleksiyonu. Bu skill'leri agent'larınıza
ekleyerek platformu bilen, ITIL4 disiplinine uyan AI teammate'ler elde edersiniz.

## İçerik

| Skill | Açıklama |
|---|---|
| [`caliptic-platform`](skills/caliptic-platform/) | Caliptic ITSM platformunun bütüncül kullanım rehberi — domain model, ITIL4 loop, API rehberi, agent davranış kuralları, task pattern'leri |

## Kurulum

### Yöntem 1: CLI ile URL'den import (önerilen)

```bash
caliptic skill import --url https://skills.sh/caliptic-org/skills/caliptic-platform
```

`skills.sh` GitHub raw content'ini proxy'liyor; private repo'lar için doğrudan `https://github.com/...` URL'si de çalışıyor.

### Yöntem 2: Lokal klasörden seed (development)

Repo'yu klonla, sonra Caliptic CLI ile yükle:

```bash
git clone https://github.com/caliptic-org/skills.git
cd skills

caliptic skill seed-dir skills/caliptic-platform \
  --attach-default-supervisor
```

`--attach-default-supervisor` bayrağı yüklenen skill'i workspace'inin default supervisor agent'ına otomatik bağlar.

### Yöntem 3: Caliptic UI

Settings → Skills → New Skill → İçeriği `SKILL.md`'den kopyala-yapıştır. Destek dosyalarını ayrı ayrı yükle.

## Skill Yapısı

Her skill bir klasör:

```
skills/<skill-name>/
├── SKILL.md          # YAML frontmatter (name, description) + ana içerik
├── <support1>.md     # opsiyonel destek dosyaları
└── <support2>.md
```

`SKILL.md` frontmatter formatı:

```yaml
---
name: skill-name
description: |
  Çok satırlı açıklama (claude-skills format uyumlu).
  Trigger kelimelerini içermesi otomatik aktivasyon için kritik.
---
# Skill body markdown
```

## Versiyon ve Uyumluluk

- Skill'ler **Caliptic server v0.1.36+** ile uyumlu (ITIL4 loop endpoint'leri ihtiyacı).
- Eski sürümlerde `release-pirs`, `spawn-change`, `improvement-evidence` endpoint'leri yok — skill çalışsa da bu özellikler 404 döner.
- `caliptic-platform` skill'i her major Caliptic release'inde güncellenir.

## Katkı

Yeni skill öneriler:

1. Fork → `skills/<your-skill-name>/` altında `SKILL.md` + destek dosyaları
2. PR aç. Reviewer: Caliptic core team

Yerel test:
```bash
caliptic skill seed-dir skills/your-skill --update
```

## License

Apache 2.0 — bkz. [LICENSE](LICENSE).
