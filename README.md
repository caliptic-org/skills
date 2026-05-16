# Caliptic Skills

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

Official skill collection for the **Caliptic** ITSM platform.
Attach these skills to your agents and they become teammates that
genuinely *know* the platform — its API, its ITIL4 service value
chain, and the operating discipline expected of a first-class
agent role.

---

## What is Caliptic?

**Caliptic is an AI-native ITSM platform.** Imagine Linear or Jira,
but built from the ground up around a different premise: **AI agents
are first-class citizens, not bolt-on chatbots**. They get assigned
issues, leave comments, change status, link records, and act with the
same authority a human teammate has — bounded by per-agent permissions
and a full audit trail.

Caliptic also takes **ITIL4 discipline seriously**. Most ITSM tools
ship the vocabulary (incident, problem, change, release, CSI) but
leave them as disconnected modules; teams end up wiring the workflow
together with conventions, spreadsheets, and tribal memory. Caliptic
wires the workflow **at the database level**:

```
[Incident] ──┐
[Problem]  ──┤── RCA → [Change Request] ──CAB──▶ [Release Package]
             │                                          │
             ▼                                          ▼
                                              [Post-Implementation Review]
                                                          │
                                                          ▼
                                              [CSI Improvement] ──▶ new Change
                                                                    (loop)
```

Every arrow in that diagram is a physical foreign key or junction
table. The platform refuses to let the loop go half-open: you can't
ship a release without a PIR record, you can't bypass a freeze
window without escalating to `change_type='emergency'`, and a problem
that's never linked to an incident is a red flag the system tracks.

### What's in the box

- **Multi-tenant SaaS + on-premise**. Workspaces are the tenant
  boundary; every query is `workspace_id`-scoped and cross-workspace
  access returns 404.
- **Polymorphic Issue model** — `task`, `incident`, `problem`,
  `change`, `service_request`, `release`. One table, different
  behaviour per type.
- **Configurable agents** — runtime (local daemon vs cloud),
  visibility (private vs workspace-shared), supervisor scoping per
  project, per-user access lists.
- **CMDB with impact analysis** — configuration items, relationships
  (`depends_on`, `hosted_on`, `connected_to`, …), and topology with
  service-centric or infrastructure views.
- **Contracts + agent analysis** — upload PDFs/DOCX; an agent can
  extract parties, dates, SLA terms, risk level on demand.
- **Service Catalog with auto-routing** — catalog items spawn
  service-request issues into the right project with the right SLA.
- **SLA engine** — response + resolution stages, pause-on-status,
  business-hours-aware breach detection, escalation groups.
- **Continual Improvement Register (CSI)** with PDCA tracking and
  full source provenance (PIR / problem RCA / incident trend / audit
  / KB feedback / manual).
- **Real-time everything** — WebSocket event bus invalidates query
  caches; multi-tab, multi-device updates within milliseconds.
- **Three frontends, one core**: a Next.js web app, an Electron
  desktop app, and a Go CLI. All share the same `packages/core` +
  `packages/views` + `packages/ui` monorepo.

### Where Caliptic sits in the ITSM landscape

| Tool family | Examples | Caliptic vs. them |
|---|---|---|
| **General work trackers** | Linear, Jira, Asana | Caliptic ships ITIL4 modules out of the box and ships AI agents as first-class actors, not integrations |
| **Heavyweight ITSM** | ServiceNow, BMC Helix | Caliptic is API-first, modern stack, multi-tenant SaaS option, dramatically faster to deploy on-prem |
| **Lightweight help desks** | Zendesk, Freshservice | Caliptic carries the full ITIL4 module set (CMDB, CSI, CAB, freeze windows, contracts) instead of just incidents |
| **DevOps tools** | PagerDuty, Atlassian Compass | Caliptic adds change management, release coordination, and ITIL4 audit on top of the modern DevOps surface |

The audience: teams of ~5–500 people who want ITIL4 discipline without
the ServiceNow weight, and who want their AI to operate inside the
work tracker rather than next to it.

---

## What this repo contains

