# Hortora — Project Handoff

*Last updated: 2026-06-10 (session 21 — spec workspace initialised)*

---

## What Hortora Is

**Federated, governed knowledge garden system** — a cross-project, machine-wide library of hard-won technical knowledge (gotchas, techniques, undocumented behaviours). Developers rediscover the same non-obvious bugs and workarounds repeatedly across projects. Hortora captures them once, organises them with quality lifecycle management, and federates them via GitHub-backed canonical gardens.

The **forage** skill (installed: `~/.claude/skills/forage/`) captures entries during sessions. The **harvest** skill deduplicates. The **garden-engine** (Quarkus) provides RAG-quality retrieval via Qdrant + Ollama.

---

## Local Folder Structure

```
~/claude/hortora/
├── spec/                   ← Hortora/spec (protocol spec, ADRs, design docs)
├── soredium/               ← Hortora/soredium (skills, validators, CI scripts)
├── garden-engine/          ← Hortora/garden-engine (Quarkus retrieval service)
└── hortora.github.io/      ← Hortora/hortora.github.io (public website + blog)

~/.hortora/
└── garden/                 ← Hortora/garden (live canonical garden, 1000+ entries)

~/claude/casehub/
└── garden/                 ← casehubio/garden (CaseHub protocol store, 68+ protocols)

~/claude/public/hortora/
├── spec/                   ← Hortora/wsp-spec (this workspace — NEW 2026-06-10)
└── garden-engine/          ← Hortora/wsp-garden-engine
```

---

## The Repos — Current State

### `Hortora/garden-engine` — Phase 1 complete, Phase 2 ready to start

**GitHub:** Hortora/garden-engine · **Workspace:** `~/claude/public/hortora/garden-engine/`

Phase 1 ships: Quarkus 3.36.1 / Java 25 / GraalVM native CI. Scans `~/.hortora/garden`
on startup, embeds via Ollama (`nomic-embed-text`, 768-dim), upserts into Qdrant.
Exposes `GET /search` (REST) and `garden_search` / `garden_status` (MCP HTTP SSE).
All issues #1–#4 closed.

**Design:** `~/claude/hortora/garden-engine/docs/DESIGN.md`

**Phase 2 — IMMEDIATE NEXT WORK:**
SPLADE hybrid search + cross-encoder reranker via `casehub-inference-splade` and
`casehub-inference-tasks`. **Gate cleared 2026-06-10.**

- `casehub-inference-splade` → `SparseEmbedder` for Qdrant named vector space upsert (published ✅)
- `casehub-inference-tasks` → `CrossEncoderReranker` for precision-mode top-N reranking (published ✅)

No garden-engine issue opened yet for phase 2 — open one before writing any code.
Design first (brainstorm), then TDD.

**Spec:** `~/claude/hortora/spec/docs/superpowers/specs/2026-04-22-garden-engine-quarkus-design.md`
**onnx-inference spec:** `~/claude/hortora/spec/docs/superpowers/specs/2026-06-03-onnx-inference-module-design.md`
**LangChain4j upstream issue (Qdrant hybrid):** `langchain4j/langchain4j#4994`

**Pending after phase 2:**
- Federation — canonical/child chain walk (Hortora-specific, no shared module) · L · High
- Incremental re-indexing / file watching (startup-only currently) · M · Med

---

### `Hortora/soredium` — Skills, validators, CI scripts

**GitHub:** Hortora/soredium · **Local:** `~/claude/hortora/soredium/`

Skills: `forage` (CAPTURE/SWEEP/SEARCH/REVISE/IMPORT) and `harvest` (MERGE/DEDUPE) —
both installed to `~/.claude/skills/`. Area 2 Phases 1–5 complete (convention schema,
`validate_schema.py`, `init_garden.py`). 836 tests passing.

**Open issues (long-horizon epics):**
- `#10` — GE-ID scheme migration and post-Phase-2 stabilisation
- `#11` — Phase 3 hybrid staleness enforcement system
- `#12` — Phase 4 full deduplication system
- `#13` — Contextual capture (WHY schema enrichment + contributor scoreboard)
- `#31` — Area 2 knowledge taxonomy: 6-garden schema, validation, tooling

**Pending maintenance tasks (not in issues):**
- CaseHub peer repo CLAUDE.md updates: `connectors`, `ledger`, `work` branches — add
  `protocols → casehub/garden` routing row to each when the branch is on main
- 16 missing protocols in `casehub/garden/docs/protocols/PENDING-MODULE-UPDATES.md` · M · Low

