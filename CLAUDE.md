# spec Workspace

**Name:** spec
**Project repo:** /Users/mdproctor/claude/hortora/spec
**Workspace type:** public

@/Users/mdproctor/claude/hortora/spec/CLAUDE.md

## What Hortora Is

Federated, governed knowledge garden system — machine-wide library of hard-won technical knowledge (gotchas, techniques, undocumented behaviours). Entries live in `~/.hortora/garden/`. The **forage** skill captures during sessions; **harvest** deduplicates; **garden-engine** (Quarkus) provides RAG search via Qdrant + Ollama.

**Key repos:**
- `~/claude/hortora/spec/` — this repo: protocol spec, ADRs, design docs
- `~/claude/hortora/soredium/` — skills (forage, harvest), validators, CI
- `~/claude/hortora/garden-engine/` — Quarkus retrieval service (phase 2 is the next build)
- `~/claude/hortora/hortora.github.io/` — public website + blog
- `~/.hortora/garden/` — live canonical garden (Hortora/garden)
- `~/claude/casehub/garden/` — CaseHub protocol store (casehubio/garden)

**Immediate work:** garden-engine phase 2 (SPLADE + cross-encoder). Gate cleared 2026-06-10. Open that session in the garden-engine workspace: `~/claude/public/hortora/garden-engine/`.

## Session Start

Run `add-dir /Users/mdproctor/claude/hortora/spec` and `add-dir /Users/mdproctor/claude/public/hortora/spec` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| adr | `adr/` |
| write-content | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`/Users/mdproctor/claude/public/hortora/spec`) — plans, blog (staging), snapshots, handover
- **Project repo** (`/Users/mdproctor/claude/hortora/spec`) — source code, ADRs (`docs/adr/`), specs

Never rely on CWD for git operations — the session may have started in either repo. Always use explicit paths:
```bash
git -C /Users/mdproctor/claude/public/hortora/spec add <file>    # workspace artifacts
git -C /Users/mdproctor/claude/hortora/spec add <file>            # project artifacts
```

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at work end |
| specs      | project     | lands in `docs/specs/` — promoted at work end |
| blog       | workspace   | staged here; published to hortora.github.io via publish-blog at work end |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | epic journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |

**Blog directory:** `/Users/mdproctor/claude/public/hortora/spec/blog/`