This repository is the **official skill registry** for Caliptic.
Each subdirectory under `skills/` is a self-contained
[Anthropic-format skill](https://docs.anthropic.com) — a `SKILL.md`
with YAML frontmatter and any supporting `.md` files that go
alongside it.

| Skill | What it teaches | When to use it |
|---|---|---|
| [`caliptic-platform`](skills/caliptic-platform/) | The Caliptic domain model, the ITIL4 loop, the REST API, the agent behavioural rules, six task patterns (triage, change implementation, release+PIR, problem RCA, improvement lifecycle, contract analysis) | Attach to your workspace's default supervisor — every agent then inherits operating knowledge of the platform |

The `caliptic-platform` skill is the **canonical knowledge base** for
agents working inside Caliptic. Once attached to the workspace's
default supervisor, every worker agent under that supervisor inherits
the skill via the supervisor/worker scope mechanism — no per-agent
configuration needed.

---

## Installing a skill

### Method 1: Import from URL (recommended)

```bash
caliptic skill import --url https://skills.sh/caliptic-org/skills/caliptic-platform
```

`skills.sh` is a thin proxy over `raw.githubusercontent.com`; behind
the scenes Caliptic fetches the `SKILL.md` and supporting files
directly from this repo. No registration on skills.sh is required —
the URL convention `skills.sh/{owner}/{repo}/{skill-name}` works as
long as the GitHub repo is public.

### Method 2: Clone + seed from a local directory

```bash
git clone https://github.com/caliptic-org/skills.git
cd skills

caliptic skill seed-dir skills/caliptic-platform \
  --attach-default-supervisor
```

The `--attach-default-supervisor` flag automatically attaches the
freshly-imported skill to your workspace's default supervisor.

### Method 3: Built-in (Caliptic server v0.1.37+)

The `caliptic-platform` skill is bundled into the Caliptic server
binary via `go:embed`. The moment a workspace's default supervisor is
set (`PUT /api/workspaces/{wsId}/default-supervisor`), the skill is
auto-created in the workspace and attached to the supervisor — no
manual step needed.

This auto-attach is **opt-out**: users can detach the skill from the
supervisor at any time via Settings → Agents → Skills. The hook fires
only when the default supervisor is set; it doesn't force re-attach
on every request.

### Method 4: Caliptic UI

Settings → Skills → New Skill → paste the `SKILL.md` content. Upload
supporting files one at a time. Then on the Agent detail page, add
the skill to the agent's skill set.

---

## Skill format

Each skill is a directory:

```
skills/<skill-name>/
├── SKILL.md          # YAML frontmatter (name, description) + main content
├── <support1>.md     # optional supporting files
└── <support2>.md
```

The `SKILL.md` frontmatter follows the standard:

```yaml
---
name: skill-name
description: |
  Multi-line description.
  Include trigger keywords here — agents auto-activate skills
  based on these.
---
# Skill body in Markdown
```

The `description` field is **especially important**: Anthropic-style
skills activate based on the description matching the user's intent.
Be specific about the triggers (verbs, nouns, scenarios).

---

## Version & compatibility

- Skills target **Caliptic server v0.1.36+** (ITIL4 loop endpoints
  introduced in migration 105 are required).
- On older servers the skill content still loads, but features like
  `release-pirs`, `spawn-change`, and `improvement-evidence` will
  return 404.
- The `caliptic-platform` skill is updated alongside major Caliptic
  releases; see [CHANGELOG.md](CHANGELOG.md).

---

## Contributing a new skill

1. Fork this repo.
2. Add `skills/<your-skill-name>/SKILL.md` + supporting files.
3. Test locally:
   ```bash
   caliptic skill seed-dir skills/<your-skill-name> --update
   ```
4. Open a pull request. The Caliptic core team reviews and merges.

Quality bar:
- The `description` field must include trigger keywords specific
  enough that an agent won't activate the skill on unrelated prompts.
- Examples are gold. Include at least one concrete API request /
  response in the body or a supporting file.
- Anti-patterns matter as much as patterns. Tell the agent what *not*
  to do.

---

## Where to learn more

- **Caliptic main repo**: <https://github.com/caliptic-org/caliptic>
- **Documentation**: <https://app.caliptic.com/docs>
- **Issues & discussion**: file in the main repo
- **License**: Apache 2.0 — see [LICENSE](LICENSE)