---

### `Hortora/garden` — Live canonical knowledge garden

**GitHub:** Hortora/garden · **Local:** `~/.hortora/garden/`

1000+ entries across domains: jvm/, tools/, casehub-*/. Garden agent installed
(post-commit hook, autonomous dedup). Validate with `validate_garden.py`.

**Open design question — `technology` field:**
51% of entries have empty `tags`, so technology-level retrieval is broken. Proposed:
best-effort `technology: quarkus` field (no certainty required, unlike `root_cause_layer`)
would restore filtering. Decision pending before the `quarkus/` → `jvm/` YAML migration.

**Pending garden work:**
- `technology` field decision · XS · Med
- `quarkus/` → `jvm/` domain YAML migration (~192 entries) — blocked on `technology` decision · M · Med
- Tags backfill (51% empty) · S · Low

**New machine setup:** `cd ~/.hortora/garden && ~/claude/hortora/soredium/scripts/garden-agent-install.sh`

---

### `Hortora/spec` — Protocol spec, ADRs, design docs

**GitHub:** Hortora/spec · **Workspace:** `~/claude/public/hortora/spec/` (this workspace)

Houses the open protocol specification, ADRs, and design documents.
Workspace initialised 2026-06-10. Blog, plans, and methodology artifacts migrated here.

**Open spec issues:**
- `#2` — Phase 2 technical specs and design artifacts
- `#3` — Phase 3–4 and contextual capture design documentation
- `#14` — Epic: Hortora project knowledge methodology (audit series + protocol/garden unification)
- `#15` — Design complete: casehubio/onnx-inference shared ONNX inference module

**ADRs:** `docs/adr/` — ADR-0001 (index-and-lazy-reference), ADR-0002 (CI scripts in soredium),
ADR-0003 (GE-ID scheme: date+6hex), ADR-0004 (SQLite aggregate state)

---

### `casehubio/garden` — CaseHub protocol store

**GitHub:** casehubio/garden · **Local:** `~/claude/casehub/garden/`

Single source of truth for all CaseHub protocols (68+ files). Always on main, no branches.
`docs/protocols/casehub/` + `docs/protocols/universal/`. CLAUDE.md present.

---

### `Hortora/hortora.github.io` — Public website + blog

**Live:** https://hortora.github.io · **Local:** `~/claude/hortora/hortora.github.io/`
Blog entries through session 20 (2026-06-10). Blog posts live at `_posts/`.

---

## Monitoring

- Protocol routing commits on 3 active CaseHub branches (connectors, ledger, work) — no action until merged or abandoned

---

## Key Design Docs and Specs

| Document | Path |
|----------|------|
| Full RAG redesign (the master spec) | `spec/docs/design/2026-04-07-garden-rag-redesign-design.md` |
| Garden ecosystem plan (7 areas) | `spec/docs/design/2026-04-16-garden-ecosystem-plan.md` |
| Hortora project knowledge methodology | `spec/docs/design/2026-05-05-hortora-project-knowledge-methodology.md` |
| garden-engine Quarkus design | `spec/docs/superpowers/specs/2026-04-22-garden-engine-quarkus-design.md` |
| onnx-inference module design | `spec/docs/superpowers/specs/2026-06-03-onnx-inference-module-design.md` |
| Phase 4 full dedup design | `spec/docs/superpowers/specs/2026-04-14-phase4-full-deduplication-design.md` |
| Contextual capture design | `spec/docs/superpowers/specs/2026-04-14-contextual-capture-design.md` |
| Area 2 taxonomy design | `spec/docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md` |
| Garden engine DESIGN.md | `garden-engine/docs/DESIGN.md` |
| ADR-0003 (GE-ID scheme) | `spec/docs/adr/0003-ge-id-scheme-date-plus-random-hex.md` |
| ADR-0004 (SQLite state) | `spec/docs/adr/0004-sqlite-aggregate-state.md` |
| Garden health check (2026-04-15) | `spec/docs/garden-health-2026-04-15.md` |

---

## Priority Order

1. **garden-engine phase 2** — open issue, brainstorm, TDD (garden-engine workspace)
2. **`technology` field decision** → enables quarkus/→jvm/ migration and tags backfill
3. **CaseHub CLAUDE.md routing** — connectors/ledger/work when each hits main
4. **16 missing protocols** in casehub/garden
5. **Soredium epics** (#11 staleness, #12 full dedup, #13 contextual capture, #31 area 2)
